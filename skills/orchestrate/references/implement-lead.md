# Implement as the engine lead — how the orchestrator runs the TDD engine

A `stage-runner` cannot spawn agents, so it cannot run the `implement` engine's fan-out. The
orchestrator therefore takes the **lead** role that
[`../../implement/references/team-exec.md`](../../implement/references/team-exec.md) describes —
without `TeamCreate` / `Workflow`, using plain agent dispatch — and keeps every other engine rule
([`tdd-loop.md`](../../implement/references/tdd-loop.md),
[`escalation.md`](../../implement/references/escalation.md),
[`decision-tree.md`](../../implement/references/decision-tree.md)) unchanged. The stage is split
into three phases.

## Phase A — plan (runner, `phase=plan`)

The runner executes `implement` protocol steps **1–7** (preconditions, settings, command detection,
DAG build, mode pick, run-plan, banner) and returns `SDD_STAGE_DONE` with `outcome: plan` and a
`plan:` block instead of executing:

```yaml
plan:
  commands: {unit: "…", integration: "…|null", lint: "…|null", vet: "…|null"}
  integration_policy: auto        # from require_integration + the Docker probe
  parallel_eligible: true         # the engine's own eligibility expression
  max_parallel: 3
  isolation: worktree
  phases:                         # Kahn order
    - [T1, T3]
    - [T2]
  lanes: [[T2, T4]]               # serialization lanes (migration, overlapping files_hint, compile-coupled)
  tasks:
    T1: {title: "…", layer: domain, acs: ["AC-1: <full text from spec §5>"], dod: "…", files_hint: ["…"], deps: []}
```

The task entries carry the **full** text (AC text pulled from spec §5, DoD, files) — the RED/GREEN
agents never see the conversation.

## Phase B — execute (orchestrator)

For each Kahn phase, take the ready tasks (deps `done`), respect lanes, and run up to `max_parallel`
at once **only if** `parallel_eligible` (otherwise strictly sequential, in topo order). Parallel
tasks are dispatched with worktree isolation (one worktree per task); sequential tasks run in place.

Per task:

1. **RED** — dispatch `sdd:test-author` with the task entry + commands (model/effort per the roster,
   `model_test_author` / `effort_test_author`, `.size` scaling). Require the first-run
   classification + the quoted failing line. `BAD red` / `false-pass` → re-dispatch with that
   feedback (counts toward `max_red_retries`). `NON-red` → follow `integration_policy`.
2. **GREEN / REFACTOR / GATE** — dispatch `sdd:implementer` with the task entry + the red handover.
   Require gate evidence: the exact commands run and their tail output. A `layer: migration` task
   promotes its staged migration first (per [`inputs.md`](../../implement/references/inputs.md)).
3. **Verify** — the orchestrator re-runs the task's unit test command itself (cheap proof; «should
   pass» is not evidence). In worktree mode, bring the task's changes onto the feature branch.
4. **Commit** — in dependency order, task-scoped, with `SDD-Task: <id>` + one `SDD-AC: <id>` per
   satisfied AC; a compile-coupled lane gets one shared gate + one commit carrying every task's
   trailers. Respect `auto_commit` (`per_phase` batches per Kahn phase; `off` leaves the tree staged
   and says so).
5. **Optional per-task review** — when `team_mode: true`, dispatch `sdd:reviewer` on the task diff
   (stage-1 AC compliance, stage-2 quality) and fix before committing. The authoritative review is
   still the `review` stage.

**Escalation** follows [`escalation.md`](../../implement/references/escalation.md) in order, with the
orchestrator as the decider:

- retries → stronger model / higher effort for this task → **split the task** (a decision: ledger it,
  update `tasks.json` + `tracker.md` with the new ids and dep edge, drive each half with its own RED).
- **Step 4 (the test may encode a wrong AC)** is a `business_rule` question for the orchestrator.
  `escalation.md` says fixing an AC is a `specify`/`clarify` change with the human in the loop; in an
  orchestrated run the orchestrator stands in for that human **only when the brief grounds the fix**:
  - *grounded* → the AC is changed **through the spec-owning stage**, never edited directly: dispatch a
    `clarify` runner with the conflict (failing line + AC text + the brief citation) as its finding, so
    its forbidden-token rule and self-check apply; ledger it as `supersedes D-nnn`. Then re-sync the
    copies of that AC's text — the task's `acs` in `tasks.json`, its `test-plan.md` / inline test-plan
    row, and any `sad.md` §6 flow note naming it — in one commit
    `tasks: <slug> re-sync AC-n (orchestrator D-nnn)`, and re-run RED against the amended AC;
  - *ungrounded* → per `orchestrator_escalate` (§2 of [`decision-policy.md`](./decision-policy.md));
    with `none` → never guess: rollback to the last green (step 5) and apply `stop_on_red`.
- **Never** weaken a test; **never** commit a red or a skipped hard gate.

`stop_on_red: true` → halt the phase loop and go straight to Phase C with the blocked task reported.
`false` → drop the task, auto-block its transitive dependents, continue independent branches.

## Phase C — finalize (runner, `phase=finalize`)

Dispatch a runner with `phase=finalize` and the execution results (per task: status, commit sha,
gate results, escalation notes). It runs protocol step 9's bookkeeping + step 10: updates
`tasks/tracker.md`, writes the summary, and returns `SDD_STAGE_DONE` with the handoff block
(`next: review`).

## Fix round (after `review` → `CHANGES REQUESTED`)

1. Dispatch a runner `phase=plan` with the round number and the review record
   (`_review/review-<date>[-r<n>].md`) as extra inputs. It first **appends** each finding resolved
   *Fix now* to `tasks.json` as a fix task (`id: R<n>-F<k>`, `acs` = the finding's cited AC,
   `dod` = the finding resolved + its regression test, `files_hint` from the cited lines, `deps: []`)
   and to `tasks/tracker.md`, commits `tasks: <slug> review round <n> fix tasks`, and then runs
   protocol steps 1–7 as usual — so the plan still comes from `tasks.json`, the engine's source of
   truth. Already-`done` tasks are not re-planned.
2. Run Phase B on those tasks (a finding that is a missing test gets RED first like any task; a
   pure-quality finding with no behaviour change runs GREEN/REFACTOR/GATE against the existing tests).
3. Phase C, then a fresh `review` runner over the **whole** change (the stricter of review's
   «re-review the changed surface» — a fix can break something outside the finding). Count the round
   against `orchestrator_max_review_loops`.
