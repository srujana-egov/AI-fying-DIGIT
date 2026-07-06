# Who Actually Talks to DIGIT at the Platform Level

A precise stakeholder map matters because different stakeholders have different interaction patterns, different risk tolerances for AI errors, and different needs from the AI layer.

---

## The Boundary

```
Citizens
    → apps built on DIGIT (PuraSeva, CCRS, HCM mobile apps)
        → DIGIT APIs
        
App partners (implementing cities, building apps)
    → DIGIT APIs directly
    → digit-cli
    → digit-client (Java)
    → [proposed] AI layer
```

Citizens never interact with DIGIT directly. They interact with apps. Whether those apps use AI is the app partner's choice — exactly as whether they use React or Angular is their choice. This proposal does not change anything about citizen-facing AI.

---

## Stakeholders Who Interact with DIGIT Directly

### 1. Implementation Teams
**Who:** eGov engineers, state implementation partners, city IT teams onboarding a new ULB.

**What they need:**
- Understand the platform quickly: what APIs exist, what sequences are required, what a valid workflow definition looks like
- Currently: digit-specs (16 OpenAPI files), digit-client (Java), digit-cli, documentation

**What AI adds:**
- Documentation Q&A via RAG: "what fields are required for workflow creation?", "what's the correct order for city setup?" — Layer 1, not execution
- Spec exploration and test case generation in developer/sandbox environments

**What AI does NOT add here:**
- Configuration of production environments: city configuration happens once per city, not at scale. This is a one-time operation, not a recurring AI problem.
- Cross-tenant template borrowing: DIGIT's tenant isolation means one city's boundary hierarchy, MDMS config, and workflow definitions cannot be read by another city's setup process. "Import from similar cities" is architecturally impossible.
- Zero-knowledge production onboarding: implementation teams arrive trained. The onboarding scenario where an org configures a production DIGIT environment with no prior knowledge does not reflect real deployments. Sandbox and developer environments are different.

**Risk tolerance:** N/A for production writes — AI-assisted production configuration is out of scope. Developer environments: high (sandboxed).

**Where AI genuinely helps this group:** Layer 1 — documentation and spec Q&A. Not Layer 2 execution on production systems.

---

### 2. City Administrators / Municipal Commissioners
**Who:** The ULB Commissioner, Deputy Commissioner, department heads.

**What they need:**
- Status across all city services: complaints, collection, workforce, works
- Intelligence: which wards are at risk, which departments are underperforming, what needs immediate attention
- Currently: DSS dashboards (static), manual reports, phone calls to department heads

**What AI adds:**
- Natural language queries over live DIGIT data
- Cross-domain synthesis (PGR + HCM + Finance in one answer)
- Proactive alerts before they ask

**Risk tolerance:** High for reads. Low for writes. A commissioner query that returns wrong data is bad. An AI-initiated state change in government records without confirmation is unacceptable.

---

### 3. State Government Officials
**Who:** State Urban Development Department officers, NUDM programme managers.

**What they need:**
- Cross-city intelligence: which cities are performing, which need intervention
- Scheme implementation tracking across all ULBs
- Budget utilisation intelligence
- Currently: Annual reports, district-level data collection, manual compilation

**What AI adds:**
- Real-time cross-city queries
- Aggregated intelligence without waiting for city reports
- Anomaly detection: which cities have unusual patterns

**Risk tolerance:** Read-only. No writes needed. Data accuracy is critical — state decisions affect budget allocation.

---

### 4. Developers / Technical Teams
**Who:** Engineers building services on DIGIT 3.0, writing integrations, testing APIs.

**What they need:**
- Understand what APIs exist and how to call them
- Navigate process definitions and workflow states
- Test API behaviour without full environment setup
- Currently: digit-specs (16 OpenAPI files), digit-client (Java), digit-cli, documentation

**What AI adds:**
- Natural language API exploration ("what fields are required for workflow creation?")
- Code generation from specs
- Test case generation

**Risk tolerance:** High. Developer environments are sandboxed.

---

### 5. AI Agents
**Who:** Claude, GPT-4, custom orchestrators, automated pipelines — acting on behalf of any of the above stakeholders.

**What they need:**
- A structured, well-documented tool surface (MCP server)
- Platform-level knowledge of what operations are valid in the current state
- Entity resolution: receive human-readable values, not internal codes
- Confirmation before executing write operations

**Risk tolerance:** The AI agent itself has no risk tolerance — it's a tool. The risk tolerance is the stakeholder's. The platform enforces it via the confirmation layer.

---

## What This Means for the AI Layer Design

| Stakeholder | Read ops | Write ops | Confirmation needed | AI value |
|---|---|---|---|---|
| Implementation teams | Low risk | Out of scope (production) | N/A | Layer 1 only — documentation Q&A, sandbox exploration |
| City administrators | Low risk | High risk | Yes, mandatory | Layer 2 + Layer 3 — operational intelligence, cross-domain synthesis |
| State officials | Low risk | None needed | N/A | Layer 2 + Layer 3 — cross-city intelligence, anomaly detection |
| Developers | Low risk | Low risk (dev env) | Recommended | Layer 1 — spec exploration, code generation |
| AI agents | Inherited from caller | Inherited from caller | Always | All layers — executes on behalf of above stakeholders |

The AI layer must propagate the authenticated user's identity to DIGIT's APIs. The AI never holds a service account with elevated permissions. DIGIT's own RBAC is the enforcement mechanism — the AI layer must not bypass it.
