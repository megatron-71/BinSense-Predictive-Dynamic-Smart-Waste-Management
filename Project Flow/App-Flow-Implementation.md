# App Flow & Implementation Document
## BinSense — Predictive and Dynamic Smart Waste Management

**Document Version:** 1.0
**Companion to:** PRD-Smart-Waste-Management.md, TRD-Smart-Waste-Management.md
**Purpose:** Describe the end-to-end application flow (screen by screen) and the concrete implementation plan to build it as a working prototype using free/open-source tools.

---

## 1. Purpose of This Document

The PRD defines *what* the system should do, and the TRD defines *what tools/algorithms* power it. This document bridges the two by specifying:
1. The **app flow** — every screen the operator sees, in order, with what happens on each.
2. The **implementation plan** — how each screen and backend module actually gets built, in what order, with which files/functions.

---

## 2. Actors

| Actor | Role in the flow |
|---|---|
| **Operator / Dispatcher** | Municipal staff who opens the app each cycle, reviews bin urgency, confirms the collection run, and dispatches vehicles |
| **System (BinSense engine)** | Runs prediction, priority scoring, bin selection, and route optimization automatically in the background |
| **Driver (future scope)** | Would receive the finalized route on a device — out of scope for the prototype's UI, included only as a data hand-off point |

---

## 3. App Flow (Screen by Screen)

### Screen 1 — Dashboard (Today's Overview)
**What the operator sees:**
- Count of bins by urgency tier (Critical / High / Medium / Low)
- Fleet status: vehicles available vs. total
- Fleet capacity utilization from the last cycle
- Predicted city-wide overflow risk indicator

**What happens behind the scenes:**
- On load, the system pulls the latest bin dataset, runs the **Fill Prediction module** for every bin, and aggregates results into the summary counts shown here.

**User action:** Reviews the overview, taps through to the Priority Map to see *where* the urgent bins are.

---

### Screen 2 — Priority Map
**What the operator sees:**
- A map of the service area with every bin plotted as a colored dot (Critical = red/orange, High = amber, Medium = yellow, Low = green)
- Depot marked distinctly
- Tapping a bin shows: current fill %, predicted fill %, time-to-overflow, last collection time

**What happens behind the scenes:**
- Bin coordinates + urgency tier come from the **Dynamic Priority Engine**, which combines current fill, predicted fill, accumulation rate, overflow risk, and time since last collection into a single score, then buckets it into a tier.

**User action:** Gets a spatial sense of where problems are concentrated, then moves to the ranked list for a working queue.

---

### Screen 3 — Priority Queue
**What the operator sees:**
- A ranked list of bins (highest priority first), each showing: bin ID, location name, urgency tier, priority score
- Ability to include/exclude specific bins from today's run (manual override)
- A running count of "bins selected for this cycle"

**What happens behind the scenes:**
- This is the **Bin Selection module**: it filters the full bin list down to the set that should be serviced this cycle, based on tier plus available fleet capacity. Manual overrides here feed directly into the next screen.

**User action:** Confirms the working list, taps "Build Routes."

---

### Screen 4 — Route Optimizer
**What the operator sees:**
- Map showing the selected bins grouped into routes, one color per vehicle, drawn along real roads (not straight lines)
- Per-vehicle summary: number of stops, total distance, estimated load vs. capacity
- A "before/after" toggle comparing this route plan against a naive straight-line/fixed-schedule baseline (distance and trip-count difference)

**What happens behind the scenes:**
- The **Road Network module** computes a real distance/time matrix between depot and selected bins using OpenStreetMap data.
- The **CVRP Solver (Google OR-Tools)** takes that matrix plus vehicle capacities and produces an optimized per-vehicle stop sequence.

**User action:** Reviews the routes, adjusts vehicle count/capacity if needed (system re-solves), then taps "Dispatch."

---

### Screen 5 — Fleet Dispatch & Tracking
**What the operator sees:**
- Live status per vehicle: en route / loading at depot / standby, with current stop progress (e.g., "stop 2 of 3")
- Summary metric: distance saved vs. baseline for this cycle
- A "Complete Cycle" action that logs actual collection outcomes

**What happens behind the scenes:**
- Route data is handed off (in the prototype, simulated; in production, pushed to a driver device).
- Once marked complete, actual collection results (which bins were emptied, at what fill level) are written back into the bin history — closing the loop that feeds the *next* cycle's prediction.

**User action:** Marks the cycle complete; the app returns to the Dashboard, now reflecting updated data.

---

## 4. Flow Diagram (Screen-Level)

