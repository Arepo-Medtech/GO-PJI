# GO-PJI: Graph-Ontology → Patient Journey Intelligence

Track **T3** of the Graph-Ontology next phases. Longitudinal, normalised patient timelines on SPINE (openEHR, FHIR, OMOP CDM), linked to Graph-Ontology concepts, deduplicated into episodes and queryable as cohorts.

## Start here
1. **[T0.md](T0.md)**: the shared foundations, identical in all four track repos (GO-Harness, GO-HGT, GO-PJI, GO-TS):
   frozen snapshots, the licence matrix, algebraic properties on predicates, the validation register, shared views,
   the SSSOM export and the quiz suite. This track consumes a T0 release; it never reads the live graph.
2. The full recipe for this track is in `COMPENDIUM.md` (track T3), in
   [Arepo-Medtech/graph-ontology-compendium](https://github.com/Arepo-Medtech/graph-ontology-compendium).
3. Method (rules R1–R17, validation layers L0–L6) is governed by `PLAYBOOK.md` in
   [Arepo-Medtech/one-shot](https://github.com/Arepo-Medtech/one-shot).

## How this track uses T0
- normalises source codes against the snapshot (through GO-TS), recording each event's mapping tier from the register;
- uses `equivalence_group` for representatives and the is-a closure for deduplication by subsumption;
- shows clinicians only concepts backed by `edge_clinical`;
- is tested on the quiz suite's normalisation items;
- needs hosting and display permissions in the licence matrix; real patient data only after governance approval.

## Pinning a release
Copy `t0.lock.example` to `t0.lock` once the first T0 release exists, and fill in its id and fingerprint (T0 §9).

Status: plan only. Nothing here has been built or run.
