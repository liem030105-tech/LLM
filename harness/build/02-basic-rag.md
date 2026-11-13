# Phase 02 — Basic retrieval with citations

**Status:** Not started
**Weeks:** 3–4

## Purpose

Make the assistant answer policy questions from documents, and prove where each
answer came from. This is the first phase where the model's output must be
grounded; the citation requirement is what makes later evaluation possible.

## Inputs

- Phase 01 complete: database with pgvector, streaming chat endpoint.
- An embedding API.
- `KE-HOACH-CHI-TIET.md`, section "Tuần 3–4".

## Steps

1. **Write the corpus.** 15–20 pages of shop policy **in Vietnamese** under
   `DB/docs/`, one file per topic: returns, warranty, shipping, payment,
   promotions, FAQ. Clear heading structure. Deliberately include hard cases —
   exceptions, and time limits that differ by product category — so later
   evaluation has something to catch.
2. **Chunk table via Alembic.** `doc_chunks` with source file, heading path,
   content, token count, content hash, embedding, and an HNSW cosine index.
   Add a `tsvector` column now, unused until phase 03.
3. **Ingest.** Chunk by heading, 300–500 tokens with 50–80 overlap, keeping the
   heading path for citation. Embed in batches. Upsert by content hash so
   re-ingesting creates no duplicates.
4. **Retrieve and answer.** Top-k (k = 5) by cosine, numbered into the prompt.
   The model must cite, and must answer "not found in the documents" when the
   context does not support an answer. The API returns a `citations` array.
5. **Retrieval mode setting.** Introduce `RAG_MODE`, with `vector` as its only
   value for now. Phase 03 adds the others.

## Acceptance criteria

- Corpus exists: 15–20 pages of Vietnamese markdown, at least six topic files.
  Observed: file listing and word count.
- Re-running ingest leaves the chunk count unchanged. Observed: counts.
- Every citation returned points to a heading that exists in `DB/docs/`.
  Observed: a test resolving each citation against the corpus.
- A question with no answer in the corpus returns the "not found" answer rather
  than an invented policy. Observed: test.
- A hand-checked set of 10 questions answered with correct citations, recorded
  in the build log. Observed: the list with answers.

## Boundaries

- No BM25, reranking, or query rewriting — phase 03.
- No formal evaluation set or metrics — phase 04. The 10 hand-checked questions
  are a smoke test, not the evaluation suite.
- Policy documents stay Vietnamese. Do not translate them.

## Verification

```
cd BE && python -m app.rag.ingest && python -m app.rag.ingest
pytest -q tests/rag
ruff check . && mypy app && pytest -q
```

Passing: chunk count stable across two ingests, citation-resolution and
not-found tests green.
