---
id: AISDLC-282
title: 'feat: RFC-0016 Phase 4 — Stage B LLM tie-breaker + Q5 ensemble'
status: Done
assignee: []
created_date: '2026-05-15'
labels:
  - rfc-0016
  - estimation-calibration
  - phase-4
  - critical-path-rfc-0035
dependencies:
  - AISDLC-281
references:
  - spec/rfcs/RFC-0016-estimation-calibration-tshirt-sizes.md
priority: high
---

## Description

<!-- SECTION:DESCRIPTION:BEGIN -->
Phase 4 of RFC-0016 Implementation Plan (§13). Stage B is the LLM-as-last-resort tier. Only invoked when Stage A escalates per §5.2 OR same-hash variance ≥2 buckets per §8.4.

## Scope

- Stage B prompt builder with full Stage A signal table passed as context per §6.1
- Escalation gate: only invoke when Stage A reports non-adjacent buckets OR variance ≥2 across the same `estimateInputHash`
- Ensemble batch aggregation writes `estimateVariance` per hash transition
<!-- SECTION:DESCRIPTION:END -->

## Acceptance Criteria

<!-- AC:BEGIN -->
- [ ] #1 Stage B only invoked under §5.2 escalation conditions OR variance ≥2
- [ ] #2 Stage A signal table passed as context per §6.1
- [ ] #3 Returns one bucket or 2-bucket range with justification string
- [ ] #4 `estimateVariance` per hash transition recorded
- [ ] #5 Q5 ensemble aggregation across multiple LLM calls (batch mode)
- [ ] #6 Stage B call rate stays below 30% of total estimates (telemetry metric)
<!-- AC:END -->
