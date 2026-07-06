# Who Actually Talks to DIGIT at the Platform Level

Citizens interact with apps built on DIGIT. Whether those apps use AI is the app partner's decision. This proposal does not touch citizen-facing applications.

The stakeholders below interact with DIGIT directly. They fall into two groups: those for whom AI-fying DIGIT adds nothing (developers, implementation teams doing one-time city setup), and those for whom it is transformative (operational users on live systems).

---

## Developers

The right tool for developers building on DIGIT is **repo access + Claude Code**.

DIGIT 3.0 ships digit-client (Java), digit-cli, and clean OpenAPI 3.0.3 specs. Add the digit3 repo to your Claude context. Claude reads the specs, understands the client library, writes integration code, executes commands. This already works — the platform team used it to write PGR services and configuration via the CLI.

**AI-fying DIGIT does not help developers build. That problem is already solved at the platform level.**

RAG V5 adds marginal value for documentation Q&A ("what fields does this endpoint require?") but a developer with repo access and Claude Code gets better answers directly from the spec files and service code than from a chatbot.

---

## Why City Configuration Is Not Worth AI-fying

City onboarding — setting up MDMS, workflow definitions, IDGen formats for a new city — is not a good use of AI for two reasons:

**It is a one-time operation.** City configuration happens once per city. AI provides leverage over recurring operations at scale — thousands of daily transactions. Automating a 10-step one-time setup adds marginal value.

**Tenant isolation prevents cross-city learning.** DIGIT's tenant isolation means one city cannot read another city's MDMS config, boundary hierarchy, or workflow definitions. An AI agent cannot suggest "configure it like Amritsar did" — that data is architecturally inaccessible.

Implementation teams arrive trained. What helps them is RAG V5 for documentation questions during setup — not AI automating the setup itself.

---

## Operational Users — What AI-fying DIGIT Enables

These stakeholders are what this proposal is built for. They work on live systems, against live data, and cannot use an IDE.

---

### City Administrator

**What they need:** Status and intelligence across city services — complaints, trade licenses, property tax, water — without waiting for compiled reports.

**What MCP gives them:** Direct queries to live DIGIT data in plain English. The MCP server exposes the full application service surface:

| Service | What the administrator can ask |
|---|---|
| PGR | "Which complaints have been open more than 30 days in Ward 5?" |
| Trade License | "How many licenses expire in the next 60 days?" |
| Property Tax | "Which properties in Ward 7 are more than 90 days behind?" |
| Water & Sewerage | "Which commercial connections have 6 months of arrears?" |

**The cross-service capability is what's new.** Claude calls Trade License AND Water & Sewerage, joins on property identifier, returns the enforcement list. No developer writes a custom query. The commissioner asks it directly, from the live system, right now.

**What RAG V5 gives them:** "How does the renewal process work?" → documentation answer from the knowledge base. "Which licenses expire this month?" → that is MCP, not RAG. The router between them is a single intent question: is this a question about how something works, or a request for live data?

**Writes:** Confirmation required before any state change. DIGIT's RBAC determines what they are allowed to change. The AI never holds elevated permissions.

---

### State Official

**What they need:** Cross-city intelligence — which cities are performing, which need intervention — without waiting for city-submitted annual reports.

**What MCP gives them:** Cross-tenant queries against live data. A state official role in DIGIT's RBAC grants read access across tenants. The MCP forwards their JWT unchanged; DIGIT enforces what they can see.

| What was previously | With AI-operable DIGIT |
|---|---|
| Annual report, manually compiled | Live query, seconds |
| "Which cities improved?" → weeks of analyst work | One question |
| "Which cities are at risk this quarter?" → nobody asks | Ask it every month |

**No writes.** State officials read. Decisions are made by city officials.

---

### Field Worker (HCM)

**What they need:** To register beneficiaries accurately in the field — often offline, often under time pressure.

**What AI gives them:** On-device deduplication before submission. Before registering a beneficiary, fuzzy matching checks name + age + GPS proximity against existing records. "Possible duplicate: Amina Kone, age 3, 200m from here. Same child?" Worker confirms or proceeds. Duplicate caught at the point of creation, not discovered weeks later in a data quality audit.

This runs in the Flutter app before any API call — not through MCP. The deduplication logic is a Flutter library. Every HCM campaign inherits it without rebuild.

---

## Summary

| Stakeholder | Right tool | What AI-fying DIGIT adds |
|---|---|---|
| Developers | Repo access + Claude Code | Nothing — already solved |
| Implementation teams (city setup) | RAG V5 for Q&A | Documentation answers only — setup itself is manual |
| City administrators | MCP | Live cross-service queries; writes with confirmation |
| State officials | MCP (read-only) | Cross-tenant live queries; no writes needed |
| Field workers | On-device AI | Deduplication before submission |

The AI layer always propagates the authenticated user's identity to DIGIT's APIs. The AI never holds elevated permissions. DIGIT's RBAC is the enforcement mechanism.
