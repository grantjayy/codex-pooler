# Issue #234 Fixed-Anchor Weekly Zero Plan

**Goal:** Make a fresh, repeatedly confirmed provider `used_percent=0` observation replace a poisoned exhausted account-weekly canonical row when all observations use the same fixed reset anchor.

**Status:** Approved outcome; immutable implementation handoff.

## Repository and worktree

- Repository: `grantjayy/codex-pooler` fork of `icoretech/codex-pooler`
- Worktree: `/tmp/codex-pooler-issue-234-fix`
- Branch: `fix/issue-234-fixed-anchor-zero`
- Base: `3ab4114b019a2034c3924656f89f4deebbd9a3cc`
- Upstream issue: `https://github.com/icoretech/codex-pooler/issues/234`

## Context and evidence

- Live provider `/backend-api/codex/usage` reports `allowed=true`, `limit_reached=false`, account weekly `used_percent=0`, and reset `2026-08-08T03:49:00Z`.
- Canonical Pooler row remains `used_percent=100` with the same reset anchor.
- Candidate metadata repeatedly records zero but loops through `candidate_restarted` and `candidate_not_ready`.
- A guarded one-row database correction to zero was immediately rewritten to 100 by reconciliation. Database-only repair is not durable.
- `EvidenceStore.sliding_restart_attrs/4` cannot prove this case because the canonical row already holds the new fixed reset anchor. Sliding, forward-anchor, and expired-cycle proofs all fail.
- Current upstream main has the same behavior.

## Root constraint

Pooler has no replay-safe confirmation path for a same-anchor zero candidate over an exhausted account-weekly canonical row. The durable fix belongs in quota-cycle confirmation logic, not the reporting card or an operator database edit.

## Settled approach

Add the narrowest account-weekly restart proof that accepts the incoming snapshot only when:

1. the canonical row is exhausted;
2. the stored candidate and incoming observations are zero;
3. candidate, incoming, and canonical reset anchors are equivalent;
4. candidate/provider observations advance and cover the existing confirmation span; and
5. existing evidence liveness checks pass.

Use repeated provider observations, not one zero. Do not weaken model-weekly safeguards, sliding-window safeguards, source-quality rules, or runtime-header behavior. Do not add a reporting workaround.

If the current normalized `Evidence` does not carry `allowed` / `limit_reached`, do not widen provider parsing or schemas in this fix. The multiple-observation confirmation and source/liveness requirements are the selected safety mechanism.

## Scope

### Modify

- `lib/codex_pooler/upstreams/quota/windows/evidence_store.ex`
- The smallest existing quota evidence-store test file that owns account-weekly restart behavior, expected to be `test/codex_pooler/upstreams/quota/windows/evidence_store_weekly_restart_test.exs`

### Non-goals

- No database migration.
- No usage-card change.
- No saved-reset change.
- No auth or account identity change.
- No routing partition change.
- No Hermes gateway restart.
- No deployment until local tests and review pass.
- No merge of the upstream PR/issue.

## Test-driven tasks

### Task 1: Reproduce the same-anchor poisoned-canonical case

1. Add a behavioral regression test using the real `EvidenceStore` persistence path.
2. Seed an account weekly canonical provider-usage row with `used_percent=100` and fixed reset anchor `R`.
3. Record a fresh zero provider observation with fixed reset anchor `R`; assert canonical pressure stays 100 and a candidate is present.
4. Record another advancing zero observation after the required confirmation span with the same anchor `R`.
5. Assert current code remains stuck at 100. Run the focused test and capture RED for the right reason.

### Task 2: Implement the narrow proof

1. Extend `restart_candidate_progressing?/5` (or its smallest cohesive helper) with a same-anchor exhausted-canonical proof.
2. Require account weekly provider evidence, exhausted canonical pressure, zero candidate/incoming, equivalent anchors, advancing liveness, valid candidate, and confirmation span.
3. Reuse existing helpers and constants. Do not add a parallel state machine.
4. Keep existing cycle decision logging.

### Task 3: Verify behavior and regressions

1. Run the new focused test; expect GREEN.
2. Run all quota window evidence-store tests.
3. Run formatter and project lint/test commands documented by the repository.
4. Confirm no unrelated diff.
5. Commit through normal hooks.

### Task 4: Independent review and runtime verification

1. Review the diff adversarially for one-observation acceptance, cached/replayed zero acceptance, model-window leakage, and same-cycle false reset.
2. Rebuild a candidate image only after tests/review pass.
3. Deploy only the Pooler app image through the durable compose contract; do not restart Hermes gateway.
4. Reconcile moa personal once.
5. Verify live provider zero, canonical zero, candidate cleared, selection zero, account active/eligible/healthy, and `/ai-usage` shows near-full remaining.
6. Open an unmerged PR on the fork and link upstream issue #234. Do not merge.

## Acceptance

- Regression test is RED before the source fix and GREEN after it.
- A same-anchor, repeatedly confirmed provider zero replaces an exhausted account-weekly canonical row.
- A single zero observation does not replace the canonical row.
- Existing sliding, forward-anchor, stale/replay, model-weekly, and positive-usage tests remain green.
- Live moa personal reconciliation persists provider zero instead of returning to 100.
- Pooler selection and usage card show the corrected remaining capacity.
- No saved reset is spent.

## Risks and escalation

- Escalate if replay safety requires changing normalized provider evidence or public schemas; that is outside this contract.
- Escalate if the fix changes model-weekly behavior or requires a migration.
- Deployment is production-impacting but reversible by restoring the current image tag.

## Routing

```yaml
reasoning_mode: procedural
execution_topology: direct
gjc_profile: procedural
gjc_workflow: direct
capability_evidence:
  - Root cause and exact EvidenceStore seam are confirmed.
  - Expected behavior is captured by one focused persistence-level regression.
  - The change is narrow, reversible, and uses established confirmation helpers.
topology_evidence:
  - One coherent fix and verification chain has one owner.
  - No independent durable workstreams or cross-session persistence need exists.
escalation_triggers:
  - Provider evidence/schema widening becomes necessary.
  - Model-weekly or public behavior would change.
  - A database migration is required.
```
