# UI/UX Brief
## Project: TrafficBalance Dashboard

---

## 1. Design Principle
The dashboard's entire job is to make one thing obvious to a judge within 5 seconds: **traffic load became more evenly distributed.** Every screen should visually subordinate everything else to that story. Avoid generic "admin dashboard" clutter — this should read like a control-room screen, not a CRUD app.

## 2. Visual Direction
- **Tone**: technical, confident, "civic control room" — dark-leaning UI with high-contrast data (think traffic-ops center, not a consumer app).
- **Color system**:
  - Congestion heat scale: green (low load) → amber (moderate) → red (high load) — intuitive, judge needs zero explanation.
  - Accent color for "optimized" state vs. neutral gray for "baseline" state, so the before/after toggle is instantly readable.
- **Typography**: one clean sans-serif (e.g. Inter) — headline metric (Load-Imbalance Index) should be the single largest text element on screen.
- **Motion**: subtle — live-updating chart lines and a pulsing map are enough; avoid decorative animation that distracts from data.

## 3. Screens

### Screen 1 — Scenario Selector (entry point)
- Choose corridor (if S1 stretch is built, otherwise fixed).
- Choose time window: 9 AM–12 Noon / 4 PM–7 PM.
- Choose mode to run: Baseline / Optimized / Both (side-by-side).
- Primary CTA: "Run Simulation."

### Screen 2 — Live Simulation View (the core screen, where judging happens)
- **Left/main**: map of the corridor, roads color-coded by live congestion.
- **Top banner**: Load-Imbalance Index, large, updating live, with a small delta arrow vs. baseline.
- **Right panel**: secondary metrics — avg travel time, avg wait time, total vehicles processed.
- **Bottom strip**: time-series chart of the Imbalance Index over the simulation run, baseline line vs. optimized line overlaid once both have run.

### Screen 3 — Before/After Summary (shown at end of run, and reusable as the pitch's key slide)
- Two-column comparison: Baseline vs. Optimized.
- Big percentage callout: "Load imbalance reduced by X%."
- Small supporting stats underneath (travel time, wait time).
- Export button (PDF/CSV) if S4 stretch feature is built.

### Screen 4 (stretch, S3) — Citizen Notification Mock
- A simple phone-frame mockup showing a simulated "alternate route suggested" notification, to illustrate the redistribution mechanism isn't purely infrastructure-side.

## 4. Key UX Flows
1. **Judge walk-up flow**: Scenario Selector → pick 9–12 window → Run Both → watch Live Simulation View → land on Before/After Summary. This full flow should take under 3 minutes and never require explanation of controls.
2. **Presenter flow**: presenter should be able to jump straight to a pre-run "Both" comparison (cached results) as a fallback if live simulation is slow, without the UI looking broken.

## 5. Accessibility & Practical Notes
- Don't rely on color alone for congestion levels — add a numeric density label on hover/tap, since red/green color coding can be inaccessible to some judges.
- Design for a projector: high contrast, large fonts, test the screen on the actual demo display resolution if possible before presenting.
- Keep the layout responsive enough to demo from a laptop screen directly if a projector isn't available.

## 6. Component Inventory (for the Frontend Lead)
- `MapHeatmap` (Leaflet + colored polylines per road segment)
- `MetricHeadline` (large animated number + delta)
- `MetricChart` (Recharts line chart, dual-series)
- `ScenarioControls` (form: corridor/time-window/mode selectors + run button)
- `BeforeAfterSummary` (comparison cards + export button)
- `ConnectionStatus` (small indicator showing WebSocket live/disconnected — important so judges never see a silently-frozen UI)
