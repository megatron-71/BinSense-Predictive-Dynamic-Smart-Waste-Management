# Implementation Plan
## BinSense — Predictive and Dynamic Smart Waste Management (Research League Prototype)

**Document Version:** 1.0
**Companion to:** PRD-Smart-Waste-Management.md, TRD-Smart-Waste-Management.md, App-Flow-Implementation.md
**Purpose:** A concrete, phase-by-phase execution plan — tasks, order, dependencies, timeline, ownership, and acceptance criteria — for building the working prototype ahead of the competition.

---

## 1. Objective of This Plan

Turn the PRD (what to build) and TRD (what to build it with) into a sequence of executable tasks with clear completion criteria, so the team can track progress day-by-day and avoid last-minute scramble before the demo.

---

## 2. Scope Recap

**In scope for the prototype:**
- Simulated bin + fill-history data
- Fill-level prediction
- Dynamic priority scoring
- Bin selection
- Real road-network distance/time matrix (OSM)
- Multi-vehicle CVRP solving (OR-Tools)
- Baseline-vs-proposed comparison
- 5-screen interactive dashboard (Streamlit)

**Out of scope for the prototype:** real IoT sensors, live traffic, multi-depot, driver navigation app (see PRD Section 3.3).

---

## 3. Phased Plan

### Phase 0 — Setup (Day 0–1)
| Task | Detail | Owner | Done when |
|---|---|---|---|
| Repo & environment | Create GitHub repo, `venv`/conda env, `requirements.txt` | Any | `pip install -r requirements.txt` runs clean |
| Install core libraries | pandas, scikit-learn, osmnx, networkx, ortools, streamlit, folium/plotly | Any | Import test script runs without error |
| Pick demo area | Choose a real city/neighborhood bounding box for OSM data (small enough to download fast) | Any | Bounding box coordinates recorded in `config.py` |
| Download & cache road graph | `osmnx.graph_from_bbox()` → save as `.graphml` | Any | File exists in `data/road_graph.graphml`, loads offline |

**Milestone 0 exit criteria:** Environment runs; road graph cached locally; team can work offline from here on.

---

### Phase 1 — Data Layer (Day 1–3)
| Task | Detail | Done when |
|---|---|---|
| Bin schema | Implement `bins.csv` schema (Section 4.1 of TRD) | Sample file with ≥30 bins generated |
| Vehicle schema | Implement `vehicles.csv` (capacity, depot, availability) | Sample file with 3–5 vehicles generated |
| Fill-history simulator | Seeded generator producing varied accumulation-rate profiles (slow/medium/fast bins) | Historical time series visibly differ across bin "zones" when plotted |
| Snap bins to road graph | Use `osmnx.distance.nearest_nodes` to attach each bin/depot to nearest graph node | Every bin has a valid nearest-node ID with no errors |

**Milestone 1 exit criteria:** A reproducible (seeded) synthetic dataset exists, visually sanity-checked, and every location resolves to a real road-network node.

---

