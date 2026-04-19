# PIPE2 — Property Investigator / Performance Enhancer
### The Master Reference Document

**Version:** 1.4.0
**Last Updated:** 10 April 2026
**Author:** Basil Demeroutis, FORE Partnership  
**Status:** Production  

---

## ⚡ Two Rules That Govern Everything

Before reading anything else, internalise these two principles of the Five Principles in the global Claude.MD file. Every architectural decision in PIPE2 flows from them.

**Principle 1 — GPTs lie.**  
GPT will confidently return hallucinated field names, fabricated API responses, invented values, and plausible-sounding nonsense. It doesn't know it's lying. Design for this. Middleware validates every payload against the Field Guide. Nothing GPT returns is stored without passing through `pipe-store`. Nothing GPT says about session state is trusted — `pipe-session` is the authority.

**Principle 2 — GPTs don't follow instructions.**  
System prompts are suggestions. GPT will ignore rules mid-conversation, revert to default behaviour, skip steps, misread schemas, and pattern-match against training data instead of the actual payload in front of it. Build the system so that correct behaviour is the path of least resistance — not a result of GPT remembering to comply. Server-side phase tracking, forced `next_actions` arrays, server-served questions — all of this exists because you cannot instruct your way out of GPT's unreliability.

These two rules are not guidelines. They are load-bearing walls.

---

## Skills to Use in This Codebase

Before writing or modifying any Val, check whether a skill applies. Skills encode hard-won trial-and-error — reading them takes 60 seconds and prevents hours of rework.

| Task | Skill |
|------|-------|
| Any Val creation or modification | `valtown-conventions` |
| Building or reviewing a new skill | `skill-engineering-protocol` |
| Superpowers framework (planning, execution, review) | `using-superpowers` / `superpowers-main` |
| FORE/MORE branded documents, presentations, spreadsheets | `fore-brand-guidelines` |
| Auditing or reviewing Excel financial models | `excel-auditor` |
| Investor memos, board papers, recommendation reports | `governance-drafting` |
| Behavioural science and psychology applied to marketing | `marketing-psychology` |
| Analysing meeting transcripts for communication insights | `meeting-insights-analyzer` |

---

## What is PIPE?

PIPE (Property Investigator / Performance Enhancer) is an automated property intelligence system for commercial real estate analysis. Give it an address — it assembles a complete investment-grade dossier from UK government datasources, validates every data point with full provenance, and generates a professional report. What previously took an analyst days of fragmented research takes PIPE minutes.

It exists because commercial property data is siloed, inconsistent, and expensive to compile. PIPE makes it deterministic.

**The output:** A structured PDF report covering physical characteristics, energy performance, sustainability certification, deprivation context, ownership, corporate structure, planning history, heritage status, transport connectivity, and CRREM carbon risk — delivered by email, generated from a single conversation.

**The interface:** A custom GPT called PIPE2, accessed via ChatGPT. The user talks to GPT. GPT orchestrates. Everything intelligent happens in middleware.

**Who uses it:** FORE Partnership deal team. Built and maintained by Basil Demeroutis.

PIPE is a node within FORE's Federated System — a distributed cognitive architecture with shared governance across all AI nodes. For system-level context (UCL, CDP, Context Binders, Voice Profile, the instruction extension pattern), see `FORE_Federated_System_README.md`.

---

## The Inversion Principle

PIPE 1.0 was monolithic: GPT did everything — prompting, validation, storage, generation. It was fragile. GPT forgot rules mid-conversation, hallucinated field names, lost context on long sessions.

PIPE 2.0 inverts the architecture:

```
PIPE 1.0:  GPT ─────────────────────────────────────────→ Output
           (does everything, reliably does nothing)

PIPE 2.0:  GPT ──→ Val.town ──→ SQLite ──→ Val.town ──→ Output
           (orchestrates)  (validates)  (stores)  (generates)
```

GPT is now a thin orchestrator. It sequences API calls, interprets results, and communicates with the user. It does not validate, store, or generate. All logic lives in Val.town middleware.

**Why this works:** PRISM (a predecessor system) proved that code-based validation with fuzzy matching catches 95%+ of GPT's mistakes. Prompting GPT into compliance fails. Catching its failures in middleware works. PIPE 2.0's store endpoint architecture is built on this foundation.

---

## Full Design Principles

1. **GPTs lie** — See Principle 1 above.
2. **GPTs don't follow instructions** — See Principle 2 above.
3. **Provenance matters** — Every stored field tagged with its source (`EPC`, `PLD`, `NHLE`, `HMLR`, etc.). Enables cross-validation, audit, and conflict detection. When sources conflict, middleware flags it — GPT never arbitrates.
4. **Resilience over elegance** — 15s timeouts, retry with exponential backoff, graceful fallbacks on every external dependency. From the first line of code, not as a hardening pass.
5. **Expansive logging** — Allows easy debugging. Include the detailed payload that gets passed back to the PIPE GPT via the OpenAPI to see what the GPT sees.
6. **Session continuity** — Users resume work across conversations, devices, days. Database persists 30 days. A session is a property, not a conversation.
7. **Middleware responses are instructions** — Vals tell GPT what to do next: `next_actions`, `post_store_actions`, `transition_message`. The response *is* the instruction set. GPT reads these and complies — it doesn't decide what to do next.

---

## A Session, Start to Finish

A user opens the PIPE2 GPT and types an address. Here's what actually happens across the 8 named phases:

### Phase 1 — INIT
`pipe-start /init` creates a session record in SQLite. Returns a `session_id` (format: `PIPE-{timestamp}-{random6}`). This ID threads through every subsequent call.

`pipe-start /address` queries Ideal Postcodes (AddressBase Core) with smart filtering. Returns a ranked candidate list. User selects. Seven address fields store automatically. LSOA code triggers IMD lookup. Duplicate UPRN detection runs automatically — if the same property was analysed in the last 30 days, user is offered Resume or Fresh.