```
 Dashboard
     │  (tap: view urgency)
     ▼
 Priority Map ──(tap a bin)──► Bin Detail popover
     │  (tap: view as list)
     ▼
 Priority Queue ──(manual include/exclude)──┐
     │ (tap: Build Routes)                   │
     ▼                                       │
 Route Optimizer ◄──────────────────────────┘
     │ (tap: Dispatch)
     ▼
 Fleet Dispatch & Tracking
     │ (tap: Complete Cycle)
     ▼
 back to Dashboard (data refreshed)
```

---

## 5. Implementation Plan

### 5.1 Build Order (Backend First, UI Layered On Top)

| Step | Module | Output |
|---|---|---|
| 1 | Bin & vehicle data model + simulator | `bins.csv`, `vehicles.csv`, seeded fill-history generator |
| 2 | Fill prediction | Function: `predict_fill(bin_history, horizon) → predicted_pct, time_to_overflow` |
| 3 | Dynamic priority engine | Function: `score_bin(bin) → priority_score, tier` |
| 4 | Bin selection | Function: `select_bins(all_bins, fleet_capacity) → selected_bins` |
| 5 | Road network + distance matrix | Function: `build_distance_matrix(depot, selected_bins) → matrix` (OSMnx + NetworkX) |
| 6 | CVRP solver | Function: `solve_routes(matrix, vehicles) → per_vehicle_routes` (Google OR-Tools) |
| 7 | Baseline comparator | Function: `baseline_routes(...)` using straight-line distance + fixed schedule, for the before/after metric |
| 8 | Visualization/dashboard | Streamlit app wiring screens 1–5 to the functions above |
| 9 | End-to-end test run | Full cycle on simulated data, sanity-check outputs |
| 10 | Demo polish | Seed a "good" demo scenario, cache OSM data offline, rehearse walkthrough |

### 5.2 Screen-to-Module Mapping

| Screen | Reads from | Calls |
|---|---|---|
| Dashboard | Bin dataset | `predict_fill()` for all bins (aggregate view) |
| Priority Map | Bin dataset + predictions | `score_bin()` for all bins |
| Priority Queue | Scored bins | `select_bins()` |
| Route Optimizer | Selected bins | `build_distance_matrix()` → `solve_routes()` → `baseline_routes()` for comparison |
| Fleet Dispatch | Solved routes | Writes back simulated "completed" fill data to bin history |

### 5.3 Suggested Repo Structure

```
binsense/
├── data/
│   ├── bins.csv
│   ├── vehicles.csv
│   └── road_graph.graphml        # cached OSM graph for offline demo
├── src/
│   ├── simulate.py                # bin/vehicle data + fill-history generator
│   ├── predict.py                 # fill-level prediction
│   ├── priority.py                # dynamic priority engine
│   ├── select.py                  # bin selection logic
│   ├── network.py                 # OSMnx graph + distance/time matrix
│   ├── routing.py                 # OR-Tools CVRP solver
│   ├── baseline.py                # naive threshold + straight-line comparator
│   └── config.py                  # tunable weights, thresholds, horizon, fleet size
├── app.py                         # Streamlit app (5 screens)
├── requirements.txt
└── README.md
```

### 5.4 Configuration (No Hardcoding)

All of the following should live in `config.py` (or a YAML file) so they can be adjusted live during the competition demo:
- Priority score weights (`w1..w5`)
- Tier thresholds (Critical/High/Medium/Low cutoffs)
- Prediction horizon (hours)
- Number of vehicles and their capacities
- Demo area / bounding box for the OSM road graph

### 5.5 Demo Data Readiness Checklist
- [ ] OSM road graph for the chosen demo area pre-downloaded and saved locally (`.graphml`)
- [ ] Bin dataset generated with a fixed random seed and visually checked (spread of urgency tiers looks realistic)
- [ ] At least one scenario prepared where the "before/after" comparison shows a clear, presentable improvement
- [ ] Full flow (Dashboard → Dispatch) tested end-to-end without internet dependency

---

## 6. Error / Empty States (App Behavior)

| Situation | App response |
|---|---|
| No bins reach a servicing tier this cycle | Dashboard and Queue show "No bins need collection this cycle" — not an error, a valid state |
| Selected bins exceed total fleet capacity | Route Optimizer flags the shortfall and lists which lowest-priority selected bins were deferred to next cycle |
| Road graph data unavailable for a bin's coordinates | Bin is flagged "location outside network coverage" rather than silently dropped |

---

## 7. Next Steps After Prototype

- Replace simulated bin data with real IoT sensor feed (data model already sensor-agnostic — see TRD Section 4.1)
- Add time-window and service-time constraints to the CVRP model
- Introduce live traffic-aware travel time instead of static road-network estimates
- Extend to multi-depot optimization

---

*End of App Flow & Implementation Document.*
