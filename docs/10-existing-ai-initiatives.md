# Existing AI Initiatives at eGov: Where They Fit and Where the Assumptions May Need Revisiting

Source: eGov AI/LCNC AoP tracking sheet (internal). As of July 2026.

This document is not a critique. Every initiative listed here is solving a real problem. The concern is that each is solving it independently — which means each team also independently solving auth propagation, confirmation UX, audit trail, entity resolution, and context management. These are platform concerns. Building them N times in N systems produces N different security postures and N maintenance burdens.

This document maps each initiative to the 4-layer architecture and names the one assumption in each that may need revisiting before it hits production.

---

## The Pattern Across All Initiatives

Before the individual breakdown, the common pattern:

```
What each team is building:
  intent understanding  ──┐
  auth to DIGIT         ──┤  all of these are
  confirmation UX       ──┤  platform concerns,
  entity resolution     ──┤  not domain concerns
  audit trail           ──┘

What each team thinks they're building:
  [domain-specific AI feature]
```

When every team builds their own AI layer, you get:
- MCP for PGR using one auth model
- HCM copilot using a different auth model
- Service config agent using a third
- Different confirmation UX patterns across the product
- No consistent audit trail schema

The platform AI layer (Layer 2) is the shared foundation that means each team only needs to build the domain-specific part.

---

## Initiative-by-Initiative

---

### Row 2 — LLM Guidance Chatbot
**Status:** Completed POC. Productization in progress (~70% done).  
**What it is:** Documentation chatbot for DIGIT principles, best practices, existing code.

**Where it fits:** Layer 1 — Knowledge.  
This is the RAG V5 architecture. It is already incorporated into this proposal as the knowledge backbone for all layers above it.

**No assumption conflict.** This is Layer 1. It works as designed.

**What it needs next:** Routing. When a user asks "how do I configure boundary?" → Layer 1 answers. When a user says "configure boundary for pb.amritsar" → Layer 2 acts. The chatbot currently handles only the first case. A thin intent router that distinguishes questions from commands would complete the picture.

---

### Row 5 — MCP for DIGIT PGR
**Status:** In progress, ~40% done. Target: Sept 2026.  
**What it is:** MCP server for DIGIT PGR service.

**Where it fits:** Layer 2 — Platform AI-Readiness. Specifically, the tool surface for PGR.

**The assumption that may need revisiting:**  
The sheet notes: *"With agentic AI emerging the MCP adoption is no longer required."*

