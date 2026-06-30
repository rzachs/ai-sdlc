# AI-SDLC Project Instructions

**Scope:** AI-SDLC is a full autonomous SDLC framework — autonomous orchestrator (RFC-0015), cross-harness review (RFC-0010 §13), decision engine (RFC-0011 DoR), operator TUI (RFC-0023), Pattern-C worktree isolation, and declarative governance. The `ai-sdlc-plugin/` package is the Claude Code plugin pillar; `pipeline-cli/` is the Step 0-13 pipeline runtime; `orchestrator/` is the CLI and agent runner layer.

## Git Flow

- **Always rebase** feature branches onto main; never merge main in.
- Update branch: `git fetch origin && git rebase origin/main`, then `git push --force-with-lease`.
- Never `gh api pulls/N/update-branch` with merge method. Keep linear history.
- `/ai-sdlc rebase <pr>` automates mechanical conflicts (test additions to same `describe`, prettier drift) and re-signs the attestation only when `contentHash` changed. Escalates semantic conflicts, modify-vs-delete, verification failures, and 3-attempt iteration cap. Refuses force-push to `main`/`master`. **CHANGELOG.md conflicts should not arise on feature branches** — if a rebase surfaces one, remove the CHANGELOG change from the feature branch rather than merging both sides (AISDLC-401).

## CI marker hygiene

GitHub Actions silently skips ALL workflows when ANY commit body contains `[skip ci]`, `[ci skip]`, `[no ci]`, `[skip actions]`, or `[actions skip]` (substring, case-insensitive). Use the paren-quoted form `(skip ci marker)` in commit messages. Backtick-wrapping does NOT defeat the parser. `scripts/check-skip-ci-marker.sh` enforces on push.

## Branches & Commits

- Branches: `feat/<desc>`, `fix/<desc>`, or `ai-sdlc/issue-<n>`.
- Conventional commits (`feat:`, `fix:`, `test:`, `docs:`, `chore:`, `style:`).
- Include `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>`.

## PRs

- **Never merge PRs** — only humans merge.
- **Never close** issues or PRs. **Never force-push to main/master.**
- Dismiss stale reviews only with documented reason (truncation, API errors).
- `auto-enable-auto-merge.yml` sets `--auto --squash` on same-repo PRs. Setting `--auto` is NOT merging. PRs merge directly once `ai-sdlc/pr-ready` + `Backlog Drift` required checks pass. See `docs/operations/merge-without-queue.md`.

## Testing

- Run `pnpm build && pnpm test && pnpm lint && pnpm format:check` before pushing.
- `.husky/pre-push` is the canonical gate; local pre-flight makes it a no-op.
- Hook scripts (`ai-sdlc-plugin/hooks/*.js`) use Node built-in `node --test`. Orchestrator + MCP server use Vitest.
- `@ai-sdlc/dogfood` tests (`dogfood/src/runner/exports.test.ts`) import from `dist/runner/index.js` to validate the built exports surface. The `pretest` lifecycle hook in `dogfood/package.json` runs `pnpm build` automatically before `pnpm test`, so `pnpm --filter @ai-sdlc/dogfood test` always works. Do NOT remove the `pretest` hook — it prevents CI failures on PRs where dogfood is selected by the `...[origin/main]` test filter but dist wasn't explicitly built (AISDLC-404).

## Hooks

`.husky/pre-push` chains in order:

