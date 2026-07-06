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

## Document Index

Ordered for a top-to-bottom read: why, who, what, how deep, the build plan, proof it works, why it scales, and objections handled last.

| Document | What it covers |
|---|---|
| [01 — Vision](docs/01-vision.md) | The end state: what AI-fying DIGIT makes possible for city administrators, state officials, and field workers |
| [02 — Stakeholders](docs/02-stakeholders.md) | Who interacts with DIGIT at the platform level, what AI genuinely adds for each, and why developers and one-time city setup are out of scope |
| [03 — Minimal AI Platform](docs/03-minimal-ai-platform.md) | **The architecture**: with specs at the certificate standard, exactly what artifacts remain and what is eliminated |
| [04 — Request Walkthroughs](docs/04-request-walkthroughs.md) | Two step-by-step traces through the architecture: an interactive write (trade license application) and a scheduled cross-module job (revenue recovery on Temporal) |
| [05 — Two-Level Spec Architecture](docs/05-two-level-spec-architecture.md) | Platform specs vs application specs: what each level needs, two types of interaction diagrams, how MCP spans both |
| [06 — Mini Projects](docs/06-mini-projects-revised.md) | The 9 concrete projects: two spec tracks, two diagram tracks, MCP, confirmation gate, audit log, RAG adaptation, Temporal workflows |
| [07 — AI Patterns Across DIGIT Products](docs/07-ai-patterns-across-digit-products.md) | 5 generalizable patterns (flagging, GIS cross-reference, proactive alerting, deduplication, process intelligence) mapped across all 18 eGov domain products |
| [08 — Security & Governance](docs/08-security-governance.md) | Every concern raised, with specific mitigations |
