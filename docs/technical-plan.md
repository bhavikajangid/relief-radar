 # Technical plan

## Tech stack

Every required Google tool gets a real job; the decisions (ranking, allocation, routing, validation) stay in deterministic Python.

| Layer | Tool | Job in RELIEF RADAR |
| --- | --- | --- |
| Hazard data | Google Earth Engine (Python API) | Builds each hazard pack's affected-area layer (flood extent, slope, rainfall) and exports it |
| Images and documents | Cloud Vision API | OCR on photographed request sheets, handwritten lists and SOP scans |
| Language and reasoning | Gemini on Vertex AI + Agent Development Kit (ADK) | Agents: Incident Interpreter, Evidence Verifier, SOP Rule Extractor, Response Planner, Plan Auditor |
| Event stream | Pub/Sub | Topics for new reports, road changes, NGO updates and plan updates; drives live replanning |
| System of record | BigQuery (+ BigQuery GIS) | All reports, needs, inventory, events, plans and audit logs; spatial joins of communities, roads and hazard zones |
| Live state | Firestore (via Firebase SDK, required) | The current incident state and plan the dashboard listens to in real time; BigQuery isn't built for instant reads |
| Services | Cloud Run | Ingestion, planning engine, API and the web app, each as a container |
| Files | Cloud Storage | Uploaded photos, SOP PDFs, exported Earth Engine layers |
| Planning engine | Python: NetworkX, OR-Tools, Shapely/GeoPandas | Road graph and isolation, cascade graph, constrained allocation and routing, constraint checks |
| Frontend | React + MapLibre, served by Firebase Hosting; Firebase Authentication for roles and a one-click demo login | Map, priority list, evidence panel, plan before/after, approval buttons, event simulator |

Check current quotas, Earth Engine project registration and billing early, since setup issues cost hackathon hours.

**Required by the submission:** the brief description must explain how we use Firebase, Firestore, Cloud Run and Gemini. Each has a visible job: Firebase hosts the app and handles logins, Firestore carries the live state the dashboard listens to, Cloud Run runs the services, and Gemini powers every agent.

## Submission and evaluation readiness

Judges will use the deployed app on their own, possibly several at once, for the whole evaluation period, with no retries if anything is down.

| Requirement | How we meet it |
| --- | --- |
| Works without us explaining it | Guided demo mode: a "Start Hyderabad flood scenario" button replays the event timeline, with a short on-screen guide at each step |
| Several judges at once | Each session gets its own sandbox copy of the scenario in Firestore (keyed by session ID); a "Reset scenario" button restores it |
| No login friction | Firebase Authentication anonymous or one-click demo login, with role switcher (coordinator, NGO, field) |
| Gemini cost and rate limits | Cached Gemini outputs for scripted reports; live Gemini only for reports a judge types in, with per-session rate limits |
| No fragile live dependencies | Precomputed Earth Engine layers; no live external APIs on the demo path |
| Stays up for weeks | Cloud Run with min instances 0 (cold starts acceptable); budget alerts; uptime check; confirm trial credits last through evaluation |
| Works in any browser | Test in a fresh incognito window, on a phone, and from another network before submitting |
| Video matches the product | Record the 3 to 4 minute video from the deployed URL, not localhost |
| Repo is reviewable | README with architecture, how each Google service is used, setup steps, scenario description; synthetic data labeled; no secrets committed |

## Architecture

An event-driven loop: every input becomes a Pub/Sub event, agents turn it into structured facts, the engine replans, and the dashboard shows the new plan for approval.

&#91;embedded content: architecture · inputs, event bus, agents, engine, state, dashboard\]

Agents and engine exchange structured needs and draft plans; approvals from the dashboard go back onto Pub/Sub as events, so every decision is logged and can trigger a replan.

## How one event flows

A bridge-closure report should go from message to approved new plan in under a minute (a target to measure); this is the loop the demo shows.

