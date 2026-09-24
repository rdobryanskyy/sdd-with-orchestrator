---
name: orchestrate
model: opus
effort: max
agents: [stage-runner, explorer, critic, devils-advocate, researcher, strategist, analyst, test-author, implementer, reviewer]
description: >
  Use to run the WHOLE SDD pipeline autonomously from a plain-language description of what you
  want. The orchestrator (this skill — the main session on the strongest model) is the single
  decision-maker: it gives every stage to a fresh stage-runner agent (survey → specify → clarify →
  design → sequences → data-model → api → tasks → plan-tests → implement → review → ship), answers
  every question those agents raise, runs every sub-agent they need (critic, researcher,
  reviewer, test-author, implementer, …), routes by size/route, loops review → implement, and
  records every decision with its grounding in a decision ledger. Triggers on "orchestrate
  {description}", "run the whole pipeline", "build this end to end", "autopilot", "sdd autopilot",
  "/sdd:orchestrate", "take this idea to a PR", "run sdd for me and decide yourself". Never merges;
  stops at an open PR (or earlier with --until=<stage>).
---

# Skill: orchestrate

The **orchestrator** sits on top of the 19 pipeline skills. You give it a description of what you
want; it drives the entire backbone to a reviewed, verified PR — **every stage runs in its own
agent, and the orchestrator answers every question and makes every decision**, writing each one
to a ledger you can audit.

It does not replace or rewrite any stage: each stage skill runs unchanged inside a
[`stage-runner`](../../agents/stage-runner.md) agent in `ORCHESTRATED MODE`, which hands its
`AskUserQuestion`s and sub-agent dispatches **up** to the orchestrator instead of to a human. The
relay contract (message formats, resume, replay, ledger) is canonical in
[`../_shared/orchestration.md`](../_shared/orchestration.md) — this file is the spine.

How it decides → [`./references/decision-policy.md`](./references/decision-policy.md) ·
how it runs the TDD engine → [`./references/implement-lead.md`](./references/implement-lead.md) ·
run log + ledger → [`./templates/run.md`](./templates/run.md).

The run log's prose follows `artifact_language` (headings, ledger columns and machine tokens stay
English) → [`../_shared/artifact-language.md`](../_shared/artifact-language.md).

## Owner

The developer who hands over the brief. The orchestrator is accountable for every decision it
records; the developer reviews the ledger (UNGROUNDED decisions first) and the PR. Merging stays
the developer's call.

## Inputs

- The **brief** — one of:
  - `/sdd:orchestrate "<what you want, in plain words>"` (slug derived),
  - `/sdd:orchestrate <slug> "<description>"`,
  - `/sdd:orchestrate <slug> --brief=<path>` (a ticket / notes file, read verbatim),
  - `/sdd:orchestrate <slug> --resume` (continue a run from its `run.md`).
- Flags: `--depth=easy|medium|hard` (dial passed to every Q&A stage; default `orchestrator_depth`),
  `--until=<stage>` (stop after that stage, e.g. `tasks` for a plan-only run), `--from=<stage>`
  (start later; earlier artifacts must exist), `--escalate=none|business|hard`.
- Settings: `.claude/sdd.local.md` — auto-created with documented defaults if absent →
  [`../implement/references/settings.md`](../implement/references/settings.md); the `orchestrator_*`
  keys plus everything the stages and the engine already read.
- The repo: `docs/architecture-map.md`, `CONTEXT.md`, `docs/features/<slug>/…`, git state.

## Protocol

1. **Settings + banner (read-only).** Read `.claude/sdd.local.md` if present (else take the
   documented defaults — nothing is written yet). Resolve depth, escalate policy, review-loop cap,
   `orchestrator_open_pr`, `stop_on_red`, `auto_commit`, `branch_strategy`, per-role models
   (`judgment_model`, `model_<role>`). Print one banner line:
   `orchestrate slug=<…> depth=<…> escalate=<…> until=<…> model=<…> review_loops≤<n> open_pr=<…>`.
2. **Preflight — before any write.** The working tree must be clean (stash nothing — a dirty tree is
   a **stop**: report it). A repo with **zero commits** (fresh `git init`) is fine: skip the clean
   check's diff against HEAD and let the bootstrap commit below be the first commit. Derive a
   kebab-case slug if none was given (refuse a slug that already has a `spec.md` unless
   `--resume`/`--from`). Create/switch the branch per `branch_strategy` (a feature branch
   `sdd/<slug>` when on the default branch).
