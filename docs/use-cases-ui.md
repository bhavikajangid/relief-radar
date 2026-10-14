Use cases and UI
How the UI adapts
The interface is one app whose layers, panels and actions are driven by five inputs, so the same screens serve every disaster, phase and user.
Driven by
What changes on screen
Example
Hazard pack
Map layers, need types, transport icons, dependency templates
Flood shows water extent and boats; landslide shows slope zones and air drops
Phase
Which panel leads and which actions are offered
Before: pre-positioning suggestions. During: dispatch plan. After: unresolved needs and shelters
Role
Which screen opens and what the user can approve or edit
Coordinator approves plans; NGO sees only its assignments; field user sees a report form
Live events
The map, rankings and plan update as Pub/Sub events arrive; changes are highlighted
A bridge closes: the road turns red, affected deliveries flash, a before/after diff appears
Approved SOP
Priority labels, thresholds and approval steps
A district SOP puts dialysis above food; the ranking and the reasons cite that clause
The frontend reads all of this from configuration and live state, not hard-coded screens. Adding a disaster or a district's SOP changes what users see without new UI code.
Main control-room screen
One screen carries the demo: priorities on the left, the live map in the middle, alerts on the right, and the plan to approve at the bottom.
The top bar switches hazard pack and phase. Clicking any priority, alert or route opens an evidence drawer with source reports, timestamps, confidence and the SOP clause behind it. The simulator drives the live replanning moment; all values shown are illustrative.
Screens by role
Three roles, three views of the same live state. For the hackathon, build the coordinator view fully and a simple NGO and field view.
Role
Screen
Can do
Control-room coordinator
Full map, priorities, plan, alerts, event simulator, SOP rules
Approve, edit or reject dispatches; approve SOP rules; mark reports verified
NGO or relief team
Its assignments, routes, overlap warnings, nearby unassigned requests
Accept a task, post status ("en route", "delivered to 120 people"), flag a problem
Field volunteer or citizen
Mobile report form (text, voice, photo, location) in their language
Send a request or road update; see that it was received
Build order: coordinator view first (it carries the demo), then the NGO status form, then the field report form.
Use cases by disaster
Each use case is a situation a control room actually faces; the same engine handles all of them, and the UI shows what matters for that hazard.
Urban flood (build)
Situation: heavy rain floods low-lying wards; roads waterlog within hours; hospitals, substations and shelters are at risk.
Problem
What the UI shows
How RELIEF RADAR helps
Hospital backup power at risk
Cascade chain: flooded road → fuel truck blocked → hospital generator hours left
Finds the smallest fix: alternate route or a nearer depot, before power fails
Hundreds of messages, many duplicates
Ranked needs with evidence and confidence
Dedupes and ranks in seconds; flags conflicts for verification
People stranded on rooftops
Rescue requests pinned above supplies
Routes boats and rescue teams first, under SOP rules
Roads changing hour by hour
Live road status; plan diff on every change
Replans automatically and explains what moved and why
Rural or river flood (build, same pack)
Situation: a river overflows; villages become islands for days.
Problem
What the UI shows
How RELIEF RADAR helps
Villages about to be cut off
Isolation warnings with estimated time
Pre-positions water, food and medicine before access is lost
Only boats can reach some places
Transport mode per route
Assigns boats where trucks can't go
Remote villages send few reports
Blind-spot markers
Sends verification teams to high-exposure, low-report areas
Landslide (build)
Situation: heavy rain in hills; one slope failure blocks the only road into a valley.
Problem
What the UI shows
How RELIEF RADAR helps
Whole valley isolated
Communities with no road route left
Recommends air drops or foot teams; lists what to send first
More slopes may fail
Risky routes along steep, saturated slopes
Keeps rescue teams off unsafe routes unless a human approves
Trapped people
Rescue requests with locations
Prioritizes search and rescue teams over supplies
Cyclone (roadmap)
Situation: a cyclone is forecast to make landfall in 48 hours.
Problem
What the UI shows
How RELIEF RADAR helps
Who to evacuate first
Surge-zone communities ranked by vulnerability
Plans evacuation order and buses, before landfall
Shelters may overflow
Shelter capacity vs expected arrivals
Balances arrivals across shelters and flags shortfalls
Post-landfall power loss
Cascade: power → water pumps → shelters
Pre-positions diesel and water where pumps will fail
Earthquake (roadmap)
Situation: a sudden quake damages buildings, bridges and hospitals with no warning.
Problem
What the UI shows
How RELIEF RADAR helps
Unknown damage extent
Intensity zones and early reports
Ranks areas for search and rescue in the first hours
Bridges may be unsafe
Bridges marked risky until inspected
Routes around them and flags inspections needed
Hospitals damaged or overloaded
Hospital capacity and access
Redirects patients and medical supplies
Cross-cutting use cases
These apply to every disaster and are worth one line each in the pitch.
Use case
Who uses it
How RELIEF RADAR helps
Multi-agency coordination
Control room, NGOs
One shared plan; overlap and unassigned-request alerts
SOP compliance
District officials
Every decision cites the SOP clause and who approved it
Delivery confirmation
NGOs, coordinator
"Delivered to whom" status closes the loop and reopens unmet needs
Preparedness drills
Disaster authorities
Run a simulated scenario and test the SOP and stock levels before monsoon
After-action review
Authorities, auditors
The event and decision log in BigQuery shows what happened, when and why