1. **Arrives.** A field photo and a Telugu text say a bridge is under water. The simulator or upload form publishes it to the `reports` topic.
2. **Read.** Vision OCR reads any text in the photo. The Incident Interpreter (Gemini) returns structured JSON: location, need type (here: road status), severity, time, source. Invalid JSON is rejected and retried.
3. **Checked.** The Evidence Verifier compares it with earlier reports and the hazard layer: duplicate, conflicting or stale? Conflicts are marked "needs verification", never silently resolved.
4. **State updated.** The road edge is set to closed or risky in Firestore, and the event is written to BigQuery. A message goes to the `state-changes` topic.
5. **Replanned.** The planning engine recomputes isolation (graph connectivity to depots), cascades (dependency graph), urgency scores under the approved SOP rules, and allocation and routes (OR-Tools).
6. **Validated.** The Plan Auditor checks stock, capacity, blocked or risky routes and SOP rules, in code first, then Gemini summarizes any issue.
7. **Explained.** The Response Planner writes the before/after explanation, citing reports and SOP clauses.
8. **Approved.** The dashboard shows the new plan; the coordinator approves, edits or rejects each dispatch. The decision is logged.

## Information sources

RELIEF RADAR takes in every source a control room hears from, converts each into the same report format, and weighs it by how trustworthy and how fresh it is. Helplines and field reports drive life-saving decisions; depot, NGO and facility data drive inventory and resources.

### Sources and what each one is for

| Source | What it gives us | Freshness | Trust | Main use |
| --- | --- | --- | --- | --- |
| Disaster helplines (112, state and district control-room numbers) | Call logs and transcripts: people trapped, medical emergencies, missing persons | Minutes | High | Life safety: rescue and medical requests |
| Citizen reports (app form, WhatsApp, SMS, voice notes) | Needs, locations, photos, road conditions | Minutes | Medium until corroborated | Life safety and needs |
| Field teams and volunteers | Verified needs, road status, delivery confirmations | Minutes | High | Verification, road status, delivery confirmation |
| NGO and relief-team updates | Assignments, stock carried, deliveries made | Minutes to hours | High | Coordination, inventory |
| Hospitals and shelters | Beds, oxygen, fuel hours left, occupancy | Hours | High | Cascade detection, medical needs |
| Depots and inventory systems (district stores, IDRN-style registries) | Stock by item and location, vehicles | Hours | High | Inventory and allocation |
| Official alerts (IMD, CWC gauges, state authorities) | Rainfall warnings, river levels, evacuation orders | Hours | Highest | Prediction, scenario triggers |
| Satellite (Earth Engine) | Affected-area extent | Days | High for extent | Baseline and periodic ground truth |
| News | Incidents, road closures, relief camp locations | Hours | Medium | Early signals, filling gaps |
| Social media (public posts) | First reports, photos, pleas for help | Minutes | Low until corroborated | Early signals, blind-spot detection |
| Utility outages (power, telecom) | Where power and signal are down | Hours | High | Cascades, explaining blind spots |

Check the exact helpline numbers and official data feeds for your state; they differ by region.

### Trust rules

1. **Life-safety reports are never dismissed for being unverified.** A report of someone trapped or a medical emergency goes to the top immediately, and verification happens in parallel with the response, not before it.
2. **Everything else needs corroboration to rise.** Two independent sources, or one high-trust source, upgrade a report from "unverified" to "confirmed."
3. **Official and field data override crowd data** when they conflict, but the conflict stays visible with timestamps for the coordinator.
4. **Duplicates merge, they don't multiply.** Ten posts about the same flooded street count as one need with higher confidence, not ten needs.
5. **Silence is a signal.** An exposed area with few reports from any source is flagged as a blind spot.
6. **Misinformation is labeled, not deleted.** Contradicted rumors stay in the log marked as false, so they don't resurface.

### How the sources support the three goals

| Goal | Sources that matter most | What RELIEF RADAR does |
| --- | --- | --- |
| Saving lives | Helplines, citizen reports, field teams, hospitals | Puts rescue and time-critical medical needs first; routes boats and rescue teams; tracks deadlines like dialysis and oxygen |
| Inventory and resources | Depots, NGO stock, hospitals and shelters, delivery confirmations | Live stock levels; forecasts when an item runs out at current use; flags shortages early; matches surplus to need |
| Coordinating help | NGO updates, field teams, news | One shared plan; overlap and unassigned-request alerts; confirms who received help |

