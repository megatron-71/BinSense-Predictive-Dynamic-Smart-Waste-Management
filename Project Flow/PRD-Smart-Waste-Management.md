# Product Requirements Document (PRD)
## Predictive and Dynamic Smart Waste Management with Road-Network-Based Multi-Vehicle Optimization

**Document Version:** 1.0
**Prepared for:** Research League Competition
**Status:** Draft

---

## 1. Executive Summary

Urban solid-waste collection today largely relies on fixed schedules or simple threshold-based triggers ("collect when bin > 80% full"). Both approaches waste vehicle time, fuel, and manpower — either by collecting bins that are barely used, or by reacting too late to bins that overflow. This project proposes a system that moves waste collection through three levels of intelligence — **Reactive → Predictive → Optimized** — by combining fill-level forecasting, dynamic multi-factor prioritization, and road-network-aware multi-vehicle route optimization (CVRP solved via Google OR-Tools).

The prototype will demonstrate this full pipeline end-to-end on simulated (or optionally real) city data, producing a working, visual, judge-ready demo suitable for a research/innovation competition.

---

## 2. Problem Statement

| Current Approach | Limitation |
|---|---|
| Fixed schedule collection | Bins collected regardless of actual fill level → wasted trips |
| Single-threshold reactive collection | Bins are only flagged after crossing a fixed % — no foresight, risk of overflow between checks |
| Straight-line / geographic distance routing | Does not reflect real road structure, turns, one-ways, or travel time |
| Single-vehicle or unconstrained routing | Ignores vehicle capacity limits and does not scale to real fleets |

**Core question the current state of the art fails to answer:**
*"Which bins will actually need collection soon, how urgent is each one, which vehicle should service it, and what real-road route should that vehicle take?"*

---

## 3. Goals and Objectives

### 3.1 Primary Goals
- Build a working prototype that predicts bin fill levels ahead of time.
- Dynamically compute a priority/urgency score per bin using multiple signals, not a single threshold.
- Select the set of bins that require servicing in the current planning cycle.
- Solve a Capacitated Vehicle Routing Problem (CVRP) across multiple vehicles using **road-network** distances/times (not straight-line).
- Visually demonstrate the full pipeline in a way that is clear and compelling to competition judges.

### 3.2 Secondary Goals
- Show measurable improvement over baseline (fixed schedule / naive threshold + straight-line routing) using simulated data.
- Architect the system so it can later plug into real IoT sensors, live traffic data, and heterogeneous vehicle fleets.

### 3.3 Non-Goals (Out of Scope for Prototype)
- Real hardware/IoT sensor deployment (bin data will be simulated or use a public dataset).
- Live traffic-aware dynamic re-routing during vehicle operation (static per-cycle optimization only, for v1).
- Mobile driver-facing navigation app (route output only; not turn-by-turn navigation UI).
- Multi-depot optimization (single depot assumed for v1).

---

## 4. Users / Stakeholders

| Stakeholder | Interest |
|---|---|
| Municipal waste management authority | Reduced operational cost, fewer overflow complaints |
| Collection vehicle fleet operators | Efficient, capacity-aware routes; less idle/wasted travel |
| Citizens | Fewer overflowing public bins, cleaner city |
| Competition judges | Clear demonstration of technical novelty, feasibility, and impact |

---

## 5. System Overview / Conceptual Workflow

```
Waste Bin Data
      │
      ▼
Current Fill Monitoring
      │
      ▼
Historical Analysis
      │
      ▼
Fill-Level Prediction
      │
      ▼
Overflow-Risk Estimation
      │
      ▼
Dynamic Priority Engine
      │
      ▼
Required Bin Selection
      │
      ▼
Road-Network Information
      │
      ▼
Multi-Vehicle CVRP Model  (Google OR-Tools)
      │
      ▼
Route Optimization
      │
      ▼
Vehicle Assignment & Routes
      │
      ▼
Collection Execution
      │
      ▼
Updated Collection Data ──────► (feeds back into Prediction)
```

---

## 6. Functional Modules & Requirements

### Module 1 — Bin Data & Monitoring
- **FR-1.1:** System shall maintain per-bin records: bin ID, location (lat/long), capacity, current fill %, fill history (time series), last collection timestamp.
- **FR-1.2:** System shall support simulated fill-rate generation (e.g., varying accumulation rates per bin/zone) as a stand-in for real IoT sensors.
- **FR-1.3:** Data model shall be sensor-agnostic, so real IoT feeds can later replace the simulator without changing downstream modules.

### Module 2 — Fill-Level Prediction
- **FR-2.1:** System shall forecast each bin's fill level over a configurable future horizon (e.g., next 6–24 hours) using historical fill-rate patterns.
- **FR-2.2:** Prediction approach for prototype: lightweight time-series/regression method (e.g., linear trend, moving-average-based rate extrapolation, or simple ML regressor) — chosen for explainability and speed over deep learning, given prototype scope.
- **FR-2.3:** Output: predicted fill % at horizon + estimated time-to-overflow per bin.

### Module 3 — Dynamic Priority Engine
- **FR-3.1:** System shall compute a composite priority score per bin using at least: current fill %, predicted fill %, accumulation rate, estimated overflow risk, and time since last collection.
- **FR-3.2:** System shall classify bins into urgency tiers: **Critical / High / Medium / Low**.
- **FR-3.3:** Scoring weights shall be configurable (not hardcoded) to allow tuning/demo scenarios.

### Module 4 — Bin Selection
- **FR-4.1:** System shall select the subset of bins to be serviced in the current planning cycle based on urgency tier and available vehicle capacity/fleet size.

