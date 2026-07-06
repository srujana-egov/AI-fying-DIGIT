# Vision: DIGIT 3.0 Becomes AI-Operable

## The Missing Piece

DIGIT 3.0 is deployed. Sixteen platform services, eighteen domain products, running across hundreds of cities.

Developers already have a path: give Claude the DIGIT codebase and build. That works — for developers, in development environments, producing code.

DIGIT's operational users — city commissioners, state officials, field workers — cannot do this. They work on live systems. They need answers from running DIGIT instances, not code. They cannot use an IDE.

Three things repo access cannot do:

- **Reach live data.** The codebase is static files.
- **Scale to non-technical users.** You cannot give thousands of city officials a repository and an IDE.
- **Run safely in production.** An LLM with repo access and terminal can do anything. A live government system needs a controlled surface: only DIGIT APIs, the user's own identity, confirmation before writes, audit trail.

This is not a capability gap. A terminal can already run `curl` against a live instance — that is real execution, not a hypothetical. It is a control gap: terminal access is unbounded, reaching any endpoint with any command, and it requires a raw credential exposed to the model. MCP does not add capability the model lacked. It removes scope — a named list of reviewed actions instead of a shell, the user's token forwarded through Kong instead of held by the model, a confirmation step that is not optional because no other path to a write exists.

This access — terminal, live credentials, whole instance — is not new. An SI partner configuring a deployment, or a support engineer debugging in production, already has it. That was safe by default, because a human is bounded by their own time and judgment, one action at a time. Handing that same access to an LLM, unsupervised, removes the bound: it can act on the entire instance at machine speed. Stakeholders will correctly refuse that trade. Without a controlled surface, DIGIT does not get AI-fied — it stays manual, because unsupervised full access is not a risk anyone signs off on.

MCP is that surface. Any LLM calls any DIGIT API on behalf of any authenticated user — with confirmation before writes and a complete audit trail.

---

## What This Enables

**City administrator**

> "Which commercial properties in Ward 5 have an active trade license but are more than 6 months behind on water charges?"

Claude calls Trade License, calls Water & Sewerage, joins on property identifier, returns the enforcement list. Two separate DIGIT services. Currently: a developer writes a custom query. With AI-operable DIGIT: the commissioner asks it directly, from the live system, right now.

**State official**

> "Which cities have high trade license registrations relative to property tax collections? That gap is usually a sign of misclassification."

Claude queries both metrics across all tenants simultaneously and flags the outliers. A state analyst compiles this once a year from city reports. With AI-operable DIGIT: live data, seconds, repeatable.

**Field worker (HCM)**

A field worker registers a beneficiary in the HCM Flutter app. Before submission: "Possible duplicate: Amina Kone, age 3, 200m from here. Same child?" Caught at the point of creation, not discovered weeks later in a data quality audit.

---

## The Second Capability: 5 Intelligence Patterns

Once DIGIT is AI-operable, scheduled agents can read DIGIT records, apply an intelligence signal, and write findings back — no human trigger required.

**Flagging** — AI compares submitted data against expected values. Flags inconsistencies. Writes flags back via DIGIT API.

**GIS Cross-Reference** — Declared use vs satellite land-use. Declared area vs satellite measurement.

> Every city running DIGIT has businesses that told the government they are a residence. Satellite imagery says they are a shop. The gap is recoverable.

**Proactive Alerting** — Ward 12 has had waterlogging complaints every July for three years. The system detects this in June. The drainage engineer gets a WhatsApp. No complaints from Ward 12.

The highest form of governance: the problem solved before the citizen knew it existed.

**Deduplication** — On-device fuzzy match before submission. Server-side MOSIP biometric matching for national identity infrastructure. Duplicates caught before they enter the system.

**Process Intelligence** — DIGIT's workflow service records every state transition with timestamps. SLA prediction, bottleneck detection, inspector delay flagging — the data is already there.

These 5 patterns apply across all 18 DIGIT domain products. See [AI Patterns Across DIGIT Products](14-ai-patterns-across-digit-products.md) for the full product-by-product map.

---

## The Scale Argument

The platform is the leverage point. AI built once at the platform layer is inherited by every domain, every city, every stakeholder.

DIGIT does not change. It becomes AI-operable.
