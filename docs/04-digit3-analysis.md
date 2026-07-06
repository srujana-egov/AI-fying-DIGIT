# DIGIT 3.0 Analysis: What It Provides and What the AI Layer Needs on Top

This document maps DIGIT 3.0's actual capabilities against what each layer of the AI architecture needs. Sources: `digitnxt/digit-specs`, `digitnxt/digit-client-tools`, `digitnxt/examples`.

---

## What DIGIT 3.0 Ships

### digit-cli (Go, cross-platform)
Repo: `digitnxt/digit-client-tools/digit-cli`

A complete command-line interface covering:
- Account management (create, configure)
- User creation and management
- Role creation and assignment
- Workflow definition from YAML templates
- IDGen format configuration
- MDMS data management
- Boundary hierarchy management
- Registry operations
- Access control
- Multi-context configuration with authentication

**Implication for the AI layer:** digit-cli closes the "no CLI for DIGIT" gap. The onboarding problem Chakshu identified is solved. However, Chakshu's auto-generated CLI is *architecturally different* from digit-cli and is not made redundant by it. Chakshu's CLI derives directly from the same Tool Registry that powers MCP — add one tool definition and you get CLI command + MCP tool + test coverage automatically, with zero per-tool code. digit-cli is independently maintained; it and the Tool Registry will diverge over time as tools are added. This is a design question the platform team should decide: unify on registry-derived CLI, or maintain two separate CLIs for different audiences.

---

### digit-client (Java Spring library)
Repo: `digitnxt/digit-client-tools/client-libraries/digit-client`

A typed HTTP wrapper for Java/Spring microservices:
- Spring Boot 4 / Spring Framework 7 / Java 21
- 9 service clients: Workflow, Individual, Filestore, IdGen, MDMS, Notification, Boundary, Billing, Registry
- Automatic header propagation: `X-Tenant-ID`, `X-Correlation-ID`
- Spring Boot auto-configuration
- Optional Redis caching for registry lookups

**What it is:** A typed HTTP client for Java developers building services on DIGIT 3.0. The right tool for Java microservices that need to call other DIGIT services.

**What it is not:** A semantic layer for AI consumption. It does not resolve `LOC_CITYA_1 → "Ward 4, Chandrapur"`. It does not connect entities across services. An AI agent does not use a Java library.

**Implication for the AI layer:** digit-client is correct for its intended purpose. The AI layer needs a different interface — an MCP server — not a replacement for digit-client.

---

### digit-specs (OpenAPI 3.0.3)
Repo: `digitnxt/digit-specs`

16 service contracts:
`common` · `account` · `individual` · `otp` · `idgen` · `mdms` · `boundary` · `billing-payment` · `workflow` · `notification` · `localization` · `filestore` · `registry` + more

Design characteristics that matter for AI consumption:
- Human-readable codes in all API paths (no UUIDs as path parameters)
- `X-Tenant-ID` resolved from Bearer token automatically (not a request body parameter)
- Clean, consistent schema design across services
- Well-documented operation descriptions

**Implication for the AI layer:** These specs are clean enough that a tool registry can be auto-generated from them, not hand-authored. Chakshu's 61 hand-written tools exist because DIGIT 2.9 APIs were not self-describing. DIGIT 3.0 specs are. An OpenAPI Generator run against digit-specs produces correct, useful tool definitions for an MCP server.

---

### Workflow Service (the most important for AI)
Spec: `digitnxt/digit-specs/v3.0.0/workflow.yaml`

The workflow service exposes process state at runtime:

```
GET /workflow/v3/process/definition/{processCode}
  → process definition: all states, valid transitions from each state,
    SLA per state, escalation rules, assignee conditions

GET /workflow/v3/transition?entityId={id}&tenantId={tenantId}
  → current workflow instance: current state, history, assignee

POST /workflow/v3/transition
  → execute a transition: {processCode, entityId, action, comment}
```

`digit-client` confirms this with:
```java
workflowClient.listStates("CITIZEN_REGISTRATION")
workflowClient.getProcessDefinition("CITIZEN_REGISTRATION")
workflowClient.executeTransition(processCode, entityId, action, comment)
```

**Implication for the AI layer:** This is significant. The workflow service already answers the question "what is valid right now for this entity?" An AI agent does not need to reason about valid next steps — it queries the platform. The `AllowedToolsResolver` in the digit-ai-orchestrator was implemented as custom code. On DIGIT 3.0, the workflow service provides this at the platform level.

