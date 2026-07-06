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

- **Progressive disclosure:** 8 core tools load on startup; agents call `enable_tools` to unlock more. Solves context rot — 61 tools in context degrades AI tool-selection accuracy; starting with 8 keeps it lean without sacrificing capability.
- **Auto-generated CLI:** 57 commands, zero per-tool code. The CLI is a derived artifact of the same registry that powers MCP — add one tool definition, get CLI command + MCP tool + test coverage automatically.
- **Unit economics:** $40–$120/month per city in AI inference cost → $10,000–$20,000/month in staff time recovered ≈ **100x ROI**. Economic case is independent of platform version.

### What It Solves
- Exposes DIGIT's capability surface to any AI agent via MCP
- Addresses context rot through progressive disclosure (8 tools → unlock on demand)
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
Interaction diagrams (from P2a and P2b), process guides, MDMS configuration references, workflow design patterns, case studies from deployed cities. API specs go into the MCP server via openapi-generator — not into RAG.

### What It Reveals
1. **The feedback loop is the most important feature.** Repeated 👍 promotes answers to cache. The bot gets smarter on the questions real users actually ask. Predetermined Q&A cache means high-frequency questions cost zero API calls.
2. **Hybrid retrieval + reranking is the right retrieval architecture.** Vector search alone misses keyword-specific queries. BM25 + vector + Cohere reranking covers both.
3. **This is Layer 1. It feeds Layer 2.** When the intent classifier in the orchestrator receives a "how do I" question rather than a "do this" command, it should route to RAG V5, not attempt to classify it as an operation intent.

### Repo
`srujana-egov/EGOV_RAG_V5` (public)

---

---

## PES Intern Projects: HCM Intelligence (In Progress)

Two projects being built by PES interns, both HCM-specific, both proving the generalizable patterns at a different scale — country-level health campaigns in LMICs (Chad, Sierra Leone, Nigeria, Liberia).

---

### HCM Project 1: Population Denominator Intelligence

**The problem:** Health campaigns like polio, SMC, and BedNet measure coverage as `beneficiaries reached / enrolled households`. But enumeration misses settlements never mapped, households not in any registry, communities invisible to field teams. The result: teams know how many children they found, not how many they should have reached. True coverage is unknowable. Missed children stay missed.

**What is being built:**

```
WorldPop API (high-resolution raster population estimates, peer-reviewed, LMIC coverage)
    +
Google Open Buildings + Population Dynamics
    (building footprints from satellite imagery + location signatures)
    ↓
Cross-reference against HCM enrolled household data
    ↓
Gap detection: areas where population estimates exceed enrolled households
    ↓
GIS map: visual prompt to field supervisor:
    "Population data suggests people live here — no households registered"
    ↓
Corrected denominator fed back into HCM microplanning module
```

**Pattern this proves:** Pattern 2 (GIS Cross-Reference) at country scale. Satellite and population models as ground truth against HCM registry — the same abstraction as Property Tax area discrepancy or Trade License land-use mismatch, applied to health campaign population coverage.

**What it reveals:**
- Two independent GIS signals (WorldPop raster + Google building footprints) can be cross-referenced against each other AND against DIGIT registry data — increasing confidence in gap detection
- The output writes back into the HCM microplanning module as a corrected denominator — not a dashboard, but a data correction that changes the campaign plan
- Country-scale pilot (not city-scale) means the GIS cross-reference pattern works at the level of national health programmes, not just urban governance

---

### HCM Project 2: On-Device Beneficiary Deduplication

**The problem:** In campaigns like polio and SMC, field workers register the same beneficiary multiple times — across cycles, across teams, or due to offline sync conflicts. Accurate cohort tracking (who received all campaign cycles) is impossible. True coverage across a campaign series is unknowable.

**What is being built:**

```
Field worker opens HCM Flutter app
    ↓
Enters beneficiary: name + age + location
    ↓
On-device fuzzy search against existing records (before submission)
    name similarity + age match + geographic proximity
    ↓
Likely match found → warning surfaced:
    "Possible duplicate: Amina Kone, age 3, 200m from here. Same child?"
    Worker confirms or proceeds
    ↓
If confirmed duplicate: merge or skip
If not: submit as new

Secondary: MOSIP deduplication engine integration for robust identity
          matching at scale (biometric-backed)
```

**Reusable output:** Flutter library published as a standalone package — any DIGIT/HCM app can consume it. Deduplication does not need to be rebuilt per campaign or per country.

**Dataset:** Synthetic (real campaign data has PII — cannot be shared for development).

**Pattern this proves:** Pattern 4 (Deduplication), with two important distinctions from server-side deduplication:
1. **On-device**: works offline (critical for field conditions in LMICs where connectivity is absent)
2. **Before submission**: catches duplicates at the point of creation, not after they are already in the system — the lowest-cost point of intervention

**What it reveals:**
- The deduplication pattern should run at point of submission, not post-hoc — the earlier the intervention, the lower the data repair cost
- Flutter library as a reusable component is the right delivery model — proves the pattern once, every campaign inherits it without rebuild
- MOSIP integration shows the pattern connects to national identity infrastructure, not just local campaign records

---

## What the Six Experiments Tell Us Together

| Question | Answer |
|---|---|
| Does DIGIT need a capability surface? | Yes. The DIGIT MCP experiment proved this. |
| Does DIGIT need a safety layer? | Yes. The orchestrator proves this. |
| Are they the same thing? | No. They compose. |
| Does DIGIT need domain intelligence? | Yes. The PGR intern project proves the flagging pattern. |
| Does DIGIT need a documentation Q&A layer? | Yes. RAG V5 proves the architecture — for "how do I" questions, not API execution. |
| Does GIS cross-reference work at country scale? | Yes. HCM population denominator project proves it — WorldPop + Google Open Buildings vs HCM enrollment at country level in LMICs. |
| Does deduplication need to be on-device? | Yes, for field conditions. HCM dedup project proves on-device fuzzy matching before submission is the right point of intervention. Offline-capable. Published as reusable Flutter library. |
| Does AI flagging generalize beyond PGR? | Yes — same pattern across all 20 eGov products. See doc 14. |
| Should dashboards be built on top of flags? | No. PowerBI, DSS, and existing tools do this. eGov flags records in DIGIT via API. Consumers visualize with their tools. |
| Does any of this replace the platform? | No. Everything calls DIGIT APIs. The platform is the foundation. |