### Phase 2 — API DATA
GPT calls each data Val in sequence. Each Val stores its results directly via `pipe-store`, tagged with the correct source enum.

| Val | What It Returns | Source Code |
|-----|----------------|-------------|
| `pipe-epc-imd` | EPC certificate (~25 fields: ratings, floor area, emissions, construction) + DEC (operational rating for public buildings; null DEC returns deliberate fallback) + IMD deprivation indices (~13 fields) + ACIR (Air Conditioning Inspection Report — 219 TM44 dt/dd pairs stored to pipe_acir table). Delegates EPC/DEC/ACIR API calls to `pipe-epc-logic` (4-tier cert search: UPRN → postcode → cross-postcode → address-text). Returns `cert_summary` aggregate across all discovered certificates. | `EPC`, `DEC`, `IMD`, `ACIR` |
| `pipe-property /planning` | Planning history from PLD (London) + PlanIt.org.uk (UK-wide, 417 authorities) — merged, deduplicated, sorted | `PLD`, `PLANIT` |
| `pipe-property /heritage` | Historic England NHLE listing status + conservation area (with LPA fallback). Stores "Not listed" explicitly — absence of evidence is evidence. | `NHLE`, `HE_CA` |
| `pipe-property /hmlr` | Land Registry corporate ownership, freehold prioritised, CCOD/OCOD via Turso | `HMLR`, `CCOD`, `OCOD` |
| `pipe-companies` | Companies House profile, officers, charges, PSC, insolvency — conditional chain. **Auto-fired from inside pipe-property's HMLR handler** when corporate ownership detected (company_reg populated). Not a separate GPT action — GPT never calls this directly. CH response is nested inside the HMLR response. | `COMPANIES_HOUSE` |
| `pipe-transport` | TfL nearest stations, lines served, walk times. London only — non-London returns coverage note. | `TFL` |
| `pipe-breeam` | BREEAM sustainability certification lookup via BRE API. Returns rating, scheme, certificate number, score. Proximity matching for buildings without exact address match. | `BREEAM` |

### Phase 3 — DROPZONE
User uploads question answers (CSV/XLSX) and property images via the Drop Zone HTML form. This completely bypasses GPT for data transport — the single biggest reliability win in the system. GPT sends the user a secure, time-limited URL. User opens it in browser, drags and drops files. Val processes directly to SQLite. No GPT hallucination possible on this path.

`pipe-dropzone-control /init` generates the token. `pipe-dropzone` serves the HTML form and processes uploads. Images are pre-processed: cropped square, resized to 472×472px at 200 DPI.

### Phase 4 — BASE BUILD FIELDS
GPT synthesises fixed fields describing the building as it exists today. These are **not API returns** — they are GPT-generated analytical fields, drawing on EPC data, uploaded documents, and the Context Binders (240+ specialist documents). Key fields:

| Field | Description | Source |
|-------|-------------|--------|
| `epc_commentary` | Analytical commentary on energy performance | `GPT` |
| `ownership_insights` | Ownership and transaction insight narrative | `GPT` |
| `planning_history` | Planning history narrative with heritage implications | `GPT` |
| `long_building_description` | 75–200 word physical description | `GPT` |

GPT proposes each field and asks the user to approve, edit, or reject. The confirmed answer is stored with source `GPT`. These are the fields most likely to need human correction — treat GPT proposals as first drafts, not facts.

### Phase 5 — BASE BUILD QUESTIONS
`pipe-session /questions` serves the question set from the database (never from GPT memory — this was a critical bug in earlier versions). 13 Base Build questions covering MEP systems, vertical transport, amenities. User answers store as `user_answer_XX` fields with word count and type validation enforced by `pipe-store`.

`pipe-session /status` returns `current_phase`, `completion_percentage`, and `next_actions` at all times. GPT knows exactly what's missing. No guessing.

### Phase 6 — STRATEGY QUESTIONS
20 Strategy questions from the same question set — investment thesis, performance targets, retrofit interventions per system, limitations. Same approve/edit/reject flow. Strategy questions unlock only after Base Build is confirmed complete.

### Phase 7 — STRATEGY FIELDS + CRREM RETROFIT
`pipe-crrem /crrem1` — base case (do-nothing scenario). The router delegates all calculations to `pipe-crrem-logic /crrem1-bundle`: use class mapping, archetype matching, EUI estimation, building CO2 projection, pathway fetch, stranding analysis (1.5°C + 2.0°C). Returns stranding year, strand severity (COMPLIANT/MARGINAL/MODERATE/SEVERE), excess emissions, and CRREM VAR (UK Treasury £241/tCO₂). Supports optional `energy_mix_override` to replace TM46 defaults with measured electricity/gas split.

`pipe-crrem /crrem2` — retrofit scenario. Calls `pipe-crrem-logic /crrem2-prep` then `pipe-energy` (never directly by GPT), then `pipe-crrem-logic /crrem2-retrofit`. Physics-based energy model using 24 CIBSE/SBEM/NCM/TM46 lookup tables. Returns 27-year staircase (2024–2050) with intervention impact per year. User and GPT collaborate on intervention packages; user says "lock in that retrofit strategy" to commit to database. 16 valid intervention types including ASHP, LED, secondary glazing, PV, BMS, DCV, DHW electrification, and night purging.

GPT then synthesises remaining Strategy fields: certification readiness (BREEAM, NABERS, WELL), cost estimates, IMD investment analysis, decarbonisation narrative. Same source: `GPT`.

Both CRREM routes fetch pathway reference data from GitHub. Chart generated by PythonAnywhere Flask `/crrem-chart` via matplotlib and inserted into the report automatically.

