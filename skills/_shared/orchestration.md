# Orchestration — the relay contract between the orchestrator and its stage agents

> **Reference-only.** Not a skill. Defines how [`../orchestrate/SKILL.md`](../orchestrate/SKILL.md)
> (the **orchestrator** — the main session, the only decision-maker) drives every pipeline stage
> through a [`stage-runner`](../../agents/stage-runner.md) agent, and how that agent hands its
> questions and sub-agent dispatches **up** instead of asking the human or spawning agents itself.
> The stage skills are **not** rewritten: they run unchanged; only the three host mechanisms below
> are redirected while `ORCHESTRATED MODE` is on.

## TL;DR

- One **orchestrator** (main session, strongest model) owns: the brief, every decision, every
  sub-agent dispatch, routing, commits of record, and the run log.
- Each stage (`survey`, `specify`, …, `ship`) runs in its own fresh **`stage-runner`** agent (this *is*
  the `/clear` between stages). The runner follows `skills/<stage>/SKILL.md` verbatim.
- Where the skill would call `AskUserQuestion` → the runner **returns `SDD_QUESTIONS`** and stops.
- Where the skill would dispatch a sub-agent (`subagent_type: "sdd:<name>"`) → the runner **returns
  `SDD_DISPATCH`** and stops (a subagent cannot spawn subagents; the lead owns fan-out).
- The orchestrator answers / runs the dispatches and **resumes the same runner** with
  `SDD_ANSWERS` / `SDD_REPORTS`. Repeat until the runner returns **`SDD_STAGE_DONE`** (or
  **`SDD_BLOCKED`**).
- Every answer the orchestrator gives is recorded in the **decision ledger** with its grounding.

## The three redirected mechanisms (stage-runner side)

| The skill says… | Interactive run | `ORCHESTRATED MODE` (stage-runner) |
|---|---|---|
| `AskUserQuestion` (any: depth dial, Socratic 4-state, critic resolution, channel pick, ledger veto) | ask the human | emit `SDD_QUESTIONS` as the **final message** of this turn; wait to be resumed |
| dispatch `sdd:<agent>` (critic, researcher, strategist, analyst, devils-advocate, explorer, reviewer) | spawn it | emit `SDD_DISPATCH` with the **complete** prompt the skill would have sent; wait |
| `/clear` + next command in the handoff | the human copies it | not executed — the handoff block is returned inside `SDD_STAGE_DONE`; the orchestrator routes |
| `ship`'s proposed PR command (never run unasked) | the human runs it | the runner **commits** the changelog + roadmap move (`ship: <slug> changelog`) and returns the command in `pr_command` without running it; the orchestrator gives (or withholds) the go-ahead per `orchestrator_open_pr` and runs push + PR itself |

Everything else is unchanged: gates still hard-refuse, artifacts are written exactly as the skill
says, the depth dial / size / route semantics hold, the coverage floors hold, the handoff block is
still produced (returned, not printed to a human). An **inline skill call** a stage makes
(`glossary`, `roadmap`, `classify-size` run *inside* `specify`) stays inline in the runner — it is a
file read, not an agent spawn.

**Commits.** Where a skill *proposes* a commit, the runner **makes** it with the proposed message,
so every stage's work is committed — possibly as several commits (`design` commits a bootstrap,
then one per section, then the finalisation; inline `glossary` / `roadmap` add their own).
`auto_commit` governs only the `implement` engine's task commits, not stage commits. The `implement` engine's per-task
commits are made by the orchestrator (it is the engine lead — see
[`../orchestrate/references/implement-lead.md`](../orchestrate/references/implement-lead.md)).

## Message formats (fenced YAML, so both sides parse them without guessing)

### Runner → orchestrator

```yaml
SDD_QUESTIONS:
  stage: design                 # the skill being run
  slug: checkout-discounts
  step: "4 §5 building blocks"  # where in the skill's protocol the run paused
  questions:                    # 1–4 per message; batch only questions that are independent
    - id: design.s5.module-boundary      # stable: <stage>.<section>.<short> — reused on replay
      kind: socratic            # depth | size-route | calibration | socratic | blast-radius | critic | clarify-finding
                                # | review-finding | glossary-definition | test-level | channel | ledger-veto | other
      reversibility: hard       # easy | hard  (hard = irreversible / multi-module / public contract)
      business_rule: false      # true when the answer defines product/domain behaviour, not tech
      question: "<the full CONTEXT / WHY IT MATTERS / READ OPTIONS text, per ask-style.md>"
      multi_select: false
      options:
        - label: "Approve new `discounts` module (Recommended)"
          description: "<the full 4-element description, per ask-style.md>"
        - label: "Edit"
          description: "…"
      grounding: ["docs/architecture-map.md §Modules", "spec.md §5 AC-3"]   # what the runner read
```

The question text and options are **exactly** what the skill would have shown a human (full
[`ask-style.md`](./ask-style.md) shape) — the orchestrator is a reader who did not see the runner's
context, so a dry question is as much a defect here as in an interactive run.

```yaml
SDD_DISPATCH:
  stage: specify
  slug: checkout-discounts
  dispatches:                   # run in parallel when more than one
    - agent: critic             # an agents/<name>.md — dispatched as subagent_type "sdd:<name>"
      model: opus               # the model the skill / roster / settings resolved (incl. judgment_model)
      effort: high              # incl. the L/XL xhigh bump
      prompt: |
        <the complete prompt, self-contained — inlined draft, edits-log, paths to Read>
```