1. **`scripts/check-coverage.sh`** — 80% lines coverage threshold per package. Skip: `AI_SDLC_SKIP_COVERAGE_GATE=1`.
2. **`scripts/squash-attestation-chores.sh`** — squashes stacked `chore: sign attestation` commits at HEAD into one to keep history clean. Must run before attestation-sign. No-op when 0 or 1 such commits. Skip: `AI_SDLC_SKIP_SQUASH_CHORES=1`.
3. **`scripts/check-dor-gate.sh`** — runs `cli-dor-check --task <path>` against every `backlog/{tasks,completed}/*.md` file changed in the push range, forcing `evaluationMode: enforce` so violations BLOCK locally even when the repo's `dor-config.yaml` is `warn-only`. When `pipeline-cli/dist/cli/dor-check.js` is missing AND the push touches backlog task files, FAILS LOUD with a build instruction (`pnpm --filter @ai-sdlc/pipeline-cli build`). When the push has NO task changes and dist is missing, silently exits 0. Skip: `AI_SDLC_SKIP_DOR_GATE=1`. AISDLC-370.
4. **`scripts/check-backlog-drift-on-push.sh`** (AISDLC-486) — catches error-severity Backlog Drift BEFORE push. Two checks: (a) inbound-reference scan — for every file renamed/moved/deleted in the push range, greps `backlog/` for references to the OLD path; (b) task-level drift scan — runs `npx backlog-drift check --since <merge-base>` on tasks touched in this push. Runs AFTER DoR and BEFORE the fixup orchestrator. Skip: `AI_SDLC_SKIP_BACKLOG_DRIFT_PUSH_GATE=1`. Hermetic tests: `scripts/check-backlog-drift-on-push.test.mjs` (run via `pnpm test:backlog-drift-push-gate`).
5. **`scripts/pre-push-fixups.sh`** (AISDLC-386) — orchestrates task-move → attestation-sign in dependency order in a single pass. Each sub-hook invoked with `AI_SDLC_INTERNAL_NO_EXIT_1=1`. If any fixup ran, exits 1 once with "re-run git push". Exit 0 silently when no fixups needed.
6. **`scripts/check-task-moved.sh`** (defense-in-depth) — auto-moves backlog task file from `backlog/tasks/` to `backlog/completed/` when any commit in the push range has `(AISDLC-N)` in its subject. Commits as `chore: auto-close AISDLC-N (AISDLC-220)`. **Silent skip when file is already git-tracked in `backlog/completed/`** (AISDLC-402): uses `git ls-files` to check tracked state. **Order is load-bearing — MUST run BEFORE attestation-sign.** Skip: `AI_SDLC_SKIP_TASK_MOVE=1`.
7. **`scripts/check-attestation-sign.sh`** (defense-in-depth) — auto-signs DSSE attestation when `<worktree>/.active-task` exists, `<worktree>/.ai-sdlc/verdicts/<task-id-lower>.json` exists, and no envelope at HEAD. Exits 0 as no-op when no verdict file exists (docs-only PRs, chore commits, ad-hoc pushes). Skip: `AI_SDLC_SKIP_ATTESTATION_SIGN=1`.

**Master bypass (emergency only):** `AI_SDLC_BYPASS_ALL_GATES=1 git push` stops the entire pre-push chain. Use exclusively during RFC-0042 / gate-rewrite cutover windows; document every use in the PR body. Per-gate `AI_SDLC_SKIP_*` vars continue to work independently. See [`docs/operations/emergency-bypass.md`](docs/operations/emergency-bypass.md).

`set -euo pipefail` aborts on first failure. `git push --no-verify` bypasses everything. All gates have hermetic tests wired via `pnpm test:drift-gate` / `test:task-move-gate` / `test:dor-gate` / `test:attestation-sign-gate` / `test:pre-push-fixups-gate` / `test:backlog-drift-push-gate`.

## CI behavior

PR merge gate is the single rollup check `ai-sdlc/pr-ready` produced by `.github/workflows/ai-sdlc-gate.yml` (re-actors/alls-green pattern); see [`docs/operations/quality-gate.md`](docs/operations/quality-gate.md) for archetype gating, cutover, and rollback.

**Main health monitor** (AISDLC-406): `.github/workflows/main-health-monitor.yml` fires on every push to `main` and runs the full test suite (`pnpm -r test` + workflow YAML tests). When any test fails, it creates a GitHub issue titled `[main-health] main is RED at <commit>` assigned to `@deefactorial`. See [`docs/operations/main-health-monitor.md`](docs/operations/main-health-monitor.md).

Workflows MUST invoke pipeline-cli CLIs via `node pipeline-cli/bin/cli-XXX.mjs` directly — never via `pnpm --filter @ai-sdlc/pipeline-cli exec cli-XXX`. `pnpm exec` does not resolve workspace own-bins, so the latter form silently fails with `Command not found`. `pipeline-cli/src/cli/bin-invocation.test.ts` enforces both directions of this rule. See AISDLC-156 + the "Invoking from CI" section of `pipeline-cli/README.md`.

## Feature flags

