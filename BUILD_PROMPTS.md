# RELIEF RADAR — prompts to paste into Claude Code, in order

Run each prompt in your repo with Claude Code. Review and test before moving on.
Start each new milestone with `/clear` so the context stays focused.

## 0. Set up the repo
Read CLAUDE.md and every file in docs/. Then create the monorepo skeleton exactly as in
docs/implementation-spec.md "Repository structure": folders, a root README stub,
.gitignore, .env.example, ruff and pytest config, and empty package files. Don't write
features yet. Show me the tree when done.

## 1. Schemas and scenario data
Using docs/implementation-spec.md (Firestore layout, event envelope, Interpreter schema)
and docs/demo-scenario.md, write Pydantic v2 models in packages/schemas for every entity,
event and agent output. Add a script that exports JSON Schema and TypeScript types.
Then generate scenarios/hyderabad_flood/: communities.csv, facilities.csv, depots.csv,
vehicles.csv, dependencies.csv, a simple edges.geojson road graph connecting all
communities, depots and facilities (including edges E1–E14 and bridge B4), timeline.yaml
matching the event timeline, and the 40 reports (18 English, 14 Telugu, 8 Hindi) with
separate hidden label files, following the "Planted reports" table. Validate every file
against the schemas in a test.

## 2. Planning engine
Implement services/planner/engine per docs/implementation-spec.md "Planning engine":
tiered scoring, road graph and isolation forecast, cascade detection with smallest fix,
CP-SAT allocation with stability term, routes, audit, diff with reason codes, and the
first-come-first-served baseline. Write unit tests on small hand-made graphs for each
module first, then make them pass.

## 3. Scenario runner
Write eval/run_scenario.py that replays timeline.yaml in-process (no cloud services),
using labeled extractions instead of Gemini, and asserts each "Expected system response"
from docs/demo-scenario.md. Also write eval/metrics.py comparing RELIEF RADAR with the
baseline. Run it and fix the engine until all assertions pass. Report the measured metrics.

## 4. Gemini agents
Implement the five agents in agents/ with Google ADK and Gemini on Vertex AI, using the
Pydantic JSON Schemas for structured output, versioned prompts, the gazetteer for
locations, embeddings plus Gemini for duplicates, and the hash-keyed cache. Add a
cache-only mode flag. Write eval/extraction_eval.py scoring extractions against the
hidden labels. Check current Gemini model names before choosing one.

## 5. Services and events
Build the api, ingest and planner FastAPI services per the spec: endpoints, Pub/Sub push
handlers with OIDC verification, idempotency, ordering by session, 2-second debounce,
Firestore writes per session sandbox, and fallback to the last valid plan. Add
infra/setup_pubsub.sh and infra/setup_bigquery.sh (BigQuery subscriptions for rr-plans and
rr-audit). Include a local mode using the Firestore and Pub/Sub emulators.

## 6. Frontend
Build apps/web per docs/use-cases-ui.md and the spec's Frontend section: anonymous Firebase
sign-in, session creation, Firestore listeners into a Zustand store, the coordinator
screen (TopBar, PriorityList, LiveMap with all layers, AlertsPanel, PlanPanel with
approve/edit/reject, EvidenceDrawer, Simulator, GuidedTour), NGO view, field report form,
metrics panel, and "Simulated" labels on synthetic data.

## 7. Earth Engine and landslide pack
Write hazard_packs/flood/ee_recipe.py (Sentinel-1 change detection for the October 2020
Hyderabad floods, minus JRC permanent water, slope mask, severity polygons) and the seed
pipeline that intersects zones with road edges. Then add the landslide pack and
scenarios/landslide_valley per docs/demo-scenario.md.

## 8. Deploy and harden
Write Dockerfiles, cloudbuild.yaml, firebase.json (with /api rewrite), firestore.rules
(from the spec) and the TTL policy setup. Deploy, then walk through the "Submission and
evaluation readiness" table in docs/technical-plan.md and fix anything that fails.

## 9. Submission materials
Write the README (architecture, how Firebase, Firestore, Cloud Run and Gemini are used,
setup, scenario, measured metrics). Draft the brief description for the submission form.