3. **Bootstrap (one commit).** Now write: auto-create `.claude/sdd.local.md` + patch `.gitignore` if
   absent (per the settings doc), the brief **verbatim** to `docs/features/<slug>/_orchestrator/brief.md`,
   and `run.md` from [`./templates/run.md`](./templates/run.md). Commit `.gitignore` + `_orchestrator/`
   as `orchestrate: <slug> bootstrap` (the settings file itself is git-ignored). From here on, the
   only uncommitted changes a runner may find are its own stage's. On `--resume`, skip 2–3: read
   `run.md` and continue from its last stage — never from conversation memory.
4. **Plan the stages.** `survey` first when `docs/architecture-map.md` is missing or stale (its
   `reflects_commit` is behind). If survey ran in **greenfield** mode it emits the scaffold
   `docs/features/_scaffold/tasks.json` — run `implement _scaffold` (the engine-lead flow, step 7)
   **before** `specify`, so the feature builds into a repo that builds and boots. Then the backbone.
   The optional stages are resolved later from `.route` — the orchestrator never pre-skips.
5. **Stage loop.** For each stage, dispatch a fresh runner — `subagent_type: "sdd:stage-runner"`,
   `model` = the stage skill's frontmatter `model` (`inherit` → this session's model), with the
   `ORCHESTRATED MODE` prompt ([`../_shared/orchestration.md`](../_shared/orchestration.md)):
   stage, slug, `--depth=<level>`, paths of `brief.md` + `run.md`. The depth is
   `orchestrator_depth`, except **`design` on a `quick` route gets `--depth=easy`** (the quick-route
   softening in size-matrix.md) unless `--depth` was passed explicitly. Then relay until the stage ends:
   - **`SDD_QUESTIONS`** → answer each per [`./references/decision-policy.md`](./references/decision-policy.md);
     append every answer to the ledger in `run.md` *first*; resume the runner with `SDD_ANSWERS`.
     In a **dashboard-driven** run, a question the policy routes to the human goes through
     `dashboard_ask`; in a terminal run through `AskUserQuestion`.
   - **`SDD_DISPATCH`** → run each requested agent (`subagent_type: "sdd:<agent>"`, the model/effort
     given — parallel when several); resume with `SDD_REPORTS` (verbatim, never summarised).
   - **`SDD_BLOCKED`** → if `needs` is a pipeline stage not yet run, run it once and retry; a second
     block on the same stage, or a block needing a human, is a **stop**.
   - **`SDD_STAGE_DONE`** → verify: the listed files exist, **every** sha in `commits` is in
     `git log`, the tree has no leftover changes from this stage, the handoff block is present,
     auto-skips carry a reason. Log the stage in `run.md`.
   Resume is `SendMessage` to the runner; when unavailable, **replay** (on-disk state + answer sheet).
