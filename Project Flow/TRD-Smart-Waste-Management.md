# Technical Requirements Document (TRD)
## Predictive and Dynamic Smart Waste Management with Road-Network-Based Multi-Vehicle Optimization

**Document Version:** 1.0
**Companion to:** PRD-Smart-Waste-Management.md
**Constraint:** All tools, libraries, data sources, and APIs used must be **free / open-source / free-tier**, suitable for a student competition build with no budget.

---

## 1. Purpose

This document specifies the concrete technical architecture, tools, data sources, algorithms, and interfaces required to build the prototype described in the PRD — using only free and open-source resources, so the system can be built and demoed without licensing cost.

---

## 2. Technology Stack (All Free / Open-Source)

| Layer | Component | Tool / Library | Why (Free & Justification) |
|---|---|---|---|
| Language/runtime | Core logic | **Python 3.10+** | Free, huge ecosystem for optimization/ML |
| Data handling | Bin records, time series | **pandas, NumPy** | Free, standard for tabular/time-series data |
| Fill prediction | Forecasting model | **scikit-learn** (Linear/Polynomial Regression, or `statsmodels` for ARIMA/Holt-Winters) | Free, lightweight, explainable — no GPU/paid model needed |
| Road network data | Real road graph | **OpenStreetMap (OSM)** data via **OSMnx** | OSM is a free, open, community-maintained map dataset; OSMnx is a free Python wrapper to download & analyze it |
| Graph computation | Shortest paths, distance/time matrix | **NetworkX** (built into OSMnx) | Free graph library, computes shortest paths on the OSM road graph |
| Alternative routing (optional) | Turn-by-turn distance/time | **OSRM (Open Source Routing Machine)** — self-hosted or public demo server | Free, open-source routing engine; avoids paid APIs like Google Maps Distance Matrix |
| Route optimization | CVRP solver | **Google OR-Tools** (`ortools` Python package) | Free, open-source (Apache 2.0), industry-grade VRP/CVRP solver |
| Visualization (maps) | Bin/route map rendering | **Folium** (Leaflet.js wrapper) or **Plotly** (scatter_mapbox, free tier, no token needed with open-street-map style) | Free, no API key required for basic OSM tile rendering |
| Visualization (dashboard) | Interactive demo UI | **Streamlit** | Free, open-source, fastest way to build an interactive Python dashboard for a live demo |
| Data storage | Bin & history data | **CSV / JSON / SQLite** | Free, no server/DB licensing needed |
| Version control | Code hosting | **GitHub** (free public/private repos) | Standard, free for competition submission |
| Environment | Dependency management | **venv / conda (Miniconda)**, `requirements.txt` | Free environment isolation |
| Notebook (optional) | Algorithm exploration | **Jupyter Notebook / Google Colab** | Free; Colab also gives free compute if needed |

> **No paid APIs required.** Google Maps Distance Matrix API, HERE, and similar paid services are intentionally excluded in favor of OSM + OSMnx/OSRM, which give free, real road-network data and routing.

---

## 3. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA LAYER                                │
│  Simulated / sample bin dataset (CSV)                            │
│  OSM road network for chosen demo city/area (via OSMnx download) │
└─────────────────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                     PREDICTION LAYER                              │
│  scikit-learn / statsmodels model per bin                        │
│  Input: historical fill % time series                            │
│  Output: predicted fill % at horizon T, time-to-overflow          │
└─────────────────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                  DYNAMIC PRIORITY ENGINE                          │
│  Weighted scoring function (pure Python)                          │
│  Inputs: current fill, predicted fill, accumulation rate,         │
│          overflow risk, time since last collection                │
│  Output: priority score + tier (Critical/High/Medium/Low)         │
└─────────────────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                     BIN SELECTION LAYER                           │
│  Filters bins by tier + fleet capacity/availability               │
└─────────────────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                ROAD NETWORK / DISTANCE MATRIX LAYER                │
│  OSMnx: download road graph for area (free OSM data)              │
│  NetworkX: shortest path distance & travel time between            │
│            depot ↔ selected bins ↔ each other                     │
│  Output: N x N distance matrix, N x N time matrix                 │
└─────────────────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                  ROUTE OPTIMIZATION LAYER (CVRP)                   │
│  Google OR-Tools Routing solver                                   │
│  Constraints: vehicle capacity, depot, num vehicles                │
│  Output: per-vehicle bin sequence, distance, load                  │
└─────────────────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                   VISUALIZATION / DEMO LAYER                       │
│  Streamlit app + Folium/Plotly map                                 │
│  Shows: bins (color by urgency), depot, optimized routes,          │
│         baseline-vs-proposed comparison metrics                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Data Design

### 4.1 Bin Dataset Schema (CSV/JSON)

