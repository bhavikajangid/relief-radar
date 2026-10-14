Demo scenario: Hyderabad flood replay
Overview
A 10-hour simulated flood response on the real October 2020 Hyderabad flood extent, with 15 communities, 3 depots, 2 hospitals, 3 shelters, 8 vehicles and teams, 2 NGOs and 40 reports. Every number here is synthetic and labeled as such in the app; it is both the demo script and the test set.
Pick one low-lying zone along the Musi that flooded in 2020 and map C01–C15 to real localities once the Earth Engine layer is ready. Until then, build against the codes below.
Planted issues the system must catch
#
Planted issue
Where
Feature tested
1
Rooftop rescue, reported 6 times
C05
Dedup, life-safety priority, rescue routing
2
Blind spot: heavily flooded, only 1 report
C03
Blind-spot detection
3
Community about to be isolated
C07
Isolation prediction, pre-positioning
4
Hospital fuel cascade
H1 via road E12
Cascade detection, smallest fix
5
Dialysis patients with a deadline
C09
Medical deadline rule
6
NGO overlap and an unassigned request
S2, C12
Coordination
7
Conflicting reports on a bridge
Bridge B4
Evidence Verifier, risky-route rule
8
Bridge closure forcing a replan
Bridge B4
Live replanning
9
Stale report
C10
Freshness handling
10
Rumor contradicted by an official source
Dam breach claim
Misinformation labeling
Entities
All values are synthetic starting state at T+0. Stock and vehicle numbers are deliberately tight, so the allocator has to make trade-offs.
Communities
ID
Population
Vulnerable (elderly / disabled / pregnant / infants)
Flood exposure
Road access
Scenario role
C01
2,400
180 / 40 / 18 / 60
Low
E1, E2
Background
C02
3,100
240 / 55 / 22 / 85
Medium
E2, E3
Food request
C03
1,800
210 / 45 / 15 / 50
High
E4
Blind spot (1 report)
C04
2,700
190 / 40 / 20 / 70
Medium
E3, E5
Water request
C05
1,500
160 / 35 / 12 / 40
High
E6
Rooftop rescue (12 people)
C06
2,200
150 / 30 / 16 / 55
Low
E5, E8
Background
C07
1,200
140 / 25 / 10 / 35
High
E7 only
Will be isolated
C08
3,400
260 / 60 / 25 / 90
Medium
E8, E9
Medical kits request
C09
1,900
230 / 50 / 14 / 45
Medium
E9, E10
3 dialysis patients
C10
2,000
170 / 35 / 15 / 50
Low
E10, E11
Stale report
C11
2,600
200 / 45 / 19 / 65
Medium
B4 bridge, E13
Affected by bridge closure
C12
1,400
120 / 25 / 11 / 35
Medium
E11
Unassigned request
C13
2,900
210 / 50 / 21 / 75
Low
E13, E14
Background
C14
1,700
130 / 30 / 12 / 40
Low
E14
Background
C15
2,100
160 / 35 / 16 / 55
Medium
E12, E14
Near H1 fuel route
Facilities
ID
Type
Capacity
State at T+0
Scenario role
H1
Hospital
120 beds
Grid power out; generator diesel for 10 hours
Fuel cascade
H2
Hospital (dialysis unit)
200 beds
Normal
Receives C09 patients
S1
Shelter (school)
400 people
210 occupied
Background
S2
Shelter (community hall)
300 people
260 occupied; water low
NGO overlap
S3
Shelter (school)
350 people
120 occupied
Reached via bridge B4
P1
Substation
n/a
At risk of flooding
Power dependency for H1
Depots and stock
Depot
Water kits
Food kits
Medical kits
Diesel (litres)
D1
400
300
60
1,000
D2
250
200
40
500
D3 (higher ground)
150
150
30
0
Vehicles and teams
ID
Type
Capacity
Base
Owner
V1
Truck
150 kits
D1
District
V2
Truck
150 kits
D2
District
V3
Diesel tanker
800 litres
D1
District
V4
Boat
20 people or 40 kits
D1
District
V5
Boat
20 people or 40 kits
D2
District
R1
Rescue team (NDRF-style)
1 rescue at a time
D1
District
N1
Van
80 kits
D2
NGO-A
N2
Van
80 kits
D3
NGO-B
Dependencies
From
To
Type
Time to failure
Road E12
Diesel delivery to H1
Access
Immediate when E12 floods
Diesel delivery
H1 generator
Fuel
10 hours from T+0
P1 substation
H1 grid power
Power
Already failed
Road E7
All access to C07
Access
Immediate when E7 floods
Bridge B4
Access to C11 and S3
Access
Immediate on closure
Event timeline
The simulator replays these events in order (compressed to about 2 minutes in demo mode). Each row's expected response is an acceptance test: if the system does not do it, the build is not done.
Time
Event
Source
Expected system response
T+0:00
IMD heavy-rain warning; forecast crosses the flood threshold
Official alert
Loads the precomputed flood scenario; shows exposed communities and facilities
T+0:05
Flood extent layer loaded (Oct 2020 Sentinel-1)
Earth Engine
Marks E4, E6 and E12 as at risk; C03, C05, C07 shown as high exposure
T+0:15
First wave of 18 reports arrives
Helpline, WhatsApp, SMS
Extracts needs; merges the 6 C05 rooftop reports into one rescue need
T+0:20
Ranking published
Engine
C05 rescue ranked #1, above all supplies; R1 and boat V4 assigned
T+0:30
Blind-spot check
Engine
C03 flagged: high exposure, 1 report; verification team suggested
T+1:00
C09 reports 3 dialysis patients, last session 2 days ago
Helpline
Hard deadline set (transfer to H2 by T+6); V5 or a truck assigned
T+1:30
Stale C10 report (12 hours old) surfaces
WhatsApp forward
Labeled stale; kept but down-weighted until confirmed
T+2:00
River gauge rising; E7 will flood in about 3 hours
Official gauge
C07 isolation warning; pre-positioning plan: water, food, medical kits to C07 before T+5
T+3:00
Social post: "Dam breached, evacuate now"
Social media
Labeled unverified; contradicted at T+3:20 by official statement; marked false, not deleted
T+4:00
E12 floods
Field team
Cascade alert: H1 diesel route blocked, generator has 6 hours left; proposes tanker V3 via alternate route or from D2
T+5:00
E7 floods
Field team
C07 now isolated; confirms pre-positioned supplies arrived; boats only from now on
T+6:00
NGO-A and NGO-B both dispatch water to S2; C12 request unassigned
NGO updates
Overlap alert; reassigns N2 to C12
T+7:00
Two reports about bridge B4: "open" and "under water"
Citizen reports
Conflict flagged; B4 set to risky; routes via B4 need approval; verification requested
T+8:00
Authority closes bridge B4
Official
Replans: C11 and S3 deliveries rerouted via E13; before/after diff with reasons; coordinator approves
T+9:00
Deliveries confirmed by teams
NGO and field
Needs closed; remaining unmet needs listed
T+10:00
End of scenario
Engine
Metrics summary vs baseline
Planted reports
Forty reports, written by the team, each with a hidden label (true location, need, urgency, and whether it is a duplicate, stale, conflicting or false). The labels are what we score extraction accuracy against.
Topic
Count
Notes
C05 rooftop rescue
6
Same event, different wording and channels; must merge into 1
C03 flooding
1
The blind spot; vague location wording
C09 dialysis patients
2
One helpline transcript in Telugu, one family WhatsApp message
C10 needs
1
Timestamp 12 hours old
Dam breach rumor
2
One social post, one forward; both false
Bridge B4 status
2
Contradict each other
C07 access and needs
3
Includes one photo of the rising water on E7
S2 shelter water
3
Includes a photographed handwritten supply list (Vision OCR)
C12 food request
1
Left unassigned by both NGOs
C02 food
3
Routine
C04 water
3
Routine
C08 medical kits
3
Routine
H1 fuel
2
Hospital staff call and an NGO note
Road status E12, E7
3
Field team updates
C11 needs
2
Affected later by bridge closure
Background noise
3
Off-topic or general messages, to test filtering
Total
40