### Phase 8 — EXPORT
When completion threshold met, `pipe-report /generateReport` bundles all fields + images into a JSON payload, POSTs to PythonAnywhere Flask `/generate`. Flask renders the Word template (`{{ field_name }}` placeholders via docxtpl), converts to PDF via LibreOffice, uploads to Dropbox, triggers Zapier webhook. User receives email with DOCX + PDF attachments.

**Total fields in a complete report:** 143 (107 required). Field Guide on GitHub is the canonical definition of both.

---

## Context Binders — The Knowledge Layer

GPT-synthesised fields in Phases 4 and 7 (building descriptions, certification commentary, decarbonisation narratives, investment thesis framing) draw on 240+ specialist documents loaded as GPT Knowledge Files across 15 thematic Context Binders. These are not publicly available and not part of standard LLM training. They include the full BREEAM V7 standard, LETI Climate Emergency Retrofit Guide, CIBSE TM54/Guide F, RICS Whole Life Carbon guidance, UKGBC frameworks, TCFD/SFDR/CSRD reporting standards, FORE's own investment letters and case studies, and research spanning behavioural science, spatial justice, and urban design.

When GPT proposes an `epc_commentary` or `planning_history` field, it's reasoning against this corpus — not guessing from training data alone. When it proposes a retrofit intervention sequence, it's drawing on CIBSE benchmarks, CRREM methodology, and FORE's own strategic lens simultaneously.

Context Binders are managed in the ChatGPT Custom GPT Knowledge File settings, not in Val.town. They are not fetchable by Vals and are invisible to middleware. For the full binder inventory, supporting infrastructure (kf_tracker, binder structure, Field Manual), and the architectural rationale, see `FORE_Federated_System_README.md` §11.

---

## System Architecture

```
                          ┌──────────────────────────────────────┐
                          │           GitHub Repository          │
                          │  field_guide.csv                     │
                          │  crrem_uk_pathways_1_5C.csv          │
                          │  crrem_uk_pathways_2_0C.csv          │
                          │  crrem_uk_grid_emission_factors.csv  │
                          │  crrem_uk_energy_mix_defaults.csv    │
                          │  crrem_fuel_emission_factors.csv     │
                          │  energy/energy_bundle.json (24 CSVs) │
                          │  test_payload.json                   │
                          └────────────────┬─────────────────────┘
                                           │ fetched at runtime (cache-busted)
                                           ▼
┌─────────┐    ┌──────────────────────────────────────────────────────────┐
│   GPT   │───▶│                       Val.town                          │
│ (PIPE2) │    │                                                          │
└─────────┘    │  ┌────────────┐  ┌───────────────┐  ┌────────────────┐  │
     │         │  │ pipe-start │  │ pipe-epc-imd  │  │ pipe-property  │  │
     │         │  │/init /addr │  │EPC+DEC+IMD    │  │/planning /hmlr │  │
     │         │  └─────┬──────┘  └───────┬───────┘  └───────┬────────┘  │
     │         │        │                 │                   │           │
     │         │        │         ┌───────┴───────────────────┘           │
     │         │        │         │  ┌──────────────┐                     │
     │         │        │         │  │pipe-companies│ (conditional)       │
     │         │        │         │  │pipe-transport│                     │
     │         │        │         │  │pipe-breeam   │ (blocked)           │
     │         │        │         │  └──────┬───────┘                     │
     │         │        └─────────┴─────────┴─────────────┐               │
     │         │                                           ▼               │
     │         │                                   ┌────────────┐         │
     │         │                                   │ pipe-store │         │
     │         │                                   │ (gateway)  │         │
     │         │                                   └─────┬──────┘         │
     │         │                                         ▼                │
     │         │                                   ┌────────────┐         │
     │         │                                   │   SQLite   │         │
     │         │                                   │ (Val.town) │         │
     │         │                                   └────────────┘         │
     │         │                                         ▲                │
     │         │  ┌──────────────┐  ┌──────────┐  ┌─────┴──────────────┐ │
     │         │  │ pipe-session │  │pipe-crrem│  │pipe-dropzone-ctrl  │ │
     │         │  │/pull /status │  │/crrem1   │  │/init /status       │ │
     │         │  │/questions    │  │/crrem2   │  └────────────────────┘ │
     │         │  │/progress/next│  └────┬─────┘                         │
     │         │  └──────────────┘       │                               │
     │         │                    pipe-energy                           │
     │         │                    (internal only)                       │
     │         │                                                          │
     │         │  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
     │         │  │pipe-dropzone │  │ pipe-report  │  │ fore-services │  │
     │         │  │  (HTML UI)   │  │              │  │INIT+knowledge │  │
     │         │  └──────────────┘  └──────┬───────┘  └───────────────┘  │
     │         └──────────────────────────┼───────────────────────────────┘
     │                                    │
     │          ┌─────────────────────────┼──────────────────────────────┐
     └─────────▶│         External Services                              │
                │  ┌──────────────┐  ┌───────┐  ┌──────────────────────┐ │
                │  │PythonAnywhere│  │Dropbox│  │       Zapier         │ │
                │  │  Flask v1.3  │─▶│       │─▶│   Email Delivery     │ │
                │  │/generate     │  └───────┘  └──────────────────────┘ │
                │  │/crrem-chart  │                                       │
                │  │/health       │                                       │
                │  └──────────────┘                                       │
                └─────────────────────────────────────────────────────────┘
```

---

## Val Registry

Current production Vals. **Always verify against deployed Val health check before editing.**

