# Phase 09 — Interview preparation

**Status:** Not started
**Weeks:** 12

## Purpose

Convert the project into interview performance. Port the agent to LangGraph on a
separate branch, to be able to say what a framework does underneath after having
built the loop by hand. Prepare system design answers grounded in the project's
own numbers.

## Inputs

- Phase 08 complete: README with all evidence tables.

## Steps

1. **LangGraph port** on a branch named `langgraph-port`. Same five tools, same
   authorization, same confirmation flow. Run the phase 04–06 suites against it.
2. **Comparison note.** What the framework gave, what it hid, what it cost — with
   numbers from the suites, not impressions.
3. **System design practice.** Prepared answers for: retrieval over a million
   documents; evaluation strategy; cost control; injection defence; prompt
   versioning and rollback. Each answer anchored to a decision taken in this
   project.
4. **CV.** Position as a backend engineer strong in AI. The project line carries
   the measured numbers: recall@5, tool-choice accuracy, cost per turn, p95.
5. **Applications.** 10–15, searched under AI Engineer, GenAI, LLM, RAG, and
   Backend (AI).

## Acceptance criteria

- The LangGraph branch passes the same eval and attack suites as `main`, or the
  differences are recorded. Observed: suite output on the branch.
- The comparison note exists with numbers from both implementations.
- Every number on the CV traces to a build log entry.

## Boundaries

- **The LangGraph port never merges into `main`.** The hand-written loop is the
  project's point.
- No new features on `main`.
- Sending applications is the user's action, not an agent's.

## Verification

```
git checkout langgraph-port
cd BE && pytest -q && python -m eval.run --all-modes --tools && python -m eval.attacks
```

Passing: suites run on the branch, results recorded beside `main`'s.
