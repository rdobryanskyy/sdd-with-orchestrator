# QA gate — the orchestrator's last check before it calls a feature finished

Runs after the engine's visual loop (`VISUAL PASS`, or N/A for no visual surface) and before the
final report. Purpose: prove the **final head** — including every review-fix and visual-fix commit
that landed after `review` — is fully tested and that every acceptance criterion has evidence.
The orchestrator **decides and dispatches**; it runs commands but never edits code or tests here.

## 1. Full regression on the final head

- Detect the commands per [`../../implement/references/command-detection.md`](../../implement/references/command-detection.md)
  (settings overrides first). Run **every tier the repo has**, whole suite, not the changed packages:
  build / typecheck → lint → unit → integration → e2e / UI tier (the surface's tier from
  [`../../_shared/surfaces.md`](../../_shared/surfaces.md)) → migrations up + down on a scratch DB
  when the feature staged migrations.
- Run it in a fresh `implementer` dispatch (`subagent_type: "sdd:implementer"`, task = «run the full
  gate, change nothing, report each command + exit code + failing test names») so long output stays
  out of the orchestrator's context.
- A tier that cannot run here (no Docker, no browser runtime) is recorded `not run — <reason>`, never
  `pass`. A tier the repo does not have is `n/a`.
- Any red → §4.

## 2. AC evidence matrix

One row per spec §5 AC (Grep the ACs; don't load the spec whole):

| Column | Source |
|---|---|
| Automated test(s) | `SDD-AC: AC-n` commit trailers (`git log --grep`), `test-plan.md` / the inline `## Test plan` rows |
| Result on final head | §1's run (the named tests passed) |
| Real-run check | `ship`'s verification (spot-checked AC ids) |
| Visual scenario | the last `_visual/` report — `VT-n` + pass/fail + screenshot, for a UI-observable AC |
| Status | `covered` / `GAP — <what is missing>` |

Rules: every AC needs ≥1 automated test that **passed on the final head**; every UI-observable AC
(per [`../../_shared/surfaces.md`](../../_shared/surfaces.md)) also needs a **passing** `VT-n`.
Accepted visual findings are listed, not counted as gaps. Open spec §8 questions owned by
`orchestrator → human review` are listed as follow-ups.

## 3. Clean-context QA audit

Dispatch `sdd:reviewer` (fresh context, `judgment_model`) with: the matrix, the §1 command log, the
last visual report path, the diff range (`<base>..HEAD`). Task: «audit, don't review style — for each
AC, does the cited test actually assert the AC's outcome (not just touch the code)? Is any tier
silently skipped? Does any fix commit after review lack a regression test? Verdict `QA PASS` /
`QA FAIL` + findings with AC id and file:line.» Its report goes into `qa-signoff.md` **verbatim**.
Each finding is adjudicated under the `review-finding` rule of
[`../../orchestrate/references/decision-policy.md`](../../orchestrate/references/decision-policy.md)
(default Fix now; ledgered).

## 4. Closing gaps (one QA fix round)

- **Red test / failing tier** → a `fix` runner with the failing command + test output as the bug
  report (the engine's visual-fix path, with the failure instead of a finding).
- **AC without a (meaningful) test** / reviewer Fix-now finding → an implement **fix round** per
  [`../../orchestrate/references/implement-lead.md`](../../orchestrate/references/implement-lead.md)
  §Fix round, fix task ids `QA-F<k>`; RED first — a new test that goes green immediately on an AC it
  should prove is re-examined, not accepted.
- **UI-observable AC without a passing `VT-n`** → one more `visual-test` runner round.
- Then re-run §1–§3 once. Still failing → `QA FAIL`, **stop**, findings in the report. Never weaken,
  skip or delete a test to pass.

## 5. PR CI

When the PR is open: push, then watch its checks (`gh pr checks --watch` / `glab ci status`, bounded
by `orchestrator_ci_timeout_min`, default 30). Red check → root-cause it: a failure in this change →
one `fix` runner, push, watch once more; a failure red on the base branch too → record it as
«not this change» with the evidence. No CI configured → recorded `no CI`. Timeout → recorded
`pending`, the run reports it (not `QA PASS` until CI is green or absent).

## 6. Verdict

`QA PASS` = §1 all green (or `n/a` / `not run` with a stated reason for a tier the repo can't run
here — listed as a follow-up), §2 no `GAP`, §3 reviewer `QA PASS` after adjudication, §5 green / no CI.
Anything else = `QA FAIL` with the reason. Write the verdict, head sha and date into `qa-signoff.md`.
