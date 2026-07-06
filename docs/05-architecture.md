# The 4-Layer Architecture

## Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 4: APPLICATIONS                            │
│                                                                     │
│  ┌──────────────────────────┐  ┌─────────────────────────────────┐  │
│  │   CCRS React Frontend    │  │  Admin Intelligence Dashboard   │  │
│  │   [INTERN - IN PROGRESS] │  │  [INTERN - IN PROGRESS]         │  │
│  │                          │  │                                 │  │
│  │  Citizen complaint form  │  │  Route: /intelligence           │  │
│  │  + microphone button     │  │  OSM map (Leaflet.js)           │  │
│  │  + AI suggestion strip   │  │  Urgency pins (red/orange/green)│  │
│  │  Web Speech API          │  │  Recurrence choropleth          │  │
│  │  English + Tamil         │  │  Predicted risk layer           │  │
│  │                          │  │  Engineer alert inbox           │  │
│  │                          │  │  Accept Work Order button       │  │
│  └──────────┬───────────────┘  └──────────────┬──────────────────┘  │
│             │ POST /analyze                    │ GET /analytics/     │
│             │ (on voice input)                 │ GET /recurrence     │
└─────────────┼──────────────────────────────────┼─────────────────────┘
              │                                  │
┌─────────────┼──────────────────────────────────┼─────────────────────┐
│             ↓      LAYER 3: INTELLIGENCE        ↓                    │
│                                                                      │
│         ┌────────────────────────────────────────────────────────┐   │
│         │           Analytics FastAPI (Python)                   │   │
│         │           [INTERN - IN PROGRESS]                       │   │
│         │                                                        │   │
│         │  POST /analyze                                         │   │
│         │    text → urgency (H/M/L) + signals[] + serviceCodes[]│   │
│         │                                                        │   │
│         │  GET  /recurrence                                      │   │
│         │    → {ward_id, recurring_months[], recurrence_score,   │   │
│         │        is_hotspot, monthly_counts[]}                   │   │
│         │                                                        │   │
│         │  POST /severity                                        │   │
│         │    {ward_id, serviceCode, urgency} → score (0-10)     │   │
│         │                                                        │   │
│         │  GET  /analytics/complaints                            │   │
│         │    ?serviceCode=&ward=&from=&to=                       │   │
│         │    → complaint records for map plotting                │   │
│         │                                                        │   │
│         │  Alert Engine (cron / on-demand)                       │   │
│         │    scan wards → recurrence_score > 0.7                │   │
│         │    AND current_month in recurring_months               │   │
│         │    → Twilio WhatsApp → engineer                        │   │
│         │    → Twilio WhatsApp → past complainants in ward       │   │
│         └────────────────────┬───────────────────────────────────┘   │
│                              │ SQL on DIGIT PGR database             │
└──────────────────────────────┼──────────────────────────────────────┘
                               │
                    eg_pgr_service_v2 / eg_pgr_address_v2