| Val | Chars | URL | GPT-facing? | Purpose |
|-----|-------|-----|-------------|---------|
| pipe-start | ~76,920 | pipe-start.val.run | ✓ `init`, `address` | Session init + address lookup (Ideal Postcodes / AddressBase Core). Decomposed v3.0.0. |
| pipe-epc-imd | ~78,204 | pipe-epc-imd.val.run | ✓ `buildingData` | **Router:** EPC+DEC+IMD+ACIR orchestration + storage. Delegates EPC/DEC/ACIR API calls to pipe-epc-logic. |
| pipe-epc-logic | ~67,799 | pipe-epc-logic.val.run | — logic Val | Stateless EPC+DEC+ACIR API logic: 4-tier cert search, field extraction, cert summary, ACIR search/fetch, /acir/data grouped query. Called by pipe-epc-imd. |
| pipe-property | ~69,000 | pipe-property.val.run | ✓ `planning`, `heritage`, `hmlr`, `transport`, `breeam` | **Router:** Planning/Heritage/HMLR/Transport/BREEAM orchestration + storage. Delegates heritage+HMLR to pipe-property-logic. |
| pipe-property-logic | 35,104 | pipe-property-logic.val.run | — logic Val | Stateless heritage (NHLE+CA) + HMLR (Turso) + address matching logic. Called by pipe-property. |
| pipe-planning | 40,048 | pipe-planning.val.run | — internal | Standalone planning engine called by pipe-property (PLD + PlanIt) |
| pipe-companies | 22,120 | pipe-companieshouse.val.run | — auto-fired | Companies House — auto-fired from pipe-property's HMLR handler when corporate owner detected. Not a GPT action. |
| pipe-transport | 24,869 | pipe-transport.val.run | — downstream | TfL transport accessibility. London only. Called by pipe-property `/transport` proxy. |
| pipe-breeam | ~23,100 | pipe-breeam.val.run | — downstream | BREEAM certification lookup via BRE API. Called by pipe-property `/breeam` proxy. |
| pipe-costs | ~59,750 | pipe-costs.val.run | ✓ `costsPreview`, `costsConfirm`, `costsEvidence` | Cost benchmarking. Exclusion markers for scoped retrofits. |
| pipe-crrem | 47,159 | pipe-crrem.val.run | ✓ `crrem1`, `crrem2` | **Router:** CRREM base case + retrofit orchestration + storage. Delegates calculations to pipe-crrem-logic, energy modelling to pipe-energy. |
| pipe-crrem-logic | 79,334 | pipe-crrem-logic.val.run | — logic Val | Stateless CRREM calculations: pathway analysis, stranding, archetype matching, EUI estimation, intervention validation. Called by pipe-crrem. **666 chars headroom.** |
| pipe-energy | 77,774 | pipe-energy.val.run | — internal | Physics-based energy model. 24 CSV lookup tables. Called only by pipe-crrem. **2,226 chars headroom.** |
| pipe-store | ~75,434 | pipe-store.val.run | ✓ `store` | Field validation + storage gateway + enforcement (first-person, word count, provenance, repeated failure). The only write path to SQLite. |
| pipe-session | ~75,677 | pipe-session.val.run | ✓ `pull`, `status`, `questions`, `progress`, `next` | Session state, phase tracking, question serving. /pull includes ACIR data. |
| pipe-dropzone | 74,479 | pipe-dropzone.val.run | — user-facing | HTML upload UI. Bypasses GPT entirely. |
| pipe-dropzone-control | 25,019 | pipe-dropzone-control.val.run | ✓ `dropInit`, `dropCheck` | Upload token management + ingest status |
| pipe-dropzone-status | 12,177 | pipe-dropzone-status.val.run | — internal | Drop Zone upload progress |
| pipe-report | ~78,175 | pipe-report.val.run | ✓ `generateReport` | Report generation orchestration |
| pipe-field-guide | ~17,720 | esm.town/v/basild/PIPE_field_guide | — import | Field definitions + validation rules |
| pipe-session-data | ~11,410 | esm.town/v/basild/PIPE_session_data | — import | Static data (instructions, constants, phase configs) extracted from pipe-session v2.0.0 |
| pipe-admin | 65,918 | pipe-admin.val.run | — standalone UI | Admin dashboard. No auth. Internal use only. |
| pipe-imd-refresh | 19,335 | pipe-imd-refresh.val.run | — batch | IMD CSV batch import |
| pipe-imd-refresh-cron | — | (cron) | — cron | Scheduled IMD refresh |
| pipe-userQues | — | pipe-userQues.val.run | ✓ `userQues` | User question loader |
| fore-services | — | fore-services.val.run | ✓ `getTime`, `trigger`, `getLoader`, `searchDocs` | INIT support + knowledge search |

### Router + Logic Decomposition (March 2026)

Three large Vals were split into router+logic pairs to manage character budget pressure:

| Router (session, DB, GPT response) | Logic Val (stateless API/calculation) | Split date |
|-------------------------------------|---------------------------------------|------------|
| pipe-epc-imd (v3.0.0+) | pipe-epc-logic (v1.0.0+) | 13 Mar 2026 |
| pipe-property (v3.0.0+) | pipe-property-logic (v1.0.0+) | 13 Mar 2026 |
| pipe-crrem (v3.0.0+) | pipe-crrem-logic (v1.0.0+) | 13 Mar 2026 |

**Pattern:** Router handles session context, pipe-store writes, GPT instruction assembly, and response envelope. Logic Val handles all external API calls and pure computation. Logic Vals are stateless — they receive data via POST body and return results. Router calls logic Val via `fetch()`. Response shapes to GPT are identical to pre-split versions.

**Character budget status (critical Vals):**

| Val | Chars | Headroom | Status |
|-----|-------|----------|--------|
| pipe-epc-imd | ~78,204 | ~1,796 | Near-frozen |
| pipe-report | ~78,175 | ~1,825 | Near-frozen |
| pipe-crrem-logic | 79,334 | 666 | Frozen |
| pipe-energy | 77,774 | 2,226 | Near-frozen |
| pipe-session | ~75,677 | ~4,323 | Active (refactored v2.0.0) |
| pipe-store | ~75,434 | ~4,566 | Active |

