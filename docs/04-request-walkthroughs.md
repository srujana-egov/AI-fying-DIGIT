# Request Walkthroughs: Interactive vs Scheduled

Two concrete traces through [the architecture](03-minimal-ai-platform.md), one per interaction mode, step by step through every box in the diagram.

---

## 1. Interactive example — a city administrator applies for a trade license via chat

**Consumer:** The city admin types into the HCM console's chat panel (or WhatsApp): "Apply for a trade license for this shop, documents attached."

**MCP Server (first contact):** The request arrives at the MCP server. The admin is already authenticated — the Bearer token is in the session. The LLM picks the tool `certificate_apply`, constructs the params (applicant, documents, certificate type = trade-license). Because this is a write, the confirmation gate kicks in: params get stored as a `pendingAction` in Redis, and the admin sees "I will apply for a Trade License for [shop name] with these documents. Confirm? YES/NO" — showing the actual endpoint and params, not AI prose.

**Human confirms YES:** MCP retrieves the pending action, forwards the HTTP call to Kong with the admin's Bearer token + an idempotency key.

**Kong:** Validates the admin's JWT, injects `X-Tenant-ID` from the token, checks RBAC — confirms this admin is allowed to touch trade-license operations in this tenant — then routes to the Trade License service.

**Application Services:** The Trade License service receives the call, and internally sequences its own dependencies — calls Platform Services: `idgen` to generate an application number, `workflow` to initiate the approval workflow instance, `filestore` to validate the attached documents.

**Audit log:** Written immediately — who asked, what was proposed, that a human said yes, what happened, what the result was.

**Response:** The admin gets back "Application CERT-APP-2026-001234 submitted, pending review" — in one conversational turn.

---

## 2. Scheduled example — the revenue recovery sweep runs on an orchestration engine

**Approval happens once, up front:** a revenue officer or supervisor approved the workflow definition — "every month, find properties with an active trade license but 6+ months behind on water charges, and issue a demand notice." That's the confirmation step for the whole recurring job.

**Trigger:** The engine's cron fires `RevenueRecoveryWorkflow` on the 1st of the month.

**The engine reads first:** calls the MCP tool `property_tax_search` to pull tax records, and `water_sewerage_search` to pull payment history — two different application services, two different tools.

**The engine joins the results** on property identifier (plain workflow code, no DIGIT call) to find the delinquent set.

**The engine writes:** for each match, calls the MCP tool `notification_send` (or a `demand_notice_create`-type tool) — this is the "action" half of "cross-module query + action."

**MCP Server:** no per-item human confirmation (satisfied at step 1), but every notice generated still gets a full audit log entry — same schema as an interactively confirmed write.

**If the engine crashes at property #800 of 5,000:** depending on engine choice, it may resume from there or restart — one of the open questions in the engine decision (see [Mini Projects](06-mini-projects-revised.md) section 6).
