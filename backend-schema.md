# Backend Schema
## Project: TrafficBalance — FastAPI service

---

## 1. Pydantic Models

```python
from pydantic import BaseModel
from enum import Enum
from typing import List, Optional

class TimeWindow(str, Enum):
    MORNING = "9AM-12PM"
    EVENING = "4PM-7PM"

class Mode(str, Enum):
    BASELINE = "baseline"
    OPTIMIZED = "optimized"

class ScenarioRequest(BaseModel):
    corridor_id: str = "corridor_1"
    time_window: TimeWindow
    mode: Mode

class RunResponse(BaseModel):
    run_id: str
    corridor_id: str
    time_window: TimeWindow
    mode: Mode
    status: str  # "queued" | "running" | "completed" | "error"

class RoadTick(BaseModel):
    road_id: str
    density: float          # vehicles per km, or occupancy %
    avg_speed: float        # m/s or km/h
    queue_length: int       # halted vehicles

class SimulationTick(BaseModel):
    run_id: str
    tick: int                     # simulation second/step
    sim_clock: str                 # e.g. "09:14:00"
    roads: List[RoadTick]
    imbalance_index: float         # Gini-style coefficient, 0 (even) - 1 (very uneven)
    avg_travel_time: float
    avg_wait_time: float
    vehicles_active: int

class RunSummary(BaseModel):
    run_id: str
    mode: Mode
    time_window: TimeWindow
    final_imbalance_index: float
    avg_travel_time: float
    avg_wait_time: float
    total_vehicles: int
    imbalance_reduction_pct: Optional[float] = None  # vs. baseline, filled when both runs exist
```

## 2. REST Endpoints

| Method | Path | Body / Params | Response | Purpose |
|---|---|---|---|---|
| POST | `/scenario` | `ScenarioRequest` | `RunResponse` | Register a new run configuration, returns `run_id` |
| POST | `/run/{run_id}/start` | — | `RunResponse` | Starts stepping the SUMO simulation for that run |
| POST | `/run/{run_id}/reset` | — | `{status:"reset"}` | Stops and clears a run |
| GET | `/run/{run_id}/summary` | — | `RunSummary` | Final metrics after a run completes |
| GET | `/corridors` | — | `List[{id, name, bounds}]` | Available corridors (for the selector UI) |
| GET | `/health` | — | `{status:"ok"}` | For deployment health checks |

## 3. WebSocket Contract

`WS /ws/simulation/{run_id}`

- Server pushes a `SimulationTick` JSON object roughly once per simulated second of wall-clock-equivalent time (batch multiple SUMO steps into one tick if SUMO is stepping faster than useful for UI rendering — target ~1 message/sec).
- On completion, server sends a final message: `{"event": "completed", "summary": RunSummary}`.
- On error, server sends: `{"event": "error", "message": "..."}` and closes the socket.

Client → server messages are not required for MVP (server drives the whole simulation); only add client-driven controls (pause/speed) if time allows.

## 4. In-Memory State (hackathon-appropriate — no DB needed)

For a 24-hour build, skip a database. Keep run state in a simple in-process dict keyed by `run_id`:

```python
RUNS: dict[str, RunState] = {}
```

Where `RunState` holds the SUMO/TraCI connection handle, current tick data, and accumulated metrics for the Gini calculation. This is sufficient because the demo runs a handful of scenarios live — persistence isn't a judged requirement. If the stretch "export report" feature is built, write the final `RunSummary` to a JSON/CSV file on disk rather than standing up a database.

## 5. Load-Imbalance Index Calculation

```python
def gini_coefficient(densities: list[float]) -> float:
    """0 = perfectly even distribution across roads, 1 = maximally uneven."""
    if not densities or sum(densities) == 0:
        return 0.0
    sorted_d = sorted(densities)
    n = len(sorted_d)
    cum = sum((i + 1) * d for i, d in enumerate(sorted_d))
    return (2 * cum) / (n * sum(sorted_d)) - (n + 1) / n
```

Compute this once per tick across all monitored roads' `density` values — this is the number that goes in `SimulationTick.imbalance_index`.

## 6. Error Handling Notes
- Wrap all TraCI calls — SUMO subprocess crashes are the most likely failure mode during a live demo; catch and surface as a WebSocket `error` event rather than letting the backend hang.
- Validate `corridor_id` and reject unknown values early with a clear 400, so the frontend never silently fails.
