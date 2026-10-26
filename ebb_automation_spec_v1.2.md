# EBB Automation Layer — Build Spec v1.2 (frozen)

| | |
|---|---|
| **Status** | FROZEN FOR BUILD — changes only via ADR amendment + contract tests in the same PR |
| **Date** | October 2026 |
| **Supersedes** | `ebb_automation_spec_v1.1.md`; `swarm_briefing_pack.md` (all parts) |
| **Source material** | swarm_briefing_pack.md (Parts 1–6); research briefs a1 (EBB landscape), a2 (NAESB standards), a3 (technology), a4 (legal/commercial); b1 (adversarial pressure test) |
| **Conflict rule** | Where sources conflict, **b1 wins**; where b1 is silent, the pack wins; no figures are invented — every externally-sourced claim carries its source URL in §14 |

Constraint summary (from b1 §D, baked in): Temporal Fairness keys replace any bespoke fair-queue broker; connector code never runs inside the FastAPI service; channel doctrine is stated in §1 and §2.4; audit design is hash-chain + Object Lock anchoring with **no QLDB anywhere**; htmx pinned to 2.0.x with **hyperscript banned**; NAESB acquisition, eTariff compliance mapping, agency onboarding, C&D protocol, storm mode, NMQR 5-minute handling, SOC 2, certification path, and the EDI build-vs-partner decision are all in scope with owners (§7).

---

## 1. Product definition

The EBB Automation Layer is a multi-tenant SaaS that submits, verifies, and receipts natural gas pipeline nominations on behalf of shippers across the ~9 EBB platforms that cover ~80+ of the ~120 interstate EBBs (b1 §A1; a1 §2.7). Its users are (a) the **gas scheduler**, who submits and re-nominates across the five daily cycles, and (b) the **desk manager**, who needs a desk-level "did today go okay" view (pack 2.1 Q1). The single worst failure the product exists to prevent, in the customer's own words: **"your system said it was submitted and it wasn't"** (pack 2.1 Q2). Every architectural decision — verification engine, receipts, hash-chained audit, escalation, kill switches, the dead man's switch — is traceable to that sentence. The product's channel doctrine (b1 §A9, §B2): **NAESB EDI/EDM is the tariff-sanctioned primary submission channel wherever the pipeline offers it and the customer's enrollment justifies it; portal automation and flat-file upload are (a) the wedge for small/mid shippers who will never enroll in EDI (>80% of US nominations are still entered manually into EBB portals — Capco, Apr 2026) and (b) the resilience fallback for EDI outages** (which force manual portal entry with extended deadlines in production — TC Energy, Jan 2026). Flat-file upload (verified on Transco 1Line, KM DART, TC eConnects) is a third sanctioned channel, cheaper and stabler than browser bots where offered. Per-pipeline channel discovery (sanctioned API? active EDI? flat-file? portal-only?) is a mandatory onboarding step and must include a check for the Energy Transfer Messenger+ Nominations API (dev.messenger.energytransfer.com), the one known pipeline-sanctioned API.

**Storm mode.** The product's ROI case is the winter-storm penalty record (a4 §5, Imp.9): Storm Uri (Feb 2021) produced $21.6M in disputed OFO penalties for Luminant/Vistra from Energy Transfer, ~$150M demanded of Spire Missouri by Symmetry at prices peaking at $1,556.96/MMBtu (617× monthly index, hitting 10 hospitals and 103 school districts), and ~$75M at Panhandle Eastern (secondary source, medium confidence). Winter Storm Elliott (Dec 2022) saw 14 force-majeure declarations, 156 firm shippers curtailed, OFO issuances peaking Dec 23 and **critical notices peaking Christmas Eve**, and Transco Zone 5 prices rising 8× in two days. Nominations, re-nominations, and OFO responses therefore fail worst exactly when humans are least available. Storm mode (REQ-039…REQ-042) is the productized answer: real-time OFO/critical-notice ingestion across all covered EBBs, penalty-exposure calculation, per-pipeline per-cycle countdowns, and auto-renom with elapsed-prorated-scheduled-quantity (EPSQ) awareness. Storm mode is always-on with elevated alerting thresholds during detected OFO/critical-notice surges or manual declaration; there is no separate "activation."

**Users, trust, and churn.** Churn in this product does not look like cancellation; it looks like "they stopped trusting it for the important paths" (pack 2.2 Q7). The trust-building product surface is shadow mode with a visible per-pipeline match-rate dashboard ("we matched your human 100% for 30 days") and the receipt viewer. The early-warning churn metric (shadow-mode match rate and live-path usage share) is a v1 instrumentation requirement (REQ-027).

---

## 2. Domain model & regulatory ground truth

### 2.1 Gas day and the five nomination cycles

**Gas day = 9:00 a.m. to 9:00 a.m. Central Clock Time (CCT)** — America/Chicago *with* DST, so a gas day is 23 or 25 hours twice a year (a3 §5). Order No. 809 (Apr 16, 2015, Docket RM14-2; pipelines complied from Apr 1, 2016) adopted the current NAESB WGQ Std 1.3.2 timeline, still current in October 2026. **All times below are Central Clock Time.**

| Cycle | Nom leaves control | Receipt by TSP | Quick Response | Confirmations due | Scheduled quantities issued | Flow effective | Bumpable? |
|---|---|---|---|---|---|---|---|
| Timely (day prior) | 1:00 p.m. | 1:15 p.m. | 2:00 p.m. | 3:30 p.m. | 4:30 p.m. | 9:00 a.m. next Gas Day | n/a (day-ahead) |
| Evening (day prior) | 6:00 p.m. | 6:15 p.m. | 6:30 p.m. | 8:30 p.m. | 9:00 p.m. | 9:00 a.m. next Gas Day | Yes |
| Intraday 1 (ID1) | 10:00 a.m. | 10:15 a.m. | 10:30 a.m. | 12:30 p.m. | 1:00 p.m. | 2:00 p.m. | Yes |
| Intraday 2 (ID2) | 2:30 p.m. | 2:45 p.m. | 3:00 p.m. | 5:00 p.m. | 5:30 p.m. | 6:00 p.m. | Yes (since Order 809) |
| Intraday 3 (ID3) | 7:00 p.m. | 7:15 p.m. | 7:30 p.m. | 9:30 p.m. | 10:00 p.m. | 10:00 p.m. | No (no-bump) |

**Bump rules:** Timely is day-ahead (no intraday bump concept); Evening and ID1 are bumpable; ID2 has been bumpable since Order 809; ID3 is no-bump. The bumpability flag must round-trip through the canonical model (ADR-001).

**Version caveat (b1 §A14, C7):** WGQ v4.1 (ratified Nov 24, 2025) cuts the Nomination Quick Response to **max 5 minutes from receipt** (new stds 1.3.83–1.3.85, replacing 1.3.37/1.3.45). The QR column above is the v4.0/tariff baseline; pipelines implementing 4.1 will QR faster. The verification engine must evaluate QRs against a 5-minute expectation per REQ-019, and the load model, countdown UI, dead man's switch, and the "no deploys after noon CT" rule must cover **all five** cycle deadlines, not just Timely 1:00 p.m.

**Per-pipeline variation:** tariffs can vary the 1.3.2 table. The canonical model encodes the full table per pipeline via a pipeline manifest (REQ-004), seeded from eTariff compliance-map ingestion (REQ-043) and reviewed by the domain expert (§7).

### 2.2 NAESB dataset inventory → X12 transaction sets

From NAESB's official X12 cross-reference (a2 §2.2; datasets are X12 v003040 unless noted):

| WGQ dataset | Name | X12 set | Dataset code |
|---|---|---|---|
| 1.4.1 | Nomination | 850 | G850NMST |
| 1.4.2 | Nomination Quick Response (NMQR) | 855 | G855NMQR |
| 1.4.3 | Request for Confirmation | 850 | G850RQCF |
| 1.4.4 | Confirmation Response | 855 | G855RRFC |
| 1.4.5 | Scheduled Quantity | 865 | G865SQTS |
| 1.4.6 | Scheduled Quantity for Operator | 865 | G865SQOP |
| 1.4.7 | Confirmation Response Quick Response | 855 | G855CRQR |
| 2.4.1 | Predetermined Allocation (PDA) | 860 | — |
| 2.4.3 | Allocation | 865 | — |
| 2.4.4 | Shipper Imbalance | 811 | — |
| 2.4.5/2.4.6 | Measurement | 867 | — |
| 3.4.1 | Invoice | 811 (v004010) | — |
| 3.4.2/3.4.3 | Remittance / Administrative message | 820 / 822 | — |
| 5.x | Capacity release family | 840/843/832 | — |

v1 EDI scope: the seven 1.4.x datasets only. Flowing gas, invoicing, and capacity release are out of scope for v1 (canonical model anticipates them; see ADR-001 versioning rule).

**Transport (a2 §2.3):** NAESB QEDM / Internet Electronic Transport descends from GISB EDM over HTTP/S, deliberately converged with IETF EDIINT AS2 (clarification C03003). An AS2-capable endpoint is the right architecture, but transport is implemented against the NAESB IET/QEDM manuals (10.x), **not** against generic-AS2 byte-compatibility assumptions. Mendelson AS2 community edition (b67, Mar 2025) is a containerized test peer only — it is GPL and must never be embedded (b1 §A15).

### 2.3 NMQR 5-minute rule and version-skew policy

- FERC currently mandates (18 CFR §284.12) **WGQ v4.0** (Sept 29, 2023) + Minor Correction MC24002 + Order 587-AA gas-electric coordination (eff. Feb 7, 2025; compliance Aug 1, 2025) + Order 587-AB (May 22, 2026). NAESB publishes and sells **v4.1** (published/filed Sept 25, 2026).
- v4.1 change that drives engineering: NMQR within **max 5 minutes of receipt** (stds 1.3.83–1.3.85, ratified Nov 24, 2025).
- **Policy (frozen):** buy v4.1 (superset; §7 acquisition plan). The **per-pipeline compliance map** (Order 587-V tariff section, REQ-043) — not the FERC baseline and not the NAESB publication date — is the authority on what each pipeline actually implements. Every pipeline manifest carries a `naesb_version_effective` field; the verification engine resolves QR windows and dataset layouts from the manifest, so per-pipeline version skew is configuration, not code.

### 2.4 Access-channel matrix per platform

~9 platforms cover ~80+ of the ~120 interstate EBBs and 14+ of the top-20 systems (a1 §2.7, §5). Connector economics are platform economics, not pipeline economics. There is no general public API catalog and no public sandbox (a1 §2.3); the sole known exception is Energy Transfer's Messenger+ Nominations API (medium confidence, aggregator source — verify at onboarding).

| Platform | Representative pipelines | Channels available | Notes |
|---|---|---|---|
| Kinder Morgan **DART** | TGP, NGPL, El Paso, CIG, Southern Natural, Cheyenne Plains, MEP, KM Louisiana, Mojave, Sierrita, TransColorado, WIC, Stagecoach, KMIL, Elba Express, Southern LNG, storage (~20 entities) | Portal; "Submitted Batch File" flat-file; EDI via DART EDI Business Support Group / Trading Partner Worksheet | GTC §34.8 contemplates agents holding logon credentials; named-user IDs + subscriber liability (§34.8–34.9); Interstate ESA agreement required |
| TC Energy **eConnects** | Columbia Gas, Columbia Gulf, ANR, ANR Storage, Millennium, Northern Border, Bison, Crossroads, PNGTS, Hardy, Blue Lake, Great Lakes/GTN legacy | Portal; nomination upload feature (flat-file); EDI (inbound EDI confirmations + nominations) | Signed Electronic Contracting Agreement (ECA) required; ANR GTC 6.6.2(g) agent designation |
| Williams **1Line** | Transco, Northwest Pipeline, Gulfstream, Pine Needle, Discovery, Black Marlin | Portal; CSV flat-file upload; EDI/EDM | Tariff language: "Transporter shall accept nominations twenty-four (24) hours a day via 1Line or EDM" |
| Enbridge **LINK** | Texas Eastern (TETCO), Algonquin, East Tennessee, Sabal Trail, Nexus, M&NE US, Valley Crossing, SESH, offshore + storage (~20 entities) | Portal; EDI/EDM (same tariff language family) | Enbridge access agreement (full text not public — collect at onboarding) |
| Energy Transfer **Messenger/ipost** | Panhandle Eastern, Florida Gas Transmission, Trunkline, Sea Robin, Transwestern, Tiger, Fayetteville Express, Southwest Gas Storage, Trunkline LNG | Portal; **Messenger+ Nominations API** (dev.messenger.energytransfer.com — verify); EDI | Candidate first sanctioned-API connector (b1 §A8) |
| Berkshire Hathaway **BHE GTS** (bhegts.com / Dekaflow CAW) | Eastern Gas Transmission & Storage, Cove Point LNG, Carolina Gas Transmission | Portal; EDI | — |
| **Quorum** (My Quorum Pipeline / myquorumcloud) | Boardwalk pipes (AlaTenn, Midla, Trans-Union, Ozark), High Point family, MountainWest (MWP, MWOP, White River Hub) | Portal; EDI (vendor is NAESB-certified) | Quorum claims 144+ assets, 80,000+ miles |
| **Trellis Energy (ptms)** | DT Midstream (Guardian, Midwestern Gas Transmission, Viking), NextEra Florida Southeast Connection | Portal; EDI | DTM migration is an M&A-drift example — re-verify platform attribution at build time |
| **gasnom.com** (shared hosted EBB) | Cameron Interstate, Golden Pass, LA Storage, Golden Triangle, Mississippi Hub, DBM Pipeline, Central Valley Gas Storage | Portal | Operator unconfirmed (WHOIS privacy) |
| **Custom one-offs** (individual builds, not platform connectors) | Northern Natural, Equitrans/MVP (Salesforce portal), Cheniere, Iroquois, Kern River, WBI, National Fuel, Southern Star, DCP, Tallgrass, Alliance, Vector, Gulf South, REX, Texas Gas | Varies per pipeline | Outside the platform connector fleet |

**EDI availability caveat (a1 §2.4):** FERC routinely grants EDI/EDM extensions "until such time as [the pipeline] is requested by a customer to provide such electronic data services" — many pipelines carry **zero active EDI demand**. Channel discovery must therefore verify not just "EDI offered in tariff" but "EDI active and enrollable." EDI enrollment is per-pipeline, bilateral, and takes weeks-to-months per pipeline (NAESB GEH whitepaper: "potentially months… with each pipeline"; generic benchmark 4–6 weeks per trading partner).

### 2.5 Definitions — the "confirmed" state ladder

"Confirmed" is not a boolean (b1 §B8). The product's state ladder, mapped to the actual NAESB datasets, with frozen UI semantics:

