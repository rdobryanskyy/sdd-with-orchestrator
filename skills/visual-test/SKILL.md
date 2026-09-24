---
name: visual-test
model: opus
effort: high
agents: [visual-tester]
description: >
  Use at the very end of the pipeline (after ship) to test a feature VISUALLY on the running app —
  a visual-tester agent launches the app locally and drives it through computer use / a real
  browser, walks every UI acceptance criterion and UI flow at every viewport, runs a generic visual
  sweep (layout, responsive, empty/loading/error states, overflow, contrast, console errors,
  design-system conformance), and reports findings with screenshots. Each finding is resolved Fix
  now (→ /sdd:fix) / Accept / Not a bug; the report lands in docs/features/{slug}/_visual/.
  Triggers on "visual test {slug}", "test the UI of {slug}", "click through {slug}", "check how
  {slug} looks", "/sdd:visual-test {slug}", "run the visual tester", "computer-use test". Skips
  itself (N/A, with the reason) when the feature has no visual surface.
---

# Skill: visual-test

The pipeline's **last gate**: after `ship` has verified the feature runs and proposed the PR,
`visual-test` looks at it the way a user will. Unit, integration and e2e tests prove behaviour the
tests were written for; this stage catches what nobody wrote a test for — a button hidden on a
phone, a toast that never appears, a table overflowing in Ukrainian, a spinner that never stops, a
screen that ignores the design system.

The skill builds the **visual test matrix** from the artifacts, dispatches the
[`visual-tester`](../../agents/visual-tester.md) agent (`subagent_type: "sdd:visual-tester"`) to run
it on the live app, writes the report, and resolves every finding. It never edits code — each
*Fix now* finding goes to [`fix`](../fix/SKILL.md) as a bug report (Accept / Not a bug are the only
other resolutions). Drivers, launch and safety →
[`./references/drivers.md`](./references/drivers.md) · what to check + severity →
[`./references/checks.md`](./references/checks.md) · report scaffold →
[`./templates/visual-report.md`](./templates/visual-report.md).

Report prose follows `artifact_language` (verdict literals, finding ids, severities and headings
stay English) → [`../_shared/artifact-language.md`](../_shared/artifact-language.md). Question
phrasing → [`../_shared/ask-style.md`](../_shared/ask-style.md).

## Owner

QA / the Tech Lead who shipped the feature (drives); the PM confirms any finding that turns out to
be a spec question.

## Inputs

- `<slug>` — feature slug.
- **Gate (hard refuse):** `spec.md` + `sad.md` with `target_surfaces`. Missing → «run `specify` /
  `design` first». **Soft gate:** a `PASS` review + a shipped (or ship-verified) change — without
  them, warn that findings may just be unfinished work, then proceed.
- **N/A (auto-skip):** no visual surface in `target_surfaces` — only `backend-service` / `worker` /
  `library-sdk`. The visual surfaces are `web-frontend`, `mobile-app`, `desktop-app` (screens) and
  `cli` (rendered terminal output). Print «auto-skipped visual-test: no visual surface
  (<surfaces>)» in the handoff and stop.
- Read: spec §5 AC, `sad.md` §6 UI-driven flows, `test-plan.md` (or the inline `## Test plan`) e2e
  rows, `docs/architecture-map.md` §Frontend (design system, tokens, how the app starts),
  `CONTEXT.md`, earlier `_visual/` reports + `_fixes/` (for re-test rounds).
- Settings (`.claude/sdd.local.md`, auto-created with documented defaults if absent →
  [`../implement/references/settings.md`](../implement/references/settings.md)): `visual_start_cmd`,
  `visual_url`, `visual_viewports`, `visual_driver`, `visual_test_accounts`, `model_visual_tester`.

## Protocol

1. **Gate + N/A.** Check the gate; read `target_surfaces`; auto-skip when no visual surface.
2. **Build the visual test matrix.** One scenario `VT-n` per UI-observable §5 AC (every AC whose
   outcome a user can see on a visual surface — per [`../_shared/surfaces.md`](../_shared/surfaces.md)),
   plus one per §6 UI-driven flow not already covered, plus the e2e-through-UI rows of the test plan.
   Each scenario: the AC it proves, user-level steps, the expected **visible** outcome, the states to
   visit (empty / loading / error / success), the viewports (`visual_viewports`; `cli` → one terminal
   width). The matrix must cover **every** UI-observable AC — coverage floor, not a dial.
3. **Resolve how to run the app** per [`./references/drivers.md`](./references/drivers.md): settings
   → `architecture-map.md` → `package.json` / Makefile / manifests. Unknown → one `AskUserQuestion`
   (the command + URL; orchestrated kind `visual-launch`) — never guess. Then **start the app once**
   (seed it, wait until it answers); the skill owns its lifecycle and stops it after step 4.
