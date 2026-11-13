# Phase 03 — Advanced retrieval

**Status:** Not started
**Weeks:** 5

## Purpose

Improve retrieval with keyword search, reranking, and query rewriting — and make
every one of those improvements switchable, so phase 04 can measure each against
the baseline. A change that cannot be turned off cannot be shown to help.

## Inputs

- Phase 02 complete: vector retrieval with citations, `RAG_MODE=vector`.
- A reranking model: a hosted rerank API, or a local cross-encoder.
- `KE-HOACH-CHI-TIET.md`, section "Tuần 5".

## Steps

1. **Keyword search.** Populate the `tsvector` column; rank with Postgres
   full-text search. Choose and document the text configuration for Vietnamese.
2. **Hybrid.** Fuse vector and keyword results with Reciprocal Rank Fusion.
3. **Rerank.** Retrieve top-20, rerank to top-5 with a cross-encoder.
4. **Query rewriting.** Rewrite follow-up questions into standalone ones using
   conversation history, behind its own switch.
5. **Modes.** `RAG_MODE` accepts `vector`, `hybrid`, and `hybrid_rerank`; query
   rewriting has its own on/off setting. All combinations run.

**Cost and latency flag:** reranking and rewriting each add a call per request.
Record their added latency now; phase 07 needs the numbers.

## Acceptance criteria

- All three `RAG_MODE` values run end to end and return cited answers. Observed:
  a parametrized test across modes.
- Setting `RAG_MODE=vector` reproduces phase 02 behaviour exactly. Observed: test.
- Query rewriting turns a follow-up into a standalone question on a fixed set of
  examples. Observed: test.
- Added latency of reranking and rewriting is recorded in the build log.
  Observed: measurements.

## Boundaries

- **No claims that retrieval improved.** That is phase 04's job, with a fixed
  question set. Here, the deliverable is that each mode runs.
- No changes to the corpus in `DB/docs/` — it is held fixed so modes are
  comparable.
- No agent or tools — phase 05.

## Verification

```
cd BE && pytest -q tests/rag
for m in vector hybrid hybrid_rerank; do RAG_MODE=$m pytest -q tests/rag/test_modes.py; done
ruff check . && mypy app && pytest -q
```

Passing: every mode green, `vector` matches phase 02.