| Rung | Internal state | Backing evidence | UI semantics (frozen) |
|---|---|---|---|
| 0 | `DRAFT` / `VALIDATED` | Local validation only | Gray, text "Not submitted" |
| 1 | `SUBMITTED_UNVERIFIED` | Connector reports transport-level acceptance | Amber, text "Submitted — awaiting quick response". **Never displayed as confirmed.** |
| 2 | `QR_RECEIVED` | NMQR (1.4.2) or portal-equivalent acknowledgment ingested and timestamped, evaluated against the 5-minute window | Amber, text "Acknowledged by pipeline" + QR timestamp |
| 3 | `CONFIRMED` | Confirmation Response (1.4.4) or portal confirmation screen captured with quantity matching the nominated quantity | Green, text "Confirmed" + confirmed quantity |
| 4 | `SCHEDULED` | Scheduled Quantity (1.4.5/1.4.6) or portal scheduled-quantity posting captured | Green, text "Scheduled" + scheduled quantity; if scheduled < confirmed, text "Scheduled (cut)" with both quantities |
| — | `FAILED` / `ESCALATED` / `KILLED` | Error taxonomy (ADR-002) / escalation policy / kill switch | Red, text naming the failure class; never a bare color |

UI rule (WCAG, REQ-045): status is **never carried by color alone** — every status pairs an icon/dot with text. Where the product's definition of "confirmed" is stricter than a scheduler's colloquial usage, the UI shows the ladder rung label verbatim rather than collapsing to a boolean (pack 2.1 Q3).

---

## 3. Frozen v1 interfaces (ADRs)

