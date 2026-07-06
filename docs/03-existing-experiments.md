# Existing Experiments: What Was Built and What Was Learned

Four experiments inform this architecture. Each solved a real problem. Each revealed something about what the AI layer needs.

---

## Experiment 1: DIGIT MCP
**Context:** DIGIT 2.9. Built to make DIGIT AI-operable via the Model Context Protocol.

### What Was Built
A 6-layer stack:

```
User Interfaces (CLI, MCP callers, test runners)
        ↓
Skills Registry (AI instructions for complex multi-step operations)
        ↓
Tool Registry (61 tools, 14 groups — one definition powers all interfaces)
        ↓
Semantic Layer (@digit-mcp/data-provider: LOC_CITYA_1 → "Ward 4, Chandrapur")
        ↓
Product/Module Layer
        ↓
DIGIT 2.9 Core
```

Key design insight: *"Tool Registry is the main component. MCP is one caller. CLI is another. Tests are a third. No AI is required for the platform to work."*

### What Was Verified (March 2026)
- **61 tools** across 14 domain groups: core, docs, mdms, boundary, pgr, employees, admin, monitoring, tracing, encryption, localization, masters, idgen, location
- **Dual transport:** stdio for local development · HTTP Streamable for cloud deployment
- **Progressive disclosure:** 8 core tools load on startup. Agents call `enable_tools` to unlock more as needed. This directly addresses context rot — starting with 61 tools degrades AI accuracy on tool selection; starting with 8 keeps the agent's working context lean without sacrificing capability.
- **Auto-generated CLI:** 57 commands, zero per-tool code. Adding a tool to the registry automatically produces a CLI command. Three output modes: JSON (piped), table (TTY), plain (scripting). The CLI is a *derived artifact* of the registry, not a separate codebase.
- **`@digit-mcp/data-provider` npm package:** Entity resolution + cross-service connection. `LOC_CITYA_1 → Ward 4 · North Zone · pg.citya`. `COMPLAINT_TYPE_1 → Garbage Not Collected`. Every consumer uses it without knowing DIGIT internals.
- **Multi-tenant:** `pg.citya → pg` auto-derived
- **Security:** Prompt injection defense + argument sanitization built in
- **180 tests:** 127 integration + 53 agent safety tests
- **Observability:** Grafana Tempo tracing, Kafka lag monitoring, DB row counts
- **Unit economics:** $40–$120/month per city in AI inference cost → $10,000–$20,000/month in staff time recovered ≈ **100x ROI**

### What It Solves
- Exposes DIGIT's capability surface to any AI agent via MCP
- Resolves opaque internal codes to human-readable values (semantic layer)
- Provides guided multi-step operations via skills
- Addresses context rot through progressive disclosure (8 tools → unlock on demand)
- Addresses the fact that DIGIT 2.9 APIs are not self-describing enough for direct AI consumption
- Makes DIGIT "headless": any HTTP client (WhatsApp bot, ERP, voice assistant) becomes a DIGIT frontend

### The 6 Use Cases Built Around
1. **Start a business** — Trade License + Fire NOC + Water connection in parallel
2. **Annual renewal at scale** — automated campaign across all licence holders
3. **Revenue recovery** — Property Tax + Water & Sewerage cross-module collection intelligence
4. **Post-disaster complaint triage** — prioritisation and assignment during flood events
5. **Building permit diagnosis** — full permit status + blockers in one query
6. **Commissioner's morning brief** — city-wide operational summary generated on demand

### What It Reveals
1. **The tool registry exists because 2.9 specs aren't clean.** The 61 tools had to be hand-authored because the underlying API documentation didn't provide enough context for AI consumption.
2. **Skills serve two different purposes.** Some skills compensate for the platform not enforcing process order. Others are developer productivity aids (`create_digit_ui`, `create_chart`, `debug`, `check_workflow_health`) — these are valid regardless of how clean the platform specs are.
3. **Progressive disclosure is a design requirement, not a workaround.** Even with perfect specs, an agent given 100 tools loses accuracy. Starting minimal and unlocking on demand is the right pattern.
4. **The semantic layer is genuinely needed.** Internal codes (`LOC_CITYA_1`, `DEPT_1`, `pb.amritsar`) are not interpretable by AI or by humans reading AI responses without resolution.
5. **"Tools encode judgment, not just structure."** Even perfect API docs can't encode which of 26 services handles a given intent, what sequence achieves a multi-step workflow, or how to recover from partial failure. Tools encode that judgment. An LLM reconstructing it from docs does so probabilistically — inconsistently.
6. **This is a capability surface, not a safety layer.** The MCP stack defines what DIGIT can do. It does not restrict what AI is allowed to do based on deployment state.

### Repo
`digitnxt/digit-mcp` (internal)

---

## Experiment 2: digit-ai-orchestrator
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

### What It Proves — and What It Does Not

**What is replicable:**

```
AI Flagging Microservice
  — submitted data inconsistency detection
  — example: complaint text says "drain overflow" but
    citizen selected "Road Repair" → flag mismatch
  — generalizes to: any DIGIT data entry where declared
    values can be checked against expected values

Proactive Alert Engine
  — recurrence detection → alert before complaint arrives
  — seasonal patterns → preventive action in June, not July

AI-assisted citizen intake
  — voice → text → AI service code suggestion
  — citizen can accept or override; dropdown still works normally
```

**What is NOT replicable as a platform pattern:**

The Admin Intelligence Dashboard (map, choropleth, alert inbox) — PowerBI, DSS, and similar tools already do visualization. eGov does not need to build another dashboard. The intelligence layer surfaces flags and alerts via DIGIT APIs. Revenue officials and administrators use their existing visualization tools on top.

**The generalizable abstraction is AI flagging:**

| Domain | Submitted data | External signal | Flag |
|---|---|---|---|
| PGR | Selected service code | AI text classification | Mismatch → flag for review |
| HCM | Age + marital status | Domain rules | Inconsistency → flag |
| HCM | Age + pregnancy field | Age rules (under 16) | Flag for field verification |
| Trade License | Declared use (residential) | GIS land-use data | Revenue gap → flag |
| Property Tax | Declared floor area | Satellite measurement | Under-declaration → flag |

The pattern: AI compares declared/submitted values in DIGIT records against expected values (from rules, models, or external data). Flags are written back to DIGIT records. No separate dashboard — query flagged records via DIGIT APIs.

PGR done → HCM next → Trade License / Property Tax (GIS-based revenue intelligence) next.

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
| Does DIGIT need a capability surface? | Yes. The DIGIT MCP experiment proved this. |
| Does DIGIT need a safety layer? | Yes. The orchestrator proves this. |
| Are they the same thing? | No. They compose. |
| Does DIGIT need domain intelligence? | Yes. The intern project proves the flagging pattern. |
| Does DIGIT need a knowledge layer? | Yes. RAG V5 proves the architecture. |
| Does AI flagging generalize beyond PGR? | Yes — HCM (age/status rules), Trade License (GIS vs declared use), Property Tax (area discrepancy), Water (consumption vs category). Same pattern, different domain rules and external signals. |
| Should dashboards be built on top of flags? | No. PowerBI, DSS, and existing visualization tools are the right layer for this. eGov builds the intelligence and flags records via DIGIT APIs. The consumer visualizes. |
| Does any of this replace the platform? | No. Everything calls DIGIT APIs. The platform is the foundation. |
