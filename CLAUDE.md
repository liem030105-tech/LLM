# CLAUDE.md

Durable guidance for Claude Code in this repository.

## Repository

- **GitHub:** https://github.com/liem030105-tech/LLM.git
- **Local path:** `C:\Users\Admin\Desktop\ShopAsssitant`

## Project

**Shop Assistant** — a customer-support assistant for an online shop, built as
the practice project of a 12-week AI Engineer roadmap
(`Lo-trinh-AI-Engineer-12-tuan.pdf`).

The assistant does two things:

1. **Retrieval** — answers policy questions (returns, warranty, shipping) from a
   Vietnamese document corpus, always with citations.
2. **Tool calling** — performs real actions on the shop system: order lookup,
   refund requests, handoff to a human.

Outcomes and non-goals: `GOALS.md`. Phase sequence and status: `PLANS.md`.
Detailed week-by-week notes in Vietnamese: `KE-HOACH-CHI-TIET.md`.

## Tech stack

| Component | Choice |
|---|---|
| Language, framework | Python + FastAPI (async) |
| Database | PostgreSQL + pgvector |
| LLM | One hosted LLM API (Claude / GPT / Gemini) |
| Tracing | Langfuse |
| Deployment | Docker, cloud host with a public demo link |
| Frontend | A single chat page with streaming (SSE) |

## Layout

```
BE/         FastAPI backend — app/, tests/, eval/
DB/         Alembic migrations, seed scripts, docs/ (Vietnamese policy corpus)
FE/         streaming chat page
.claude/    agents/ and commands/ — the working framework
harness/    build/ (one file per phase), context/, build-log.md
```

## Hard rules

- **Authorization lives in the tool layer, never in a prompt.** Every query over
  order, user, or refund data filters by the `user_id` from the session — never
  from a parameter the model produced.
- **The agent loop is hand-written.** No LangChain or LangGraph until phase 09,
  and then only on a separate branch that never merges.
- **Tools do not raise.** They return `{"error": ..., "hint": ...}` so the model
  can recover.
- **Consequential actions are two-phase** — refund creation, human handoff:
  dry-run → user confirmation → execute, with replay protection.
- **Retrieval changes are toggleable** through `RAG_MODE`
  (`vector` | `hybrid` | `hybrid_rerank`), so each can be measured.
- **Evaluation is never skipped.** Every improvement needs before-and-after
  numbers. Existing eval questions are frozen once a baseline exists.
- **Every LLM call records** tokens, cost, and latency.
- **No customer personal data** in logs, traces, fixtures, or prompts.

## Conventions

- Migrations via **Alembic**; never `create_all`.
- Seed scripts are idempotent.
- Secrets in `.env` (never committed); every setting mirrored in `.env.example`.
- Lint and format with `ruff`; type-check with `mypy`; test with `pytest`.
- **Language:** code, prompts, and framework files are in English. The policy
  corpus in `DB/docs/` is in **Vietnamese** — it is what Vietnamese customers
  query, and must not be translated. Chat with the user is in Vietnamese.

## Workflow

Work runs through four subagents in `.claude/agents/`, matched to complexity:

| Agent | Model | Tools | Role |
|---|---|---|---|
| `planner` | opus | read-only + web | Designs a phase; drafts its build file for the main session to write. Never writes code. |
| `coder` | sonnet | full | Implements an approved build file. Never commits or pushes. |
| `qa` | sonnet | full; writes tests, eval, and build-log entries | Verifies with real command output; records evidence. |
| `reviewer` | opus | read-only + git inspection | Gates the diff before push: `PASS` / `FIX-REQUIRED`. |

Choosing by size of the request:

- **Small** — a typo, a rename, one field: do it directly, no agent.
- **Medium** — one endpoint, one tool: `coder` → `qa`.
- **Large** — a whole phase, a new layer: `planner` → `coder` → `qa`.
- **Before every push**: `reviewer`. No exceptions.

The loop for each phase, via `.claude/commands/`:

1. `/phase-plan NN` — `planner` drafts the build file; the main session writes
   it. No code.
2. **Approval gate** — the user approves scope before any code is written.
3. `/phase-run NN` — `coder` implements, red → green → refactor, inside the
   build file's boundaries.
4. `/verify NN` — `qa` runs the verification and appends evidence to
   `harness/build-log.md`.
5. `/pre-push` — `reviewer` gates the unpushed diff.
6. **Approval gate** — the user approves commit and push, separately, each time.
7. `/retro NN` — improve the framework from evidence; one owner per change.

Rules that hold the loop together:

- **`harness/build-log.md` is the source of truth** for progress, and it is
  append-only. `PLANS.md` and build file statuses follow it, never lead it.
- **A phase is `Complete` only with observed evidence** for every acceptance
  criterion. A plan is not evidence; the existence of code is not evidence.
- **Commits, pushes, and deployments always need the user's approval.** No agent
  does them on its own initiative, and an earlier approval does not carry over.
- **Stay inside the phase.** Work belonging to the next phase is out of scope.
- **Prefer tightening an existing agent or command over adding a new one.**

## Environment

Node 20 is installed. Python 3.12 and Docker Desktop are not yet installed —
phase 01, step 0. The Claude Code CLI is not on `PATH`; the framework runs from
the desktop app through slash commands.
