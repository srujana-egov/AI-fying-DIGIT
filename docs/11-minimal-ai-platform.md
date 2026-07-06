# The Minimal AI Platform: Assuming Specs at the Certificate Service Standard

**The central question the principal architect posed:**  
If all DIGIT specs were as good as the certificate service spec — self-describing, human-readable codes, rich descriptions, interaction diagrams — what would the AI platform architecture be, and what artifacts would still be needed?

This document answers that precisely.

---

## What the Current digit-specs Look Like vs the Target Standard

Application-level specs span all DIGIT domain products — 20 in total across local governance, public health, public finance, water & sanitation, revenue, and justice. Not all have specs today.

**7 found in `digitnxt/digit-specs` (2.9-era):**
`tl-service.yml` · `bpa.yaml` · `property-services.yml` · `fire-noc.yaml` · `water-sewerage.yaml` · `pgr.yml` · `birth-registration.yaml`

These are 2.9-era specs. Opaque codes, minimal descriptions, RequestInfo wrappers, no examples, no error semantics.

**~11 additional products need specs created at certificate standard:**
Works Management · HCM · 10 Bed ICU · DIVOC · Social Benefits · iFix · Waste Management · Water Supply O&M · Water Schemes O&M · mCollect · DRISTI

**The target standard** (certificate service spec) has:

| Property | 2.9 specs | Certificate service standard |
|---|---|---|
| Codes | Opaque integers / short codes | Human-readable: `LICENSE`, `TRADE`, `INSTITUTION` |
| Descriptions | Minimal | Business logic, validation rules, holder vs applicant |
| Examples | None | Realistic examples for every endpoint |
| tenantId | In request body | From Bearer token — not a body field |
| Idempotency | None | `Idempotency-Key` header on all writes |
| Error semantics | Generic 500s | Meaningful: 409 for duplicate, 403 for inactive, 404 for not found |
| Cross-service deps | Not mentioned | Named in descriptions ("Form Registry manages certificateDetail schema") |

**What raising all application specs to this standard eliminates:**

| What the 2.9 experiment built | Why it existed | Status at certificate standard |
|---|---|---|
| 61 hand-authored tool definitions | 2.9 specs too opaque for AI | **Eliminated** — auto-generate from specs |
| Semantic layer (`@digit-mcp/data-provider`) | Codes uninterpretable | **Eliminated** — codes are already human-readable |
| Tool descriptions rewritten for AI | Spec summaries insufficient | **Eliminated** — descriptions are already AI-readable |
| Skills for single-service sequencing | Spec didn't explain dependencies | **Reduced** — interaction diagrams encode this |

---

## What Still Needs to Exist

With all specs at the certificate service standard, the remaining artifacts split into two owners: platform team and AI execution layer.

---

### Platform Team Artifacts (5 things)

**1. Specs at the certificate service standard** ← the principal architect's commitment  
All 7 current services rewritten. Every new service starts at this standard.  
Not an AI concern — this is platform quality.

---

**2. Interaction diagrams — one per major operation**

Format: standard UML sequence diagram (Mermaid or websequencediagrams text source — not a PNG).  
Lives: alongside the spec in the same repo directory.

The diagram must capture three things or it is insufficient:

```
title applyForCertificate — Full Integration

note over Client,WfSvc: Prerequisites: certificateType.isActive=true,
                         definitionVersion configured

Client->Service: POST /certificate-types/{code}/certificates
Service->FormRegistry: validateDefinition(definitionVersion)
FormRegistry-->Service: valid / 400

Service->IDGen: generate(CERT_APP) → applicationNumber
Service->WfSvc: initiate(CERT_APPLY, applicationNumber)
WfSvc->WfSvc: Validate State, Role, Role-Action
WfSvc-->Service: workflowInstanceId [201] / [500]

alt WfSvc returns 500
  Service->IDGen: release(applicationNumber)
  Service-->Client: 500
end

Service->FileStore: validate(documents[])
Service-->Client: 201 Certificate (applicationNumber, workflowInstanceId)
```

The agreed sequence diagram format (websequencediagrams) is correct.  
What it needs that the reference diagram didn't have: prerequisites, data on arrows, failure alt block.  
This is the only format change needed — not the YAML manifest.

---

**3. Response enrichment — code + label in same response**

```json
{
  "instrumentType": "LICENSE",
  "instrumentTypeLabel": "Trade License",
  "status": "IN_PROGRESS",
  "statusLabel": "Application under review"
}
```

No resolution middleware needed. The AI gets both the code (for API calls) and the label (for responses to humans). Eliminates the localization lookup layer entirely.

---

**4. Gateway configuration**

Standard API gateway (Kong / NGINX / any). Two rules:

