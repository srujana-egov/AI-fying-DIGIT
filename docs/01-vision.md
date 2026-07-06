# Vision: DIGIT 3.0 Becomes AI-Operable

## The Missing Piece

DIGIT 3.0 already exists. Sixteen platform services. Eighteen domain products. JWT authentication, tenant isolation, RBAC, workflow engine, notification service — all built and deployed across hundreds of cities.

The missing piece is not more platform. The missing piece is that no LLM can currently operate it.

Give Claude the DIGIT codebase today. It can read it and explain it. It cannot call any live API. It cannot authenticate as a user. It cannot confirm before making a change. It cannot log what it did.

Reading about DIGIT is not operating DIGIT.

That is the gap this proposal closes.

---

## What AI-Operable DIGIT Looks Like

Certificate-standard OpenAPI specs → `openapi-generator-cli` → typed HTTP clients → MCP tool definitions. Auto-generated. Kong (digit3) validates the JWT and forwards the Bearer token. The MCP server adds a confirmation gate for all writes and an audit log for every action.

The result: any LLM — Claude, GPT, a custom orchestrator — can call any DIGIT API, on behalf of any authenticated stakeholder, with human confirmation before writes and a complete audit trail after.

DIGIT does not change. It becomes AI-operable.

---

## What This Enables, Per Stakeholder

**Implementation team**

> "Set up trade license for pb.amritsar."

Claude calls the MDMS configuration API, the workflow definition API, the IDGen format configuration API — in sequence. Before each write, it shows exactly what it is about to do and asks for confirmation. The setup that takes a trained engineer half a day to script becomes a supervised conversation. RAG V5 answers "what does this field mean?" instantly without opening a spec file.

**City administrator**

> "Which grievance complaints have been open more than 30 days in Ward 5?"

Claude calls the PGR search API with the right filters. It gets real data from the running DIGIT instance. It returns a plain-English summary. No dashboard needed. No report requested. A direct answer from a live system.

> "How many trade licenses expire in the next 60 days?"

Same pattern. One tool call. Real data. Immediate answer.

**State official**

> "Compare grievance resolution rates across cities this quarter."

Claude calls PGR aggregate APIs across tenants. Returns a ranked list with the outliers flagged. No waiting for city-level reports to be manually compiled.

**Field worker (HCM)**

A field worker registers a beneficiary in the HCM Flutter app. Before submission, on-device fuzzy matching checks: name + age + GPS proximity. "Possible duplicate: Amina Kone, age 3, 200m from here. Same child?" Worker confirms or proceeds. The duplicate is caught at the point of creation, not discovered weeks later in a data quality audit.

---

## The Second Capability: 5 Intelligence Patterns as Scheduled Agents

Once DIGIT is AI-operable, a second class of capability becomes possible: scheduled agents that read DIGIT records, apply an intelligence signal, and write findings back — without any human trigger. These run on top of the same MCP server, the same auth propagation, the same audit trail.

**Pattern 1 — Flagging: declared vs expected**
AI compares submitted data in DIGIT records against expected values — domain rules, models, or external signals. Flags inconsistencies. Writes flags back via DIGIT API. One microservice per domain. No dashboard — consumers query flagged records using their existing tools.

**Pattern 2 — GIS Cross-Reference: satellite vs registry**
Declared use vs GIS land-use. Declared area vs satellite measurement. Population register vs WorldPop + Google Open Buildings.

> Every city running DIGIT has businesses that told the government they are a residence. Satellite imagery says they are a shop. The government has been collecting residential tax rates on commercial properties — for years. The gap is recoverable.

The HCM version — cross-referencing WorldPop and Google Open Buildings against campaign enrollment data in Chad, Sierra Leone, Nigeria, and Liberia — is being built by the platform team now. The same pattern applies to Property Tax and Trade License in any DIGIT city.

**Pattern 3 — Proactive Alerting: act before the event**
Ward 12 has had waterlogging complaints every July for three consecutive years. The system detects this in June. The drainage engineer gets a WhatsApp. Preventive work happens June 10th. No complaints from Ward 12.

The highest form of governance: the problem solved before the citizen knew it existed. Every component is already in DIGIT — PGR, workflow, notification. The recurrence detector is the only new piece.

**Pattern 4 — Deduplication: point of submission**
On-device fuzzy match (Flutter library, offline-capable) for field conditions. Server-side MOSIP biometric matching for national identity infrastructure. Both modes catch duplicates before they enter the system — not after.

**Pattern 5 — Process Intelligence: workflow history → signal**
DIGIT's workflow service records every state transition with timestamps. That history exists for every application, in every city. SLA prediction, bottleneck detection, inspector delay flagging — no new data collection needed. The data is already there.

These 5 patterns apply across all 18 DIGIT domain products. See [AI Patterns Across DIGIT Products](14-ai-patterns-across-digit-products.md) for the full product-by-product map.

---

## The Scale Argument

When the platform AI layer is built once:

- Every city running DIGIT gets GIS revenue intelligence — not one pilot city
- Every complaint service gets proactive recurrence alerting — not one ward
- Every health campaign gets satellite-verified population denominators — not one country
- Every implementation team gets an AI-assisted setup process — not one trained engineer

The platform is the leverage point. AI-operable DIGIT means every domain, every city, every stakeholder benefits from the same infrastructure — without each team rebuilding auth, confirmation, audit, and entity resolution from scratch.

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
