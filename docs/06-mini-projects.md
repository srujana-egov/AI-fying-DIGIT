# Mini Projects: Making the Architecture Real

Seven projects to complete the 4-layer architecture. Two are already running.

---

## [IN PROGRESS] Mini Project 1: EGOV RAG V5 — Layer 1 Knowledge
**Repo:** `srujana-egov/EGOV_RAG_V5`  
**Status:** Live. DIGIT Studio documentation chatbot.  
**Team:** Srujana Dadi  

### What Exists
- Streamlit UI with cache → RAG → streaming answer pipeline
- Predetermined Q&A cache with feedback-driven promotion
- Hybrid retrieval: vector search + BM25 + Cohere reranking
- Admin dashboard: query history, satisfaction rate, content management
- Currently trained on: DIGIT Studio documentation

### What It Needs Next
Expand the knowledge base beyond Studio docs:
1. Ingest all 16 digit-specs OpenAPI files → AI can answer API questions accurately
2. Ingest DIGIT process guides and configuration documentation
3. Ingest case studies from deployed cities (what worked, what patterns recur)
4. Add routing: "how do I" queries → RAG; "do this" queries → orchestrator

This becomes the knowledge backbone for all layers above it.

---

## [IN PROGRESS] Mini Project 2: Smart Grievance Mapping — Layer 3 + Layer 4
**Repo:** Internal (intern project)  
**Status:** Week 3 of 8. Due July 30, 2026.  
**Team:** 4 interns (ML-A, ML-B, Frontend, CCRS)

### What Is Being Built
See [Existing Experiments doc](03-existing-experiments.md) for full detail.

Analytics FastAPI (`/analyze`, `/recurrence`, `/severity`, alert engine) + Admin Intelligence Dashboard + Voice intake in CCRS + DIGIT PGR integration.

### Why It Matters Beyond PGR
This is the replicable pattern for Layer 3. The same structure — domain intelligence microservice + admin dashboard + proactive alert engine — applies to every DIGIT service domain:

| Domain | Next Batch |
|---|---|
| HCM | Attendance anomaly, posting gap detection, training completion alerts |
| Property Tax | Non-filer prediction, assessment anomaly detection, collection intelligence |
| Water & Sewerage | Usage spike detection, billing anomaly, leakage pattern intelligence |
| Finance / iFIX | Budget utilisation intelligence, expenditure pattern anomalies |

Each gets `/analyze`, `/recurrence`, `/severity`, alert engine, and an intelligence dashboard. PGR done = pattern proven.

---

## Mini Project 3: DIGIT MCP Server — Layer 2
**Status:** Proposed  
**Team:** 1-2 developers  
**Effort:** 3 weeks  
**Dependencies:** digit-specs (exists)

### What It Is
An MCP server auto-generated from the 16 digit-specs OpenAPI 3.0.3 files. Any AI agent — Claude, GPT, the orchestrator, a custom pipeline — can call DIGIT APIs via this server.

### What It Is Not
A hand-authored tool registry. The entire point is that DIGIT 3.0's specs are clean enough to generate from. Hand-authoring 61 tools (as Chakshu did for 2.9) was correct for 2.9. It is unnecessary on 3.0.

### Steps
1. Run OpenAPI Generator against each spec file → raw MCP tool definitions
2. Write a thin curation config: which endpoints to expose, description overrides for AI clarity
3. Add Bearer token forwarding: the calling user's token, not a service account
4. Wire Entity Resolution (Mini Project 4) to enrich responses: replace codes with human-readable values before returning to AI
5. Deploy as standalone service

### Output
```
Any AI agent calls:
list_tools()
  → [workflow_create, workflow_list_states, individual_create, 
     idgen_generate, mdms_search, boundary_get_hierarchy, ...]

workflow_create({processCode: "PROPERTY_TAX", states: [...]}, bearerToken)
  → {success: true, processCode: "PROPERTY_TAX", stateCount: 3}
```

### How It Relates to the Orchestrator
The orchestrator's `AllowedToolsResolver` currently holds a hand-coded list of 10 tools. Mini Project 3 replaces that list with the full DIGIT 3.0 API surface. The orchestrator's safety logic (intent inference, state gating, confirmation) stays exactly as is. It gets a richer tool surface underneath.

---

## Mini Project 4: Entity Resolution Service — Layer 2
**Status:** Proposed  
**Team:** 1 developer  
**Effort:** 2 weeks  
**Dependencies:** DIGIT localization service (exists), Redis

### What It Is
A resolution microservice that wraps DIGIT's existing localization service with a Redis cache. Makes internal codes human-readable for AI consumption.

### API
```
GET /resolve?code=LOC_CITYA_1         → "Ward 4, Chandrapur North"
GET /resolve?code=StormWaterDrain     → "Storm Water Drain Repair"
GET /resolve?code=pb.amritsar        → "Amritsar Municipal Corporation"
GET /resolve?tenantId=pb.chennai     → "Chennai Metropolitan Corporation"
GET /hierarchy?wardCode=WARD_04      → {ward: "Ward 4", zone: "Zone 2", 
                                         district: "Chandrapur", state: "Maharashtra"}
```

