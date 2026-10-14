Pitch deck content
How to use this tab
This is the text for each slide of the AI Builder Cup template, in the template's order. Paste it into the template's own slides without changing their titles, layout or branding, then download as .pptx as the template instructs.
• Keep every template slide, including optional ones; fill "(optional)" slides rather than deleting them.
• The instruction slide (slide 2, "Please download this template") is guidance for you; check the challenge rules on whether to keep it before deleting it.
• Items marked [after build] need real screenshots or measured numbers. Never fill them with made-up results.
• Slide numbers below follow the template PDF.
Slide 1: Team details
• Team name: [your team name]
• Team leader name: [leader's name]
• Problem Statement: Sustainability & Social Impact. During disasters, control rooms allocate scarce relief using fragmented, conflicting and outdated reports, so they miss communities about to be cut off, essential services about to fail, and needs nobody is covering.
Slide 3: Brief about the idea
RELIEF RADAR: an AI decision engine for disaster relief.
• Takes in every signal a control room receives: helpline calls, citizen reports, field teams, NGOs, news, social media, official alerts and satellite data.
• Gemini turns messy, multilingual reports and photos into verified, structured needs.
• Predicts which communities are about to be cut off and which services (hospitals, power, water) will fail next.
• Allocates limited supplies, vehicles and rescue teams under real constraints, following the authority's own SOPs.
• Replans in seconds when a road closes or a new emergency arrives, and explains every change.
• Works for any disaster through hazard packs: flood and landslide built, cyclone and earthquake next.
• AI recommends, humans decide: a coordinator approves every dispatch.
Slide 4: Opportunities
How different is it from existing ideas?
• Flood Hub and SACHET forecast and alert; RELIEF RADAR decides what to do after the alert.
• Ushahidi-style crisis maps collect reports; RELIEF RADAR verifies, deduplicates and prioritizes them.
• Resource registries list stock; RELIEF RADAR allocates it under real constraints.
• Most tools handle one hazard; RELIEF RADAR is hazard-agnostic.
How will it solve the problem?
• Turns hundreds of chaotic messages into one ranked, evidence-backed list of needs.
• Warns before communities are isolated, so supplies are pre-positioned.
• Detects chain reactions (blocked road → fuel delay → hospital power failure) and proposes the smallest fix.
• Gives NGOs and government one shared plan, ending duplicate deliveries and uncovered requests.
USP
• Predicts what fails next, not just what already happened.
• Live replanning with plain-language explanations and SOP citations.
• Life-safety first: rescue and time-critical medical needs are never delayed for verification.
• One engine for every disaster type.
Slide 5: List of features
1. Multi-source report understanding: helplines, texts, voice notes, photos, news and social posts in English, Telugu and Hindi become structured needs; duplicates, stale reports and conflicts are flagged.
2. Life-safety-first urgency ranking: rescue, time-critical medical needs (dialysis, oxygen, pregnancy) and vulnerable people come first, with evidence and confidence shown.
3. Isolation prediction: warns which communities will be cut off, and when.
4. Cascading failure detection: shows how one disruption threatens hospitals, power and water, and the smallest fix.
5. Constrained allocation and routing: supplies, vehicles, boats and rescue teams assigned without exceeding stock or capacity; unsafe routes need approval.
6. Live replanning: a road closure or new report updates the plan in seconds, with a before/after explanation.
7. Blind-spot detection: highly exposed areas with few reports are flagged for verification.
8. Multi-agency coordination: overlap and unassigned-request alerts; delivery confirmation.
9. SOP-driven rules: upload a district SOP; Gemini drafts rules, an officer approves them, every decision cites its clause.
10. Hazard packs: flood and landslide built; cyclone and earthquake on the roadmap.
Slide 6: Process flow diagram
Use the event-loop flow (redraw in the template's style):
1. Signals arrive (helplines, reports, NGOs, news, alerts, satellite)
2. Gemini extracts structured needs; Vision reads photos and scans
3. Evidence Verifier dedupes, checks freshness, flags conflicts
4. Planning engine: isolation, cascades, urgency under SOP rules, allocation and routes
5. Plan Auditor validates constraints
6. Coordinator approves, edits or rejects
7. Teams deliver and confirm; new events loop back to step 1
Side note on the slide: life-safety reports skip the wait and go straight to step 4 while verification runs in parallel.
Slide 7: Wireframes (optional, but fill it)
 Use the coordinator screen wireframe from the "Use cases and UI" tab: top bar (hazard, phase, clock, SOP), priorities, live map, alerts, plan with approve/edit/reject, event simulator. Add a small inset of the mobile field-report form if space allows.
Slide 8: Architecture diagram
Use the architecture diagram from the "Technical plan" tab, updated so the four required services are clearly labeled:
• Firebase Hosting serves the React app; Firebase Authentication handles role and demo logins.
• Firestore holds live incident state and plans; the dashboard updates through Firestore real-time listeners.
• Cloud Run runs the ingestion service, the planning engine (Python, OR-Tools) and the API.
• Gemini on Vertex AI (ADK agents) interprets reports, verifies evidence, extracts SOP rules, explains and audits plans.
• Supporting: Pub/Sub (event bus), BigQuery (record and audit log), Earth Engine (hazard zones), Cloud Vision (OCR), Cloud Storage (files).
Slide 9: Technologies
Area
Technology
AI
Gemini on Vertex AI, Agent Development Kit (ADK), Cloud Vision API
App and hosting
Firebase Hosting, Firebase Authentication, React, MapLibre
Real-time data
Firestore (live state, real-time listeners), Pub/Sub (event bus)
Compute
Cloud Run (ingestion, planning engine, API)
Geospatial
Google Earth Engine (Sentinel-1, elevation, rainfall), BigQuery GIS, OpenStreetMap
Analytics and audit
BigQuery, Cloud Storage
Optimization
Python, Google OR-Tools, NetworkX
Slide 10: Estimated implementation cost (optional, but fill it)
• Prototype: runs within Google Cloud's always-free limits and trial credits; Gemini calls are the main cost, controlled by caching scripted outputs and rate limits.
• Pilot for one district [estimate after build]: list the cost drivers rather than guessing totals: Gemini tokens per report, Cloud Run and Firestore usage, BigQuery storage, maps, and SMS or helpline integration.
• Fill in numbers from your actual billing report during the build, not from assumptions.
Slide 11: Snapshots of the prototype [after build]
Four screenshots from the deployed app:
1. Coordinator screen with priorities, map and alerts
2. Isolation and cascade warning on the map
3. Before/after plan after the bridge closure, with the explanation
4. Landslide pack loaded on the same engine
Slide 12: Performance report and benchmarking [after build]
Run the same planted scenario with a first-come, first-served baseline and with RELIEF RADAR. Report only measured results.
Metric
Baseline
RELIEF RADAR
Urgent needs unmet
[measured]
[measured]
Planted issues caught (of total)
[measured]
[measured]
Constraint violations
[measured]
[measured]
Time to replan after a disruption
[measured]
[measured]
Report extraction accuracy
n/a
[measured]
Recommendations citing the correct source
n/a
[measured]
Slide 13: Additional details and future development
• Cyclone and earthquake hazard packs
• Live helpline, SMS and WhatsApp integrations with state control rooms
• Real-time river gauge, rainfall and flood-forecast feeds as scenario triggers
• Offline-capable field app for areas with no signal
• Recovery module: shelters, damage tracking, rehabilitation
• Preparedness drills: simulate scenarios to test SOPs and stock before monsoon
• Privacy: data minimization and compliance with India's DPDP Act
Slide 14: Links
• GitHub public repository: [link]
• Demo video (3 minutes): [link] (note: the submission rules say 3 to 4 minutes; the template says 3, so aim for about 3:30 at most)
• Final product link: [deployed Firebase Hosting URL]
Slide 15 (blank template slide)
Use it for the planted-scenario story: the Hyderabad flood timeline (blind spot → isolation → cascade → NGO overlap → bridge closure and replan), or delete it only if the rules allow.