- **`AI_SDLC_DEPS_COMPOSITION`** (RFC-0014): **On by default** (AISDLC-410). Opt out via `AI_SDLC_DEPS_COMPOSITION=off` (or `0`/`false`/`no`). Phase 1: `cli-deps snapshot/gc/inspect`. See [`docs/operations/deps-composition.md`](docs/operations/deps-composition.md) and [`pipeline-cli/docs/deps.md`](pipeline-cli/docs/deps.md).
- **`AI_SDLC_AUTONOMOUS_ORCHESTRATOR`** (RFC-0015): **On by default** (AISDLC-411). Opt out via `AI_SDLC_AUTONOMOUS_ORCHESTRATOR=off` (or `0`/`false`/`no`). Phase 1: `cli-orchestrator {start,tick,status}` via `node pipeline-cli/bin/cli-orchestrator.mjs`. See [`pipeline-cli/docs/orchestrator.md`](pipeline-cli/docs/orchestrator.md) and [`spec/rfcs/RFC-0015-autonomous-pipeline-orchestrator.md`](spec/rfcs/RFC-0015-autonomous-pipeline-orchestrator.md).

## Code Style

- TypeScript strict, ESM. Prettier + ESLint. No premature abstractions — three similar lines beat one wrong abstraction.

## Subagent Governance — Scope Creep Prevention (AISDLC-308)

**Agents must not auto-expand scope beyond the original ask.**

When a review / audit / read-only task surfaces work that would be useful to do next, the agent MUST:

1. **Present the recommendation** in the review output (PR body, task summary, comment).
2. **Stop.** Wait for explicit operator authorization before:
   - Filing new backlog tasks
   - Opening any PR beyond the original ask
   - Dispatching new subagents for downstream work
3. **Treat "Pre-work required" / "Pre-conditions" / "OQ walkthrough needed" prose as a HARD precondition.** If a task body or referenced RFC flags an unresolved OQ or walkthrough requirement, the agent MUST NOT proceed to dispatch implementation until the operator confirms the precondition is met.

Every scope expansion is a decision that belongs in the [Decision Catalog (RFC-0035)](spec/rfcs/RFC-0035-decision-catalog-operator-routing.md). Surface it there for operator routing — do not self-authorize.

### Reviewer gate (AISDLC-308)

The `code-reviewer` and `test-reviewer` subagents check for scope-creep candidates in every PR: if the PR BOTH (a) implements a "review" or "audit" task AND (b) creates new files under `backlog/tasks/`, it is flagged as **critical** with the message "scope-creep candidate — verify operator authorized task creation."

### Read-only agent constraint (AISDLC-308)

Agents whose role is read-only (exploration, audit, refinement review) MUST NOT use `Write`, `Edit`, task-create MCP tools, or dispatch downstream agents. These constraints are enforced in each agent's frontmatter `disallowedTools` list and are re-stated in the agent body as **Hard rules**.

### Subagent model defaults (AISDLC cost control)

Agent frontmatter pins model by role to prevent session-model bleed:

- `developer`, `code-reviewer`, `test-reviewer` → **sonnet** (cost-efficient for mechanical tasks)
- `security-reviewer` → **opus** (reasoning-heavy; the one role where Opus pays for itself)

On dispatch paths (`/ai-sdlc execute`, `/ai-sdlc orchestrator-tick`), code-review and test-review SHOULD be routed to the `-codex` variants (`code-reviewer-codex` / `test-reviewer-codex`) — Codex plan billing is zero Claude usage. Security review stays on the Claude-native `security-reviewer` at opus. Mechanical work (sign, reconcile, rebase) MUST NOT be wrapped in subagents.

## Subagent Governance — OQ-resolution prohibition (AISDLC-298)

**Dev subagents MUST NOT resolve RFC Open Questions inline during implementation.**

### What counts as inline OQ resolution

Any addition of a `**Resolution:**` (or `RESOLVED:` / `✅ RESOLVED`) marker to an RFC `## Open Questions` section by a developer subagent during task implementation. This includes picking an implementation approach and writing rationale into the RFC, removing or replacing an OQ bullet, or writing code that implicitly resolves an OQ without documenting the escalation.

### Required behavior: escalate, do not resolve

When a dev subagent encounters an open question that blocks or constrains implementation:

1. **Stop and escalate** — do not pick an approach and resolve the OQ inline
2. **Return `prUrl: null` with a `notes` field** explaining which OQ in which RFC is blocking and what options exist
3. **Do not write Resolution markers** into RFC bodies — that is exclusively the operator's role after a walkthrough

If an OQ is genuinely non-blocking (implementation can proceed without resolving it), proceed with a documented assumption in the PR body — not a Resolution marker in the RFC.

### RFC-0035 Decision Catalog (default-ON since AISDLC-392)

**Feature flag `AI_SDLC_DECISION_CATALOG` is default-ON.** File decisions with:

```bash
node pipeline-cli/bin/cli-decisions.mjs add --summary "<one-line>" --scope <area> --option "<id>:<description>"
node pipeline-cli/bin/cli-decisions.mjs list
```

To opt out: set `AI_SDLC_DECISION_CATALOG=off` (or `0`/`false`/`no`/`disabled`).

Dev subagents that hit an OQ-class architectural question during implementation should still escalate by returning `prUrl: null` per the protocol above. The Decision Catalog is for OPERATOR-side decision routing, not a license for dev subagents to resolve OQs in code.

### Reviewer gate (AISDLC-298)

A new `**Resolution:**` marker added by a developer in an RFC's `## Open Questions` section is a **critical** finding that blocks approval.

## Review attestations

**Attestation is required for code PRs.** `/ai-sdlc execute` runs three reviewer subagents locally and writes a DSSE envelope to `.ai-sdlc/attestations/<patch-id>.dsse.json`. Docs-only PRs (`spec/rfcs/**`, `docs/**`, `backlog/**`, root `*.md`) skip `verify-attestation.yml` via `paths-ignore` and need no envelope. `verify-attestation.yml` posts `ai-sdlc/attestation: success/failure` feeding into the `ai-sdlc/pr-ready` rollup.

**Envelope filename** is `<git-patch-id>.dsse.json` — content-addressed, survives conflict-free rebases without invalidation. Per-SHA legacy filenames written as compat bridge.

**Default schema: v6** (RFC-0042, complete as of AISDLC-409). New envelopes use the RFC-6962 Merkle-transcript model. v5 opt-out: `--schema-version v5` or `AI_SDLC_V5_LEGACY=1`. The verifier falls back to v5/v4/v3 for legacy envelopes — all historical PRs remain auditable.

**v6 head-binding survives rebase + chore commits** via two relaxations: (1) **Attestation-only descendant** (AISDLC-419) — diff between `subject.sha1` and HEAD touches ONLY `.ai-sdlc/attestations/`, `.ai-sdlc/transcript-leaves.jsonl`, and `.ai-sdlc/transcript-leaves/`; (2) **Tree-equivalent modulo attestation** (AISDLC-448) — `subject.sha1` not an ancestor of HEAD (rebase orphaned it) but source-tree byte-identical modulo those same paths. Both helpers in `scripts/verify-attestation.mjs` (`isAttestationOnlyDescendant`, `isTreeEquivalentModuloAttestation`); share `ATTESTATION_PATH_EXCLUSIONS`. Adding paths to either relaxation requires extending `ATTESTATION_PATH_EXCLUSIONS` in lockstep with `pipeline-cli/src/attestation/patch-id.ts:PATCH_ID_EXCLUSIONS` — asymmetric lists reproduce the AISDLC-421 hotfix class of bug.

**`CONTENTHASH_SHARED_CHURN_FILES`** excludes from both signer and verifier: `pnpm-lock.yaml`, `CHANGELOG.md`, `pipeline-cli/CHANGELOG.md`, `orchestrator/CHANGELOG.md`, `reference/src/core/generated-schemas.ts`. DO NOT add source files, test files, configs, `package.json`, or RFCs. `generated-schemas.ts` is the **only** sanctioned `.ts` source-file exception (AISDLC-342).

**Branch protection** (AISDLC-388 AC-2): `main` should require ONLY `ai-sdlc/pr-ready` and `Backlog Drift` — NOT `ai-sdlc/attestation` directly. Fix: `gh api -X PATCH repos/<org>/<repo>/branches/main/protection/required_status_checks -F 'contexts[]=Backlog Drift' -F 'contexts[]=ai-sdlc/pr-ready' -F 'strict=true'`

## Remote agents (`/schedule`) — read-only by design (AISDLC-442)

