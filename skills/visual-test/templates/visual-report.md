---
slug: <slug>
round: <n>
date: <YYYY-MM-DD>
verdict: VISUAL PASS         # FINAL, after resolution: VISUAL PASS (no Fix-now left) | VISUAL ISSUES | VISUAL BLOCKED
tester_verdict: VISUAL ISSUES  # the visual-tester's raw verdict, before resolution
surfaces: [<web-frontend>]
drivers: {<web-frontend>: <playwright-script (chromium headless)>}
app: {start: "<command>", url: "<url>"}
viewports: [1440x900, 390x844]
---

# Visual test — <slug> (round <n>)

<!-- instruction: written by skills/visual-test from the visual-tester report. Headings, ids,
     severities, kinds and verdict literals stay English; prose follows artifact_language. -->

## Scenarios

| ID | AC | Viewports | Result | Note |
|---|---|---|---|---|
| VT-1 | AC-3 | 1440x900, 390x844 | pass / fail / blocked | <one line> |

## Findings

<!-- instruction: one block per finding, verbatim from the tester, plus the resolution. -->

### R<n>-VF1 — <headline> · <severity> · <kind>

- **AC / scenario:** AC-3 / VT-1 · **Surface / viewport:** web-frontend / 390x844
- **Steps:** 1. … 2. …
- **Expected:** …
- **Actual:** …
- **Evidence:** ![R<n>-VF1](./screenshots/r<n>/VT-1-390x844-2.png)
- **Suspect area:** `<path>` (tester's hint)
- **Resolution:** Fix now · Accept — <reason> · Not a bug — <cited line>
  <!-- the fix record is linked back by its `finding:` field; this report is not edited after commit -->

## Not exercised

- <scenario / step> — <reason (side effect, missing account, no driver)>

## Summary

- Findings: <b> blocker · <M> major · <m> minor · <c> cosmetic — <k> Fix now · <a> Accept · <n> Not a bug
- Re-test: <required (Fix-now findings) | not required>
