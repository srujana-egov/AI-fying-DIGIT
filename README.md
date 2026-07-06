# AI-fying DIGIT

**A proposal for making DIGIT AI-operable — built from internal experiments, platform analysis, and a clear-eyed view of what the platform already provides.**

Audience: Principal architect of DIGIT.

---

## What This Is

This repository documents the thinking, experiments, and architectural proposal behind making DIGIT AI-ready. It is not a critique of the platform. It is a study of what the platform already enables, what small additions would unlock significantly more AI capability, and what the AI layer itself needs to look like.

Four experiments informed this:
- **DIGIT MCP** — capability surface for DIGIT 2.9: what the platform can do, exposed as tools for AI agents
- **digit-ai-orchestrator** — safety layer: what AI is allowed to do right now, based on deployment state
- **Smart Grievance Mapping** (intern project, due July 30) — the intelligence + application layers for PGR, proving a replicable pattern
- **EGOV RAG V5** — DIGIT Studio documentation chatbot with feedback loop (live)

These four pieces, studied together with DIGIT 3.0's actual capabilities and the certificate service spec as the platform's own quality standard, produce a coherent and minimal AI architecture.

---

## The Central Claim

> With specs at the certificate service standard and interaction diagrams, the remaining AI layer is three things: an MCP server (auto-generated from specs), a confirmation gate (~100 lines), and Temporal for cross-module workflows. Everything else is eliminated or already in the platform.

Specifically:
- Specs at certificate standard → MCP tools **auto-generated**, not hand-authored. Human-readable codes → **semantic/entity resolution layer eliminated**
- Workflow service exposes valid transitions → **no custom state-gating code**; query the platform instead
- LLM tool selection from spec descriptions → **no custom intent classifier**
- Certificate service (`digitnxt/license-certificate`) is the quality standard for application-level specs. Platform-level specs (`digitnxt/digit-specs/v3.0.0`) are a separate level needing different improvements

What still needs building: specs at the right standard (two levels — see doc 13), interaction diagrams (two types), MCP server, confirmation gate, Temporal cross-module workflows (one engine for all 5 — saga compensation for Start a Business, durable execution for the rest).

---

## Document Index

| Document | What it covers |
|---|---|
| [01 — Vision](docs/01-vision.md) | The end state: what AI-fying DIGIT makes possible for city administrators, state officials, and field workers |
| [02 — Stakeholders](docs/02-stakeholders.md) | Who interacts with DIGIT at the platform level, what AI genuinely adds for each, and why developers and one-time city setup are out of scope |
| [07 — Security & Governance](docs/07-security-governance.md) | Every concern raised, with specific mitigations |
| [11 — Minimal AI Platform](docs/11-minimal-ai-platform.md) | **The architecture**: with specs at the certificate standard, exactly what artifacts remain and what is eliminated |
| [12 — Mini Projects](docs/12-mini-projects-revised.md) | The 9 concrete projects: two spec tracks, two diagram tracks, MCP, confirmation gate, audit log, RAG adaptation, Temporal workflows |
| [13 — Two-Level Spec Architecture](docs/13-two-level-spec-architecture.md) | Platform specs vs application specs: what each level needs, two types of interaction diagrams, how MCP spans both |
| [14 — AI Patterns Across DIGIT Products](docs/14-ai-patterns-across-digit-products.md) | 5 generalizable patterns (flagging, GIS cross-reference, proactive alerting, deduplication, process intelligence) mapped across all 18 eGov domain products |
| [15 — Request Walkthroughs](docs/15-request-walkthroughs.md) | Two step-by-step traces through the architecture: an interactive write (trade license application) and a scheduled cross-module job (revenue recovery on Temporal) |

---

## The Architecture (summary)

```
┌──────────────────────────────────────────────────────────────┐
│  Any consumer: city admin · state official · field worker    │
│  AI agent · HCM console · PGR copilot                        │
└─────────────────────┬────────────────────────────────────────┘
                      │
         ┌────────────▼────────────┐
         │      Kong (digit3)       │
         │  JWT validate + forward  │
         │  X-Tenant-ID inject      │
         │  RBAC (accesscontrol)    │
         └──────┬──────┬──────┬────┘
                │      │      │
      ┌─────────┘      │      └───────────────┐
      │                │                      │
┌─────▼──────┐  ┌──────▼────────┐  ┌──────────▼──────┐
│  RAG V5    │  │  MCP Server   │  │  Temporal       │
│            │  │               │  │                 │
│  "How do   │  │  auto-gen     │  │  5 cross-module │
│  I..."     │  │  from specs   │  │  workflows      │
│            │  │  confirm gate │  │  (calls MCP     │
│            │  │  audit log    │  │  tools)         │
└────────────┘  └──────┬────────┘  └──────┬──────────┘
                       └────────┬──────────┘
                                │ Bearer token forwarded
                                ▼
┌──────────────────────────────────────────────────────────────┐
│  Application Services  (certificate standard)                │
│  certificate · pgr · trade-license · bpa · property ·       │
│  fire-noc · water · works · hcm · ...                        │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  Platform Services  (digit-specs/v3.0.0)                     │
│  workflow · individual · boundary · idgen · mdms ·           │
│  notification · filestore · billing · ...                    │
└──────────────────────────────────────────────────────────────┘

                              ↕ scheduled agents (no human trigger)
┌──────────────────────────────────────────────────────────────┐
│  Intelligence Layer                                          │
│  Flagging · GIS Cross-Reference · Proactive Alerting ·       │
│  Deduplication · Process Intelligence                        │
│  (reads application service records, writes flags via API)   │
└──────────────────────────────────────────────────────────────┘
```

No semantic layer. No custom intent classifier. No entity resolution service. No hand-authored tool registry.

---

## Status

| Component | What it is | Status |
|---|---|---|
| EGOV RAG V5 | Documentation Q&A chatbot | Live (DIGIT Studio docs) |
| digit-ai-orchestrator | Confirmation gate pattern (10 setup intents) | Built |
| Smart Grievance Mapping | Intelligence microservice for PGR (intern project) | In progress — due July 30 |
| HCM Population Denominator | GIS cross-reference: WorldPop + Google Open Buildings vs HCM enrollment (intern project) | In progress |
| HCM Beneficiary Deduplication | On-device fuzzy match Flutter library (intern project) | In progress |
| P1a — Platform spec improvements | Examples, idempotency on 16 platform specs | Proposed |
| P1b — Application spec rewrites | ~18 domain product specs → certificate standard | Proposed |
| P2a — Platform interaction diagrams | Internal behavior diagrams (~35-48) | Proposed |
| P2b — Application interaction diagrams | Cross-service orchestration diagrams (~55-70) | Proposed |
| P3 — MCP Server | Auto-generated from specs (both levels) | Proposed |
| P4 — Confirmation gate | ~100 lines, Redis, for all write operations | Proposed |
| P5 — Audit log | Platform-level schema, all AI callers | Proposed |
| P6 — RAG V5 diagram ingestion | Ingest interaction diagrams into knowledge base | Proposed |
| P7 — Temporal workflows | 5 cross-module workflows, one engine (saga compensation for Start a Business, durable execution for the rest) | Proposed |