```
Whitelisted (no JWT required — bootstrap):
  POST /account/create
  POST /auth/token
  POST /otp/send
  POST /otp/verify
  POST /user/create (initial admin only)

Protected (validate JWT → propagate downstream):
  everything else
```

If the gateway sits above everything and requires JWT for all calls, account creation deadlocks. The whitelist solves the bootstrap problem. After account creation and login, the user's JWT propagates on every downstream call. DIGIT's own RBAC enforces what that JWT can do.

---

**5. Audit log schema — defined at platform level**

One schema, all AI callers produce it consistently:

```json
{
  "timestamp": "ISO 8601",
  "userId": "ramesh.kumar",
  "tenantId": "pb.amritsar",
  "sessionId": "uuid-v4",
  "userQuery": "apply for trade license for my shop",
  "operationId": "applyForCertificate",
  "confidence": 0.94,
  "proposedAction": {
    "endpoint": "POST /certificate-types/trade-license/certificates",
    "params": { "applicantIds": [...], "certificateDetail": {...} }
  },
  "humanConfirmation": "yes",
  "idempotencyKey": "uuid-v4",
  "httpStatus": 201,
  "result": { "applicationNumber": "CERT-APP-2026-001234" }
}
```

Defined by the platform team. Implemented by every AI caller. RTI-answerable.

---

### AI Execution Layer (3 things — built once, shared by all)

**1. MCP server — auto-generated, not authored**

Build step, not a project:
```
digit-specs/*.yaml
    → OpenAPI Generator
    → raw MCP tool definitions
    → + progressive disclosure group tags (see below)
    → MCP server
```

