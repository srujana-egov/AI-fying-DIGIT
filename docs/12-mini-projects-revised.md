# Mini Projects — Revised Under the Certificate Service Standard

This replaces the earlier mini projects doc. Based on the conversation with the principal architect and the certificate service spec as the target standard.

---

## 1. Raise All Specs to the Right Standard

There are **two levels of specs**, and they need different things. See [13 — Two-Level Spec Architecture](13-two-level-spec-architecture.md) for the full picture.

---

### P1a — Platform Services: Improve (not rewrite)

**Owner:** Platform team  
**Effort:** ~1 week per service  
**Services:** The 16 in `digitnxt/digit-specs/v3.0.0` — workflow, individual, boundary, idgen, mdms, localization, notification, filestore, account, accesscontrol, employee, billing-payment, otp, registry, url-shortener, common

These are already OpenAPI 3.0.3 with clean design. The gap is not a rewrite — it is targeted additions:

| Gap | What to add |
|---|---|
| Examples | 2-3 realistic examples per write operation |
| Idempotency | `Idempotency-Key` header on all writes |
| Internal cross-service deps | Document in description field (e.g. "workflow service calls notification service on every transition") |
| Error semantics | Typed responses where still generic (409, 403, 404, 422) |

**Checklist per platform spec:**
```
□ 2+ realistic examples per write endpoint
□ Idempotency-Key header on all write operations
□ Description field names any internal service calls
□ Error responses: 400, 403, 404, 409, 500 — all typed
□ No opaque codes in response fields
```

---

### P1b — Application Services: Rewrite to Certificate Standard

**Owner:** Platform team + service owners  
**Effort:** ~3-4 weeks per service  
**Scope:** All ~18 DIGIT domain products (20 total minus Dashboards & Analytics as UI-only, minus DIGIT platform itself)

Two tracks within P1b:
- **Rewrite** (7 exist as old Swagger 2.0 in digit-specs): tl-service, bpa, property-services, fire-noc, water-sewerage, pgr, birth-registration
- **Create new** (~11 products with no OpenAPI spec or spec not yet at certificate standard): Works Management, HCM, 10 Bed ICU, DIVOC, Social Benefits, iFix, Waste Management, Water Supply O&M, Water Schemes O&M, mCollect, DRISTI

These are Swagger 2.0 files. The gap is too large for incremental fixes. Full rewrite using `Certificate-3.0.0.yaml` (from `digitnxt/license-certificate`) as the template.

| Property | Current old specs | Certificate standard |
|---|---|---|
| OpenAPI version | Swagger 2.0 / early OpenAPI 3.0 | OpenAPI 3.0.3 |
| Request pattern | POST-only with `RequestInfo` wrapper in body | REST: GET reads, POST writes, no wrapper |
| tenantId | Query parameter in every call | Resolved from Bearer token |
| Codes | Opaque: integer IDs, short codes | Human-readable: `LICENSE`, `TRADE`, `INSTITUTION` |
| Descriptions | 1-2 sentences | Business logic, validation rules, 3-10 lines |
| Examples | None | Realistic examples for every endpoint |
| Idempotency | None | `Idempotency-Key` header on all writes |
| Error semantics | Generic `ErrorRes` | Typed: 409 duplicate, 403 inactive, 404 not found, 422 validation |
| Cross-service deps | Not mentioned | Named explicitly ("calls IDGen, Workflow, FileStore") |

**Checklist per application spec:**
```
□ OpenAPI 3.0.3 (not Swagger 2.0)
□ No RequestInfo wrapper in request body
□ tenantId from Bearer token — not a body or query field
□ All codes are human-readable strings (not integers)
□ Every operation has: summary (1 line) + description (business logic, 3-10 lines)
□ Every write endpoint has at least 2 realistic examples
□ Every write endpoint has Idempotency-Key header
□ Error responses: 400, 403, 404, 409, 500 — all typed
□ Cross-service dependencies named in the description field
□ Response fields are human-readable
```

**AI team's contribution:** Can run an automated gap analysis script against each spec and produce a checklist report. Speeds up the platform team's work.

---

## 2. Interaction Diagrams

