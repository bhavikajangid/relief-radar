# RELIEF RADAR: Idea Brief

Oct 11, 2026 ·

## In one line

RELIEF RADAR is a hazard-agnostic decision engine for disaster relief: plug in any disaster, and it predicts who will be cut off, what will fail next, and how to allocate limited resources, with a plan that updates as conditions change.

**Theme:** Sustainability & Social Impact (resilience, supporting communities, informed decision-making).

**Positioning line:** Others show where the disaster is. We show who's about to be cut off, what's about to fail, and what to send where.

**Principle:** AI recommends, humans decide. A coordinator approves every dispatch.

## The problem

Relief rarely fails for lack of supplies. It fails because control rooms decide on scattered, conflicting and outdated information, and the plan doesn't change when conditions do.

**Problem statement:** In a disaster, control rooms allocate scarce relief using fragmented reports, so they miss communities about to be cut off, services about to fail, and needs nobody is covering.

Three specific failures we solve:

1. **Hidden chain reactions.** A blocked road stops a fuel delivery, and a hospital loses backup power hours later. Nobody saw the link.
2. **Invisible communities.** Areas that lose signal or have no volunteers send few reports, so they get less help even when they are worst hit.
3. **Uncoordinated responders.** NGOs and government teams duplicate deliveries to visible places while other requests go unassigned.

Today, most control-room coordination runs on phone calls, spreadsheets and WhatsApp groups.

## Who it's for

The primary user is the district or city emergency control-room coordinator. Every screen answers their question: what should I do next, and why?

| User | Role | What they get |
| --- | --- | --- |
| Control-room coordinator (primary) | Approves dispatches and priorities | Ranked needs, allocation plan, isolation and cascade warnings, replanning |
| NGOs and relief teams | Deliver supplies and report status | Clear assignments, overlap warnings, a way to post updates |
| Field volunteers and citizens | Send requests | A simple channel (text, voice, photo) in their language |

## Generic platform

Every disaster reduces to the same eight concepts, so the features run on those concepts, not on a specific hazard. Adding a disaster means writing a hazard pack, not building new features.

&#91;embedded content: generic platform · hazard packs, shared model, core engine\]

Each pack fills the shared model; the engine produces a plan the coordinator approves, and every new event re-runs it.

| Hazard pack | Affected-area data | Typical needs | Transport modes | Signature insight |
| --- | --- | --- | --- | --- |
| Flood (build) | Sentinel-1 flood extent, rainfall, elevation | Drinking water, boats, medicine, diesel | Trucks, boats | Cascading failure of city services |
| Landslide (build) | Slope, rainfall, susceptibility | Rescue teams, food, medicine | Trucks, air drops, foot teams | One blockage isolates a whole valley |
| Cyclone (roadmap) | Track forecast, surge zones | Evacuation, shelter, power | Buses, trucks | Pre-positioning before landfall |
| Earthquake (roadmap) | Damage zones | Search and rescue, medical, shelter | Trucks, air | Damaged roads and hospitals together |

## Features

We build six core features and two supporting ones; everything else is roadmap. All of them run on the generic engine, so they work for every hazard pack.

| # | Feature | What it does | Tier | Buildability |
| --- | --- | --- | --- | --- |
| 1 | Report understanding | Turns texts, voice transcripts and photographed request sheets (English, Telugu, Hindi) into structured needs; flags duplicates, stale reports and conflicts | Core | High |
| 2 | Urgency ranking with evidence | Transparent score per community with sources, freshness and confidence; confirmed life-threatening needs always come first | Core | High |
| 3 | Isolation prediction | Removes blocked routes from the road graph and flags communities that have lost, or will soon lose, all access | Core | High |
| 4 | Cascading failure detection | Dependency map (road → fuel → hospital power) showing which disruption threatens which service, and the smallest fix | Core | Medium |
| 5 | Constrained allocation and routing | Who gets what, from which depot, on which vehicle and route, without exceeding stock or capacity (OR-Tools) | Core | Medium |
| 6 | Live replanning | A road closes or a report arrives, and the plan recalculates with a before/after explanation | Core (the wow moment) | Medium |
| 7 | Blind-spot detection | High hazard exposure plus few reports means "send verification here" | Supporting | High |
| 8 | NGO overlap and gap detection | Flags two teams heading to the same place and requests with no one assigned | Supporting | High |

