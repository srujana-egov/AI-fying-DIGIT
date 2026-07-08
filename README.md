# AI-fying DIGIT

**A proposal for making DIGIT AI-operable — built from internal experiments, platform analysis, and a clear-eyed view of what the platform already provides.**

---

## Document Index

Ordered for a top-to-bottom read: why, who, what, how deep, the build plan, proof it works, why it scales, and objections handled last.

| Document | What it covers |
|---|---|
| [01 — Vision](docs/01-vision.md) | The end state: what AI-fying DIGIT makes possible for city administrators, state officials, and field workers |
| [02 — Stakeholders](docs/02-stakeholders.md) | Who interacts with DIGIT at the platform level, what AI genuinely adds for each, and why developers and one-time city setup are out of scope |
| [03 — Minimal AI Platform](docs/03-minimal-ai-platform.md) | **The architecture**: how spec quality shapes what gets auto-generated vs hand-authored, and the complete platform + AI execution layer |
| [04 — Request Walkthroughs](docs/04-request-walkthroughs.md) | Two step-by-step traces through the architecture: an interactive write (trade license application) and a scheduled cross-module job (revenue recovery on Temporal) |
| [05 — Two-Level Spec Architecture](docs/05-two-level-spec-architecture.md) | Platform specs vs application specs: what each level needs, two types of interaction diagrams, how MCP spans both |
| [06 — Mini Projects](docs/06-mini-projects-revised.md) | The 9 concrete projects: two spec tracks, two diagram tracks, MCP, confirmation gate, audit log, RAG adaptation, Temporal workflows |
| [07 — AI Patterns Across DIGIT Products](docs/07-ai-patterns-across-digit-products.md) | 5 generalizable patterns (flagging, GIS cross-reference, proactive alerting, deduplication, process intelligence) mapped across all 18 eGov domain products |
| [08 — Security & Governance](docs/08-security-governance.md) | Every concern raised, with specific mitigations |
