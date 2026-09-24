# GO-PJI Playbook: Graph-Ontology → Patient Journey Intelligence

**The authoritative guide to turning heterogeneous clinical records into longitudinal, normalised, deduplicated
patient journeys:**
- stored in the OMOP Common Data Model;
- linked to Graph-Ontology concepts;
- summarised into episodes;
- queryable as timelines and reproducible cohorts;
- governed for Australian patient data.

Version 1 · 24 Sep 2026 · Arepo-Medtech/GO-PJI

This playbook is self-contained. Its only dependency inside this repository is **`T0.md`**: the frozen, validated
Graph-Ontology release this track normalises against. It assumes T0 is complete and that this track runs in isolation
from the other Graph-Ontology tracks. Code normalisation is done here, from T0's views. An external FHIR terminology
service (for example the national one) may optionally be used for SNOMED expression-constraint expansions.

> Real patient data enters only after the governance gate in §11. Until then this track runs on synthetic data, and
> synthetic results are never presented as clinical performance. Status: plan. Nothing described here is built yet.

---

## Contents

1. [Mission and definition of done](#1-mission-and-definition-of-done)
2. [Non-negotiable rules](#2-non-negotiable-rules)
3. [Architecture](#3-architecture)
4. [Inputs](#4-inputs)
5. [Stages and gates](#5-stages-and-gates)
6. [The event model](#6-the-event-model)
7. [Ingestion](#7-ingestion)
8. [Normalisation against T0](#8-normalisation-against-t0)
9. [Loading OMOP CDM](#9-loading-omop-cdm)
10. [Deduplication, episodes and journeys](#10-deduplication-episodes-and-journeys)
11. [Governance, privacy and security](#11-governance-privacy-and-security)
12. [Data quality and validation](#12-data-quality-and-validation)
13. [Cohorts and phenotypes](#13-cohorts-and-phenotypes)
14. [Clinical notes (later phase)](#14-clinical-notes-later-phase)
15. [Serving](#15-serving)
16. [Scorecard](#16-scorecard)
17. [Operations](#17-operations)
18. [Risk and failure-mode register](#18-risk-and-failure-mode-register)
19. [Roles and effort](#19-roles-and-effort)
20. [Prompt library](#20-prompt-library)
21. [Decisions to make](#21-decisions-to-make)
22. [References](#22-references)
- [Appendix A: Hand-check protocol](#appendix-a-hand-check-protocol)
- [Appendix B: Templates](#appendix-b-templates)

---

## 1. Mission and definition of done

### 1.1 Mission
Answer "what happened to this patient, in what order, and what does it mean" reliably and reproducibly:
- **Normalise:** every clinical fact is mapped to a standard concept, recording the mapping's tier.
- **Order:** facts become events on a timeline, with explicit time precision.
- **Deduplicate:** repeated records of the same event are merged without losing lineage.
- **Summarise:** events are grouped into episodes (condition eras, drug exposures and eras, visits, treatment lines).
- **Query:** timelines per person, and cohorts defined against versioned concept sets.

### 1.2 Definition of done (v1, synthetic data)
1. Synthetic Australian and US cohorts load end to end: ingest → normalise → OMOP → deduplicate → episodes → quality
   checks → publish.
2. The data-quality check suite passes its thresholds (§12).
3. Mapping coverage meets its target per domain (§12.3).
4. Deduplication **merge precision** has a Wilson lower bound ≥ 0.90 by hand check (§12.5).
5. Cohort definitions reproduce exactly by concept-set expansion hash under a pinned release.
6. The governance pack (§11) is complete and approved **before** any real data.

---

## 2. Non-negotiable rules

| # | Rule | Enforced by |
|---|---|---|
| P1 | **No real patient data before governance approval**: ethics or HREC approval, a data-governance agreement, a privacy impact assessment. | §11 gate G-0 |
| P2 | **Patient data never goes to an external model or API.** Any model used on patient data runs inside the governed environment. | network egress rules; architecture review |
| P3 | **Never guess a code.** An unmapped source code becomes standard concept 0, with the source value and code kept, and is queued. It is never mapped by guesswork. | normaliser; §8.4 |
| P4 | **Every event keeps its lineage:** source system, record id, raw code and value, mapping route and tier. | event model (§6) |
| P5 | **Merge only what is the same event.** Deduplication requires concept compatibility **and** attribute compatibility, within a time window. A laterality or body-site conflict always blocks a merge. | §10.2 |
| P6 | **Clinicians see only concepts backed by T0's `edge_clinical`.** Other mappings are usable for analytics, and are labelled. | serving layer |
| P7 | **Cohorts are versioned by release:** concept sets are pinned to the T0 release, and their expansion hash is recorded. | cohort registry |
| P8 | **Quality gates refuse.** A load that fails a data-quality threshold is not published. | orchestrator |
| P9 | **Minimum necessary data**, and de-identified analytic copies by default. | data design; §11.4 |
| P10 | **Only licensed use** of every vocabulary (T0 §2). | `t0 licences check --track T3` |

---

## 3. Architecture

```
 SOURCES                      INGEST                NORMALISE (T0)             STORE                 SERVE
 FHIR (AU Core) bundles  ─┐                         source code → standard     OMOP CDM v5.4         timeline API
 openEHR compositions    ─┼─→ staging (raw,  ──→    concept via T0 views   ──→ (Postgres)      ──→   cohort API
 CSV / extracts          ─┤   immutable)            + tier; units → UCUM       + journey tables      quality reports
 HL7 v2 (if in scope)    ─┘                         (AU conventions)           (events, links,       patient-mode tools
                                                                                episodes)             (governed only)
                                    ▼                         ▼                        ▼
                               lineage ids            unmapped queue          dedup + episodes
                                                                               data-quality checks
                                                                               (DataQualityDashboard,
                                                                               Achilles)
```

**Three models, one join key:**
- **openEHR** is the clinical record (archetyped, versioned).
- **FHIR** (Australian profiles) is exchange.
- **OMOP CDM** is analytics.

Graph-Ontology concept codes join all three. The journey tables sit beside OMOP and reference OMOP ids. They never
duplicate the reference graph per patient.

---

## 4. Inputs

### 4.1 From T0

| T0 artefact (see `T0.md`) | Used for |
|---|---|
| `t0.lock` → release | pinning; recorded in every ETL run manifest |
| `views/edge_clinical`, `views/edge_displayable` | mapping edges (source code → standard concept) and their tiers |
| `views/equivalence_group` | representatives for codes the graph declares equivalent; conflict flags |
| `views/synonym` | matching of free-text source values, if needed (local only) |
| is-a edges in the release (SNOMED CT-AU, other hierarchies) | subsumption for deduplication and cohort concept sets |
| unit edges (LOINC → UCUM, Australian preferred units, conversion factors) | unit harmonisation |
| medicine edges (AMT product → ingredient; salt → base) | drug exposure and era construction |
| `register/route_register` | mapping tier recorded per event |
| `config/licence_matrix.yaml` | permission to host and display each vocabulary |
| `quiz/items` tagged for normalisation | regression tests for the normaliser |

### 4.2 External (standards and software)

| Item | Role |
|---|---|
| **OMOP CDM v5.4** and the OHDSI standardised vocabularies | analytic store; the standard concept ids OMOP tools expect |
| **OHDSI tools:** Achilles, DataQualityDashboard, ATLAS / Circe, CohortDiagnostics, PheValuator | characterisation, quality, cohorts, phenotype evaluation |
| **openEHR** platform (for example EHRbase) and archetypes from the openEHR Clinical Knowledge Manager | clinical record store (optional, if openEHR is a source) |
| an openEHR → OMOP transformer (for example Eos) | if openEHR is the source of record |
| **HL7 AU Core** (currently R2, v2.0.0) and AU Base FHIR profiles | Australian FHIR input profiles |
| an orchestrator (Dagster or Airflow) and dbt or SQL transforms | pipeline |
| synthetic data: the CSIRO Australian Synthea collection (CC BY 4.0) and US Synthea | development and testing |

---

## 5. Stages and gates

| Stage | Objective | Deliverables | Exit gate |
|---|---|---|---|
| **J0 Frame** | uses, questions, governance plan | `docs/use-cases.md`, `docs/governance-plan.md` | signed by the pipeline lead |
| **J1 Event model and schema** | the canonical event model; OMOP plus journey tables | DDL; data dictionary | reviewed by an informatician |
| **J2 Ingest** | adapters for each source type | adapters; staging; lineage | 100% of synthetic records staged with lineage |
| **J3 Normalise** | codes → standard concepts, units → UCUM | normaliser; unmapped queue; tests | mapping coverage targets met (§12.3); normalisation quiz items pass |
| **J4 OMOP load** | populate the CDM | ETL; ETL specification document | DataQualityDashboard thresholds pass (§12.2) |
| **J5 Dedup and episodes** | same-event merges; eras; treatment lines | rules; code; lineage | merge precision lower bound ≥ 0.90; non-merge boundary check passes (§12.5) |
| **J6 Cohorts** | versioned concept sets and cohort definitions | cohort registry; diagnostics | expansions reproduce by hash; phenotype PPV reported (§13) |
| **J7 Serve** | timeline and cohort APIs | services; audit | access-control tests pass; audit complete |
| **G-0 Governance gate** | approval for real data | governance pack (§11) | approvals in hand |
| **J8 Real-data release** | the first governed load | release checklist | all gates pass on real data |

---

## 6. The event model

### 6.1 The canonical event
Every clinical fact becomes one event row before OMOP loading. Loading is then a deterministic projection.

| Field | Meaning |
|---|---|
| `event_id` | stable id: hash of source system, record id and element path |
| `person_id` | internal pseudonymous id (§11.4) |
| `domain` | condition, drug, measurement, procedure, observation, device, visit, death |
| `start`, `end` | timestamps. `end` is null for point events |
| `time_precision` | year, month, day, minute, second: never invent precision |
| `concept_system`, `concept_code` | the standard concept after normalisation |
| `concept_tier` | tier of the mapping route (from the T0 register); `source` if no mapping was needed |
| `source_system`, `source_code`, `source_value` | exactly as received |
| `value` fields | `value_number`, `value_unit_ucum`, `value_concept`, `value_text` |
| attributes | `laterality`, `body_site`, `specimen`, `route`, `dose`, `status` (active, resolved, cancelled, entered in error), `certainty` (confirmed, provisional, refuted) |
| `provenance` | record id, author and role, recorded time, ingest run id |
| `merged_into` | the event id this row was merged into, if a duplicate (§10) |

### 6.2 Time semantics
- **Event time, not record time.** Use onset, administration or collection time where the source has it. Keep the
  record time in the provenance.
- **Precision is explicit.** A condition recorded as "2019" has `time_precision = year`. Don't assign 1 January as if
  it were known.
- **Status matters.** `entered in error` events are kept for lineage but excluded from journeys. `refuted` conditions
  are never counted as present.

---

## 7. Ingestion

### 7.1 Principles
- **Staging is immutable.** Raw payloads are stored as received, with a checksum, before any transformation.
- **One adapter per source type**, each producing canonical events plus lineage.
- **Idempotent:** re-ingesting the same payload produces the same event ids.

### 7.2 FHIR (AU Core / AU Base)
- **Resources to map:** Patient (to the pseudonymous id; demographics minimised), Encounter, Condition, Observation,
  MedicationRequest, MedicationStatement, MedicationAdministration, Procedure, Immunization, AllergyIntolerance,
  DiagnosticReport.
- **Codings:** take every `coding`; prefer the SNOMED CT-AU coding, then AMT for medicines, then LOINC for
  observations. Keep the others as source codes.
- **Encounter class:** map the HL7 ActCode abbreviations (for example `AMB`, `IMP`, `EMER`, `HH`, `VR`) to the visit
  concept. Test with real AU payloads, because vendors differ.
- **Dosage:** keep `dosageInstruction` text and structured fields. Don't compute days' supply where the source doesn't
  support it.

### 7.3 openEHR
- Transform compositions through the chosen templates. **Node identity is the archetype's at-code path.**
- One entry per clinical statement. An entry carrying many clusters can collapse into one downstream row in some
  transformers, so test that each answer or finding yields its own event.
- Coded text values are mapped through the normaliser, not by joining on display text.

### 7.4 CSV and extracts
Each extract gets a written **source-to-event mapping specification**: field-by-field transforms, code-system
assumptions, and date formats. Review it before implementation.

### 7.5 Lineage
Every event records the ingest run id. The run manifest records the source files and their checksums, the adapter
versions, the T0 release id and the normaliser version.

---

## 8. Normalisation against T0

### 8.1 Order of resolution
For each source (system, code):
1. **Already standard?** If the code is in a standard system for its domain (SNOMED CT-AU for conditions and
   procedures; AMT or RxNorm for drugs; LOINC for measurements) and is active in the pinned release, keep it, with
   `concept_tier = source`.
2. **Retired?** Resolve it through the release's historical-association edges (replaced by, same as) to an active
   code. Record the route. Several candidates → go to the unmapped queue.
3. **Mapped?** Follow mapping edges from `edge_clinical` first, then `edge_displayable`, recording the route and its
   tier. Prefer an exact or equivalent mapping over a broader one. Record the kind (exact, broader) in the event.
4. **Equivalence:** replace by the T0 `equivalence_group` representative for the domain, unless the group is flagged
   `has_conflict`.
5. **OMOP standard concept:** for the OMOP load, resolve the final code to the OHDSI standard concept id through the
   release's OMOP mapping edges. Where the OMOP vocabularies lack the Australian concept, use the release's
   nearest-standard-ancestor edges and record that as a broader mapping.
6. **Otherwise:** unmapped. Standard concept 0, source fields kept, queued (§8.4).

### 8.2 Units
- Normalise units to UCUM, using the release's LOINC → UCUM and Australian preferred-unit edges.
- Convert values only through a conversion edge. Mass ↔ molar conversions use the analyte's molecular-weight factor
  **only** when the release carries it with its source.
- Keep the original value and unit on the event.
- Australian reporting conventions (for example phosphate reported as phosphorus) are applied only as recorded in the
  release. Never assume them.

### 8.3 Medicines
- Map products to ingredients through the release's product → ingredient edges. Apply the release's salt → base
  policy, so salts group with their base ingredient for exposure purposes.
- Keep the product-level code on the event, and derive the ingredient in the drug-exposure projection.
- Route of administration comes from the product's dose form (SNOMED's intended-site attribute) or the source.

### 8.4 The unmapped queue
`unmapped(source_system, source_code, source_value, domain, count, first_seen, last_seen, example_event_ids)`, ranked
by count. It is reviewed by an informatician. Resolutions go back as mapping requests to the T0 producer; they are
never hot-patched here. Coverage reports (§12.3) show the queue's share per domain.

### 8.5 Tests
- Unit tests per resolution step.
- The T0 quiz items tagged for normalisation.
- **Golden files:** synthetic patients with hand-verified expected events.

---

## 9. Loading OMOP CDM

### 9.1 Conventions
Follow the OMOP CDM v5.4 specification and the OHDSI community conventions (THEMIS) for each table:

| Table | Holds |
|---|---|
| `person`, `observation_period`, `visit_occurrence`, `visit_detail` | the person and their encounters |
| `condition_occurrence`, `drug_exposure`, `procedure_occurrence`, `measurement`, `observation`, `device_exposure`, `death` | the clinical events |
| `condition_era`, `drug_era` | derived episodes |
| `cdm_source` | records the T0 release and ETL version |

### 9.2 Projection rules (event → OMOP)
- `*_concept_id` = the standard concept; `*_source_concept_id` = the source concept if it's in the vocabularies,
  otherwise 0; `*_source_value` = the raw code or value.
- `*_type_concept_id` records the provenance type (EHR, claim, registry).
- Measurements: `value_as_number` and `unit_concept_id` from UCUM; `value_as_concept_id` for coded results.
- **Store derived ingredients in `drug_era` only.** `drug_exposure` keeps the product-level concept.

### 9.3 ETL specification document
Write it before coding (Appendix B.2). For each source element: target table and column, transform, vocabulary
route, and test case. Keep it in version control; reviewers sign it off.

---

## 10. Deduplication, episodes and journeys

### 10.1 Why
The same clinical event is often recorded more than once: in a referral, a discharge summary and a problem list;
as a medication order and a dispensing record; as a lab result in two feeds. Without deduplication, counts and
timelines are wrong. With over-eager deduplication, distinct events are fused. That's worse, because it's invisible.

### 10.2 Merge rules (conservative by design)
1. **Blocking:** candidate pairs share person and domain, with starts within a window. Default windows:
   - conditions: same encounter, or ±7 days;
   - measurements: ±1 hour of collection time, and the same specimen;
   - drugs: overlapping or adjacent exposure periods (≤ 1 day apart) for the same ingredient;
   - procedures: the same day.
2. **Concept compatibility:** the codes are equal, **or** one subsumes the other in the release's is-a hierarchy
   ("pain of hip" subsumes "pain of left hip"). Codes related only by a broad classification never merge.
3. **Attribute compatibility:** no conflict in laterality (SNOMED attribute 272741003), finding or body site
   (363698007), specimen, route, or value:
   - measurements: values equal after unit conversion, within the assay's reporting precision;
   - conditions: certainty not contradictory (a refuted and a confirmed version don't merge).
4. **Result:** the survivor takes the **most specific compatible concept** ("left hip pain" over "hip pain"), the
   earliest reliable start, and the union of lineage. The others set `merged_into`.

Deduplication **never deletes**; it links. Every merge is reversible.

### 10.3 Episodes
- **Visits:** from encounters, typed by encounter class.
- **Condition eras:** consecutive occurrences of the same condition concept, joined when the gap is ≤ a persistence
  window. The default is 30 days, following the OHDSI era convention.
- **Drug exposures and eras:** exposures per ingredient, joined into eras when the gap is ≤ 30 days (the default).
  Record the gap parameter in the run manifest.
- **Treatment lines** (for defined conditions): ordered sequences of ingredient sets, with rule-defined line changes
  (a new ingredient added, or a switch), documented per condition. They need clinical review before use.

### 10.4 Journey tables
- `journey_event (event_id, person_id, start, end, time_precision, domain, concept, tier, merged_into, episode_id)`
- `journey_episode (episode_id, person_id, kind, concept, start, end, members)`
- `journey_link (event_id, reference_concept, relation)`: links events to reference-graph concepts (is-a ancestors
  used for grouping; interpretation links such as lab result → finding), **by id only**.

The reference graph itself is never copied per patient.

---

## 11. Governance, privacy and security

### 11.1 Legal and ethical frame (Australia)
Confirm each item with the data custodian and privacy officer for the specific data source.
- **Privacy Act 1988 (Cth)** and the Australian Privacy Principles, including APP 6 (use and disclosure), APP 8
  (cross-border disclosure) and APP 11 (security).
- **State and territory health records legislation**, where applicable: for example the Health Records Act 2001
  (Vic), the Health Records and Information Privacy Act 2002 (NSW), and the Health Records (Privacy and Access) Act
  1997 (ACT).
- **My Health Records Act 2012**, if any My Health Record data is involved.
- **NHMRC National Statement on Ethical Conduct in Human Research:** HREC review, and consent or a waiver of consent.
- The data custodian's own data-access and data-sharing agreements.

### 11.2 The governance gate (G-0): the pack
1. HREC approval (or a documented exemption) and the approved protocol.
2. Data-sharing or access agreement with the custodian.
3. Privacy impact assessment.
4. A de-identification plan (§11.4) and a re-identification risk assessment.
5. Security plan: hosting in an **Australian region**; encryption; access control; audit; incident response.
6. Data flow diagram, showing that **no patient data leaves the governed environment** (rule P2).
7. Retention and destruction schedule.
8. Named roles: custodian, privacy officer, data steward, technical lead.

### 11.3 Security controls
- Encryption at rest and in transit.
- Role-based access with least privilege. Separate environments for identifiable staging and de-identified
  analytics.
- Complete audit logs: who accessed what, when, and why.
- **Network egress denied by default** from the governed environment.
- Use a recognised baseline: the Australian Signals Directorate's Essential Eight for system hardening, and ISO/IEC
  27001-aligned information security management where the custodian requires it.

### 11.4 De-identification
Follow the OAIC and CSIRO Data61 **De-identification Decision-Making Framework** (2017). It treats de-identification
as a risk judgement about the data **and its environment**, not a one-off transformation.
- **Pseudonymous `person_id`:** a keyed hash of the source identifier, with the key held by the custodian or data
  steward, outside the analytic environment.
- **Direct identifiers** (names, addresses, Medicare numbers, IHIs, contact details) are removed from analytic copies.
- **Quasi-identifiers** (dates, postcodes, rare conditions) are handled by the plan: date shifting per person
  (preserving intervals), postcode generalisation, suppression of small cells in outputs.
- Output controls: minimum cell sizes (for example no counts below 5 in exported aggregates), and a review of outputs
  before release. This follows the **Five Safes** model of safe people, projects, settings, data and outputs.

### 11.5 Synthetic data
The synthetic cohorts (CSIRO Australian Synthea, CC BY 4.0; US Synthea) are for development and testing. Record the
attribution. **Never present synthetic results as evidence of clinical performance**, and don't calibrate thresholds
on synthetic data alone.

---

## 12. Data quality and validation

### 12.1 Framework
Use the harmonised data-quality framework of Kahn et al. (2016):
- **conformance:** values fit formats, ranges and relationships;
- **completeness:** presence of expected data;
- **plausibility:** believable values and distributions.

Each is assessed by **verification** (against internal expectations) and **validation** (against external truth).

### 12.2 Automated checks
- **DataQualityDashboard** (OHDSI) runs its configurable library of data-quality checks against the CDM, grouped by
  the Kahn categories, with pass or fail per check against thresholds (Blacketer et al., JAMIA 2021).
  - Start from the defaults. Set per-check thresholds in `dq/thresholds.csv`, each with a written rationale.
  - **A load fails** if any check marked `fatal` fails, or the overall pass rate drops below the previous load's
    beyond a set tolerance.
- **Achilles** characterisation after each load: distributions, counts and heel warnings, reviewed for anomalies.
- **Custom checks:**
  - no refuted conditions counted as present;
  - no events outside the person's observation period;
  - measurement values plausible per LOINC code (ranges from the release where available);
  - era construction invariants (eras don't overlap for the same concept).

### 12.3 Mapping coverage
Per domain: the share of events with a non-zero standard concept, and the split by `concept_tier`. Example starting
targets: drugs ≥ 98%, measurements ≥ 98%, conditions ≥ 95%, procedures ≥ 90%. Set per source in the intended-use
document. Report the top unmapped source values (§8.4).

### 12.4 Round-trip tests
For sampled synthetic patients: source → events → OMOP → back out as FHIR. Key facts must survive: codes, dates at
their precision, values with units, statuses. Every loss is either documented as intended or treated as a defect.

### 12.5 Deduplication validation (hand check)
- **Merge precision:** draw **80 merges** at random (seeded). Readers look at the source records and judge "same
  event" or not, per Appendix A. **Gate: Wilson lower bound ≥ 0.90**, which needs **at least 78 of 80** correct. A
  wrong merge corrupts a timeline, so the bar is higher than elsewhere.
- **Boundary non-merges:** draw 80 candidate pairs that were **not** merged but fell inside the blocking window, and
  judge whether they should have been. Report the missed-merge rate with its interval. Use it to tune the windows,
  never the precision bar.

### 12.6 Journey review
Clinicians read 20 full synthetic timelines for plausibility, ordering and completeness. Every defect becomes a test
case.

---

## 13. Cohorts and phenotypes

### 13.1 Concept sets
Define concept sets as **SNOMED expression-constraint (ECL) expressions or explicit lists**. Expand them against the
pinned release's hierarchy (or an external FHIR terminology service at the same edition). Record the expansion's
**SHA-256** in the cohort registry. A cohort is reproducible only if its expansion hash is unchanged.

### 13.2 Cohort definitions
Write them in ATLAS (Circe JSON) or SQL, each with:
- the entry event;
- inclusion rules;
- the exit strategy;
- the concept sets (with their hashes);
- the T0 release;
- the author, reviewer and version.

### 13.3 Phenotype evaluation
- **CohortDiagnostics:** incidence, index-event breakdown, visit context, orphan codes (codes that should be in the
  set but aren't), and overlap with related cohorts.
- **PheValuator:** estimated sensitivity, specificity and PPV using a probabilistic reference, **or** chart review
  of a random sample (Appendix A), giving PPV with its Wilson interval.
- **Report every phenotype** with its performance estimates and their method. Don't use a phenotype for analysis until
  its PPV is reported.

### 13.4 Edition changes
When the T0 release changes, re-expand every registered concept set. If a hash changes, the cohort is marked
**changed**: the diff (codes added and removed) is reviewed, and the phenotype is re-evaluated before further use.

---

## 14. Clinical notes (later phase)

This phase only starts after the governance gate, and only inside the governed environment.
- **Extraction targets:** conditions, medications, findings, with **assertion** (present, absent, possible,
  hypothetical, historical, about someone else). The negation and context algorithms NegEx and ConText remain useful
  baselines.
- **Normalisation:** extracted mentions go through the same normaliser (§8), with a mention-level confidence. Events
  from notes carry `provenance.source_kind = note` and are **never** merged into coded events without the same merge
  rules (§10.2).
- **Evaluation:**
  - an annotated gold set with double annotation and κ reported;
  - precision, recall and F1 per entity type and assertion class;
  - normalisation accuracy@1.
- **Models:** any model used runs inside the governed environment (rule P2). No external API receives note text.

---

## 15. Serving

- **Timeline API:** `GET /persons/{id}/timeline?from&to&domains&include_merged=false`. Returns journey events and
  episodes, each with concept, tier, precision and lineage ids. Governed users only.
- **Cohort API:** `POST /cohorts/{id}/count`, `GET /cohorts/{id}/definition`. Counts are subject to the output
  controls (§11.4).
- **Quality API:** the latest DataQualityDashboard summary, coverage and unmapped queue, for data stewards.
- **Display rule (P6):** clinician-facing views label any concept whose mapping isn't backed by `edge_clinical`.
- **Audit:** every call is logged with user, purpose and parameters.

---

## 16. Scorecard

| # | Metric | Gate |
|---|---|---|
| Q1 | DataQualityDashboard fatal checks | **0 failures** |
| Q2 | DataQualityDashboard overall pass rate | not below the previous load beyond tolerance |
| Q3 | mapping coverage per domain | meets target (§12.3) |
| Q4 | merge precision | Wilson lower bound **≥ 0.90** (≥ 78/80) |
| Q5 | missed-merge rate at the boundary | reported with its interval |
| Q6 | round-trip loss | only documented, intended losses |
| Q7 | cohort reproducibility | 100% of registered cohorts reproduce by hash on an unchanged release |
| Q8 | phenotype PPV | reported with an interval for every phenotype in use |
| Q9 | normalisation quiz items | all pass |
| Q10 | governance (real data) | pack approved; audit logs complete; no egress violations |

---

## 17. Operations

### 17.1 Pipeline
The orchestrated DAG: ingest → stage → normalise → events → deduplicate → episodes → OMOP projection → quality
checks → publish. It runs incrementally, with idempotent steps. Each run writes a manifest: sources and checksums, T0
release, code versions, parameters (windows, gaps), check results.

### 17.2 On each T0 release
1. Re-normalise.
2. Diff concept assignments.
3. Re-expand cohort concept sets and flag hash changes.
4. Re-run the quality checks.

Publish release notes for data users.

### 17.3 Monitoring
- load duration and failure rate;
- DataQualityDashboard pass rate over time;
- coverage per domain;
- unmapped queue size;
- merge rate per domain (a sudden change means a feed or rule changed);
- access anomalies.

### 17.4 Incidents
- **Severity 1:** a privacy breach, or suspected re-identification. Follow the custodian's incident process and the
  notifiable data breach obligations under the Privacy Act.
- **Severity 2:** a published load found to be wrong. Withdraw it, correct it, re-publish with notes.

---

## 18. Risk and failure-mode register

| # | Risk | Control | Detected by |
|---|---|---|---|
| J1 | wrong merges fuse different events | subsumption plus attribute compatibility; laterality guard; high precision bar | Q4 |
| J2 | concept 0 records hide gaps | coverage reporting; the unmapped queue | Q3 |
| J3 | cohorts drift with vocabulary editions | expansion hashes; re-evaluation on change | Q7 |
| J4 | invented time precision distorts sequences | explicit `time_precision` | custom checks; review |
| J5 | refuted or entered-in-error facts counted | status handling | custom checks |
| J6 | patient data leaves the environment | egress rules; architecture review; rule P2 | audit; network logs |
| J7 | re-identification through quasi-identifiers | De-identification Decision-Making Framework plan; output controls | risk assessment; output review |
| J8 | synthetic data flatters the pipeline | real-data validation at G-0; no clinical claims from synthetic data | release checklist |
| J9 | unit errors in measurements | UCUM normalisation only through conversion edges; plausibility ranges | custom checks |
| J10 | vendor FHIR variations break adapters | per-vendor test payloads; contract tests | ingest failures |

---

## 19. Roles and effort

| Role | Responsibilities |
|---|---|
| Pipeline lead | scope, gate sign-off |
| Data engineer | adapters, normaliser, ETL, orchestration |
| Clinical informatician | event model, ETL specification, unmapped queue, merge rules |
| Clinical reviewers (at least 2) | merge hand checks, journey review, phenotype chart review |
| Privacy officer and data steward | governance pack, de-identification, access |
| Custodian (external) | data access, approvals |

**Indicative effort** (synthetic phase):

| Stage | Effort |
|---|---|
| J1 | 1 week |
| J2 | 2 weeks |
| J3 | 2 weeks |
| J4 | 2 weeks |
| J5 | 2 weeks, plus about 10 reviewer-hours |
| J6 | 1–2 weeks |
| J7 | 1 week |

G-0 elapsed time depends on HREC and custodian timelines: plan for months, not weeks.

---

## 20. Prompt library

These are for engineering assistants working on synthetic or de-identified material only. Never paste patient data
into any external tool.

**20.1 ETL specification review.**
> Review `docs/etl-spec.md` against GO-PJI `PLAYBOOK.md` §6–§9. For each source element, check: target table and
> column, standard-concept route, time-precision handling, status handling, lineage, and a test case. List gaps.

**20.2 Merge-rule red team.**
> Propose 30 pairs of clinical records that the merge rules in §10.2 would wrongly merge, or wrongly keep apart
> (laterality, body site, specimen, certainty, unit, timing). For each, give the expected correct outcome and the rule
> change or test that would catch it.

**20.3 Data-quality threshold rationale.**
> For each DataQualityDashboard check we override in `dq/thresholds.csv`, write a one-line rationale tied to the
> source's known characteristics, and flag any threshold looser than the default without justification.

**20.4 Governance pack check.**
> Check the governance pack against §11.2. List missing items and any data flow that could move patient data outside
> the governed environment.

---

## 21. Decisions to make

| # | Decision | Recommended default |
|---|---|---|
| JD1 | source systems for v1 | synthetic FHIR (AU Core) and openEHR |
| JD2 | orchestrator | Dagster, or the team's existing standard |
| JD3 | merge windows | as in §10.2; tune only on the missed-merge rate, never the precision bar |
| JD4 | era persistence windows | 30 days for conditions and drugs, recorded per run |
| JD5 | small-cell threshold for outputs | 5 |
| JD6 | external terminology service for ECL | only if needed; otherwise expand locally from the pinned release |
| JD7 | first real-data custodian | to be identified; this starts the G-0 timeline |

---

## 22. References

**Checked when this playbook was written (24 Sep 2026):**
- Blacketer, DeFalco, Ryan & Rijnbeek, "Increasing trust in real-world evidence through evaluation of observational
  data quality", *JAMIA* 28(10):2251–2257, 2021: DataQualityDashboard ([OUP](https://academic.oup.com/jamia/article/28/10/2251/6328963); [code](https://github.com/OHDSI/DataQualityDashboard))
- OAIC and CSIRO Data61, *The De-identification Decision-Making Framework*, September 2017 ([OAIC](https://www.oaic.gov.au/privacy/privacy-guidance-for-organisations-and-government-agencies/handling-personal-information/de-identification-decision-making-framework))
- HL7 Australia, AU Core FHIR Implementation Guide, current version 2.0.0 (R2) ([hl7.org.au](https://hl7.org.au/fhir/core/)); Sparked AU FHIR Accelerator ([Sparked](https://sparked.csiro.au/index.php/products-resources/au-core-fhir-ig-r1/))
- OHDSI tools: [Achilles](https://github.com/OHDSI/Achilles), [PheValuator](https://github.com/OHDSI/PheValuator), [CohortDiagnostics](https://github.com/OHDSI/CohortDiagnostics)

**From the author's knowledge; not re-checked when written:**
- OHDSI, *The Book of OHDSI* (2019+); OMOP CDM v5.4 specification; THEMIS conventions; the era-building conventions
  (30-day persistence windows).
- Kahn et al., "A Harmonized Data Quality Assessment Terminology and Framework for the Secondary Use of Electronic
  Health Record Data", *eGEMs* 2016.
- Swerdel et al., "PheValuator: Development and evaluation of a phenotype algorithm evaluator", *J Biomed Inform*
  2019.
- Chapman et al., NegEx (2001); Harkema et al., ConText (2009).
- The Privacy Act 1988 and the APPs; the Health Records Act 2001 (Vic); the HRIP Act 2002 (NSW); the Health Records
  (Privacy and Access) Act 1997 (ACT); the My Health Records Act 2012; the NHMRC National Statement on Ethical Conduct
  in Human Research.
- The Five Safes framework (as used by the ABS and AIHW); ASD Essential Eight; ISO/IEC 27001.
- openEHR specifications; EHRbase; the openEHR Clinical Knowledge Manager; CSIRO Australian Synthea collection
  (CC BY 4.0).

---

## Appendix A: Hand-check protocol

**A.1 Sampling.** A simple random sample with a recorded seed. Stratify by domain or source when they differ, and
weight back. Targeted reads never count toward a grade.

**A.2 Sample size and grades.** The grade is the 95% Wilson lower bound:
lower bound = (p + z²/2n − z·√(p(1−p)/n + z²/4n²)) / (1 + z²/n), with z = 1.96.

| Target | Need |
|---|---|
| any grade | n ≥ 30 |
| lower bound ≥ 0.80 | at n = 80, at least 72 correct |
| **lower bound ≥ 0.90** (merge precision) | at n = 80, **at least 78 correct** (77 fails) |
| lower bound ≥ 0.99 | at least 381 read with zero errors |

Fix n before reading. No optional stopping.

**A.3 Reading rules.**
- Read the **source records**, not the pipeline's output alone.
- Blind to model or rule confidence.
- Verdicts: `correct`, `wrong`, `ambiguous` (counts as wrong), with a reason for every wrong or ambiguous.
- A second reader covers 20%; κ ≥ 0.8 is required; a third person adjudicates.
- **Reviewers work inside the governed environment when real data is involved.**

**A.4 Evidence file.**
- sample id, version, seed, population, number drawn;
- per row: item id (pseudonymous), verdict, reason, readers, adjudication, date;
- κ, number correct, Wilson lower bound, outcome.

No identifiable data.

---

## Appendix B: Templates

**B.1 `docs/use-cases.md`:** the questions the journeys must answer; their users; required domains and time
precision; the display rules for each use.

**B.2 ETL specification** (per source): element; target table and column; transform; vocabulary route; time
precision; status handling; test case; reviewer.

**B.3 Run manifest:** run id; sources and checksums; adapter versions; T0 release; normaliser version; parameters
(windows, gaps); data-quality results; coverage; counts per table.

**B.4 Cohort registry entry:** id; name; version; author and reviewer; definition file; concept sets with ECL and
expansion SHA-256; T0 release; phenotype evaluation results; status (active, changed, retired).

**B.5 Real-data release checklist:**
- G-0 approvals attached;
- de-identification plan applied and verified;
- egress test passed;
- all §16 gates passed on real data;
- output controls configured;
- audit logging verified;
- custodian notified.
