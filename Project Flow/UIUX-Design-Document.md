# UI/UX Design Document
## BinSense — Predictive and Dynamic Smart Waste Management

**Document Version:** 1.0
**Companion to:** PRD, TRD, App-Flow-Implementation.md, Implementation-Plan.md
**Purpose:** Define the visual language, interaction patterns, and screen-level UI specification so the prototype looks and behaves consistently, and so anyone on the team (or a judge reading this doc) understands the design intent.

---

## 1. Design Principles

1. **Urgency should be readable at a glance.** Color and position always communicate which bins matter right now — no digging through menus to find what's critical.
2. **Real data, not decoration.** Every visual element (map dot, bar, badge) represents an actual value from the pipeline. Nothing is illustrative filler.
3. **One bold signal, everything else quiet.** The urgency color system (red/amber/green) is the one loud element; layout, type, and surfaces stay calm and structured so the signal doesn't compete with itself.
4. **Operators can always see why.** A tap on any bin or route should reveal the underlying numbers (fill %, predicted %, distance) — the system should never feel like a black box.
5. **Built for a live demo.** Screens must read clearly on a shared projector/screen — sufficient contrast, no tiny text, no reliance on hover-only interactions.

---

## 2. Visual Identity

### 2.1 Concept
BinSense is framed as a **night-operations control center for a cleaner city** — closer to a fleet/ops dashboard than a generic "green eco app." The dark surface reads as a live monitoring tool; the amber and leaf-green accents carry the actual meaning (urgency vs. resolved/efficient), rather than being decorative brand colors.

### 2.2 Color System

| Token | Hex (dark mode) | Usage |
|---|---|---|
| `--bg` | `#0E1912` | App background |
| `--surface` | `#182620` | Cards, panels |
| `--surface-2` | `#1E2F27` | Nested elements inside cards (mini charts, list rows) |
| `--line` | `rgba(241,239,230,0.10)` | Hairline dividers |
| `--text` | `#F1EFE6` | Primary text |
| `--text-dim` | `#A9B3AC` | Secondary text |
| `--text-faint` | `#6E7A72` | Tertiary / metadata text |
| `--leaf` (green) | `#6FAE7C` | Positive / low urgency / optimized route / efficiency gains |
| `--amber` | `#E8A13D` | High urgency / warnings / attention needed |
| `--danger` | `#D9714E` | Critical urgency / overflow risk |

A light-mode variant exists using the same semantic roles on a warm off-white base, for environments where a dark UI isn't preferred (see the published landing page for the exact light-mode token set).

**Rule:** Color always maps to meaning — green/amber/red are reserved exclusively for urgency and performance signals. They are never used purely for decoration or unrelated UI chrome, so operators can trust the color coding completely.

### 2.3 Typography

| Role | Typeface | Usage |
|---|---|---|
| Display / headings | **Space Grotesk** (weights 500–700) | Screen titles, key numbers, section headers |
| Body / UI text | **Inter** (weights 400–600) | Body copy, labels, list content, buttons |

- Numbers that matter (priority scores, distances, %) are set in the display face at a larger size than surrounding text — the number should be the first thing read.
- Avoid all-caps labels and unnecessary eyebrow text; use sentence case throughout (see Section 6, Writing Guidelines).

### 2.4 Iconography & Markers
- Bin urgency is always shown as a **filled dot**, sized slightly larger for higher urgency, with a subtle pulse animation reserved only for the Critical tier (so pulsing motion itself signals "needs attention now").
- Depot is shown as a small filled square, visually distinct from the circular bin markers.
- Routes are drawn as dashed, animated lines (motion indicates "in progress / live"), colored per vehicle.

### 2.5 Spacing & Shape
- Corner radius: 8–16px depending on component size (buttons ~9px, cards ~12–16px) — consistent within each size tier, not a single flat radius everywhere.
- Base spacing unit: 4px grid (8/12/16/24/32px steps).
- Cards use a 1px hairline border rather than heavy shadows, to keep the dark UI feeling flat and calm rather than glossy.

---

## 3. Layout & Navigation Model

- **Primary navigation:** a persistent top bar with four sections — *Dashboard, Priority Map, Priority Queue, Route Optimizer* — plus implicit access to *Dispatch* from the Route Optimizer's "Dispatch" action (dispatch is a resulting state, not a separately-browsed tab).
- **Flow direction:** left-to-right, top-to-bottom through the operational sequence (see App-Flow-Implementation.md Section 4) — the UI should always make it obvious what the *next* action is (e.g., a bin queue always ends in a visible "Build Routes" call-to-action).
- **Single-column focus on mobile / narrow widths:** map and list stack vertically; the priority queue becomes the default view (map remains one tap away) since operators are more likely to scan a list on a small screen.

---

## 4. Screen-Level UI Specification

### Screen 1 — Dashboard
| Element | Behavior |
|---|---|
| Urgency count tiles (Critical/High/Medium/Low) | Static summary cards; tapping a tile jumps to Priority Queue pre-filtered to that tier |
| Fleet status | Shows vehicles available / total, updates live from vehicle dataset |
| Fleet capacity bar | Horizontal bar, fills proportionally; color shifts from leaf → amber as utilization approaches 100% |
| Overflow risk indicator | Single labeled bar (Low/Moderate/High) summarizing city-wide predicted risk |

