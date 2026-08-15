# Technical Requirements Document (TRD)
## Project: TrafficBalance

---

## 1. Architecture Overview

```
┌─────────────────┐      OSM export       ┌──────────────────────┐
│  osmWebWizard /  │ ───────────────────▶ │   SUMO Network Files  │
│  OpenStreetMap    │                      │  (.net.xml/.rou.xml)  │
└─────────────────┘                      └──────────┬────────────┘
                                                      │
                                                      ▼
                                          ┌───────────────────────┐
                                          │   SUMO Engine (sumo)   │
                                          │  controlled via TraCI  │
                                          └──────────┬────────────┘
                                                      │ Python (TraCI API)
                                                      ▼
                                   ┌──────────────────────────────────┐
                                   │  Control Layer (Python)           │
                                   │  - Baseline: fixed-time signals   │
                                   │  - Optimized: adaptive signals +  │
                                   │    congestion-weighted rerouting  │
                                   │  - Metrics engine (Gini index,    │
                                   │    avg wait/travel time)          │
                                   └──────────────┬────────────────────┘
                                                  │ in-process call
                                                  ▼
                                   ┌──────────────────────────────────┐
                                   │  FastAPI Backend                  │
                                   │  - REST: /scenario, /run, /reset  │
                                   │  - WebSocket: /ws/simulation      │
                                   │    (streams per-tick metrics)     │
                                   └──────────────┬────────────────────┘
                                                  │ WebSocket (JSON)
                                                  ▼
                                   ┌──────────────────────────────────┐
                                   │  React Frontend                   │
                                   │  - Live heatmap/map (Leaflet)     │
                                   │  - Metrics charts (Recharts)      │
                                   │  - Before/After toggle            │
                                   └────────────────────────────────────┘
```

## 2. Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Simulation | SUMO (Eclipse SUMO) + TraCI | Free, open-source, real OSM import, Python-controllable step-by-step |
| Control logic | Python 3.11 | Team's existing strength; TraCI is Python-native |
| Backend | FastAPI + WebSockets (uvicorn) | Team's existing strength; async fits real-time streaming |
| Frontend | React (Vite) + Leaflet/deck.gl + Recharts | Team's existing strength; Leaflet for map, Recharts for metric charts |
| Data interchange | JSON over WebSocket, REST for control actions | Simplicity, no need for gRPC/heavier protocols in 24h |
| Hosting (backend) | Render / Railway free tier (see deployment doc) | SUMO needs a real process — needs a container, not serverless |
| Hosting (frontend) | Vercel / Netlify free tier | Static React build, fast free deploy |
| Version control | GitHub, one repo, feature branches per module | 5-person parallel work needs branch isolation |

## 3. Key Technical Components

### 3.1 Simulation Layer
- `network.net.xml` — road network exported from OSM for the chosen Nagpur corridor.
- `routes.rou.xml` — two demand profiles (9–12, 4–7), generated via SUMO's `randomTrips.py` or manually authored flows tuned to look like realistic peak-hour asymmetric demand (this asymmetry is what creates the "unevenness" to solve).
- `baseline.sumocfg` / `optimized.sumocfg` — two run configs.

### 3.2 Control Layer (the differentiator — build and test this in isolation first)
- **Adaptive signal logic**: each junction's green-phase duration is a function of the queue length on its approach lanes (read via `traci.lane.getLastStepHaltingNumber`), bounded by min/max green time.
- **Congestion-weighted rerouting**: periodically (e.g. every 60 sim-seconds) recompute a "cost" per edge = base travel time × congestion multiplier (based on occupancy). Reroute a fraction of vehicles via `traci.vehicle.rerouteTraveltime` using updated edge weights, so vehicles are steered toward less-loaded parallel roads.
- **Load-Imbalance Index**: compute Gini coefficient (or coefficient of variation) of per-road vehicle density at each timestep. This is the number the dashboard leads with.

### 3.3 Backend API (contract — lock this by hour 1)
- `POST /scenario` — select corridor + time window (9–12 or 4–7) + mode (baseline/optimized)
- `POST /run` — starts a simulation run, returns a `run_id`
- `POST /reset` — resets/stops current run
- `WS /ws/simulation/{run_id}` — streams JSON ticks: `{tick, per_road: [{road_id, density, avg_speed, queue}], imbalance_index, avg_travel_time}`
- `GET /results/{run_id}` — final summary after a run completes (for the report/export stretch feature)

### 3.4 Frontend
- Map view with per-road color intensity = live congestion.
- Metrics panel: Load-Imbalance Index line chart (baseline vs optimized overlay), avg travel/wait time.
- Scenario controls: pick time window, toggle baseline/optimized, start/reset.

## 4. Non-Functional Requirements
- **Reliability**: the demo path must run 5+ times without crashing before presenting.
- **Latency**: WebSocket updates at a rate the UI can render smoothly (batch ticks if needed — don't push every single SUMO step if it's sub-second; 1 update/sec is enough for human perception).
- **Reproducibility**: fixed random seeds for demo runs so results are consistent every rehearsal.
- **Portability**: everything must run from a fresh clone with one setup script — critical when 5 people are joining code late in the day.

## 5. Environment & Dependencies
- Python 3.11, `sumo` + `traci` (`pip install sumolib traci`, plus the SUMO binary itself — see integration doc for install steps on Windows since the team develops on Windows).
- Node 18+, `npm create vite@latest` for frontend scaffold.
- `requirements.txt` and `package.json` committed early so all 5 machines match.
