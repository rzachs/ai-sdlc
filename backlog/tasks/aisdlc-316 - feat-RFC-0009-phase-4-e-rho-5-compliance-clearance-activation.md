---
id: AISDLC-316
title: 'feat: RFC-0009 Phase 4.1 — Eρ₅ Compliance Clearance activation (OQ-5 hard-regulatory scope)'
status: To Do
assignee: []
created_date: '2026-05-16'
labels:
  - rfc-0009
  - tessellated-did
  - phase-4
  - compliance
dependencies:
  - AISDLC-313
  - AISDLC-315
references:
  - spec/rfcs/RFC-0009-tessellated-design-intent-documents.md
  - spec/rfcs/RFC-0022-compliance-posture-audit-surface.md
priority: medium
---

## Description

<!-- SECTION:DESCRIPTION:BEGIN -->
Phase 4.1 of RFC-0009. Eρ₅ Compliance Clearance sub-dimension activates when souls declare `complianceRegimes` against the hard-regulatory-only scope (per OQ-5 sub-decision). Composes with RFC-0022 (Compliance Posture — currently Draft) for the canonical regime-declaration surface.

## Scope (RFC-0009 §10 Phase 4, §7.1 Eρ₅)

- Souls can declare `complianceRegimes` field per §7.1.
- Scope is hard-regulatory-only (OQ-5 resolution): HIPAA, SOC2, PCI-DSS, GDPR, etc. Soft / advisory regimes deferred.
- Eρ₅ sub-dimension evaluates compliance-clearance against declared regimes during admission.
- Gated on adopter opt-in initially; promotion to default behavior subject to ecosystem feedback.
- RFC-0022 (Compliance Posture, Draft) provides the canonical regime-declaration surface — this task wires the consumption side.
<!-- SECTION:DESCRIPTION:END -->

## Acceptance Criteria

<!-- AC:BEGIN -->
- [ ] #1 Souls can declare `complianceRegimes` field with the hard-regulatory whitelist per §7.1 + OQ-5
- [ ] #2 Eρ₅ sub-dimension evaluates clearance against declared regimes during admission
- [ ] #3 Adopter opt-in gate respected (default off)
- [ ] #4 RFC-0022 consumption surface wired
- [ ] #5 Test coverage: hard-regime opt-in / opt-out / soft-regime (rejected at declaration time per OQ-5 scope)
<!-- AC:END -->