---

## GPT Action Slots

GPT has exactly 10 action slots. Hard platform limit. Current allocation (verify in ChatGPT custom GPT settings):

| Slot | Operations | Val |
|------|-----------|-----|
| 1 | `getTime`, `trigger`, `getLoader`, `searchDocs` | fore-services.val.run |
| 2 | `store` | pipe-store.val.run |
| 3 | `dropInit`, `dropCheck` | pipe-dropzone-control.val.run |
| 4 | `pull`, `status`, `questions`, `progress`, `next` | pipe-session.val.run |
| 5 | `init`, `address` | pipe-start.val.run |
| 6 | `generateReport` | pipe-report.val.run |
| 7 | `planning`, `heritage`, `hmlr`, `transport`, `breeam` | pipe-property.val.run (proxies to downstream Vals) |
| 8 | `crrem1`, `crrem2` | pipe-crrem.val.run |
| 9 | `buildingData` | pipe-epc-imd.val.run |
| 10 | `costsPreview`, `costsConfirm`, `costsEvidence` | pipe-costs.val.run |

**Not GPT actions** (internal/auto-fired): pipe-companieshouse (auto-fired from HMLR handler), pipe-transport (proxied via pipe-property), pipe-breeam (proxied via pipe-property), pipe-planning (proxied via pipe-property), pipe-energy (called by pipe-crrem), pipe-epc-logic (called by pipe-epc-imd), pipe-crrem-logic (called by pipe-crrem), pipe-property-logic (called by pipe-property), pipe-userQues (questions loaded via Drop Zone, not GPT action).

**Critical:** All OpenAPI schemas must include `x-openai-isConsequential: false` on every operation. Without it, GPT prompts confirmation on every API call. With it: one approval per session.

**Slot pressure:** All 10 slots are full. Adding any new GPT-facing Val requires consolidating into existing slots or retiring old ones.

---

## The Drop Zone — Architecture Note

The Drop Zone is one of the most important reliability mechanisms in PIPE2. It deserves its own section.

**The problem it solves:** GPT cannot reliably transport large payloads (question sets, images). It truncates, hallucinates, loses data. For anything larger than ~2KB, GPT as a data transport channel fails.

**How it works:**
1. GPT generates a secure, time-limited URL containing a session token (`pipe-dropzone-control /init`)
2. GPT sends URL to user with instructions
3. User opens URL in browser — served by `pipe-dropzone` as a rich HTML form
4. User drags/drops CSV (questions), XLSX (questions), or images (JPEG/PNG/WebP) onto the form
5. `pipe-dropzone` validates, parses, and writes directly to SQLite — no GPT in the loop
6. User returns to GPT and confirms upload
7. GPT calls `pipe-dropzone-control /status` (or `/dropCheck`) to verify ingest

**Security:** Token required, validated against DB, expires after 10 minutes, SHA-256 dedup within session, MIME type validation.

**Image processing:** Auto-cropped to square, resized to 472×472px at 200 DPI. This ensures consistent sizing in the Word report template (60mm square via InlineImage).

**File types accepted:** CSV, XLSX (question answers → `pipe_fields`), JPEG/PNG/WebP (property images → `pipe_images` + metadata to `pipe_fields`).

---

## GitHub Repository

**URL:** `github.com/basildemeroutis/pipe`

What lives here — fetched at runtime by Vals, or referenced for maintenance:

| File | Purpose |
|------|---------|
| `field_guide.csv` | **The canonical Field Guide.** 143 fields, 107 required. All Vals fetch from here at runtime. |
| `report_template.docx` | **The Word report template.** Jinja2 `{{ field_name }}` placeholders. Updated here, Flask pulls at report generation time. |
| `user_questions.csv` | **The user question set.** Loaded by Drop Zone and pipe-userQues. |
| `imd_all_fields_2025.csv` | IMD 2025 full dataset (source for pipe-imd-refresh batch import) |
| `crrem_uk_pathways_1_5C.csv` / `.json` | CRREM UK 1.5°C target pathway (2024–2050, by building type) |
| `crrem_uk_pathways_2_0C.csv` / `.json` | CRREM UK 2.0°C target pathway |
| `crrem_uk_grid_emission_factors.csv` | UK grid carbon intensity by year |
| `crrem_uk_energy_mix_defaults.csv` | UK energy mix defaults by building type |
| `crrem_fuel_emission_factors.csv` | Fuel emission factors (gas, oil, etc.) |
| `eui_reality_factors.csv` | EUI reality adjustment factors for EPC→operational conversion (tunable without code change) |
| `crrem_architecture_v2.md` | CRREM methodology documentation |
| `energy/energy_bundle.json` | 24 CIBSE/SBEM/NCM/TM46 CSV lookup tables, bundled for pipe-energy |
| `test_payload.json` | Test payload for pipe-report end-to-end test |
| `README.md` | CRREM methodology and data files reference |

**What is NOT on GitHub:**
- `user_questions.xlsx` — local only (export to CSV for GitHub)
- Flask app → PythonAnywhere `/home/BasilDemeroutis/PIPE2/flask_app.py`
- Sample report PDF → `www.forepartnership.com/pipe/PIPE_report_example.pdf`

**Deployment note:** The report template and user_questions.csv moved to GitHub as of March 2026. Flask and pipe-userQues now fetch from the repo rather than forepartnership.com hosting. If forepartnership.com links appear in older Vals, update them.

---

## PythonAnywhere Flask App (v1.5.1)