| Field | Type | Description |
|---|---|---|
| `bin_id` | string | Unique identifier |
| `latitude`, `longitude` | float | Location (within chosen demo area's OSM bounds) |
| `capacity_liters` | float | Max bin capacity |
| `current_fill_pct` | float | Current fill level (0–100) |
| `fill_history` | list[(timestamp, fill_pct)] | Historical readings (simulated) |
| `avg_accumulation_rate` | float | %/hour, derived from history |
| `last_collected_at` | datetime | Last collection timestamp |

### 4.2 Vehicle Dataset Schema

| Field | Type | Description |
|---|---|---|
| `vehicle_id` | string | Unique identifier |
| `capacity_liters` | float | Max carrying capacity |
| `depot_id` | string | Starting/ending depot |
| `available` | bool | Availability flag |

### 4.3 Simulation Approach (Free, No Real Sensors Needed)
- Use OSMnx to pull real road network + node coordinates for a chosen area (e.g., a city or a defined bounding box) — this data is free and requires only an internet connection at build time (can be cached locally afterward for offline demo).
- Randomly (seeded) place bins at real road-network node locations within the area.
- Generate synthetic fill-history time series per bin using different randomized accumulation-rate profiles (e.g., slow/medium/fast filling zones) to create realistic variety for the prediction module to learn from.

---

## 5. Algorithm Specifications

### 5.1 Fill-Level Prediction
- **Method (v1, prototype-appropriate):** Linear regression or exponential smoothing per bin on its historical fill series → extrapolate to horizon `T` hours ahead.
- **Formula (simple linear case):**
  `predicted_fill(t+T) = current_fill + accumulation_rate × T`
  where `accumulation_rate` is estimated via linear regression over recent history.
- **Overflow risk:** `time_to_overflow = (100 − current_fill) / accumulation_rate`
- **Library:** `scikit-learn.linear_model.LinearRegression`, or `statsmodels` for a slightly more robust trend/seasonality fit if time permits.

### 5.2 Dynamic Priority Engine
- **Composite score (example weighted formula, weights configurable):**
  ```
  priority_score = w1 * current_fill_pct
                 + w2 * predicted_fill_pct
                 + w3 * accumulation_rate_normalized
                 + w4 * overflow_risk_normalized
                 + w5 * time_since_last_collection_normalized
  ```
- **Tiering (example thresholds, tunable):**
  - Critical: score ≥ 85
  - High: 65 ≤ score < 85
  - Medium: 40 ≤ score < 65
  - Low: score < 40

### 5.3 Road-Network Distance/Time Matrix
- Use `osmnx.graph_from_place()` or `graph_from_bbox()` to fetch the drivable road network (free OSM data) for the demo area.
- Snap each bin & depot lat/long to the nearest graph node (`osmnx.distance.nearest_nodes`).
- Compute shortest-path distance/time between all relevant node pairs using `networkx.shortest_path_length` (weighted by `length` or estimated travel time using default/free speed assumptions per road type).
- Assemble into an `N x N` matrix for input to OR-Tools.

### 5.4 CVRP Solver (Google OR-Tools)
- Use `ortools.constraint_solver.routing_enums_pb2` and `pywrapcp.RoutingModel`.
- Inputs: distance matrix, vehicle capacities, bin demands (e.g., estimated waste volume), number of vehicles, depot index.
- Constraints: `AddDimensionWithVehicleCapacity` for capacity; depot as start/end node for every vehicle.
- Objective: minimize total route distance (or time).
- Output: `SolutionPrinter`-style extraction of each vehicle's ordered node sequence, load, and distance.

---

## 6. Interfaces / Module Contracts

| Module | Input | Output |
|---|---|---|
| Prediction | Bin fill history (per bin) | Predicted fill %, time-to-overflow |
| Priority Engine | Current + predicted bin data | Priority score, urgency tier |
| Bin Selection | Bins + tiers + fleet capacity | List of bins to service this cycle |
| Distance Matrix | Bin/depot coordinates + OSM graph | N×N distance & time matrices |
| CVRP Solver | Distance matrix, vehicle list, bin demands | Per-vehicle route (ordered bin list), total distance |
| Visualization | Routes + bin urgency data | Rendered map + metrics dashboard |

---

## 7. Deployment / Demo Environment (Free)

| Need | Free Option |
|---|---|
| Run the prototype live | Local machine (laptop) — no server cost |
| Share/host for judges pre-demo | **Streamlit Community Cloud** (free tier) or **GitHub Pages** (for a static exported report/map) |
| Code hosting & submission | **GitHub** free repository |
| Offline safety net | Pre-download and cache the OSM road graph (`osmnx` supports saving graph to GraphML) so the demo does not depend on live internet during the actual presentation |

---

## 8. Non-Functional / Engineering Requirements

- **Reproducibility:** All randomized simulation must use a fixed random seed.
- **Offline resilience:** OSM graph and any external data must be pre-fetched and cached locally before the competition demo (avoid live-internet dependency during judging).
- **Modularity:** Each layer (prediction, priority, routing, visualization) implemented as a separate Python module/function with clear input/output contracts (Section 6), so components can be swapped or improved independently.
- **Config-driven parameters:** Priority weights, tier thresholds, prediction horizon, number of vehicles, and vehicle capacity should be adjustable via a config file or dashboard controls — not hardcoded — to support live "what-if" demonstration to judges.

---

## 9. Testing Plan (Prototype Scope)

| Test | Purpose |
|---|---|
| Unit test: priority scoring function | Verify correct tier assignment for known input combinations |
| Unit test: distance matrix generation | Verify matrix symmetry/values against a small known OSM area |
| Integration test: end-to-end run | Confirm pipeline runs from simulated data → final routes without error |
| Comparison test | Baseline (threshold + straight-line) vs. proposed (predictive + road-network CVRP) on the same simulated dataset, to produce the metrics used in the demo |

---

## 10. Free-Resource Summary (Quick Reference)

- **Road network data:** OpenStreetMap (free, open data) via OSMnx
- **Routing/graph computation:** NetworkX (free)
- **Optimization solver:** Google OR-Tools (free, open-source)
- **ML/prediction:** scikit-learn / statsmodels (free)
- **Visualization:** Folium / Plotly + Streamlit (free)
- **Hosting for demo:** Streamlit Community Cloud / GitHub Pages (free tier)
- **Compute (if needed):** Google Colab (free tier)
- **Version control:** GitHub (free)

No paid APIs, licenses, or cloud compute are required to build or demo this prototype.

---

*End of TRD.*
