# Phase 04 — Evaluation

**Status:** Not started
**Weeks:** 6

## Purpose

Build the measuring instrument. Every improvement in later phases is only an
improvement if this suite shows it. This is the least exciting phase and the one
that separates this project from a tutorial: the README's before-and-after table
comes from here. **It is not skipped, shortened, or deferred.**

## Inputs

- Phase 03 complete: three switchable retrieval modes.
- A stronger model to act as judge.
- `KE-HOACH-CHI-TIET.md`, section "Tuần 6".

## Steps

1. **Question set.** `BE/eval/questions.yaml`, 50–100 items. Each has the
   question, the expected chunk or heading, the expected answer points, and a
   category: policy, order lookup, refund, out of scope, adversarial. Include the
   hard cases planted in the corpus during phase 02.
2. **Retrieval metrics.** recall@k, precision@k, MRR, and nDCG, computed per
   `RAG_MODE`. Output a markdown table.
3. **LLM as judge.** Score answers for faithfulness to sources, completeness,
   and citation validity, returning structured scores with reasons.
4. **Calibrate the judge.** Hand-score 10 answers; report agreement between the
   judge and the hand scores. A judge that disagrees with you is not evidence.
5. **Baseline.** Record the first full run as the baseline in the build log.
6. **CI.** A workflow running a small subset on each change, failing if recall@5
   drops more than 3 points from the baseline.

## Acceptance criteria

- Question set has 50–100 items covering all five categories. Observed: count
  per category.
- Metrics table produced for all three modes in one run. Observed: the table,
  pasted in the build log.
- Judge agreement with hand scores reported. Observed: the agreement figure and
  the 10 cases.
- Baseline recorded in the build log with date and configuration.
- CI fails on an artificially degraded retrieval. Observed: a run that fails for
  that reason.

## Boundaries

- **Existing questions are frozen once the baseline is recorded.** New questions
  can be added; existing ones are never edited to move a number. Changing the
  instrument invalidates every comparison.
- No retrieval changes in this phase — measure what exists.
- No agent tools — phase 05. Agent evaluation extends this suite then.

## Verification

```
cd BE && python -m eval.run --all-modes
pytest -q tests/eval
ruff check . && mypy app && pytest -q
```

Passing: a metrics table for three modes, a recorded baseline, judge agreement
reported, and a demonstrated CI failure on degradation.
