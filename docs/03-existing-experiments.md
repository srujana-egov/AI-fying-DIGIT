# Existing Experiments: What Was Built and What Was Learned

Four experiments inform this architecture. Each solved a real problem. Each revealed something about what the AI layer needs.

---

## Experiment 1: DIGIT MCP (ChakshuGautam)
**Context:** DIGIT 2.9. Built to make DIGIT AI-operable via the Model Context Protocol.

### What Was Built
A 6-layer stack:

```
User Interfaces (CLI, MCP callers, test runners)
        ↓
Skills Registry (AI instructions for complex multi-step operations)
        ↓
Tool Registry (61 tools, 14 groups — hand-authored)
        ↓
Semantic Layer (entity resolution: LOC_CITYA_1 → "Ward 4, Chandrapur")
        ↓
Product/Module Layer
        ↓
DIGIT 2.9 Core
```

Key design insight from Chakshu: *"Tool Registry is the main component. MCP is one caller. CLI is another. Tests are a third. No AI is required for the platform to work."*

### What It Solves
- Exposes DIGIT's capability surface to any AI agent via MCP
- Resolves opaque internal codes to human-readable values (semantic layer)
- Provides guided multi-step operations via skills
- Addresses the fact that DIGIT 2.9 APIs are not self-describing enough for direct AI consumption

### What It Reveals
1. **The tool registry exists because 2.9 specs aren't clean.** The 61 tools had to be hand-authored because the underlying API documentation didn't provide enough context for AI consumption.
2. **Skills exist because the platform doesn't enforce process.** When the platform doesn't say "you must create an account before configuring workflow," a skill has to instruct the AI to do it in the right order.
3. **The semantic layer is genuinely needed.** Internal codes (`LOC_CITYA_1`, `DEPT_1`, `pb.amritsar`) are not interpretable by AI or by humans reading AI responses without resolution.
4. **This is a capability surface, not a safety layer.** The MCP stack defines what DIGIT can do. It does not restrict what AI is allowed to do based on deployment state.

### Repo
`digitnxt/digit-mcp` (internal)

---

## Experiment 2: digit-ai-orchestrator (Srujana Dadi)
**Context:** Built as a safety layer — what AI is allowed to do right now, based on deployment state.

### What Was Built

```
POST /mcp/ai (McpController)
        ↓
SessionStore.getSession(X-Session-Id)      [ConcurrentHashMap, future Redis]
        ↓
YES/NO check ──────────────────────────── deterministic, zero AI calls
        ↓ (other input)
AllowedToolsResolver
  ConfigState: account created?   → if NO, only account.create allowed. STOP.
  ConfigState: account configured? → if NO, only account.configure allowed. STOP.
  ConfigState: role exists?        → role.assign requires role. STOP if missing.
  Otherwise: all tools available
        ↓
OpenAiToolSelector (GPT-4o-mini, temperature=0)
  Infers one of 10 intents from natural language
  Fallback: keyword matching if AI call fails
        ↓
AiDecision: EXECUTE or EXPLAIN
  EXPLAIN → store pendingAction in session, return confirmation request
  EXECUTE → call tool
        ↓
Audit log (proposed, not yet built)
```

### Supported Intents
`bootstrap` · `account.configure` · `idgen` · `workflow` · `registry` · `boundary` · `notification` · `user` · `role` · `role.assign`

All 10 are implementation/setup intents — configuring a new city on DIGIT.

### Test Coverage
| Suite | Tests | Coverage |
|---|---|---|
| Intent inference | 148 | All intents, typos, edge cases, adversarial |
| Session integration E2E | 21 | Full flows, YES/NO, non-linear journeys |
| Allowed tools | 5 | State gate scenarios |
| Tool selector | 7 | Prerequisite validation |
| Orchestrator | 2 | Core flow |
| Unit | 2 | Tool implementations |
| Application context | 1 | Spring context loads |
| **Total** | **186** | |

All tests run offline. No API key required. Under 1 minute. Accuracy on intent inference: 95-98%.

### Latency
- Total: 500-800ms
- OpenAI call: ~400ms (80% of total)
- YES/NO path: near-instant (zero AI calls)

### What It Reveals
1. **State-gated tool selection is the right pattern for government operations.** Offering AI only the tools that are valid given current deployment state prevents entire categories of error.
2. **YES/NO confirmation with zero AI involvement is the right confirmation pattern.** Deterministic, instant, no inference risk on the execute path.
3. **10 intents covers the implementation team use case well.** It does not cover city administrator queries, state official queries, or read-only intelligence synthesis — those are a different interaction model.
4. **The orchestrator is the safety layer. The MCP stack is the capability surface. They compose.**

### Repo
`srujana-egov/digit-ai-orchestrator` (private)

---

