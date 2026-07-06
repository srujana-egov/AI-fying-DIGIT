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

**4. Gateway — Kong (digit3) is already configured**

Kong is the existing API gateway in `digitnxt/digit3`. The core requirements are already met:

| Capability | Status |
|---|---|
| JWT validation (RS256, Keycloak JWKS) | ✓ `dynamic-jwt` Lua plugin |
| Bearer token forwarded to upstream services | ✓ Authorization header passed through unchanged |
| X-Tenant-ID extracted from JWT and injected as header | ✓ `header-enrichment` plugin |
| RBAC enforcement via accesscontrol service | ✓ `rbac` plugin |
| Bootstrap whitelist (`/accounts`, `/keycloak` bypass JWT) | ✓ Configured in `setup.py` |

**What still needs doing:**
- Register MCP server, RAG V5, and n8n as protected services in Kong's `setup.py` (same pattern as workflow, boundary, idgen)
- Add per-tenant, per-agent rate limiting — AI agents call at machine speed; existing limits are designed for human-paced interaction
- Confirm OTP endpoint whitelist behaviour (currently gets JWT + RBAC applied — may need to be whitelisted for unauthenticated OTP flows)

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

**For execution:** n8n (already deployed at eGov) handles 4 of the 5 workflows — Commissioner's Brief, Revenue Recovery, Post-disaster Triage, and Annual Renewal are parallel-reads or single-path writes that n8n covers well. Start a Business requires parallel execution + saga compensation (if Fire NOC fails after Trade License succeeded, roll back Trade License) — Temporal is the right engine for that specific workflow. See [Mini Projects](12-mini-projects-revised.md) section 6 for the workflow-by-workflow breakdown.

---

## The Complete Architecture

Two distinct interaction modes. The diagram shows both.

```
                        INTERACTIVE  (user-initiated)
┌──────────────────────────────────────────────────────────────────┐
│  Any consumer: city admin · state official · AI agent ·          │
│  HCM console · PGR copilot · WhatsApp bot                        │
└──────────────────────────┬───────────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │      Kong  (digit3)      │
              │  JWT validate + fwd  ✓  │
              │  X-Tenant-ID inject  ✓  │
              │  RBAC (accesscontrol)✓  │
              │  Bootstrap whitelist ✓  │
              └──────┬──────┬──────┬────┘
                     │      │      │
          ┌──────────┘      │      └────────────────────────────┐
          │                 │                                  │
  ┌───────▼──────┐  ┌───────▼────────┐    ┌─────────────────────▼──────────────┐
  │   RAG V5     │  │   MCP Server   │    │   n8n                               │
  │              │  │                │    │                                     │
  │  Q&A from    │  │  auto-gen from │    │  cross-module orchestration         │
  │  corpus      │  │  specs         │    │  5 workflows · saga                 │
  │  docs +      │  │  + confirm     │    │                                     │
  │  diagrams    │  │  gate + audit  │    │  sequences / parallelises           │
  │              │  │                │    │  calls across app services          │
  │  no DIGIT    │  │  direct ops    │    │  (each app svc calls platform       │
  │  API calls   │  │  (app +        │    │  services internally)               │
  └──────────────┘  │  platform)     │    └──────────────────────┬──────────────┘
                    └───────┬────────┘                           │
                            │ Bearer token forwarded             │
                            │ Audit log written                  │
                            └────────────────────┬───────────────┘
                                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  Application Services  (certificate standard)                   │
│                                                                 │
│  certificate · pgr · trade-license · bpa · property ·          │
│  fire-noc · water · works · hcm · social-benefits ·            │
│  ifix · waste · mCollect · DRISTI · ...                         │
│                                                                 │
│  each: spec + interaction diagram                               │
└──────────────────────────┬──────────────────────────────────────┘
                           │ each app service calls platform services
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Platform Services  (digit-specs/v3.0.0)                        │
│                                                                 │
│  workflow · individual · boundary · idgen · mdms ·              │
│  notification · filestore · billing · registry · ...           │
└─────────────────────────────────────────────────────────────────┘

            ↑ reads application service records (scheduled)
            │ writes flags back via application service APIs
            │ alert engine sends via DIGIT notification (platform)
            │
┌───────────┴──────────────────────────────────────────────────────
│         INTELLIGENCE LAYER  (scheduled · domain-specific · shared pattern)
│
│  Flagging microservices (one per domain):
│    GIS cross-reference  —  declared use vs satellite land-use
│    Area discrepancy     —  declared sq ft vs satellite measurement
│    Demographic rules    —  age/status/pregnancy field consistency
│    Deduplication        —  on-device (Flutter) + server-side (MOSIP)
│
│  Alert engine (shared):
│    recurrence detector → DIGIT notification → WhatsApp / SMS
│
│  External data:
│    WorldPop · Google Open Buildings · GIS land-use layers
└──────────────────────────────────────────────────────────────────
                    INTELLIGENCE  (scheduled, automated)
```

