# Decision policy — how the orchestrator answers a runner's questions

The orchestrator answers every `SDD_QUESTIONS` entry itself. This file is the rulebook: **where an
answer may come from, how ties break, what happens when nothing grounds it**, and the per-kind
defaults. Every answer lands in the ledger (`run.md`) *before* the runner is resumed.

## 1. Grounding sources — in precedence order (highest wins)

1. **The brief** (`_orchestrator/brief.md`) — the developer's own words. Explicit statements beat
   everything below.
2. **Earlier ledger decisions** in this run + **Accepted ADRs** — consistency with what was already
   decided. Never contradict one silently; if a later fact forces a change, write a new row
   `supersedes D-nnn` with the reason.
3. **Written upstream artifacts** of this feature (`spec.md`, `sad.md`, `data-model.md`, contracts,
   `tasks.json`) — the stage chain's own record.
4. **`docs/architecture-map.md` + the repo's existing conventions** — match the repo, copy the
   closest precedent (module layout, persistence, naming, test tooling, design system).
5. **`CONTEXT.md`** (glossary) — canonical roles and terms.
6. **The skill's own «(Recommended)» option** — acceptable only when it does not conflict with 1–5,
   and cited as `grounded: skill recommendation + <why it fits>`.

Tie-breakers when 1–6 leave two options standing: **reversible over irreversible** → **smaller
scope (the MVP rule in size-matrix.md)** → **no new dependency / datastore / service** → **the
option the existing repo already uses**.

## 2. When nothing grounds the answer (UNGROUNDED)

Classify the question from the runner's `business_rule` + `reversibility` fields, then apply
`orchestrator_escalate` (settings; `--escalate=` overrides per run):

| Question | `none` (default) | `business` | `hard` |
|---|---|---|---|
| technical, reversible | decide (tie-breakers), ledger `UNGROUNDED` | decide, ledger | decide, ledger |
| technical, irreversible | decide (most conservative), ledger `UNGROUNDED` | decide, ledger | **ask the human** |
| business rule (any) | decide the **most conservative** option, ledger `UNGROUNDED — BUSINESS` | **ask the human** | **ask the human** |

- **Most conservative** = the option that commits to the least behaviour: deny over allow, no
  automation over automation, keep existing behaviour over change it, narrower scope over wider,
  an explicit error over a silent fallback.
- **Ask the human** = one `AskUserQuestion` from the orchestrator itself (the runner's question
  verbatim, plus one line on why it could not be grounded); batch pending human questions (≤4).
  Headless / no answer → **stop** (SKILL.md stop conditions), never guess.
- With `none`, UNGROUNDED business rules are still visible: they are listed **first** in the final
  handoff, so the developer reviews exactly the decisions the orchestrator had to invent.

## 3. Per-kind defaults

