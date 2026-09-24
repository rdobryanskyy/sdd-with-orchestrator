# Checks — what the visual tester looks for, and how severe it is

Two layers run on every visual surface: the **matrix** (the feature's own AC + flows — what the
spec promised) and the **generic sweep** (what any good screen must satisfy — what nobody wrote down).

## 1. Matrix scenarios (from the artifacts)

For each `VT-n`: perform the steps, and at each checkpoint compare the screenshot with the expected
**visible** outcome — the element is present, readable, in the right state, with the right text
(in the project's language), and the flow reaches its end state. Visit the states the scenario
lists: **empty** (no data yet), **loading** (slow network if the driver can throttle), **error**
(invalid input, a failing dependency the app can simulate), **success**. A UI AC with an
authorization clause is checked with each role the matrix names.

## 2. Generic sweep (every screen reached)

| Check | What «wrong» looks like |
|---|---|
| Layout | overlapping / clipped / off-screen elements, broken grid, content hidden behind a sticky header or modal, unexpected horizontal scroll |
| Responsive | at the phone viewport: controls unreachable, tap targets < ~44 px, tables not scrollable, text not wrapping, navigation missing |
| Text | overflow / truncation without an ellipsis or tooltip — **especially for long translations** (`artifact_language` / the app's locale, e.g. Ukrainian or German strings are longer), untranslated keys (`billing.export.title`), placeholder copy, wrong number / date / currency format for the locale |
| States | no empty state, a spinner that never ends, an error shown as raw JSON / stack trace, a success with no visible feedback, a disabled control with no reason |
| Forms | no validation message, the message far from its field, input lost after an error, the submit button still active during submission (double submit) |
| Accessibility (visible basics) | text contrast clearly too low, focus outline missing when tabbing, an icon-only button with no visible label or tooltip, information carried by colour alone |
| Console / network | uncaught errors, failed requests (4xx/5xx) on a screen that looks fine, mixed-content or CORS errors (web only) |
| Design system | a hand-rolled component where the design system has one, off-token colours / spacing / typography, inconsistent with the closest existing screen (per `architecture-map.md` §Frontend) |
| Theme | if the app supports dark mode / high contrast: the same screens in that mode |
| CLI output | misaligned columns, wrapping that breaks tables at 80 columns, colour codes leaking into piped output, an error without a non-zero exit code, help text that doesn't match the behaviour |

## 3. Severity (the tester assigns it by this table, not by feel)

| Severity | Meaning | Examples |
|---|---|---|
| **blocker** | an AC's visible outcome does not happen, or the flow cannot be completed | the export button does nothing; the page crashes; a role sees data it must not |
| **major** | the AC technically happens but a user would likely fail or be misled | success toast missing so users click twice; unusable on a phone; error shown as a stack trace |
| **minor** | works and is understandable, but visibly wrong | truncated label; missing empty state; off-token colour; console error with no visible effect |
| **cosmetic** | polish | 2 px misalignment; inconsistent icon size |

`kind` values: `ac-violation`, `flow-broken`, `layout`, `responsive`, `state`, `a11y`,
`console-error`, `design-system`, `copy`. An `ac-violation` is at least **major**.

## 4. Resolution guidance (used by the skill's step 6 and the orchestrator's `visual-finding` rule)

- **Fix now** — default for blocker / major, for any `ac-violation`, and for minor findings that
  contradict the spec, the design system, or the locale.
- **Accept** — only minor / cosmetic, with a written reason (e.g. «matches the existing screen X,
  which the design system team owns»). Never for a blocker or an `ac-violation`.
- **Not a bug** — only with a cited AC / spec / SAD line showing the observed behaviour is correct, or
  proof the tester misread the screen (a second screenshot).
