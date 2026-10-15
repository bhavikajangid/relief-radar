# Architecture decisions

Decisions that refine `implementation-spec.md` and `technical-plan.md`. Where the docs
disagree, this file wins. Each entry: decision, why, consequence. Change an entry only by
editing it here in the same commit as the code that changes the decision.

GCP project: `relief-radar-511713` · Region: `asia-south1` · Decided 2026-10-14.

## D1 Toolchain
- **Decision:** a uv workspace with one root `pyproject.toml` and one `uv.lock`. Python 3.12
  is pinned in `.python-version`. ruff and pytest are configured at the root. Frontend uses
  Node 22, npm, Vite, React and TypeScript.
- **Why:** three services, the agents, eval and pipelines share schemas and code. One
  lockfile stops versions drifting between them. 3.12 has stable wheels for OR-Tools and ADK.
- **Consequence:** `uv sync` sets up everything, and `uv run pytest` tests everything.
  Exact versions are pinned when packages are first installed.

## D2 Package layout
Every Python member uses a src layout and an `rr_` import prefix.

| Path | Import | Contains |
|---|---|---|
| `packages/schemas` | `rr_schemas` | All Pydantic models: entities, event envelope and typed payloads, agent outputs, plan, hazard pack config. The `rr_schemas.export` module writes JSON Schema and `apps/web/src/lib/types.ts` (via json-schema-to-typescript) |
| `packages/core` | `rr_core` | Ports and adapters: `Bus` (in-memory, Pub/Sub), `StateStore` (in-memory, Firestore), ID helpers, settings, JSON logging, OIDC push verification |
| `agents` | `rr_agents` | The five ADK agents, `prompts/*.vN.md`, gazetteer matcher, AI cache |
| `services/api` | `rr_api` | FastAPI shell plus handlers |
| `services/ingest` | `rr_ingest` | FastAPI shell plus handlers |
| `services/planner` | `rr_planner` | FastAPI shell plus handlers; the engine is `rr_planner.engine` |
| `eval`, `pipelines` | scripts run via `uv run` | Depend on the members above |

- **Why:** each service image installs only what it needs, and eval can import the
  service handlers directly.
- **Consequence:** Earth Engine and geo libraries live in an optional `pipelines` dependency
  group, so service images stay slim. `packages/core` is new compared with the spec's tree.

## D3 Ports and adapters
- **Decision:** service logic is written as `handle(event, store, bus)` functions, and
  `app.py` only adapts HTTP and Pub/Sub push to them. The engine is a pure function:
  `plan(SessionState, prev_plan, rules) -> PlanResult`. The `RR_BUS` setting
  (`memory | pubsub`) and the `RR_STORE` setting (`memory | firestore`) choose the adapters.
- **Why:** milestone 3 (the in-process scenario runner) has to work before any cloud
  wiring exists (milestone 5). Pure handlers are also easy to unit test.
- **Consequence:** tests and `eval/run_scenario.py` use in-memory adapters, local dev uses
  the emulators, and cloud uses Pub/Sub and Firestore. One code path serves all three.

## D4 IDs and time
- **Decision:** IDs are prefixed ULIDs: `ses_`, `evt_`, `rep_`, `need_`, `alt_`. Plan
  versions are integers. The envelope keeps the spec's `scenario_time: "T+HH:MM"` string,
  validated by pattern, with a `.minutes` integer accessor. The engine works only in minutes.
- **Why:** ULIDs sort by time, and the prefixes make logs and Firestore easy to read.

## D5 Delivery, idempotency, debounce
- `rr-reports-raw` is ordered, with ordering key `session_id`, because duplicate detection
  depends on arrival order.
- `rr-state-changes` is **unordered**. The planner is state-based (it re-reads the full
  session snapshot), so event order doesn't matter. Ordered push delivery would also hold
  each message until the previous one is acked, which would serialize bursts and defeat
  the debounce.
- **Debounce:** the planner handler increments `sessions/{sid}.replan_gen` in a
  transaction, sleeps 2 seconds, then replans only if `replan_gen` still equals its own
  value. Otherwise it acks, because a later message owns the replan. This works on
  stateless scale-to-zero instances with no Cloud Tasks.
- **Idempotency:** each handler first `create()`s `processed_events/{event_id}`. If the
  document already exists, the handler skips the event.
- **In-process runner:** no debounce, one replan per timeline tick.

## D6 BigQuery (minimal scope)
- **Decision:** dataset `relief_radar` with these tables:
  - `events`, filled by the rr-audit subscription
  - `plans`, filled by the rr-plans subscription
  - reference tables `communities`, `road_edges` and `hazard_zones`, loaded by
    `pipelines/seed` for GIS joins

  Subscription tables store the message as JSON `data` plus metadata columns. No service
  writes to BigQuery directly.
- **Why:** the spec's design gets a full audit log for free. Technical-plan's 12-table
  mirror would need a writer for each table and adds build time without demo value.

## D7 AI modes and cache
- **Decision:** `RR_AI_MODE = live | cache_only | labels`.
  - `labels` returns the hidden scenario labels as extractions, so eval runs before the
    agents exist and in CI.
  - `cache_only` never calls Gemini; a cache miss becomes "needs human review".
  - The cache key is sha256(model + prompt_version + canonical JSON input).
- **Scripted-path outputs** are committed as JSON under `agents/cache/` and seeded into
  Firestore `ai_cache/`, so the demo is reproducible and survives running out of quota.
