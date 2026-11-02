# GO-PJI: Graph-Ontology → Patient Journey Intelligence

Track **T3** of the Graph-Ontology next phases. Longitudinal, normalised patient timelines on SPINE (openEHR, FHIR, OMOP CDM), linked to Graph-Ontology concepts, deduplicated into episodes and queryable as cohorts.

## Start here
This repository is self-contained. Two documents hold everything this track needs:

1. **[PLAYBOOK.md](PLAYBOOK.md)**: the authoritative playbook for this track. It covers mission, rules, architecture,
   stages and gates, engineering guidance, evaluation, scorecard, operations, governance, risks, prompts and
   references, with the hand-check protocol and templates as appendices.
2. **[T0.md](T0.md)**: the shared foundations this track consumes (frozen snapshots, the licence matrix, predicate
   properties, the validation register, shared views, the SSSOM export and the quiz suite). It is identical across
   the four Graph-Ontology track repositories.

## How this track uses T0
- normalises source codes against the snapshot (through GO-TS), recording each event's mapping tier from the register;
- uses `equivalence_group` for representatives and the is-a closure for deduplication by subsumption;
- shows clinicians only concepts backed by `edge_clinical`;
- is tested on the quiz suite's normalisation items;
- needs hosting and display permissions in the licence matrix; real patient data only after governance approval.

## Pinning a release
`t0.lock` pins Release 2, `GO-20261103-4e9f3709` (3 Nov 2026), the first full T0 release: every gate passed (docs/t0.md in Graph-Ontology). Upgrade by bumping `t0.lock` in a pull request that carries the release diff and this track's regression results (T0 §9).

Status: plan only. Nothing here has been built or run.
