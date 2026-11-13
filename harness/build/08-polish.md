# Phase 08 — Polish

**Status:** Not started
**Weeks:** 11

## Purpose

Turn the working system into something a reviewer understands in two minutes.
The evidence already exists in the build log; this phase puts it where an
interviewer will look.

## Inputs

- Phases 01–07 complete, with their evidence in `harness/build-log.md`.

## Steps

1. **README.md** with:
   - the problem, in three sentences
   - an architecture diagram (Mermaid)
   - how to run it locally
   - **the evaluation table**: recall@k for each retrieval mode, from phase 04 onward
   - **the attack results table**, from phase 06
   - **cost and latency**, before and after, from phase 07
   - decisions and trade-offs, one line each with the alternative rejected
   - remaining limitations, stated honestly
2. **Demo video**, two minutes: a policy question with citation → order lookup →
   refund with confirmation → an injection attempt that fails → the trace in
   Langfuse.
3. **Technical article**, one: for example how hybrid search and reranking moved
   recall@5 on a Vietnamese corpus, with the real numbers.
4. **Repo hygiene.** Tests green, `ruff` and `mypy` clean, `.env.example`
   complete, dependencies pinned.

## Acceptance criteria

- Every number in the README traces to a build log entry. Observed: a
  cross-check listing each number and its entry.
- Following the README on a clean checkout brings the system up. Observed: a run
  from a fresh clone.
- Video and article exist and are linked from the README.
- `ruff check`, `mypy`, `pytest` clean. Observed: output.

## Boundaries

- **No new numbers.** The README reports measured results only. If a number is
  missing, go back to the phase that should have produced it; do not estimate.
- No new features. Polish surfaces what exists.
- Publishing the article and video is the user's action, not an agent's.

## Verification

```
git clone <repo> /tmp/fresh && cd /tmp/fresh   # follow README from here
cd BE && ruff check . && mypy app && pytest -q
```

Passing: fresh-clone setup works by the README alone; every README number maps
to evidence.
