# Implementation Plan
## Project: TrafficBalance — detailed build, integration, deployment, and testing plan

---

## 1. Which AI tool for what

| Task | Best tool | Why |
|---|---|---|
| Backend/algorithm scaffolding (FastAPI, TraCI scripts, control logic) | **Claude Code** (terminal or desktop) | Can read your whole repo, run commands, iterate against real errors — best fit for a real codebase, not snippets |
| Frontend component generation (React + Tailwind) | **Claude Code**, or **v0.dev** for fast first-draft visual layout | v0 is strong for quick visual scaffolds; Claude Code for wiring real data/state after |
| SUMO/TraCI-specific syntax questions (config file formats, edge cases) | **Claude or ChatGPT (chat, with web search on)** | These configs are fiddly XML/CLI formats — a quick chat lookup is faster than digging docs mid-hackathon |
| Debugging a specific stack trace | Whichever coding agent already has the file open (Claude Code/Cursor) | Context of the actual code matters more than raw model quality here |
| Pitch deck copy and slide structure | **Claude (chat)** | Strong at concise persuasive writing; ask for bullet-point tight copy, not paragraphs |
| Quick one-off boilerplate (a Pydantic model, a Gini function) | Any chat AI (Claude/ChatGPT) — doesn't need repo context | Fast, no setup needed |
| Generating realistic demand data / traffic flow numbers | Claude or ChatGPT **with web search enabled** | Ground it in something real (typical peak-hour vehicle counts) rather than made-up numbers, so your "realistic" claim holds up under judge questioning |

Practical rule for the team: use an **agentic coding tool (Claude Code)** for anything touching multiple files or that needs to run/test itself — that's most of the backend and control logic. Use **plain chat AI** for isolated code snippets, config syntax, and writing (deck, docs). Don't use a chat AI for tasks that need to see your actual running code — you'll waste time re-pasting context that an agentic tool already has.

## 2. Integration Plan

**Step 1 — Contract first (hour 0–1).** Backend and Frontend leads agree on the exact JSON shapes in `04-backend-schema.md` before either writes real logic. Commit the schema doc to the repo root so every AI prompt can reference it.

**Step 2 — Mock the seams (hour 1–6).** Backend serves a WebSocket that emits fake but correctly-shaped `SimulationTick` data (a simple loop generating random-but-plausible numbers) so Frontend can build against real message shapes immediately, without waiting on SUMO to be ready. Frontend and Backend are now fully decoupled and parallel.

**Step 3 — Swap in the real pipeline (hour 6–14).** Once Simulation + Algorithm leads have a working `sim_runner.py` producing real tick data, Backend Lead replaces the mock generator with real callback data from `sim_runner.run_simulation(...)`. Frontend needs zero changes if the schema was respected — this is the payoff of Step 1.

**Step 4 — Continuous integration checks (from hour 6 onward).** Integration Lead runs a full flow (start app → run scenario → see it complete) every ~2 hours, not just at the end. Log every break in a shared doc/board so the right person fixes it fast — catching a broken contract at hour 8 costs 10 minutes; catching it at hour 22 can cost the demo.

**Step 5 — Freeze and stabilize (hour 19–22).** No new features. Only bug fixes. Run the exact demo script 5+ times.

## 3. Deployment Plan (free hosting)

SUMO needs a real running process (not serverless functions), so backend and frontend need different hosting strategies:

**Backend (FastAPI + SUMO):**
- **Render.com free web service** — supports a `Dockerfile`, which lets you install the `sumo` apt package alongside Python deps. Free tier sleeps after inactivity, so ping it a few minutes before your judging slot.
- **Railway.app free tier** — alternative with a similar Docker-based deploy, generous enough for a hackathon's usage window.
- Minimal `Dockerfile` approach:
  ```dockerfile
  FROM python:3.11-slim
  RUN apt-get update && apt-get install -y sumo sumo-tools sumo-doc
  ENV SUMO_HOME=/usr/share/sumo
  WORKDIR /app
  COPY requirements.txt .
  RUN pip install -r requirements.txt
  COPY . .
  CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
  ```

**Frontend (React/Vite):**
- **Vercel** or **Netlify free tier** — connect the GitHub repo, auto-deploys on push, trivial for a Vite static build. Set the `VITE_API_URL` env var to point at the deployed backend's URL (and `wss://` for the WebSocket).

**Fallback — local-only demo:** If free-tier cold starts or WebSocket support over the free hosting proves flaky (common failure mode under hackathon wifi), **run the whole stack locally on the presenting laptop** and treat the hosted deployment as a bonus "yes it's deployed" link for judges to check afterward, not the live demo path. Judges care far more about a smooth demo than about where it's hosted.

**Order of operations:** deploy early (by hour 14, once the real pipeline is integrated) so you have time to discover and fix hosting-specific issues (CORS, WebSocket proxy support, cold starts) well before presenting — don't deploy for the first time at hour 23.

## 4. Testing Plan

| Test type | What | When | Who |
|---|---|---|---|
| Unit tests | `gini_coefficient()`, adaptive signal timing math, individual TraCI wrapper functions | As each function is written | Algorithm Lead |
| Component tests | Does `MapHeatmap` render correctly given mock tick data at various density levels? | During Frontend build | Frontend Lead |
| API contract tests | Does every REST/WS response actually match `04-backend-schema.md`? | Right after mock endpoints exist, and again after real data is swapped in | Backend Lead |
| Integration/E2E | Full flow: select scenario → run → see live updates → see summary, with no manual restarts | Every ~2 hours from hour 6 onward | Integration Lead (whole team present for the hour-19 run) |
| Load/stability test | Run the full demo scenario 5+ times back-to-back without restarting the backend, watch for memory leaks or SUMO subprocess zombies | Hour 19–22 | Integration Lead |
| Deployed-environment test | Same E2E flow against the hosted URLs, not just localhost | Once deployed (~hour 14–16), and once more right before presenting | Integration Lead |
| Judge Q&A rehearsal | Each person can explain their module's logic in plain language without reading code | Hour 22–24 | Whole team |

**Definition of "successfully tested" for this hackathon:** the exact sequence you'll perform live (open app → pick 9–12 window → run Both → watch it complete → land on summary) has run **five consecutive times with zero manual intervention and zero crashes**, on the actual device/network you'll present from. Anything short of that means keep testing, not adding features.

## 5. Master Timeline Reference

See the earlier 7-phase hackathon timeline (hour 0–24) discussed in chat — this implementation plan should be read alongside it: that doc gives the *when*, this doc gives the *how*.
