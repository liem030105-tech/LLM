# Build log

The single source of truth for progress. **Append-only:** never edit or delete
an earlier entry. Record failed attempts and corrections too — the history is the
point.

A phase is `Complete` only when every acceptance criterion in its build file has
observed evidence below. A plan is not evidence. The existence of code is not
evidence.

Entries are written by `qa` through `/verify`, and by `/retro` when the framework
itself changes.

## Entry format

```
## Phase <NN> — <title> — <ISO date>

Status after this run: Not started | In progress | Blocked | Complete

### Commands run
<command>
<real output, trimmed to what matters>

### Acceptance criteria
- [x] <criterion> — evidence: <what the output showed>
- [ ] <criterion> — unverified: <why>

### Findings
- <file:line> — <problem> — <how to reproduce>
```

---

## Framework — initial setup — 2026-11-13

Status after this run: Complete

Created the working framework before any project code:

- `.claude/agents/` — `planner` (opus), `coder` (sonnet), `qa` (sonnet),
  `reviewer` (opus).
- `.claude/commands/` — `/phase-plan`, `/phase-run`, `/verify`, `/pre-push`,
  `/retro`.
- `GOALS.md`, `PLANS.md`, nine build files under `harness/build/`, this log.

Structure follows two references: subagent format and model tiers from
VoltAgent/awesome-claude-code-subagents; the phase, gate, and evidence loop from
the OpenAI cookbook on iterating development workflows with Codex.

No project code exists yet. All nine phases are `Not started`.

## Framework — fixes from the first trial run — 2026-11-13

Status after this run: Complete

Trial run of `/phase-plan 01` in the desktop app. `planner` returned a full
build file for phase 01, written to `harness/build/01-shop-backend-streaming.md`
with `Status: Not started`. It is awaiting the user's scope approval.

### Commands run
git check-ignore -v harness/build/01-shop-backend-streaming.md
.gitignore:20:build/	harness/build/01-shop-backend-streaming.md

### Findings
- `.gitignore:20` — the unanchored `build/` ignored `harness/build/`, so no
  build file could ever be committed. Fixed: replaced with `/build/` and
  `BE/build/`. After the fix, `git check-ignore` reports the build files as not
  ignored, and `git status` lists all nine.
- `.claude/commands/{phase-plan,phase-run,verify,retro}.md` — positional `$1`
  was not substituted when the command ran from the desktop app: the text kept
  `$1` literally and the argument was appended as `ARGUMENTS: 01`. Fixed:
  replaced every `$1` with `$ARGUMENTS`. Unverified until the next command run.
- Not fixed, for `/retro 01`: the phase 01 plan came out at 12 steps and 15
  criteria for two weeks, so `planner`'s "Right-size the plan" check may be too
  loose. `qa`'s description says it writes only under `BE/tests/` and
  `BE/eval/`, but its body also allows `harness/build-log.md`.

## Framework — fixes from the first pre-push review — 2026-11-13

Status after this run: In progress

Corrects the previous entry: its `Complete` status was premature, because the
`$ARGUMENTS` fix it records is still unverified. This entry supersedes that
status; the earlier entry is left as written.

`/pre-push` on the framework-only diff returned `PASS` with non-blocking
findings. The user chose to fix them before the first commit. Phase 01 scope
was approved the same day; see
`harness/context/phase-01-shop-backend-streaming-context.md`.

### Commands run
git check-ignore -v --no-index harness/build/01-shop-backend-streaming.md
(no output, exit 1 — not ignored)
git check-ignore -v --no-index .env .claude/settings.local.json BE/build/x
.gitignore:2:.env	.env
.gitignore:7:.claude/settings.local.json	.claude/settings.local.json
.gitignore:22:BE/build/	BE/build/x
grep -rnE '\$[0-9]' .claude/
(no output, exit 1 — no positional placeholders left)
git status --short --untracked-files=all -- harness/build | wc -l
9

### Findings
- `.claude/agents/coder.md`, `.claude/commands/phase-run.md` — both banned every
  edit under `BE/eval/`, yet phases 04–06 build it. Fixed: `coder` may create or
  extend `BE/eval/` files only when the build file's Steps name them, and never
  edits an existing question, attack case or scoring rule once a baseline exists.
  `coder.md` owns the rule; `phase-run.md` points to it.
- `.claude/agents/qa.md` description and the `CLAUDE.md` agent table said `qa`
  writes only tests and eval. Fixed: both now include build-log entries.
- `.gitignore` — `.claude/settings.local.json` was not ignored and can hold
  secrets. Fixed: added under `# Secrets`.
- `CLAUDE.md`, `PLANS.md` — `/phase-plan` was called "read-only" although the
  main session writes the build file. Fixed: "no code", and the approval gate
  reads "before any code is written".
- `harness/build/01-shop-backend-streaming.md` — two out-of-date lines (no
  context file; the `build/` problem) corrected. Scope unchanged.
- Open: `$ARGUMENTS` substitution is still unverified until the next slash
  command with an argument runs.
- Open, for `/retro 01`: the commit/push and read-only rules for agents are
  enforced by prompt text only. A checked-in `.claude/settings.json` with `ask`
  rules for `git commit`, `git push`, `git add` was proposed; the user decides.
- Open, for `/retro 01`: phase 01's plan is large for two weeks (12 steps,
  15 criteria).