### In the prototype

Every source goes through its own small adapter into the same `reports` table, so adding a real source later means writing one adapter.

| Source | Prototype approach |
| --- | --- |
| Helpline calls | Synthetic call transcripts (English, Telugu, Hindi) |
| Citizen reports | Upload form plus synthetic WhatsApp-style messages and photos |
| Field teams and NGOs | Simple status form in the NGO view |
| Depots, hospitals, shelters | Generated CSVs, updated by simulator events |
| News | Synthetic headlines; optionally real coverage from the GDELT news dataset in BigQuery public datasets (check availability) |
| Social media | Synthetic posts; live platform APIs are paid and restricted, so roadmap only |
| Official alerts and satellite | Real historical rainfall and flood extent; a scripted forecast threshold event |

## Real-time data strategy

Satellites alone are not real time, so RELIEF RADAR layers sources by speed: satellite data sets the baseline and checks it, while sensors and reports keep the picture current minute by minute.

| Phase | What happens | Sources |
| --- | --- | --- |
| Before the disaster | Earth Engine builds hazard-prone zones and pre-computed scenarios ("if rainfall reaches X mm or the river reaches Y m, these areas flood") with exposure of communities, hospitals and roads | Historical satellite data, elevation, rainfall records, population |
| Alert | A forecast or gauge crosses a threshold; the matching scenario loads instantly; isolation warnings and pre-positioning plans appear before any report arrives | IMD warnings, CWC gauges, flood forecasts |
| During, between satellite passes | Reports, helplines and sensors update road status and needs continuously; every update is a Pub/Sub event that can trigger a replan | Helplines, citizen and field reports, NGO updates, news, social media, rainfall nowcasts |
| During, on a satellite pass | An automated pipeline recomputes the actual extent and publishes it as an event; conflicts with reports are flagged | Sentinel-1 via Earth Engine |
| After | Damage, water persistence and unresolved needs guide recovery and shelters | Satellite comparisons, delivery confirmations, field surveys |

Satellite revisit and processing delays vary by region and season; check current Sentinel-1 coverage before promising a refresh rate.

**Demo moment:** a scripted forecast threshold is crossed, the predicted flood zone loads, and isolation warnings appear before any report arrives. This shows Earth Engine is useful in real time, not just as a historical replay.

**Production note:** Earth Engine's free noncommercial terms may not cover an operational government deployment; a real rollout needs to confirm licensing.

## Hazard packs: data for each disaster

Each disaster is a pack: one Earth Engine recipe that outputs hazard zones, plus a config file. The engine only ever sees the output: hazard polygons, road-edge status, need templates and dependencies.

### What every pack produces

1. **Hazard zones:** polygons with a severity level (e.g., flooded, high risk, moderate).
2. **Road impact:** OSM road edges intersected with the zones, giving each edge open, risky or closed.
3. **Exposure:** communities and facilities inside or near zones, with population from WorldPop or GHSL.
4. **Config (YAML):** need types, typical quantities per person, transport modes, dependency templates and default rules.

### Per-disaster recipe

| Pack | Earth Engine recipe | Road rule | Needs and transport | Status |
| --- | --- | --- | --- | --- |
| Flood | Sentinel-1 GRD, before vs after the event (VV backscatter drop = water), minus JRC permanent water, masked by low slope from the DEM | Edge inside flood polygon = closed; within a buffer = risky | Rescue, water, medicine, diesel; trucks and boats | Build |
| Landslide | Slope from SRTM DEM + recent rainfall from CHIRPS or GPM IMERG = susceptibility zones; one planted slope-failure polygon for the event | Edge crossing a failure = closed; steep edges in high-susceptibility zones = risky | Rescue, food, medicine; air drops and foot teams when no road remains | Build |
| Cyclone | Track from IMD or IBTrACS, wind buffer around it, low-elevation coastal zone for surge | Edges in surge zone = risky before landfall, closed after | Evacuation, shelter, power; buses and trucks | Roadmap |
| Earthquake | USGS ShakeMap intensity zones | Bridges and edges in high intensity = risky until inspected | Search and rescue, medical, shelter | Roadmap |

