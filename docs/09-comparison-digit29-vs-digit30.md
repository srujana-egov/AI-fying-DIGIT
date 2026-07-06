# Comparison: DIGIT MCP (2.9) vs What DIGIT 3.0 Changes

This document maps DIGIT MCP (2.9) design decisions against DIGIT 3.0's actual capabilities. The goal is not to declare a winner — it is to understand which design decisions were 2.9-specific and which are still correct regardless of platform version.

Source: "Building Products on DIGIT at the Speed of AI" deck (2026), verified against digitnxt/digit-specs and digitnxt/digit-client-tools.

---

## The Starting Problem

> "DIGIT is powerful. Accessing it is hard. Every persona needs the same data — but today they each have to learn a completely different way to get it."

- **System Administrator:** No CLI. Must write custom scripts or use a browser UI not built for power users.
- **Developer / SI Partners:** Each team builds their own DIGIT client. No shared standard SDK. 100% duplicated effort.
- **Result:** Fragmented access, duplicated code, no AI leverage, steep learning curve for every new team.

This problem statement is accurate for DIGIT 2.9. DIGIT 3.0 partially addresses it — but not completely.

---

## The Solution: One Registry, Every Interface

The core design insight:

> "The Tool Registry is the main component. MCP is one caller. CLI is another. Tests are a third. No AI needed for the platform to work."

A single ToolRegistry powers three interfaces from one definition:

| Interface | What it provides |
|---|---|
| MCP Server | stdio + HTTP · AI agents · 61 tools + progressive disclosure |
| CLI (digit ...) | Auto-generated · 57 commands · zero per-tool code |
| Unit Test Suite | ~746 assertions across 24 test files · CI/CD ready |

**This design principle is correct regardless of DIGIT version.** One definition, every interface, no duplication — this is the right answer on 2.9 and on 3.0.

---

## What He Built — Full Detail

### Tool Registry
- 61 typed tools across 14 domain groups
- Groups: core, docs, mdms, boundary, pgr, employees, admin, monitoring, tracing, encryption, localization, masters, idgen, location
- Enum-guarded parameters (prevents invalid calls)
- Named intent per tool (not just HTTP method + path)

### Progressive Disclosure
**8 core tools load on startup. Agents call `enable_tools` to unlock more as needed.**

This is a specific, correct answer to context rot. An AI agent given 61 tools at once loses accuracy on tool selection. Starting with 8 and unlocking on demand keeps the agent's active context lean without sacrificing capability.

This is not solved by having clean OpenAPI specs. Even a perfect spec with 100 endpoints degrades AI accuracy if all are in context simultaneously.

### Auto-Generated CLI
57 commands, zero per-tool code. Add an MCP tool → get a CLI command automatically. Three output modes: JSON (piped), table (TTY), plain (scripting).

The CLI is a **derived artifact** of the registry, not a separately maintained codebase. This is the critical difference from DIGIT 3.0's digit-cli, which is independently maintained.

### Semantic Layer
`LOC_CITYA_1 → Ward 4 · North Zone · pg.citya`
`COMPLAINT_TYPE_1 → Garbage Not Collected`

A DIGIT SDK that turns raw platform data into meaningful, connected domain objects. Every consumer uses it without needing to know how DIGIT works internally.

This is not a typed HTTP wrapper. It is entity resolution + cross-service connection. The `@digit-mcp/data-provider` npm package implements this.

### Skills Registry
`create_digit_ui` · `create_chart` · `debug` · `check_workflow_health`

Guided workflows that combine tools into outcomes. Composable, AI-reasoned, human-ready. These are for complex multi-step operations that require reasoning across tools — not for simple CRUD.

### What's Built (Verified March 2026)
- 61 Tools · 14 Groups
- Dual transport: stdio for local · HTTP Streamable for cloud
- Progressive disclosure: 8 core tools on start
- `@digit-mcp/data-provider` npm package
- Multi-tenant: pg.citya → pg auto-derived
- Prompt injection defense + arg sanitization
- 127 integration + 53 agent safety tests (180 total)
- Grafana Tempo tracing, Kafka lag, DB rows observability

---

## The Argument Against "Just Use Clean OpenAPI Specs"

The DIGIT MCP deck addresses this directly:

> "Even perfect API docs can't encode: which of 26 services handles a given intent, what sequence of calls achieves a multi-step workflow, or how to recover from a partial failure mid-transaction. Tools encode that judgment as code. An LLM reading docs reconstructs it probabilistically on every call — inconsistently."