For any entity in any DIGIT workflow:
1. Query current state: `GET /workflow/v3/transition?entityId=...`
2. Query valid transitions: `GET /workflow/v3/process/definition/{processCode}` → filter to current state
3. Offer only valid transitions to AI

No custom state-gating code needed for workflow-managed operations.

---

## The Convergence: Chakshu's 6 Layers vs DIGIT 3.0

| Chakshu's 2.9 Layer | Problem It Solved | DIGIT 3.0 Status |
|---|---|---|
| CLI (auto-generated from registry) | No command-line access to the platform | **Partially addressed.** digit-cli exists. But it's separately maintained — not derived from the same registry as MCP tools. |
| Progressive disclosure (8 tools → unlock) | 61 tools in context degrades AI accuracy | **Not addressed by 3.0.** Still required regardless of how clean the specs are. |
| Tool Registry (61 tools, hand-authored) | 2.9 APIs not self-describing | **Reduced effort.** 16 clean OpenAPI specs → generate a baseline. Judgment encoding on top still needed (~30% of original effort). |
| Skills (multi-step guidance) | Platform didn't enforce process order; developer productivity | **Partially replaced.** Workflow service handles single-module process order. Cross-module orchestration skills (Revenue recovery, Commissioner's brief, Triage across modules) are still needed. Developer productivity skills (`create_digit_ui`, `create_chart`, `debug`) are still valid. |
| Semantic Layer (code resolution) | Internal codes not human-readable | **Not addressed.** Localization service exists but not wired for AI consumption. Gap unchanged. |
| MCP transport | No AI agent entry point | **Needs building.** Thin, generated from specs. Not hand-authored. |
| Product/Module Layer | Context per DIGIT module | **In specs.** Each service spec provides sufficient context. |

**The 6-layer stack simplifies on DIGIT 3.0 but does not collapse:**
1. Thin MCP server (generated from digit-specs as baseline, judgment encoding on top, progressive disclosure groups required)
2. Entity resolution service (wire the localization service, build the resolution cache)
3. Confirmation + intent layer (orchestrator pattern)
4. Cross-module skills (where workflow service can't help)

The difference from 2.9 is magnitude, not kind. The work is lighter because the specs are better. The architectural decisions Chakshu made remain correct.

---

## What the AI Layer Still Needs

### Needs Building
**MCP server** — auto-generated from digit-specs. Not hand-authored. This is the capability surface. Maps each DIGIT API endpoint to an MCP tool definition with the OpenAPI description as context.

**Entity resolution service** — wire DIGIT's existing localization service into a resolution cache. Input: any DIGIT code (`LOC_CITYA_1`, `StormWaterDrain`, `pb.amritsar`). Output: human-readable label. Redis-cached (codes rarely change).

**Confirmation middleware** — generalize the orchestrator's YES/NO pattern for all write operations across all stakeholders. Currently implemented for implementation team intents only.

**Intent classifier v2** — extend from 10 setup intents to cover city administrator queries, state official queries, read-only intelligence synthesis.

### Needs Small Platform Additions (Platform Roadmap Items)

**Dry-run mode on write APIs** — `?dryRun=true` returns what would change without executing. Currently the confirmation message is AI-generated prose about what the API call will do. It should be a deterministic preview of the actual state change.

**Idempotency keys on write operations** — AI agents can retry on network errors. Without idempotency keys, a retry creates duplicate records. Standard for any API that AI agents will call.

**AI-specific rate limiting** — AI agents can call APIs at machine speed. Existing rate limits are designed for human-paced interaction. A per-tenant, per-agent rate limit prevents runaway operations.

**Structured audit log schema** — define the schema for AI-assisted operation logs at the platform level so it's consistent across all callers. See [Security & Governance doc](07-security-governance.md) for detail.

### Does Not Need Building
- A replacement for digit-client (correct for its purpose; AI needs MCP, not Java)
- A completely new CLI if digit-cli is sufficient for the audience (though the registry-derived CLI architecture is worth adopting long-term)
- Skills for single-module sequential workflow operations (the platform handles these)

### Decision Points for the Platform Team
- **CLI architecture:** Adopt registry-derived CLI (add tool once → CLI + MCP + tests), or maintain digit-cli as a separate codebase alongside the MCP tool registry?
- **Progressive disclosure groups:** When generating the MCP server from digit-specs, define which tools load on startup (core 8) and which require `enable_tools`. This is a design decision, not an implementation detail.
- **`@digit-mcp/data-provider` adaptation:** Port Chakshu's resolution package to 3.0 API responses rather than rebuilding entity resolution from scratch.