Progressive disclosure groupings (also the principal architect's point — useful for humans too):
```
core (loads on startup — 8 tools):
  certificate_list, certificate_get, workflow_get_transitions,
  individual_search, boundary_get, mdms_search, enable_tools, help

setup (unlock for implementation teams):
  certificate_type_create, workflow_definition_create, idgen_configure...

operations (unlock for city administrators):
  certificate_apply, certificate_transition, certificate_renew...

state (unlock for state officials):
  certificates_search_cross_type, analytics_aggregate...
```

This grouping is a one-time design decision. After that, add a spec → get a tool automatically.

---

**2. Confirmation gate — ~100 lines, not a framework**

```
Any write operation:
    AI selects tool, constructs params
    → store as pendingAction in Redis (TTL: 10 min)
    → return to user: "I will {plain English description}. Confirm? YES/NO"
      showing exact endpoint + params, not AI prose

User: YES
    → retrieve pendingAction
    → execute HTTP call with user's Bearer token + Idempotency-Key
    → write audit log
    → clear pendingAction

User: NO
    → clear pendingAction
    → done
```

Read operations bypass confirmation entirely — execute directly.  
This is the only place where "AI proposes, human confirms, AI executes" happens. It is not an orchestration framework. It is 100 lines of code.

---

**3. Cross-module workflow definitions — authored, ~5 key workflows**

The interaction diagrams handle single-service understanding. For operations that span multiple services, a workflow definition is still needed. These are authored once and don't change often.

The five that matter:

| Workflow | Services spanned |
|---|---|
| Start a business | Trade License + Fire NOC + Water Connection (parallel) |
| Annual renewal campaign | Certificate service × all active holders |
| Revenue recovery | Property Tax + Water & Sewerage (cross-module query + action) |
| Post-disaster triage | PGR + boundary + assignment (prioritise and dispatch) |
| Commissioner's brief | PGR + PT + W&S + HCM (aggregate read, no writes) |

**For execution:** Temporal (Row 20 in AoP) is the right engine for these. It handles parallel execution, wait-for-all, partial failure compensation, and long-running workflows natively. OpenFn handles data integration well but is not designed for durable multi-step transactional flows the way Temporal is.

---

## The Complete Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Any consumer: city admin, state official, implementation team, │
│  AI agent, HCM console, PGR copilot, WhatsApp bot              │
└──────────────────────────┬──────────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │       API Gateway        │
              │  Whitelist: bootstrap    │
              │  Protected: JWT → fwd   │
              └────────────┬────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
   ┌─────▼──────┐  ┌───────▼──────┐  ┌──────▼──────┐
   │  RAG V5    │  │  MCP Server  │  │  Temporal   │
   │  Layer 1   │  │  Layer 2     │  │  Layer 2    │
   │            │  │              │  │             │
   │  "How do   │  │  Auto-gen    │  │  Cross-     │
   │  I..."     │  │  from specs  │  │  module     │
   │  queries   │  │  + confirm   │  │  workflows  │
   │            │  │  gate        │  │             │
   └─────┬──────┘  └───────┬──────┘  └──────┬──────┘
         │                 │                 │
         └─────────────────┼─────────────────┘
                           │ Bearer token forwarded
                           │ Audit log written
                           ▼
         ┌─────────────────────────────────────┐
         │         DIGIT APIs                  │
         │  (specs at certificate standard)    │
         │  + interaction diagrams             │
         │  + response enrichment              │
         │                                     │
         │  certificate · pgr · tl · bpa ·      │
         │  property · fire-noc · water ·      │
         │  works · hcm · waste · mcollect ·   │
         │  social-benefits · ifix · dristi    │
         │  + all other DIGIT products         │
         └─────────────────────────────────────┘
```

No semantic layer. No tool registry. No custom intent classifier. No skills for single-service operations.

---

## AoP Projects: Which Are Relevant

### Directly relevant — should be coordinated under this architecture

| # | Initiative | Role in architecture | Direction |
|---|---|---|---|
| 2 | LLM guidance chatbot | RAG V5 — Layer 1 | Keep. Expand knowledge base to include all specs + interaction diagrams. |
| 5 | MCP for DIGIT PGR | MCP Server — Layer 2 | Generalize. Don't build PGR-specific — build the generator that produces MCP from any spec. PGR is the first test case, not the final scope. |
| 8 | Service Config AI Agent | Limited — see note | Reframe. Configuration is one-time per city (not at-scale) and tenant isolation prevents cross-city template borrowing. The confirmation gate mechanism is still needed at the platform level — but for operational write operations (transition approvals, batch renewals), not configuration. |
| 13 | Console config chatbot | Layer 1 + Layer 2 | Don't build RAG for config generation — RAG answers questions, it can't execute. Wire RAG V5 (questions) + confirmation gate (actions) with a thin intent router between them. |
| 17 | HCM support chatbot | Layer 2 + Layer 4 | Keep. This is the HCM-facing interface to the shared Layer 2. Should call the MCP server, not embed its own AI logic. |
| 19 | HCM conversational data layer | Layer 2 (conversational) + Layer 3 (fraud) | Split the two concerns. Fraud detection = Layer 3 intelligence microservice (domain-specific). Conversational data layer = shared Layer 2, not HCM-specific. |
| 20 | PGR integration with Temporal | Cross-module orchestration | Critical. Temporal is the right engine for cross-module workflows. Scope should expand beyond PGR to all five key cross-module flows. |
| 22 | HCM Console AI Copilot | Layer 4 application | Keep as the application pattern. Ensure it calls the shared MCP server + confirmation gate rather than embedding its own AI layer. |

### Indirectly relevant — useful acceleration, not core architecture

| # | Initiative | How it helps |
|---|---|---|
| 4 | AI for development (Claude/Cursor) | Accelerates building everything above |
| 6 | Tech design automation | Could auto-generate interaction diagrams from service code — high-value if it works |
| 12 | Automating HCM implementation tasks | Layer 2 setup intents for HCM — add to orchestrator, don't build separately |

### Not relevant to platform AI architecture

| # | Initiative | Why |
|---|---|---|
| 1 | Test automation | Internal engineering tooling |
| 3 | LCNC exploration | Separate concern |
| 7 | Log anomaly detection | Infrastructure/DevOps, not platform AI |
| 10 | User analytics | Product analytics, not platform AI |
| 11 | Analytics stack modernisation | Data infrastructure, not platform AI |
| 14 | UI module generation | Developer tooling |
| 15 | Prompt library | Marginal |
| 16 | UI test automation | Internal engineering |
| 18 | Reports framework | Product feature |
| 21 | Beckn Onix | Protocol integration, separate concern |
| 23 | HCM debugger | Developer tooling |

---

## The Ask — Precisely

**From the platform team (5 items):**

1. Raise all 7 existing specs to the certificate service standard — human-readable codes, rich descriptions, examples, tenantId from Bearer, idempotency keys
2. Add interaction diagrams (websequencediagrams/Mermaid format) for each major operation — prerequisites, data on arrows, failure alt block
3. Add response enrichment — return code + label in the same response field
4. Configure gateway — bootstrap whitelist + JWT propagation
5. Define audit log schema at the platform level

**From the AI execution layer (3 items — built once):**

1. MCP server generator — auto-generate from specs, add progressive disclosure groups, deploy
2. Confirmation gate — Redis pending action store, confirm endpoint, audit log writer
3. Cross-module Temporal workflows — 5 workflows authored, Temporal as execution engine

**What the 8 relevant AoP teams need to do differently:**

Stop building separate auth, confirmation, and entity resolution in each initiative. Call the shared MCP server + confirmation gate instead. The domain-specific work (HCM fraud detection, PGR intelligence, console UX) is where each team's effort belongs.