| `kind` | How the orchestrator answers |
|---|---|
| `depth` | Not asked — the orchestrator passes `--depth=<orchestrator_depth>` on every Q&A stage. If a runner still asks, answer with that level. |
| `calibration` (survey greenfield: how do you want to engage) | «Pick sound defaults for me» — the orchestrator is the engaged party; every foundational pick it then answers is ledgered, irreversible ones become the foundational ADRs survey writes. |
| `size-route` | Score the four signals in size-matrix.md from the brief (+ the map); take the matrix default for size **and** route. Never `quick` for L/XL. Ledger the dominant signal. |
| `socratic` (4-state) | **Approve** when the decision is grounded and consistent. **Edit** (with the new wording) when it contradicts the brief, a ledger row or the repo. **Save as Open Question** only when no downstream stage depends on the answer — owner `orchestrator → human review`, due a stage trigger; the SKILL.md step 6 open-question sweep revisits it at that stage, and if it still can't be grounded it is listed in the final report as a human follow-up. **Drop** only for an optional decision the brief rules out, with the reason. |
| `critic` / devils-advocate finding | Accept the revert/amendment when the finding cites the brief or an upstream artifact and is correct on re-read. Override only with a cited rationale (it becomes the §1 ¶4 override bullet the skill writes). Never dismiss an F4 (silent edit) finding. |
| `channel` (which extra sources to read) | Only what the brief names (a ticket, a doc path, a reference module); plus the closest precedent module from the architecture map. Otherwise `none`. No broad scans. |
| `ledger-veto` (easy-depth assumptions) | Accept all, except any assumption that contradicts the brief or an earlier ledger row — veto those with the correct value. |
| `blast-radius` (→ ADR) | Approve the ADR when the decision is irreversible or cross-module; the orchestrator never down-scopes an ADR to avoid writing it. |
| `clarify-finding` (Resolve now / Defer / Not an ambiguity) | **Resolve now** with the tightened wording whenever §1 grounds it. **Defer** (owner + due, as for Save as OQ) only when nothing grounds it and no downstream stage depends on it. **Not an ambiguity** only with a cited artifact line proving the text already has one reading. |
| `review-finding` (Fix now / Defer / Not an issue) | The orchestrator led the implementation, so it must not wave its own work through. Default **Fix now**. A stage-1 finding (an AC not met / not traced) is **never** Defer or Not-an-issue. A stage-2 (quality) finding may be **Not an issue** only with a cited artifact or repo convention that makes it correct, and **Defer** only for a non-blocking quality item, with owner + due. |
| `glossary-definition` (free-text definition, NOT-reference) | Write the definition from the brief / spec / map in the project's own words (`CONTEXT.md` style); no source → UNGROUNDED (BUSINESS if it defines domain behaviour). |
| `test-level` (plan-tests: which tier covers an AC) | The lowest tier that observes the AC's behaviour (unit < integration < e2e), plus the surface's UI tier for a UI AC (per surfaces.md); match the repo's existing test layout. |
| `visual-finding` (Fix now / Accept / Not a bug) | Default **Fix now** — for every blocker / major, every `ac-violation`, and every minor that contradicts the spec, the design system or the locale. **Accept** only a minor / cosmetic finding with a written reason grounded in the repo (e.g. it matches the existing screen the design system owns) — never a blocker or an `ac-violation`. **Not a bug** only with a cited AC / spec / SAD line showing the behaviour is correct. Same rules as `skills/visual-test/references/checks.md` §4. |
| `fix-intake` (fix step 1) | Answer from the finding / bug report itself; a `VF-n` needs nothing more. Missing reproduction detail → UNGROUNDED; never invent steps. |
| `fix-spec-patch` (fix step 5: confirm an AC patch or a new AC) | Approve when the brief grounds the corrected behaviour. Otherwise UNGROUNDED — BUSINESS, and `orchestrator_escalate` applies as everywhere: with `none` the orchestrator still decides — it approves the **most conservative** wording (spec-bug: the narrower reading that matches the expected behaviour in the finding; gap: the new AC stating exactly the expected visible outcome, marked `added-by-fix`), ledgers it UNGROUNDED — BUSINESS, and adds a spec §8 open-question row (owner `orchestrator → human review`, due «before merge») so it appears in the final report. |
| `visual-launch` (visual-test: how to start the app / which URL) | Only from settings, `architecture-map.md`, the repo's scripts/compose files, or the brief — cite it. Nothing grounds it → never invent a command or URL: answer «unknown», the round is `VISUAL BLOCKED`, and the stop tells the human which setting to fill (`visual_start_cmd` / `visual_url`). |
| `other` | Apply §1 → §2. |

## 4. Answer quality

- **Cite, specifically.** `grounded: brief ¶2 "only admins can…"`, `grounded: D-009`,
  `grounded: architecture-map §Persistence (Postgres, sqlc)` — not `grounded: context`.
- **Edit answers carry the full new wording** the runner will write — never «make it better».
- **Stay product-level where the stage is product-level.** Answers to `specify` / `clarify` never
  introduce tech names into acceptance criteria (the spec's forbidden-token rule still applies).
- **One run, one voice.** Terms, role names and numbers used in an answer match `CONTEXT.md` and
  earlier ledger rows exactly.