**Owner:** Platform team  
**Effort:** 2-3 days per service (after specs are done)  
**Format:** websequencediagrams text source (agreed format) — NOT PNG, NOT drawn

Interaction diagrams are needed at **both levels**, but they answer different questions.

---

### P2a — Platform Internal Diagrams

**Question:** How does this platform service work internally?

These show the internal behavior of each platform service — what happens inside workflow when a transition is executed, what happens inside idgen when a number is generated. They live in `digit-specs/v3.0.0` alongside each service spec.

Not every operation. Only operations with non-trivial internal logic:

| Service | Operations needing diagrams |
|---|---|
| workflow | executeTransition (state validation, role check, notification trigger), getProcessDefinition |
| individual | create (duplicate check, ID assignment), search |
| idgen | generate (format resolution, sequence increment, lock/release) |
| filestore | upload (virus scan, storage, ID assignment) |
| mdms | search (hierarchy traversal), create |
| localization | resolve (fallback chain) |
| boundary | getHierarchy, search |
| account | create (OTP flow, initial role assignment) |

~2-3 diagrams per platform service × 16 services = **~35-48 platform diagrams total**

---

### P2b — Application Cross-Service Diagrams

**Question:** How does this application orchestrate platform services?

These show how application services (certificate, PGR, trade license, etc.) call platform services in sequence. They live alongside the application spec. This is the diagram type shown in the reference example.

Not every endpoint. Only operations with cross-service calls:

| Operation type | Needs diagram? | Why |
|---|---|---|
| Apply / Create | Yes | Calls IDGen + Workflow + FileStore |
| Transition / Update status | Yes | Calls Workflow + Notification |
| Approve / Reject | Yes | Calls Workflow + Notification + (sometimes Payment) |
| Renew | Yes | Calls Workflow + IDGen (new certificate number) |
| Search / List | Usually no | Single service, no cross-service calls |
| Get by ID | No | Single service |

~3-4 diagrams per product × ~18 products = **~55-70 application diagrams total**

### Exact template required for AI to understand everything

The diagram must answer three questions or it is insufficient for AI:

```
1. BEFORE — what must be true before this operation can be called?
2. DURING — which services, in what order, what data flows between them?
3. ON FAILURE — which step can fail and what gets rolled back?
```

**Exact template (copy this for every operation):**

```
title {ServiceName}.{operationId} — Cross-Service Integration

note over Client,{LastParticipant}: Prerequisites:
  - {condition 1, e.g. certificateType.isActive = true}
  - {condition 2, e.g. applicant registered in Individual Registry}

participant Client
participant {ServiceName}
participant IDGen
participant WorkflowService
participant FileStore
participant NotificationService

Client->{ServiceName}: {HTTP method} {path}\n({key input fields})

{ServiceName}->IDGen: generate({format})\nIDGen-->{ServiceName}: {assignedField}

{ServiceName}->WorkflowService: initiate({processCode}, {entityId})\nWorkflowService->WorkflowService: Validate State, Role, Role-Action\nWorkflowService-->{ServiceName}: workflowInstanceId [201] / [500]

alt WorkflowService returns 500
  {ServiceName}->IDGen: release({assignedField})
  {ServiceName}-->Client: 500 Internal Server Error
end

{ServiceName}->FileStore: validate(documents[])\nFileStore-->{ServiceName}: valid / 400

{ServiceName}-->Client: {HTTP status} {ResponseSchema}\n({key output fields: assignedField, workflowInstanceId, status})
```

**Filled example — applyForCertificate:**