┌──────────────────────────────────────────────────────────────────────┐
│                    LAYER 2: PLATFORM AI-READINESS                    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │              digit-ai-orchestrator [BUILT]                     │  │
│  │                                                                │  │
│  │  POST /mcp/ai {message, X-Session-Id}                         │  │
│  │       ↓                                                        │  │
│  │  SessionStore.getSession(sessionId)   [ConcurrentHashMap]      │  │
│  │       ↓                                                        │  │
│  │  Is message "yes" or "no"?                                     │  │
│  │    YES → retrieve pendingAction → execute (zero AI calls)      │  │
│  │    NO  → clear pendingAction → done (zero AI calls)            │  │
│  │    other ↓                                                     │  │
│  │  AllowedToolsResolver(ConfigState)                             │  │
│  │    account created?      → NO: only account.create. STOP.     │  │
│  │    account configured?   → NO: only account.configure. STOP.  │  │
│  │    role + user exist?    → NO: block role.assign. STOP.       │  │
│  │    all clear             → full tool surface                   │  │
│  │       ↓                                                        │  │
│  │  GPT-4o-mini (temp=0) → infer one of 10 intents               │  │
│  │  Fallback: keyword matching if AI call fails                   │  │
│  │       ↓                                                        │  │
│  │  AiDecision: EXECUTE or EXPLAIN                                │  │
│  │    EXPLAIN → store pendingAction, return confirmation request  │  │
│  │    EXECUTE → call tool → call DIGIT API                        │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │         MCP Server [TO BUILD — Mini Project 3]                 │  │
│  │                                                                │  │
│  │  Source: digitnxt/digit-specs (16 OpenAPI 3.0.3 files)        │  │
│  │  Generation: OpenAPI Generator → MCP tool definitions          │  │
│  │  Curation: thin config file (which endpoints, descriptions)    │  │
│  │                                                                │  │
│  │  Exposes all DIGIT 3.0 APIs as MCP tools                      │  │
│  │  Forwards: authenticated user's Bearer token                   │  │
│  │  DIGIT RBAC enforced downstream — no service account           │  │
│  │                                                                │  │
│  │  Called by: digit-ai-orchestrator, Claude, GPT, any MCP agent │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │       Entity Resolution Service [TO BUILD — Mini Project 4]    │  │
│  │                                                                │  │
│  │  GET /resolve?code=LOC_CITYA_1  → "Ward 4, Chandrapur North"  │  │
│  │  GET /resolve?code=StormWaterDrain → "Storm Water Drain Repair"│  │
│  │  GET /hierarchy?wardCode=WARD_04 → {ward, zone, district, state}│  │
│  │                                                                │  │
│  │  Source: DIGIT localization service (already exists)           │  │
│  │  Cache: Redis (codes change infrequently)                      │  │
│  │                                                                │  │
│  │  Used by: MCP server (enriches API responses before returning) │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │         digit-cli [EXISTS — DIGIT 3.0]                         │  │
│  │                                                                │  │
│  │  Alternative interface for implementation teams                │  │
│  │  Go, cross-platform                                            │  │
│  │  Same DIGIT APIs as orchestrator — different interface         │  │
│  │  Scripting, automation, CI/CD integration                      │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │         digit-client [EXISTS — DIGIT 3.0]                      │  │
│  │                                                                │  │
│  │  Java Spring typed HTTP wrapper                                │  │
│  │  For Java microservices built on DIGIT                         │  │
│  │  Used by: CCRS backend, PGR backend, other Java services       │  │
│  │  NOT used by: AI agents, Python services, CLI                  │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                       LAYER 1: KNOWLEDGE                             │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                  EGOV RAG V5 [LIVE]                            │  │
│  │                                                                │  │
│  │  User query                                                    │  │
│  │       ↓                                                        │  │
│  │  Predetermined Q&A cache check (Neon pgvector — zero cost)     │  │
│  │       ↓ miss                                                   │  │
│  │  Query rewriter (OpenAI)                                       │  │
│  │       ↓                                                        │  │
│  │  Hybrid: vector search + BM25 full-text (pgvector on Neon)    │  │
│  │       ↓                                                        │  │
│  │  Cohere reranking (MMR diversity)                              │  │
│  │       ↓                                                        │  │
│  │  OpenAI streaming answer                                       │  │
│  │       ↓                                                        │  │
│  │  👍 → promote to cache (confidence 0.7 → rises)               │  │
│  │  👎 → logged for review                                        │  │
│  │                                                                │  │
│  │  Currently: DIGIT Studio documentation                         │  │
│  │  Roadmap: full DIGIT docs, digit-specs, process guides         │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                    DIGIT 3.0 CORE (foundation)                       │
│                                                                      │
│  DIGIT APIs: workflow · individual · pgr · idgen · mdms ·            │
│  boundary · billing · notification · filestore · localization ·      │
│  registry · otp · account                                            │
│                                                                      │
│  digit-specs: 16 OpenAPI 3.0.3 files (source of truth)              │
│  → input to MCP server generation                                    │
│  → input to RAG V5 knowledge expansion                               │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Data Flows

### Flow 1: Implementation Team Sets Up a City

```
Engineer: "configure workflow for property tax"
    → POST /mcp/ai (X-Session-Id: eng-001)
    → SessionStore: new session
    → YES/NO check: "configure workflow" → not yes/no
    → AllowedToolsResolver:
        account exists? YES (set in previous step)
        account configured? YES
        → workflow tool allowed
    → GPT-4o-mini: intent = "workflow", confidence = 0.94
    → AiDecision: EXPLAIN
    → SessionStore: pendingAction = {tool: workflow.create, 
                                     params: {processCode: PROPERTY_TAX_ASSESSMENT}}
    → Response: "I will create workflow PROPERTY_TAX_ASSESSMENT with states
                 INITIATED → PENDING_VERIFICATION → APPROVED. Confirm? YES/NO"

Engineer: "yes"
    → POST /mcp/ai (X-Session-Id: eng-001)
    → YES/NO check: "yes" → EXECUTE pendingAction (zero AI calls)
    → MCP Server: POST /workflow/v3/definition (engineer's Bearer token)
    → DIGIT Workflow API: created
    → Audit log: {userId, intent, confidence, action, confirmation, result}
    → Response: "Workflow PROPERTY_TAX_ASSESSMENT created."
```

