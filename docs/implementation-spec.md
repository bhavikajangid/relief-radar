Implementation spec
Design principles
Five rules shape every implementation choice below.
1. Deterministic core, AI at the edges. Gemini reads, verifies and explains. Python ranks, allocates, routes and validates. No number in a plan comes from an LLM.
2. Everything is an event. Reports, road changes, approvals and simulator ticks all travel as Pub/Sub messages with one envelope, so live replanning and the audit log come for free.
3. One sandbox per session. Each judge or user gets an isolated copy of the scenario in Firestore. Nothing one session does is visible to another.
4. Schemas first. One set of Pydantic models generates the JSON Schemas for Gemini structured output and the TypeScript types for the frontend. Backend, ML and frontend never drift.
5. Demo-proof by default. Precomputed hazard layers, cached AI outputs on the scripted path, and a plan that falls back to the last valid version if anything fails.
Repository structure
One monorepo, so the README, CI and judges see everything in one place.
relief-radar/
├── apps/web/                    React + Vite + TypeScript, MapLibre, Firebase SDK
│   └── src/
│       ├── components/          TopBar, PriorityList, LiveMap, AlertsPanel, PlanPanel,
│       │                        EvidenceDrawer, Simulator, GuidedTour, SopRules,
│       │                        NgoView, FieldReportForm, MetricsPanel
│       ├── lib/                 firebase.ts, api.ts, listeners.ts, types.ts (generated)
│       └── layers/              hazard, roads, communities, facilities, routes
├── services/
│   ├── api/                     FastAPI on Cloud Run: sessions, reports, approvals, simulator
│   ├── ingest/                  FastAPI on Cloud Run: OCR + Interpreter + Verifier agents
│   └── planner/                 FastAPI on Cloud Run: engine + Auditor + Explainer
│       └── engine/              scoring.py, isolation.py, cascade.py, allocate.py,
│                                routes.py, audit.py, diff.py, baseline.py
├── agents/                      ADK agent definitions, prompts/, prompt versions
├── packages/schemas/            Pydantic models → JSON Schema + TS types
├── hazard_packs/
│   ├── flood/                   pack.yaml, ee_recipe.py
│   └── landslide/               pack.yaml, ee_recipe.py
├── pipelines/
│   ├── earth_engine/            run recipes, export GeoJSON to Cloud Storage
│   └── seed/                    load scenarios into Firestore templates and BigQuery
├── scenarios/
│   ├── hyderabad_flood/         communities.csv, facilities.csv, depots.csv,
│   │                            vehicles.csv, edges.geojson, dependencies.csv,
│   │                            timeline.yaml, reports/*.json, labels/*.json
│   └── landslide_valley/
├── eval/                        run_scenario.py, metrics.py, extraction_eval.py
├── infra/                       firebase.json, firestore.rules, firestore.indexes.json,
│                                cloudbuild.yaml, setup_pubsub.sh, setup_bigquery.sh
└── README.md
Services
Three Cloud Run services plus static hosting. Firebase Hosting serves the web app and rewrites /api/** to the api service, so the frontend has one origin and no CORS setup.
Service
Trigger
Responsibilities
Writes to
web (Firebase Hosting)
Browser
React app; reads live state through Firestore listeners; calls /api for actions
Nothing directly (read-only Firestore)
api (Cloud Run, FastAPI)
HTTPS from web
Create, reset and fetch sessions; submit reports; approve or reject plans; approve SOP rules; advance the simulator clock
Firestore (session meta, decisions); publishes events
ingest (Cloud Run, FastAPI)
Pub/Sub push: rr-reports-raw
Vision OCR on media; Incident Interpreter; Evidence Verifier; resolves locations against the gazetteer
Firestore (reports, needs, edge updates); publishes rr-state-changes
planner (Cloud Run, FastAPI)
Pub/Sub push: rr-state-changes
Scoring, isolation, cascade, allocation, routes; Plan Auditor; Response Planner explanation; diff vs previous plan
Firestore (plan versions, alerts); publishes rr-plans
API endpoints (api service)
Method and path
Purpose
POST /api/sessions
Create a sandbox from a scenario template; returns session_id
POST /api/sessions/{sid}/reset
Restore the template state and clock to T+0
POST /api/sessions/{sid}/sim/advance
Publish all timeline events up to the requested scenario time
POST /api/sessions/{sid}/pack
Switch hazard pack (flood or landslide scenario)
POST /api/sessions/{sid}/reports
Submit a new report (text, optional media upload URL)
POST /api/sessions/{sid}/plans/{version}/decision
Approve, edit or reject assignments
POST /api/sessions/{sid}/sop
Upload an SOP; returns draft rules for review
POST /api/sessions/{sid}/sop/{ruleset}/approve
Activate approved rules
GET /api/health
Uptime check
The simulator is client-driven: the Play button calls sim/advance every few seconds. No background scheduler is needed, and a judge who closes the tab simply stops the clock.
Event bus
Topic
Published by
Consumed by
Purpose
rr-reports-raw
api (new reports, simulator)
ingest
Unprocessed reports
rr-state-changes
ingest, api (road closures, decisions, pack switch)
planner
Anything that can change the plan
rr-plans
planner
BigQuery subscription
Plan versions for the audit log
rr-audit
all services
BigQuery subscription
Every event, for after-action review and metrics
Pub/Sub writes rr-plans and rr-audit straight into BigQuery through BigQuery subscriptions, so no logging service has to be written.
Every message uses one envelope:
{
  "event_id": "evt_01J…",
  "session_id": "ses_8f2…",
  "type": "report.received | road.status | facility.status | decision | pack.switch | sim.tick",
  "scenario_time": "T+04:00",
  "created_at": "2026-10-13T10:00:00Z",
  "source": { "channel": "helpline", "trust": "high" },
  "payload": { },
  "trace_id": "…"
}
• Ordering: ordering key = session_id, so each session's events arrive in order.
• Idempotency: services record processed event_ids per session and skip repeats, since Pub/Sub can deliver more than once.
• Auth: push subscriptions use OIDC tokens; Cloud Run services accept only those, not public traffic (except api).
• Debounce: the planner waits about 2 seconds after a state change to batch bursts (e.g., 18 reports at once) into one replan.
Firestore
Firestore holds the live state of every session; the dashboard subscribes to it with real-time listeners. BigQuery keeps the permanent record.
Layout
scenarios/{scenario_id}                 read-only templates (seeded by pipelines/seed)
  └── communities, facilities, depots, vehicles, edges, dependencies, timeline

sessions/{sid}                          owner_uid, scenario_id, pack, clock, status, created_at, expires_at
  ├── communities/{id}                  population, vulnerable, exposure, isolation_eta, priority_score
  ├── facilities/{id}                   type, capacity, occupancy, fuel_hours_left, status
  ├── edges/{id}                        status (open | risky | closed), source, updated_at
  ├── stock/{depot_id}                  items: { water_kits, food_kits, medical_kits, diesel_l }
  ├── vehicles/{id}                     type, capacity, location, status, owner_org
  ├── reports/{id}                      raw text, channel, language, media_uri, extraction, verification
  ├── needs/{id}                        community, type, people, quantity, deadline, life_safety, status, evidence[]
  ├── alerts/{id}                       kind (isolation | cascade | blind_spot | overlap | conflict), refs, severity
  ├── plans/{version}                   assignments[], unmet[], explanation, diff, audit, status
  ├── decisions/{id}                    plan_version, action, by, at, notes
  ├── sop_rules/{ruleset}               rules[], clause refs, draft | approved
  └── processed_events/{event_id}       idempotency markers

ai_cache/{hash}                         cached Gemini outputs for scripted inputs
Rules for writing
• The browser only reads session data. Every write goes through the api, ingest or planner services using their service accounts, so a judge can't corrupt state from the console.
• Firestore security rules allow a signed-in user to read sessions/{sid}/** only when owner_uid == request.auth.uid, and deny all client writes.
• A TTL policy on sessions.expires_at (24 hours) deletes old sandboxes automatically, keeping storage and listener reads low.
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    match /sessions/{sid}/{document=**} {
      allow read: if request.auth != null
        && get(/databases/$(db)/documents/sessions/$(sid)).data.owner_uid == request.auth.uid;
      allow write: if false;
    }
    match /scenarios/{document=**} { allow read: if request.auth != null; allow write: if false; }
    match /ai_cache/{document=**} { allow read, write: if false; }
  }
}
Read budget
The free tier allows 50K reads a day. Listeners re-read only changed documents, so one full scenario run is roughly a few hundred to a couple of thousand reads. Measure it in testing; if judges' usage gets close to the limit, upgrade or merge small collections into a single summary document.
AI agents
Five Gemini agents built with the Agent Development Kit, each with a narrow job, a strict output schema, and a deterministic check around it.
Agent
Runs in
Input
Output
Guardrail
Incident Interpreter
ingest
Raw text, OCR text, channel, language hint, gazetteer of place names
ReportExtraction JSON
Schema validation; one retry with the error; else "needs human review"
Evidence Verifier
ingest
New extraction + recent needs in the same area + edge status
Duplicate, conflict and staleness verdicts with reasons
Code finds candidates first; Gemini only judges ambiguous pairs
SOP Rule Extractor
api
SOP text (Vision OCR for scans)
Draft RuleSet with clause references
Rules stay draft until an officer approves them
Response Planner
planner
Engine output: assignments, diff, structured reason codes, evidence IDs
Plain-language explanation with citations
May only cite IDs it was given; unknown IDs are rejected
Plan Auditor
planner
Plan + state + approved rules
Pass or fail with violations
Code checks decide; Gemini only summarizes violations for humans
Interpreter output schema
{
  "report_type": "need | road_status | facility_status | resource_update | info | irrelevant",
  "language": "te",
  "summary_en": "12 people stranded on rooftops near the temple in C05",
  "location": { "community_id": "C05", "landmark_text": "near Hanuman temple", "confidence": 0.8 },
  "life_safety": true,
  "needs": [
    { "type": "rescue", "people": 12, "quantity": null, "unit": null, "deadline_hours": null }
  ],
  "vulnerable": { "elderly": 3, "disabled": 0, "pregnant": 1, "infants": 2 },
  "road_updates": [],
  "facility_updates": [],
  "confidence": 0.85
}
Need types: rescue, medical_time_critical, medical, water, food, shelter, fuel. The schema is passed to Gemini as structured output, so responses are always valid JSON of this shape.
How location is resolved
The gazetteer lists every community, landmark and road with English, Telugu and Hindi names and common spellings. The Interpreter picks from it; it never invents coordinates. A report with no match is pinned to "location unknown" and raised for verification, which is also how vague reports from blind-spot areas surface.
How duplicates are found
1. Candidates: same community and need type within a 3-hour window.
2. Similarity: Vertex AI text embeddings of summary_en; above a threshold counts as a likely duplicate.
3. Gemini judges only the borderline pairs.
4. Duplicates merge into one need; its confidence rises and all source reports stay attached as evidence.
Model choice and cost
• Use a fast, low-cost Gemini Flash-class model for extraction and verification, and the same or a Pro-class model for explanations if budget allows. Check current model names and regional availability on Vertex AI.
• Cache: key = hash(model + prompt version + input). Scripted demo reports always hit the cache; only new reports typed by a user call Gemini live.
• Rate limit: at most a handful of live Gemini calls per session per minute.
• Prompts are versioned in agents/prompts/, and each extraction stores the prompt version for traceability.
Planning engine
Pure Python in the planner service, with no AI inside, run on every debounced state change. It reads the session state, produces a new plan version, and must finish within about 5 seconds for the demo scenario.
state → score needs → road graph → isolation → cascades → allocation (CP-SAT)
      → routes → audit → diff vs previous plan → explain → write plan v(n+1)
1. Urgency scoring
Needs are ranked in tiers first, then by score within a tier. A lower tier never outranks a higher one.
Tier
Need types
Rule
1
rescue
Always first; acted on before verification completes
2
medical_time_critical (dialysis, oxygen, near-term pregnancy)
Hard deadline from SOP rules
3
medical, water, fuel for critical facilities
Score-ranked
4
food, shelter, other
Score-ranked
Within a tier, each factor is normalized to 0–1 and weighted by the approved SOP rules (defaults shown):
score = 0.30 · severity        (need type and stated condition)
      + 0.20 · vulnerability   (share of elderly, disabled, pregnant, infants)
      + 0.15 · isolation       (1 if isolated, rising as isolation_eta approaches)
      + 0.15 · people          (log-scaled number affected)
      + 0.10 · waiting_time    (time since first report)
      + 0.10 · deadline        (closeness to a deadline, if any)
Confidence never lowers a score; low confidence creates a verification task instead. This keeps poorly reported communities from being penalized for missing data.
2. Road graph and isolation
• NetworkX graph: nodes are junctions, communities, depots and facilities; edges carry travel_min, status, mode (road, water, air) and flood_eta from the hazard layer or gauge forecast.
• Isolated now: no path from any depot to the community using open edges (risky edges allowed only with approval).
• Isolation forecast: remove edges in order of flood_eta; the first time a community disconnects is its isolation_eta. With about 15 communities this is instant.
• An isolation alert fires when isolation_eta is within the warning window (default 6 hours), and its needs get the isolation factor and a pre-positioning suggestion.
3. Cascade detection
• Directed dependency graph from the dependencies table (road → delivery → facility function).
• Each facility function has a time to failure (e.g., H1 generator: diesel hours left). If every supply route to it is blocked, its failure time is that clock.
• Smallest fix: enumerate candidate interventions (each depot × each feasible route × each suitable vehicle, or a facility transfer), and pick the cheapest one that delivers before failure. Cost = travel time + vehicles diverted from other needs.
• The chain and the fix appear as one cascade alert with each link's evidence.
4. Allocation (OR-Tools CP-SAT)
One planning cycle assigns one trip per vehicle; the system replans as trips complete, so a full vehicle-routing model isn't needed.
Sets       N needs, K items, D depots, V vehicles
Variables  y[v,n] ∈ {0,1}        vehicle v serves need n this cycle
           x[d,v,n,k] ≥ 0 int    units of item k from depot d on vehicle v to need n
           u[n,k] ≥ 0 int        unmet demand

Constraints
  Σ_v,n x[d,v,n,k] ≤ stock[d,k]                        stock
  Σ_d,k x[d,v,n,k] ≤ capacity[v] · y[v,n]              vehicle capacity
  Σ_n y[v,n] ≤ 1                                       one trip per cycle
  y[v,n] = 0 if no allowed route for v's mode to n     access and mode (boat, truck, air)
  Σ_d,v x[d,v,n,k] + u[n,k] = demand[n,k]              demand balance
  Σ coverage[c] ≥ min_share · demand[c]  if reachable  minimum coverage per community
  tier-1 needs: some rescue unit assigned if any can reach

Objective  minimize
    Σ priority[n] · u[n,k]                 unmet need, weighted by tier and score
  + λ1 · Σ travel_min[v,n] · y[v,n]        travel time
  + λ2 · Σ lateness[n]                     past-deadline arrivals
  + λ3 · Σ changes vs approved plan        stability: don't reshuffle without reason
The stability term (λ3) matters for the demo: after the bridge closes, only the deliveries that used the bridge should change. Solver time limit: 5 seconds; return the best feasible solution found.
5. Routes
For each chosen assignment, Dijkstra on travel_min over edges allowed for that vehicle's mode. Risky edges are used only if no open route exists, and the assignment is then marked "needs approval."
6. Audit, diff and fallback
• Audit (code): stock not exceeded, capacities respected, no closed edges, risky edges flagged, every tier-1 need assigned or explained, SOP rules satisfied.
• On failure: the new plan is rejected, the previous valid plan stays live, and an alert explains why.
• Diff: compare with the last approved plan: added, removed, rerouted and resized assignments, each with a reason code (edge_closed, new_life_safety_need, stock_depleted, …). The Response Planner turns these codes into the explanation.
7. Baseline for benchmarking
baseline.py allocates first-come, first-served by report time using the same constraints and routes. Both run on the same event stream, so the comparison is fair.
Hazard packs and Earth Engine
A hazard pack is a folder with a config file and an Earth Engine recipe. The engine, agents and UI read the config; nothing else changes when a pack is added.
Pack config
# hazard_packs/flood/pack.yaml
id: flood
label: Flood
hazard_layer:
  recipe: ee_recipe.py
  output: gs://relief-radar-layers/flood/{scenario}/zones.geojson
  severity_levels: [flooded, high_risk, moderate]
road_impact:
  closed_if: { zone: flooded }
  risky_if:  { zone: high_risk, buffer_m: 100 }
need_types: [rescue, medical_time_critical, medical, water, food, shelter, fuel]
per_person_daily:            # starting defaults; SOP rules can override
  water_kits: 0.25
  food_kits: 0.2
transport_modes: [truck, boat]
dependency_templates:
  - road -> fuel_delivery -> hospital_generator
  - substation -> hospital_grid_power
  - road -> water_tanker -> shelter_water
ui:
  map_layers: [flood_zones, roads, communities, facilities, routes]
  icons: { boat: boat, truck: truck }
The landslide pack differs in hazard_layer, road_impact (slope-failure polygons close crossing edges; steep edges in high-susceptibility zones are risky), transport_modes: [truck, helicopter, foot], and its dependency templates.
Flood recipe (Earth Engine, Python API)
1. Area of interest: the chosen Hyderabad zone polygon.
2. Sentinel-1 GRD, VV polarization, same orbit direction, for a window before the event and a window just after peak flooding (October 2020).
3. Speckle smoothing (a focal median), then the change ratio of after vs before backscatter; a drop below a threshold marks new water.
4. Remove permanent water with the JRC Global Surface Water occurrence layer.
5. Mask steep terrain using slope from a DEM (flooding is unlikely on steep slopes).
6. Classify severity from the size of the drop and proximity to the main flooded area; vectorize into polygons.
7. Export GeoJSON to Cloud Storage; pipelines/seed intersects it with OSM road edges (Shapely) and writes edge flood_eta and status into the scenario template.
Threshold values need tuning against what's visibly flooded; record the final values in the README so results are reproducible. Confirm Sentinel-1 coverage for the dates first.
Landslide recipe
Slope from the DEM plus rainfall accumulation (CHIRPS or GPM IMERG) for the event window, combined into susceptibility classes; one planted slope-failure polygon on the valley road for the scenario.
Forecast scenarios
For the "before any report arrives" moment, precompute zone sets for two or three rainfall or river-level thresholds. The scenario's IMD alert event loads the matching set, which sets flood_eta on edges and triggers the isolation forecast.
Frontend
React + Vite + TypeScript, served by Firebase Hosting. All live data arrives through Firestore listeners; all actions go through /api.
Components
Component
Reads
Does
TopBar
session meta
Hazard pack, phase, scenario clock, active SOP, role switcher, reset
PriorityList
needs, communities
Tiered ranked list; tier badges; click opens evidence
LiveMap
hazard layer, edges, communities, facilities, plan routes
MapLibre layers; animates what changed after each replan
AlertsPanel
alerts
Isolation, cascade, blind spot, overlap, conflict, rumor
PlanPanel
latest plan
Assignments, unmet needs, explanation, approve / edit / reject
EvidenceDrawer
reports, needs, sop_rules
Source reports, timestamps, confidence, SOP clause, original language
Simulator
timeline, clock
Play, pause, step; close a road; add a report; vehicle down
GuidedTour
clock
Step cards that explain what to watch at each scenario stage
SopRules
sop_rules
Upload SOP, review drafted rules, approve
NgoView
plan (filtered by org)
Assignments and status buttons for an NGO
FieldReportForm
none
Text, voice note, photo and location upload
MetricsPanel
plans, evaluation results
RELIEF RADAR vs baseline at the end of the run
Map layers
Layer
Style
Hazard zones
Fill by severity
Roads
Grey open, amber dashed risky, red closed
Communities
Circle sized by population, colored by top tier; isolation ring when forecast
Facilities
Icons for hospital, shelter, depot, substation; warning badge in a cascade
Routes
Line per assignment; changed routes pulse after a replan
Base map: MapLibre with OpenStreetMap-based tiles from a free provider (check its terms and limits), to avoid Maps API charges.
Real-time state
• On load: Firebase anonymous sign-in → POST /api/sessions → subscribe to sessions/{sid}/* with onSnapshot.
• Listeners update a small store (e.g., Zustand); components re-render from it.
• Pending actions (an approval) show optimistic state until the planner's write confirms it.
Demo mode
• Landing screen: one button, "Start Hyderabad flood scenario," plus "Try landslide."
• Play advances the scenario clock (about 2 minutes for the full timeline); the GuidedTour card at each stage says what just happened and what to look at.
• Every synthetic item carries a "Simulated" label; the real Earth Engine layer is labeled with its source and date.
Auth and roles
• Firebase Authentication anonymous sign-in for judges (no sign-up wall); optional Google sign-in.
• The role switcher (coordinator, NGO-A, NGO-B, field) only changes the view; all three are available in demo mode so judges can see each.
• In a real deployment, roles become Firebase custom claims checked by the api service.
Deployment
Item
Choice
Region
asia-south1 (Mumbai) for Cloud Run, Firestore and Pub/Sub; Vertex AI in a region where the chosen Gemini model is available (check)
Containers
One Dockerfile per service; images in Artifact Registry
CI/CD
Cloud Build trigger on push to main: run tests, build images, deploy the three Cloud Run services, then firebase deploy for hosting and Firestore rules
Hosting
Firebase Hosting with a rewrite of /api/** to the api Cloud Run service
Identity
One service account per service with least privilege (Firestore user, Pub/Sub publisher, Vertex AI user, Storage object viewer); no API keys in code
Secrets
Secret Manager for anything non-IAM; .env.example in the repo, never real values
Scaling
min instances 0, max a small number per service; request timeout long enough for a replan
Setup scripts
infra/setup_pubsub.sh (topics, push subscriptions, BigQuery subscriptions), infra/setup_bigquery.sh (dataset and tables), pipelines/seed (scenario templates)
Testing
Level
What
Where
Unit
Scoring tiers, isolation forecast, cascade fix, allocator constraints, audit checks on tiny hand-made graphs
services/planner/tests
Schema
Every agent output validates against its schema; prompts tested on 10 sample reports
agents/tests
Scenario acceptance
Replay the full Hyderabad timeline in-process and assert each expected response from the Demo scenario tab
eval/run_scenario.py
Extraction accuracy
Compare extractions with the hidden labels of all 40 reports
eval/extraction_eval.py
Benchmark
Run baseline and RELIEF RADAR on the same events; output the metrics table for the deck
eval/metrics.py
Concurrency
Three sessions playing the scenario at once against the deployed app; check isolation and timings
Manual plus a small script
Fresh-browser check
Incognito, phone, another network, before every submission
Manual checklist
Observability and cost control
• Structured JSON logs with session_id and trace_id in every service; Error Reporting on.
• Uptime check on the hosting URL and /api/health, with email alerts.
• Budget alerts on the billing account at low thresholds.
• A small admin page (team only) shows active sessions, Gemini calls today and cache hit rate.
Runbook for the evaluation period
1. Freeze main after submission; deploy only critical fixes.
2. Check the uptime alerts and budget daily.
3. If Gemini quota or credits run low, switch the live path to cache-only mode with a config flag; the scripted demo keeps working.
4. If a service breaks, roll back to the previous Cloud Run revision (one command, no rebuild).
5. Keep the repo public and the video link accessible until results are announced.