# AI-fying DIGIT

**A proposal for making DIGIT AI-operable — built from internal experiments, platform analysis, and a clear-eyed view of what the platform already provides.**

Audience: Principal architect of DIGIT.

---

## What This Is

This repository documents the thinking, experiments, and architectural proposal behind making DIGIT AI-ready. It is not a critique of the platform. It is a study of what the platform already enables, what small additions would unlock significantly more AI capability, and what the AI layer itself needs to look like.

Two internal experiments informed this:
- **DIGIT MCP** (ChakshuGautam) — capability surface for DIGIT 2.9: what the platform can do, exposed as tools for AI agents
- **digit-ai-orchestrator** (Srujana Dadi) — safety layer: what AI is allowed to do right now, based on deployment state

A third is underway:
- **Smart Grievance Mapping** (intern project, due July 30) — the intelligence + application layers for PGR, proving a replicable pattern

And a fourth is live:
- **EGOV RAG V5** — DIGIT Studio documentation chatbot with feedback loop

These four pieces, studied together with DIGIT 3.0's actual capabilities, produce a coherent 4-layer architecture.

---

## The Central Claim

> DIGIT 3.0 already provides most of what Chakshu's 2.9 experiment needed to build. The remaining AI layer is thin, derivable, and almost entirely about the confirmation and intent translation boundary — not about compensating for platform gaps.

Specifically:
- 16 clean OpenAPI 3.0.3 specs → tool registry can be **seeded from generation, not hand-written** (reduces effort ~70%; encoding domain judgment on top still required)
- `digit-cli` already ships → the "no CLI" gap is closed; Chakshu's registry-derived CLI is architecturally different (add one tool → CLI + MCP + tests auto-update) but not blocking
- Workflow service exposes valid transitions via API → replaces hand-coded state gating for single-module operations; cross-module orchestration skills still needed
- `digit-client` is a typed Java HTTP wrapper for microservices — not a semantic layer for AI consumption, and not intended to be

What still needs building: an MCP server (generated from specs + progressive disclosure groups), entity resolution (wire the localization service, port `@digit-mcp/data-provider`), a confirmation middleware, and an intent classifier that covers all stakeholders.

What Chakshu's 2.9 design got right that carries forward to 3.0: progressive disclosure, semantic layer, "tools encode judgment not just structure," the one-registry-every-interface principle, dual transport, and unit economics. See [09 — Comparison](docs/09-comparison-digit29-vs-digit30.md) for detail.

---

## Document Index

| Document | What it covers |
|---|---|
| [01 — Vision](docs/01-vision.md) | The end state: what AI-fying DIGIT makes possible |
| [02 — Stakeholders](docs/02-stakeholders.md) | Who actually interacts with DIGIT at the platform level |
| [03 — Existing Experiments](docs/03-existing-experiments.md) | DIGIT MCP, orchestrator, RAG V5, intern project — what was built and what was learned |
| [04 — DIGIT 3.0 Analysis](docs/04-digit3-analysis.md) | What 3.0 provides, what it doesn't, what it means for the AI layer |
| [05 — Architecture](docs/05-architecture.md) | The full 4-layer architecture with component detail and data flows |
| [06 — Mini Projects](docs/06-mini-projects.md) | 7 concrete projects to complete the architecture, 2 already running |
| [07 — Security & Governance](docs/07-security-governance.md) | Every concern raised, with specific mitigations |
| [08 — Roadmap](docs/08-roadmap.md) | Timeline, sequencing, dependencies |
| [09 — Comparison: 2.9 vs 3.0](docs/09-comparison-digit29-vs-digit30.md) | Decision-by-decision: what 3.0 changes about Chakshu's approach, and what remains correct |
| [10 — Existing AI Initiatives](docs/10-existing-ai-initiatives.md) | 10 active eGov AI initiatives mapped to the 4-layer architecture, with the assumption each carries that may need revisiting |
| [11 — Minimal AI Platform](docs/11-minimal-ai-platform.md) | If all specs were at the certificate service standard: exactly what artifacts remain, what is eliminated, and which 8 of 23 AoP projects are relevant |
| [12 — Mini Projects Revised](docs/12-mini-projects-revised.md) | 7 concrete projects under the new architecture: spec gaps, exact interaction diagram template, MCP build steps, RAG adaptation, Temporal use cases and questions for Ghanshyam |
| [13 — Two-Level Spec Architecture](docs/13-two-level-spec-architecture.md) | Platform services (digit-specs/v3.0.0) vs application services (license-certificate, PGR, TL): what each level needs, two types of interaction diagrams, how MCP spans both |

---

## The Four Layers (summary)

```
┌─────────────────────────────────────────────────────────┐
│  Layer 4: Applications                                  │
│  CCRS, PuraSeva, HCM apps — citizen and admin UIs      │
│  AI is added here by app partners, not eGov            │
├─────────────────────────────────────────────────────────┤
│  Layer 3: Intelligence Microservices                    │
│  Domain-specific AI per DIGIT service area             │
│  classify · score · predict · alert                    │
├─────────────────────────────────────────────────────────┤
│  Layer 2: Platform AI-Readiness                         │
│  DIGIT itself, AI-operable                             │
│  MCP server · intent classifier · confirmation layer   │
│  entity resolution · state-aware tool gating           │
├─────────────────────────────────────────────────────────┤
│  Layer 1: Knowledge                                     │
│  What DIGIT knows, what DIGIT is                       │
│  RAG over documentation, specs, process guides         │
└─────────────────────────────────────────────────────────┘
```

Citizens interact with apps **built on DIGIT** (PuraSeva, CCRS), not DIGIT directly. AI within those apps is the app partner's concern. This proposal is about the platform layer — serving implementation teams, city administrators, state officials, and AI agents operating on their behalf.

---

## Status

| Component | Layer | Status |
|---|---|---|
| EGOV RAG V5 | 1 | Live (DIGIT Studio docs) |
| digit-ai-orchestrator | 2 | Built, 186 tests passing |
| Smart Grievance Mapping — Analytics FastAPI | 3 | In progress (intern, due July 30) |
| Smart Grievance Mapping — Admin Intelligence Dashboard | 4 | In progress (intern, due July 30) |
| MCP Server (auto-generated from digit-specs) | 2 | Proposed |
| Entity Resolution Service | 2 | Proposed |
| Confirmation Middleware | 2 | Proposed |
| Intent Classifier (all stakeholders) | 2 | Proposed |
| Intelligence Microservices (HCM, Property Tax, W&S) | 3 | Proposed |