**Location:** `/home/BasilDemeroutis/PIPE2/flask_app.py`  
**Endpoints:**

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/health` | Health check |
| POST | `/generate` | Generate report from JSON payload → DOCX + PDF → Dropbox → Zapier |
| POST | `/crrem-chart` | Generate CRREM pathway chart as base64 PNG (matplotlib) |

**Templating:** docxtpl (`{{ field_name }}` Jinja2 placeholders). Images via `InlineImage` at 60mm square. Number formatting applied: `£`, `sq ft`, `tCO₂`. Excluded cost categories render as "—" not "£0". Dates normalised to DD MMM YYYY.

**Conversion:** LibreOffice headless → PDF.

**Output:** Dropbox shared links (DOCX + PDF), returned to pipe-report, forwarded to Zapier for email delivery.

---

## Database Schema (SQLite — Val.town)

**Retention:** 30 days. Sessions rejoinable within this window at property level via UPRN.

### `sessions`
| Column | Type | Notes |
|--------|------|-------|
| `session_id` | TEXT PK | `PIPE-{timestamp}-{random6}` |
| `created_at` | INTEGER | Unix ms |
| `last_activity` | INTEGER | Unix ms |
| `expires_at` | INTEGER | Unix ms (30 days from creation) |
| `user_name` | TEXT | From init call |
| `uprn` | TEXT | Set on address selection |
| `address_line_1` | TEXT | |
| `current_phase` | TEXT | DATA_COLLECTION / DROPZONE / REPORTING |
| `field_guide_version` | TEXT | Version at session creation |
| `completion_pct` | REAL | Calculated dynamically by pipe-session |

### `pipe_fields`
| Column | Type | Notes |
|--------|------|-------|
| `id` | INTEGER PK | Auto-increment |
| `session_id` | TEXT | FK → sessions |
| `field_name` | TEXT | Validated against Field Guide |
| `value` | TEXT | All values stored as text |
| `source` | TEXT | Provenance enum (see Valid Sources) |
| `stored_at` | INTEGER | Unix ms |
| `stored_by` | TEXT | Which Val stored this |

### `pipe_images`
Images uploaded via Drop Zone. Metadata also written to `pipe_fields`.

### `address_cache`
Caches Ideal Postcodes results. 14-day retention. Prevents duplicate API spend on same postcode.

### `imd_data`
Self-hosted IMD 2019/2025 data. Imported from CSV by pipe-imd-refresh. LSOA-indexed.

---

## External APIs

| API | Val(s) | Auth | Failure Behaviour |
|-----|--------|------|-------------------|
| Ideal Postcodes (AddressBase Core) | pipe-start | `IDEAL_API_KEY` | Falls back to PAF; 402 handled gracefully |
| EPC Register | pipe-epc-logic | `epc_auth_token` | Returns empty; does not block pipeline |
| Planning London Datahub (PLD) | pipe-planning | None (public) | Timeout handled; non-London returns graceful empty |
| PlanIt.org.uk | pipe-planning | None (public) | Coordinate + radius search; UK-wide; non-blocking alongside PLD |
| Historic England (NHLE) | pipe-property-logic | None (public) | "Not listed" stored explicitly; CA fallback via LPA; circuit breaker (3 failures → 60s open) |
| Land Registry (HMLR) | pipe-property-logic | `TURSO_OWNERSHIP_URL`, `TURSO_TOKEN` | Circuit breaker; freehold prioritised; address filtering |
| Companies House | pipe-companies | `CH_API_KEY` | Conditional chain: profile → officers → charges → PSC → insolvency |
| TfL Unified API | pipe-transport | `TFL_APP_KEY` | London only; non-London returns coverage note |
| BREEAM Data API | pipe-breeam | `BREEAM_USERNAME`, `BREEAM_PASSWORD` | **BLOCKED — credentials required from breeam@bregroup.com** |
| PythonAnywhere Flask | pipe-report | Internal URL | 15s timeout, retry ×2 |
| Dropbox API | pipe-report | `DROPBOX_REFRESH_TOKEN` | Token auto-refresh on 302 |
| Zapier Webhook | pipe-report | `ZAPIER_WEBHOOK` | Non-blocking; email failure doesn't fail report |

---

## Environment Variables

| Variable | Val(s) | Purpose |
|----------|--------|---------|
| `IDEAL_API_KEY` | pipe-start | Ideal Postcodes API key |
| `epc_auth_token` | pipe-epc-logic | EPC Register Basic Auth token (moved from pipe-epc-imd in v3.0.0 decomposition) |
| `TURSO_OWNERSHIP_URL` | pipe-property-logic | Turso DB URL (HMLR data, without libsql://) |
| `TURSO_TOKEN` | pipe-property-logic | Turso auth token |
| `CH_API_KEY` | pipe-companies | Companies House REST API key |
| `TFL_APP_KEY` | pipe-transport | TfL Unified API key |
| `BREEAM_USERNAME` | pipe-breeam | BREEAM API username (pending) |
| `BREEAM_PASSWORD` | pipe-breeam | BREEAM API password (pending) |
| `DROPBOX_REFRESH_TOKEN` | pipe-report | Long-lived Dropbox token (never expires) |
| `ZAPIER_WEBHOOK` | pipe-report | Zapier webhook URL for email delivery |
| `ANTHROPIC_API_KEY` | fore-services | Claude API (knowledge search, CRREM narrative) |

---

## Valid Source Taxonomy

pipe-store validates `source` on every stored field. Only these codes accepted:

```
USER, GPT, GPT_TRAINING, INTERNET_SEARCH, OS_PLACES, EPC, DEC, IMD, COMPANIES_HOUSE,
LAND_REGISTRY, PLANNING, GOOGLE_PLACES, CALCULATED, BROCHURE, CONTEXT_BINDER,
DOCUMENT, USER_TEMPLATE, DROPZONE, PLD, PLANIT, NHLE, HE_CA, HMLR, CCOD, OCOD,
POSTCODES_IO, IDEAL_POSTCODES, BENCHMARK, FALLBACK, TFL, BREEAM, ACIR
```

**Aliases** (auto-normalised by pipe-store — GPT often sends these):

| Sent by GPT | Normalised to |
|-------------|---------------|
| `PLANNING_LONDON` | `PLD` |
| `PLANNING_API` | `PLD` |
| `PLANIT_API` | `PLANIT` |
| `HISTORIC_ENGLAND` | `NHLE` |
| `LISTED_BUILDING` | `NHLE` |
| `CONSERVATION_AREA` | `HE_CA` |
| `LAND_REGISTRY` | `HMLR` |
| `IMD_2025` / `IMD2025` | `IMD` |
| `TRANSPORT_FOR_LONDON` | `TFL` |
| `COMPANIES_HOUSE_API` | `COMPANIES_HOUSE` |
| `DEC_REGISTER` | `DEC` |
| `DISPLAY_ENERGY` | `DEC` |

**When adding a new API Val:** register its source code here, in pipe-store's `VALID_SOURCES` array, and in this table if aliasing is needed.

---

## The Field Guide

**The Field Guide is the single source of truth for all field definitions.** If it's not in the Field Guide, it doesn't exist. If it's in the Field Guide, that definition overrides anything in a Val, a GPT instruction, or a conversation.

**Location:** `github.com/basildemeroutis/pipe/blob/main/field_guide.csv`  
**Val wrapper:** `esm.town/v/basild/PIPE_field_guide` (v2.1.4)  
**Current count:** 143 fields total, 107 required  

**Dynamic import pattern (cache-busting — mandatory in all Vals):**
```typescript
const { FIELD_GUIDE, TOTAL_FIELDS, TOTAL_REQUIRED_FIELDS } =
  await import(`https://esm.town/v/basild/PIPE_field_guide/main.ts?v=${Date.now()}`);
