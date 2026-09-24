# Visual test → fix loop — how the orchestrator closes the pipeline

The last stage of an orchestrated run. `ship` has committed the changelog and the PR is open (or its
command is ready). Now the orchestrator has the running app tested **visually**, and every defect
that matters is fixed by a dedicated `fix` agent — never by the orchestrator itself.

## Budget

`orchestrator_max_visual_loops` = **N fix rounds** (default 2). Round 1 tests; if it has Fix-now
findings and a fix round is left, they are fixed and round 2 re-tests; … Round N+1 is the last
test: its Fix-now findings are **reported, not fixed** (the stop). `N = 0` → test once, report, fix
nothing. Visual-loop re-reviews draw on the **remaining** `orchestrator_max_review_loops` budget
(what the review stage did not use); an exhausted review budget turns a needed re-review's
`CHANGES REQUESTED` into a stop.

## Round r (starting at r = 1)

1. **Visual test.** Dispatch a `stage-runner` for `visual-test` (it has no depth dial). It resolves
   how to start the app — an unknown command / URL comes up as a `visual-launch` question, answered
   only from settings / the map / the repo / the brief, else the round is `VISUAL BLOCKED` — starts
   the app once, builds the matrix and returns `SDD_DISPATCH` for `visual-tester` (one per visual
   surface, run **one after another**). The orchestrator runs each dispatch **from the main
   session**, because that is where the computer-use / browser tools (Claude in Chrome, the built-in
   browser, a computer-use / Playwright MCP) live; the tester inherits them. Its `VISUAL_REPORT` goes
   back verbatim as `SDD_REPORTS`; the runner stops the app and writes
   `_visual/visual-test-<date>[-r<n>].md`.
2. **Decide each finding.** Finding ids are round-qualified (`R<r>-VF<k>`). The runner raises one
   `visual-finding` question per finding; the orchestrator answers per
   [`decision-policy.md`](./decision-policy.md) §3 (default **Fix now**; `Accept` never for a
   blocker or an `ac-violation`; `Not a bug` only with a cited spec line) and ledgers each. The
   runner sets the **final verdict** from the resolutions, commits the report
   (`visual-test: <slug> <verdict> (r<r>)`) and returns `SDD_STAGE_DONE` with the Fix-now list.
3. **Verdict.**
   - `VISUAL PASS` (no Fix-now left — including a round whose findings were all Accepted / Not a
     bug) → the runner has moved the feature to **Shipped** in the roadmap and drafted the PR note;
     the orchestrator posts the note (`orchestrator_open_pr: true`, PR open) → the loop ends.
   - `VISUAL BLOCKED` → **stop**: report what is missing (no driver, the app won't start, no test
     account, unknown start command) — never treat it as a pass. The roadmap stays in *Now*.
   - `VISUAL ISSUES` with a fix round left → step 4. None left → **stop** with the open findings
     and their screenshots; the roadmap stays in *Now*.
4. **Fix each Fix-now finding — one `fix` runner per finding.** In severity order (blocker →
   cosmetic), one at a time on the feature branch (fixes to UI code usually touch shared components —
   parallel fixes would collide):
   - Dispatch `stage-runner` for `fix` with the bug report = the finding verbatim (id, steps,
     expected, actual, screenshot path, AC, suspect area) + the report path.
   - Its `explorer` dispatch comes up as `SDD_DISPATCH` → the orchestrator runs it.
   - Its questions: intake (normally none — a finding is a complete report) and the **spec-patch
     confirmation** (`fix-spec-patch`, when triage says spec-bug or gap) → answered per the decision
     policy.
   - The runner pins the defect with a failing test (a component / e2e-through-UI test for a UI
     defect), makes the minimal fix through the gate, writes `_fixes/<date>-<short>.md` with
     `finding: R<r>-VF<k>` and `commit: see SDD-Fix trailer`, commits with `SDD-Fix` (+ `SDD-AC`) and
     returns `SDD_STAGE_DONE`. A fix that cannot be pinned by a test → the finding is reported as
     unfixed (never «fixed» without RED proof).
   - Ledger: `R<r>-VF<k> → fix <record> (<sha>)`.
5. **Re-review when needed.** If any fix in this round touched >5 files, crossed a module boundary,
   or patched a spec AC → dispatch a `review` runner over the whole change (its `reviewer` in clean
   context). `CHANGES REQUESTED` → an implement fix round per
   [`implement-lead.md`](./implement-lead.md) §Fix round, then `review` again — repeated until `PASS`
   within the remaining review budget, else **stop**. Only a `PASS` continues to step 6.
6. **Push** the new commits to the PR branch if the PR is open (plain push, never force). `ship` is
   not re-run — the changelog describes the feature, and the fixes belong to the same PR; the final
   PR note (step 3) lists them.
7. **Next round.** r + 1 → back to step 1 (the tester re-runs the **whole** matrix, not just the fixed
   scenarios — a fix can break a neighbouring screen).

## What goes in `run.md`

The **Visual rounds** table (round, final verdict, findings by severity, Fix-now ids, fix records +
commits, re-review yes/no) and one ledger row per finding decision. The final report lists Accepted
findings (with reasons) right after the UNGROUNDED decisions — they are visible deviations a human
should know about.

## Guarantees

- The orchestrator never edits code for a visual finding — every change comes from a `fix` runner
  with a failing test first.
- The visual tester never runs against production or with real credentials
  ([`../../visual-test/references/drivers.md`](../../visual-test/references/drivers.md) §Safety).
- A `VISUAL BLOCKED` round never ends the run as successful.
