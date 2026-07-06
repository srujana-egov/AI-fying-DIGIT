# Mini Projects — Revised Under the Certificate Service Standard

This replaces the earlier mini projects doc. Based on the conversation with the principal architect and the certificate service spec as the target standard.

---

## 1. Raise All Specs to the Right Standard

There are **two levels of specs**, and they need different things. See [05 — Two-Level Spec Architecture](05-two-level-spec-architecture.md) for the full picture.

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

An automated gap analysis script can run against each spec and produce a checklist report — speeds up the platform team's review.

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
  // Kong (digit3) has already validated the JWT before this request
  // arrives. MCP just forwards the Authorization header it receives.
  for each generated operation:
    server.tool(
      operationId,           // from spec
      description,           // from spec — already AI-readable
      inputSchema,           // from spec parameters + requestBody
      async (params, ctx) => {
        return await {service}Client.{operation}(
          params,
          { Authorization: ctx.headers['authorization'] } // forward, don't re-validate
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

### How thin is the MCP server with certificate-standard specs?

With specs at certificate standard, almost every decision in the MCP server is auto-derived:

| Component | With Swagger 2.0 specs (2.9 approach) | With certificate-standard specs |
|---|---|---|
| Tool name | Hand-authored per tool | `operationId` from spec |
| Tool description | Hand-authored per tool | `description` from spec — already AI-readable |
| Input schema | Hand-authored per tool | `parameters` + `requestBody` from spec |
| Auth handling | Custom per-service | Bearer token forwarded uniformly |
| Cross-service logic | Encoded in tool | In spec descriptions + interaction diagrams |
| Progressive disclosure groups | One config file | One config file (~20 lines JSON) |
| Confirmation gate | ~100 lines middleware | ~100 lines middleware |
| Audit log | ~30 lines | ~30 lines |

**Total hand-authored code: ~300-400 lines.** Everything else is generated from specs.

The 2.9 MCP experiment needed 61 hand-authored tools because the specs were not self-describing. With certificate-standard specs, those same tools generate automatically. The platform team's investment in spec quality eliminates the hand-authoring investment in the MCP server — permanently.

### What happens if there is a future agentic AI standard or MCP v2?

The MCP server is a transport layer. The permanent investment is the specs.

**What changes if the protocol changes:**
- The MCP server code (the transport wrapper) — swap in 1-2 weeks
- The generated client code — run `openapi-generator-cli` again with a new output target

**What does NOT change:**
- The spec descriptions — tool names, descriptions, schemas are pure data from the YAML files
- The generated HTTP clients — they call DIGIT APIs directly, not the MCP protocol
- The confirmation gate logic — pure business rule, protocol-agnostic
- The audit log schema — governance requirement, survives any protocol change
- The progressive disclosure group config — pure configuration

If MCP is replaced by a future standard (Google/Microsoft A2A protocol, an OpenAPI-native agent standard, or any future Anthropic agent protocol), the migration is a transport swap. The specs are the permanent investment. The MCP wrapper is the temporary binding — important now, cheap to replace later.

**If future AI models can read OpenAPI specs natively** without a tool layer, the MCP server shrinks further — to just auth forwarding + confirmation gate + audit log. The AI navigates the spec directly. Confirmation gate and audit log survive this future regardless. They are governance requirements, not protocol choices.

---

## 4. RAG V5 — Retrieval Architecture Unchanged, Diagram Ingestion Needed

**The retrieval architecture (hybrid vector + BM25 + Reciprocal Rank Fusion) is untouched. Specs do not go into RAG — they go into the MCP server via openapi-generator. What RAG needs is the interaction diagrams.**

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

### What needs building in RAG V5

**Current state of the actual repo** (`srujana-egov/EGOV_RAG_V5`):
- Chunking is entirely manual. Developers hand-write chunk text as Python dicts or JSONL files.
- Schema: `id`, `document`, `embedding`, `section` (one of: FAQ, User Stories, Architecture, Overview, General).
- Corpus: DIGIT Studio docs only (`studio_chunks.jsonl`, `normalized_chunks.jsonl`). No interaction diagram content.
- Retrieval: hybrid vector + BM25 via RRF, with section-hint keyword filter.

Specs go into the MCP server, not RAG. What RAG needs is interaction diagrams — so implementation teams can ask "what's the sequence for applying for a certificate?" and get the right answer.

**Step 1 — Write the diagram ingester** (`ingest_diagrams.py`, ~20 lines):
Each interaction diagram file becomes one chunk. Diagrams are short enough to stay whole.

```python
def ingest_diagram(diagram_path: Path, service: str, operation_id: str):
    content = diagram_path.read_text()
    return {
        'id': f'diagram/{service}/{operation_id}',
        'document': f"Interaction diagram for {service}.{operation_id}:\n\n{content}",
        'section': 'Architecture',
        'service': service,
        'operation_id': operation_id,
    }
```

**Step 2 — Test retrieval** (1 day): Query "what happens when workflow transition fails" → verify it retrieves the correct diagram. Add process guides and MDMS config references as further corpus expansion.

**Effort:** 3-5 days. The diagram ingester is straightforward. Most time is testing retrieval quality.

---

## 5. P4 — Confirmation Gate *(component inside the MCP server, not a separate service)*

**~100-150 lines. Redis-backed. Intercepts all write operations before execution.**

The digit-ai-orchestrator experiment already proved this pattern works: store a `pendingAction`, return a plain-English description, wait for YES/NO, then execute or discard. The MCP server version is the same pattern, generalized to all operations.

### Components

```
1. Redis pending action store (TTL 10 minutes)
2. is_mutation() check — reads pass through, writes are intercepted
3. intercept() — store action, return confirmation prompt
4. confirm() — YES → execute → audit → clear | NO → clear
5. human_readable() — convert operationId + params to plain English
```

### Implementation

```python
import json, redis.asyncio as redis
from datetime import datetime

class ConfirmationGate:
    def __init__(self, redis_client):
        self.redis = redis_client

    async def intercept(self, operation_id: str, params: dict, session_id: str) -> dict | None:
        if not self.is_mutation(operation_id):
            return None  # reads pass through — no interception

        pending = {
            "operationId": operation_id,
            "params": params,
            "timestamp": datetime.utcnow().isoformat(),
            "sessionId": session_id
        }
        await self.redis.set(
            f"pending:{session_id}",
            json.dumps(pending),
            ex=600  # 10 min TTL — user walks away, no orphaned action
        )
        return {
            "confirm": True,
            "description": self.human_readable(operation_id, params),
            "warning": "This will modify data. Reply YES to proceed, NO to cancel."
        }

    async def confirm(self, session_id: str, answer: str) -> dict:
        raw = await self.redis.get(f"pending:{session_id}")
        if not raw:
            return {"error": "No pending action, or it expired (10 min timeout)."}
        
        pending = json.loads(raw)
        await self.redis.delete(f"pending:{session_id}")

        if answer.strip().upper() != "YES":
            return {"cancelled": True, "operationId": pending["operationId"]}

        return {"approved": True, "pending": pending}

    def is_mutation(self, operation_id: str) -> bool:
        # Writes by convention: any operationId starting with these prefixes
        mutation_prefixes = [
            'create', 'apply', 'update', 'delete', 'transition',
            'renew', 'approve', 'reject', 'assign', 'cancel',
            'submit', 'initiate', 'register'
        ]
        op = operation_id.lower()
        return any(op.startswith(p) for p in mutation_prefixes)

    def human_readable(self, operation_id: str, params: dict) -> str:
        # Convert to plain English — this is the one authored part (~20 lines)
        # Example: applyForCertificate + {type: "trade-license"} →
        # "Apply for a trade-license certificate"
        descriptions = {
            'applyForCertificate':    lambda p: f"Apply for a {p.get('type', '')} certificate",
            'transitionCertificate':  lambda p: f"Transition certificate {p.get('applicationNumber', '')} to {p.get('action', '')}",
            'createComplaint':        lambda p: f"File a grievance: {p.get('description', '')[:60]}...",
            'assignComplaint':        lambda p: f"Assign complaint {p.get('serviceRequestId', '')} to {p.get('assignee', '')}",
        }
        formatter = descriptions.get(operation_id)
        if formatter:
            return formatter(params)
        return f"Call {operation_id} with {json.dumps(params)[:100]}"
```

### Wiring into the MCP server

In Step 5 of the MCP build (from Section 3 above), the tool handler becomes:

```python
async def tool_handler(operation_id: str, params: dict, ctx):
    # Intercept writes
    gate_result = await confirmation_gate.intercept(operation_id, params, ctx.session_id)
    if gate_result:
        return gate_result  # returns confirm: true + description to user

    # On YES, confirm() is called separately via a confirm_action tool:
    # server.tool("confirm_action", ..., async (params, ctx) => {
    #   result = await confirmation_gate.confirm(ctx.session_id, params.answer)
    #   if result.approved:
    #     return await execute_and_audit(result.pending, ctx)
    # })

    # Reads execute directly
    return await execute_operation(operation_id, params, ctx.bearer_token)
```

**Effort: 3-4 days.** The pattern is proven. The main authored part is `human_readable()` — adding plain-English descriptions for each operationId as the spec count grows.

---

## 5b. P5 — Audit Log Writer *(component inside the MCP server, not a separate service)*

**~30 lines of implementation code. PostgreSQL-backed. Written after every confirmed write.**

### Schema

```json
{
  "timestamp": "2026-07-06T10:23:45Z",
  "userId": "user-uuid-from-jwt",
  "tenantId": "pb.amritsar",
  "sessionId": "uuid-v4",
  "userQuery": "Apply for a trade license for my shop at 45 Gandhi Nagar",
  "operationId": "applyForCertificate",
  "toolName": "certificate_apply",
  "confidence": 0.94,
  "proposedAction": {
    "method": "POST",
    "path": "/certificate-types/trade-license/certificates",
    "params": { "...": "..." }
  },
  "humanConfirmation": "yes",
  "idempotencyKey": "uuid-v4",
  "httpStatus": 201,
  "durationMs": 342,
  "result": { "applicationNumber": "CERT-APP-2026-001234" }
}
```

### DDL

```sql
CREATE TABLE ai_audit_log (
  id                  BIGSERIAL PRIMARY KEY,
  timestamp           TIMESTAMPTZ NOT NULL,
  user_id             TEXT NOT NULL,
  tenant_id           TEXT NOT NULL,
  session_id          UUID NOT NULL,
  user_query          TEXT,
  operation_id        TEXT NOT NULL,
  tool_name           TEXT,
  confidence          FLOAT,
  proposed_action     JSONB NOT NULL,
  human_confirmation  TEXT CHECK (human_confirmation IN ('yes', 'no', 'timeout')),
  idempotency_key     UUID,
  http_status         INTEGER,
  duration_ms         INTEGER,
  result              JSONB
);

CREATE INDEX ai_audit_user_time      ON ai_audit_log (user_id, timestamp);
CREATE INDEX ai_audit_tenant_time    ON ai_audit_log (tenant_id, timestamp);
CREATE INDEX ai_audit_operation_time ON ai_audit_log (operation_id, timestamp);
```

Indexed on `(userId, timestamp)`, `(tenantId, timestamp)`, `(operationId, timestamp)` — covers all expected query patterns (user activity audit, per-tenant audit, per-operation analysis).

### Implementation

```python
import uuid
from datetime import datetime, timezone

async def execute_and_audit(pending: dict, ctx, http_client):
    idempotency_key = str(uuid.uuid4())
    start = datetime.now(timezone.utc)
    
    response = await http_client.call(
        pending["operationId"],
        pending["params"],
        headers={
            "Authorization": f"Bearer {ctx.bearer_token}",
            "Idempotency-Key": idempotency_key
        }
    )
    
    duration_ms = int((datetime.now(timezone.utc) - start).total_seconds() * 1000)
    
    await db.execute("""
        INSERT INTO ai_audit_log
        (timestamp, user_id, tenant_id, session_id, user_query, operation_id,
         tool_name, proposed_action, human_confirmation, idempotency_key,
         http_status, duration_ms, result)
        VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13)
    """,
        start, ctx.user_id, ctx.tenant_id, ctx.session_id, ctx.user_query,
        pending["operationId"], pending.get("toolName"),
        json.dumps({"method": pending["method"], "path": pending["path"],
                    "params": pending["params"]}),
        "yes", idempotency_key, response.status, duration_ms,
        json.dumps(response.body)
    )
    
    return response.body
```

**Effort: 2-3 days** (1 day DDL + schema, 1 day implementation + wiring, 1 day testing).

**Why PostgreSQL over append-only log:** Queryable for RTI/accountability requests ("show all actions by user X in tenant Y in the last 30 days"). Filterable by operationId for impact analysis. ELK/Loki ship-out can be added later without changing the writer.

---

## 6. Temporal — Use Case and Why It Replaces n8n

### How the 5 workflows call DIGIT

Temporal activities call the **MCP server's tools** — not DIGIT APIs directly. This keeps every cross-module write on the same schema-checked, audit-logged path the interactive MCP path uses (see [03 — Minimal AI Platform](03-minimal-ai-platform.md)). Each activity carries the triggering user's Bearer token, or, for scheduled runs, the identity that approved the workflow definition up front.

```
User/scheduler triggers workflow (confirmed at entry)
  → Temporal activity: apply_for_certificate("trade-license", ...)
    → calls MCP tool `certificate_apply`
      → Kong validates JWT → Trade License service
      → audit log written
  → Temporal activity: apply_for_certificate("fire-noc", ...)      (parallel)
  → Temporal activity: apply_for_water_connection(...)              (parallel)
```

### What Temporal is

Open-source workflow orchestration engine. Handles:
- Durable execution (if server crashes mid-workflow, it resumes exactly where it stopped)
- Parallel execution (run Trade License + Fire NOC + Water Connection simultaneously, wait for all)
- Automatic retry with backoff (DIGIT API returns 503 → Temporal retries, not your code)
- Compensation (if step 3 fails, automatically run rollback for steps 1 and 2)
- Long-running workflows (a permit approval process that takes days)

### Why these 5 workflows

They are not the only cross-module need DIGIT will ever have — they are five representative *shapes* of orchestration, chosen because between them they cover the failure profiles a cross-module workflow can have:

| Workflow | Shape |
|---|---|
| Start a business | Parallel writes with dependent rollback — a saga |
| Annual renewal campaign | The same simple write, repeated at scale — a durable bulk job |
| Revenue recovery | Read, join across services, then act |
| Post-disaster triage | Read, prioritize, then dispatch |
| Commissioner's brief | Pure aggregate read, no writes |

Covering these five shapes is enough to prove the orchestration layer handles the categories of cross-module work DIGIT needs. A new cross-module requirement is very likely a variation on one of these five, not a sixth category.

### The 5 cross-module workflows — all on Temporal

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
        # Each activity below calls the matching MCP tool (e.g. certificate_apply),
        # not the DIGIT API directly — same schema check, same audit log as the interactive path
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

### Why not n8n or OpenFn for these workflows?

**n8n: capable of 4 of 5, but not chosen.**

n8n is already deployed at eGov. It has an HTTP Request node, can call DIGIT APIs with Bearer tokens, has a visual workflow editor, and can run steps in parallel using Split In Batches / Merge nodes.

| Workflow | n8n capable? | Why / Why not |
|---|---|---|
| Commissioner's Brief | Yes | Parallel reads + aggregate. This is n8n's sweet spot. |
| Revenue Recovery | Yes | Query PT + W&S → flag → write. Sequential, no compensation needed. |
| Post-disaster Triage | Yes | Webhook trigger → parallel ward assignment. n8n handles this. |
| Annual Renewal Campaign | Mostly | Loop through certificate holders, call API per holder. Weakness: if n8n crashes mid-batch, no resume. For thousands of records this matters. |
| Start a Business | No | Parallel TL + Fire NOC + Water is doable. Compensation (if Fire NOC fails after TL succeeded → cancel TL) is not built-in. Needs custom error-handling code, and even then is not a guarantee. |

The deciding factor is rollback. Start a Business needs Temporal's compensation guarantee — n8n has no built-in equivalent. Once Temporal is the engine of record for that one workflow, running n8n for the other four buys nothing: it means two orchestration engines, two failure models, two places to debug when a cross-module workflow breaks, for capability Temporal already has. Annual Renewal Campaign reinforces this independently of rollback — looping through thousands of certificate holders in n8n has no resume if it crashes mid-batch; the same workflow in Temporal resumes exactly where it stopped. Standardizing on the engine with both guarantees — rollback and crash-resume — is simpler than deciding, workflow by workflow, which of two engines it needs.

**OpenFn: not the right tool here.**  
OpenFn is designed for ETL and data integration between systems. It is good at mapping and transforming data between APIs. It is not designed for long-running sagas with compensation, parallel execution with wait-for-all, or human-in-the-loop mid-workflow. Not recommended for any of the 5 workflows.

### Recommendation

**Temporal for all 5 cross-module workflows.** Not because all 5 need saga compensation — only Start a Business does — but because once Temporal is required for that one workflow, using a second engine (n8n) for the rest adds an operational surface without adding capability. One engine, one failure model, one place to look when something breaks.

**Build order:**
1. Commissioner's Brief (Temporal, read-only, safe to iterate)
2. Revenue Recovery (Temporal, write-heavy but single-path)
3. Post-disaster Triage (Temporal, webhook/schedule-triggered)
4. Annual Renewal Campaign (Temporal, durable execution handles batch resume natively)
5. Start a Business (Temporal, saga compensation)

**Confirmation gate placement:** Human YES/NO confirmation sits before the workflow is triggered — the user confirms the whole workflow at entry, not each individual API call mid-execution.

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
| P3 | MCP server (auto-generate from specs — both levels) | AI Execution | Platform team, 1 developer | 2 weeks | P1a, P1b |
| P4 | Confirmation gate *(inside MCP server — not a separate service)* | AI Execution | Platform team, 1 developer | 3-4 days | P3 |
| P5 | Audit log writer *(inside MCP server — not a separate service)* | AI Execution | Platform team, 1 developer | 2-3 days | P3 |
| P6 | RAG V5 — diagram ingestion (specs go into MCP, not RAG) | AI Execution | Platform team, 1 developer | 3-5 days | P2a, P2b |
| P7 | 5 cross-module workflows (Temporal for all 5 — saga compensation for Start a Business, durable execution for the rest) | AI Execution | Platform team, 1-2 developers | 4-6 weeks | P1b, P3 |

**What's eliminated vs the original mini project list:**
- Semantic layer → eliminated
- Custom intent classifier → eliminated (LLM tool selection handles this)
- Hand-authored tool registry → eliminated (auto-generate from specs)
- Skills for single-service operations → eliminated (interaction diagrams + specs)

**What's new vs the original list:**
- Interaction diagrams (P2) → added, owned by platform team
- Temporal (P7) → replaces vague "cross-module orchestration" with a specific engine
- RAG adaptation (P6) → diagram ingestion only; specs go into MCP server via openapi-generator, not RAG