```
title CertificateService.applyForCertificate — Cross-Service Integration

note over Client,NotificationService: Prerequisites:
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

### What this gives AI (with nothing else)

Given spec + this diagram, the AI knows:
- Which tool to call for any given user intent (from spec description + operationId)
- What parameters are required (from spec schema + examples)
- What conditions must be met first (from diagram prerequisites)
- What happens if it fails (from diagram alt block)
- What values come back (from diagram response line)

**Nothing else is needed for the AI to understand the platform.**

---

## 3. MCP Server — Is It Still Needed?

**Yes — but it is a build step, not a project.**

Specs + diagrams give the AI *understanding*. The MCP server gives the AI *execution*. These are different things.

| What specs + diagrams provide | What MCP server provides |
|---|---|
| What to call and why | Ability to actually call it |
| Parameter schemas | Auth forwarding (Bearer token) |
| Cross-service sequence | Confirmation gate for writes |
| Error handling logic | Audit log per operation |
| | Progressive disclosure grouping |

**How to build it — concrete steps:**

```
Step 1 (1 day): Install tooling
  npm install @modelcontextprotocol/sdk @anthropic-ai/sdk
  npm install -g @openapitools/openapi-generator-cli

Step 2 (2-3 days): Generate tool definitions from specs
  for each spec in digit-specs/*.yaml:
    openapi-generator-cli generate \
      -i {spec}.yaml \
      -g typescript-fetch \
      -o ./generated/{service}
  
  → Produces typed HTTP client per service

Step 3 (2 days): Wrap generated clients as MCP tools
  for each generated operation:
    server.tool(
      operationId,           // from spec
      description,           // from spec — already AI-readable
      inputSchema,           // from spec parameters + requestBody
      async (params, ctx) => {
        return await {service}Client.{operation}(
          params,
          { Authorization: `Bearer ${ctx.bearerToken}` }
        )
      }
    )

Step 4 (1 day): Add progressive disclosure
  const GROUPS = {
    core: ['certificate_list', 'certificate_get', 
           'workflow_get_transitions', 'individual_search',
           'boundary_get', 'mdms_search', 'enable_tools'],
    operations: ['certificate_apply', 'certificate_transition', ...],
    setup: ['certificate_type_create', 'workflow_definition_create', ...],
    state: ['certificates_search_cross_type', ...]
  }
  // load only core on startup, unlock on demand

Step 5 (1 day): Add confirmation gate
  // intercept all write operations before executing
  if (isMutation(operationId)) {
    await redis.set(`pending:${sessionId}`, JSON.stringify({
      operationId, params, timestamp
    }), 'EX', 600) // 10 min TTL
    return { confirm: true, description: humanReadable(operationId, params) }
  }

Step 6 (1 day): Add audit logging
  // after every confirmed execution, write:
  await auditLog.write({ userId, tenantId, sessionId, operationId,
                         params, httpStatus, result, timestamp })

Step 7 (3-4 days): Test against real DIGIT instance
```

**Total: ~2 weeks for 1 developer.**  
Not a research project. A build project with clear inputs (specs) and clear outputs (running MCP server).

---

## 4. Is EGOV RAG V5 Good Enough for Specs + Interaction Diagrams?

**Short answer: RAG V5 works for its current purpose. It needs adaptation for specs. For AI agent operation, you don't need RAG at all.**

### Why standard RAG doesn't work well for OpenAPI specs

RAG V5 chunks documents by paragraph/character count. An OpenAPI spec doesn't chunk that way — one endpoint's definition spans 50-200 lines of YAML and must stay together to be meaningful. If a chunk breaks mid-schema, the retrieved chunk is useless.

```
What RAG V5 does:
  spec YAML (2000 lines)
    → chunk by ~500 chars
    → chunk 3 contains half of requestBody schema
    → chunk 4 contains the other half + response schema
    → neither chunk is useful alone

What specs need:
  spec YAML (2000 lines)
    → chunk by operationId (keep each endpoint together)
    → chunk = {operationId, summary, description, parameters, 
               requestBody, responses, examples}
    → each chunk is self-contained and meaningful
```

### For AI agent operation — RAG not needed at all

When the AI agent uses MCP tools, it doesn't need to search the specs. The MCP server already has every spec description embedded in each tool's definition. The LLM reads tool descriptions and selects the right tool — no retrieval needed.

### Where RAG V5 IS the right tool

| Query type | Right tool |
|---|---|
| "How do I configure trade license?" | RAG V5 — documentation query |
| "What's the renewal process for BPA?" | RAG V5 — process question |
| "Apply for trade license for my shop" | MCP server — agent action |
| "Show all IN_PROGRESS licenses" | MCP server — agent query |

RAG V5 handles the "how do I" questions. MCP handles the "do this" commands. A thin router between them (is this a question or a command?) connects the two.

### What needs changing in RAG V5 for specs + diagrams

1. **Endpoint-aware chunking**: chunk by `(service, operationId)`, not by character count
2. **Structured metadata**: index `service`, `operationId`, `tags`, `method`, `path` as filterable fields
3. **Interaction diagram ingestion**: diagrams are short enough to keep whole — ingest each as one document with metadata `{service, operationId, type: interaction_diagram}`

**Effort:** 1-2 weeks to add endpoint-aware chunking and re-ingest all specs.

---

## 5. (Covered in section 3 above — MCP build steps)

---

## 6. Temporal — Use Case and Open Questions

### What Temporal is

Open-source workflow orchestration engine. Handles:
- Durable execution (if server crashes mid-workflow, it resumes exactly where it stopped)
- Parallel execution (run Trade License + Fire NOC + Water Connection simultaneously, wait for all)
- Automatic retry with backoff (DIGIT API returns 503 → Temporal retries, not your code)
- Compensation (if step 3 fails, automatically run rollback for steps 1 and 2)
- Long-running workflows (a permit approval process that takes days)

### The 5 cross-module workflows that need Temporal

| Workflow | Services | Why Temporal |
|---|---|---|
| Start a business | TL + Fire NOC + Water (parallel) | Parallel execution + wait-for-all + partial failure compensation |
| Annual renewal campaign | Certificate × all active holders | Long-running, thousands of parallel calls, resume on failure |
| Revenue recovery | Property Tax + Water & Sewerage | Cross-module query + action sequence with state |
| Post-disaster triage | PGR + Boundary + Assignment | Time-sensitive, parallel ward processing |
| Commissioner's brief | PGR + PT + W&S + HCM | Parallel reads + aggregation |

### What a Temporal workflow looks like for "Start a Business"

```python
@workflow.defn
class StartBusinessWorkflow:
    @workflow.run
    async def run(self, params: StartBusinessParams):
        
        # Present to human for confirmation before any writes
        await workflow.execute_activity(
            confirm_with_human,
            args=["Apply for Trade License, Fire NOC, and Water Connection in parallel?"],
            start_to_close_timeout=timedelta(minutes=10)
        )
        
        # Run all three in parallel — wait for all to complete
        results = await asyncio.gather(
            workflow.execute_activity(apply_for_certificate, 
                args=["trade-license", params.tradeLicenseData]),
            workflow.execute_activity(apply_for_certificate,
                args=["fire-noc", params.fireNocData]),  
            workflow.execute_activity(apply_for_water_connection,
                args=[params.waterData])
        )
        
        return {
            "tradeLicense": results[0],
            "fireNoc": results[1], 
            "waterConnection": results[2],
            "status": "all applications submitted"
        }
```

If the Fire NOC activity fails halfway, Temporal automatically retries it. If it permanently fails, it runs the compensation (cancel Trade License + Water Connection that already succeeded).

### Open questions for the architecture review

1. **Infrastructure:** Is Temporal already deployed in DIGIT's infrastructure, or does it need to be set up? Self-hosted Temporal or Temporal Cloud?
2. **Confirmation gate placement:** Should the human YES/NO confirmation sit inside the Temporal workflow (as a human-in-the-loop activity) or outside it (before the workflow is triggered)? This affects the UX significantly.
3. **Priority workflow:** Of the 5 cross-module workflows, which should be built first? "Start a business" is the most visible. "Commissioner's brief" is read-only and safest to start with.
4. **Row 20 in AoP (PGR integration with Temporal):** This is currently scoped to PGR only. Should it be expanded to be the general Temporal setup for all 5 workflows? This existing work is the foundation — scope expansion now saves rebuilding later.

---

## 7. Are the Certificate-Standard Responses Human-Readable? Is Semantic Layer Still Needed?

### What the current specs return (example — pgr.yml search response)

```json
{
  "serviceCode": "NoStreetLight",
  "applicationStatus": "PENDINGFORASSIGNMENT", 
  "serviceRequestId": "PGR-2024-001234",
  "tenantId": "pb.amritsar",
  "accountId": "8a2c4a74-11e5-4c3e-9d9f-f7f4c0b1b9a3"
}
```

`serviceCode`, `applicationStatus`, `accountId` — opaque to a human or AI reading this response.

### What certificate service standard responses return

```json
{
  "instrumentType": "LICENSE",
  "sector": "TRADE", 
  "status": "IN_PROGRESS",
  "certificateTypeCode": "trade-license",
  "applicationNumber": "CERT-APP-2026-001234"
}
```

All primary fields are human-readable. What's still opaque:
- `applicantIds: ["IND-8a2c4a74-..."]` — Individual Registry UUID
- `holderIds: ["ORG-CPR-2020-007890"]` — Institution Registry UUID
- `workflowInstanceId: "WF-INS-..."`
- `fileStoreId: "fs-id-001"`

### Does this require a semantic layer?

**No — for two reasons:**

**Reason 1: Response enrichment** (cleanest — platform team decision)  
Embed the resolved entity inline in the response:
```json
{
  "applicantIds": ["IND-8a2c4a74-..."],
  "_applicant": { "name": "Ramesh Kumar", "mobile": "9876543210" }
}
```
AI gets everything it needs in one call. No semantic layer, no follow-up call.

**Reason 2: AI multi-hop** (if response enrichment isn't added)  
AI agent calls `individual_search` as a second tool call to look up the applicant by ID. Two tool calls instead of one. Works fine — this is standard agentic behavior. No semantic layer component needed.

### Verdict

| Need | Certificate standard spec | Semantic layer needed? |
|---|---|---|
| Human-readable codes | Yes — `LICENSE`, `TRADE`, `INSTITUTION` | No |
| Cross-entity joins (name from ID) | Partial — needs enrichment or multi-hop | No |
| Tenant hierarchy (pb.amritsar → pb) | Not in spec | No — AI calls boundary service |
| Localization (English ↔ other languages) | Not in scope for AI layer | No — UI concern |

**Semantic layer as a standalone component: eliminated.**  
The remaining cross-entity needs are handled by either response enrichment (platform decision) or AI multi-hop tool calls (agent behavior). Neither requires a separate service.

---

## Summary — The Revised Mini Project List

| # | Project | Level | Owner | Effort | Blocks |
|---|---|---|---|---|---|
| P1a | Improve 16 platform specs (examples, idempotency, internal deps) | Platform | Platform team | 1 wk/service | P2a, P3 |
| P1b | Application specs for all ~18 domain products (7 rewrite, ~11 create new) | Application | Platform team + service owners | 3-4 wks/service | P2b, P3 |
| P2a | Platform internal diagrams (~35-48 total, ~2-3 per service) | Platform | Platform team | 1-2 days/service | MCP quality, RAG quality |
| P2b | Application cross-service diagrams (~28 total, ~4 per service) | Application | Platform team | 2-3 days/service | MCP quality, RAG quality |
| P3 | MCP server (auto-generate from specs — both levels) | AI Execution | AI team, 1 developer | 2 weeks | P1a, P1b |
| P4 | Confirmation gate | AI Execution | AI team, 1 developer | 1 week | P3 |
| P5 | Audit log writer | AI Execution | AI team, 1 developer | 3 days | P3 |
| P6 | RAG V5 — endpoint-aware chunking + spec ingestion | AI Execution | Platform team | 1-2 weeks | P1a, P1b, P2a, P2b |
| P7 | Temporal setup + 5 cross-module workflows | AI Execution | AI team, 1-2 developers | 4-6 weeks | P1b, P3 |

**What's eliminated vs the original mini project list:**
- Semantic layer → eliminated
- Custom intent classifier → eliminated (LLM tool selection handles this)
- Hand-authored tool registry → eliminated (auto-generate from specs)
- Skills for single-service operations → eliminated (interaction diagrams + specs)

**What's new vs the original list:**
- Interaction diagrams (P2) → added, owned by platform team
- Temporal (P7) → replaces vague "cross-module orchestration" with a specific engine
- RAG adaptation (P6) → added, endpoint-aware chunking for specs