Dataset names are from general knowledge; confirm availability, date coverage and licenses before building.

### Precompute, don't compute live

Run each Earth Engine recipe ahead of time, export the zones to Cloud Storage and BigQuery, and load them in the demo. Live Earth Engine calls on stage are slow and can fail. The "live" part of the demo is the event stream, not the satellite processing.

### Adding a new disaster

Write one Earth Engine recipe and one YAML config. No changes to the engine, the agents or the dashboard. This is the platform claim, and the demo proves it by switching from flood to landslide.

## Data model

Twelve BigQuery tables hold the shared model; Firestore mirrors only the current state and plan. Freeze these fields in the first hours, because every module depends on them.

| Table | Key fields |
| --- | --- |
| `hazard_zones` | zone\_id, hazard\_type, severity, geometry, source, as\_of |
| `communities` | community\_id, name, geometry, population, vulnerable\_counts (elderly, disabled, pregnant, infants) |
| `facilities` | facility\_id, type (hospital, shelter, depot, power, water), capacity, occupancy, fuel\_hours\_left |
| `road_edges` | edge\_id, from\_node, to\_node, length\_km, travel\_min, status (open, risky, closed), status\_source, updated\_at |
| `reports` | report\_id, raw\_text, media\_uri, language, received\_at, channel, source\_type |
| `needs` | need\_id, report\_id, community\_id, need\_type (rescue, medical, water, food, shelter, fuel), people, quantity, deadline, confidence, verification\_status |
| `resources` | resource\_id, depot\_id, item, quantity, unit |
| `vehicles` | vehicle\_id, type (truck, boat, helicopter, team), capacity, location, owner\_org, status |
| `dependencies` | from\_entity, to\_entity, type (fuel, power, water, access), time\_to\_failure\_hours |
| `sop_rules` | rule\_id, area, rule\_json, sop\_clause\_ref, approved\_by, approved\_at |
| `events` | event\_id, type, payload, created\_at, source |
| `plans` | plan\_id, version, assignments\_json, explanation, unmet\_needs, created\_at, approved\_by, decision |

## 48-hour build plan

The first working loop (report in, plan out, on the map) should exist by hour 20; everything after that deepens it.

| Hours | Backend (Python, Pub/Sub, BigQuery) | ML (Gemini, ADK, Vision, Earth Engine) | Frontend (React, map) |
| --- | --- | --- | --- |
| 0–6 | Freeze schema; create BigQuery tables, Pub/Sub topics, Cloud Run skeletons | Flood recipe in Earth Engine; export zones; write 40 synthetic reports with planted issues | Map shell with communities, roads, facilities from static files |
| 6–14 | Road graph, isolation check, urgency scoring with default rules | Incident Interpreter and Evidence Verifier with JSON validation; Vision OCR | Priority list and evidence panel |
| 14–20 | OR-Tools allocation and routing; constraint checks | Response Planner explanations; connect agents to Pub/Sub | Plan view: routes, vehicles, remaining stock. **First end-to-end loop** |
| 20–28 | Event-driven replanning; cascade graph; NGO overlap and gaps | SOP Rule Extractor and approval flow; Plan Auditor | Event simulator buttons; before/after comparison; approve and reject |
| 28–36 | Rescue, vulnerability, medical deadline and route-safety rules | Landslide recipe and pack config | Hazard-pack switch; isolation and cascade visuals |
| 36–42 | Baseline run vs RELIEF RADAR; record metrics | Extraction accuracy test on labeled reports | Metrics panel; polish |
| 42–48 | Deploy to Cloud Run; freeze | Backup demo video | Rehearse the script three times |

## Demo reliability rules

The demo must work even if Wi-Fi, quotas or an API fail on stage.

- Precompute all Earth Engine layers; never call it live during the demo.
- Script the event timeline (T+0 to the bridge closure) as a replayable file the simulator publishes to Pub/Sub.
- Cache Gemini outputs for the scripted reports, with a switch between live and cached mode.
- Keep the planning engine fast: about 15 communities and 5 vehicles solve in seconds with OR-Tools.
- Label every synthetic item as simulated on screen.
- Record a backup video of the full demo by hour 46.