**Empty state:** If no bins are above Medium tier, the tiles show "0" in leaf green with the copy *"No bins need urgent attention right now."* — a calm, positive empty state, not a warning.

### Screen 2 — Priority Map
| Element | Behavior |
|---|---|
| Map canvas | Pan/zoom enabled; bins rendered as colored dots per tier |
| Bin tap | Opens a small popover: bin ID, address, current fill %, predicted fill %, time-to-overflow, last collected |
| Legend | Fixed bottom-left chip showing tier-color key at all times |
| Filter toggle | Show/hide tiers (e.g., hide Low to reduce clutter on a dense map) |

**Interaction detail:** Only Critical-tier bins pulse; High/Medium/Low are static, so motion is reserved as a true attention signal rather than applied uniformly.

### Screen 3 — Priority Queue
| Element | Behavior |
|---|---|
| Ranked list | Sorted by priority score descending; each row shows tier badge, bin ID, location name |
| Row checkbox | Include/exclude from this cycle's run; excluded rows visually dim but remain in list (not deleted) |
| Running counter | "X bins selected · Y% of fleet capacity" updates live as rows are toggled |
| Primary action | "Build Routes" button, disabled until at least one bin is selected |

**Interaction detail:** Manual overrides are reversible within the same cycle — nothing is destructive until "Build Routes" is confirmed.

### Screen 4 — Route Optimizer
| Element | Behavior |
|---|---|
| Map canvas | One line color per vehicle; depot marked; stops numbered along each route |
| Per-vehicle summary cards | Stops count, distance, load vs. capacity — one card per active vehicle |
| Before/After toggle | Switches the same map/metrics between "Baseline" (straight-line/threshold) and "BinSense" (road-network/predictive) without navigating away |
| Re-solve control | Adjusting vehicle count/capacity re-runs the solver and updates the map in place |

**Interaction detail:** The Before/After toggle is the single most important interaction on this screen for a competition demo — it should be prominent, not buried in a menu.

### Screen 5 — Fleet Dispatch & Tracking
| Element | Behavior |
|---|---|
| Vehicle status list | Status dot (en route / loading / standby) + current stop progress |
| Distance-saved metric | Prominent, in leaf green, comparing this cycle to the baseline |
| Complete Cycle action | Confirms the run; writes outcomes back to bin history; returns to Dashboard with refreshed data |

**Empty/edge state:** If a vehicle has no assigned stops this cycle, it's shown as "Standby" in muted gray, not hidden — operators should always see the full fleet, active or not.

---

## 5. Interaction & Motion Guidelines

- **One motion per meaningful state**, not decoration on every element: the Critical-tier pulse, the animated dashed route line, and the before/after cross-fade are the only three recurring motion patterns in the app.
- **No hover-only reveals** — every interactive element must also work via tap, since the target usage is a touch dashboard or shared-screen demo, not a mouse-only desktop.
- **Loading states** show a skeleton of the target screen's layout (not a generic spinner), so the operator already knows what's about to appear.
- **Reduced motion:** all animations respect `prefers-reduced-motion` — pulsing and dashed-line animation are disabled in favor of a static color/border indicator when this is set.

---

## 6. Writing Guidelines (Content Voice)

- **Active voice, plain verbs.** "Build Routes," not "Submit Route Request."
- **Consistent vocabulary end-to-end.** If a button says "Dispatch," the resulting screen says "Dispatched" — never rename the same action mid-flow.
- **No apologetic or vague error copy.** E.g., *"Selected bins exceed fleet capacity — 3 lowest-priority bins moved to next cycle,"* not *"Something went wrong."*
- **Numbers lead, labels follow.** e.g. large "31%" with a small "distance saved vs. baseline" beneath it, not the reverse.
- **Empty states are informative, not empty.** Always state the current condition in plain language (see Dashboard empty state above) rather than leaving a blank panel.

---

## 7. Accessibility & Responsiveness

- Minimum text contrast ratio of 4.5:1 against both dark and light backgrounds (verified for `--text`, `--text-dim`, and all tier colors against `--bg`/`--surface`).
- All interactive elements have a visible keyboard focus outline.
- Urgency is never communicated by color alone — each tier also has a text label ("Critical," "High," etc.) and, on the map, a distinct dot size, so the app remains usable for colorblind users.
- Layout is responsive: map/list stack vertically below ~800px width; touch targets are a minimum 40×40px for tablet/kiosk use during the live demo.

---

## 8. Component Inventory (Reusable Across Screens)

| Component | Used on |
|---|---|
| Urgency tier badge (Critical/High/Medium/Low) | Priority Map popovers, Priority Queue rows |
| Metric tile (big number + small label) | Dashboard, Impact/metrics section |
| Map canvas with legend chip | Priority Map, Route Optimizer |
| Ranked list row with checkbox | Priority Queue |
| Vehicle status row | Fleet Dispatch |
| Before/After toggle | Route Optimizer |
| Primary/ghost buttons | All screens |

Keeping this as a shared component set (rather than one-off styling per screen) is what keeps the five screens feeling like one coherent product rather than five separate mockups.

---

## 9. Relationship to the Published Landing Page

The landing page and app-flow mockups already published for this project (BinSense) implement this design system directly — same color tokens, type pairing, and urgency-marker conventions — and can be treated as the visual reference implementation when building the actual Streamlit/dashboard prototype described in the Implementation Plan.

---

*End of UI/UX Design Document.*