- **Open until milestone 4:** the Gemini model names (`RR_GEMINI_MODEL`) and the Vertex
  region (`VERTEX_LOCATION`). Check the current Vertex AI catalog before choosing.

## D8 Configuration
- **Decision:** pydantic-settings with an `RR_` prefix, defined in `rr_core.settings`.
  `.env.example` lists these variables:
  - `GOOGLE_CLOUD_PROJECT=relief-radar-511713`
  - `RR_REGION=asia-south1`
  - `VERTEX_LOCATION`, `RR_GEMINI_MODEL`, `RR_AI_MODE`, `RR_BUS`, `RR_STORE`
  - `FIRESTORE_EMULATOR_HOST`, `PUBSUB_EMULATOR_HOST`, `FIREBASE_AUTH_EMULATOR_HOST`
  - `VITE_FIREBASE_*` (the Firebase web config is public, not a secret)

  Real values never go in the repo. Services use ADC and service accounts.

## D9 Local development
- **Decision:** `make dev` starts the Firebase emulators (Firestore, Auth, Pub/Sub), the
  three uvicorn services and Vite. Emulator push subscriptions point at localhost.
- **Prerequisites:** `firebase-tools` and Java (for the emulators).

## D10 Build and deploy
- **Decision:** one Dockerfile per service at `services/<svc>/Dockerfile`. The build
  context is the repo root, and each image installs with
  `uv sync --frozen --no-dev --package rr-<svc>`.
- **CI/CD:** a Cloud Build trigger on push to main runs the tests, builds the images,
  deploys the three Cloud Run services, then runs `firebase deploy`.
- **Identity:** one least-privilege service account per service, created by
  `infra/bootstrap.sh`.

## D11 Doc conflicts resolved
Where the docs disagree, the implementation spec wins over the technical plan:
1. The web app is served by Firebase Hosting, not Cloud Run.
2. Need types include `medical_time_critical`, so the full list is rescue,
   medical_time_critical, medical, water, food, shelter and fuel.
3. BigQuery scope is minimal (D6), not the 12-table mirror.
4. `firebase.json` and `.firebaserc` go at the repo root (Firebase CLI convention), not in
   `infra/`. Everything else in `infra/` stays as the spec says.

## Repository tree
This is the spec's tree with the additions marked `+`.

```
relief-radar/
├── + pyproject.toml              uv workspace root, ruff + pytest config
├── + uv.lock  + .python-version  + .gitignore  + .env.example  + Makefile
├── + firebase.json  + .firebaserc               hosting (/api rewrite), emulators, rules
├── apps/web/                     React + Vite + TS, MapLibre, Firebase SDK, Zustand
│   └── src/
│       ├── components/           TopBar, PriorityList, LiveMap, AlertsPanel, PlanPanel,
│       │                         EvidenceDrawer, Simulator, GuidedTour, SopRules,
│       │                         NgoView, FieldReportForm, MetricsPanel
│       ├── lib/                  firebase.ts, api.ts, listeners.ts, store.ts, types.ts (generated)
│       └── layers/               hazard, roads, communities, facilities, routes
├── services/
│   ├── api/                      + Dockerfile, pyproject.toml, src/rr_api/, tests/
│   ├── ingest/                   + Dockerfile, pyproject.toml, src/rr_ingest/, tests/
│   └── planner/                  + Dockerfile, pyproject.toml, src/rr_planner/, tests/
│       └── src/rr_planner/engine/  scoring.py, isolation.py, cascade.py, allocate.py,
│                                   routes.py, audit.py, diff.py, baseline.py
├── agents/                       pyproject.toml, src/rr_agents/, prompts/, + cache/, tests/
├── packages/
│   ├── schemas/                  src/rr_schemas/ (models, export.py), generated/, tests/
│   └── + core/                   src/rr_core/ (bus, store, ids, settings, logging, auth), tests/
├── hazard_packs/
│   ├── flood/                    pack.yaml, ee_recipe.py
│   └── landslide/                pack.yaml, ee_recipe.py
├── pipelines/
│   ├── earth_engine/             run recipes, export GeoJSON to Cloud Storage
│   └── seed/                     scenario templates → Firestore, reference tables → BigQuery
├── scenarios/
│   ├── hyderabad_flood/          communities.csv, facilities.csv, depots.csv, vehicles.csv,
│   │                             edges.geojson, dependencies.csv, timeline.yaml,
│   │                             + gazetteer.yaml, reports/*.json, labels/*.json
│   └── landslide_valley/
├── eval/                         run_scenario.py, metrics.py, extraction_eval.py
├── infra/                        firestore.rules, firestore.indexes.json, cloudbuild.yaml,
│                                 setup_pubsub.sh, setup_bigquery.sh, + bootstrap.sh
├── docs/
└── README.md
```

## Project bootstrap checklist
Do these before milestone 5:
1. Run `gcloud auth login`, then `gcloud config set project relief-radar-511713`.
2. Run `npm i -g firebase-tools`, install Java, then run `firebase login`.
3. Register the project for Earth Engine.
4. Confirm billing and trial credits, then set low-threshold budget alerts.
5. Confirm Gemini model availability per region in Vertex AI (D7).
6. Run `infra/bootstrap.sh`: enable APIs, create service accounts, create the Artifact
   Registry repo.
