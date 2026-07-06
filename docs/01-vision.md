# Vision: DIGIT 3.0 Becomes AI-Operable

## The Missing Piece

DIGIT 3.0 already exists. Sixteen platform services. Eighteen domain products. JWT authentication, tenant isolation, RBAC, workflow engine, notification service — all built and deployed across hundreds of cities.

The missing piece is not more platform. The missing piece is that no LLM can currently operate it.

A developer can give Claude the DIGIT codebase in their IDE. Claude reads it, helps write code, executes commands. That works — for one developer, in a development environment, producing code as output.

That is not the problem this proposal solves.

DIGIT is used operationally by city commissioners, state officials, municipal staff, and field workers across hundreds of cities. They do not have an IDE. They work in running systems against live data. When a city administrator asks "which complaints have been open more than 30 days in Ward 5", they want the answer — from the live system, right now. Not code that someone else has to run.

Three things repo access in an IDE cannot do:

- **It cannot reach live data.** The codebase is static files. Answering real operational questions requires calling a running DIGIT instance.
- **It cannot scale to non-technical users.** There are thousands of city officials, state officers, and field workers across hundreds of ULBs. You cannot give each of them a code repository and an IDE.
- **It is not safe for production.** An LLM with full repo access and terminal can do anything — push code, delete files, run arbitrary commands. In a live government system you need a controlled surface: only the DIGIT APIs, with the user's own identity, with confirmation before every write, with a log of every action.

MCP is that controlled surface. It is how an LLM operates DIGIT on behalf of any user — not just a developer, and not just in development.

That is the gap this proposal closes.

---

## What AI-Operable DIGIT Looks Like

Any LLM — Claude, GPT, a custom orchestrator — can call any DIGIT API, on behalf of any authenticated stakeholder, with human confirmation before writes and a complete audit trail after.

DIGIT does not change. It becomes AI-operable.

---

## What This Enables, Per Stakeholder

**Implementation team**

> "The Trade License workflow for Amritsar has 4 configured states. Which state transitions are missing notification templates?"

Claude calls the workflow service to get the state list. It calls the notification service to get configured templates. It cross-references the two and returns the gaps. A trained engineer does this manually today — reading two separate service configs, matching by state name, hoping nothing was missed. Claude does it in one question. RAG V5 answers "what should the escalation template say?" without opening a spec file.

**City administrator**

> "Which commercial properties in Ward 5 have an active trade license but are more than 6 months behind on water charges?"

Claude calls the Trade License service (active licenses, Ward 5 commercial), calls the Water & Sewerage service (payment history per connection), joins on property identifier, returns the enforcement list. This crosses two completely separate DIGIT services. Currently, answering this requires a developer to write a custom query against two databases. With AI-operable DIGIT, the commissioner asks it directly — from a live system, in plain English, right now.

**State official**

> "Which cities have high trade license registrations relative to property tax collections? That gap is usually a sign of misclassification."

Claude queries both metrics across all tenants simultaneously, computes the ratio, and flags the outliers. A state analyst compiles this once a year from city-submitted reports. With AI-operable DIGIT, the same intelligence runs against live data in seconds — and can be asked again tomorrow.

**Field worker (HCM)**

A field worker registers a beneficiary in the HCM Flutter app. Before submission, on-device fuzzy matching checks: name + age + GPS proximity. "Possible duplicate: Amina Kone, age 3, 200m from here. Same child?" Worker confirms or proceeds. The duplicate is caught at the point of creation, not discovered weeks later in a data quality audit.

---

## The Second Capability: 5 Intelligence Patterns as Scheduled Agents

Once DIGIT is AI-operable, a second class of capability becomes possible: scheduled agents that read DIGIT records, apply an intelligence signal, and write findings back — no human trigger required.

**Pattern 1 — Flagging: declared vs expected**
AI compares submitted data against expected values — domain rules, models, or external signals. Flags inconsistencies. Writes flags back via DIGIT API.

**Pattern 2 — GIS Cross-Reference: satellite vs registry**
Declared use vs GIS land-use. Declared area vs satellite measurement. Population register vs WorldPop + Google Open Buildings.

> Every city running DIGIT has businesses that told the government they are a residence. Satellite imagery says they are a shop. The government has been collecting residential tax rates on commercial properties — for years. The gap is recoverable.

**Pattern 3 — Proactive Alerting: act before the event**
Ward 12 has had waterlogging complaints every July for three consecutive years. The system detects this in June. The drainage engineer gets a WhatsApp. Preventive work happens June 10th. No complaints from Ward 12.

The highest form of governance: the problem solved before the citizen knew it existed.

**Pattern 4 — Deduplication: point of submission**
On-device fuzzy match (Flutter library, offline-capable) for field conditions. Server-side MOSIP biometric matching for national identity infrastructure. Both modes catch duplicates before they enter the system — not after.

**Pattern 5 — Process Intelligence: workflow history → signal**
DIGIT's workflow service records every state transition with timestamps. That history exists for every application, in every city. SLA prediction, bottleneck detection, inspector delay flagging — no new data collection needed. The data is already there.

These 5 patterns apply across all 18 DIGIT domain products. See [AI Patterns Across DIGIT Products](14-ai-patterns-across-digit-products.md) for the full product-by-product map.

---

## The Scale Argument

The platform is the leverage point. AI built once at the platform layer is inherited by every domain, every city, every stakeholder — without each team rebuilding auth, confirmation, or audit from scratch.

