# App Flow
## Project: TrafficBalance

---

## 1. High-Level System Flow

```
User opens dashboard
       │
       ▼
Scenario Selector screen
  - pick time window (9-12 / 4-7)
  - pick mode (Baseline / Optimized / Both)
       │
       ▼
POST /scenario  ──▶  backend creates RunState, returns run_id
       │
       ▼
POST /run/{run_id}/start
       │
       ▼
Backend spins up SUMO via TraCI with the selected .sumocfg
       │
       ▼
Frontend opens WS /ws/simulation/{run_id}
       │
       ▼
Loop: backend steps SUMO ──▶ reads TraCI state ──▶ computes
      per-road metrics + imbalance index ──▶ pushes SimulationTick
       │                                              │
       │◀─────────────── (repeats until sim ends) ────┘
       ▼
Frontend live-updates map heatmap + metric charts each tick
       │
       ▼
Simulation ends ──▶ backend sends {"event":"completed", summary}
       │
       ▼
Frontend renders Before/After Summary screen
       │
       ▼
(optional) User exports report (S4 stretch)
```

## 2. Detailed User Journey (Judge Walk-through)

1. **Landing**: judge/presenter sees the Scenario Selector — clean, minimal, one clear CTA.
2. **Selection**: presenter selects "9 AM–12 Noon" and mode "Both" (baseline + optimized run back-to-back or side-by-side).
3. **Run starts**: loading state briefly shown while SUMO initializes (should be under a few seconds — pre-warm the SUMO process if init is slow).
4. **Live view**: map lights up with color-coded roads; Load-Imbalance Index ticks in the top banner; chart begins plotting.
5. **Comparison**: once both runs complete (or run concurrently if resources allow), the dual-line chart shows baseline vs. optimized imbalance over time — the optimized line should visibly settle lower.
6. **Summary**: Before/After screen shows the final percentage improvement — this is the slide/screen the pitch should linger on.
7. **(Stretch) Export**: presenter clicks export, a report is generated for the "Planning Authority" persona.

## 3. Internal Data Flow (per simulation tick)

```
TraCI step
  → traci.edge.getLastStepVehicleNumber() per monitored road
  → traci.edge.getLastStepMeanSpeed() per monitored road
  → traci.lane.getLastStepHaltingNumber() for queue length
  → control layer decides: adjust signal phase? reroute vehicles?
  → metrics engine: compute gini_coefficient(densities), avg travel/wait time
  → package into SimulationTick
  → push over WebSocket
  → frontend updates MapHeatmap, MetricHeadline, MetricChart components
```

## 4. Failure/Edge-Case Flows
- **WebSocket disconnects mid-run**: `ConnectionStatus` component shows "reconnecting," frontend attempts reconnect using the same `run_id`; backend keeps `RunState` alive for a grace period so reconnection resumes the same view.
- **SUMO subprocess crash**: backend catches the TraCI exception, sends an `error` event, frontend shows a clear message and offers "restart run" rather than freezing on the last frame.
- **Judge wants to see it twice**: `/run/{run_id}/reset` then `/run/{run_id}/start` again with a fixed seed reproduces the same run deterministically — important so the story doesn't change between rehearsal and live demo.

## 5. Demo-Day Flow (operational, not app UX)
1. Presenter has the app already loaded and a completed "Both" run cached in the browser as a fallback screenshot/video.
2. Live run attempted first; if WebSocket/SUMO has any hiccup, presenter switches to the fallback video without breaking narrative flow — rehearse this exact switch beforehand.
3. Pitch closes on the Before/After Summary screen, not the live map — the summary is the takeaway image judges remember.