### Flow 2: Citizen Files Complaint by Voice

```
Citizen speaks (Tamil): "வீட்டிற்கு அருகில் கழிவுநீர் மூன்று நாளாக넘்பி வழிகிறது"
    → CCRS Web Speech API → transcribed text
    → POST /analyze {text: "..."}
    → Analytics FastAPI:
        urgency: HIGH
        signals: ["3 days" → time, "overflow" → critical service]
        service_codes: [StormWaterDrainRepair, DrainageBlocked, WaterStagnation]
    → CCRS: shows top 3 suggestions below dropdown
    → Citizen picks StormWaterDrainRepair
    → CCRS: POST to DIGIT PGR API (citizen's auth)
    → DIGIT PGR: stored in eg_pgr_service_v2
                  additionalDetails JSONB: {urgency: "high", urgency_signals: [...]}
    → Admin Intelligence Dashboard: red pin appears on Ward 12
```

### Flow 3: Proactive Alert Before Monsoon

```
Alert Engine (June 1st):
    → GET /recurrence (all wards)
    → Ward 12: recurrence_score = 1.0, recurring_months = [6, 7]
    → current month = 6, score > 0.7 → FLAGGED
    → Twilio WhatsApp → engineer:
       "Ward 12 (Tukum Area) — projected waterlogging risk.
        Historical: 46 complaints in July last year.
        Recurrence: 3 consecutive years.
        Recommended: initiate preventive drain cleaning."

Engineer opens /intelligence dashboard:
    → Engineer alert inbox: Ward 12 flagged, trend chart visible
    → Clicks "Accept Work Order"
    → DIGIT PGR work order API: work order created (engineer's auth)
    → Query eg_pgr_service_v2: mobileNumber of past complainants in Ward 12
    → Twilio WhatsApp → citizens:
       "Preventive drain maintenance starting in your area.
        Temporary disruptions possible 10–14 June."
```

### Flow 4: Knowledge Query (Developer / Implementation Team)

```
Developer: "what fields are required for workflow definition in DIGIT 3.0?"
    → RAG V5
    → Predetermined Q&A cache: miss
    → Query rewriter: "DIGIT 3.0 workflow definition API required fields"
    → Vector search + BM25 on studio_manual
    → Cohere reranking
    → OpenAI streams answer (with field names from digit-specs)
    → Developer: 👍
    → Promoted to predetermined_qa (confidence 0.7)
    → Next query: cache hit, instant
```

---

## Component Dependency Map

```
Admin Intelligence Dashboard
    → Analytics FastAPI (/recurrence, /analytics/complaints)
    → DIGIT PGR API (work order creation via CCRS)
    → digit-client [Java, CCRS backend]

CCRS citizen form
    → Analytics FastAPI (/analyze)
    → DIGIT PGR API (complaint submission)
    → digit-client [Java, CCRS backend]

Analytics FastAPI
    → DIGIT PGR database (direct SQL: eg_pgr_service_v2)
    → Twilio WhatsApp API
    → NOT via digit-client (Python service, not Java)

digit-ai-orchestrator
    → OpenAI GPT-4o-mini (intent classification)
    → MCP Server [once built]
    → DIGIT APIs [currently direct]

MCP Server [to build]
    → digit-specs (source for generation)
    → DIGIT APIs (forwarding caller's Bearer token)
    → Entity Resolution Service (enriching responses)

Entity Resolution Service [to build]
    → DIGIT localization service (source)
    → Redis (cache layer)

RAG V5
    → Neon PostgreSQL + pgvector
    → Cohere API (reranking)
    → OpenAI (answer generation + query rewriting)
    → digit-specs [to ingest]

digit-cli
    → DIGIT APIs (user's auth token)
    → Used independently of AI layer

digit-client (Java)
    → DIGIT APIs
    → Used by: CCRS backend, PGR backend, other Java services
    → Not used by: AI agents, Python services
```