```

**Change propagation:** Edit CSV on GitHub → pipe-field-guide fetches at runtime (automatic) → pipe-store validates against it (automatic) → pipe-session phase calculation updates (automatic) → Word template `{{ field_name }}` placeholder (MANUAL) → Flask renders (automatic after template update).

**Steps 1–4 propagate automatically. The Word template is always manual.**

---

## Resilience Standards

Every Val must implement all of these. Not optional. Read `val-pipe-report` as the gold standard before building any new external API call.

| Pattern | Standard |
|---------|---------|
| Timeout | 15,000ms via `AbortController` |
| Retry | 2 attempts, exponential backoff (2,000ms × attempt) |
| CORS | `Access-Control-Allow-Origin: *` on every response |
| Health check | `GET /` returns `{ok: true, service: "...", version: "..."}` |
| OPTIONS | Returns 204 with CORS headers |
| Logging prefix | `[Turso]`, `[EPC API]`, `[TfL]`, `[PLD]`, `[Status]` etc. |
| SQLite rows | `rowToObject(columns, row)` helper — val.town returns arrays, not objects |
| Secrets | `Deno.env.get("KEY")` — never hardcoded |
| Field Guide | `?v=${Date.now()}` cache-bust on every import |
| Character budget | Val.town has an **80,000 character limit** per Val. Several Vals (pipe-session, pipe-property) are near this ceiling. Always check character count with `python3 -c "print(len(open('file').read()))"` before deploying. `wc -c` counts bytes, not characters — it overcounts when box-drawing or Unicode chars are present. |

**Logging format:**
```
╔══════════════════════════════════════╗
║ REQUEST {id} - {timestamp} - v{ver}
╚══════════════════════════════════════╝
▶ PHASE NAME
  [Subsystem] Detail
  ✓ Success
  ✖ Failure
  ⚠ Warning
▶ COMPLETE — {ms}ms
═══════════════════════════════════════
```

---

## Common Failure Modes & Fixes

**Template placeholder missing**  
Field stores correctly, passes validation, appears in `/status` — renders blank in PDF.  
Fix: Add `{{ field_name }}` to Word template, upload to PythonAnywhere, run `/test`.

**"unknown source" warnings in pipe-store logs**  
Fields store but logs are noisy. API Val sending unregistered source code.  
Fix: Add to `VALID_SOURCES` in pipe-store AND to Valid Source Taxonomy above.

**GPT hallucinating question text**  
Questions shown don't match `user_questions.csv`. GPT pattern-matching from training data.  
Fix: `pipe-session /questions` serves from database. Verify pipe-session is working.

**Duplicate session on same UPRN**  
Expected behaviour. pipe-start detects and returns `existing_session` prompt.  
User chooses "Resume" (switches session_id) or "Fresh" (new session).

**Phase not advancing**  
Required fields still missing. Check `/status` for `missing_fields`.  
Usually: a question the user skipped, or an API that returned empty silently.

**Val at 80k character limit**  
Val.town ceiling. Val.town refuses to save.  
Fix: Extract helpers to a shared utility Val, import dynamically. Do not comment out code to save space.

**BREEAM returns error**  
Expected. Credentials not yet obtained. "Not certified" is the default until resolved.

---

## Change Checklists

### Adding a New Field
1. ☐ Add row to `field_guide.csv` on GitHub
2. ☐ Verify `pipe-field-guide` exports correct count (check `/status`)
3. ☐ Test `pipe-store` accepts the field
4. ☐ Add `{{ field_name }}` to Word template (if report field)
5. ☐ Upload template to PythonAnywhere via SFTP
6. ☐ Update relevant API Val if API-sourced
7. ☐ Run smoke test, then `/test` endpoint, verify field in PDF
8. ☐ Regenerate sample report PDF, upload to forepartnership.com

### Adding a New Val
1. ☐ Read `valtown-conventions` skill before writing a line
2. ☐ Use correct naming: `val-pipe-{function}_{major}.{minor}.{patch}.ts`
3. ☐ Include mandatory file header (see valtown-conventions)
4. ☐ Register new source code in pipe-store VALID_SOURCES + taxonomy above
5. ☐ Add to Val Registry in this document
6. ☐ If GPT-facing: update OpenAPI schema, check slot count (max 10)
7. ☐ Write README.md for the Val (see per-Val README template)
8. ☐ Add endpoint to smoke-test.py

### Modifying User Questions
1. ☐ Edit `user_questions.xlsx` locally
2. ☐ Export to CSV (UTF-8)
3. ☐ Commit `.csv` to GitHub repo (`basildemeroutis/pipe`)
4. ☐ If numbering changed, verify Drop Zone field mapping
5. ☐ Add `{{ user_question_XX_text }}` and `{{ user_answer_XX }}` to Word template
6. ☐ Upload template to PythonAnywhere
7. ☐ Run smoke test

---

## Verification & Testing

### Layer 1: Smoke Test (run after every deploy)
Fast infrastructure check. No GPT, no Chrome, no UI. Tests 13+ Vals across health, validation, CORS, and E2E pipeline routes.

```bash
cd "/Users/basil.demeroutis/Documents/FORE - local/AI tools/_Claude_resources/pipe2-test-harness"

