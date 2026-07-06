# AI-fying DIGIT

**A proposal for making DIGIT AI-operable — built from internal experiments, platform analysis, and a clear-eyed view of what the platform already provides.**

Audience: Principal architect of DIGIT.

---

## What This Is

This repository documents the thinking, experiments, and architectural proposal behind making DIGIT AI-ready. It is not a critique of the platform. It is a study of what the platform already enables, what small additions would unlock significantly more AI capability, and what the AI layer itself needs to look like.

Two internal experiments informed this:
- **DIGIT MCP** — capability surface for DIGIT 2.9: what the platform can do, exposed as tools for AI agents
- **digit-ai-orchestrator** — safety layer: what AI is allowed to do right now, based on deployment state

A third is underway:
- **Smart Grievance Mapping** (intern project, due July 30) — the intelligence + application layers for PGR, proving a replicable pattern

And a fourth is live:
- **EGOV RAG V5** — DIGIT Studio documentation chatbot with feedback loop

These four pieces, studied together with DIGIT 3.0's actual capabilities and the certificate service spec as the platform's own quality standard, produce a coherent and minimal AI architecture.

---

## The Central Claim

> With specs at the certificate service standard and interaction diagrams, the remaining AI layer is three things: an MCP server (auto-generated from specs), a confirmation gate (~100 lines), and Temporal for cross-module workflows. Everything else is eliminated or already in the platform.

Specifically:
- Specs at certificate standard → MCP tools **auto-generated**, not hand-authored. Human-readable codes in specs → **semantic/entity resolution layer eliminated**
- Workflow service exposes valid transitions → **no custom state-gating code**; query the platform instead
- LLM tool selection from spec descriptions → **no custom intent classifier**
- Certificate service (`digitnxt/license-certificate`) is the quality standard for application-level specs. Platform-level specs (`digitnxt/digit-specs/v3.0.0`) are a separate level needing different improvements

What still needs building: specs at the right standard (two levels — see doc 13), interaction diagrams (two types), MCP server (auto-generated), confirmation gate, Temporal for 5 cross-module workflows.

What the 2.9 design got right that carries forward: progressive disclosure, "tools encode judgment not just structure," the one-registry-every-interface principle, dual transport, unit economics. See [09 — Comparison](docs/09-comparison-digit29-vs-digit30.md) for detail.

---

## Document Index

| Document | What it covers |
|---|---|
| [01 — Vision](docs/01-vision.md) | The end state: what AI-fying DIGIT makes possible for city administrators and state officials |
| [02 — Stakeholders](docs/02-stakeholders.md) | Who interacts with DIGIT at the platform level and what AI genuinely adds for each |
| [03 — Existing Experiments](docs/03-existing-experiments.md) | DIGIT MCP, orchestrator, RAG V5, intern project — what was built and what was learned |
| [04 — DIGIT 3.0 Analysis](docs/04-digit3-analysis.md) | What 3.0 provides: digit-cli, digit-client, digit-specs, workflow service — capability by capability |
| [07 — Security & Governance](docs/07-security-governance.md) | Every concern raised, with specific mitigations |
| [09 — Comparison: 2.9 vs 3.0](docs/09-comparison-digit29-vs-digit30.md) | Decision-by-decision: what 3.0 changes about the 2.9 approach, and what remains correct |
| [10 — Existing AI Initiatives](docs/10-existing-ai-initiatives.md) | 23 AoP AI initiatives mapped to the architecture — what each carries that may need revisiting |
| [11 — Minimal AI Platform](docs/11-minimal-ai-platform.md) | **The architecture**: with specs at the certificate standard, exactly what artifacts remain, what is eliminated, which AoP projects are relevant |
| [12 — Mini Projects](docs/12-mini-projects-revised.md) | The 9 concrete projects: two spec tracks (P1a platform, P1b application), two diagram tracks (P2a, P2b), MCP, confirmation gate, audit log, RAG adaptation, Temporal |
| [13 — Two-Level Spec Architecture](docs/13-two-level-spec-architecture.md) | Platform specs vs application specs: what each level needs, two types of interaction diagrams, how MCP spans both |

---

## The Architecture (summary)

```
┌──────────────────────────────────────────────────────────────┐
│  Any consumer: city admin, state official, AI agent,         │
│  HCM console, PGR copilot                                    │
└─────────────────────┬────────────────────────────────────────┘
                      │
         ┌────────────▼────────────┐
         │       API Gateway        │
         │  Whitelist: bootstrap    │
         │  Protected: JWT → fwd   │
         └──────┬──────┬───────────┘
                │      │
      ┌─────────┘      └──────────────┐
      │                               │
┌─────▼──────┐  ┌──────────────┐  ┌──▼──────────┐
│  RAG V5    │  │  MCP Server  │  │  Temporal   │
│            │  │              │  │             │
│  "How do   │  │  Auto-gen    │  │  Cross-     │
│  I..."     │  │  from specs  │  │  module     │
│  queries   │  │  + confirm   │  │  workflows  │
│            │  │  gate        │  │             │
└─────┬──────┘  └──────┬───────┘  └──────┬──────┘
      └─────────────────┼─────────────────┘
                        │ Bearer token forwarded
                        ▼
      ┌─────────────────────────────────────┐
      │      DIGIT APIs (two levels)        │
      │                                     │
      │  Platform: workflow · individual ·  │
      │  boundary · idgen · mdms · ...      │
      │  (digit-specs/v3.0.0)               │
      │                                     │
      │  Application: certificate · PGR ·  │
      │  trade license · BPA · ...          │
      │  (license-certificate, etc.)        │
      └─────────────────────────────────────┘
```

No semantic layer. No custom intent classifier. No entity resolution service. No hand-authored tool registry.

---

## Status

| Component | What it is | Status |
|---|---|---|
| EGOV RAG V5 | Documentation Q&A chatbot | Live (DIGIT Studio docs) |
| digit-ai-orchestrator | Confirmation gate pattern (10 setup intents) | Built, 186 tests passing |
| Smart Grievance Mapping | Intelligence microservice + dashboard for PGR | In progress (due July 30) |
| P1a — Platform spec improvements | Examples, idempotency on 16 platform specs | Proposed |
| P1b — Application spec rewrites | 7 old specs → certificate standard | Proposed |
| P2a — Platform interaction diagrams | Internal behavior diagrams (~35-48) | Proposed |
| P2b — Application interaction diagrams | Cross-service orchestration diagrams (~28) | Proposed |
| P3 — MCP Server | Auto-generated from specs (both levels) | Proposed |
| P4 — Confirmation gate | ~100 lines, Redis, for all write operations | Proposed |
| P5 — Audit log | Platform-level schema, all AI callers | Proposed |
| P6 — RAG V5 spec ingestion | Endpoint-aware chunking for specs + diagrams | Proposed |
| P7 — Temporal workflows | 5 cross-module workflows | Proposed |
