# Phase 05 — Agent with tool calling

**Status:** Not started
**Weeks:** 7–8

## Purpose

Turn the assistant from a question-answerer into an agent that acts on the shop.
The loop is written by hand — no LangChain, no LangGraph — so the mechanism is
understood well enough to explain in an interview and to debug when it breaks.

## Inputs

- Phase 04 complete: evaluation suite and baseline.
- Phase 01's shop API and its tool-layer authorization.
- `KE-HOACH-CHI-TIET.md`, section "Tuần 7–8".

## Steps

1. **Five tools**, each with a JSON schema stating when to use it and when not to:
   - `search_policy` — retrieval, read-only
   - `get_order` — one order by id
   - `list_my_orders` — recent orders
   - `create_refund_request` — consequential
   - `handoff_to_human` — consequential
2. **Authorization flag.** Tools that touch orders take the user id from the
   session, never from tool arguments. The model cannot name whose order to read.
3. **Loop.** Call the model with tools; execute tool calls; feed results back;
   stop on a text answer. Cap steps at 6, timeout each tool, log every step.
4. **Errors.** Tools never raise. They return a structured error with a hint so
   the model can recover and ask the user.
5. **Confirmation.** Consequential tools run as dry-run first, return a preview,
   and execute only on explicit confirmation from the chat page, using a
   short-lived single-use confirmation token.
6. **Multi-turn.** Persist conversations and messages. When history grows, keep
   recent turns plus a summary of older ones; measure the token saving.
7. **Agent evaluation.** Extend the phase 04 suite with tool-choice cases: for a
   given request, which tool should be called.

## Acceptance criteria

- The refund journey works end to end: lookup → policy check → confirmation
  prompt → request created. Observed: an integration test.
- A refund is never created without confirmation, and one confirmation cannot
  create two refunds. Observed: tests.
- `get_order` for another user's order fails at the tool layer, **with the
  system prompt removed entirely**. Observed: test.
- A tool failure produces a recovery, not a crash. Observed: test with a forced
  database error.
- Tool-choice accuracy reported on the extended suite. Observed: the figure.
- Retrieval metrics unchanged from the baseline. Observed: eval run.

## Boundaries

- No agent framework. A framework import in this phase is a review finding.
- No prompt-injection suite yet — phase 06. Authorization here is structural;
  attacks on it are tested next phase.
- Existing eval questions stay frozen; tool-choice cases are added, not edited in.

## Verification

```
cd BE && pytest -q tests/agent
python -m eval.run --all-modes --tools
ruff check . && mypy app && pytest -q
```

Passing: refund journey, confirmation, cross-user, and tool-failure tests green;
tool-choice accuracy reported; retrieval at or above baseline.
