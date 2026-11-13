# Phase 07 — Operations and deployment

**Status:** Not started
**Weeks:** 10

## Purpose

Make the system observable, affordable, and reachable. This phase produces the
third evidence item — cost and latency per request, with the optimizations that
moved them — and the live demo link.

## Inputs

- Phase 06 complete.
- A Langfuse project (public and secret keys).
- A cloud host with managed Postgres that supports pgvector.
- `llm_calls` data recorded since phase 01.
- `KE-HOACH-CHI-TIET.md`, section "Tuần 10".

## Steps

1. **Tracing.** Langfuse traces covering a whole turn: retrieval, each tool call,
   each model call, tokens, cost. Tagged with session and user id. **PII flag:**
   traces go through the masking from phase 06.
2. **Metrics endpoint.** `/admin/metrics`, staff only: p50 and p95 latency, and
   mean cost per turn. **Authorization flag:** a customer must not reach it.
3. **Measure before optimizing.** Record baseline cost and latency in the build log.
4. **Caching.** Embedding cache keyed by query hash; answer cache for repeated FAQ
   questions; provider prompt caching for the long system prompt.
5. **Resilience.** Retry with exponential backoff on rate-limit and server errors.
6. **Rate limiting** per user.
7. **Measure after.** Record cost and latency again; the difference is the result.
8. **Deploy.** Multi-stage Dockerfile; deploy to a cloud host with managed
   Postgres. **Deployment needs the user's approval** — it is an outward action.

## Acceptance criteria

- One full turn appears as a single trace in Langfuse with all its spans.
  Observed: trace id in the build log.
- p50, p95, and mean cost per turn reported before and after optimization.
  Observed: both tables.
- A customer gets 403 on `/admin/metrics`. Observed: test.
- Rate limit returns 429 past the threshold. Observed: test.
- Demo link reachable and completes the refund journey. Observed: link and a
  recorded run.
- Retrieval, tool-choice, and attack results unchanged. Observed: eval run.

## Boundaries

- No optimization without a before-measurement. An unmeasured speedup is a claim.
- Caching must not serve one user's answer to another. Anything user-specific is
  never cached across users.
- No README or video work — phase 08.

## Verification

```
cd BE && pytest -q tests/ops
python -m eval.run --all-modes --tools && python -m eval.attacks
ruff check . && mypy app && pytest -q
docker build -t shop-assistant .
```

Passing: tests green, before-and-after tables recorded, demo reachable, earlier
metrics held.