### Why It Can Be Redis-Cached
Locality codes, service codes, and tenant IDs change only when MDMS data is updated — infrequently. A 24-hour Redis TTL with invalidation on MDMS update events gives near-instant resolution at zero localization service load.

### Integration Point
The MCP server (Mini Project 3) calls Entity Resolution before returning any API response to the AI agent. The AI always receives human-readable values, never raw codes.

---

## Mini Project 5: Confirmation Middleware — Layer 2
**Status:** Proposed  
**Team:** 1 developer  
**Effort:** 2 weeks  
**Dependencies:** digit-ai-orchestrator (reference implementation)

### What It Is
The orchestrator's YES/NO confirmation pattern, extracted and generalized into a reusable middleware for all DIGIT write operations.

Currently the orchestrator implements confirmation for 10 setup intents only. As the MCP server expands the tool surface and the intent classifier extends to city administrator and state official operations, confirmation needs to work for all write operations uniformly.

### Changes From Orchestrator
1. Replace ConcurrentHashMap with Redis (persistent across restarts, horizontally scalable)
2. Generalize `pendingAction` schema to cover any DIGIT write operation, not just setup intents
3. Add TTL: pending action expires after 10 minutes if not confirmed
4. Add re-authentication trigger for high-risk operations (role assignment, demand generation)

### The Confirmation Contract
```
Any write operation:
    → AI proposes: {tool, params, plain_english_description}
    → Stored in Redis as pendingAction (TTL: 10 min)
    → Response to user: "I will {plain_english_description}. Confirm? YES/NO"

User says YES:
    → retrieve pendingAction from Redis
    → execute via MCP server (forwarding Bearer token)
    → write audit log
    → clear pendingAction

User says NO:
    → clear pendingAction
    → "Okay, what would you like to do instead?"
```

---

## Mini Project 6: Platform Intent Classifier — Layer 2
**Status:** Proposed (starts after Mini Project 3 defines full tool surface)  
**Team:** 1 ML developer  
**Effort:** 4 weeks  
**Dependencies:** Mini Project 3 (MCP server defines what operations exist)

### What It Is
Extension of the orchestrator's 10-intent classifier to cover all DIGIT operations across all stakeholder types. Returns structured operation intent rather than just a named intent.

### Output Schema
```json
{
  "service": "pgr",
  "operation": "list",
  "entities": {
    "ward": "Ward 12",
    "status": "open",
    "urgency": "high"
  },
  "operation_type": "read",
  "confidence": 0.94
}
```

### Examples
```
"show me all high urgency complaints in Zone 3"
  → {service: pgr, operation: list, entities: {zone: 3, urgency: high}, 
     operation_type: read, confidence: 0.96}

"which 10 cities are behind on property tax this quarter"
  → {service: billing, operation: aggregate, entities: {metric: collection_rate, 
     period: current_quarter, rank: bottom_10}, operation_type: read, 
     confidence: 0.91}

"assign drainage engineer role to Ramesh Kumar"
  → {service: user, operation: role.assign, 
     entities: {role: DRAINAGE_ENGINEER, user: "Ramesh Kumar"}, 
     operation_type: write, confidence: 0.88}
```

### Routing
```
operation_type = read  → execute directly via MCP server (no confirmation)
operation_type = write → send to Confirmation Middleware (Mini Project 5)
query_type = knowledge → route to RAG V5 (Mini Project 1)
```

---

## Mini Project 7: Intelligence Microservices — Layer 3 (per domain)
**Status:** Proposed (PGR done by July 30 via intern project)  
**Team:** Interns per domain  
**Effort:** 4-6 weeks per domain

### The Pattern (proven by intern project)
```
Domain intelligence microservice:
    POST /analyze      → real-time classification + scoring
    GET  /recurrence   → pattern detection over historical data
    POST /severity     → composite risk score
    Alert Engine       → proactive notifications (Twilio/DIGIT notification service)

Admin intelligence dashboard:
    Map or tabular view with intelligence overlay
    Accept action button (triggers write via DIGIT API with confirmation)
    Citizen communication log
```

### Domains to Implement

**HCM Intelligence**
- Attendance anomaly detection: field worker attendance vs deployment expectation
- Posting gap detection: positions unfilled beyond SLA
- Training completion alerts: mandated trainings overdue
- Alert: posting supervisor when attendance drops below threshold

**Property Tax Intelligence**
- Non-filer prediction: properties at risk of non-payment based on history
- Assessment anomaly: properties with suspiciously low assessed value vs area comps
- Collection velocity: which zones are tracking behind target
- Alert: revenue officer before quarter-end with actionable non-filer list

**Water & Sewerage Intelligence**
- Usage spike detection: anomalous consumption (leakage signal)
- Billing anomaly: connection billed at wrong rate
- Non-payment prediction before disconnection order
- Alert: zone engineer on anomalous consumption in a ward

**Finance / iFIX Intelligence**
- Budget utilisation tracking: departments at risk of returning allocation
- Expenditure pattern anomalies: unusual spending velocity
- Cross-department comparison: which departments consistently underspend
- Alert: finance officer at 60% of fiscal year if utilisation < 40%