### Phase 2 — Prediction Module (Day 3–5)
| Task | Detail | Done when |
|---|---|---|
| Implement `predict_fill()` | Linear regression / trend extrapolation per bin over configurable horizon | Function returns predicted % and time-to-overflow for any bin+horizon |
| Validate against synthetic ground truth | Compare predicted vs. actual simulated future fill on held-out time steps | Prediction error is reasonable/explainable (document the number, doesn't need to be perfect) |
| Unit tests | Cover edge cases: bin with zero accumulation, bin already near 100% | Tests pass |

**Milestone 2 exit criteria:** Prediction module runs on the full bin dataset and produces sensible, presentable forecasts.

---

### Phase 3 — Dynamic Priority Engine (Day 4–6, overlaps Phase 2)
| Task | Detail | Done when |
|---|---|---|
| Implement `score_bin()` | Weighted formula combining current fill, predicted fill, accumulation rate, overflow risk, time since last collection (TRD Section 5.2) | Function returns a numeric score for any bin |
| Tiering logic | Map score → Critical/High/Medium/Low using configurable thresholds | Tier distribution across the dataset looks reasonable (not all one tier) |
| Config file | Move weights & thresholds into `config.py` | Changing a weight changes tier outcomes without code edits |

**Milestone 3 exit criteria:** Every bin in the dataset has a priority score and tier, adjustable via config.

---

### Phase 4 — Bin Selection (Day 6)
| Task | Detail | Done when |
|---|---|---|
| Implement `select_bins()` | Filter by tier + available fleet capacity | Returns a bin subset ≤ total fleet capacity |
| Manual override hook | Support include/exclude list (for Priority Queue screen) | Overriding a bin changes the selected set correctly |

**Milestone 4 exit criteria:** A clean, capacity-aware candidate list is produced automatically for any given cycle.

---

### Phase 5 — Road Network & Distance Matrix (Day 6–8)
| Task | Detail | Done when |
|---|---|---|
| Implement `build_distance_matrix()` | Shortest-path distance & time between depot and all selected bins using NetworkX on the cached OSM graph | Returns a correct, symmetric N×N matrix |
| Sanity check | Compare a few matrix entries against manually estimated real distances | Values are realistic (not wildly off) |
| Performance check | Matrix build time acceptable for demo-scale bin counts (20–50 bins) | Completes in a few seconds |

**Milestone 5 exit criteria:** Real road-based distances are available as solver input, computed fast enough for a live demo.

---

### Phase 6 — CVRP Solver Integration (Day 8–10)
| Task | Detail | Done when |
|---|---|---|
| Implement `solve_routes()` | Wire distance matrix + vehicle capacities + bin demand into OR-Tools `RoutingModel` | Solver returns a feasible solution for the demo dataset |
| Capacity constraint | `AddDimensionWithVehicleCapacity` respects each vehicle's limit | No vehicle route exceeds its capacity in output |
| Multi-vehicle test | Run with 3–5 vehicles and confirm sensible route split | Each vehicle gets a non-trivial, distinct route |
| Extract results | Per-vehicle ordered stop list, distance, load | Output structure matches Section 6 interface contract (TRD) |

**Milestone 6 exit criteria:** The solver reliably produces valid, capacity-respecting multi-vehicle routes on real road distances.

---

### Phase 7 — Baseline Comparator (Day 10–11)
| Task | Detail | Done when |
|---|---|---|
| Implement `baseline_routes()` | Naive threshold-based bin selection + straight-line-distance routing (nearest-neighbor heuristic is sufficient) | Produces a comparable route set for the same scenario |
| Compute comparison metrics | Total distance, trip count, bins at risk of overflow, vehicle utilization | Metrics computed for both baseline and BinSense on identical input |

**Milestone 7 exit criteria:** A credible, same-dataset before/after comparison exists to show judges.

---

### Phase 8 — Dashboard / UI (Day 11–15)
| Task | Detail | Done when |
|---|---|---|
| Screen 1 — Dashboard | Wire summary counts + fleet status | Loads and displays live data from the pipeline |
| Screen 2 — Priority Map | Folium/Plotly map with color-coded bins | Bin colors correctly reflect tier |
| Screen 3 — Priority Queue | Ranked list + include/exclude controls | Overrides affect what reaches the solver |
| Screen 4 — Route Optimizer | Map with per-vehicle routes + before/after toggle | Matches solver output; toggle switches cleanly between baseline and optimized |
| Screen 5 — Dispatch & Tracking | Status list + "Complete Cycle" action | Marking complete writes back to bin history, refreshes Dashboard |
| Config controls in UI | Expose key weights/thresholds/fleet size as adjustable inputs | Judges can tweak a parameter live and see results update |

**Milestone 8 exit criteria:** Full 5-screen app runs end-to-end locally, reflecting the App-Flow-Implementation document.

---

### Phase 9 — Testing & Hardening (Day 15–17)
| Task | Detail | Done when |
|---|---|---|
| Unit tests | Priority scoring, distance matrix, selection logic | All pass |
| Integration test | Full cycle: simulate → predict → prioritize → select → route → compare | Runs without manual intervention |
| Offline test | Disconnect internet, confirm app still runs using cached road graph | No network errors during a full run |
| Edge cases | Zero bins selected; fleet capacity exceeded; bin outside network coverage | App handles gracefully per Section 6 of App-Flow-Implementation.md |

**Milestone 9 exit criteria:** App is stable, offline-safe, and handles edge cases without crashing — ready for judged demo conditions.

---

### Phase 10 — Demo Packaging (Day 17–19)
| Task | Detail | Done when |
|---|---|---|
| Curate a strong demo scenario | Seed data that clearly shows Critical/High/Medium/Low variety and a compelling before/after gap | Rehearsed run tells a clear story in under 3 minutes |
| Prepare fallback | Record a video/screen-capture of a full run in case of live demo issues | Video ready as backup |
| Slide/pitch alignment | Sync app walkthrough with PRD Section 11 (Competition Demo Plan) | Pitch narrative matches what's on screen |
| Final README | Setup instructions, architecture summary, how to run | A new team member/judge could run it from README alone |

**Milestone 10 exit criteria:** Prototype + narrative + fallback video are all ready, at least 1–2 days before the competition date.

---

## 4. Suggested Timeline (Overview)

| Days | Phase |
|---|---|
| 0–1 | Setup |
| 1–3 | Data layer |
| 3–6 | Prediction + Priority engine (parallel) |
| 6 | Bin selection |
| 6–8 | Road network / distance matrix |
| 8–10 | CVRP solver |
| 10–11 | Baseline comparator |
| 11–15 | Dashboard / UI |
| 15–17 | Testing & hardening |
| 17–19 | Demo packaging |

*(Compress or stretch proportionally based on actual days available before the competition deadline — flag this against the "time budget" open question in the PRD.)*

---

## 5. Dependencies Between Phases

```
Setup ─► Data Layer ─┬─► Prediction ─────┐
                      │                    ├─► Priority Engine ─► Bin Selection ─┐
                      │                    │                                     │
                      └─► Road Network ────┘                                     ├─► CVRP Solver ─► Baseline Comparator ─► Dashboard ─► Testing ─► Demo Packaging
```

Prediction and Road Network work can run in parallel since neither depends on the other. Priority Engine needs Prediction's output. CVRP Solver needs both Bin Selection and the Road Network's distance matrix.

---

## 6. Roles (Adapt to Team Size)

| Role | Responsible for |
|---|---|
| Data/Backend lead | Phases 1–4 (data, prediction, priority, selection) |
| Optimization lead | Phases 5–7 (road network, CVRP, baseline) |
| Frontend/Dashboard lead | Phase 8 (Streamlit UI) |
| QA / Demo lead | Phases 9–10 (testing, packaging, pitch alignment) |

*(One person can hold multiple roles on a small team — the phase breakdown still gives a clear checklist.)*

---

## 7. Risk Log & Contingencies

| Risk | Contingency |
|---|---|
| OSM download slow/unavailable on demo day | Road graph cached locally in Phase 0 — no live dependency needed afterward |
| OR-Tools solver too slow at larger bin counts | Cap demo scenario to 20–50 bins (documented as a prototype-scale choice, not a system limitation) |
| Prediction looks "too simple" to judges | Present it explicitly as a deliberate prototype-scope choice; name LSTM/Prophet as documented future work (PRD Section 12) |
| UI bugs during live demo | Rehearsed fallback video ready (Phase 10) |
| Running out of time before deadline | Phases 1–7 (backend pipeline) are the core novelty — prioritize these over UI polish if time is short; a notebook-based walkthrough can substitute for the full dashboard if needed |

---

## 8. Definition of Done (Whole Prototype)

- [ ] End-to-end pipeline runs on simulated data without manual steps
- [ ] Routes are computed on real road-network distances, not straight-line
- [ ] Multi-vehicle capacity constraints are respected in every solved route
- [ ] Priority tiers visibly reflect predicted (not just current) fill level
- [ ] A clear, same-dataset before/after comparison exists
- [ ] App works fully offline (cached road graph)
- [ ] 5-screen flow matches App-Flow-Implementation.md
- [ ] README allows a stranger to set up and run the prototype
- [ ] Fallback demo video recorded

---

*End of Implementation Plan.*