This conflates the transport (MCP) with the architecture (Tool Registry + Semantic Layer). Even if MCP as a protocol is eventually superseded, you still need:
- A structured tool surface that AI agents can call reliably
- Entity resolution (complaint codes → human-readable labels)  
- Auth propagation (caller's JWT forwarded, not a service account)
- An audit trail of what the AI called and what it executed

These requirements exist regardless of whether the transport is MCP, HTTP tool-use, function calling, or whatever comes after MCP. Stopping because "MCP may not be the right transport" is like stopping because "REST may be replaced by gRPC" — the interface contract matters more than the wire format.

**The second assumption:** Building this for PGR specifically means rebuilding auth, entity resolution, and confirmation for every service added next. The Tool Registry approach (one registry, every service) is more maintainable. The PGR work is a valid starting point — but the architecture should be designed to generalize from day one, not refactored later.

---

### Row 8 — Service Config Creation AI Agent
**Status:** In progress, ~15% done. "Rethinking for generic capability."  
**What it is:** "Natural language to config generation" — converts natural language input into DIGIT service configurations.

**Where it fits:** Limited. See concern below.

**The assumption that needs revisiting:**  
Two structural problems with AI-assisted configuration as a platform AI priority:

1. **One-time, not at-scale:** City configuration happens once per city. This is a developer productivity tool, not a recurring AI problem at scale. The real at-scale AI problems are operational — thousands of complaints per day, daily administrator briefs, monthly renewal campaigns.

2. **Tenant isolation prevents cross-city templates:** DIGIT's multi-tenant architecture means one city cannot read another city's MDMS config, boundary hierarchy, or workflow definitions. The premise of "learn from similar cities already on DIGIT" does not work architecturally.

3. **No real-world zero-knowledge production onboarding:** Implementation teams arrive trained. Organisations do not configure production DIGIT environments with zero prior knowledge. This use case is sandbox/developer environments only.

**What this means for the initiative:** The confirmation gate mechanism (natural language → confirm → execute) is still correct and is needed at the platform level — but for **operational use cases** (city admin approving a workflow transition, executing a batch renewal), not configuration. The initiative should be redirected toward those operational flows, not extended to cover more configuration intents.

---

### Row 13 — AI Chatbot for Console Config Generation
**Status:** In progress. Tech design underway.  
**What it is:** "AI-Powered Configuration Generation for DIGIT Flow Builder." RAG model being explored.

**Where it fits:** Layer 1 (if answering configuration questions) + Layer 2 (if generating and executing configurations).

**The assumption that may need revisiting:**  
RAG retrieves and synthesizes from documentation. It answers "what fields are required for a workflow definition?" accurately. But when a user says "create this workflow configuration for me," retrieval alone hits a hard wall:

- RAG can tell you the shape of a workflow definition
- It cannot tell you the current deployment state of the tenant
- It cannot enforce "account must exist before workflow can be configured"
- It cannot show the user what will change before executing
- It cannot write an audit record that proves the human confirmed the action

The moment this goes from Q&A to "actually create things in DIGIT," it needs intent classification + state gating + confirmation + auth propagation + audit trail — not retrieval.

The RAG model is the right foundation for the Q&A half. The orchestrator pattern is the right foundation for the action half. A thin routing layer between them ("is this a question or a command?") is what connects them. Building a new system that tries to do both from scratch is the expensive path.

---

### Row 19 — Conversational Data Layer for HCM
**Status:** In progress.  
**What it is:** "Fraud Detection and conversational data layer" for HCM.

**Where it fits:**
- Fraud detection → Layer 3 — Intelligence (domain-specific ML, HCM analytics)
- Conversational data layer → Layer 2 — Platform AI-Readiness

**The assumption that may need revisiting:**  
"Conversational data layer" framed as an HCM feature will be scoped to HCM data only. But the conversational data layer — intent understanding, natural language → DIGIT query translation, entity resolution, auth propagation — is identical for HCM, PGR, Property Tax, and every other DIGIT service. The domain changes; the machinery does not.

If this is built as HCM-specific, the Property Tax team builds the same thing for Property Tax. The Water & Sewerage team builds the same thing for Water. Each time re-solving the same auth, entity resolution, and confirmation problems.

The fraud detection work is genuinely domain-specific (HCM attendance patterns, payroll anomalies) and belongs in a Layer 3 intelligence microservice. The conversational layer it calls is shared platform infrastructure.

---

### Row 22 — AI Copilot for HCM Console (Ram)
**Status:** Completed.  
**What it is:** Console AI Copilot — embedded in the HCM console.

**Where it fits:** Layer 4 — Application. An AI-assisted UI built on top of DIGIT.

**The assumption that may need revisiting:**  
The concern is not with what was built but with what it calls underneath. If the HCM copilot calls OpenAI directly and constructs DIGIT API calls from AI output, then:

- Auth: is it using the user's Bearer token or a service account?
- Confirmation: does the AI execute directly or propose → human confirms?
- Audit: is there a record of what the AI proposed and what the human confirmed?

These are not implementation details — they are the difference between a demo and a production system. A city commissioner who asks the AI copilot to approve a payroll batch and finds out later the AI executed it without their explicit confirmation has a grievance with a legal dimension.

The copilot pattern is right. It belongs in Layer 4. The question is whether the AI operations it executes flow through a shared Layer 2 (with confirmation, audit, auth) or bypass it.

---

### Row 12 — Automating Repetitive Tasks in HCM Implementation (unassigned)
**Status:** Not started.  
**What it is:** "Automating repetitive tasks in HCM implementation."

**Where it fits:** Layer 2 — Platform AI-Readiness. Specifically, the orchestrator extended to HCM setup intents.

**No design conflict — this is what the orchestrator was built for.** The 10 current intents cover DIGIT platform setup (account, workflow, idgen, boundary, etc.). HCM implementation tasks (create attendance types, configure posting workflows, set up leave policies) are the same pattern with different intents. Extending the orchestrator is the path, not a new system.

---

### Row 6 — Tech Design Automation (Kavi)
**Status:** In progress. Auto-generate API specs, sequence diagrams, code review.  
**What it is:** Developer tooling. Requirement gap analysis, API spec generation, sequence diagram generation from requirements.

**Where it fits:** Developer tooling — orthogonal to the platform AI architecture. Not in conflict.

**One note:** If this generates API specs from requirements, those specs should be validated against digit-specs as the source of truth. New service specs that deviate from the conventions in digit-specs (clean codes in paths, X-Tenant-ID from Bearer token, etc.) will not be AI-consumable without additional work. Building in a spec linter against digit-specs conventions would preserve the clean-specs property that makes Layer 2 generation feasible.

---

### Row 7 — Log Anomaly Detection with AI Agents (Naresh/Nikhil)
**Status:** In progress (~60% done). Kubernetes cluster monitoring.  
**What it is:** AI agent for Kubernetes log analysis and anomaly detection.

**Where it fits:** Infrastructure/DevOps layer — orthogonal to the platform AI architecture. No conflict.

This is an operations intelligence tool for eGov's own infrastructure, not for DIGIT's AI-readiness. It does not interact with the 4-layer architecture described in this proposal.

---

### Hackathon: AI Co-Pilot for Grievance Redressal (PGR + Urban Services)
**Where it fits:** Layer 3 + Layer 4 for PGR — same pattern as the intern project.

This is the same architecture the Smart Grievance Mapping intern project is proving: analytics microservice + intelligence dashboard + alert engine + voice intake. When the intern project is done in July, the pattern is proven and replicable. This hackathon use case is a direct beneficiary.

---

## The Summary Table

| Initiative | Layer | Fits cleanly? | Key risk if built in isolation |
|---|---|---|---|
| LLM guidance chatbot (Row 2) | Layer 1 | Yes — already integrated | Needs routing to Layer 2 for action queries |
| MCP for PGR (Row 5) | Layer 2 | Partial — needs generalization | PGR-specific means rebuilding for every next service |
| Service Config AI Agent (Row 8) | Layer 2 | Yes — this is the orchestrator | Rebuilding session, confirmation, state-gating from scratch |
| Console config chatbot (Row 13) | Layer 1 + Layer 2 | Partial — RAG half is Layer 1 | RAG can't execute; action half needs orchestrator pattern |
| Conversational data layer (Row 19) | Layer 2 | Partial — needs to be generic | HCM-scoped means re-solving auth + entity resolution per domain |
| HCM Console AI Copilot (Row 22) | Layer 4 | Yes | Bypassing Layer 2 means no shared audit, auth, or confirmation |
| Automate HCM impl tasks (Row 12) | Layer 2 | Yes — orchestrator extension | Scope conflict if built as separate system |
| Tech design automation (Row 6) | Dev tooling | Yes — no conflict | Spec divergence from digit-specs conventions |
| Log anomaly detection (Row 7) | Infrastructure | Yes — orthogonal | None |
| Hackathon: PGR co-pilot | Layer 3 + 4 | Yes | None — benefits from intern project pattern |

---

## What Shared Infrastructure Would Save Each Team

If Layer 2 is built as shared platform infrastructure, every initiative above benefits:

| Capability | Who rebuilds it in isolation | Who gets it free from Layer 2 |
|---|---|---|
| Auth propagation (user JWT, not service account) | Rows 5, 8, 22, 19 | All of them |
| AI proposes → human confirms → executes | Rows 8, 22, 19 | All of them |
| Audit trail per AI operation | Everyone | Everyone |
| Entity resolution (codes → human-readable) | Rows 5, 19 | All of them |
| Keyword fallback when OpenAI is unavailable | Rows 8, 5 | All of them |
| Session management | Rows 8, 5 | All of them |

The platform AI layer is not overhead. It is the difference between each team spending 60% of their time on infrastructure concerns and spending 100% of their time on the domain-specific problem they actually set out to solve.
