---
slug: <slug>
verdict: QA PASS           # QA PASS | QA FAIL
head: <sha>                # the PR head this sign-off is about
date: <YYYY-MM-DD>
qa_fix_rounds: 0
---

# QA sign-off — <slug>

<!-- Written by the orchestrator (skills/orchestrator, references/qa-gate.md). Headings, column names,
     verdict tokens and ids stay English; prose follows artifact_language. -->

## Regression run (final head)

| Tier | Command | Result | Notes |
|---|---|---|---|
| build / typecheck | `<cmd>` | pass / fail / n/a / not run — <reason> | |
| lint | `<cmd>` | | |
| unit | `<cmd>` | | <n> tests |
| integration | `<cmd>` | | |
| e2e / UI | `<cmd>` | | |
| migrations up/down | `<cmd>` | | |

## AC evidence matrix

| AC | Automated test(s) | Final-head result | Real-run check (ship) | Visual (VT-n) | Status |
|---|---|---|---|---|---|
| AC-01 | `<test file>::<name>` | pass | observed: <…> | VT-1 pass (`_visual/screenshots/r<n>/…`) | covered |

## QA audit (reviewer, verbatim)

<the reviewer's report, unedited>

## Gaps closed

| Id | Gap / finding | Path taken | Fix record / commit |
|---|---|---|---|
| QA-F1 | <…> | implement fix round / fix runner / visual re-test | <sha> |

## PR CI

- Checks: <green | red — <check> (<reason>) | pending | no CI>

## Verdict

- **<QA PASS | QA FAIL — reason>**
- Follow-ups for the human: <tiers not run here, open §8 questions, Accepted visual findings>
