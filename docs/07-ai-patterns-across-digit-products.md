# AI Patterns Across All DIGIT Products

Source: egov.org.in Products & Solutions (20 products across 7 domains).

The point of this document: the AI platform architecture being proposed is not specific to PGR or HCM. The same 5 patterns — and the same underlying platform layer — apply across every DIGIT product. Building the platform once (MCP server, confirmation gate, flagging microservice pattern, Temporal) means every domain gets AI capability without rebuilding the infrastructure.

---

## The 5 Generalizable Patterns

### Pattern 1 — AI Flagging: Declared vs Expected

AI compares submitted data in DIGIT records against expected values (domain rules, models, or external signals). Flags inconsistencies. Writes flags back via DIGIT API. No dashboard — consumers query flagged records using their existing tools.

```
Input:  DIGIT record (any form submission)
Signal: domain rules OR external data (GIS, satellite, field surveys)
Output: flag written to DIGIT record with reason + severity
```

| Domain | Submitted data | Expected signal | Revenue / compliance impact |
|---|---|---|---|
| PGR | Selected service code | AI text classification | Correct routing, faster resolution |
| HCM | Age + marital status | Demographic rules | Data quality, correct beneficiary records |
| HCM | Age + pregnancy | Age rules (under 16) | Flag for field verification |
| Trade License | Declared use (residential) | GIS land-use classification | Revenue gap recovery |
| Property Tax | Declared floor area | Satellite measurement | Under-declaration recovery |
| Water Connection | Declared usage category | Actual consumption pattern | Mis-tariffing correction |
| Works Management | Declared project progress | Satellite imagery | Contract compliance |
| BPA | Declared building use | GIS land-use | Permit compliance |
| Social Benefits | Declared identity | Deduplication check | Leakage prevention |
| Birth/Death | Age at death vs date of birth | Date arithmetic | Fraud detection |

**What builds this:** one flagging microservice per domain, sharing the same pattern. The MCP server exposes the DIGIT APIs needed to read records and write flags. No new platform work needed.

---

### Pattern 2 — GIS Cross-Reference: Ground Truth vs Registry

Specifically uses GIS/satellite imagery as the external truth signal. Highest impact on government revenue. Same pattern as HCM microplanning (satellite rooftop count vs population register = invisible population).

| Product | Registry claim | GIS ground truth | Impact |
|---|---|---|---|
| Trade License | Declared use: residential | GIS: commercial activity | Revenue recovery per mis-classified business |
| Property Tax | Declared area: 500 sq ft | Satellite: 800 sq ft | Under-declaration at scale |
| BPA (Construction Permit) | Approved building use | Satellite: actual structure | Compliance, penalty |
| Works Management | Reported road completion | Satellite: incomplete | Fraud detection in contracts |
| HCM Microplanning | Population register count | Satellite rooftop count | Invisible population for vaccination/benefits |
| Water Schemes | Declared connections | Satellite: actual settlements | Coverage gap in rural water |

**What builds this:** GIS intelligence microservice. Input: DIGIT registry records + satellite/population data. Output: discrepancy flags written to DIGIT via API. Runs on a schedule. Not a dashboard — flags live in DIGIT, visualization is the consumer's concern.

**Data sources in use:**
- WorldPop — high-resolution raster population estimates, peer-reviewed, REST API, LMIC coverage
- Google Open Buildings + Population Dynamics — building footprints from satellite imagery, location signatures per community
- Standard GIS land-use classification layers (for trade license / property tax)

**Already being built:** PES interns are building the HCM version (population denominator for campaigns in Chad, Sierra Leone, Nigeria, Liberia). Same pattern applies to Property Tax, Trade License, Works Management.

---

### Pattern 3 — Proactive Alerting: Act Before the Event

The recurrence detector from the PGR intern project generalizes. Any time DIGIT holds enough historical data to predict a future state, the alert engine fires before the event rather than after.

| Product | Historical signal | Prediction | Alert to |
|---|---|---|---|
| PGR | Ward 12: waterlogging complaints every July | Monsoon risk, June | Drainage engineer |
| Property Tax | Same non-filer list, 3 consecutive years | Will not pay without contact | Revenue officer |
| Water | Consumption spikes in April–May | Demand surge before summer | O&M team |
| HCM | Campaign X: consistently misses ward Y | Coverage gap prediction | Campaign supervisor |
| Works Management | Projects starting in June fail in monsoon | Weather/seasonal risk | Project manager |
| Water Schemes | Pump failure pattern: 3 months before breakdown | Predictive maintenance | O&M engineer |
| mCollect | Collection rate drops in certain zones after 15th | Optimal visit schedule | Field collector |

**What builds this:** alert engine (shared cron service). Each domain registers its recurrence query and threshold. When threshold crossed, WhatsApp/notification via DIGIT notification service (already exists).

---

### Pattern 4 — Beneficiary / Record Deduplication

Any DIGIT product that registers individuals or entities at scale has a deduplication problem. The same person, property, or business may exist under multiple IDs across systems or across time.

| Product | Deduplication problem |
|---|---|
| Social Benefits | Same person, multiple registrations → duplicate benefit payments |
| HCM | Same beneficiary across multiple health campaigns → over-counting coverage |
| DIVOC | Duplicate credential requests for the same individual |
| Birth/Death | Duplicate death records → person "dies twice" in the system |
| Property Tax | Same property split into multiple records to stay below assessment threshold |
| Trade License | Same business registered under multiple names at same address |

