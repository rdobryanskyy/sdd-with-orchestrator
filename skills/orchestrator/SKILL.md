---
name: orchestrator
model: opus
effort: max
agents: [stage-runner, explorer, critic, devils-advocate, researcher, strategist, analyst, test-author, implementer, reviewer, visual-tester]
description: >
  Use when the developer wants a feature FINISHED for them from one description — type
  `/sdd:orchestrator <what you need>` and walk away. The one-shot front door over the orchestrate
  engine: it front-loads the few things only a human can supply (dirty tree, how to start the app,
  a visual driver, forge auth), then runs the whole SDD pipeline unattended with the full-delivery
  preset (no --until, PR opened, higher loop caps), and ends with a QA gate — the full regression
  suite on the final head, an AC → test → visual evidence matrix audited by a clean-context
  reviewer, and the PR's CI — so "done" means reviewed, fully tested, visually checked and
  QA-signed. Triggers on "/sdd:orchestrator", "orchestrator {description}", "finish this feature
  for me", "do everything for me", "build it end to end with QA", "autopilot", "sdd autopilot",
  "take it all the way to a PR". Never merges.
---

# Skill: orchestrator

`/sdd:orchestrator "<what you need>"` — you describe the feature once; the orchestrator takes every
responsibility from there and hands back an open PR that is **implemented test-first, reviewed,
fully regression-tested, visually tested and QA-signed**, plus a ledger of every decision it made.

This skill is the **front door**, not a second engine. The engine is
[`orchestrate`](../orchestrate/SKILL.md) (stage runners, relay contract, decision policy,
implement lead, visual fix loop) — this skill runs it **unchanged**, adding three things around it:

1. **Intake** — everything a human must supply is asked **once, up front**, so the run never parks
   mid-way waiting on you.
2. **Full-delivery preset** — the run always goes to the end (PR opened, visual gate, loops sized
   for finishing, not for a demo).
3. **QA gate** — after the visual loop, before the final report: full regression, AC evidence
   matrix, clean-context QA audit, PR CI → [`./references/qa-gate.md`](./references/qa-gate.md) ·
   record → [`./templates/qa-signoff.md`](./templates/qa-signoff.md).

The QA sign-off's prose follows `artifact_language` (verdicts, ids, headings stay English) →
[`../_shared/artifact-language.md`](../_shared/artifact-language.md).

## Owner

The developer who typed the command. They answer the intake (≤4 questions, often none), then review
the result: UNGROUNDED decisions first, the QA sign-off, the PR. Merging stays their call.

## Inputs

- **The brief** — the text after the command (`$ARGUMENTS`):
  - `/sdd:orchestrator <plain-language description>` — slug derived,
  - `/sdd:orchestrator <slug> "<description>"` or `<slug> --brief=<path>` (ticket / notes file),
  - `/sdd:orchestrator <slug> --resume` — continue a stopped run from its `run.md`.
  - Empty → print one line «describe what you need: `/sdd:orchestrator <description>`» and stop.
- **Optional flags** (passed through to the engine): `--depth=easy|medium|hard`,
  `--escalate=none|business|hard`, `--from=<stage>`. `--until` is **not** accepted here — a partial
  run is `/sdd:orchestrate … --until=<stage>`.
- Settings: `.claude/sdd.local.md` (documented defaults →
  [`../implement/references/settings.md`](../implement/references/settings.md)).

## Protocol

1. **Parse + banner.** Split slug / description / flags. Print:
   `orchestrator slug=<…> preset=full-delivery depth=<…> escalate=<…> review_loops≤<n> visual_loops≤<n> qa=on`.