> "Progressive disclosure also can't be replicated by docs — an agent starting with 61 tools loses context; starting with 8 and unlocking on demand works."

**This argument is correct and applies to DIGIT 3.0 as well.**

DIGIT 3.0's 16 OpenAPI specs are clean enough to seed a tool registry from. They are not clean enough to replace one. An AI reading the workflow spec still needs to know: for a property tax assessment request, which service, in what sequence, with what error recovery. That is judgment, not structure.

---

## Decision-by-Decision: What 3.0 Changes

### 1. CLI — Partially addressed by 3.0

| 2.9 problem | 2.9 solution | 3.0 status |
|---|---|---|
| No CLI exists | Auto-generated from ToolRegistry | digit-cli ships with 3.0 |

**What changes:** digit-cli in 3.0 already exists. The onboarding gap is closed.

**What doesn't change:** digit-cli is independently maintained. The auto-generated CLI from the 2.9 experiment derives from the same registry as MCP tools — add one tool, get CLI + MCP + tests automatically. 3.0's digit-cli doesn't have this property. They diverge over time.

**Verdict:** 3.0 closes the "no CLI" gap but the registry-derived approach is architecturally cleaner for long-term maintainability.

---

### 2. Tool Registry — Partially addressed by 3.0

| 2.9 problem | 2.9 solution | 3.0 status |
|---|---|---|
| 2.9 APIs not self-describing | 61 hand-authored tools | 16 clean OpenAPI 3.0.3 specs |

**What changes:** DIGIT 3.0 specs are clean enough to seed the tool registry from. The hand-authoring effort is substantially reduced — generate a baseline, then encode judgment on top.

**What doesn't change:** The tool registry still needs to exist. Clean specs produce the structure. Tools encode the judgment (intent routing, sequence, error recovery, progressive disclosure groups).

**Verdict:** 3.0 reduces hand-authoring effort from 100% to ~30%. The registry concept is still correct.

---

### 3. Skills — Partially replaced by 3.0

| 2.9 problem | 2.9 solution | 3.0 status |
|---|---|---|
| Platform doesn't enforce process order | Skills guide AI through multi-step sequences | Workflow service exposes valid transitions |

**What changes:** DIGIT 3.0's workflow service answers "what transitions are valid from the current state of this entity?" An AI can query the platform instead of reasoning about valid next steps from scratch. This constrains the option set — the AI picks from valid transitions, not from all possible API calls.

**What doesn't change:** The workflow service tells you what's *valid*. It doesn't tell you what's *wise*. `PENDING_VERIFICATION → [APPROVE, REJECT, REQUEST_MORE_INFO]` — all three are valid; choosing correctly still requires reasoning. More importantly: setup operations (city onboarding, workflow definition, idgen, boundary setup) have no workflow entity — there's nothing for the service to query. Cross-module operations (Trade License + Fire NOC + Water Connection in parallel, Revenue recovery across PT + W&S, Commissioner's brief across four modules) span separate workflows the platform doesn't coordinate. Skills for these cases are still needed.