**What builds this:** deduplication at point of submission — not post-hoc. AI compares incoming record against existing records (name + age + location, fuzzy matching). Flags probable duplicates before commitment. Worker confirms or proceeds.

**Two deployment modes:**
- **On-device** (critical for offline field conditions): fuzzy search runs locally in the app before sync. Flutter library approach — publish once, every DIGIT/HCM app consumes it without rebuild. Being built now by PES interns for HCM campaigns.
- **Server-side** (for higher-confidence identity matching at scale): MOSIP integration for biometric-backed deduplication. Connects to national identity infrastructure.

**Already being built:** PES interns are building the HCM version using synthetic campaign data. Flutter library will be reusable across all HCM campaigns without rebuild.

---

### Pattern 5 — Process Intelligence: Prediction and Bottleneck Detection

DIGIT workflow service records every state transition with timestamps. Enough history exists to predict: how long will this take, where is it stuck, what usually causes delays.

| Product | Process intelligence |
|---|---|
| BPA (Construction Permit) | Predict approval time based on application type, ward, season |
| PGR | Predict resolution time based on complaint type, assigned team, historical SLA |
| Trade License | Flag applications stuck at the same stage for > N days |
| Fire NOC | Identify inspectors with systematic delays |
| Works Management | Predict cost overrun based on early-stage indicators |
| Water Connection | Predict which connections will face fieldwork delays |

**What builds this:** workflow analytics layer. Reads from DIGIT workflow service history (already recorded). No new data collection needed — the data exists.

---

## Product-by-Product AI Use Case Map

| # | Product | Pattern 1 (Flagging) | Pattern 2 (GIS) | Pattern 3 (Alert) | Pattern 4 (Dedup) | Pattern 5 (Process) |
|---|---|---|---|---|---|---|
| 1 | CCRS / PGR | ✓ text vs code mismatch | — | ✓ monsoon recurrence | — | ✓ SLA prediction |
| 2 | Construction Permit (BPA) | ✓ declared use vs actual | ✓ satellite vs approved plan | — | — | ✓ approval time prediction |
| 3 | Water & Sewerage Connections | ✓ declared category vs usage | ✓ satellite: unconnected settlements | ✓ demand surge alert | — | ✓ fieldwork delay prediction |
| 4 | Trade License | ✓ declared use vs GIS | ✓ land-use classification | ✓ renewal expiry alert | ✓ duplicate business | ✓ approval stage flag |
| 5 | Birth & Death Certificate | ✓ date arithmetic checks | — | — | ✓ duplicate records | — |
| 6 | Fire NOC | ✓ compliance vs building state | ✓ building modification detection | ✓ overdue reinspection | — | ✓ inspector delay flag |
| 7 | Works Management | ✓ declared progress vs actual | ✓ satellite: incomplete works | ✓ monsoon/weather risk | — | ✓ cost overrun prediction |
| 8 | HCM | ✓ age/status/pregnancy rules | ✓ WorldPop + Google Open Buildings vs enrollment [**in progress — PES interns**] | ✓ campaign coverage gap | ✓ on-device fuzzy dedup before submission [**in progress — PES interns**] | — |
| 9 | 10 Bed ICU | — | — | ✓ bed occupancy surge alert | ✓ patient duplicate | — |
| 10 | DIVOC | — | — | — | ✓ credential dedup | — |
| 11 | Social Benefits | ✓ eligibility inconsistency | — | ✓ disbursal anomaly | ✓ cross-SHG dedup | — |
| 12 | iFix | ✓ expenditure pattern anomaly | — | ✓ budget burn alert | — | — |
| 13 | Waste Management | ✓ collection claim vs complaint | — | ✓ missed collection alert | — | ✓ route compliance |
| 14 | Water Supply O&M (rural) | ✓ billing vs connection | ✓ coverage gap | ✓ infrastructure failure | — | — |
| 15 | Water Schemes O&M | ✓ consumption anomaly | ✓ unserved settlement | ✓ predictive maintenance | — | — |
| 16 | Property Tax | ✓ declared area vs satellite | ✓ area measurement | ✓ non-filer alert | ✓ property split detection | ✓ assessment anomaly |
| 17 | mCollect | ✓ collection pattern anomaly | — | ✓ optimal visit timing | — | — |
| 18 | DRISTI | — | — | ✓ case overdue alert | ✓ duplicate case | ✓ case outcome prediction |

---

## What This Means for the Platform AI Layer

**Every product benefits from the same platform layer.** The MCP server auto-generated from DIGIT specs covers all products. The confirmation gate applies to every write operation across every product. The cross-module workflows connect products that share data (Property Tax + Water + Trade License at the same address).

**The intelligence layer is one pattern, replicated per domain:**

```
AI Flagging Microservice (one per domain)
    reads DIGIT records via MCP (platform layer)
    applies domain rules / GIS comparison / model inference
    writes flags back via DIGIT API (platform layer)
    triggers alert engine if threshold crossed

Alert Engine (shared)
    notification via DIGIT notification service (already exists)
    WhatsApp / SMS / dashboard annotation

No custom dashboard built — flags live in DIGIT
Consumers use PowerBI, DSS, existing tools
```

**What the intern project (PGR) proves:** the pattern, not just the feature. PGR done = pattern available to all 18 other products.

---

## What Is NOT the AI Layer's Job

- Building visualization dashboards — PowerBI, DSS, and Metabase do this. eGov's AI layer flags records in DIGIT. Consumers visualize.
- Replacing the workflow service — it already records process state. AI reads from it, does not replicate it.
- AI-assisting one-time city configuration — not at-scale, tenant isolation prevents cross-city template reuse.
