# RELIEF RADAR — project context for Claude Code

## What we are building
RELIEF RADAR is a hazard-agnostic AI decision engine for disaster relief, built for the
Google Cloud AI Builder Cup (theme: Sustainability & Social Impact). It ingests helpline
calls, citizen reports, NGO updates, news, social posts, official alerts and satellite
data; predicts which communities will be cut off and which services will fail next; and
allocates scarce supplies, vehicles and rescue teams under real constraints and the
authority's SOPs. It replans live when conditions change and explains every change.
**AI recommends, humans decide**: a coordinator approves every dispatch.

Full design lives in `docs/` (exported from the team's planning doc). Read the relevant
file before starting any task:
- `docs/architecture-decisions.md` — tooling, package layout, adapters, debounce, config,
  final repo tree. **Wins where the other docs conflict.** Read it before every milestone.
- `docs/idea-brief.md` — problem, users, features, positioning
- `docs/technical-plan.md` — stack, data sources, real-time strategy, evaluation readiness
- `docs/implementation-spec.md` — services, events, Firestore, agents, engine algorithms
- `docs/demo-scenario.md` — entities, event timeline, 40 reports, pass criteria
- `docs/use-cases-ui.md` — screens, roles, wireframe
- `docs/pitch-deck.md` — submission deck content

## Non-negotiable rules
1. **Deterministic core, AI at the edges.** Gemini only extracts, verifies, explains and
   drafts SOP rules. Ranking, allocation, routing and validation are plain Python
   (NetworkX, OR-Tools CP-SAT). No number in a plan may come from an LLM.
2. **Life safety first.** `rescue` and `medical_time_critical` needs are never delayed by
   verification. Low confidence creates a verification task; it never lowers priority.
3. **Every input is an event** with the shared envelope (see implementation spec).
   Handlers must be idempotent on `event_id`.
4. **One sandbox per session** in Firestore under `sessions/{sid}`. Browser is read-only;
   all writes go through the api, ingest or planner services.
5. **Schemas first.** Pydantic models in `packages/schemas` are the single source of
   truth; generate JSON Schema (for Gemini structured output) and TS types from them.
6. **Demo-proof.** Precomputed Earth Engine layers only; cached Gemini outputs on the
   scripted path; if a new plan fails audit, keep the previous valid plan.
7. **Honesty.** All synthetic data is labeled "Simulated" in the UI. Never hard-code or
   invent benchmark results; metrics come from `eval/`.
8. **No secrets in the repo.** Use ADC/service accounts and Secret Manager; keep
   `.env.example` only.

## Required Google services (judges check these)
Firebase Hosting + Firebase Authentication (anonymous demo login), Firestore (live state,
real-time listeners), Cloud Run (api, ingest, planner), Gemini on Vertex AI via ADK.
Supporting: Pub/Sub, BigQuery (via Pub/Sub BigQuery subscriptions), Earth Engine
(offline pipelines), Cloud Vision (OCR), Cloud Storage.

## Stack and conventions
- Python 3.11+, FastAPI, Pydantic v2, pytest, ruff. One Dockerfile per service.
- Frontend: React + Vite + TypeScript, MapLibre, Firebase JS SDK, Zustand.
- Region: `asia-south1` for Cloud Run, Firestore, Pub/Sub. Vertex AI region: wherever the
  chosen Gemini model is available.
- Pin dependency versions; check current Gemini model names on Vertex AI before coding.
- Small, reviewable commits; run tests before every commit.

## Repository layout
See `docs/architecture-decisions.md` → "Repository tree" (the spec's tree plus a few
additions). Keep to it.

## Build order (milestones)
1. Schemas + scenario data files (`scenarios/hyderabad_flood/*`) from `docs/demo-scenario.md`
2. Planning engine with unit tests (scoring, isolation, cascade, allocation, audit, diff, baseline)
3. In-process scenario runner (`eval/run_scenario.py`) asserting the timeline's expected responses
4. Agents (Interpreter, Verifier, Planner explanation, Auditor summary, SOP extractor) + cache
5. Services (api, ingest, planner) + Pub/Sub wiring + Firestore writes
6. Frontend: coordinator screen, simulator, guided tour, evidence drawer, NGO and field views
7. Earth Engine flood layer + seed pipeline; landslide pack
8. Deploy (Cloud Build, Cloud Run, Firebase Hosting), evaluation readiness checklist
9. Benchmarks, README, video, deck snapshots

## Definition of done
All items in `docs/demo-scenario.md` → "Pass criteria" pass on the deployed app from a
fresh incognito browser, with two sessions running at once.
