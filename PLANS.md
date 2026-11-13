# Plans

The roadmap as a dependency-ordered sequence of phases. Each phase has a build
file under `harness/build/` that defines its scope, and is verified against
evidence recorded in `harness/build-log.md`.

**Status here mirrors the build log, never leads it.** A phase moves to
`Complete` only after `/verify` records evidence for every acceptance criterion.

## Phases

| Phase | Weeks | Build file | Delivers | Status |
|---|---|---|---|---|
| 01 | 1–2 | [01-shop-backend-streaming](harness/build/01-shop-backend-streaming.md) | Simulated shop backend and a streaming chat endpoint | Not started |
| 02 | 3–4 | [02-basic-rag](harness/build/02-basic-rag.md) | Policy answers from the Vietnamese corpus, with citations | Not started |
| 03 | 5 | [03-advanced-retrieval](harness/build/03-advanced-retrieval.md) | Hybrid search, reranking, query rewriting — each toggleable | Not started |
| 04 | 6 | [04-evaluation](harness/build/04-evaluation.md) | Fixed question set, recall@k per retrieval mode, LLM judge, CI | Not started |
| 05 | 7–8 | [05-agent-tool-calling](harness/build/05-agent-tool-calling.md) | Hand-written agent loop with five tools and confirmation | Not started |
| 06 | 9 | [06-safety](harness/build/06-safety.md) | Injection suite, tool-layer authorization, PII masking, handoff | Not started |
| 07 | 10 | [07-operations](harness/build/07-operations.md) | Tracing, cost and latency per request, caching, deploy | Not started |
| 08 | 11 | [08-polish](harness/build/08-polish.md) | README with architecture and evidence tables, demo video | Not started |
| 09 | 12 | [09-interview-prep](harness/build/09-interview-prep.md) | LangGraph port on a branch, system design prep, CV | Not started |

## Dependencies

```
01 ──► 02 ──► 03 ──► 04 ──► 05 ──► 06 ──► 07 ──► 08 ──► 09
                      │
                      └── every later improvement is measured against 04
```

The chain is strict. Two edges matter more than the rest:

- **03 before 04.** Phase 03 must make each retrieval change switchable through
  `RAG_MODE`, otherwise phase 04 has nothing to compare.
- **04 before everything after it.** Without the evaluation suite, no later
  change can show it helped. Phase 04 is never skipped or shortened.

## The loop for each phase

```
/phase-plan NN   planner drafts the build file           no code
   approval      user approves scope                      gate
/phase-run  NN   coder implements, red → green → refactor
/verify     NN   qa runs verification, appends evidence   build log
/pre-push        reviewer gates the diff                  PASS / FIX-REQUIRED
   approval      user approves commit and push            gate
/retro      NN   improve the framework from evidence
```

Commits, pushes, and deployments always need the user's approval, separately,
each time.

## Schedule

Planned for 10–12 hours per week. With 5–6 hours per week, stretch to 20 weeks;
do not cut phases.
