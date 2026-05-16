---
id: AISDLC-315
title: 'feat: RFC-0009 Phase 3 — AgentRole + AdapterBinding + ProvenanceRecord + QualityGate soul-scoping'
status: To Do
assignee: []
created_date: '2026-05-16'
labels:
  - rfc-0009
  - tessellated-did
  - phase-3
  - resource-extensions
dependencies:
  - AISDLC-313
references:
  - spec/rfcs/RFC-0009-tessellated-design-intent-documents.md
  - spec/rfcs/RFC-0017-in-soul-variant-pattern.md
  - spec/rfcs/RFC-0018-in-soul-journey-pattern.md
priority: high
---

## Description

<!-- SECTION:DESCRIPTION:BEGIN -->
Phase 3 of RFC-0009. Four framework resource types gain soul-scoping fields per §8. In-soul variation patterns (variant + journey) are explicitly OUT OF SCOPE — see RFC-0017 (In-Shard Variant Pattern) and RFC-0018 (In-Shard Journey Pattern), both Reserved, pending normative spec.

## Scope (RFC-0009 §10 Phase 3, §8 resource extensions)

- `AgentRole` resource gains `soulScope` field per §8.1: per-role scope to a Soul DID slug or `platform`.
- `AdapterBinding` resource gains `soulScope` field per §8.2: per-binding scope.
- `ProvenanceRecord` resource gains `soulScope` + `tessellatedSoulRef` fields per §8.3: provenance traces per soul.
- `QualityGate` resource gains `soulScope` field per §8.4: per-gate scope.
- All four resources remain backwards-compatible (omitted soulScope = platform-wide, existing behavior).
- In-soul variation patterns NOT shipped here (RFC-0017 + RFC-0018 carve-outs).
<!-- SECTION:DESCRIPTION:END -->

## Acceptance Criteria

<!-- AC:BEGIN -->
- [ ] #1 `AgentRole` schema extended with `soulScope` field per §8.1
- [ ] #2 `AdapterBinding` schema extended with `soulScope` field per §8.2
- [ ] #3 `ProvenanceRecord` schema extended with `soulScope` + `tessellatedSoulRef` per §8.3
- [ ] #4 `QualityGate` schema extended with `soulScope` field per §8.4
- [ ] #5 Backwards-compat: all four resources work with omitted `soulScope` (= platform-wide; existing behavior preserved)
- [ ] #6 RFC-0017 + RFC-0018 carve-outs explicitly NOT in scope; cross-referenced in implementation notes
- [ ] #7 Test coverage: soul-scoped + platform-wide + mixed-scope scenarios per resource
<!-- AC:END -->
