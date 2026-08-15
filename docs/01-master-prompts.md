# AI Development Master Prompt Plan
## Project: TrafficBalance — ready-to-use prompts per module

Each team member can paste the relevant prompt into their AI coding tool as a starting point, then iterate. Fill in the `{{...}}` placeholders as the project firms up. Always paste the relevant section of the TRD/backend-schema doc alongside the prompt for grounding — don't rely on the AI remembering earlier conversation turns.

---

## 1. Simulation Lead — SUMO/TraCI setup

> "I'm building a SUMO traffic simulation of a real road corridor in Nagpur, India for a hackathon. Help me: (1) generate the workflow to export a corridor around {{specific road/area, e.g. Wardha Road near X junction}} from OpenStreetMap using SUMO's osmWebWizard, (2) write a `routes.rou.xml` demand profile with two named flows — 'morning_peak' (9AM-12PM) and 'evening_peak' (4PM-7PM) — where traffic load is deliberately asymmetric across parallel roads to simulate real uneven distribution, (3) write a baseline TraCI Python script that runs the simulation with fixed-time traffic lights and logs, per simulation second: per-edge vehicle count, mean speed, and halting (queued) vehicle count. Output clean, commented Python using the `traci` and `sumolib` packages."

Follow-up prompt once baseline works:
> "Now extend this script into a reusable Python module `sim_runner.py` with a function `run_simulation(sumocfg_path, mode: Literal['baseline','optimized'], on_tick: Callable)` that calls `on_tick(tick_data: dict)` once per simulated second, so a FastAPI layer can consume it via callback."

## 2. Algorithm Lead — control logic

> "Given a running SUMO simulation controlled via TraCI, write Python functions for: (1) `adaptive_signal_control(junction_id)` that reads queue length via `traci.lane.getLastStepHaltingNumber` for each approach lane and extends/shortens the current green phase proportionally, bounded by a min 10s / max 45s green time, (2) `congestion_weighted_reroute(edge_ids: list[str])` that computes a congestion multiplier per edge from `traci.edge.getLastStepOccupancy`, updates edge travel-time weights via `traci.edge.setEffort` (or `adaptTraveltime`), and calls `traci.vehicle.rerouteTraveltime` for a sampled fraction of vehicles so some are steered toward less-congested parallel roads. Explain the tunable parameters (reroute fraction, update interval, weight formula) so I can adjust them during testing."

Follow-up for the metric:
> "Write a `gini_coefficient(values: list[float]) -> float` function (standard economic inequality formula) and a `load_imbalance_index(road_densities: dict[str,float]) -> float` wrapper I can call once per tick to score how evenly traffic load is distributed across monitored roads."

## 3. Backend Lead — FastAPI service

> "Scaffold a FastAPI app with: (1) Pydantic models exactly matching this schema: {{paste backend-schema.md section 1}}, (2) REST endpoints matching this table: {{paste backend-schema.md section 2}}, (3) a WebSocket endpoint `/ws/simulation/{run_id}` that, given a background task already producing `SimulationTick` objects via a queue or callback, streams them to the connected client as JSON at roughly 1 message/second, and sends a final `{"event":"completed","summary":...}` message when the run ends. Use `asyncio` background tasks, not blocking calls, so multiple runs could theoretically run concurrently. Include CORS setup permissive enough for a Vite dev server on localhost."

## 4. Frontend Lead — React dashboard

> "Build a React (Vite) component set for a live traffic-simulation dashboard: (1) `ScenarioControls` — a form to pick time window (9AM-12PM / 4PM-7PM) and mode (Baseline/Optimized/Both), with a Run button that POSTs to `/scenario` then `/run/{run_id}/start`, (2) `MapHeatmap` using Leaflet — render a fixed set of road polylines colored on a green-amber-red scale based on a `density` prop that updates on every WebSocket tick, (3) `MetricHeadline` — a large animated number showing `imbalance_index` with a delta arrow vs. a baseline value, (4) `MetricChart` using Recharts — a live-updating dual-line chart plotting `imbalance_index` over time for baseline vs optimized runs. Wire all of this to a WebSocket hook `useSimulationSocket(runId)` that parses incoming `SimulationTick` JSON matching this schema: {{paste backend-schema.md section on SimulationTick}}."

## 5. Integration/Pitch Lead — repo, testing, deck

> "Write a `setup.sh`/`setup.ps1` script that installs Python deps from `requirements.txt`, Node deps via `npm install` in the frontend folder, and checks that the `sumo` binary is on PATH, printing a clear error with install instructions if not found (Windows-friendly, since the team develops on Windows)."

> "Draft a 5-slide pitch deck outline for a hackathon judging panel: Slide 1 problem reframe ('uneven distribution, not just congestion'), Slide 2 our approach (traffic load balancer analogy), Slide 3 live demo cue (no content, just a placeholder), Slide 4 before/after results (placeholder for real numbers), Slide 5 what's next (RL, more corridors, real sensor integration). Keep each slide to 3 bullet points max."

## 6. General prompting tips for the team
- Always include the relevant schema/contract snippet in the prompt — don't let each person's AI tool invent its own field names, or integration will break.
- Ask the AI to explain non-obvious code (especially the TraCI/SUMO calls) — you'll need to answer judges' questions about how it works, and "the AI wrote it" is not an acceptable answer in Q&A.
- When an AI-generated function doesn't work, paste the exact error back rather than re-describing the problem — faster convergence.
- Re-generate rather than hand-patch when a component drifts far from the schema — cheaper than debugging a half-AI half-human file at 2 AM.