4. **Dispatch the tester.** `subagent_type: "sdd:visual-tester"` (model `model_visual_tester`,
   default `sonnet`; `opus` for L/XL per [`../_shared/agent-roster.md`](../_shared/agent-roster.md)) with
   a self-contained prompt: slug, round `r<n>` (1 + the number of earlier `_visual/` reports), the
   matrix, surfaces, start command / URL, viewports, driver preference, test accounts, the screenshot
   dir `docs/features/<slug>/_visual/screenshots/r<n>/`, the app URL (already running), and the paths
   of `drivers.md` + `checks.md`. Finding ids are round-qualified `R<n>-VF<k>`. Several visual
   surfaces → one tester per surface, **one after another** (one app instance, one desktop, one
   browser session — parallel testers would fight over them). No computer-use / browser tool and
   no Playwright → the tester returns `VISUAL BLOCKED`; record it, never downgrade to «looks fine».
5. **Write the report.** `docs/features/<slug>/_visual/visual-test-<date>.md` (a second run the same
   day → suffix `-r<n>`, never overwrite) from [`./templates/visual-report.md`](./templates/visual-report.md):
   driver, app, the scenario table (every VT-n with pass/fail/blocked), findings verbatim with
   screenshot links, `not_exercised`, and the tester's raw verdict.
6. **Resolve each finding** — one `AskUserQuestion` per finding (screenshot path, expected vs actual,
   severity in the question), per [`../_shared/ask-style.md`](../_shared/ask-style.md):
   **Fix now** (→ becomes a `/sdd:fix` bug report: the steps, expected, actual, screenshot, AC) /
   **Accept** (a known, harmless deviation — recorded with the reason; never for a blocker or an
   `ac-violation`) / **Not a bug** (the tester misread the screen or the spec allows it — cite the
   AC/spec line). Record each resolution in the report, then set the **final verdict**: any *Fix now*
   → `VISUAL ISSUES`; none left (no findings, or all Accepted / Not a bug) → `VISUAL PASS` (Accepted
   findings stay listed); nothing could run → `VISUAL BLOCKED`.
7. **Commit.** Add the report + the screenshots referenced by findings (passing checkpoints
   `pass-*.png` stay local — the skill adds `docs/features/*/_visual/screenshots/**/pass-*.png` to
   `.gitignore` once). On a final `VISUAL PASS`, **move the feature to Shipped** in `docs/roadmap.md`
   (via `roadmap` — `ship` leaves a visual-surface feature in *Now* as «visual test pending») and draft
   a one-paragraph PR note (verdict, rounds, fixes, accepted findings) with the proposed
   `gh pr comment` / `glab mr note` command — proposed, not run. Propose commit
   `visual-test: <slug> <verdict> (r<n>)`.
8. **Handoff.** **Emit the stage-handoff block** per [`../_shared/handoff.md`](../_shared/handoff.md)
   (terminal variant): *What I did* (verdict, scenarios run/blocked, findings by severity + resolution,
   driver) + *Review* (the report + finding screenshots) + *Run next*: `VISUAL PASS` → **Done** (the
   PR; merge is your call); `VISUAL ISSUES` → one `/sdd:fix <slug> "<VF-n: expected … / actual …>"`
   per *Fix now* finding, then `/sdd:visual-test <slug>` to re-test (and `/sdd:review <slug>` if the
   fixes were wide); `VISUAL BLOCKED` → what to install/configure (driver, start command, test account),
   then `/sdd:visual-test <slug>` again.

**Orchestrated mode.** Under `/sdd:orchestrate` the tester dispatch goes up as `SDD_DISPATCH`, the
finding resolutions as `SDD_QUESTIONS` (kind `visual-finding`), and the orchestrator runs the fix
loop itself — each *Fix now* finding in its own `fix` runner, then a re-test round →
[`../orchestrate/SKILL.md`](../orchestrate/SKILL.md).

## Definition of Done

- Either auto-skipped with a stated N/A reason, or: every UI-observable AC has a scenario, every
  scenario is pass / fail / blocked with evidence, and a report with a verdict exists.
- Every finding has a screenshot, steps, expected vs actual, a severity, and exactly one resolution.
- No code, test or spec was changed by this stage — fixes go through `fix`.
- **Structural self-check** ([`../_shared/self-check.md`](../_shared/self-check.md)): before the
  handoff, assert — scenario count ≥ UI-observable AC count; every finding's screenshot file exists;
  every finding has a resolution; `Accept` never used on a blocker / `ac-violation`; the final verdict
  matches the resolutions (no Fix-now left ⇒ PASS, any Fix-now ⇒ ISSUES); finding ids are `R<n>-VF<k>`
  and none repeats an earlier round's id.

## Anti-patterns

- **Testing through the API** instead of the screen — the whole point is what a user sees.
- **A pass without a screenshot** someone actually looked at.
- **Running against production** or with real customer data / credentials — ever.
- **Fixing in this stage.** The tester and the skill report; `fix` pins each bug with a failing test.
- **Accepting an AC violation** because «it's just visual» — a visible AC is still an AC.
- **Guessing the start command / URL** — ask, or read it from the map; a wrong URL tests the wrong app.

## References & template

[`./references/drivers.md`](./references/drivers.md) · [`./references/checks.md`](./references/checks.md) ·
[`./templates/visual-report.md`](./templates/visual-report.md) · [`../fix/SKILL.md`](../fix/SKILL.md) ·
[`../_shared/surfaces.md`](../_shared/surfaces.md) · [`../_shared/tool-adapters.md`](../_shared/tool-adapters.md).
