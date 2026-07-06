# Two-Level Spec Architecture: Platform Services vs Application Services

The certificate service spec provided as the quality standard is **not** a platform service. It is an **application** built on top of DIGIT's platform services. This distinction matters for every downstream decision: what gets rewritten, what gets improved, and what kind of interaction diagrams each level needs.

---

## The Two Levels

```
Level 2 — Application Services
  Built USING platform services
  Repo: digitnxt/license-certificate, eGov/DIGIT-Works, etc.
  Examples: Certificate, PGR, Trade License, BPA, Property Tax,
            Water & Sewerage, Fire NOC, Birth Registration

  Quality standard: Certificate-3.0.0.yaml
  Current state of old services: Swagger 2.0, RequestInfo wrappers,
                                  opaque codes — need full rewrite

──────────────────────────────────────────────────────────────────

Level 1 — Platform Services
  The foundation everything is built on
  Repo: digitnxt/digit-specs/v3.0.0
  Services (16): workflow · individual · boundary · idgen · mdms ·
    localization · notification · filestore · account · accesscontrol ·
    employee · billing-payment · otp · registry · url-shortener · common

  Current state: Already OpenAPI 3.0.3, BearerAuth, clean design.
  Need improvement, not rewrite.
```

---

## What the Certificate Service Standard Applies To

The certificate service spec (`Certificate-3.0.0.yaml`) is the right template for **Level 2 application services**. It shows how a well-designed DIGIT application looks: it calls platform services (workflow, individual, boundary, form-registry), uses human-readable codes, provides rich descriptions, and follows clean REST conventions.

It is NOT the template for Level 1 platform services. Platform services have a different purpose: they are the primitives that application services compose. Their spec quality standard is about correctness, completeness, and internal behavior documentation — not about cross-service orchestration.

---

## What Each Level Needs

### Level 1 — Platform Services (improve, don't rewrite)

These are already at a good baseline. The gaps:

| Gap | What's missing | Fix |
|---|---|---|
| Examples | Few realistic examples per endpoint | Add 2-3 examples per write operation |
| Idempotency | No `Idempotency-Key` header | Add to all write operations |
| Internal behavior | What happens inside the service on each call? | Internal interaction diagrams |
| Cross-service dependencies | Does workflow call notification internally? | Document in description field |
| Error semantics | Some services use generic errors | Add typed error responses |

