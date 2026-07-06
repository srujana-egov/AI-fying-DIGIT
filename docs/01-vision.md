# Vision: What AI-fying DIGIT Makes Possible

## The Revenue Question Government Has Always Wanted Answered

Every city running DIGIT has businesses that told the government they are a residence. Satellite imagery says they are a shop. The government has been collecting residential tax rates on commercial properties — for years. The gap is not accidental. It is systematic, large, and recoverable.

> *"Which businesses declared residential use in their trade license application, but GIS data shows they are operating commercially?"*

We can answer that question for any city running DIGIT, in minutes. Not a report — a list of flagged records in DIGIT with the evidence attached: declared use, GIS land-use classification, source, confidence. Revenue officials query the flagged records and act. The AI does the cross-referencing. The government collects the gap.

The same logic, applied across domains:

| Domain | Declared in DIGIT | Ground truth signal | Gap |
|---|---|---|---|
| Trade License | Use: residential | GIS: commercial activity | Revenue under-collection |
| Property Tax | Area: 500 sq ft | Satellite measurement: 800 sq ft | Assessment under-declaration |
| Water Connection | Category: domestic | Consumption pattern: commercial | Mis-tariffing |
| Construction Permit | Approved use: residential | Satellite: commercial structure | Compliance violation |
| Health Campaign | Population register: 400 households | WorldPop + satellite rooftops: 520 | Invisible population, missed coverage |

Same AI, same platform, same pattern. Not a dashboard — flags written to DIGIT records via the API. Revenue officials and programme managers query flagged records using their existing tools. eGov surfaces the intelligence; the government decides and acts.

**This is already being built.** The health campaign version — cross-referencing WorldPop and Google Open Buildings against HCM enrollment data in Chad, Sierra Leone, Nigeria, and Liberia — is being developed by the platform team now. The same pattern applies to Property Tax and Trade License in any DIGIT city.

---

## Before the Flood, Not After

Ward 12 has had waterlogging complaints every July for three consecutive years. The system detects this pattern in June. It alerts the drainage engineer. Preventive cleaning happens June 10th. July arrives. No complaints from Ward 12.

The citizen receives a WhatsApp via DIGIT notification service: *"We have completed preventive drain maintenance in your area ahead of the monsoon."*

The highest form of governance: the problem solved before the citizen knew it existed. Every component needed to build this is either already in DIGIT (PGR, workflow, notification) or being built now (the recurrence detector and alert engine in the current intern project).

The same pattern generalises:
- Property tax non-filers who appear on the same list three years running
- Pump failure signatures that emerge 90 days before actual breakdown in water schemes
- Health campaign wards that consistently under-report coverage cycle after cycle

The signal is already in DIGIT. The action is already in DIGIT. The AI is the connection between them — running on a schedule, writing flags and triggering alerts, requiring no new platform infrastructure.

---

## The Scale Argument

DIGIT runs across hundreds of cities in India and internationally. Each city has property tax records, trade licenses, water connections, complaints, and workforce data. None of it is currently being cross-checked against ground truth.

When the platform AI layer is built once:

- Every city running DIGIT gets GIS revenue intelligence — not one pilot city
- Every complaint service gets proactive recurrence alerting — not one ward
- Every health campaign gets satellite-verified population denominators — not one country
- Every trade license portfolio gets land-use cross-referencing — not one inspector's manual effort

The same five patterns — flagging, GIS cross-reference, proactive alerting, deduplication, process intelligence — apply across all 18 DIGIT domain products. Build the platform layer once. Every domain gets AI capability without rebuilding the infrastructure.

---

## What This Is Not

**Not a dashboard.** PowerBI and DSS exist. The intelligence layer flags records in DIGIT. Consumers visualise with their own tools. eGov's job is the intelligence, not the visualisation.

**Not citizen-facing AI.** Citizens interact with apps built on DIGIT. Whether those apps have AI features is the app partner's decision — the same way whether they use React or Flutter is their decision. This proposal does not touch citizen-facing applications.

**Not AI-assisted city configuration.** Configuration happens once per city. Tenant isolation means one city cannot read another city's setup. This is operational intelligence — thousands of transactions per day, recurring patterns, at-scale problems. One-time configuration is not in scope.

---

## What the Platform Becomes

Any AI agent — Claude, GPT, a custom orchestrator — calls DIGIT via an MCP server auto-generated from clean OpenAPI 3.0.3 specs. The agent queries the workflow service to know what transitions are valid right now. It does not reason about what is possible — it asks the platform.

A city commissioner asks in natural language. A flagging microservice runs on a schedule. A workflow spans Trade License, Fire NOC, and Water Connection in parallel. All of them call the same DIGIT APIs, through the same confirmation gate, with the same audit trail.

DIGIT does not change. It becomes AI-operable.