**Roadmap only (mention, don't build):** cyclone and earthquake packs, live social-media ingestion, recovery and shelter management, government system integrations.

**Rule for the AI:** Gemini extracts, verifies and explains. Python and OR-Tools rank, allocate and validate. No allocation is invented by the LLM.

## Life-safety rules and SOPs

RELIEF RADAR covers four gaps that supply-only tools miss, and every rule behind them follows the authority's own SOPs instead of hard-coded logic.

### Blind spots we address

| Gap | What we add | Fits into feature |
| --- | --- | --- |
| Rescue and evacuation | Rescue requests (stranded, trapped, cannot evacuate) are a separate need type, routed to boats and rescue teams, ranked above supplies | Report understanding, allocation |
| Reaching vulnerable people | Vulnerability in the urgency score (elderly, disabled, pregnant women, infants, migrants) and a delivery-confirmation loop: "was it received, and by whom?" | Urgency ranking, NGO coordination |
| Responder safety | Routes carry a safety status (open, risky, closed). Risky routes need explicit approval; teams are never sent into danger just because a path is shortest | Allocation and routing |
| Time-critical medical needs | Dialysis, insulin, oxygen and near-term pregnancies get hard deadlines; post-flood disease and snakebite kits are planned for the following days | Urgency ranking, allocation |

### SOP-driven rules

Control rooms already follow SOPs (district plans, state SOPs, NDMA guidelines, the Incident Response System). RELIEF RADAR adapts to them:

1. **Upload the SOP.** The coordinator uploads the district or state SOP document.
2. **Gemini drafts the rules.** It extracts priority order, thresholds, route-safety limits, escalation steps and who approves what, each linked to the SOP clause it came from.
3. **A human approves the rules.** Nothing extracted from an SOP is enforced until an officer reviews it.
4. **The engine enforces them.** Ranking, allocation and routing follow the approved rules, and every recommendation cites the SOP clause behind it.
5. **Defaults when no SOP exists.** A sensible built-in rule set applies, clearly labeled as default.

| Rule area | Default | What an SOP can change |
| --- | --- | --- |
| Priority order | Rescue, then time-critical medical, then water, then food and shelter | Order, categories, tie-breaks |
| Vulnerability weighting | Elderly, disabled, pregnant women, infants weighted higher | Groups and weights |
| Route safety | Risky routes need approval; unsafe routes are never used | Thresholds (water depth, slope), who can approve |
| Medical deadlines | Dialysis and oxygen within hours | Conditions and time limits |
| Minimum coverage | Every affected community gets a minimum share | Minimum per community or per person |
| Approvals and escalation | Coordinator approves every dispatch | Roles per decision, following the Incident Response System |

This also answers the judge question "who decided these weights?": the authority's SOP did, and a human approved them.

## Positioning

Existing tools warn, map or list resources; RELIEF RADAR closes the last mile by deciding what to send where, and replanning when things change.

| Existing tool | What it does | What RELIEF RADAR adds |
| --- | --- | --- |
| Google Flood Hub, SACHET | Forecasts and alerts | Decides what to do after the alert |
| Ushahidi, crisis maps | Collect and map reports | Verifies, deduplicates and prioritizes them |
| IDRN | Lists available resources by district | Allocates them under real constraints |
| HYDRAA and city control rooms | Monitor hotspots, pre-position teams | Predicts isolation and cascades, coordinates NGOs |
| WhatsApp groups and spreadsheets | Manual coordination | One shared plan, overlap detection, replanning |

Tool descriptions are from general knowledge; verify each before quoting it in the pitch.

**Why generic matters:** India faces floods, cyclones, landslides and heatwaves, often in the same state and season. Control rooms shouldn't need a different tool for each.

**What not to claim:**

- That it is deployed or used by any authority.
- That it replaces officials. It is decision support.
- That the scoring weights are validated emergency standards.
- That simulated reports are real events. Say "real flood extent, simulated response."

## Demo scenarios

The demo shows one disaster in depth, then switches hazard packs to prove the engine is generic.

### Deep dive: Hyderabad flood replay (about 2 minutes)

Real October 2020 flood extent from satellite data; simulated reports, inventory and responders. One zone with about 15 communities, 3 depots, 2 hospitals, 3 shelters, 5 vehicles and 2 NGOs.

1. **T+0, blind spot:** the flood layer and incoming reports load. One heavily flooded community has almost no reports, and the system flags it for verification.
2. **T+2, isolation:** roads under water drop out of the graph. A community will be cut off in about 3 hours, so the system recommends pre-positioning supplies now.
3. **T+4, cascade:** a flooded road blocks the fuel route, putting a hospital's backup power at risk. The chain lights up, and the system proposes the smallest intervention that breaks it.
4. **T+6, coordination:** two NGOs are heading to the same shelter while another request has no assignee. The system reassigns them.
5. **Climax, live disruption:** a bridge closes. The plan re-optimizes and shows a before/after comparison with reasons. The coordinator approves.

### Switch: landslide simulation (about 20 seconds)

Load the landslide pack on a hill-district scenario. The same engine finds that a single slope failure isolates a whole valley, no road route remains, and it recommends air drops or foot teams. This proves the platform claim without building everything twice.

## Data sources

Use real data for geography and hazards, and clearly labeled synthetic data for everything operational.

| Data | Used by | Real or synthetic | Where to get it |
| --- | --- | --- | --- |
| Flood extent | Flood pack | Real | Sentinel-1 SAR in Earth Engine (Oct 2020, Hyderabad) |
| Permanent water | Flood pack | Real | JRC Global Surface Water (Earth Engine) |
| Elevation and slope | Flood and landslide packs | Real | SRTM or similar DEM (Earth Engine) |
| Rainfall | Both packs | Real | CHIRPS or GPM IMERG (Earth Engine); IMD if accessible |
| Landslide susceptibility | Landslide pack | Real or derived | Slope plus rainfall in Earth Engine; public susceptibility maps if available |
| Roads | Core engine | Real | OpenStreetMap (OSMnx or Overpass) |
| Population | Core engine | Real | WorldPop or GHSL |
| Hospitals, schools as shelters | Core engine | Real locations | OpenStreetMap, healthsites.io |
| Fuel, beds, shelter occupancy | Cascade feature | Synthetic | Generated and labeled |
| Depots, inventory, vehicles | Allocation | Synthetic | Generated CSVs |
| Emergency reports and photos | Report understanding | Synthetic | Written by the team in English, Telugu and Hindi |
| NGO assignments | Coordination | Synthetic | Generated with one planted overlap and one gap |

Dataset IDs, date coverage and licenses are from general knowledge. Confirm Sentinel-1 coverage over Hyderabad for mid-October 2020 first, since the deep dive depends on it.

**Planted scenario:** decide in advance one blind-spot community, one isolation, one hospital fuel cascade, one NGO overlap, duplicate and stale reports, and one bridge closure. The system must catch each one.

## How we prove it works

Run the same planted scenario twice, once with a first-come, first-served baseline and once with RELIEF RADAR, and report only the measured results.

| Metric | How we measure it |
| --- | --- |
| Urgent needs unmet | Count of critical needs not served, baseline vs RELIEF RADAR (headline metric) |
| Planted issues caught | Blind spot, isolation, cascade, overlap, duplicates: how many of the planted issues the system flags |
| Constraint violations | Allocations exceeding stock, vehicle capacity or using blocked roads (target: zero) |
| Replan time | Seconds from a disruption event to a new valid plan |
| Extraction accuracy | Share of reports with correct location, need and urgency against our labeled set |
| Evidence links | Share of recommendations that cite the correct source report and timestamp |

## Open decisions

- [ ] Exact Hyderabad zone for the replay (visible 2020 flooding, a hospital nearby)
- [ ] Landslide scenario location (e.g., a Wayanad-style hill district)
- [ ] Resource types; suggested four: drinking water, medical kits, food kits, diesel
- [ ] Report languages; suggested English plus Telugu
- [ ] The headline metric for the pitch
- [ ] Team size and owners per feature
- [ ] Submission format and pitch length (live demo, video, deck)

* [ ] Which real SOP to load for the demo (e.g., a Telangana or Hyderabad flood SOP, or NDMA flood guidelines)