6. **Route.** Take the runner's `next` and check it against `.route` + the N/A table in
   [`../_shared/size-matrix.md`](../_shared/size-matrix.md). On `standard`, the `↳ or` skip offered in
   the handoff is a decision: take it only when the N/A condition demonstrably holds (cite the file),
   and ledger it. On `quick` the stage auto-skipped already; on `full` nothing is skipped. Never skip
   a never-skippable stage (`specify`, `design`, `tasks`, `implement`, `review`, `ship`).
   **`next: glossary`** (clarify's forward handoff) → go to `design` — clarify already reconciled
   terms in-flow — unless the handoff names terms still missing from `CONTEXT.md`; then run
   `glossary` in a runner first. Ledger the choice either way.
   **Open-question sweep:** before dispatching a stage, grep spec §8 / SAD §11 for Open-Question rows
   owned by `orchestrator → human review` whose due is this stage; each one is answered now (grounded
   or UNGROUNDED, ledgered) by dispatching `clarify` (spec rows) or re-running the owning section's
   stage — or, if it cannot be grounded, it stays open and is listed in the final report.
7. **Implement as the engine lead.** For `implement`, the orchestrator is the lead the engine
   describes: runner `phase=plan` → the orchestrator runs every task through `test-author` (RED) →
   `implementer` (GREEN/REFACTOR/GATE) → commit with `SDD-Task`/`SDD-AC` trailers, parallel only where
   the engine's eligibility + lane rules allow → runner `phase=finalize`. Full protocol →
   [`./references/implement-lead.md`](./references/implement-lead.md).
8. **Review loop.** `review` runs in a runner (its `reviewer` dispatched by the orchestrator, clean
   context). The reviewer's findings are adjudicated under the **review-finding** rule in the
   decision policy (the orchestrator led the implementation, so it may not wave findings away).
   `CHANGES REQUESTED` → an implement **fix round** over the review record's findings
   (implement-lead §Fix round), then a full `review` again. After `orchestrator_max_review_loops`
   rounds still `CHANGES REQUESTED` → **stop** and report the open findings. `PASS` → `ship`.
9. **Ship + report.** Commit `run.md` as `orchestrate: <slug> run log` so the branch carries the
   ledger. `ship` runs in a runner: it verifies the feature for real, writes the changelog (+ the
   roadmap move) and **commits them** (`ship: <slug> changelog` — the orchestrated-mode exception in
   [`../_shared/orchestration.md`](../_shared/orchestration.md)), and returns the **proposed** PR
   command without running it. The go-ahead to push + open the PR is the orchestrator's:
   `orchestrator_open_pr: true` (default) → the orchestrator pushes the feature branch (plain push,
   never force) and runs the PR command itself; `false` → it prints the command for you. Then it
   appends the PR link + final status to `run.md`, commits `orchestrate: <slug> final`, pushes again if
   it opened the PR, and **emits the stage-handoff block** per
   [`../_shared/handoff.md`](../_shared/handoff.md) (terminal variant): *What I did* (stages run /
   skipped with reasons, commits, PR) + *Review* — **UNGROUNDED decisions first**, then still-open
   questions, then `run.md` (ledger), `spec.md`, `sad.md`, the PR + *Run next* (**Done** — the PR
   URL; merge is your call). On any **stop**, emit the same block with *Run next* = the exact resume
   command (`/sdd:orchestrate <slug> --resume`) and what must change first.

**Context hygiene.** The orchestrator holds the brief, the ledger and the runners' messages — not
the artifacts. It reads an artifact section only to answer a question (Grep the section, don't load
the file), and re-reads `run.md` after any compaction. Long outputs stay in the runners.

**Dashboard (optional).** When `dashboard_enabled: true` and the `dashboard_update` / `dashboard_log`
tools are present, call them at each stage start/end so the panel shows the autonomous run live.

## Stop conditions (the orchestrator halts and reports, never guesses past these)

- A gate blocks twice, or blocks on something only a human can supply (credentials, a missing repo).
- `--until=<stage>` reached (a normal, successful stop).
- The review loop cap is hit, or a red task survives the escalation ladder with `stop_on_red: true`.
- An UNGROUNDED question that the `orchestrator_escalate` policy routes to the human, when no human
  answers (headless) — record it and stop rather than invent the rule.
- A dirty working tree at start, or a runner reports a destructive action it would need (force-push,
  history rewrite, deleting user files) — never done autonomously.

## Definition of Done

- Every planned stage ran in its own `stage-runner` (or was skipped with a cited N/A reason), each
  ending in `SDD_STAGE_DONE` with its handoff block; every stage commit exists.
- Every question a runner raised has exactly one ledger row (id, stage, reversibility, kind,
  choice, grounding or UNGROUNDED); every `SDD_DISPATCH` was run and its report returned verbatim.
- `implement` ran test-first per task with gate evidence and trailers; `review` ended `PASS`
  (or the stop is reported); `ship` committed the changelog and the PR was opened (or its command
  printed, with `orchestrator_open_pr: false`) — never merged.
- `run.md` is complete; the final stage-handoff block lists UNGROUNDED decisions first.
- **Structural self-check** ([`../_shared/self-check.md`](../_shared/self-check.md)): before the final
  handoff, re-read `run.md` and assert — ledger row count = answered question count; no stage in the
  log without a DONE/SKIPPED/STOP status; every commit in the log is in `git log`; no ledger decision
  is contradicted by a later one without an explicit `supersedes D-nnn` row.

## Anti-patterns

- **Answering from vibes.** Every answer cites its grounding — or is marked UNGROUNDED. A
  plausible-sounding business rule with no source is the most expensive bug this skill can ship.
- **Letting a runner decide.** Runners return questions; they never pick «(Recommended)» for you.
- **Grading your own homework.** `critic` / `devils-advocate` / `reviewer` always run in their own
  clean-context agent — never inline in the orchestrator or the runner that wrote the draft.
- **Summarising agent reports** before handing them back to the runner — pass them verbatim.
- **Loading every artifact into the orchestrator's context.** It is the decider, not the reader.
- **Skipping a stage because it "looks trivial".** Only the N/A table in `size-matrix.md` skips.
- **Weakening a test, merging the PR, force-pushing** — never, at any setting.

## References & template

[`../_shared/orchestration.md`](../_shared/orchestration.md) (relay contract) ·
[`./references/decision-policy.md`](./references/decision-policy.md) ·
[`./references/implement-lead.md`](./references/implement-lead.md) ·
[`./templates/run.md`](./templates/run.md) ·
[`../_shared/agent-roster.md`](../_shared/agent-roster.md) · [`../_shared/size-matrix.md`](../_shared/size-matrix.md) ·
[`../_shared/tool-adapters.md`](../_shared/tool-adapters.md) (Codex / Cursor: sequential, replay-only).
