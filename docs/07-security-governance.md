# Security & Governance: Concerns and Mitigations

Every concern listed here was raised in analysis and stress-tested. None are dismissed. Where a mitigation exists, it is described specifically. Where a gap remains, it is named.

---

## Security Concerns

### 1. Prompt Injection on Government Data
**Concern:** An attacker crafts natural language to manipulate the AI into accessing or modifying data the attacker is not authorised to touch.

**Clarification:** DIGIT's code is open source. An attacker already knows the API schema. The concern is not about revealing system design — it is about runtime data access and unauthorised operations.

**Mitigation:** Authorization propagation (see concern 3). The AI layer never holds elevated permissions. DIGIT's own RBAC is the enforcement mechanism. If an attacker's identity doesn't have access to a resource, the DIGIT API rejects the call regardless of how the AI constructed the request. The AI cannot grant permissions that the underlying RBAC does not grant.

---

### 2. PII Sent to External AI Providers
**Concern:** Complaint descriptions and other DIGIT data containing citizen PII are sent to OpenAI (a US commercial service) for classification.

**Current State:** PII is masked before data is shared externally (confirmed practice — same standard applied to student data in the intern project).

**Mitigation:**
- PII masking happens at the API boundary before text reaches OpenAI — not just at the data handoff layer
- Classification task does not require PII: "drain overflow near house" classifies identically whether or not the citizen's name is present
- Long-term: self-hosted open-source model for classification (Mistral 7B, LLaMA 3) handles intent classification at acceptable accuracy without any external API call

---

### 3. Authorization Not Enforced at the AI Layer
**Concern:** If the MCP server uses a service account with broad permissions, DIGIT's RBAC is bypassed by the AI layer.

**Mitigation (design requirement):** The MCP server must never use a service account. The authenticated user's Bearer token is forwarded on every DIGIT API call. The AI is a translator, not a permission granter.

```
User authenticates → JWT issued by DIGIT
↓
MCP server receives request with JWT
↓
Every DIGIT API call: Authorization: Bearer {user's JWT}
↓
DIGIT API enforces RBAC as normal
↓
AI cannot escalate privileges the user does not already hold
```

This is a design constraint, not an afterthought. It must be enforced in the MCP server implementation.

---

### 4. Session Hijacking on the Confirmation Layer
**Concern:** If session IDs are guessable or shared, an attacker hijacks another user's pending action and confirms it on their behalf.

**Mitigation:**
- Session IDs: UUID v4 (cryptographically random, not sequential or guessable)
- Pending action bound to authenticated user identity, not just session header — confirmation validates that the confirming user is the same user who initiated the action
- Pending action TTL: 10 minutes. Unconfirmed actions expire and must be re-proposed
- High-risk operations (role assignment, demand generation): require re-authentication before confirmation executes

---

### 5. Alert Engine as Phishing Vector
**Concern:** Twilio WhatsApp alerts appear to come from the municipality. An attacker who can trigger the alert engine sends arbitrary content to citizens.

**Mitigation — three constraints that eliminate the attack surface:**
1. **Template-only content:** Alert messages are strictly templated with DIGIT data variables. No AI-generated free text in alerts. No external URLs. No payment links. Ever.
2. **Rate limiting:** Maximum N alerts per ward per day, enforced in code before any Twilio call
3. **Verified sender:** WhatsApp Business API with official green tick for the municipality. Citizens can independently verify the institutional sender identity

An attacker cannot inject arbitrary content into a template-only alert system.

---

## Technical Concerns

### 6. 2-5% AI Misclassification on Write Operations
**Concern:** 95-98% accuracy on intent inference means 2-5% of queries result in wrong classification. For government records, this is not acceptable.

**Mitigation:**
- **Confidence threshold gate:** If classifier confidence < 80%, do not propose an action. Ask the user to clarify. Only high-confidence classifications proceed to confirmation.
- **Plain-English confirmation before execute:** The confirmation message shows the exact API call parameters, not AI-generated prose. "I will create workflow PROPERTY_TAX_ASSESSMENT with states INITIATED → PENDING → APPROVED" — the user reads and confirms the actual change, not a description of it.
- **Human confirmation is mandatory for all writes.** AI misclassification that produces a wrong proposal can be caught at confirmation. A user who reads the confirmation message catches it before execution.
- **High-risk writes:** Two-step confirmation. Confirm the action type, then confirm the exact parameters. Role assignment and demand generation fall in this category.

---

### 7. OpenAI Dependency
**Concern:** Every AI operation depends on OpenAI's availability, pricing, and model versioning. A model update could break accuracy overnight.

**Mitigation — three layers:**
1. **Keyword matching fallback** (already built in orchestrator): If OpenAI call fails, keyword matching handles common intents. Operations continue at reduced NL capability, not zero.
2. **Predetermined Q&A cache** (already in RAG V5): High-frequency queries are served from cache. Zero OpenAI dependency for cached answers.
3. **Abstracted AI provider interface** (design requirement): The intent classifier is behind an interface. Swapping OpenAI for a self-hosted Mistral 7B is a configuration change, not an architecture change. Intent classification into 50 intents is well within open-source model capability.

---

### 8. No Audit Trail
**Concern:** Government operations require legally defensible audit records. "The AI did it" is not sufficient.

**Mitigation:** A structured audit log for every AI-assisted operation.

