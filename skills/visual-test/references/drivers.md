# Drivers — how the visual tester starts the app and drives each surface

## 1. Start the app (resolve in this order — first hit wins)

1. `visual_start_cmd` + `visual_url` in `.claude/sdd.local.md` (the escape hatch — always wins).
2. `docs/architecture-map.md` — the run/dev command and local URL/port `survey` recorded.
3. The repo's own conventions: `package.json` scripts (`dev`, `start`, `preview`, `storybook`),
   Makefile targets (`run`, `dev`, `serve`), `docker compose up` for a compose file, `manage.py
   runserver`, `dotnet run`, `go run ./cmd/<app>`, `cargo run` …
4. Still unknown → the skill asks (one `AskUserQuestion`: command + URL). Never guess.

Then the **skill** (not each tester) starts it once in the background, polls the URL (or window)
until it answers (≤ 120 s), records the exact command + URL in the report, hands the URL to the
testers (run one after another), and stops it at the end. Seed data: use the repo's seed/fixture command if one exists
(`db:seed`, `make seed`, fixtures in the test tree) — never hand-write rows into a shared database.
Stop everything you started at the end.

The tester never runs against a URL nobody gave it: in an orchestrated run an unknown start command
or URL is a `visual-launch` question the orchestrator answers only from the repo / settings / brief —
otherwise the round is `VISUAL BLOCKED`.

**Allowed targets:** `localhost` / `127.0.0.1` / a docker-compose service / an emulator / a URL the
settings explicitly name as a test environment. Anything else — especially a production host — is
refused, and the scenarios that needed it are `blocked`.

## 2. Drive it (per surface — first available driver wins)

| Surface | 1st choice | 2nd choice | 3rd choice | No driver → |
|---|---|---|---|---|
| `web-frontend` | a browser/computer-use tool in the agent's tool list (Claude in Chrome, the built-in browser, a Playwright MCP, a computer-use MCP) | **Playwright script via Bash** — headless Chromium (`PLAYWRIGHT_BROWSERS_PATH` if preset), one script per scenario: `page.setViewportSize`, user-level actions (`getByRole` / `getByText`), `page.screenshot`, collect `console` + `pageerror` + failed requests | — | `blocked` |
| `desktop-app` | a computer-use tool that sees the desktop (e.g. Windows-MCP / a computer-use MCP): launch, screenshot, click, type | the framework's own UI-test driver if the repo already has one (WinAppDriver, Playwright-Electron, Tauri driver) | — | `blocked` |
| `mobile-app` | a computer-use tool driving a running emulator/simulator | the repo's existing mobile UI-test driver (Detox, Maestro, XCUITest, Espresso) run via Bash, with its screenshots | — | `blocked` |
| `cli` | run the commands via Bash at the configured terminal width (`COLUMNS`), capture stdout/stderr; render to an image only if a tool exists (e.g. `ansi2html` + a headless browser) | plain captured text — judged as rendered output (alignment, wrapping, colour codes, truncation) | — | never blocked |

`visual_driver` (`auto | browser-tool | playwright | computer-use`) pins the choice; `auto` walks the
table. The tester states the driver it used per surface.

**Look at every screenshot.** A screenshot is evidence only after the agent has opened it (Read the
image) and compared it with the expected outcome. Saving images without looking is not a test.

## 3. Viewports

`visual_viewports` (default `[1440x900, 390x844]` — a laptop and a phone). `web-frontend` runs every
scenario at every viewport; `desktop-app` at the window sizes listed (default: maximised + the
minimum supported size if the SAD names one); `mobile-app` at the emulator's device; `cli` at
`COLUMNS=120` and `COLUMNS=80`.

## 4. Test accounts and secrets

Only what `visual_test_accounts` (settings) or the repo's seed/fixtures provide. A scenario that
needs a login nobody gave you is `blocked` («no test account for role X») — never create accounts in
a shared system, never read secrets from the developer's own browser profile beyond an already
signed-in local test session the prompt explicitly allows.

## 5. Side effects

Stop before anything that leaves the machine for real: payment, outbound e-mail / SMS / push, a
third-party production API, deleting shared data. Record it under `not_exercised` with the reason;
if the repo has a sandbox/mock mode for that integration (a mail catcher, a payment sandbox), use it
and say so.