### Module 5 — Road Network & Distance/Time Matrix
- **FR-5.1:** System shall compute a road-network-based distance and travel-time matrix between depot and selected bins (not straight-line/Euclidean).
- **FR-5.2:** Prototype implementation option: OpenStreetMap road graph (via OSMnx) + shortest-path computation, or a routing API (e.g., OSRM), depending on network access in the demo environment.

### Module 6 — Multi-Vehicle Route Optimization (CVRP)
- **FR-6.1:** System shall formulate the selected-bin collection problem as a Capacitated Vehicle Routing Problem.
- **FR-6.2:** System shall solve the CVRP using **Google OR-Tools**, respecting: vehicle capacity, single depot start/end, number of available vehicles.
- **FR-6.3:** System shall output, per vehicle: ordered bin visit sequence, route distance, estimated route time, load carried.
- **FR-6.4:** Architecture shall allow future extension with service time, time windows, and vehicle availability constraints without redesign.

### Module 7 — Visualization / Demo Layer
- **FR-7.1:** System shall visually display bins (color-coded by urgency), depot, and optimized multi-vehicle routes on a map.
- **FR-7.2:** System shall show a before/after comparison: naive (threshold + straight-line) vs. proposed (predictive + road-network CVRP) — e.g., total distance, number of trips, bins at overflow risk.
- **FR-7.3:** System shall present key metrics (distance saved, overflow incidents avoided, vehicle utilization) in a simple dashboard suitable for a live competition demo.

---

## 7. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance | Route optimization for a demo-scale scenario (e.g., 20–50 bins, 2–5 vehicles) should complete in a few seconds, suitable for live demo |
| Usability | Judges/non-technical viewers should be able to understand the map/dashboard output without deep explanation |
| Modularity | Each module (prediction, priority, routing) should be independently swappable/upgradable |
| Reproducibility | Simulated data generation should be seed-controlled for consistent demo runs |
| Portability | Prototype should run locally (no dependency on paid/private APIs where possible) |

---

## 8. Proposed Tech Stack

| Layer | Technology |
|---|---|
| Core language | Python |
| Prediction | pandas, scikit-learn / statsmodels (simple regression/time-series) |
| Priority engine | Custom scoring logic (Python) |
| Road network | OSMnx + NetworkX (OpenStreetMap), or OSRM if network access allows |
| Route optimization | Google OR-Tools (CVRP solver) |
| Visualization | Folium / Plotly (map + charts), optionally packaged as a simple web dashboard |
| Data storage | CSV / JSON for prototype; extensible to a DB later |

*(Final stack to be confirmed based on the environment used for the live prototype — see open questions.)*

---

## 9. What Is Genuinely New vs. Prior Work

| Previous Work | This Proposal |
|---|---|
| Decide which bins to collect using current fill level | Predict future fill level and overflow risk, not just current state |
| Fixed-threshold based bin flagging | Multi-factor dynamic priority scoring with urgency tiers |
| Single-vehicle / geographic-distance routing | Multi-vehicle, capacity-constrained, **road-network-based** CVRP via OR-Tools |

---

## 10. Success Metrics (for Prototype Evaluation)

- **% reduction in total travel distance** vs. naive threshold + straight-line baseline.
- **% of bins serviced before predicted overflow** vs. after (reactive baseline).
- **Vehicle capacity utilization** (average load carried vs. capacity).
- **Number of unnecessary trips avoided** (bins that would have been visited under fixed schedule but weren't needed).

---

## 11. Competition Demo Plan

1. **Problem framing slide/section** — reactive vs. threshold vs. proposed (30 seconds).
2. **Live/recorded demo** — simulated city map with bins appearing, fill levels changing, priority tiers highlighting, then optimized multi-vehicle routes drawn on real roads.
3. **Comparison view** — baseline vs. proposed metrics side-by-side.
4. **Architecture walkthrough** — the 12-step conceptual pipeline diagram.
5. **Future scope** — real IoT integration, live traffic, heterogeneous fleets, time windows.

---

## 12. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| No real sensor/road data available at prototype time | Use realistic simulation + OpenStreetMap public road data |
| Road-network API access may be restricted in demo environment | Pre-fetch/cache road graph for demo area in advance |
| Prediction model may look like a "toy" if overly simple | Keep it explainable and clearly framed as a prototype-scope choice, with a stated path to more advanced models (LSTM/Prophet) as future work |
| Judges may ask about real-world scalability | Prepare a clear "future scope" answer: multi-depot, live traffic, larger fleets, real sensors |

---

## 13. Open Questions (to confirm before build)

1. Prototype format: interactive web dashboard vs. Python/Jupyter notebook vs. both.
2. Data source: fully simulated vs. real bin/road data supplied by team vs. simulated bins on a real city's road network (via API).
3. Target demo scale (number of bins, vehicles, depot count) for the competition presentation.
4. Time budget available before the competition deadline (affects how much of "future scope" gets partially implemented vs. left as roadmap).

---

## 14. Milestones (Suggested)

| Phase | Deliverable |
|---|---|
| Phase 1 | Data simulation module + bin data model |
| Phase 2 | Fill prediction module |
| Phase 3 | Dynamic priority engine |
| Phase 4 | Road-network distance/time matrix |
| Phase 5 | CVRP solver integration (OR-Tools) |
| Phase 6 | Visualization/dashboard layer |
| Phase 7 | Baseline-vs-proposed comparison + metrics |
| Phase 8 | Demo polish & presentation packaging |

---

*End of PRD.*