**Schema:**
```json
{
  "timestamp": "ISO 8601",
  "userId": "ramesh.kumar",
  "tenantId": "pb.chandrapur",
  "sessionId": "uuid",
  "userQuery": "the raw input",
  "inferredIntent": "workflow",
  "confidence": 0.94,
  "allowedTools": ["workflow.create", "workflow.list"],
  "proposedAction": {
    "tool": "workflow.create",
    "params": {"processCode": "PROPERTY_TAX_ASSESSMENT", "states": [...]}
  },
  "userConfirmation": "yes",
  "digitApiCall": "POST /workflow/v3/definition",
  "digitApiResponse": {"success": true, "processCode": "PROPERTY_TAX_ASSESSMENT"},
  "executionResult": "success"
}
```

Append-only. Immutable. This is the RTI trail. "The AI classified the intent as workflow with 94% confidence, the user confirmed, the system executed POST /workflow/v3/definition with these parameters" — legally defensible.

1-2 weeks to build on top of the orchestrator.

---

### 9. No Dry-Run Mode
**Concern:** DIGIT write APIs don't have a dry-run mode. The confirmation message is AI-generated prose, not a deterministic preview of the state change.

**Mitigation (short-term):** Read-before-write. For operations modifying existing state, first READ the current state and show it alongside the proposed change:
> "Current state: No workflow exists for PROPERTY_TAX. Proposed: Create PROPERTY_TAX_ASSESSMENT with 3 states. After confirmation, this cannot be undone via this interface."

**Mitigation (long-term — platform roadmap):** Add `?dryRun=true` to DIGIT 3.0 write APIs. Returns what would change without executing. This is the platform team's work, not the AI layer's work. It is the single most valuable platform addition for safe AI-assisted operations.

---

### 10. Cost at Scale
**Concern:** 600 cities, millions of interactions, variable OpenAI costs on fixed government IT budgets.

**Mitigation:**
- **Cache-first:** RAG V5's predetermined Q&A cache + Redis intent cache reduce OpenAI calls by an estimated 70-80% after warm-up. High-frequency government queries (complaint status, workflow steps, setup questions) are repetitive by nature.
- **Self-hosted classification:** Intent classification does not require GPT-4 capability. Self-hosted Mistral 7B or equivalent handles 50-intent classification at near-zero marginal cost.
- **Budget as infrastructure:** AI inference cost is analogous to SMS notification cost — already a known line item in DIGIT city deployments. Per-city, per-month, estimable upfront. Not a surprise at billing.

---

## Governance Concerns

### 11. Accountability Gap
**Concern:** If AI produces a wrong work order or wrong tax demand, who is accountable?

**Answer:** The human who confirmed it.

The confirmation layer is the legal line. The engineer who clicks "Accept Work Order" is accountable for that work order — the same way they have always been accountable for work orders they approve. The AI generated a proposal. The human approved it. The human is the decision-maker.

The audit log proves this: it records that the human confirmed the proposed action with full visibility into what the action was.

This is the same model as e-signatures in government: technology assists, human authorizes, human is accountable. The AI is a proposal engine. It is not a decision-maker and must never be positioned as one.

---

### 12. Bias in Intelligence Models
**Concern:** Recurrence and urgency models trained on historical complaint data will systematically under-serve wards with lower digital literacy, fewer smartphones, or language barriers — because those wards have historically under-reported problems.

**Mitigation:**
- **Infrastructure-based risk signals** (not complaint-based): OSM geo-risk data — wards near water bodies, low-lying topography, older drainage infrastructure — provides risk assessment independent of complaint history. The intern project's geo-risk layers implement this.
- **Accessibility-adjusted normalization:** Use complaints per 100 households with smartphone access as the metric, not raw complaint count.
- **Equity audit as a scheduled job:** Automated monthly review of which wards receive preventive maintenance actions. If results systematically skew to higher-income zones, the model requires correction. Not a manual process.

---

### 13. Explainability Under RTI
**Concern:** India's Right to Information Act requires government decisions to be explainable. "A machine learning model produced a score" fails RTI scrutiny.

**Mitigation — use interpretable models only for decision-affecting scores:**

**Urgency scorer:** Rule-based with explicit weights. Not a neural network. Every score is reproducible by any officer from the same input text.
```
RESULT: HIGH urgency
REASON: time signal detected ("3 months" → +2 pts)
        vulnerability signal detected ("children" → +2 pts)
        critical service signal detected ("no water" → +3 pts)
        Total: 7/10 → HIGH
```

**Recurrence detector:** Pure statistical logic.
```
RESULT: Ward 12 flagged as recurrence hotspot
REASON: Waterlogging complaints detected in July 2022, July 2023, July 2024
        3 of 3 years in dataset = 100% recurrence score
        is_hotspot: true (threshold: years_with_complaints >= 2)
```

Both are RTI-answerable. Both are reproducible. Both are documentable as public policy methodology.

---

### 14. State/Political Resistance to Intelligence Dashboards
**Concern:** A system that surfaces comparative performance across cities creates political sensitivity. Department heads and state officials may block adoption.

**Reality:** This concern is real and cannot be resolved by technical design alone.

**Approach:**
- **Cities own their data:** Each city's intelligence dashboard is visible to that city by default. State access requires explicit opt-in from the city.
- **Improvement framing, not ranking framing:** The dashboard shows each city's performance over time — not a leaderboard vs other cities. "Chandrapur improved complaint resolution by 22% this quarter" is a headline a city commissioner wants. "Chandrapur ranks 8th worst in the state" is not.
- **Lead with willing champions:** Find city commissioners who want this capability. Let them demonstrate results. Peer persuasion from a commissioner who used intelligence dashboards to justify additional budget is more effective than top-down mandate.
- **State access is aggregated:** State-level dashboards show domain-level trends across all cities — not city-level scorecards. Which service types are most problematic statewide. Which interventions have shown results.

Political resistance is not a reason to not build this. It is a reason to build it city-first, improvement-framed, and opt-in for state sharing.
