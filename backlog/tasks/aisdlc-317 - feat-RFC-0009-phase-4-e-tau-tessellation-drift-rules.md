---
id: AISDLC-317
title: 'feat: RFC-0009 Phase 4.2 — Eτ_tessellation_drift rules #1 (AST scan) + #3 (cross-soul provenance)'
status: To Do
assignee: []
created_date: '2026-05-16'
labels:
  - rfc-0009
  - tessellated-did
  - phase-4
  - drift-detection
dependencies:
  - AISDLC-313
  - AISDLC-315
references:
  - spec/rfcs/RFC-0009-tessellated-design-intent-documents.md
  - spec/rfcs/RFC-0019-embedding-provider-adapter.md
priority: medium
---

## Description

<!-- SECTION:DESCRIPTION:BEGIN -->
Phase 4.2 of RFC-0009. Eτ_tessellation_drift detects design coherence drift across tessellated souls. Per OQ-6 resolution: detection is orchestrator-side, not in-pipeline.

## Scope (RFC-0009 §10 Phase 4, §7.2 Eτ_tessellation_drift)

- Eτ_tessellation_drift **rule #1 (AST scan)** activates orchestrator-side per §7.2 + OQ-6. Detects design-coherence drift by static-analysis pass over soul-imports.
- Eτ_tessellation_drift **rule #3 (cross-soul provenance audits)** activates once the §8.3 ProvenanceRecord extension lands (AISDLC-315) and tessellated provenance accumulates.
- Eτ_tessellation_drift **rule #2 (embedding distance)** is explicitly DEFERRED to RFC-0019 implementation — NOT in scope for this task.
- Drift events emit to `events.jsonl` via RFC-0015 substrate.
- Adopter opt-in gate respected (default off; per §10 Phase 4 promotion convention).
<!-- SECTION:DESCRIPTION:END -->

## Acceptance Criteria

<!-- AC:BEGIN -->
- [ ] #1 Eτ_tessellation_drift rule #1 (AST scan) ships orchestrator-side
- [ ] #2 Eτ_tessellation_drift rule #3 (cross-soul provenance audits) ships, gated on §8.3 ProvenanceRecord availability (AISDLC-315 dependency)
- [ ] #3 Drift events emitted to events.jsonl
- [ ] #4 Rule #2 (embedding distance) explicitly NOT shipped — deferred to RFC-0019
- [ ] #5 Adopter opt-in gate respected (default off)
- [ ] #6 Test coverage: rule #1 AST scan / rule #3 provenance audit / no-drift baseline / opt-out short-circuits
<!-- AC:END -->