**Verdict:** 3.0 reduces skill authoring for single-entity, single-module workflow progression. It does not eliminate skills. The strongest use cases (Revenue recovery, Commissioner's brief, Start a business) all cross module boundaries — the workflow service doesn't touch them.

---

### 4. Semantic Layer — Not addressed by 3.0

| 2.9 problem | 2.9 solution | 3.0 status |
|---|---|---|
| Internal codes not human-readable | `@digit-mcp/data-provider` resolves codes to domain objects | Localization service exists but not wired for AI consumption |

**What changes:** Nothing. DIGIT 3.0 uses cleaner codes in API paths (no UUIDs), but `LOC_CITYA_1`, `COMPLAINT_TYPE_1`, `pb.amritsar` are still internal codes that AI and humans cannot interpret without resolution.

**What doesn't change:** The semantic layer is still required. The localization service still needs to be wired. `@digit-mcp/data-provider`'s approach — entity resolution + cross-service connection into meaningful domain objects — is still the right pattern.

**Verdict:** The semantic layer from the 2.9 experiment is directly applicable to 3.0 with minor adaptation. This is the layer that changes least between versions.

---

### 5. Progressive Disclosure — Not addressed by 3.0

| 2.9 problem | 2.9 solution | 3.0 status |
|---|---|---|
| 61 tools degrades AI accuracy | 8 core tools + `enable_tools` on demand | Not addressed by clean specs |

**What changes:** Nothing. 3.0 having 16 clean service specs doesn't reduce the context rot problem. If an AI is given all 100+ endpoint tools at once, accuracy degrades. Clean specs produce more tools, not fewer.

**What doesn't change:** The progressive disclosure pattern — start minimal, unlock on demand — is still the correct architecture for AI agents operating on a large platform.

**Verdict:** The progressive disclosure design from the 2.9 experiment is directly applicable to 3.0. The MCP server built from digit-specs should implement the same unlock pattern.

---

### 6. MCP as Transport — Unchanged

| 2.9 | 3.0 |
|---|---|
| MCP is a transport, not an architecture | MCP is a transport, not an architecture |

The DIGIT MCP deck explicitly addresses protocol risk: "MCP is the thinnest layer in our stack. The Semantic Layer, Tool Registry, and domain model are all protocol-agnostic. Estimated migration effort: 1–2 sprints to swap transport. Zero tool rewrites."

This is correct and carries forward identically.

---

### 7. Unit Economics — Unchanged

Unit economics from the 2.9 experiment: a city with 20,000 transactions/month, 20% AI-assisted:
- AI inference cost: $40–$120/month
- Staff time recovered: $10,000–$20,000/month
- Return: ~100x on AI inference spend

These numbers don't change between 2.9 and 3.0. The economic case for AI-assisted DIGIT operations is independent of the platform version.

---

## What Needs Updating on 3.0

These are things in the 2.9 design that need adaptation for 3.0:

| Component | 2.9 approach | 3.0 adaptation needed |
|---|---|---|
| Tool authoring | Hand-written for all 61 | Seed from OpenAPI Generator, encode judgment on top |
| CLI | Auto-generated, ships as part of MCP package | Integrate with existing digit-cli OR maintain registry-derived parallel |
| RequestInfo wrapper | DIGIT 2.9's verbose request envelope | 3.0 has cleaner request structure — tool implementations need updating |
| Auth | 2.9 auth model | 3.0 Bearer token from tenant (X-Tenant-ID resolved automatically) |
| Semantic layer | `@digit-mcp/data-provider` npm | Adapt to 3.0 API responses; most resolution logic is unchanged |

---

## The Correct Synthesis

The 2.9 approach and DIGIT 3.0's improvements are **not in competition**. 3.0 makes the 2.9 approach cheaper and more maintainable to build. The core design decisions remain correct.

```
What 3.0 changes about the approach:
├── Tool authoring effort      reduced (seed from OpenAPI, not hand-write)
├── CLI need                   reduced (digit-cli exists)
└── Single-module process skills  reduced (workflow service handles these)

What stays exactly the same:
├── One registry, every interface           ✓ still correct
├── Progressive disclosure (8 → unlock)    ✓ still required
├── Semantic layer (code resolution)        ✓ still required
├── Tools encode judgment, not just structure  ✓ still true
├── AI features degrade gracefully          ✓ still the right design
├── Cross-module skills (Revenue recovery,  
    Commissioner's brief, Triage)           ✓ still needed
└── Unit economics                          ✓ unchanged
```

**The 6 use cases identified in the 2.9 experiment (start a business, annual renewal, revenue recovery, post-disaster triage, building permit diagnosis, commissioner's brief) are the right use cases. They all work better on 3.0, not differently.**

---

## What This Means for the Roadmap

The AI-fying DIGIT mini projects should be informed by the 2.9 design:

1. **MCP Server (Mini Project 3)**: Generate from digit-specs as a baseline, then implement progressive disclosure groups matching the 14-group structure from the 2.9 experiment. Don't just expose all endpoints flat.

2. **Entity Resolution (Mini Project 4)**: `@digit-mcp/data-provider` already solves this for 2.9. Port and adapt to 3.0 API responses rather than building from scratch.

3. **Skills for cross-module orchestration**: The revenue recovery and commissioner's brief use cases require skills that span Property Tax + Water + PGR + Finance. These are not solved by clean OpenAPI specs or workflow service queries. They need explicit authoring.

4. **The CLI question**: Decide whether to adopt the registry-derived CLI approach (add tool once → CLI + MCP + tests auto-update) or maintain digit-cli as a separate codebase. The former is more maintainable at scale.