Languages: 18 English, 14 Telugu, 8 Hindi. Channels: 10 helpline transcripts, 16 WhatsApp or SMS texts, 6 with photos, 4 social posts, 4 news snippets.
Each report file carries: report_id, channel, language, received_at, raw_text, optional media_uri, and a separate label object kept out of what Gemini sees.
Pass criteria
The build is demo-ready when all of these hold on the deployed app, from a fresh browser session:
[ ] All 10 planted issues are caught at the right time in the timeline
[ ] C05 rescue is ranked first at T+0:20 and is never delayed by verification
[ ] The 6 C05 reports become 1 need; the dam rumor is labeled false
[ ] C07 supplies are delivered before E7 floods at T+5
[ ] H1 gets diesel before its generator runs out
[ ] C09 patients reach H2 before the T+6 deadline
[ ] Zero constraint violations: stock, vehicle capacity, closed roads
[ ] Replan after the bridge closure completes and shows a before/after explanation
[ ] Baseline (first-come, first-served) run is recorded with the same events, for the benchmarking slide
[ ] Two judges running the scenario at once don't affect each other
Landslide switch scenario
A short second scenario to prove the engine is generic (about 20 seconds in the video).
• Setting: a hill district valley with 5 communities (L01–L05), one road in, one depot outside the valley, 1 helicopter (20 kits per trip), 2 foot teams, 1 truck.
• Hazard layer: slope plus heavy rainfall zones from Earth Engine; one planted slope failure on the only road.
• Events: heavy rain warning → slope failure blocks the road → 2 trapped-family reports from L03 → steep road segments in L04 marked risky.
• Expected response: all 5 communities flagged isolated; truck plan replaced by helicopter and foot teams; L03 rescue ranked first; risky L04 route requires approval.
• Pass: the same screens, rules and engine handle it with only a new pack config and hazard layer.