## Experiment 3: Smart Grievance Mapping (Intern Project)
**Context:** 8-week internship, week 3 of 8 as of July 2026. Due July 30. Students from partner universities.

### What Is Being Built
A complete vertical slice through Layers 3 and 4 for the PGR domain.

**Analytics FastAPI (Python, port 8080):**
```
POST /analyze
  text → urgency (HIGH/MEDIUM/LOW) + urgency_signals[] + suggested_service_codes[]
  Urgency signals: time ("3 months"), vulnerability ("children"), 
                   critical service ("no water"), repetition ("third time")

GET  /recurrence
  → [{ward_id, ward_name, serviceCode, recurring_months[], 
      recurrence_score, is_hotspot, monthly_counts[]}]

POST /severity
  {ward_id, serviceCode, urgency} → severity_score (0-10)

GET  /analytics/complaints?serviceCode=&ward=&from=&to=
  → complaint records for map plotting

Alert Engine (cron / on-demand)
  scan wards → recurrence_score > 0.7 AND current_month in recurring_months
  → Twilio WhatsApp → department engineer
  → Twilio WhatsApp → past complainants in flagged ward
```

**Admin Intelligence Dashboard (React, inside CCRS codebase):**
- New route: `/intelligence`
- OSM map using Leaflet.js (reused from existing CCRS dependency)
- Complaint pins coloured by urgency (red/orange/green)
- Recurrence hotspot choropleth (ward-level, darker = more chronic)
- Predicted risk layer (seasonal patterns)
- Engineer alert inbox + Accept Work Order button
- Citizen communication log

**Voice Intake (added to CCRS citizen complaint form):**
- Microphone button next to description textarea
- Web Speech API → transcribed text → populates description field
- Calls `POST /analyze` → displays top 3 suggested serviceCodes below dropdown
- "AI Suggestion: Storm Water Drain — click to select"
- Citizen can accept or ignore. Dropdown still works normally.

**DIGIT PGR Integration:**
- GCC Chennai historical data (FY 2022-25) loaded into `eg_pgr_service_v2`
- Urgency stored on submission in `additionalDetails` JSONB field
- Work orders created via DIGIT PGR API when engineer accepts from dashboard

### What It Proves
This is not a feature. It is a **replicable pattern**:

```
Domain intelligence microservice (/analyze, /recurrence, /severity)
    +
Voice/AI-assisted citizen intake
    +
Admin intelligence dashboard
    +
Proactive alert engine
```

PGR done → Property Tax next → HCM next → Water & Sewerage next. Each domain gets the same architecture.

---

## Experiment 4: EGOV RAG V5
**Context:** Live. DIGIT Studio documentation chatbot.

### What Was Built
```
User query
    ↓
Predetermined Q&A cache check (pgvector on Neon — instant, zero API cost)
    ↓ cache miss
Query rewriter (OpenAI)
    ↓
Hybrid retrieval: vector search + BM25 full-text search (pgvector on Neon)
    ↓
Cohere reranking (MMR for result diversity)
    ↓
OpenAI streaming answer
    ↓
👍/👎 feedback
    👍 → promoted to predetermined_qa (confidence: 0.7 → rises with repeated votes)
    👎 → logged for improvement
```

Admin dashboard: query history, satisfaction rate, Q&A cache management, content ingestion.

### What It Currently Knows
DIGIT Studio documentation: `studio_chunks.jsonl`, `normalized_chunks.jsonl`, `pitch_deck_chunks.jsonl`

### What It Needs to Know
Full DIGIT documentation: API specs (the 16 digit-specs files), process guides, MDMS configuration references, workflow design patterns, case studies from deployed cities.

### What It Reveals
1. **The feedback loop is the most important feature.** Repeated 👍 promotes answers to cache. The bot gets smarter on the questions real users actually ask. Predetermined Q&A cache means high-frequency questions cost zero API calls.
2. **Hybrid retrieval + reranking is the right retrieval architecture.** Vector search alone misses keyword-specific queries. BM25 + vector + Cohere reranking covers both.
3. **This is Layer 1. It feeds Layer 2.** When the intent classifier in the orchestrator receives a "how do I" question rather than a "do this" command, it should route to RAG V5, not attempt to classify it as an operation intent.

### Repo
`srujana-egov/EGOV_RAG_V5` (public)

---

## What the Four Experiments Tell Us Together

| Question | Answer |
|---|---|
| Does DIGIT need a capability surface? | Yes. Chakshu proved this. |
| Does DIGIT need a safety layer? | Yes. The orchestrator proves this. |
| Are they the same thing? | No. They compose. |
| Does DIGIT need domain intelligence? | Yes. The intern project proves the pattern. |
| Does DIGIT need a knowledge layer? | Yes. RAG V5 proves the architecture. |
| Does any of this replace the platform? | No. Everything calls DIGIT APIs. The platform is the foundation. |