Platform services do **not** need:
- Removal of their existing design (it's already clean)
- Rewrite to certificate spec format (different level, different concerns)
- Cross-service orchestration diagrams (that's Level 2's concern)

### Level 2 — Application Services (rewrite, don't improve)

The old application specs (pgr.yml, tl-service.yml, bpa.yaml, water-sewerage.yaml, property-services.yml, fire-noc.yaml, birth-registration.yaml) are 2.9-era Swagger 2.0 files. The gap is too large for incremental fixes. Full rewrite to certificate service standard.

What the rewrite delivers:
- OpenAPI 3.0.3 (not Swagger 2.0)
- No RequestInfo wrapper
- tenantId from Bearer token
- Human-readable codes throughout
- Rich descriptions with business logic
- Realistic examples per endpoint
- Idempotency-Key on all writes
- Typed error responses (409, 403, 404, 422)
- Cross-service dependencies named explicitly

---

## Interaction Diagrams at Two Levels

Both levels need diagrams. They show different things.

### Level 1 — Platform Internal Diagrams

Question answered: **How does this platform service work internally?**

Example — workflow service processing a transition:

```
title WorkflowService.executeTransition — Internal Processing

participant Caller
participant WorkflowService
participant StateEngine
participant RoleValidator
participant NotificationService
participant AuditLog

Caller->WorkflowService: POST /workflow/v3/transition
  (processCode, entityId, action, comment, assignee)

WorkflowService->StateEngine: currentState(entityId)
StateEngine-->WorkflowService: PENDING_VERIFICATION

WorkflowService->StateEngine: validTransitions(PENDING_VERIFICATION)
StateEngine-->WorkflowService: [APPROVE, REJECT, REQUEST_MORE_INFO]

WorkflowService->RoleValidator: canPerform(userId, action=APPROVE, processCode)
RoleValidator-->WorkflowService: authorized / 403

alt 403 Unauthorized
  WorkflowService-->Caller: 403 Forbidden
end

WorkflowService->StateEngine: executeTransition(entityId, APPROVE)
StateEngine-->WorkflowService: newState=APPROVED, workflowInstanceId

WorkflowService->NotificationService: notify(assignee, entityId, APPROVED)
WorkflowService->AuditLog: record(userId, entityId, PENDING→APPROVED)

WorkflowService-->Caller: 200 WorkflowInstance
  (entityId, currentState=APPROVED, workflowInstanceId)
```

These diagrams live in the platform spec repo (digit-specs/v3.0.0) alongside each service spec. ~2-3 diagrams per platform service = ~35-48 diagrams total.

---

### Level 2 — Application Cross-Service Diagrams

Question answered: **How does this application orchestrate platform services?**

Example — certificate service applying for a certificate:

```
title CertificateService.applyForCertificate — Cross-Service Integration

note over Client,FileStore: Prerequisites:
  - certificateType.isActive = true
  - certificateType.definitionVersion configured in FormRegistry
  - applicant exists in Individual Registry

participant Client
participant CertificateService
participant FormRegistry
participant IDGen
participant WorkflowService
participant FileStore

Client->CertificateService: POST /certificate-types/{code}/certificates
  (applicantIds, holderIds, documents, certificateDetail)

CertificateService->FormRegistry: validateDefinition(definitionVersion, certificateDetail)
FormRegistry-->CertificateService: valid / 400 schema error

CertificateService->IDGen: generate(CERT_APP)
IDGen-->CertificateService: applicationNumber

CertificateService->WorkflowService: initiate(CERT_APPLY, applicationNumber)
WorkflowService->WorkflowService: Validate State, Role, Role-Action
WorkflowService-->CertificateService: workflowInstanceId [201] / [500]

alt WorkflowService returns 500
  CertificateService->IDGen: release(applicationNumber)
  CertificateService-->Client: 500
end

CertificateService->FileStore: validate(documents[])
FileStore-->CertificateService: valid / 400

CertificateService-->Client: 201 Certificate
  (applicationNumber, workflowInstanceId, status=IN_PROGRESS)
```

These diagrams live in the application repo (license-certificate, PGR, etc.) alongside the service spec. ~3-4 diagrams per application service.

---

## How This Affects the MCP Server

The MCP server spans **both levels**. An AI agent needs tools from both:

```
MCP Tool Sources:

Platform tools (from digit-specs/v3.0.0):
  workflow_get_transitions      — "what can I do next with this entity?"
  workflow_execute_transition   — "approve this application"
  individual_search             — "find this person in the registry"
  boundary_get                  — "what jurisdiction is this?"
  idgen_generate                — (internal — used by application services)
  mdms_search                   — "what certificate types are active?"
  filestore_upload              — "store this document"

Application tools (from Certificate-3.0.0.yaml, rewritten pgr.yaml, etc.):
  certificate_apply             — "apply for a trade license"
  certificate_transition        — "approve / reject this application"
  certificate_renew             — "renew before expiry"
  pgr_complaint_create          — "file a grievance"
  pgr_complaint_transition      — "assign / resolve this complaint"
```

Progressive disclosure groups map to both levels:

```
core (loads on startup):
  certificate_list, certificate_get,    ← application tools
  workflow_get_transitions,             ← platform tool
  individual_search, boundary_get,      ← platform tools
  mdms_search, enable_tools

operations (unlock for city admins):
  certificate_apply, certificate_transition, certificate_renew,
  pgr_complaint_create, pgr_complaint_transition

setup (unlock for implementation teams):
  certificate_type_create, workflow_definition_create,
  idgen_configure, boundary_hierarchy_create

state (unlock for state officials):
  certificates_search_cross_type, analytics_aggregate
```

---

## Summary: Two Tracks for Two Levels

| Decision | Level 1 (Platform) | Level 2 (Application) |
|---|---|---|
| Current state | Good — OpenAPI 3.0.3, clean design | Poor — Swagger 2.0, opaque, no examples |
| Action needed | Improve (examples, idempotency, diagrams) | Rewrite to certificate standard |
| Quality template | Platform spec conventions | Certificate-3.0.0.yaml |
| Interaction diagrams | Internal behavior diagrams | Cross-service orchestration diagrams |
| Count (diagrams) | ~35-48 (~2-3 per service × 16) | ~28 (~4 per service × 7) |
| MCP tools generated | Platform tools (workflow, individual, etc.) | Application tools (certificate, pgr, etc.) |
| Owner | Platform team | Platform team + service owners |

**The certificate service is the quality standard for Level 2. The v3.0.0 platform specs define the standard for Level 1. These are different things.**
