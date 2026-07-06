# Roadmap

## What Exists Today

| Component | Layer | Status |
|---|---|---|
| EGOV RAG V5 | 1 — Knowledge | Live. DIGIT Studio docs. Needs knowledge expansion. |
| digit-ai-orchestrator | 2 — Platform AI-Readiness | Built. 186 tests. 10 setup intents. Implementation team use case. |
| Smart Grievance Mapping — Analytics FastAPI | 3 — Intelligence | In progress. Week 3 of 8. Due July 30. |
| Smart Grievance Mapping — Admin Intelligence Dashboard | 4 — Applications | In progress. Week 3 of 8. Due July 30. |
| digit-cli | 2 — Platform AI-Readiness | Exists in DIGIT 3.0. Not AI layer work. |
| digit-client | 2 — Platform AI-Readiness | Exists in DIGIT 3.0. Java services. Not AI layer work. |
| digit-specs | Foundation | Exists. 16 OpenAPI 3.0.3 files. Source for MCP generation. |

---

## Sequence

```
July 2026
├── Smart Grievance Mapping — due July 30
│   Completes: Layer 3 pattern for PGR
│   Proves: intelligence microservice + dashboard + alert engine
│
├── RAG V5 knowledge expansion (parallel, ongoing)
│   Ingest: digit-specs, DIGIT documentation, process guides
│   Completes: Layer 1 coverage for full platform

August 2026
├── Mini Project 3: MCP Server (3 weeks)
│   Generates from: digit-specs
│   Completes: capability surface for all of DIGIT 3.0
│
├── Mini Project 4: Entity Resolution Service (2 weeks, parallel with 3)
│   Wires: DIGIT localization service → Redis cache
│   Completes: code → human-readable for AI responses
│
├── Mini Project 5: Confirmation Middleware (2 weeks, parallel with 3+4)
│   Generalises: orchestrator YES/NO pattern
│   Redis-backed, any write operation, TTL on pending actions
│   Completes: safe write execution for all stakeholders

September 2026
├── Mini Project 6: Platform Intent Classifier (4 weeks, starts after MP3)
│   Extends: 10 setup intents → all DIGIT operations
│   Covers: implementation teams + city admins + state officials
│   Completes: Layer 2 for all stakeholder types
│
├── Intelligence Microservices — HCM (next intern batch, parallel)
│   Reuses: intern project pattern
│   Covers: attendance anomaly, posting gaps, training completion

Q4 2026
├── Layer 2 complete
│   MCP Server + Entity Resolution + Confirmation + Intent Classifier
│   DIGIT is AI-operable for all stakeholder types
│
├── Layer 3 growing per domain
│   HCM done, Property Tax underway, Water & Sewerage following
│
├── Full vision demonstrable
│   Commission a new city using natural language
│   City administrator queries live DIGIT data conversationally
│   State official gets cross-city intelligence in seconds
│   Ward engineer gets proactive monsoon alert before complaints arrive
```

---

## Platform Additions Requested (not AI layer work)

These are small additions to DIGIT 3.0 that would unlock significantly safer AI-assisted operations. Flagging for the platform team:

| Addition | Why Needed | Effort (est.) |
|---|---|---|
| `?dryRun=true` on write APIs | Deterministic state-change preview before confirmation | Medium |
| Idempotency keys on writes | Prevent AI retry from creating duplicate records | Low |
| AI-specific rate limits per tenant | Prevent machine-speed API abuse | Low |
| Structured audit log schema | Consistent AI operation record across all callers | Low |

None of these block the AI layer from being built. They make it safer in production.

---

## What the Architecture Does Not Require

- Replacing digit-cli (it already works; AI builds on top of it)
- Replacing digit-client (correct for Java services; AI needs MCP, not Java)
- Hand-authoring a tool registry (digit-specs are clean enough to generate from)
- Writing skills for multi-step operations (workflow service exposes valid transitions; query the platform)
- A new authentication system (forward the user's existing Bearer token)
- A new RBAC system (DIGIT's RBAC enforces at the API level)

---

## Dependency Graph

```
RAG V5 expansion ─────────────────────────────────────────► Layer 1 complete
                                                                    │
Intern project (July 30) ─────────────────────────────────► Layer 3+4 for PGR
                                                                    │
MP3: MCP Server ──┐                                                │
MP4: Entity Res. ─┤ (parallel) ────────────────────────────► Layer 2 foundation
MP5: Confirmation ┘                                                │
                                                                    │
MP6: Intent Classifier (needs MP3) ────────────────────────► Layer 2 complete
                                                                    │
MP7: Domain Intelligence (parallel per domain) ────────────► Layer 3 growing
                                                                    │
                                                            Full vision live
```

---

## The Ask

If this proposal is directionally correct, the specific asks from the platform team are:

1. **Confirm that auto-generating the MCP server from digit-specs is the right approach** — or identify specs gaps that would make generation incomplete
2. **Confirm that Bearer token forwarding through the MCP server is the right auth model** — or identify cases where a service identity is needed
3. **Confirm the workflow service can serve as AllowedToolsResolver** — `GET /workflow/v3/process/definition/{processCode}` returning valid transitions from current state is the claim; validation from the platform team would close this
4. **Consider `?dryRun=true` on write APIs** — the single highest-value platform addition for safe AI operation
5. **Define the audit log schema at the platform level** — so all AI callers produce consistent records