CCR remote sandboxes are **read-only by design**. They lack: `~/.ai-sdlc/signing-key.pem`, plugin install, worktree filesystem, and operator filesystem access.

**Acceptable in CCR**: PR/backlog status surveys, cron metric digests, Slack workflows, CI run-list / flake detection, `mcp__backlog__task_create`, `mcp__github__create_issue`.

**Prohibited in CCR**: `/ai-sdlc execute`, signing-key flows, plugin subagents (`developer`, `code-reviewer`, etc.), worktree ops, sibling-repo writes.

### Local vs. remote — what works where

| Task type | Works in CCR? | Works locally? | Notes |
|---|---|---|---|
| Survey open PRs | Yes | Yes | `gh pr list` |
| Check CI run health | Yes | Yes | `gh run list` |
| Post Slack digest | Yes | Yes | Webhook call |
| File a backlog task | Yes | Yes | `mcp__backlog__task_create` |
| File a GitHub issue | Yes | Yes | `mcp__github__create_issue` |
| Run `/ai-sdlc execute` | **No** | Yes | Requires signing key + worktree |
| Sign attestation envelopes | **No** | Yes | Signing key is operator-machine-local |
| Open worktrees | **No** | Yes | `git worktree add` fails in sandbox |
| Run developer subagent | **No** | Yes | Plugin subagents unavailable in CCR |

### Supported handoff workflow

When a CCR `/schedule` task detects work requiring local execution: file a backlog task via `mcp__backlog__task_create` (or `mcp__github__create_issue` for broad work) with full context. The local operator session picks it up on the next `/ai-sdlc orchestrator-tick` or via `/ai-sdlc execute <task-id>`.

> `/ai-sdlc execute` detects CCR sandboxes at startup (AISDLC-442) and refuses. See `docs/operations/remote-agents-readonly.md`.

Detection (first match wins): `CLAUDE_CODE_ENV=ccr` → `CLAUDE_REMOTE_EXECUTION=1` → `CLAUDE_CODE_ENV` set (any value) + `~/.ai-sdlc/signing-key.pem` absent.

## RFCs

Live in `spec/rfcs/RFC-NNNN-*.md`. Process: [`spec/rfcs/README.md`](spec/rfcs/README.md). Template: [`spec/rfcs/RFC-0001-template.md`](spec/rfcs/RFC-0001-template.md).

**Lifecycle field** (frontmatter, separate from sign-off checklist): `Draft` → `Ready for Review` → `Signed Off` → `Implemented`, or `Superseded`. Drafts land on main early — sign-off doesn't gate visibility. Legacy `status:` field retained for `scripts/check-rfc-docs.mjs`'s `requiresDocs` gate.

