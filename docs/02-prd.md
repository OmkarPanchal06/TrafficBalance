# Product Requirements Document (PRD)
## Project: TrafficBalance — AI-Driven Load Balancer for Uneven Traffic Distribution
**Event:** Vikasit Nagpur Hackathon (24 hours) · **Team size:** 5

---

## 1. Problem Statement
The Planning Authority's jurisdiction experiences **uneven distribution of traffic** across its road network — some roads/junctions are heavily congested while parallel or nearby roads remain underused — specifically during two daily peak windows: **9 AM–12 Noon** and **4 PM–7 PM**.

Most conventional solutions treat this as a single-junction signal-timing problem. The actual ask is network-level: balance load *across* roads, not just reduce wait time at one point.

## 2. Product Vision
Build a simulation-backed system that behaves like a **load balancer for road traffic**: it continuously senses congestion across the jurisdiction and actively redistributes vehicle flow — via adaptive signal timing and congestion-aware routing — toward underused roads, provably reducing the *imbalance* of load, not just the average wait time.

## 3. Target "User" (for hackathon judging purposes)
- Primary: the **Planning Authority** — needs a decision-support tool showing where imbalance exists and how an intervention would help, with clear before/after evidence.
- Secondary: a citizen-facing angle — commuters who would receive/benefit from rerouting suggestions.

## 4. Goals
1. Demonstrate measurable reduction in traffic-load imbalance across a real Nagpur road segment during both peak windows.
2. Provide a live, visual, judge-legible dashboard (map/heatmap + metrics) proving the claim in real time.
3. Ship a fully working, crash-resistant demo within 24 hours, built by a 5-person team.

## 5. Non-Goals (Out of Scope for the hackathon)
- Real-time integration with actual traffic cameras/IoT sensors (simulate this instead).
- City-wide coverage — one well-modeled corridor (3–5 junctions) is sufficient and safer.
- Full reinforcement-learning training pipeline (use heuristic control; mention RL as future work).
- Mobile app — a responsive web dashboard is sufficient.

## 6. Features

### Must-Have (MVP — this is what gets judged)
| # | Feature | Description |
|---|---|---|
| M1 | Real road network | SUMO simulation of an actual Nagpur corridor (via OSM), 3–5 junctions |
| M2 | Dual peak-hour scenarios | Configurable demand profiles for 9–12 and 4–7 windows |
| M3 | Baseline mode | Fixed-timing signal simulation, metrics logged |
| M4 | Optimized mode | Queue-proportional adaptive signals + congestion-weighted rerouting |
| M5 | Load-Imbalance Index | Headline metric (Gini-style) showing distribution across roads, before vs. after |
| M6 | Live dashboard | Map/heatmap of per-road congestion, updating in real time from the sim |
| M7 | Before/After comparison view | Toggle or side-by-side of baseline vs. optimized run |
| M8 | Stable end-to-end demo | One full run, start to finish, without crashing |

### Nice-to-Have (Stretch — only after MVP is solid)
| # | Feature | Description |
|---|---|---|
| S1 | Second corridor | Shows the approach generalizes beyond one road segment |
| S2 | Forecast panel | Simple moving-average prediction of next peak window's imbalance |
| S3 | Citizen notification mock | Simulated "alternate route suggested" push notification |
| S4 | Exportable authority report | One-click PDF/CSV summary for the Planning Authority |
| S5 | Scenario editor | Let judges tweak demand and re-run live |

## 7. Success Metrics (for the demo)
- Load-Imbalance Index reduced by a visible, explainable margin (target: state a concrete % drop once measured — don't promise a number you haven't tested).
- Average travel time not worse than baseline (ideally improved).
- Zero crashes across at least 5 consecutive full demo runs before presenting.

## 8. Risks & Mitigations
| Risk | Mitigation |
|---|---|
| SUMO/OSM network too large to model in time | Scope to one corridor early (hour 0–1 decision), never expand mid-hackathon |
| Live demo fails during judging | Record a fallback video by hour 19; rehearse switching to it seamlessly |
| Algorithm doesn't show a clean improvement | Start baseline + metric extraction by hour 2 so there's time to tune before hour 14 |
| Team blocked on API contract | Lock the WebSocket/REST contract in hour 0–1; frontend/backend build against mocked data first |

## 9. Judging Narrative
Lead with the reframe: this solves *distribution*, not just *congestion* — most competitors will misread the brief. Show the Load-Imbalance Index dropping live. Close with the "traffic load balancer" analogy — instantly legible to a technical panel.