```yaml
SDD_STAGE_DONE:
  stage: specify
  slug: checkout-discounts
  outcome: done                 # done | plan (implement phase=plan) | PASS | CHANGES REQUESTED (review — the verdict literals)
  files: [docs/features/checkout-discounts/spec.md, docs/features/checkout-discounts/.size, docs/features/checkout-discounts/.route]
  commits: ["<sha> spec: checkout-discounts"]       # every commit this stage made, in order ([] if none)
  pr_command: null              # ship only: the proposed PR command, NOT run
  next: clarify                 # the stage the handoff's Run next resolved to (route-aware)
  skipped: []                   # e.g. [{stage: clarify, reason: "zero §8 open questions"}]
  assumptions: []               # the easy-depth assumptions ledger, verbatim, if any
  handoff: |
    <the stage-handoff block, verbatim>
```

```yaml
SDD_BLOCKED:
  stage: api
  slug: checkout-discounts
  reason: "gate refused: schema change but no data-model.md"
  needs: data-model             # the stage (or human input) that would unblock it
```

### Orchestrator → runner (the resume message)

```yaml
SDD_ANSWERS:
  - id: design.s5.module-boundary
    choice: "Approve new `discounts` module (Recommended)"   # an option label verbatim, or…
    text: null                  # …free text when the choice is Edit / Other (the new wording)
    rationale: "brief §2 asks for isolated pricing rules; map shows no existing pricing module"
    user_reason: null           # required for Edit / Drop / Save as OQ / Defer — goes verbatim into the edits-log
    owner: null                 # Save as OQ / Defer only — e.g. "orchestrator → human review"
    due: null                   # Save as OQ / Defer only — a stage trigger ("before `sdd:tasks`") or a date
```

For an Edit / Drop / Save-as-Open-Question (and `clarify`'s Defer) the orchestrator always fills
`user_reason` — the edits-log needs it verbatim — and for Save as OQ / Defer the **owner + due**, so
the socratic-loop rule «no owner/due ⇒ Drop» is never hit by accident.

```yaml
SDD_REPORTS:
  - agent: critic
    report: |
      <the agent's final message, verbatim — never summarized>
```

## Resuming a runner (and the replay fallback)

1. **Preferred — resume in place.** The orchestrator sends `SDD_ANSWERS` / `SDD_REPORTS` to the same
   runner (`SendMessage` to its agent id / name). The runner keeps its in-memory section state, its
   edits-log and its place in the protocol.
2. **Fallback — replay.** When the host cannot continue a finished agent (no `SendMessage`, the
   agent expired, Codex/Cursor), the orchestrator dispatches a **fresh** runner with an **answer
   sheet**: every `SDD_ANSWERS` + `SDD_REPORTS` entry given so far for this stage, plus the stage's
   commits so far. Stages write to disk *during* their walk (`specify` writes `.size`/`.route` early,
   `design` bootstraps `sad.md` and commits each section, `sequences` writes each flow), so a replay
   **resumes from on-disk state, it does not restart**: the runner reads what this stage already
   wrote and committed, treats those steps/sections as done (never re-bootstraps a template over a
   written file, never re-commits), rebuilds the in-progress section's state from the sheet — applying
   sheet answers by `id` silently and reusing sheet reports by agent+step — and stops only at the
   first *new* question id or dispatch. Stable ids + the document-is-the-state property make this
   deterministic.

A runner never answers its own question, never picks the «(Recommended)» option on the
orchestrator's behalf, and never runs an agent inline to avoid a dispatch — except under the replay
sheet, or when the orchestrator explicitly answers `SDD_REPORTS: inline` for a host without agents.

## The decision ledger (orchestrator side)

Every answer is appended to `docs/features/<slug>/_orchestrator/run.md` (template:
[`../orchestrate/templates/run.md`](../orchestrate/templates/run.md)) before the resume is sent:

```
| D-017 | design | design.s5.module-boundary | hard | tech | Approve new `discounts` module | grounded: brief §2, architecture-map §Modules |
| D-018 | specify | specify.s5.refund-window | hard | BUSINESS | 14 days | UNGROUNDED — conservative default; not in brief — review |
```

- **grounded** — the answer follows from the brief, a written artifact, `CONTEXT.md`, an Accepted ADR,
  or the repo's existing conventions (cite which).
- **UNGROUNDED** — the orchestrator had to choose without a source. Always flagged; always listed
  first in the final report. A `business_rule: true` question that is UNGROUNDED is governed by
  `orchestrator_escalate` (see [`../orchestrate/references/decision-policy.md`](../orchestrate/references/decision-policy.md)).

The ledger is also the orchestrator's memory: after a context compaction, or on `--resume`, it
re-reads `run.md` instead of relying on conversation history.

## Why this shape

- **One decider.** Decisions made by many agents drift; one orchestrator with a written ledger keeps
  the spec, SAD, contracts and tasks mutually consistent — and gives a human one place to audit.
- **Fresh context per stage.** Each runner starts empty and re-reads its inputs from disk — the same
  property `/clear` gives the interactive pipeline — so one stage's chatter never leaks into the next,
  and the orchestrator's own context holds only reports, questions and the ledger.
- **Independence preserved.** `critic` / `devils-advocate` / `reviewer` are still dispatched with a
  clean context by the orchestrator; the runner that wrote the draft never grades it.
- **No skill rewrites.** The redirection lives here and in the runner; the 19 stage skills remain
  valid interactive skills.
