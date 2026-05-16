---
id: AISDLC-319
title: 'feat: RFC-0009 Phase 4.4 — DatabaseBranchPool shared+RLS default + Operator role platform-scoping wiring'
status: To Do
assignee: []
created_date: '2026-05-16'
labels:
  - rfc-0009
  - tessellated-did
  - phase-4
  - infrastructure
dependencies:
  - AISDLC-315
references:
  - spec/rfcs/RFC-0009-tessellated-design-intent-documents.md
  - spec/rfcs/RFC-0022-compliance-posture-audit-surface.md
priority: medium
---

## Description

<!-- SECTION:DESCRIPTION:BEGIN -->
Phase 4.4 of RFC-0009. Bundles two small wiring tasks: DatabaseBranchPool default + Operator role scoping. Per OQ-10 + OQ-11 resolutions.

## Scope (RFC-0009 §10 Phase 4, §8.7 DatabaseBranchPool + §8.8 Operator role)

### DatabaseBranchPool (OQ-11 resolution)

- Default = shared+RLS per §8.7.
- `init` wizard walks the trigger checklist for per-soul opt-in:
  - Regulatory hard requirement
  - Customer contract requirement
  - Operator security review
- RFC-0022 (Compliance Posture) declarations drive the gate automatically when adopters use it.
- Per-soul opt-in upgrades pool to per-soul-branch when any trigger fires.

### Operator role wiring (OQ-10 resolution)

- Confirm Operator role is **platform-scoped, not tessellated**.
- No soul-vertex Operator field shipped (explicit OQ-10 outcome).
- Existing AgentRole + platform-Operator wiring preserved.

## Why bundled

Both are small operator-touchable wiring tasks (config schema + init wizard prompts). Combining them into one task is more efficient than two separate small PRs.
<!-- SECTION:DESCRIPTION:END -->

## Acceptance Criteria

<!-- AC:BEGIN -->
- [ ] #1 DatabaseBranchPool default = shared+RLS per §8.7
- [ ] #2 `init` wizard walks the 3-trigger checklist (regulatory / contract / operator review)
- [ ] #3 RFC-0022 declarations drive gate automatically when adopter has them
- [ ] #4 Per-soul opt-in upgrades pool to per-soul-branch when any trigger fires
- [ ] #5 Operator role confirmed platform-scoped per OQ-10; no soul-vertex Operator field shipped
- [ ] #6 Test coverage: default shared+RLS / trigger-fires-per-soul-upgrade / Operator-stays-platform-scoped
<!-- AC:END -->