**Number lookup**: the canonical registry is the [Registry](spec/rfcs/README.md#registry) table in `spec/rfcs/README.md` (AISDLC-165). Read the "Next available number" line — do NOT scan the filesystem, the registry includes reservations that have no file yet.

**`requires:` vs `assumes:` — dependency-kind semantics (AISDLC-311).** RFC frontmatter splits inter-RFC dependencies:

- **`requires:`** — runtime-code dependency. This RFC's implementation IMPORTS code from the listed RFCs. They MUST ship (`Implemented`) before this RFC's implementation can ship.
- **`assumes:`** — design-contract dependency. Reads listed RFCs as a design contract but does NOT code-import. They only need to EXIST at `Ready for Review` or higher.

**Gate composition:**
- **DoR upstream-OQ gate** — tasks BLOCK dispatch on open OQs / pre-`Signed Off` lifecycle of RFCs under `requires:` (or `references:` for legacy). Tasks listing an RFC under `assumes:` skip the gate.
- **Lifecycle promotion** — when promoted to `Implemented`, `requires:` entries SHOULD also be `Implemented`. `assumes:` entries only need to exist.
- **Docs-drift linter** (`scripts/check-rfc-docs.mjs`) — `requires:` / `assumes:` entries must reference real RFC IDs.

See [`spec/rfcs/README.md#requires-vs-assumes--dependency-kind-semantics-aisdlc-311`](spec/rfcs/README.md#requires-vs-assumes--dependency-kind-semantics-aisdlc-311) for the full contract.

## Backlog Workflow

Tasks live in `backlog/tasks/` (open) and `backlog/completed/` (closed); managed via `mcp__backlog__*` MCP tools. Filename **must be ASCII**; titles may use unicode (`scripts/check-backlog-ascii.sh` enforces on commit).

### Non-dispatchable tasks (`dispatchable: false`) — AISDLC-243

Tasks that are **never** meant to be picked up by the autonomous orchestrator's developer subagent should carry `dispatchable: false` in frontmatter:

```yaml
dispatchable: false
dispatchableReason: "Operator soak phase — no code work; operator monitors stability"
```

- **Default is `true`** — omitting the field means the task IS dispatchable.
- Use `dispatchable: false` for **permanently** non-LLM-dispatchable tasks. Use `blocked.reason` for temporary holds.
- The `Dispatchability` filter runs AFTER `DependencyReadiness` and BEFORE `DorReadiness` in the orchestrator's admission chain.

### Drift gate

`backlog-drift` checks every reference in task frontmatter resolves. **Required** on commit (per-task pre-commit) + CI (full repo, fails on `error`-severity only). Local-only escape: `AI_SDLC_SKIP_DRIFT_GATE=1` (pre-commit hook only — NOT honored in CI). Auto-fix: `npx backlog-drift fix --task AISDLC-N`.

### Upstream-OQ gate (AISDLC-296 / RFC-0011 extension)

`refineBacklogTask()` runs an **upstream-OQ gate** before the seven-point rubric. Rejects the task when any referenced RFC has `lifecycle: Draft/Ready for Review` OR has unresolved `## Open Questions` entries (no `**Resolution:**` / `RESOLVED:` / `✅ RESOLVED` marker).

**Manual override**: tasks with `blocked.reason` in frontmatter skip the gate:

```yaml
blocked:
  reason: "RFC-0024 OQs acknowledged; operator walkthrough scheduled for 2026-05-20"
```

**Code surface**: `pipeline-cli/src/dor/upstream-oq-gate.ts` — `checkUpstreamOqs()` entry point.

### DoR ingress workflow gate (AISDLC-379)

`.github/workflows/dor-ingress.yml`'s `evaluate-pr-tasks` job **fails the `Evaluate backlog tasks changed by PR` status check** when any PR-staged backlog task has `overallVerdict: 'needs-clarification'` AND no `blocked.reason` override in frontmatter. Tasks with `blocked.reason` bypass the gate (comment still posts as `(override applied)`).

**Branch-protection helper**: `scripts/sync-dor-branch-protection.sh` PATCHes the canonical required-checks list (idempotent). Full runbook at [`docs/operations/dor-ingress-gate.md`](docs/operations/dor-ingress-gate.md).

### Canonical execution paths

| Use case | Command | Billing |
|---|---|---|
| Internal dogfood (backlog tasks) | `/ai-sdlc execute <task-id>` (e.g. `AISDLC-393`) | Subscription (Claude Code Max) |
| Internal dogfood (GitHub issues, subscription billing) | `/ai-sdlc execute <issue-number>` (e.g. `612`, `#612`, `gh:612`) | Subscription (Agent SDK credit pool; refuses to fall back to API key) |
| **Autonomous loop — single-session drain (Pattern X v2, AISDLC-396)** | `/ai-sdlc orchestrator-tick` (once, ScheduleWakeup loops). Conductor dispatches background `Agent(developer)` per manifest; next tick reconciles, fans out 3 reviewers, signs attestation, flips draft → ready. | Subscription — Sonnet for dev/code/test, Opus only for security. |
| **Autonomous loop — N>4 parallel via sibling Workers (Pattern Z)** | `/ai-sdlc orchestrator-tick` + N sessions running `/ai-sdlc dispatch-worker` | Subscription |
| Operator-driven single-PR | `cli-orchestrator tick --task-from-file <path>` (AISDLC-373) | Configured `--spawner` |
| Manual cleanup | `/ai-sdlc cleanup [<task-id>]` | n/a |
| Shell-driven autonomous tick (Pattern Y) | `cli-orchestrator tick --spawner claude` | Subscription |
| GitHub issue / unattended / CI | `pnpm --filter @ai-sdlc/dogfood watch --issue <id>` | API key |

`/ai-sdlc execute` is the default for internal work. Worktree-isolated, auto-creates sibling-repo PRs from `permittedExternalPaths`, marks Done + moves task file in the same PR.

**`/ai-sdlc execute` argument forms** (AISDLC-393): `gh:<n>` (explicit GH-issue), `<prefix>-<number>` (backlog task like `AISDLC-393`, including sub-IDs like `AISDLC-100.5`), `<number>` / `#<number>` (bare/hash-prefixed numeric → GH-issue). On the GH-issue path, NO backlog task file is created — the issue is the source of truth and the PR closes it via `Closes #N`. Refuses to fall back to `ANTHROPIC_API_KEY`-based SDK dispatch without explicit operator opt-in via the watcher path.

**Spawner kinds for `cli-orchestrator tick --spawner <kind>`** (AISDLC-349, default `claude` since AISDLC-352):
- `mock` — fixtures only; for plumbing tests. Billing: none.
- `api-key` — uses `ANTHROPIC_API_KEY` via the Claude Code SDK. Billing: API token.
- `claude` — **(DEFAULT)** shells out to `claude -p` via `child_process.spawn`. Use for autonomous tick from a shell. Billing: subscription. **Warning**: if `ANTHROPIC_API_KEY` is also set + `AI_SDLC_ORCHESTRATOR_SPAWNER_FALLBACK=api-key`, a spawner error can silently fall to paid API tokens — CLI warns at tick start.
- `codex` — dispatches via Codex CLI bridge (`CODEX_SPAWN_AGENT_BIN`). Billing: Codex plan.
- `copilot` — dispatches via GitHub Copilot CLI bridge (`COPILOT_SPAWN_AGENT_BIN`). Refuses to fall back to `ANTHROPIC_API_KEY`. Billing: Copilot subscription. See [`docs/operations/copilot-spawner.md`](docs/operations/copilot-spawner.md).

**New dispatch patterns (RFC-0041 Conductor/Worker Architecture)**:
- `in-session-agent` — each Worker is a separate CC session running `/ai-sdlc dispatch-worker`; tasks claimed from `.ai-sdlc/dispatch/queue/`. **Recommended default for autonomous drain.**
- `claude-p-shell` — Workers are `env -u CLAUDECODE claude -p` subprocesses spawned by `cli-dispatch-supervisor`. 30 min watchdog. For headless/CI contexts.

The Step 0-13 pipeline lives in `pipeline-cli/` (`@ai-sdlc/pipeline-cli`). Tier 1 = slash command body (subscription). Tier 2 = `executePipeline()` library + `SubagentSpawner` injection. Refs: `pipeline-cli/{README,docs/spawner,docs/steps}.md`, RFC-0012.

### Done semantics

Task file is moved to `backlog/completed/` in the originating PR's diff via `scripts/check-task-moved.sh`. The hook detects `(AISDLC-N)` in any commit subject in the push range and commits the move as a chore commit.

- **`/ai-sdlc execute` path**: developer subagent moves file to `backlog/completed/` BEFORE push. Hook detects the file is already there and no-ops (idempotent).
- **Ad-hoc / external contributor path**: if still in `backlog/tasks/` at push time, hook auto-moves it.

### Cross-repo writes — `permittedExternalPaths`

Tasks needing sibling-repo writes declare an allowlist:

```yaml
permittedExternalPaths:
  - '../ai-sdlc-io/'
```

The PreToolUse hook reads `<worktree>/.active-task` to resolve which allowlist applies. Without the file, cross-repo writes are denied. Env fallback: `AI_SDLC_ACTIVE_TASK_ID`.

### Parallel runs

Each `/ai-sdlc execute` runs in its own Claude Code session with its own per-worktree sentinel. Fan out via `/loop /ai-sdlc execute <task-id>` or multiple terminals — no shared mutable state to race on.

### Lifecycle rules

- **Create-before-execution**: when a plan spans multiple tasks, create them ALL before dispatching. In Pattern C projects, use `mcp__plugin_ai-sdlc_ai-sdlc__task_create`. In plain projects `mcp__backlog__task_create` is fine.
- **Claim on start**: status → `In Progress` (auto by `/ai-sdlc execute`).
- **Complete = TWO steps**: `mcp__backlog__task_edit` (status, ACs, finalSummary) + `mcp__backlog__task_complete` (moves file). Run the workspace test suite + lint before flipping.
- **Never leave `To Do` after implementation.** A task isn't closed until it's in `backlog/completed/`.

### `finalSummary` template

```markdown
## Summary
<one-paragraph: what shipped>

## Changes
- `path/to/file.ts` (new|modified): <what + why>

## Design decisions
- **<Decision>**: <reason + tradeoff>

## Verification
- `pnpm build` — clean
- `pnpm test` — <counts>
- `pnpm lint` — clean

## Follow-up
<next steps or "(none)">
```

### When NOT to create a backlog task

- Inline fixes caught during review (use the PR).
- Trivial chores (deps, config, typos).
- Exploration/spikes (retroactively if it becomes real work).

## Releases

**CHANGELOG.md is managed exclusively by release-please. Contributors MUST NOT edit it manually.**

### release-please rolling PR model (AISDLC-401)

`release.yml` fires on every push to `main`, runs `googleapis/release-please-action@v4`, and maintains a **single rolling PR** (`chore: release main` on branch `release-please--branches--main`). Regular feature PRs MUST NOT touch CHANGELOG.md.

The pre-push hook (`scripts/check-changelog-edit.sh`) WARNs when a feature branch touches CHANGELOG.md. Revert the CHANGELOG changes — release-please will reconstruct them from your commit messages. See [`docs/operations/release-flow.md`](docs/operations/release-flow.md).

### Package configuration

`.github/workflows/release.yml` runs `pnpm -r publish --no-git-checks`. Every non-`"private": true` workspace package MUST carry:

```jsonc
"publishConfig": { "access": "public", "registry": "https://registry.npmjs.org/" }
```

Without it, npm rejects with E402 silently per-package. `pnpm lint:publishable` (wired into `pnpm test`) catches regressions.

When adding a new publishable package: add to `pnpm-workspace.yaml`, add the `publishConfig` block (or mark `"private": true`), add to `release-please-config.json`. release-please does NOT add `publishConfig` automatically.

## Plugin MCP server — project root resolution (AISDLC-99, AISDLC-216)

The plugin's MCP server (`mcp__plugin_ai-sdlc_ai-sdlc__*` tools) resolves the project directory: `AI_SDLC_PROJECT_ROOT` env → `CLAUDE_PROJECT_DIR` env → walk up from `process.cwd()` for ancestor with `backlog/` → throw. Override with `AI_SDLC_PROJECT_ROOT=/abs/path`.

### Pattern C routing (AISDLC-216)

In Pattern C (non-bare parent repo + `.worktrees/<task-id>/` isolates), the parent's working tree is **read-only**. After resolving the candidate root, if `<root>/.worktrees/` exists with at least one subdirectory, the following routing applies:

1. **`AI_SDLC_ACTIVE_TASK_ID` env var** — routes to `<parent>/.worktrees/<task-id-lower>/`
2. **Per-worktree `.active-task` sentinels** — scans `<parent>/.worktrees/<id>/.active-task`. Most-recently-modified wins when multiple worktrees have sentinels.
3. **No signal → refuse** with the Pattern C error message.

Set `AI_SDLC_ACTIVE_TASK_ID=AISDLC-NNN` before launch when the env-var path is preferred.

### Pattern C hard guards (AISDLC-358)

The parent working tree MUST be on `main` at all times. Enforced by `scripts/check-orchestrator-state.sh` (Step 0 of every `/ai-sdlc execute` and `/ai-sdlc orchestrator-tick`) and by `runParentBranchGuard()` in `pipeline-cli/src/orchestrator/loop.ts`.

Guard logic:
- **Parent on non-main branch, clean working tree** → auto-recover: `git checkout main && git reset --hard origin/main`.
- **Parent on non-main branch, dirty working tree** → REFUSE. Prints offending branch name, dirty paths, and manual recovery command. Aborts the tick.

Recovery: stash or commit changes in the parent, then `git checkout main && git reset --hard origin/main`.

**Sanctioned vs. ad-hoc reset (AISDLC-450):** `git reset --hard origin/main` is ONLY permitted when invoked by `scripts/check-orchestrator-state.sh` on a verifiably clean working tree. Ad-hoc `git reset --hard` by the Conductor or a subagent is forbidden — if the script refuses (dirty parent), escalate to the Decision Catalog.