Each interface below is an Architecture Decision Record: decision, rationale, schema/contract, versioning rule. **Global rule: interfaces are immutable within v1. Any change requires an ADR amendment and updated contract tests in the same PR (pack 4.1 #2). Contract tests enforce these interfaces at every merge.**

---

### ADR-001 — Canonical Nomination schema

**Decision.** One canonical nomination model is the only internal representation; every connector, the EDI layer, the ingestion API, and the verification engine speak it. Per-pipeline dialects live in pipeline manifests, never in the model.

**Rationale.** A wrong canonical model is the most expensive possible refactor (b1 §E risk 7). The model anticipates named ETRM mappings (NatGasHub ships neutral APIs for Endur, Allegro, FIS, PCI; ION Allegro bundles NatGasHub Connect — a4 §2, b1 C14), so field names and types are stable and map-shaped.

**Schema (v1, frozen; JSON representation; `snake_case`; all datetimes UTC ISO-8601 instants):**

| Field | Type | Req'd | Definition |
|---|---|---|---|
| `nomination_id` | UUIDv7 | yes (server-set) | Internal primary key |
| `idempotency_key` | string, ≤128 chars | yes | Client-supplied; unique per (tenant, pipeline); duplicate submission with the same key returns the original nomination, never double-submits |
| `trace_id` | UUIDv7 | yes (server-set) | Distributed trace root, queue-to-receipt (REQ-051) |
| `tenant_id` | UUID | yes | Tenant scope (server-set from auth context; never trusted from payload) |
| `pipeline_id` | string (registry key) | yes | Pipeline registry identifier, e.g. `km_dart:tgp` (platform:entity) |
| `cycle` | enum: `TIMELY`, `EVENING`, `ID1`, `ID2`, `ID3` | yes | One of the five WGQ 1.3.2 cycles |
| `gas_day` | date (YYYY-MM-DD) | yes | The gas day the nomination flows for; deadlines computed from (gas_day, cycle) via the pipeline manifest by the time engine (REQ-003) |
| `service_requester` | object | yes | `{ "entity_code": string, "name": string, "role": enum: SHIPPER, AGENT }` — NAESB Service Requester = "the Shipper or its Agent" |
| `agent` | object \| null | yes (null allowed) | `{ "entity_code": string, "name": string }` — the dedicated Agent data element (FERC Order 587-Y); populated when we act as agent |
| `service_agreement` | object | yes | `{ "agreement_id": string, "rate_schedule": string, "contract_number": string \| null }` |
| `receipt_point` | object | yes | `{ "location_code": string, "location_name": string \| null, "upstream_party": string \| null, "zone": string \| null }` — location_code is the pipeline's published location/DRN code |
| `delivery_point` | object | yes | Same shape, with `downstream_party` |
| `quantity_dth` | decimal(12,3) | yes | Daily quantity in dekatherms; positive |
| `begin_datetime_utc` | datetime | server-derived | Flow start instant, computed from (gas_day, cycle) — never client-trusted |
| `end_datetime_utc` | datetime | server-derived | Flow end instant |
| `rollable` | boolean | yes | Rollable flag where the pipeline supports it |
| `bumpable` | boolean | yes | Must be `false` for ID3 (no-bump cycle); validated against §2.1 |
| `fuel_recourse` | enum: `IN_KIND`, `CASH`, `PIPELINE_DEFAULT` \| null | no | Anticipates ETRM mappings; optional in v1 |
| `rank` | integer \| null | no | Service-requester ranking where supported |
| `deal_id` | string \| null | no | Customer/ETRM deal reference (free-form, echoes back on receipts) |
| `source` | enum: `UI`, `API`, `FILE_IMPORT`, `AUTO_RENOM` | yes | Ingestion path |
| `channel_hint` | enum: `EDI`, `FLAT_FILE`, `PORTAL`, `API`, `AUTO` \| null | no | Preferred channel; `AUTO`/null = router picks per channel-discovery state |
| `metadata` | object | no | Namespaced extension bag; reserved prefixes: `etrm.*`, `storm.*` |

**Validation invariants (server-side, tested):** `quantity_dth > 0`; `cycle`/`bumpable` consistency; `gas_day` resolvable against the pipeline manifest; duplicate `idempotency_key` → idempotent replay of original response; `tenant_id` always from auth context.

**Versioning rule.** Additive optional fields → minor ADR amendment, allowed in v1.x. Type changes, removals, or semantic changes → v2 model with server-side migration and dual-accept window; never silent.

---

### ADR-002 — Connector interface (narrow internal contract)

**Decision.** Connectors are versioned plugins running in a separate worker tier, implementing exactly six operations as Temporal activities. Plugin code is untrusted by default; the contract is the only coupling.

**Rationale.** Pack 3.4 as amended by b1 §B4: untrusted browser automation never runs in-process with FastAPI; one plugin version = one immutable image digest; old/new versions run side-by-side per pipeline.

**Operations (all async, all keyed by `(tenant_id, pipeline_id)`, all emit `trace_id` spans):**

```
discover_channels(pipeline_id, credentials_ref) -> ChannelReport
    ChannelReport = { channels: [{type: EDI|FLAT_FILE|PORTAL|API,
        status: ACTIVE|OFFERED_INACTIVE|UNAVAILABLE, evidence: string,
        discovered_at: datetime}], api_candidate: {present: bool, url: string|null} }
    # includes the Messenger+ API check for ET Messenger pipelines

submit(nomination: CanonicalNomination, channel, credentials_ref) -> SubmitResult
    SubmitResult = { connector_txn_id: string, transport_ack: bool,
        submitted_at: datetime, raw_artifact_refs: [ArtifactRef] }

poll_status(connector_txn_id, credentials_ref) -> StatusPollResult
    StatusPollResult = { ladder_rung: 0..4, qr: QRRecord|null,
        confirmation: ConfirmationRecord|null, observed_at: datetime,
        raw_artifact_refs: [ArtifactRef] }

fetch_confirmation(connector_txn_id, credentials_ref) -> ConfirmationRecord
    ConfirmationRecord = { confirmed_quantity_dth: decimal,
        confirmed_at: datetime, reducing_conditions: [string],
        raw_artifact_refs: [ArtifactRef] }

fetch_scheduled_quantities(pipeline_id, gas_day, cycle, credentials_ref) -> ScheduledQuantityRecord
    ScheduledQuantityRecord = { scheduled_quantity_dth: decimal,
        issued_at: datetime, operator_view: bool, raw_artifact_refs: [ArtifactRef] }

health_check(pipeline_id) -> HealthResult
    HealthResult = { canary_ok: bool, drift_detected: bool,
        last_successful_canary: datetime|null, details: string|null }
```

`ArtifactRef = { object_key: string, sha256: string, content_type: string }` — references into the Object Lock artifact bucket (ADR-004/ADR-006).

**Error taxonomy (frozen — connectors must classify, never invent):**
| Class | Meaning | Orchestrator action |
|---|---|---|
| `RETRYABLE` | Transient transport/portal failure | Temporal retry policy; capped backoff |
| `TERMINAL_VALIDATION` | Pipeline rejected the nomination (bad quantity/point/agreement) | Fail the workflow; surface reason verbatim; escalate per policy |
| `AUTH` | Credential rejected/expired | Fail; mark credential status; escalate immediately (never retry loops against auth) |
| `BLOCKED` | Platform actively blocking automation (throttle, challenge, C&D signal) | **Stop all activity for that pipeline, engage per-pipeline kill state, alert** (REQ-026; Power Ventures line — never circumvent) |
| `DRIFT` | Portal/EDI structure changed vs. golden fixtures | Quarantine connector version; canary-fail; alert; no auto-submit on drifted connector |
| `INTERNAL` | Connector bug | Retry once; then TERMINAL for the attempt; alert |

**Version negotiation.** Every connector declares `interface_version: "1"` and `supported_channels`. The orchestrator refuses to schedule activities to a connector whose `interface_version` it does not understand. Connector deployments are digest-pinned and rolled per REQ-046.

**Versioning rule.** Six operations only in v1. A seventh operation or a breaking signature change requires ADR amendment + contract-test updates in one PR.

---

### ADR-003 — Submission state machine (Temporal workflow)

**Decision.** Each nomination's lifecycle is one Temporal workflow execution (`WorkflowId = nomination_id`, idempotent start). All timers, retries, signals, and escalations are workflow code. No second scheduler exists anywhere in the system (b1 §B4).

**Rationale.** Hours pass between submission and scheduled-quantity publication; durable timers must survive restarts and deployments (pack 3.1 Q3). Temporal Task Queue Fairness (`fairness_key = f"{tenant_id}:{pipeline_id}"`, tenant-tier weights) supplies per-tenant/per-pipeline fairness at the five daily cutoff spikes (b1 §A4/§B5).

**States and transitions (aligned 1:1 to workflow code):**

```
DRAFT → VALIDATED → QUEUED → SUBMITTING → SUBMITTED_UNVERIFIED
    → QR_RECEIVED → CONFIRMED → SCHEDULED            (happy path)

VALIDATED → FAILED(TERMINAL_VALIDATION)
SUBMITTING → SUBMITTED_UNVERIFIED            (transport ack)
SUBMITTING → FAILED (after retry budget)     → ESCALATED
SUBMITTED_UNVERIFIED --[QR window elapsed]--> ESCALATED(QR_LATE)
QR_RECEIVED --[confirmation deadline per manifest elapsed]--> ESCALATED(CONFIRMATION_LATE)
CONFIRMED --[scheduled-quantity deadline elapsed]--> ESCALATED(SCHEDULED_LATE)
CONFIRMED --[scheduled qty < confirmed]--> CUT --> SCHEDULED (records nominated-vs-scheduled delta; notifies per escalation policy; offers next-cycle EPSQ-aware renomination, REQ-060)
CONFIRMED --[amendment submitted before cycle deadline]--> AMENDED --> QR_RECEIVED (re-verified from the QR rung; REQ-059)
SUBMITTING/QUEUED --[pipeline unreachable; retries pending]--> PIPELINE_DOWN --> SUBMITTING (distinct UI state, never shown as submitted; REQ-063)
any state --[kill switch]--> KILLED          (terminal; no further pipeline contact)
ESCALATED --[human: retry|edit|abandon]--> QUEUED | VALIDATED | CLOSED
any non-terminal state --[C&D/BLOCKED]--> KILLED + pipeline-level halt
```

**Timers (all computed fresh per gas day by the time engine; scheduled as UTC instants):**
- `T_submit_by`: pipeline leave-control time minus submission-latency budget (per-manifest).
- `T_qr_deadline`: **5 minutes after observed receipt** on v4.1-effective pipelines; the §2.1 tariff QR column otherwise (per-manifest `naesb_version_effective`). Missing/late QR inside the window triggers escalation **while there is still time to renominate** (b1 C7(b)).
- `T_confirm_deadline`, `T_scheduled_deadline`: per §2.1/manifest.
- Retry policy per error class (ADR-002); `RETRYABLE` capped exponential backoff; `AUTH`/`BLOCKED`/`DRIFT` never auto-retried.

**New states (v1.2 additions).** `AMENDED` (REQ-059): an accepted nomination amended before the cycle deadline re-enters verification at the QR rung; post-deadline amendments are rejected with the deadline in the error. `CUT` (REQ-060): scheduled quantity below nominated quantity records the nominated-vs-scheduled delta, notifies per escalation policy, and offers next-cycle EPSQ-aware renomination before proceeding to `SCHEDULED`. `PIPELINE_DOWN` (REQ-063): portal/EDI unreachability detected via canaries or submit failures; the UI shows this state distinctly and **QUEUED-for-retry is never surfaced as submitted** — a nomination sitting in a retry queue while the pipeline was unreachable is never reported as "submitted" (§2.5 UI semantics; the product's founding failure sentence).

**Escalation triggers (to the escalation policy engine, REQ-021):** QR late; confirmation late; scheduled late; TERMINAL_VALIDATION; AUTH; BLOCKED; DRIFT; cutoff-at-risk (ETA to `T_submit_by` breach); cut recorded; pipeline-down detected. Escalation carries `trace_id`, ladder rung, and time remaining to the relevant deadline.

**Exactly-once rule.** Dedup by `idempotency_key` at intake + workflow-id reuse + post-submit reconciliation (poll_status cross-check against pipeline-visible state) is the accepted control set; the residual double-submit risk is owned contractually in the ToS/MSA (pack 3.1 Q4, §7 legal).

**Versioning rule.** State names and transitions are frozen; new escalation-trigger types are additive minor amendments.

---

### ADR-004 — Receipt format

**Decision.** Every nomination attempt produces a receipt: a human-readable, exportable record that is the proof object for "it was submitted" (pack 2.3). Receipts are write-once artifacts in the S3 Object Lock bucket (COMPLIANCE mode) with per-object SHA-256 recorded in the audit chain (ADR-006).

**Rationale.** The receipt viewer is the demo feature and the dispute artifact; it must survive operator tampering accusations. Object Lock COMPLIANCE retention must be enabled at bucket creation and is irreversible — retention budgeting is an infra checklist item (b1 §A7).

**Schema (v1; JSON index record + referenced artifacts):**

| Field | Type | Definition |
|---|---|---|
| `receipt_id` | UUIDv7 | Receipt key |
| `nomination_id` / `trace_id` | UUID | Join keys |
| `tenant_id`, `pipeline_id`, `cycle`, `gas_day` | — | Echo of ADR-001 identity fields |
| `ladder_rung_final` | int 0–4 | Highest verified rung reached |
| `events` | array | Ordered `{at: datetime, event: string, evidence_artifact: ArtifactRef|null}` — submission, QR, confirmation, scheduled quantity, each with its captured timestamp |
| `screenshot_artifacts` | array of ArtifactRef | Portal screenshots (or rendered EDI 855/865 human-readable views), per-object `sha256` |
| `quantities` | object | `{nominated, confirmed, scheduled}` Dth |
| `pipeline_reference` | string \| null | Pipeline-side transaction/control number when visible |
| `generated_at` | datetime | Receipt generation instant (UTC) |
| `audit_row_ref` | string | audit_events row id binding this receipt into the hash chain |

**Export contract.** Per-receipt export as (a) self-contained PDF bundle (human-readable timeline + embedded screenshots + quantity table + hash manifest) and (b) JSON (this schema + hash manifest). Bulk export API per ADR-005. Every exported artifact verifies against its recorded SHA-256; the export bundle includes the verification instructions.

**SLO:** receipt available within **60 seconds p99** of the triggering pipeline evidence (REQ-046).

**Versioning rule.** Additive fields only in v1.x; `events.event` vocabulary is a closed enum extended by amendment.

---

### ADR-005 — Internal REST API contracts (FastAPI surface)

**Decision.** One FastAPI application serves both JSON REST and htmx partials. Tenant scope comes from the authenticated session/JWT via dependency injection (RLS context); no endpoint trusts client-supplied `tenant_id`. The UI never touches a pipeline and never decides business logic (pack 5.2); client-side validation/filtering/sorting is allowed, but anything involving money, deadlines, or state transitions is server-side.

**Rationale.** One service keeps the small-team "single control plane" goal (b1 §B4): FastAPI is presentation + API only, with all orchestration delegated to Temporal and all pipeline contact delegated to the sandboxed connector tier. Keeping JSON and htmx partials in one app guarantees the UI and any future API consumers see identical state.

**Endpoint list (v1; all prefixed `/api/v1`, all return `trace_id` in headers):**

| Method & path | Purpose | Request → Response (shape) |
|---|---|---|
| `POST /nominations` | Create nomination | CanonicalNomination (ADR-001) → `{nomination_id, trace_id, state}`; 200 on idempotent replay |
| `GET /nominations` | List/filter nominations | query: gas_day, cycle, pipeline, state → paginated list |
| `GET /nominations/{id}` | Detail + ladder state | → `{nomination, ladder, events, receipt_ref}` |
| `POST /nominations/{id}/cancel` | Abandon before submit | → `{state}` |
| `POST /nominations/{id}/renominate` | Create follow-cycle renom (EPSQ-aware fields) | CanonicalNomination delta → as POST /nominations |
| `POST /nominations/{id}/amend` | Amend a pipeline-accepted nomination (quantity/points) before the cycle deadline per NAESB rules (REQ-059) | CanonicalNomination delta → `{state}`; post-deadline → RFC-9457 error carrying the deadline |
| `POST /ingest/file` | File import (ETRM/csv) mapped to canonical model | multipart → `{accepted[], rejected[{row, error}]}` |
| `GET/POST/PUT/DELETE /escalation-policies` | Escalation policy CRUD (thresholds, channels, on-call rotation, quiet hours) | policy JSON |
| `GET /on-call/current` | Current on-call per tenant | → `{user, until}` |
| `GET /pipelines` | Pipeline registry + per-pipeline health | → `[{pipeline_id, platform, health, last_canary, drift}]` |
| `GET /pipelines/{id}/health` | Single pipeline health detail | HealthResult + canary history |
| `POST /pipelines/{id}/toggle` | Per-pipeline automation toggle (kill) | `{enabled: bool}` → ack + audit event |
| `POST /tenant/kill-switch` | Customer-level one-click stop-all | → ack + audit event; verified weekly per REQ-025 |
| `GET /receipts/{id}` | Receipt JSON | ADR-004 schema |
| `GET /receipts/{id}/export?format=pdf\|json` | Receipt export bundle | file download |
| `GET /admin/users`, `POST /admin/users`, `PUT /admin/users/{id}` | User/role admin | RBAC objects |
| `GET /admin/credentials` | Credential **status** (never values) | `[{credential_id, pipeline_id, status, last_used_at}]` |
| `GET /admin/audit/export` | Audit export (admin-only, itself audited) | query: range, filters → JSONL + chain-head proof |
| `GET /agency-onboarding`, `POST /agency-onboarding` | Agency-onboarding workflow state/tasks | workflow objects (REQ-029) |
| `GET /shadow/match-rate` | Shadow-mode match-rate dashboard data | → per-pipeline match-rate series |
| `GET /storm/notices`, `GET /storm/exposure` | Storm-mode surfaces | notice list; exposure calculator input/output |
| `GET /healthz`, `GET /readyz` | Service health (unauthenticated, minimal) | — |

**Conventions.** Errors: RFC-9457 problem+json with frozen error codes. Pagination: cursor-based. All mutating endpoints write audit events (ADR-006).

**Versioning rule.** URL prefix `/api/v1`; breaking changes require `/api/v2` served alongside, never mutation of v1 shapes. Additive endpoints/fields are minor ADR amendments allowed in v1.x, with contract tests updated in the same PR.

---

### ADR-006 — Audit event schema (append-only, hash-chained)

**Decision.** One append-only Postgres `audit_events` table, trigger-immutable (UPDATE/DELETE revoked + trigger enforcement), written by a **single-writer audit service**, with a per-row SHA-256 hash chain. A nightly job re-walks the chain; the daily chain head is anchored into an S3 Object Lock COMPLIANCE-mode object for operator-independent tamper evidence. **No QLDB (end of support July 31, 2025). immudb only if auditors later demand per-record third-party proofs.**

**Rationale.** b1 §A7/§B7: the composition (queryable hash-chained table + external anchoring), not an either/or. The audit store is also a SOC 2 evidence generator (b1 C4).

**Schema (v1):**

| Column | Type | Definition |
|---|---|---|
| `seq` | bigserial PK | Strictly monotonic, gap-checked on re-walk |
| `event_id` | UUIDv7 | Event key |
| `at` | timestamptz | UTC instant (server clock) |
| `tenant_id` | UUID \| null | Null only for system-level events |
| `actor` | object | `{type: USER|SYSTEM|CONNECTOR, id: string, credential_ref: string|null}` |
| `action` | closed enum | e.g. `NOMINATION_CREATED`, `SUBMIT_DISPATCHED`, `QR_INGESTED`, `RECEIPT_GENERATED`, `KILL_SWITCH_ENGAGED`, `PIPELINE_TOGGLED`, `CREDENTIAL_ACCESSED`, `PORTAL_ACTION`, `ADMIN_CHANGE`, `AUDIT_EXPORTED` |
| `object_refs` | jsonb | `[{kind, id, sha256?}]` — nominations, receipts, artifacts |
| `trace_id` | UUID \| null | Join to the nomination trace |
| `payload_digest` | string | SHA-256 of the canonical JSON of the event payload (payload itself in jsonb `payload`) |
| `payload` | jsonb | Full event detail |
| `prev_hash` | string(64) | SHA-256 of the previous row's `row_hash` (genesis row: 64 zeroes) |
| `row_hash` | string(64) | SHA-256 over `(seq, event_id, at, actor, action, object_refs, payload_digest, prev_hash)` |

**Invariants (tested):** single writer enforced by role grants; `prev_hash` linkage unbroken on nightly re-walk; gap-free `seq`; chain head anchored daily to Object Lock; anchor object records `{date, seq_hi, row_hash_head}`. Retention and legal-hold policy per REQ-037.

**Versioning rule.** Hash input composition is frozen for v1 (any change forks the chain with an explicit `CHAIN_FORK` event recording both algorithms). The `action` enum extends by amendment only.

---

### ADR-007 — Tenancy and data isolation

**Decision.** Shared application, shared database with Postgres row-level security (RLS) enforced at the **database level** (not application code); tenant context injected per request via FastAPI dependency injection (per ADR-005). A cross-tenant access test suite (attempts to read/write other tenants' rows through every repository path) runs as a blocking CI gate.

**Credentials.** Separate vault service (Vault-class or cloud KMS envelope encryption), per-tenant data-encryption keys; credentials never in the application database, logs, or traces; every credential access is an audited event; named-user model per pipeline ESA structures (REQ-031/REQ-032).

**Artifacts (receipts/screenshots).** Shared Object Lock bucket, per-tenant prefixes, bucket policy denies cross-prefix access, per-object SHA-256 (unchanged from ADR-004/006).

**Cache.** Shared instance; every key tenant-namespaced via a single key-builder function (never string concatenation at call sites); Redis ACLs per service role; nothing cached that isn't derivable from the database.

**Audit store.** Shared, append-only, hash-chained (ADR-006); tenant-scoped rows, global chain (the global chain is tamper evidence for all tenants including against the operator).

**Enterprise tier.** "Dedicated schema" or "dedicated database" is a **configuration** of the same codebase (not a fork) sold as an enterprise deployment tier. Never per-customer deployed stacks (version-skew trap for connector hotfixes).

**Rationale.** The execution tier is already isolated per-run (gVisor ephemeral containers, REQ-010); data isolation is a separate concern best enforced at the database boundary. The dominant multi-tenant failure mode is a missing tenant filter in one query; RLS + the CI gate is the defense. Nomination data is trading-position data under tariff confidentiality (e.g., TransColorado GTC §2.5, REQ-038), so a cross-tenant leak can put the **customer** in violation of their tariff obligations.

**Versioning rule.** Tenancy model is frozen; enterprise-tier configurations are additive.

---

## 4. Requirements register

Numbering is sequential with no gaps. Priorities: **v1** (build gate), **v1.1** (first post-v1 increment), **later** (tracked, not scheduled). Test types: unit / contract / E2E / UI / a11y / security / load / drill. Every unit and E2E test names the REQ it covers (REQ↔test traceability, §6).

**REQ-001 — Canonical Nomination model + ingestion API** — *v1*
Source: pack 5.2, b1 C14, ADR-001. The canonical schema of ADR-001 is the only internal nomination representation; `POST /nominations` and `POST /ingest/file` accept it, validate server-side, and honor `idempotency_key`.
Acceptance: (a) schema validation rejects all invariant violations enumerated in ADR-001 (bumpable+ID3, quantity ≤ 0, unknown cycle); (b) resubmission with a duplicate `idempotency_key` returns the original `nomination_id` and starts no second workflow; (c) a file-import fixture of ≥100 rows maps losslessly to canonical records with row-level reject reasons.
Test: unit + contract.

**REQ-002 — Per-pipeline channel discovery at onboarding** — *v1*
Source: b1 §A8/§A9/§B2. Onboarding a pipeline runs `discover_channels` and records a ChannelReport: sanctioned API (ET Messenger+ check mandatory for ET pipelines), EDI offered vs. active/enrollable, flat-file availability, portal availability.
Acceptance: (a) a pipeline cannot be marked live without a stored ChannelReport; (b) the report distinguishes `OFFERED_INACTIVE` from `ACTIVE` EDI; (c) the ET Messenger+ API check executes and records evidence for every ET Messenger pipeline.
Test: unit + contract + E2E (against simulators).

**REQ-003 — Time engine** — *v1*
Source: b1 §A14/§A16, a3 §5. Python `zoneinfo` + pinned PyPI `tzdata` (pytz banned). Every cutoff modeled as `(IANA zone, wall-clock)`; UTC instants stored; deadlines computed fresh per gas day; gas day = 9:00–9:00 CCT with 23/25-hour DST days; all five cycles per §2.1 with per-manifest overrides.
Acceptance: (a) Hypothesis property suite passes: "every gas day is 23–25 h", deadline monotonicity across DST, UTC round-trips; (b) golden tests pass for both 2026 US transition Sundays (2026-03-08 spring-forward gap; 2026-11-01 fall-back `fold=0/1`) with defined behavior for 02:30-local timers; (c) all five cycles' §2.1 table values reproduce from the engine for a fixed set of golden gas days; (d) `timedelta(days=1)` never appears in deadline math (static check).
Test: unit (property + golden).

**REQ-004 — Pipeline manifests** — *v1*
Source: b1 §A14, C6/C7. Per-pipeline declarative manifest: platform, full 1.3.2 cycle-table overrides, `naesb_version_effective`, location-code lists, channel configuration, submission-latency budget, tariff references. Manifests are data, versioned, and reviewed by the domain expert.
Acceptance: (a) a pipeline is disabled end-to-end without a valid manifest; (b) manifest changes are audit events; (c) manifest schema validation rejects unknown cycles/versions.
Test: unit + contract.

**REQ-005 — Connector framework implementing ADR-002** — *v1*
Source: pack 3.4, b1 §B4. Plugin SDK enforcing the six-operation contract, the frozen error taxonomy, `interface_version` negotiation, and digest-pinned packaging.
Acceptance: (a) contract test suite runs against any connector implementing the SDK and fails on taxonomy violations or unknown error classes; (b) orchestrator refuses a connector with an unknown `interface_version`; (c) a sample connector passes end-to-end against both simulators.
Test: contract + E2E.

**REQ-006 — Portal simulator** — *v1*
Source: pack 1.3/4.1#5, b1 §A5/§B3. FastAPI-served HTML clones of target portal flows driven by recorded fixtures (real HTML/DOM — Playwright must drive them); Playwright page objects + `storage_state` for auth; WireMock standalone only for non-browser HTTP dependencies. **mountebank is banned** (maintainer departed May 2024; docs domain hijacked).
Acceptance: (a) simulators for the first three build-order platforms (KM DART, TC eConnects, 1Line) serve full submit→QR→confirm→scheduled flows; (b) fixtures are tamper-evident (hash manifest, REQ-052); (c) the portal connector's E2E suite passes against simulators with zero live-network egress.
Test: E2E.

**REQ-007 — EDI simulator** — *v1 (envelope-level) / v1.1 (IG-level)*
Source: b1 §A6/§B3. pyx12 ≥4.0 as the X12 engine. Day-one fidelity is **envelope-level** (ISA/GS/ST wrapping, 997/999 ack generation); IG-level validation arrives only after the v4.1 purchase funds authoring/validation of the NAESB gas maps G850NMST, G855NMQR, G850RQCF, G855RRFC, G865SQTS, G865SQOP, G855CRQR (pyx12 is HIPAA-first — budget the map authoring).
Acceptance: (a) v1: round-trip a synthetic G850NMST through envelope parse + 997 generation; (b) v1.1: all seven 1.4.x maps validate against fixtures authored from the purchased v4.1 manuals, with rejection tests for malformed segments.
Test: unit + contract.

**REQ-008 — EDI transport endpoint (NAESB IET/QEDM)** — *v1.1*
Source: a2 §2.3, b1 §A15. AS2-capable endpoint implemented against the NAESB Internet Electronic Transport / QEDM manuals — not generic-AS2 assumptions. Mendelson AS2 community edition (GPL) runs as a containerized test peer only, never embedded.
Acceptance: (a) interop suite exchanges 850/855/865 messages with the Mendelson peer over the NAESB-conformant packaging; (b) license scan proves no GPL linkage in shipped artifacts.
Test: E2E + security (license scan).

**REQ-009 — EDI connector** — *v1.1 (v1: behind adapter interface)*
Source: b1 §A9, C9. Production EDI submission/receipt per ADR-002, sitting behind the EDI adapter interface that an aggregator (Trellis Connect / Enercross / Systrends) could also implement (§7 decision).
Acceptance: (a) the aggregator adapter and the native pyx12 adapter pass the same contract suite; (b) full submit→QR→confirmation→scheduled-quantity lifecycle against the EDI simulator.
Test: contract + E2E.

**REQ-010 — Portal connector (browser tier)** — *v1*
Source: b1 §B4, a3 §3. Playwright-driven bots executing in short-lived gVisor (`runsc`) containers: read-only rootfs, dropped capabilities, seccomp, non-root, ephemeral per run. Egress default-deny via allowlisting forward proxy permitting only the target portal FQDN (+ empirically discovered CDN/API hosts); block 169.254.169.254 and all RFC1918.
Acceptance: (a) a container attempting egress to any non-allowlisted host is denied and the attempt is logged; (b) metadata-endpoint and RFC1918 probes from inside the container fail; (c) containers leave no writable persistent state between runs; (d) full lifecycle passes against portal simulators.
Test: E2E + security.

**REQ-011 — Flat-file upload channel** — *v1*
Source: b1 §A9, a1 §2.2. CSV/batch upload as a first-class channel where offered (Transco 1Line, KM DART, TC eConnects verified), preferred over browser-form bots when available.
Acceptance: (a) canonical nominations render to each supported platform's flat-file format from golden fixtures; (b) upload round-trip verified against simulators; (c) channel router prefers flat-file over portal-form automation when both are ACTIVE.
Test: contract + E2E.

**REQ-012 — Sanctioned-API connector path** — *v1.1*
Source: b1 §A8. Where per-pipeline discovery finds a pipeline-sanctioned API (ET Messenger+ first candidate), an API connector implements ADR-002 over that API instead of browser automation.
Acceptance: (a) Messenger+ verification at onboarding produces a recorded yes/no with evidence; (b) if present, the API connector passes the ADR-002 contract suite.
Test: contract + E2E (mocked API).

**REQ-013 — Connector versioning and packaging** — *v1*
Source: b1 §B4, a3 §3. One plugin version = one immutable image digest; old/new versions run side-by-side per pipeline; per-pipeline routing pins the digest.
Acceptance: (a) two digests of the same platform connector serve different pipelines simultaneously; (b) rollback to the previous digest takes effect without redeploy of the control plane; (c) digest allow-list enforced at worker launch.
Test: E2E + security.

**REQ-014 — Connector drift detection** — *v1*
Source: b1 C15, a3 §6. Golden fixtures from recorded (sanitized) real portal responses replay against production portals as canaries; structural divergence raises `DRIFT`, quarantines the connector version, and blocks auto-submit on that pipeline.
Acceptance: (a) a deliberately mutated simulator response triggers `DRIFT` and halts auto-submit within one canary cycle; (b) drift events appear on the pipeline health page and in the audit log; (c) human override of a drift quarantine is itself audited.
Test: E2E + drill.

**REQ-015 — Submission lifecycle as Temporal workflow** — *v1*
Source: ADR-003, pack 3.4, b1 §B4. One workflow per nomination; all timers as durable UTC timers; states/transitions exactly as ADR-003; no second scheduler anywhere in the system (static architecture check).
Acceptance: (a) killing and resuming workers mid-lifecycle loses no timer or state (crash-recovery test at every ADR-003 state); (b) the ADR-003 transition table is exhaustively exercised against simulators; (c) no cron/APScheduler/ celery-beat dependency exists in the dependency tree.
Test: E2E + unit.

**REQ-016 — Fair queuing via Temporal Fairness keys** — *v1*
Source: b1 §A4/§B5, a3 §2. `fairness_key = f"{tenant_id}:{pipeline_id}"` with tenant-tier weights; `matching.enableFairness=true`; thin durable intake (Postgres `FOR UPDATE SKIP LOCKED`) for ingress backpressure before workflow start. **No bespoke fair-queue broker.** Documented caveats accepted: fairness enforced per task-queue partition; per-key rate limits apply at schedule time, not to existing backlog.
Acceptance: (a) load harness replays a cutoff-spike shape (all five cycles' windows) with a noisy-tenant surge and demonstrates no starvation of other tenants' submissions (every submission dispatched before its `T_submit_by`); (b) the harness runs in CI on the ingestion-critical path.
Test: load.

**REQ-017 — Per-connector circuit breakers and bulkheads** — *v1*
Source: pack 3.1 Q2, 3.4. One failing connector degrades no other pipeline; isolation is per-connector AND per-tenant.
Acceptance: (a) a fault-injected connector (100% failure) leaves the other platform simulators' SLOs unmet in no run; (b) breaker open/half-open/closed transitions are observable and audited.
Test: load + E2E.

**REQ-018 — Exactly-once submission controls** — *v1*
Source: pack 3.1 Q4, ADR-003. Dedup by `idempotency_key` at intake, workflow-id reuse, and post-submit reconciliation (poll_status cross-check against pipeline-visible state before any retry that could double-submit).
Acceptance: (a) fault injection at submit-ack boundary never produces two pipeline-visible submissions for one nomination (verified on simulators); (b) reconciliation runs before any ambiguous retry and its outcome is audited.
Test: E2E (fault injection).

**REQ-019 — Verification engine (QR windows, version skew)** — *v1*
Source: b1 §B8, C7. Ingests, timestamps, and evaluates QRs against the 5-minute expectation on v4.1-effective pipelines and the §2.1 tariff column otherwise (per-manifest `naesb_version_effective`); drives the state ladder of §2.5; late/missing QR inside a cycle escalates while renomination is still possible.
Acceptance: (a) golden fixtures for both version regimes produce correct `QR_LATE`/pass verdicts; (b) an escalation fires with time-to-deadline metadata sufficient for renomination; (c) skew test: two pipelines on different versions, same gas day, both evaluated against their own manifest.
Test: unit + E2E.

**REQ-020 — Receipt generation and storage** — *v1*
Source: ADR-004, pack 2.3. Receipts per ADR-004; screenshots/raw responses in the Object Lock COMPLIANCE bucket (enabled at bucket creation — infra checklist) with per-object SHA-256 recorded in the audit row; PDF + JSON export.
Acceptance: (a) receipt p99 availability ≤ 60 s from triggering evidence (REQ-046); (b) every exported artifact verifies against its recorded hash; (c) tampering with an artifact is detected by re-verification; (d) Object Lock retention cannot be shortened by any principal in tests.
Test: E2E + security + performance.

**REQ-021 — Escalation policy builder** — *v1*
Source: pack 2.3. Per-tenant policies: thresholds (time-to-deadline, ladder rung), channels (SMS/voice/email/Slack), on-call rotation binding, quiet hours (with override classes for QR-late and cutoff-at-risk).
Acceptance: (a) policy CRUD via ADR-005; (b) simulation mode shows which policy fires for a synthetic event without paging anyone; (c) quiet hours suppress only the policy classes they declare.
Test: unit + E2E + UI.

**REQ-022 — Notification delivery and paging SLO** — *v1*
Source: pack 1.1, 3.3 Q10. Delivery via Twilio-class SMS/voice/email and Slack; page dispatched within **2 minutes** of escalation trigger (REQ-046).
Acceptance: (a) delivery with per-provider retry and failover ordering; (b) SLO measured and reported per escalation; (c) unacknowledged pages re-escalate per policy.
Test: E2E + performance.

**REQ-023 — On-call rotation and shift-change ownership** — *v1*
Source: pack 2.1 Q4, 2.4. Desks are 24/7; an escalated nomination has exactly one owner at any instant, computed from the tenant's rotation; `GET /on-call/current` resolves it.
Acceptance: (a) rotation editor supports weekly patterns + overrides; (b) an escalation at 02:00 routes to the rotation-derived owner, never to "nobody"; (c) ownership changes are audit events.
Test: unit + E2E.

**REQ-024 — Dead man's switch (own service)** — *v1*
Source: pack 6.1. External third-party heartbeat watcher independent of our infrastructure; customer-visible status page; rehearsed manual-fallback runbook ("submit manually, like today") with customer notification within minutes of outage detection. v1 of the fallback is the documented runbook, not software.
Acceptance: (a) heartbeat monitor detects a killed service and flips the status page without our infrastructure participating; (b) the manual-fallback runbook drill is executed and timed; (c) customers are notified within the runbook's stated window.
Test: drill.

**REQ-025 — Kill switches (customer + per-pipeline)** — *v1*
Source: pack 2.3, b1 C2. Customer-level one-click "stop all automation, humans only" AND per-pipeline automation toggles; engagement halts in-flight workflows to `KILLED` with no further pipeline contact; tested weekly.
Acceptance: (a) one click from the admin console halts all tenant automation (measured in the weekly drill); (b) per-pipeline toggle halts only that pipeline; (c) both are audit events; (d) re-enable requires explicit confirmation and is audited.
Test: E2E + drill (weekly automated).

**REQ-026 — C&D protocol (no circumvention)** — *v1*
Source: b1 §A12, C2; a4 §4. Immediate-honor cease-and-desist runbook: on any pipeline objection (C&D, block, ESA notice), the per-pipeline kill state engages immediately; **no circumvention of technical blocks ever** (DMCA §1201 avoidance); Power Ventures line — continued access after objection is where CFAA risk crystallizes.
Acceptance: (a) runbook documented and drilled; (b) `BLOCKED` error class engagement is automatic and irreversible without named-human approval; (c) static egress IPs make automation attributable and reversible (REQ-054).
Test: drill + E2E.

**REQ-027 — Shadow mode + match-rate dashboard** — *v1*
Source: pack 2.3, 2.2 Q7. First-class feature, not a testing phase: humans still submit; we run the full pipeline in parallel and compare; per-pipeline match-rate dashboard ("matched your human 100% for 30 days"); match rate is the early-warning churn metric.
Acceptance: (a) shadow runs produce comparison records without any pipeline submission; (b) dashboard shows per-pipeline match rate over rolling windows; (c) a forced mismatch renders correctly with both quantities.
Test: E2E + UI.

**REQ-028 — Pipeline health page** — *v1*
Source: pack 2.3, 3.3 Q12. Per-pipeline status, last successful canary, detected portal changes (drift), EDI endpoint status; silent-breakage detection via canaries + REQ-014 fixtures.
Acceptance: (a) health page reflects a fault-injected canary failure within one canary interval; (b) every covered pipeline has a scheduled canary; (c) health surfaces use only non-confidential data (REQ-038).
Test: E2E + UI.

**REQ-029 — Agency-onboarding workflow** — *v1*
Source: b1 C1, §A12; a4 §1A/Imp.2. Product workflow that generates/files each pipeline's written agent-designation notice and agency consent forms (ANR GTC 6.6.2(g), National Fuel upstream agency consent form, etc.), tracks execution status of DART ESA / TC Energy ECA / Enbridge access agreements (full texts collected at onboarding), and stores the shipper's indemnity acknowledgment. Agency is tariff-recognized standard practice — this is a checklist workflow, not a legal blocker.
Acceptance: (a) a pipeline cannot go live for a tenant without completed agency paperwork state; (b) ESA/ECA status transitions are tracked and audited; (c) the indemnity acknowledgment is stored as an Object-Locked artifact.
Test: E2E + UI.

**REQ-030 — Customer onboarding journey** — *v1*
Source: pack 2.2 Q6, 2.4. Defined funnel: credential collection → agency paperwork → channel discovery → shadow mode → live; every step has state, owner, and age.
Acceptance: (a) onboarding dashboard shows per-customer stage and days-in-stage; (b) credential handover path never exposes values in UI (status only); (c) stage transitions are audit events.
Test: E2E + UI.

**REQ-031 — Credential vault (named-user model)** — *v1*
Source: b1 C10, a4 §1B/Imp.4. Secrets vault with per-tenant isolation; models **per-user IDs** (not just shared logins) because some EBBs issue individual user IDs with subscriber liability (KM DART §34.8–34.9); supports pipeline External Security Administrator workflows; static per-customer egress identity (REQ-054).
Acceptance: (a) credential values are never readable via any API (status-only, ADR-005); (b) vault access is per-action audited (REQ-032); (c) per-user and shared credential models both representable; (d) tenant A cannot reference tenant B's credential under any role (isolation test).
Test: security + unit.

**REQ-032 — Per-action credential/portal audit logging** — *v1*
Source: b1 C10. Every portal action is logged per credential (actor, credential_ref, action, artifact refs) into ADR-006.
Acceptance: (a) a full portal submission replay produces a complete per-credential action trail; (b) trail rows hash-chain verify.
Test: E2E + security.

**REQ-033 — Admin console** — *v1*
Source: pack 2.3. Users, roles (RBAC), credential status (never values), audit export, kill switches, pipeline toggles.
Acceptance: (a) RBAC denies cross-role admin actions (matrix tested); (b) audit export requires admin role and is itself an audit event; (c) credential endpoints return status fields only (schema test).
Test: E2E + UI + security.

**REQ-034 — Desk-manager view** — *v1*
Source: pack 2.1 Q1, 2.4. Desk-level "did today go okay": per-gas-day rollup of ladder states across pipelines/cycles, open escalations, upcoming deadlines.
Acceptance: (a) view renders the five-cycle day with counts per ladder rung; (b) manager role sees desk view without connector internals; (c) loads within the dashboard SLO (REQ-046).
Test: UI + E2E.

**REQ-035 — Multi-tenancy isolation and encryption** — *v1*
Source: pack 3.2 Q6. Shared application, siloed per-tenant credentials/keys; tenant-scoped DB sessions (RLS context) via FastAPI dependency injection; encryption at rest with per-tenant data keys.
Acceptance: (a) cross-tenant read attempts fail at RLS for every table (generated test matrix); (b) per-tenant key rotation does not expose plaintext; (c) tenant context cannot be supplied by the client (negative tests).
Test: security + unit.

**REQ-036 — Audit store (ADR-006)** — *v1*
Source: b1 §A7/§B7, a3 §4. Append-only Postgres `audit_events`, trigger-immutable, single-writer service, SHA-256 chain, nightly re-walk, daily chain-head anchor to S3 Object Lock COMPLIANCE. **No QLDB. No mountebank anywhere in the stack.**
Acceptance: (a) UPDATE/DELETE on `audit_events` fails for every role including superuser paths tested; (b) tampering with a historical row is caught by the nightly re-walk; (c) daily anchor object written and verifiable; (d) dependency scan proves no QLDB client anywhere.
Test: security + unit.

**REQ-037 — Audit retention and legal hold** — *v1.1*
Source: pack 3.2 Q8. Retention policy per data class; legal-hold flag suspending deletion for held tenants/matters; Object Lock retention budgeting (irreversible) documented.
Acceptance: (a) held records survive retention sweeps; (b) retention/hold actions are audit events; (c) retention budget document reviewed at infra sign-off.
Test: unit + security.

**REQ-038 — Data segregation; public-only cross-customer surfaces** — *v1*
Source: pack 6.4, b1 C11, a4 Imp.5. No cross-tenant analytics on position data; no model training on customer data without written consent; data-use terms in the MSA; access controls audited. Only **public-by-regulation** postings (OFOs, critical notices, capacity) may feed cross-customer aggregated surfaces (pipeline health page OK; anything derived from nomination data is not). Tariff redisclosure restrictions (e.g., TransColorado §2.5) and FERC Standards of Conduct (18 CFR Part 358) honored.
Acceptance: (a) static data-lineage check: no cross-tenant query plan touches nomination tables; (b) aggregation tables provably source from public-postings pipelines only; (c) policy text in MSA (§7 legal).
Test: security + unit (data-lineage tests).

**REQ-039 — Storm mode: OFO/critical-notice ingestion** — *v1*
Source: b1 C5, a2 §2.4. Real-time OFO/critical-notice parsing across all covered EBBs from public Informational Postings sites **plus** the sanctioned email/EDI notification channel (18 CFR §284.12(b)(3)(vi); NAESB WGQ Stds 0.3.14, 5.3.34, 5.3.35) — not scraping alone. Notices peak at night/holidays (Elliott critical notices peaked Christmas Eve).
Acceptance: (a) parser handles the standardized notice formats of the covered platforms from fixtures; (b) email/EDI-channel intake path exists and is tested; (c) a notice ingested at 02:00 on a holiday produces the same artifacts and alerts as a business-hours notice.
Test: E2E + unit.

**REQ-040 — Per-pipeline per-cycle cutoff countdowns** — *v1*
Source: b1 C5, §A14. Live countdowns to all five cycles' leave-control and QR/confirmation deadlines per pipeline, driven by the time engine and manifest overrides; SSE-delivered (htmx SSE extension).
Acceptance: (a) countdowns render for all five cycles and update live without reload; (b) golden-test alignment: displayed deadlines equal time-engine outputs for the 2026 transition Sundays; (c) accessible per REQ-045 (never color-alone).
Test: UI + E2E + a11y.

**REQ-041 — Penalty-exposure calculator** — *v1.1*
Source: b1 C5, a4 §5. Given an active OFO and the shipper's position: exposure = tariff multiple × index, per the tariff's penalty clause; reference case: Spire at $1,556.96/MMBtu = 617× monthly index.
Acceptance: (a) calculator consumes parsed OFO + manifest penalty terms + index input and produces a per-cycle exposure figure; (b) reference fixture reproduces a known multiple×index computation; (c) output labeled as estimate with tariff-clause citation.
Test: unit + E2E.

**REQ-042 — Auto-renom (EPSQ-aware)** — *v1.1*
Source: b1 C5, a4 Imp.9. Automatic re-nomination on cut/OFO with elapsed-prorated-scheduled-quantity awareness; policy-gated per tenant; every auto-renom produces the full receipt/audit trail.
Acceptance: (a) given a cut confirmation, the renom quantity respects EPSQ rules from the manifest; (b) tenant policy can disable per pipeline; (c) auto-renoms are indistinguishable from manual in audit completeness.
Test: E2E.

**REQ-043 — eTariff compliance-map ingestion** — *v1*
Source: b1 C6, a2 §2.4. Scrape each pipeline's Order 587-V tariff section (every NAESB standard, incorporation status, waivers) from FERC eTariff/eLibrary (public display is PDF post-Nov-2024 eTariff 2.0; no official eLibrary API); combine with the NAESB URL directory; re-verify platform attribution per pipeline at build time (M&A drift — e.g., DTM pipes migrated to Trellis). This is the seed of the dialect registry and feeds REQ-004/REQ-019.
Acceptance: (a) for every registered pipeline, a stored compliance map with retrieval timestamp; (b) change detection diffs successive retrievals and alerts; (c) manifest `naesb_version_effective` is provably sourced from the map.
Test: E2E + unit.

**REQ-044 — ETRM integration surface** — *v1*
Source: b1 C14, a4 §2. v1 stays file/API import (`POST /ingest/file`, `POST /nominations`); the canonical model anticipates named ETRM mappings (Endur, Allegro, FIS, PCI — the NatGasHub competitive bar) via reserved `etrm.*` metadata.
Acceptance: (a) import mapping templates per ETRM export format are versioned fixtures; (b) round-trip import→nomination→receipt preserves `deal_id` and `etrm.*` fields.
Test: contract + E2E.

**REQ-045 — Accessibility: WCAG 2.2 AA** — *v1*
Source: pack 4.4, a3 §7. Status never carried by color alone (dots paired with text); full keyboard navigation of nomination tables; visible focus management; form labels on every input; contrast compliance in receipt and escalation views (the screens read under stress); WCAG 2.2 additions: 2.5.8 target size 24×24, 2.5.7 dragging alternatives, 3.3.8 accessible authentication.
Acceptance: (a) `@axe-core/playwright` with tags `wcag2a/wcag2aa/wcag22aa` gates CI on serious/critical for every key screen; (b) nightly full scan; (c) manual keyboard/screen-reader pass per release (axe covers only ~57% of issues by volume).
Test: a11y.

**REQ-046 — SLOs** — *v1*
Source: pack 3.3 Q10. (a) Receipt availability ≤ **60 s p99** from triggering pipeline evidence. (b) Escalation page dispatched ≤ **2 min** from trigger. (c) Dashboard (scheduler + desk-manager) load p95 ≤ 2 s under reference load *(spec-set target; no external source — flagged in §12)*. All SLOs instrumented with per-request trace context.
Acceptance: (a)+(b)+(c) measured in the load harness and reported per release; regression = gate failure.
Test: load + performance.

**REQ-047 — Storage plan at 1,000 nominations/day/tenant** — *v1*
Source: pack 3.3 Q11. Receipts + screenshots + raw responses retained per tenant in Object Lock; transactional state in Postgres. Per-nomination artifact budget ≤ 5 MB compressed *(spec-set budget)*; measured actuals reported monthly; lifecycle/tiering policy per data class; cost curve documented.
Acceptance: (a) synthetic 1,000-nom/day/tenant soak for 30 gas days stays within budget and within the dashboard SLO; (b) artifact-size distribution report produced; (c) lifecycle rules execute on schedule.
Test: load + soak.

**REQ-048 — Disaster recovery** — *v1*
Source: pack 6.9. RPO 5 minutes (continuous replication of transactional state), RTO 60 minutes; receipts and audit store in a separate failure domain from compute (object storage with versioning + Object Lock).
Acceptance: (a) DR drill restores service ≤ 60 min with ≤ 5 min state loss (measured); (b) receipts+audit remain readable while compute is down; (c) drill results recorded.
Test: drill.

**REQ-049 — Release management and rollback** — *v1*
Source: pack 6.8, b1 C15. Feature flags per pipeline AND per customer; staged connector rollout (shadow percent of a customer's traffic first); instant rollback (REQ-013); **no connector deploys after noon CT on business days** — enforced in CI/CD, covering all five cycle deadlines.
Acceptance: (a) deploy pipeline hard-blocks connector releases after 12:00 CT Mon–Fri (tested with clock fixtures); (b) staged rollout at configurable shadow percentage; (c) one-command rollback drill passes.
Test: E2E + drill.

**REQ-050 — Temporal operations and determinism discipline** — *v1*
Source: b1 C13, a3 §1. Self-hosted Temporal OSS (MIT) with Postgres persistence (skip Elasticsearch initially); determinism budget (activities vs. workflow code; idempotent side effects) enforced by replay tests; Hatchet (MIT, Postgres-only, native tenant fairness) documented as the fallback if operational burden dominates; Restate is BSL — flag for legal if ever evaluated. Temporal ops runbook + capacity budget maintained.
Acceptance: (a) workflow replay tests in CI catch nondeterminism; (b) Temporal cluster backup/restore drill passes; (c) ops runbook exists and is reviewed monthly.
Test: unit (replay) + drill.

**REQ-051 — Observability: per-nomination trace ID + trace viewer** — *v1*
Source: pack 3.4, b1 §B6. One distributed trace ID from queue to receipt (ADR-001 `trace_id`); one trace shows the entire life of one nomination — the debugging backbone and the customer-support interface. Trace viewer is a UI surface within the htmx complexity budget (§5).
Acceptance: (a) every audit event, receipt event, and connector activity for a nomination joins on `trace_id`; (b) trace viewer renders queue→receipt for a golden nomination; (c) support role can query by nomination and see the full trace.
Test: E2E + UI.

**REQ-052 — Threat model and security scanning (DevSecOps)** — *v1*
Source: pack 3.2 Q9, 4.1 #4, b1 §B4. STRIDE-lite threat model covering stolen credentials, malicious tenant, compromised plugin, insider (owner: DevSecOps); CI gates: secret scanning, static analysis, dependency scanning, container scanning; simulator fixtures tamper-evident (hash manifests verified in CI); sandbox egress rules (REQ-010) tested as security tests.
Acceptance: (a) threat model document versioned with the code; (b) each scanner is a merge gate with a failing-fixture proof; (c) fixture tampering is detected by CI.
Test: security.

**REQ-053 — SOC 2 evidence generation** — *v1 (design) / later (Type II report)*
Source: b1 C4. The audit store, access controls, and change management are designed as SOC 2 evidence generators; observation period starts immediately (§7).
Acceptance: (a) control-to-evidence mapping document exists; (b) every REQ-052/036 control emits auditor-consumable evidence artifacts; (c) change-management trail complete for any sampled release.
Test: security (evidence completeness check).

**REQ-054 — Static egress IPs per customer** — *v1*
Source: pack 6.5, b1 §A10/§E2. Automation egress is attributable and reversible per customer; static IP allocation recorded per tenant; supports C&D response and pipeline conversations. (Note: no pipeline vendor programs are known to exist — relationship strategy is NAESB WGQ participation + aggregator partnership + direct conversation.)
Acceptance: (a) portal-connector egress for tenant A originates only from tenant A's allocated IPs; (b) IP allocation table audited; (c) failover behavior documented and tested.
Test: security + E2E.

**REQ-055 — NAESB certification readiness** — *later*
Source: b1 C3, a4 §6. Official NAESB WGQ software certification program (self-certification under oath, 2-year validity); the Jan 2026 certified-products list is the competitor set (NatGasHub/Enercross 4.0, Quorum, Emerson, OATI, Vector, Keywest). EDI layer designed to be certifiable; simulator conformance evidence doubles as certification prep; certification milestone post-v1.
Acceptance: (a) EDI simulator conformance reports are certification-format-ready; (b) certification target date set in §7 table post-v1.
Test: contract (conformance evidence).

**REQ-056 — 24/7 operations commitment** — *v1*
Source: pack 6.2. On-call for the service itself from first live customer; escalation runbooks; war-room procedure for weather events; founders carry initially, budgeted honestly as the real operating cost.
Acceptance: (a) on-call schedule staffed before first live customer; (b) war-room runbook drilled before storm season; (c) service on-call pages route per REQ-021/022.
Test: drill.

**REQ-057 — Shadow-mode match definition** — *v1*
Source: v1.1 frozen-spec review adjudication; REQ-027, pack 2.2 Q7/2.3. In shadow mode, the system reads the human's actual submitted nominations from the EBB (we hold the credentials) and diffs them against what the system would have submitted. **Match =** all of (pipeline, cycle, gas day, receipt/delivery points, service requester, contract) equal **AND** quantity within 0.5% tolerance. Match rate computed per pipeline per cycle, rolling 30 days.
Acceptance: (a) a scripted set of human submissions containing single-field mismatches produces non-match verdicts naming the divergent field; (b) a 0.4% quantity divergence matches and a 0.6% divergence does not; (c) the dashboard series reproduces the rolling-30-day per-pipeline per-cycle computation from fixtures.
Test: E2E against portal mock with scripted human submissions.

**REQ-058 — Shadow-to-live promotion criteria** — *v1*
Source: v1.1 frozen-spec review adjudication; REQ-027. A pipeline+customer pair exits shadow mode only after 30 consecutive gas days at 100% Timely-cycle match rate **and** explicit human sign-off recorded as an audit event; demotion back to shadow on any live mismatch.
Acceptance: (a) promotion is blocked at 29 days or at any sub-100% Timely-cycle match rate; (b) sign-off without the 30-day record (or the record without sign-off) does not promote; (c) a live mismatch demotes automatically and the demotion is an audit event and an escalation.
Test: unit + E2E.

**REQ-059 — Amend-after-submit** — *v1*
Source: v1.1 frozen-spec review adjudication; ADR-003/ADR-005. A nomination already accepted by the pipeline can be amended (quantity/points) before the cycle deadline per NAESB rules; the state machine gains an `AMENDED` path (ADR-003) and ADR-005 gains `POST /nominations/{id}/amend`. Amendments after deadline rejected with the deadline in the error.
Acceptance: (a) a pre-deadline amendment round-trips through `AMENDED` back onto the verification ladder with a fresh QR evaluation; (b) a post-deadline amendment is rejected with an RFC-9457 error carrying the cycle deadline; (c) amendments produce the full receipt/audit trail indistinguishable in completeness from original submissions.
Test: contract + E2E on both simulators.

**REQ-060 — Cut handling** — *v1*
Source: v1.1 frozen-spec review adjudication; §2.5 rung 4, ADR-003. When scheduled quantity returns below nominated quantity, the workflow enters a `CUT` state recording nominated vs scheduled deltas, notifies per escalation policy, and offers next-cycle renomination (EPSQ-aware).
Acceptance: (a) a scheduled < nominated fixture drives the workflow through `CUT` with both quantities recorded on the receipt; (b) the notification fires per the tenant's escalation policy with time-to-next-deadline metadata; (c) the offered renomination quantity respects manifest EPSQ rules.
Test: contract + E2E.

**REQ-061 — Server clock discipline** — *v1*
Source: v1.1 frozen-spec review adjudication; REQ-003, b1 §A14. All deadline-bearing hosts run chrony/NTP with >=3 sources; clock skew monitored; if skew exceeds 500 ms the system alerts per escalation policy and blocks auto-submission until resolved (manual override audited).
Acceptance: (a) injected skew >500 ms blocks auto-submit and pages per policy; (b) the block clears on skew resolution without restart; (c) manual override during a skew block is an audit event; (d) the ops runbook documents source configuration, monitoring, and recovery.
Test: unit (injected skew) + ops runbook.

**REQ-062 — Escalation acknowledgement loop** — *v1*
Source: v1.1 frozen-spec review adjudication; REQ-021/022/023. Pages require acknowledgement; unacknowledged after N minutes (policy-configurable, default 5) escalates to the next on-call target; the full ack/escalation chain is audit-logged.
Acceptance: (a) an acknowledged page stops the chain; (b) an unacknowledged page escalates to the next rotation target at N minutes with the chain intact in the audit log; (c) N is policy-configurable per tenant with default 5.
Test: E2E with fake notifier.

**REQ-063 — Pipeline outage and maintenance handling** — *v1*
Source: v1.1 frozen-spec review adjudication; ADR-003, REQ-028, b1 §B2. Detect portal/EDI down states (health canaries + submit failures); per channel doctrine, attempt fallback channel when sanctioned; ingest pipeline-announced deadline extensions from notices; **NEVER report "submitted" for nominations sitting in a retry queue while the pipeline was unreachable** — UI shows `PIPELINE_DOWN` state distinctly (ADR-003).
Acceptance: (a) induced simulator downtime flips affected nominations to `PIPELINE_DOWN` (never "Submitted") and attempts the sanctioned fallback channel; (b) a notice-announced deadline extension re-computes the workflow timers; (c) recovery resumes submission and the outage interval is on the receipt.
Test: E2E on simulators with induced downtime.

**REQ-064 — Authentication: local accounts + TOTP MFA (v1); SSO (v1.1)** — *v1 (MFA) / v1.1 (SSO)*
Source: v1.1 frozen-spec review adjudication; ADR-005. v1 ships local accounts + TOTP MFA; SSO (SAML/OIDC) is v1.1 and **required before enterprise MSAs**. Session/JWT per ADR-005.
Acceptance: (a) login without a second factor is impossible for MFA-enrolled accounts; (b) recovery-code flow is rate-limited and audited; (c) v1.1: SAML and OIDC login round-trips against a test IdP with role mapping.
Test: security scan + E2E auth flows.

**REQ-065 — Support access model** — *v1*
Source: v1.1 frozen-spec review adjudication; REQ-033/038, b1 C10. Read-only support role scoped per tenant with time-boxed grants; write/impersonation access is break-glass only, dual-approved, fully audit-logged, and auto-expiring. Support views of position data governed by the data-use policy (no bulk export).
Acceptance: (a) support role reads only the granted tenant, only for the grant window; (b) break-glass requires two distinct approvers and expires automatically; (c) every support-session action hash-chain verifies; (d) bulk export is denied to all support roles (negative test).
Test: RBAC E2E + audit verification.

---

## 5. Architecture (v1)

### 5.1 Component diagram

```
                         ┌─────────────────────────────────────────────┐
                         │              TENANT BROWSERS                │
                         └───────────────────┬─────────────────────────┘
                                             │ HTTPS (htmx partials + JSON)
┌───────────────────────── CONTROL PLANE ────┴─────────────────────────┐
│  FastAPI app (ADR-005): REST /api/v1 + Jinja2 + htmx 2.0.x partials  │
│  + Alpine.js islands + htmx SSE extension │ tenant-scoped sessions   │
│  (RLS context) │ NO pipeline contact │ NO business state machines    │
│  │ admin console │ desk view │ receipts │ escalation policies │      │
│  │ kill switches │ onboarding/agency workflows                       │
└───┬──────────────────┬───────────────────────────┬──────────────────┘
    │ reads/writes     │ starts/signals workflows  │ writes audit events
    ▼                  ▼                           ▼
┌─────────────┐  ┌────────────────────────── WORKFLOW PLANE ───────────────┐
│  Postgres   │  │  Temporal Server (self-hosted OSS, MIT; Postgres       │
│  (SQLAlch.  │  │  persistence; no Elasticsearch initially)              │
│  2.0 async, │  │  • one workflow per nomination (ADR-003)               │
│  RLS)       │  │  • Task Queue Fairness: fairness_key = tenant:pipeline │
│             │  │  • thin durable intake: Postgres FOR UPDATE SKIP       │
│  audit_     │  │    LOCKED (ingress backpressure only)                  │
│  events     │  │  • ALL timers/retries/escalations here — no second     │
│  (ADR-006,  │  │    scheduler anywhere                                  │
│  single-    │  └───────┬───────────────────────────────┬────────────────┘
│  writer,    │          │ activities                    │ activities
│  hash chain)│          ▼                               ▼
└─────────────┘  ┌──────────── CONNECTOR WORKER TIER ────────────────────┐
                 │  Portal workers: Playwright bots in short-lived       │
                 │  gVisor (runsc) containers — read-only rootfs,        │
                 │  dropped caps, seccomp, non-root, ephemeral per run   │
                 │  1 plugin version = 1 immutable digest, side-by-side  │
                 │  EDI workers: pyx12 ≥4.0 + authored NAESB maps        │
                 │  Flat-file workers │ API workers (Messenger+ path)    │
                 │  Egress: default-deny allowlisting forward proxy →    │
                 │  target portal FQDN only; block 169.254.169.254 +     │
                 │  RFC1918; static egress IPs per customer (REQ-054)    │
                 │  Circuit breakers per connector (REQ-017)             │
                 └───────┬───────────────────────────────┬────────────────┘
                         │ artifacts (screenshots, raw   │ X12 over NAESB
                         │ responses, exports)           │ IET/QEDM transport
                         ▼                               ▼
┌──────────── DATA PLANE ─────────────────────┐  ┌──── SIMULATORS (dev/CI) ────┐
│ S3 Object Lock bucket (COMPLIANCE mode,     │  │ Portal sims: FastAPI HTML  │
│ enabled at bucket creation): receipts,      │  │ clones + recorded fixtures │
│ screenshots, raw responses, export bundles; │  │ + Playwright page objects  │
│ per-object SHA-256 → audit row              │  │ EDI sim: pyx12 + maps      │
│ Postgres: nominations, manifests, policies, │  │ AS2 peer: Mendelson (GPL,  │
│ users, onboarding state                     │  │ containerized, never       │
│ Nightly: audit chain re-walk → daily chain- │  │ embedded)                  │
│ head anchor object in Object Lock           │  │ Fixtures: tamper-evident   │
│ DR: receipts+audit = separate failure       │  │ (hash manifests)           │
│ domain from compute                         │  └────────────────────────────┘
└─────────────────────────────────────────────┘
Observability: per-nomination trace_id from queue to receipt across every
component (ADR-001/REQ-051); SLO instrumentation (REQ-046); canaries + drift
detection feeding the pipeline health page (REQ-028).
External: third-party heartbeat watcher + status page (dead man's switch,
REQ-024); notification providers SMS/voice/email/Slack (REQ-022).
```

### 5.2 Plane responsibilities

- **Control plane (FastAPI).** REST + htmx partials only; tenant-scoped sessions; **control/API plane only** (b1 §B4): it owns no connectors, no EDI, no scraping, no scheduling. UI rule (pack 5.2): the UI never decides — client-side validation/filtering/sorting allowed; money, deadlines, and state transitions are server-side.
- **Workflow plane (Temporal).** Owns ALL orchestration and scheduling. Fairness keys replace the deleted bespoke broker (b1 §B5). Determinism discipline per REQ-050.
- **Connector worker tier.** Untrusted plugin code in gVisor-sandboxed, per-run-ephemeral containers (a3 §3: plain Docker insufficient — runc escapes CVE-2024-21626, CVE-2025-31133/52565/52881). Kata if on K8s. One plugin version = one immutable image digest.
- **Data plane.** Postgres for transactional state; Object Lock for artifacts and audit anchoring (composition per b1 §B7, not an either/or).
- **Simulators.** First-class infrastructure built before any feature agent (§6). Real credentials touch only the designated integration environment, never dev or CI (pack 5.1).

### 5.3 Version pins appendix (frozen baselines; CI-enforced)

| Component | Pin | Source note |
|---|---|---|
| Python | 3.12+ | a3 §5 |
| FastAPI | 0.13x+ (baseline 0.141.x, Jul 2026) | a3 §8 |
| Starlette | 1.0.x (1.0.0 Mar 2026) | a3 §8 |
| Pydantic | 2.x, ≥2.9, **minor pinned in CI** | a3 §8 |
| SQLAlchemy | 2.0 async | a3 §8 |
| Temporal server + Python SDK | current OSS GA incl. Task Queue Fairness (GA ~Apr 2026); `matching.enableFairness=true` | a3 §1–2 |
| htmx | **2.0.x pinned** (2.x supported indefinitely; 4.0.0 shipped Aug 28, 2026 but stays off npm `latest` until early 2027; plan migration via `npx htmx.org@4.0.0 upgrade-check`) | a3 §7, b1 §A13 |
| Alpine.js | current 3.x | a3 §7 |
| htmx SSE extension | matching 2.x | a3 §7 |
| **hyperscript** | **BANNED** (separate pre-1.0 project, 0.9.x) | b1 §A13 |
| pyx12 | ≥4.0 (v4.0.0 May 2026; HIPAA-first — map authoring budgeted) | a2 §2.5, b1 §A6 |
| Playwright + @axe-core/playwright | current (axe ≥4.10, tags wcag2a/wcag2aa/wcag22aa) | a3 §7 |
| tzdata (PyPI) | explicitly pinned; re-run transition tests on update | a3 §5 |
| Jinja2 | current 3.x | pack 5.3 |
| gVisor (runsc) | current release | a3 §3 |
| **mountebank** | **BANNED** (stalled; docs domain hijacked) | b1 §A5 |
| **QLDB** | **BANNED** (EOS 2025-07-31) | b1 §A7 |
| Mendelson AS2 | community b67+ — containerized test peer only, **GPL: never embed** | b1 §A15 |

### 5.4 Threat model summary (STRIDE-lite — owner: DevSecOps, REQ-052)

| Threat | Primary controls |
|---|---|
| Stolen credentials | Vault (REQ-031), per-action audit (REQ-032), named-user model, AUTH error class with no auto-retry, kill switches |
| Malicious tenant | RLS isolation (REQ-035), per-tenant keys, fairness keys vs. surge (REQ-016), data-segregation lineage checks (REQ-038) |
| Compromised connector plugin | gVisor sandbox, read-only rootfs, dropped caps, seccomp, non-root, ephemeral runs, default-deny egress + FQDN allowlist, digest pinning, contract-test gates, drift quarantine |
| Insider | Single-writer audit chain + Object Lock anchoring (operator-independent tamper evidence), RBAC, audit-export auditing, least-privilege roles |
| Platform objection (legal) | Per-pipeline toggles, C&D runbook, no circumvention of technical blocks (DMCA §1201), static egress IPs (attributable/reversible) |

### 5.5 Multi-tenancy and frontend complexity budget

Shared application; siloed per-tenant credentials/keys; per-tenant data encryption keys; tenant context injected server-side only (REQ-035). Frontend: FastAPI + Jinja2 + htmx 2.0.x + Alpine.js + htmx SSE extension (b1 §B6). **Complexity budget (frozen):** the escalation policy builder (REQ-021) and the trace viewer (REQ-051) are the two richest surfaces; if either requires client state beyond Alpine's comfort zone, build **those islands** as lightweight components (or escalate to an SPA reconsideration via ADR) rather than drifting mid-build. The decision must not be made by default.

---

## 6. Agent build pipeline (Kimi Code execution)

Corrected flow (pack Part 4.1–4.2 as amended by b1). Order is mandatory:

```
spec v1.2 frozen (this document): interfaces (§3), REQ IDs (§4), acceptance criteria
        │
        ▼
INFRA AGENT FIRST: pipeline simulators (portal HTML clones + EDI sim envelope-level),
fixture library (tamper-evident), CI skeleton        ← before ANY feature agent
        │
        ▼
INFRA AGENT VALIDATION GATE (blocking): Temporal Fairness spike-load validation —
fairness-key behavior (fairness_key = tenant:pipeline, tenant-tier weights) validated
under a simulated FIVE-CYCLE cutoff spike, exercising the per-partition and
schedule-time caveats (REQ-016, §8 risk 5)         ← BEFORE parallel dev agents spawn
        │
        ▼
┌─ parallel dev agents (against simulators ONLY, against frozen interfaces) ─┐
│ connector framework │ core/state machine (Temporal) │ receipt/verification │
│ web UI │ notification/escalation │ audit store │ onboarding/agency │      │
└── each agent: unit tests + contract tests against the frozen ADRs ─────────┘
        │
        ▼
CI GATES (merge-blocking, in this order): unit → contract → E2E → UI →
WCAG 2.2 AA (axe serious/critical = fail) → security scans (secret, static,
dependency, container, license)
        │
        ▼
merge → DevSecOps deploys to the container-based dev environment
(Docker Desktop/Colima + compose; every deploy from a locked image,
reproducible from a clean machine — pack 4.3)
        │
        ▼
QA agent: requirement coverage closure against the RUNNING system
        │
        ▼
shadow mode with pilot customers (humans still submit; we match — REQ-027)
```

**Rules:**
1. **E2E/UI/a11y are merge gates, not post-merge steps** (pack 4.1 #1). QA validates reality; gates protect main.
2. **Interfaces frozen before parallel agents** (pack 4.1 #2): §3 ADRs are the contract; agents build against mocks of each other's components; contract tests enforce at every merge.
3. **REQ↔test traceability** (pack 4.1 #3): every test names its REQ IDs; QA's job is coverage closure — which REQ has no passing test, which test covers no REQ. Unverifiable requirements do not exist.
4. **DevSecOps scope** (pack 4.1 #4): secret scanning, static analysis, dependency scanning, container scanning, license scanning (GPL/Mendelson, BSL/Restate), the threat model, and simulator-fixture tamper-evidence — plus merge/deploy.
5. **Simulators before feature agents** (pack 4.1 #5, b1 §B3): day-one simulators = portal HTML clones + cycle-timing engine (fully specified in public tariffs); day-one EDI tests are envelope-level only; IG-level EDI simulation unlocks at the v4.1 purchase (§7).
5a. **Temporal Fairness spike-load validation is an infra-agent gate, not a parallel-dev finding**: the infra agent must validate fairness-key behavior under a simulated five-cycle cutoff spike — including the per-partition and schedule-time caveats of REQ-016 — **before** parallel dev agents spawn, because the ingestion design (thin durable intake, workflow-start shape) depends on the result (§8 risk 5).
6. **Contract-test drift gates** (b1 C15): golden fixtures wire into connector-fleet health and into the noon-CT release rule (REQ-014/REQ-049).

**Connector build order — by platform, not pipeline** (b1 C8; re-asks the PM's "which five pipelines" as "which five platforms"; validate against the first-20-customer census, §8 risk 4):
**KM DART → TC eConnects → Williams 1Line → Enbridge LINK → ET Messenger (+Messenger+ API check) → BHE GTS/Dekaflow → Quorum → Trellis ptms → gasnom.** Custom one-offs (Northern Natural, Equitrans/MVP Salesforce portal, Gulf South, REX) are individual builds, scheduled only after the platform fleet.

**Definition of done (build phase)** — pack 6.10 + b1 C2:
1. Spec v1.2 frozen with REQ IDs (this document).
2. Simulators pass the full fixture suite.
3. CI gates green on main: unit, contract, E2E, UI, WCAG 2.2 AA, security scans.
4. QA coverage-closure report: every v1 REQ has a passing test; every test maps to a REQ.
5. One scripted demo runs clean: a synthetic nomination flows end-to-end through the EDI mock and the portal mock with receipts; a forced failure triggers the dead man's switch; **a per-pipeline C&D drill** (toggle engages, automation halts, audit complete); **a one-click customer kill-switch drill**.
Ship the build when that demo runs clean — not before.

---

## 7. Non-code critical path (owners + lead times)

Four dependencies run in parallel before first live submission (pack 6.7): legal review of pipeline terms (weeks), MSA + insurance (weeks), EDI enrollment where applicable (**weeks-to-months per pipeline, serial worst case** — b1 §A11), and pilot customer credentials (the customer's own internal process). This table is maintained as a one-page plan, reviewed weekly.

| # | Item | Owner | Lead time / deadline | Budget / numbers (sourced) |
|---|---|---|---|---|
| 1 | **NAESB 3-day Locklizard evaluation waiver** (WGQ v4.1; one waiver per company; naesb@naesb.org / 713-356-0060) | Founder | **Now (week 1)** | Free |
| 2 | **Buy WGQ v4.1 full set** (Materials Order Form; all 10 manuals + data dictionaries + Edisim extracts). Minimum viable trio: Nominations 1.x + QEDM 4.x + IET 10.x = $750; +Flowing Gas 2.x + Cybersecurity 12.x = $1,250 | Founder | Week 1–2; **gates IG-level EDI simulator (b1 §B3)** | **$2,000 non-member** full set (beats $250/manual beyond 8 manuals) |
| 3 | **Membership decision**: $8,000/yr company-wide membership (all standards free, all employees, voting) vs. **$2,000/yr single-subcommittee pass** (WGQ IR/Technical or EDM); ad-hoc $250/meeting-day. Decision after first WGQ meeting attended | Founder | Decide by day 60 | Year-1 total **~$4,000** (set + subcommittee pass) or **$8,000** membership route |
| 4 | **MSA**: agent language mirroring tariff indemnity structure (mutual indemnities, aggregate cap tied to fees, submission SLA, force-majeure/EBB-outage carve-outs, penalty pass-through exclusions) + data-use terms (REQ-038) | Founder + counsel | Weeks; gates first paid pilot | — |
| 5 | **Penalty coverage-gap analysis with counsel**: are shipper OFO/imbalance penalties from a missed nomination covered by E&O or excluded as contractual penalties? (likely gap → explicit caps) | Founder + counsel + broker | Before MSA sign-off | — |
| 6 | **Insurance**: $1–2M tech E&O + $1–2M cyber, scalable to $5M (enterprise energy MSAs demand $1M–5M + SOC 2 Type II); bind via digital carriers (days) or broker (2–4 weeks) | Founder | Start month 1; gates first enterprise deal | ~$5–15K/yr early stage (E&O ~$1,516/yr at $1M/$1M; combined $2,500–6,000/yr under $1M revenue) |
| 7 | **SOC 2 Type II**: start immediately — the observation period is the long pole; controls per REQ-053 | DevSecOps + founder | Start now | — |
| 8 | **EDI build-vs-partner decision**: build native (pyx12 + per-pipeline enrollment) vs. partner behind the EDI adapter interface (Trellis Connect: 150+ pipelines, NAESB 3.2 certified, unified API; Enercross: 100+ EDI partners, 4M transactions/month, flat subscription; Systrends). **Decision deadline: end of day 90**, before the EDI adapter design freezes. Mitigation: the adapter interface (REQ-009) lets an aggregator sit behind the same contract, so the decision is reversible in design if made by the deadline | Founder | **Deadline: day 90** | — |
| 9 | **Pilot customer credentials + agency paperwork** (ESA/ECA execution per REQ-029) | Founder + pilot | Customer-dependent; target: full agency workflow with one pilot on one platform **in first 30 days** (§8 risk 1) | — |
| 10 | **Domain-expert hire**: former gas scheduler (≈10 yrs LDC desk or marketing shop); owns canonical-model edge cases, reviews every pipeline manifest, human in the review queue. **Before the connector framework freezes** — the highest-leverage hire (pack 6.3, b1 §E risk 7) | Founder | Before connector framework freeze | — |
| 11 | **Interim free dev work before purchase** (a2 §3): data model, cycle-timing engine, AS2 scaffolding on Mendelson, pyx12 envelope parsing, eTariff/eLibrary scraping, EBB/IPW notice scrapers | Build team | Weeks 1–2 | Free sources only |

Also on the path (post-v1, tracked): NAESB certification milestone (REQ-055); WGQ meeting participation for relationship-building (b1 §A10: no pipeline vendor programs exist — NAESB participation, aggregator partnership, and direct conversations with the largest platform operators are the relationship strategy).

---

## 8. Open risks register

The 10 ranked risks from b1 §E — the first 90 days must validate each. Deadlines are day-numbers from build start.

| # | Risk | Validation action | Owner | Deadline |
|---|---|---|---|---|
| 1 | Credential + agency onboarding speed: pilots complete credential handover, agent-designation notices, and EBB access agreements (ECA/DART ESA) fast enough to feed shadow mode | Run the full agency-onboarding workflow (REQ-029) with one pilot on one platform | Founder | Day 30 |
| 2 | Portal automation tolerance: platforms won't block/throttle/C&D credential-based automation; named-user/ESA terms permit the access pattern | Canaries + static egress IPs (technical tolerance); agreement review per pipeline (contractual permission). Prerequisite kill: per-pipeline toggle + C&D runbook exist first (REQ-025/026) | Founder + counsel | Day 60 |
| 3 | EDI channel strategy: unnecessary (portal-first) vs. partnerable vs. months-per-pipeline slog; whether target pipelines even offer active EDI | Channel discovery on the pilot's pipeline list (REQ-002); aggregator evaluations (Trellis Connect/Enercross/Systrends) | Founder | Day 90 (decision, §7 #8) |
| 4 | Platform-coverage thesis: ~9 platforms really cover ~80% of target EBBs for the first 20 customers' actual flows (a1's table is industry-wide; M&A drifts attribution) | PM pipeline census of the first-20-customer list before locking connector build order | PM + domain expert | Day 45 |
| 5 | Temporal Fairness under cutoff-spike load: the 2026-GA primitive must deliver per-tenant+per-pipeline isolation at 5 daily deadline spikes with our fairness-key scheme; per-partition and schedule-time caveats untested at our shape | Load harness against simulators (REQ-016) before ingestion design freezes | Architect | Day 60 |
| 6 | Simulator/fixture fidelity: synthetic clones + recorded fixtures produce shadow match rates customers trust; drift fixtures catch silent portal changes; IG-level EDI simulation achievable once manuals arrive (pyx12 map effort unknown) | Shadow-mode pilot match rates (REQ-027); drift-detection drill (REQ-014); map-authoring spike post-purchase | QA + connector lead | Day 75 |
| 7 | Canonical model completeness: one model + manifests absorb dialect edge cases without model rework | Domain-expert review of manifests against v4.1 manuals before connector framework freezes | Domain expert | Before framework freeze |
| 8 | Time-engine correctness: CCT/DST/cutoff math across all five cycles, both transition Sundays, per-pipeline tariff variations | Hypothesis property suite + golden DST tests (REQ-003) before any live submission | Core lead | Before first live submission |
| 9 | Legal/insurance closure: MSA agent language, liability caps surviving the OFO-penalty coverage gap, bound E&O/cyber | Term sheet, not assumption (§7 #4–6) | Founder + counsel | Day 90 |
| 10 | Sandboxed browser-fleet operability: gVisor-wrapped, ephemeral, egress-allowlisted workers viable at fleet scale (hundreds of concurrent 1pm-window sessions) at acceptable cost | Worker-tier load test + cost-curve measurement before committing the connector fleet design | Architect + DevSecOps | Day 60 |

---

**Pre-build human verification tasks (blocking gates, owner-assigned):**

| # | Task | Why | Owner | Deadline |
|---|---|---|---|---|
| V1 | Verify the Energy Transfer Messenger+ Nominations API exists and its capabilities at dev.messenger.energytransfer.com before the connector build order leans on it | Single medium-confidence source (a1 §2.3; §2.4 matrix); the sanctioned-API path (REQ-012) and ET's place in the build order depend on it | Founder/lead | Before sprint 1 |
| V2 | Re-verify NAESB prices on the current Materials Order Form before purchase | §7 figures are from the Feb 2024 membership sheet; prices drift | Ops | Before purchase |
| V3 | Temporal Fairness behavior under a simulated five-cycle cutoff spike | Already moved to the infra agent as a blocking validation gate (§6 flow, rule 5a); recorded here as a validation gate too — per-partition and schedule-time caveats must be exercised at our load shape | Architect | Before parallel dev agents spawn |

---

## 10. Explicit non-goals for v1

Stated so no agent builds them by drift; revisiting any item requires an ADR amendment.

| Non-goal | Why out of v1 |
|---|---|
| Invoicing | NAESB 3.4.x invoice/remittance datasets are outside the seven 1.4.x v1 EDI scope (§2.2); invoicing follows pipeline statements, not nominations |
| Capacity release | The 5.x dataset family (840/843/832) is a separate market with its own bidding/posting rules; no customer demand signal justifies it before the nomination core is trusted |
| Imbalance management / trading | Shipper-imbalance (2.4.4) and trading workflows are position-management, not submission-verification; the canonical model anticipates them but v1 builds no screens |
| Storage-specific flows | Storage nomination variants (injection/withdrawal, ratchets) differ per storage tariff; the platform fleet (§2.4) is built for transportation nominations first |
| Billing / metering automation | Billing is manual in v1 — measurement (2.4.5/2.4.6) and invoice data are not ingested; see §13 |
| Historical backfill of pre-onboarding nominations | Shadow mode (REQ-027/057) builds trust prospectively from onboarding day; backfill adds ingestion complexity with no verification value (v1.1 candidate, §13) |
| Mobile apps | The desk is a desktop workstation; htmx partials + SSE (§5.5) cover the alerting surfaces, and REQ-022 paging covers after-hours reach |

---

## 11. Glossary

Written for an engineer with zero gas industry background.

| Term | Definition |
|---|---|
| **Gas Day** | The industry's operating "day": 9:00 a.m. to 9:00 a.m. Central Clock Time. It is 23 or 25 hours long on the two US DST transition days (§2.1). |
| **CCT (Central Clock Time)** | America/Chicago wall-clock time *including* DST — i.e., whatever the clock says in Chicago right now, not a fixed UTC offset. All pipeline deadlines are stated in CCT. |
| **The five cycles (Timely / Evening / ID1 / ID2 / ID3)** | The five daily windows in which shippers may submit or revise nominations for a gas day (NAESB WGQ Std 1.3.2, §2.1). Timely and Evening are day-ahead; Intraday 1–3 are same-day revisions with progressively later deadlines and reduced flexibility. |
| **QR (Quick Response)** | The pipeline's immediate machine acknowledgment that it received a nomination (dataset 1.4.2). Under WGQ v4.1 it is due within max 5 minutes of receipt; a missing/late QR is an escalation trigger (REQ-019). |
| **NMQR** | Nomination Quick Response — the formal NAESB name for the QR dataset (X12 855, code G855NMQR). QR and NMQR are the same thing in this spec. |
| **EPSQ (elapsed prorated scheduled quantity)** | For intraday reductions, the pipeline prorates the scheduled quantity over the remaining hours of the gas day — you cannot retroactively "un-flow" gas already flowed. Auto-renomination must respect EPSQ rules per pipeline manifest (REQ-042/060). |
| **OFO (Operational Flow Order)** | A pipeline-issued order compelling shippers to keep flows within stated tolerances during system stress, backed by penalty multipliers. The storm-mode trigger class (REQ-039). |
| **Critical notice** | A pipeline's highest-severity informational posting (impending OFOs, force majeure, curtailments). Elliott's critical notices peaked on Christmas Eve — hence 24/7 ingestion. |
| **Bump / no-bump** | On bumpable cycles, a later-submitted higher-priority nomination can displace ("bump") an earlier lower-priority one. ID3 is no-bump: once scheduled there, you cannot be displaced. The `bumpable` flag is in the canonical model (ADR-001). |
| **Service requester** | The NAESB term for the party submitting a nomination: "the Shipper or its Agent." A canonical-model identity field (ADR-001). |
| **Agent (tariff sense)** | A third party formally designated in writing (per the pipeline's tariff/ESA/ECA paperwork) to act on a shipper's behalf. This product operates as an agent; agency paperwork is REQ-029. |
| **EBB / CAW / IPW** | Electronic Bulletin Board — the pipeline's customer web portal where nominations are entered and notices posted. CAW (Customer Activities Website) and IPW (Informational Postings Website) are vendor names for the same thing (IPW often denotes the public-notices portion). |
| **NAESB WGQ** | North American Energy Standards Board, Wholesale Gas Quadrant — the body that writes the data/transport standards (WGQ v4.0 mandated by FERC; v4.1 published) governing nominations, confirmations, and EDI transport (§2.2–2.3). |
| **EDI / EDM** | Electronic Data Interchange — machine-to-machine X12 message exchange instead of portal typing. EDM (Electronic Delivery Mechanism) is the NAESB/GISBN HTTP/S-AS2 transport lineage those messages ride on (§2.2). |
| **VAN (Value-Added Network)** | A store-and-forward intermediary some parties use to exchange EDI instead of point-to-point AS2. Relevant to EDI enrollment topology; not built by us. |
| **Shadow mode** | Onboarding phase where the system runs the full pipeline in parallel with the human's manual submissions and diffs the two, submitting nothing (REQ-027/057). Promotion to live is gated by REQ-058. |
| **Cut** | When the pipeline schedules less gas than nominated, the difference is a "cut." The workflow records nominated-vs-scheduled deltas and offers next-cycle renomination (REQ-060). |
| **Dialect / manifest** | Each pipeline varies the standard (timings, location codes, dataset versions); that variation is its dialect, captured declaratively in a per-pipeline manifest (REQ-004) so dialects are data, not code. |
| **Fixture** | A recorded, sanitized real portal/EDI response used as test gold and as a drift canary baseline. Fixtures are tamper-evident via hash manifests (REQ-006/014/052). |
| **Canary** | A scheduled synthetic probe that replays fixtures against production portals to detect silent breakage (layout drift, outages) before customers do (REQ-014/028/063). |

---

## 12. Writer's ambiguity log (spec-set values)

Six judgment calls in this spec are the writer's, not sourced facts. Each is flagged where it appears; all are listed here so reviewers can challenge them in one place.

| # | Value | Basis |
|---|---|---|
| (a) | Dashboard load p95 ≤ 2 s (REQ-046(c)) | Spec-set target; no external source |
| (b) | ≤ 5 MB/nomination compressed artifact budget (REQ-047) | Spec-set budget |
| (c) | Storm-mode split: OFO ingestion (REQ-039) + countdowns (REQ-040) in v1; penalty calculator (REQ-041) + EPSQ auto-renom (REQ-042) in v1.1 | Writer's phasing judgment; b1 C5 groups them |
| (d) | EDI connector phasing: adapter interface + envelope-level EDI sim in v1 (REQ-007/009); native/aggregator EDI connector + NAESB IET/QEDM transport in v1.1, gated on the day-90 build-vs-partner decision (§7 #8) and the WGQ v4.1 purchase (§7 #2) | Writer's phasing judgment from b1 §B3/C9 |
| (e) | DST golden-test dates 2026-03-08 and 2026-11-01 (REQ-003) | US calendar facts, not spec-set |
| (f) | ADR-001 anticipates ETRM mappings via reserved `etrm.*` namespace + `deal_id`; no per-ETRM field mappings yet | Writer's scoping call; mappings deferred to named-ETRM work |

---

## 13. Deferred items (stated, not silent)

- **Billing / metering automation** — manual in v1 (§10); no measurement or invoice ingestion built. Revisit only with customer demand.
- **Tenant data lifecycle detail** — the principle is stated now: audit metadata is retained per the retention schedule (WORM); position artifacts are deletable on offboarding with deletion certificates. The tension between deletion rights and WORM audit is resolved as: **the audit keeps hashes and event metadata, not position payloads beyond retention** (ADR-006/REQ-037).
- **Historical backfill of pre-onboarding nominations** — v1.1 candidate (§10); shadow mode covers trust-building prospectively.

---

## 14. Sources

All URLs below are extracted verbatim from the four research briefs and the pressure test; none are invented. Grouped by topic.

### 14.1 EBB landscape, pipelines, and platforms
- https://www.ferc.gov/industries-data/natural-gas/overview/general-information/natural-gas-industry-forms/three-digit
- https://www.ogj.com/pipelines-transportation/article/17209510/natural-gas-pipeline-profits-construction-both-up
- https://www.naesb.org/members/urls_of_pipelines.htm
- https://www.gao.gov/assets/rced-87-133br.pdf
- https://www.oecd.org/content/dam/oecd/en/publications/reports/2008/02/energy-policies-of-iea-countries-united-states-2007_g1gh7ed9/9789264030749-en.pdf
- https://trellisenergy.com/trellis-blog/intro-naesb-standards-ferc-compliance
- https://contentsunderpressurebook.com/2015/05/29/terminology-what-is-an-ebb/
- https://www.vnf.com/1743
- https://www.naesb.org/pdf4/geh_final_report_072823.pdf
- https://www.1line.williams.com/Transco/files/training/Nominations.pdf
- https://pipeline2.kindermorgan.com/Documents/KGS/Keystone_Gas_Storage_DART_Users_Guide-20180307115947.pdf
- https://www.piperiv.com/feed/notice/millennium/25975716/38955
- https://pipeline2.kindermorgan.com/Documents/EPNG/EPNG_ContactUs.pdf
- https://pipeline2.kindermorgan.com/Documents/CP/CP_Jobtitlesanddescriptions-20231106091315.pdf
- https://www.tceconnects.com/infopost/ReportViewer.aspx?%2FInfoPost%2FNoticesSubreport&pNoticeId=25975715&AssetNbr=44
- https://infopost.enbridge.com/regulatory/tariff/GNGSVolumeNo1.pdf
- https://apis.io/providers/kinder-morgan/
- https://www.piperiv.com/ip/mountain-valley
- https://www.naesb.org/pdf4/geh062923w2.pdf
- https://documents.dps.ny.gov/public/Common/ViewDoc.aspx?DocRefId=%7BDE6165C7-E468-4C11-B08C-5ACBACD7D193%7D
- https://www.nrdc.org/sites/default/files/2024-09/Joint_Petition_for_Review_CP2_1.pdf
- https://pscdocs.utah.gov/misc/21docs/2199902/321118NorthwestPipelineGasTrfNAmrcEnrgBrd11-10-2021.pdf
- https://www.ferc.gov/sites/default/files/2020-05/005939_000100__contents.pdf
- https://www.capco.com/intelligence/capco-intelligence/modernizing-the-most-challenging-job-in-energy
- https://www.pcienergysolutions.com/2023/10/03/the-complete-guide-to-streamlining-natural-gas-scheduling/
- https://www.linkedin.com/company/trellis-energy
- https://www.mwpipe.com/
- https://www.quorumsoftware.com/solutions/my-quorum-pipeline/
- https://www.prnewswire.com/news-releases/trellis-energy-enables-naesb-30-compliance-for-natural-gas-pipelines-300314559.html
- http://www.gasnom.com/ip/CENTRALVALLEY/tariff.cfm?page=6
- https://www.oati.com/news/semco-energy-goes-live-with-oatis-natural-gas-pipeline-transportation-management-system/
- https://www.pandell.com/pdf/brochure_esg_latitude_quicknom.pdf
- https://customers.equitransmidstream.com/CustomerPortal/
- https://gascompressionmagazine.com/2025/04/02/two-transco-projects-enter-service-to-meet-southeast-us-energy-demand/
- https://nepis.epa.gov/Exe/ZyPURL.cgi?Dockey=P1004FL2.TXT
- https://www.enbridge.com/~/media/Enb/Documents/Factsheets/ProvinceStateEconomicBenefits/FS_Tennessee_economic_benefits.pdf
- https://apis.io/apis/energy-transfer/energy-transfer-nominations-api/

### 14.2 NAESB standards, cycles, X12, transport, FERC
- https://naesb.org/pdf4/naesb_092526_wgq_version_4.1_report.pdf
- https://www.law.cornell.edu/cfr/text/18/284.12
- https://www.federalregister.gov/documents/2024/12/09/2024-28090/standards-for-business-practices-of-interstate-natural-gas-pipelines
- https://regulations.justia.com/regulations/fedreg/2024/04/05/2024-06561.html
- https://www.naesb.org/pdf/membership_benefits.pdf
- https://www.naesb.org/wgq/imp.asp
- https://www.naesb.org/misc/NAESB_Nonmember_Evaluation.pdf
- https://www.naesb.org/committee_activities.asp
- https://www.naesb.org/monthly_update.asp
- https://www.gasnom.com/ip/gulfshore/tariff.cfm?page=3
- https://www.ferc.gov/sites/default/files/2020-08/M-1.pdf
- https://www.jdsupra.com/legalnews/ferc-issues-order-no-809-on-gaselectri-81867/
- https://northwest.williams.com/NWP_Portal/file_download?hfFileURL=Files/Northwest/Downloads/Order809.pdf
- http://naesb.org/pdf/X12names.pdf
- https://www.naesb.org/pdf/c03003.pdf
- https://www.govinfo.gov/content/pkg/FR-2012-07-26/pdf/2012-18105.pdf
- https://www.rfc-editor.org/info/rfc4130/
- https://www.federalregister.gov/documents/2024/12/03/2024-28244/
- https://www.ferc.gov/ferc-online/etariff
- https://naesb.org/pdf4/weq_wgq_rmq_bps050924reqcom_INGAA.pdf
- https://enercast.com/pipeline/tgp/
- http://www.naesb.org/pdf/idaywk3.pdf
- https://www.jdsupra.com/legalnews/naesb-response-to-ferc-nopr-... (truncated in a3 source)
- https://www.naesb.org/materials/certification.asp
- https://www.naesb.org/pdf2/cert_products.pdf

### 14.3 Technology: workflow engines, sandboxing, audit, time, simulators, frontend
- https://automationatlas.io/answers/temporal-pricing-explained-2026/
- https://valueaddvc.com/blog/temporal-valuation-2026-12-55b-series-e-and-250m-arr-explained
- https://temporal.io/change-log/type/feature
- https://docs.temporal.io/develop/task-queue-priority-fairness
- https://temporal.io/blog/task-queue-priority-and-fairness-your-task-queue-your-way
- https://community.temporal.io/t/how-to-achive-a-fair-qos-in-a-multi-tenant-saas/13644
- https://www.youngju.dev/blog/culture/2026-05-14-durable-execution-engines-2026-temporal-restate-inngest-trigger-dev-dbos-deep-dive-2026.en
- https://kanopylabs.com/blog/restate-vs-temporal-vs-dbos-durable-execution
- https://onidel.com/blog/nats-jetstream-rabbitmq-kafka-2025-benchmarks
- https://www.ofashandfire.com/blog/multi-tenant-saas-architecture-b2b-platforms
- https://zylos.ai/research/2026-04-04-ai-agent-sandboxing-security-isolation/
- https://www.labo-llm.fr/en/carnets/deny-rules-securite
- https://grigio.org/ai-agent-sandbox-technologies-a-complete-2026-comparison/
- https://fly.io/learn/firecracker-vs-gvisor/
- https://www.tigera.io/blog/secure-egress-access-with-dns-policy-and-networksets/
- https://www.infoq.com/news/2024/07/aws-kill-qldb/
- https://sigilbase.io/writing/best-cryptographic-audit-log-solutions/
- https://immudb.io/blog/
- https://blog.elest.io/immudb-ai-agents-build-a-tamper-proof-audit-log/
- https://www.digitalapplied.com/blog/agent-audit-trail-design-7-best-practices-2026
- https://veriprajna.com/services/continuous-monitoring-audit-trails
- https://oneuptime.com/blog/post/2026-02-12-configure-s3-object-lock-worm-compliance/view
- https://www.trustwarden.ai/blog/why-s3-object-lock-over-qldb/
- https://academify.com.br/en/python-zoneinfo-time-zones/
- https://runebook.dev/en/docs/python/library/zoneinfo
- https://pypi.org/project/pyx12/
- https://github.com/azoner/pyx12
- https://pyx12.readthedocs.io/
- https://mendelson.de/blog/mendelson-as2-opensource-b67-released
- https://mendelson-e-c.com/opensource
- https://sourceforge.net/projects/mec-as2/files/
- https://mendelson.de/opensource
- https://www.wiremock.io/post/top-mountebank-alternatives
- https://tessl.io/registry/testland/mountebank-imposters/1.3.6
- https://htmx.org/essays/future/
- https://morello.dev/blog/htmx-4
- https://simonwillison.net/tags/carson-gross/
- https://hyperscript.org/posts/2026-03-29-hyperscript-0.9.90-is-released/
- https://www.birjob.com/blog/htmx-production-2026
- https://pockit.tools/blog/htmx-vs-react-2026-when-you-dont-need-spa/
- https://testparty.ai/blog/continuous-accessibility-testing
- https://www.qamadness.com/a-you-oriented-guide-to-axe-core-playwright-accessibility-testing/
- https://fastapi.tiangolo.com/release-notes/
- https://github.com/fastapi/fastapi/issues/15466

### 14.4 Legal: agency, tariffs, access agreements, case law
- https://ebb.anrpl.com/tariff/ANRTariff.pdf
- https://www.blackhillsenergy.com/sites/blackhillsenergy.com/files/black_hills_shoshone_tariff.pdf
- https://www.pembina.com/getmedia/09d7ba84-8dad-45b2-aa50-1b8ef3d8c73c/Alliance-Pipeline-Canadian-Tariff-Effective-2025-09-01.pdf
- https://www.nationalfuel.com/business-partners/how-to-become-a-marketer-in-new-york/
- https://www.ferc.gov/sites/default/files/2020-06/Order-587-Y.pdf
- https://www.ferc.gov/sites/default/files/2020-05/023632_000120__contents.pdf
- https://www.enbridgegas.com/-/media/Extranet-Pages/ohio/large-business-services/transportation-services/natural-gas-supplier/ebb-access-agreement.pdf
- https://pipeline2.kindermorgan.com/Documents/common/DART_License_Inter-20180312151645.pdf
- https://www.ferc.gov/sites/default/files/2020-05/032620_000100__contents.pdf
- https://www.piperiv.com/feed/notice/anr/12576/29843
- https://forage.ai/blog/us-web-scraping-laws/
- https://www.jlcw.org/case-law/facebook-v-power-ventures-9th-cir-2016
- https://truthonthemarket.com/2026/06/11/wargames-shopping-bots-and-the-statute-trap-the-cfaa-and-amazon-v-perplexity/
- https://prospeo.io/s/web-data-extraction
- https://www.govinfo.gov/content/pkg/FR-2022-06-15/xml/FR-2022-06-15.xml

### 14.5 Commercial: competitors, pricing, insurance, EDI services
- https://www.energytechreview.com/natgas-hub
- https://natgashub.com/
- https://finance.yahoo.com/sectors/energy/articles/natgashub-com-launches-voicenoms-revolutionary-225700863.html
- https://enercross.com/edi/
- https://trellisenergy.com/platform/trellis-connect
- https://trellisenergy.com/info
- https://iongroup.com/resource-center/commodities/allegro-natgashub-connect-gas-scheduling-software-for-pipeline-scale-operations/
- https://www.systrends.com/edi-billing
- https://ecgrid.com/b2b-edi-solutions-for-business/
- https://www.commport.com/how-edi-is-used-in-the-oil-and-gas-industry/
- https://www.linkedin.com/top-content/supply-chain-management/digital-transformation-in-supply-chains/electronic-data-interchange-edi-in-supply-chains/
- https://anvo-insurance.com/blog/tech-saas-insurance-faq/
- https://seedpodcyber.com/cyber-insurance-for-startups/
- https://joinalliancerisk.com/startup-insurance-enterprise-contracts/
- https://www.insureon.com/technology-business-insurance/saas-companies/cost
- https://help.coalitioninc.com/hc/en-us/articles/7608450648475
- https://www.corgi.insure/blog/what-insurance-providers-have-the-fastest-underwriting-for-startups

### 14.6 Storm evidence (Uri / Elliott)
- https://www.powermag.com/vistra-claims-extortion-by-pipeline-company-says-natural-gas-supply-to-texas-power-plants-threatened/
- https://www.efis.psc.mo.gov/Document/Display/23913
- https://www.whitecase.com/insight-alert/summary-ferc-meeting-agenda-october-2021
- https://goarbo.com/blog/texas-winter-storm-operational-flow-order-penalties-imposed-on-some-shippers/
- https://www.ferc.gov/sites/default/files/2023-11/24_Winter-Storm_Elliot_1107_1300.pdf

---

*End of Build Spec v1.2 (frozen). Changes only via ADR amendment with contract tests in the same PR.*