python3 smoke-test.py                        # all tests
python3 smoke-test.py --verbose              # with full response detail
python3 smoke-test.py --endpoint health      # health checks only (all endpoints)
python3 smoke-test.py --endpoint epc         # EPC+DEC+IMD only
python3 smoke-test.py --endpoint start       # pipe-start (init + address)
python3 smoke-test.py --endpoint session     # pipe-session
python3 smoke-test.py --endpoint store       # pipe-store
python3 smoke-test.py --endpoint property    # planning + heritage + HMLR
python3 smoke-test.py --endpoint crrem       # CRREM base + retrofit
python3 smoke-test.py --endpoint costs       # pipe-costs
python3 smoke-test.py --endpoint companies   # Companies House
python3 smoke-test.py --endpoint transport   # TfL transport
python3 smoke-test.py --endpoint breeam      # BREEAM (will show BLOCKED)
python3 smoke-test.py --endpoint report      # report generation
python3 smoke-test.py --endpoint dropzone    # dropzone-control
```

**Priority levels in smoke test:**
- P0: Core pipeline (start, epc-imd, session, store, property, crrem, costs, companies, transport)
- P1: Report + dropzone-control
- P2: Field Guide reachability

### Layer 2: End-to-End Test (generates real report, triggers Zapier, delivers email)
```bash
curl -X POST https://pipe-report.val.run/test
```

### Admin Dashboard
`https://pipe-admin.val.run`  
Sessions (last 30 days), field completion, raw API responses, source attribution, CRREM CSV export, address cache viewer.

---

## Key Files

| File | Location |
|------|----------|
| Field Guide | `github.com/basildemeroutis/pipe` → `field_guide.csv` |
| Report Word template | `github.com/basildemeroutis/pipe` → `report_template.docx` |
| User Questions CSV | `github.com/basildemeroutis/pipe` → `user_questions.csv` |
| IMD dataset | `github.com/basildemeroutis/pipe` → `imd_all_fields_2025.csv` |
| CRREM pathways + reference data | `github.com/basildemeroutis/pipe` → multiple CSVs + JSONs |
| Energy bundle | `github.com/basildemeroutis/pipe` → `energy/energy_bundle.json` |
| Test payload | `github.com/basildemeroutis/pipe` → `test_payload.json` |
| GPT Instructions | ChatGPT custom GPT configuration (v7.80+) |
| OpenAPI schemas | `/Users/basil.demeroutis/Documents/FORE - local/AI tools/_Claude_resources/PIPE/pipe-openapi_actions/openapi-pipe-*.json` |
| Flask app | PythonAnywhere `/Users/basil.demeroutis/Documents/FORE - local/AI tools/_Claude_resources/PIPE/pipe-flask/flask_app.py` |
| Sample report PDF | `www.forepartnership.com/pipe/PIPE_report_example.pdf` |
| Smoke test | `/Users/basil.demeroutis/Documents/FORE - local/AI tools/_Claude_resources/PIPE/pipe-testing/pipe_test_harness/smoke-test.py` |
| Test harness skill | `/Users/basil.demeroutis/Documents/FORE - local/AI tools/_Claude_resources/PIPE/pipe-testing/pipe_test_harness/SKILL.md` |
| Admin dashboard | `https://pipe-admin.val.run` |
| Federated System README | `_Claude_resources/Readme files/FORE_Federated_System_README.md` |

**Note:** `user_questions.xlsx` is local only. Export to CSV before committing to GitHub.

---

## If You Are Claude Starting a New Session

Read this document first. Then read the README.md of the specific Val you're working on. Then read the `valtown-conventions` skill. Then build.

**Never guess at field names.** Fetch the Field Guide from GitHub.  
**Never hardcode source codes.** Check the Valid Source Taxonomy above.  
**Never skip resilience patterns.** Read `val-pipe-report` before building any new external API call.  
**Never assume the deployed version matches the latest file in `_Claude_resources`.** The filename version is the version — cross-check against the Val Registry and the health check endpoint.  
**Before any significant change:** run the smoke test first to establish a baseline. Run it again after to confirm nothing broke.  
**When in doubt about architecture, stop and ask.** This is a production system used for real investment decisions. Fidelity over fluency, always.

The session ID format is `PIPE-{timestamp}-{random6}`. If a user pastes one, it's a resumable session — check pipe-admin for its current state before touching anything.

---

*This document is the source of truth for PIPE2 architecture. When you change something significant — new Val, schema change, new source code, new external API — update this document in the same session.*

*PIPE2_README.md | v1.4.0 | 10 April 2026*