No semantic layer. No tool registry. No custom intent classifier. No skills for single-service operations.

---

## AoP Projects: Which Are Relevant

The pattern across all relevant AoP initiatives: each team is solving a real domain problem, but is also independently building auth propagation, confirmation UX, audit trail, and entity resolution. These are platform concerns, not domain concerns. Building them once as shared infrastructure means each team only needs to solve the domain-specific part. Building them N times produces N different security postures and N maintenance burdens.

### Directly relevant — should be coordinated under this architecture

| # | Initiative | Role in architecture | Direction |
|---|---|---|---|
| 2 | LLM guidance chatbot | RAG V5 | Keep. Expand knowledge base to include interaction diagrams and process guides. Specs go into the MCP server, not RAG. |
| 5 | MCP for DIGIT PGR | MCP Server | Generalize. Don't build PGR-specific — build the generator that produces MCP from any spec. PGR is the first test case, not the final scope. |
| 8 | Service Config AI Agent | Limited — see note | Reframe. Configuration is one-time per city (not at-scale) and tenant isolation prevents cross-city template borrowing. The confirmation gate is still needed — but for operational write operations (transition approvals, batch renewals), not configuration. |
| 13 | Console config chatbot | RAG V5 + MCP Server | Don't build RAG for config generation — RAG answers questions, it can't execute. Wire RAG V5 (questions) + confirmation gate (actions) with a thin intent router between them. |
| 17 | HCM support chatbot | MCP Server + application interface | Keep. Should call the shared MCP server, not embed its own AI logic. |
| 19 | HCM conversational data layer | MCP Server (conversational) + Intelligence microservice (fraud detection) | Split the two concerns. Fraud detection = domain-specific intelligence microservice. Conversational layer = shared MCP server, not HCM-specific. |
| 20 | PGR integration with Temporal | Cross-module orchestration | n8n handles most workflows (already deployed at eGov). Scope should expand beyond PGR to all five cross-module flows. Temporal specifically for Start a Business saga compensation. |
| 22 | HCM Console AI Copilot | Application layer | Keep as the application pattern. Ensure it calls the shared MCP server + confirmation gate rather than embedding its own AI logic. |

### Indirectly relevant — useful acceleration, not core architecture

| # | Initiative | How it helps |
|---|---|---|
| 4 | AI for development (Claude/Cursor) | Accelerates building everything above |
| 6 | Tech design automation | Could auto-generate interaction diagrams from service code — high-value if it works |
| 12 | Automating HCM implementation tasks | MCP Server setup intents for HCM — add to the shared server, don't build separately |

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

1. Raise all ~18 domain product specs to the certificate service standard — human-readable codes, rich descriptions, examples, tenantId from Bearer, idempotency keys (7 rewrite from Swagger 2.0, ~11 create new)
2. Add interaction diagrams (websequencediagrams/Mermaid format) for each major operation — prerequisites, data on arrows, failure alt block
3. Add response enrichment — return code + label in the same response field
4. Register MCP server, RAG V5, and n8n with Kong (digit3) — add to `setup.py` as protected services; add AI-specific rate limiting
5. Define audit log schema at the platform level

**From the AI execution layer (3 items — built once, shared by all):**

1. MCP server — auto-generate from specs, add progressive disclosure groups, deploy
2. Confirmation gate + audit log writer — Redis pending action store, confirm endpoint, PostgreSQL audit log
3. Cross-module n8n workflows — 5 workflows, n8n-first (already deployed); Temporal specifically for Start a Business saga compensation

**From the intelligence layer (one per domain, shared pattern):**

4. Flagging microservices — one per domain, built to the shared pattern (read records, cross-reference external signal, write flag via DIGIT API)
5. GIS cross-reference engine — Trade License + Property Tax as first domains; WorldPop + Google Open Buildings for HCM (in progress)
6. Alert engine — shared recurrence detector + DIGIT notification integration

**What the relevant AoP teams need to do differently:**

Stop building separate auth, confirmation, and audit trail in each initiative. Call the shared MCP server + confirmation gate instead. The domain-specific work (HCM fraud detection, PGR intelligence, console UX) is where each team's effort belongs.
