---
slug: <slug>
status: running            # running | stopped | done
started: <YYYY-MM-DD>
depth: medium              # the --depth passed to every Q&A stage
escalate: none             # none | business | hard
orchestrator_model: <model id or tier alias>
branch: <branch>
---

# Orchestrated run — <slug>

<!-- Written and maintained by the orchestrator (skills/orchestrate). Re-read on --resume and after
     any context compaction: this file, not the conversation, is the run's memory. Headings, column
     names and status tokens stay English; prose follows artifact_language. -->

## Brief

See `brief.md` in this folder (verbatim). One-line intent: <the orchestrator's one-line reading of the brief>.

## Stage log

| # | Stage | Status | Runner turns | Commit | Notes |
|---|---|---|---|---|---|
| 1 | survey | DONE / SKIPPED (<reason>) / BLOCKED / STOP | <n> | <sha> | <files, auto-skips, anomalies> |

## Decision ledger

<!-- One row per answered question, appended BEFORE the runner is resumed. Grounding is a specific
     citation, or UNGROUNDED (+ BUSINESS when business_rule). Superseding rows name the row they replace. -->

| ID | Stage | Question id | Reversibility | Kind | Decision | Grounding |
|---|---|---|---|---|---|---|
| D-001 | specify | specify.size-route | easy | tech | size M, route standard | grounded: brief ¶1 (new API + migration) |

## Agent dispatches

| Stage | Agent | Model / effort | Result |
|---|---|---|---|
| specify | critic | opus / high | 2 findings → both accepted (D-014, D-015) |

## Implementation

| Task | RED | GREEN / gate | Commit | Notes |
|---|---|---|---|---|
| T1 | GOOD red (`<quoted line>`) | gate clean | <sha> | |

## Review rounds

| Round | Verdict | Findings | Fix tasks |
|---|---|---|---|
| 1 | CHANGES REQUESTED / PASS | <n> | R1-F1, R1-F2 |

## Outcome

- Status: <done | stopped — reason>
- PR: <url or the command to open it>
- **UNGROUNDED decisions to review first:** D-0nn, D-0nn
- Resume (if stopped): `/sdd:orchestrate <slug> --resume` after <what must change>