2. **Intake — ask everything now, never later.** Read-only probes, then **one** `AskUserQuestion`
   batch (≤4 questions) for only what failed; nothing failed → no questions:
   - **Working tree dirty** → «Carry them onto the feature branch as a first commit / Stop». Carry =
     `git switch -c sdd/<slug>` (changes travel with it; the default branch stays untouched), commit
     `wip: before orchestrator`; the engine's preflight then finds a clean tree on a feature branch.
     Never stash, never discard.
   - **Visual readiness** (only when the brief or `architecture-map.md` shows a UI — web / mobile /
     desktop / cli): resolve start command + URL per
     [`../visual-test/references/drivers.md`](../visual-test/references/drivers.md) §1 (settings →
     map → repo scripts). Unresolved → ask for them (Other = free text), the answer is passed to the
     engine as a grounded source for `visual-launch`. No driver available (no browser / computer-use
     tool in this session, no Playwright in the repo) → ask «Continue — the visual gate will report
     BLOCKED / Stop to enable a driver».
   - **Forge auth** (`gh auth status` / `glab auth status` for the remote's forge) → failing is not a
     question: note it; the PR command will be printed instead of run.
   - **Survey needed** (no/stale `docs/architecture-map.md`) → not a question: the engine runs it.
   Ledger every intake answer as `D-000a…` rows (stage `intake`, grounding `human`) when `run.md`
   exists — the engine's bootstrap creates it; carry them into it on step 3.
3. **Run the engine.** Read [`../orchestrate/SKILL.md`](../orchestrate/SKILL.md) and execute its
   Protocol **steps 1–10 exactly**, with the **full-delivery preset** applied as run-level overrides
   (in the `run.md` frontmatter, never written to the settings file):
   - no `--until` — every stage through `visual-test`;
   - `orchestrator_open_pr: true` (unless forge auth failed → print the command);
   - `orchestrator_max_review_loops` and `orchestrator_max_visual_loops` = `max(setting, 3)`;
   - `orchestrator_escalate: none` unless `--escalate` was passed — the human was already asked
     everything up front; every other decision is the orchestrator's, ledgered (UNGROUNDED flagged).
   All engine rules hold: stage runners, verbatim reports, decision policy, never merge / force-push /
   weaken a test. A stop condition of the engine is a stop here — go to step 6.
4. **QA gate.** After the engine's step 10 (visual `VISUAL PASS`, or N/A for no visual surface), run
   [`./references/qa-gate.md`](./references/qa-gate.md): full regression on the final head → AC
   evidence matrix → clean-context QA audit (`reviewer`) → gaps closed through the engine's fix paths
   (capped by one QA fix round) → PR CI. Write
   `docs/features/<slug>/_orchestrator/qa-signoff.md` from
   [`./templates/qa-signoff.md`](./templates/qa-signoff.md) with `QA PASS` / `QA FAIL`; commit
   `orchestrator: <slug> qa-signoff`; push to the PR if open, and post its summary as a PR comment.
5. **Finish.** Run the engine's step 11 (run log final, push, report) with the QA verdict added to
   `run.md` §Outcome.
6. **Handoff.** **Emit the stage-handoff block** per [`../_shared/handoff.md`](../_shared/handoff.md)
   (terminal variant): *What I did* — stages run / skipped, commits, review + visual rounds, QA
   verdict, PR + CI state; *Review* — **UNGROUNDED decisions first**, then `qa-signoff.md`, `run.md`,
   the last `_visual/` report, the PR; *Run next* — **Done** (the PR URL; merge is your call), or on a
   stop: what must change, then `/sdd:orchestrator <slug> --resume`.

**Resume.** `--resume` skips intake questions already ledgered, re-reads `run.md`, continues the
engine from its last stage; a run whose engine part is done but has no `qa-signoff.md` resumes at
step 4.

## Definition of Done ("finished")

All of: every backbone stage DONE or SKIPPED with a cited N/A · review `PASS` · `VISUAL PASS` (or N/A —
no visual surface) · `QA PASS` · the PR open with its CI green (or no CI configured, stated) · nothing
merged. Anything short of that is a **stop**, reported as such — never called done.

**Structural self-check** ([`../_shared/self-check.md`](../_shared/self-check.md)) — before the
handoff, assert: the engine's self-check passed; `qa-signoff.md` exists and its verdict matches its
rows (any AC row without a passing automated test, or a UI-observable AC without a passing `VT-n`,
⇒ not `QA PASS`); the regression run's command and result are recorded; the QA audit report is the
reviewer's verbatim text; the head sha in `qa-signoff.md` equals the PR head.

## Anti-patterns

- **Asking mid-run.** Everything human is asked in intake; after that the orchestrator decides and
  ledgers. A question that surfaces late and truly needs a human is a stop, not a silent guess.
- **"Done" without the gates.** A PR without `VISUAL PASS` (for a UI) or without `QA PASS` is a stop.
- **Regression on changed files only.** Fix commits land after `review`; the QA gate runs the
  **whole** suite on the final head.
- **Self-certifying QA.** The matrix is audited by a fresh `reviewer`, never by the orchestrator.
- **Re-implementing the engine here.** Stage logic lives in `orchestrate` and the stage skills.
- **Merging, force-pushing, weakening or skipping a test** to reach green — never.

## References & templates

[`../orchestrate/SKILL.md`](../orchestrate/SKILL.md) (the engine) ·
[`./references/qa-gate.md`](./references/qa-gate.md) · [`./templates/qa-signoff.md`](./templates/qa-signoff.md) ·
[`../orchestrate/references/decision-policy.md`](../orchestrate/references/decision-policy.md) ·
[`../visual-test/references/drivers.md`](../visual-test/references/drivers.md) ·
[`../implement/references/command-detection.md`](../implement/references/command-detection.md) ·
[`../_shared/tool-adapters.md`](../_shared/tool-adapters.md).
