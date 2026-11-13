---
description: Implement an approved phase with the coder agent
argument-hint: <phase number, e.g. 01>
---

Implement phase **$ARGUMENTS** of the Shop Assistant roadmap.

## Preflight — read-only, before any write

1. Read `harness/build/<NN>-*.md` for phase $ARGUMENTS. If it is missing, or still has
   placeholder content, stop and tell the user to run `/phase-plan $ARGUMENTS` first.
2. Confirm the user has approved this phase's scope in this conversation or in
   its context file. **If there is no approval, stop and ask for it.** Never
   begin writing on the assumption that approval is implied.
3. Confirm the phases this one depends on are `Complete` with evidence in
   `harness/build-log.md`. A plan claiming completion is not evidence.
4. Report current state in two or three lines: what exists, what this phase adds.

## Context file

Only after approval, create `harness/context/phase-<NN>-<slug>-context.md` if it
does not exist. Record only material items — decisions that constrain later work,
constraints discovered on the way, and unresolved questions. Apply a materiality
test: if it would not change a future decision, a verification, or someone's
understanding of why the code looks this way, leave it out. This is not a diary.

## Implementation

Launch the `coder` agent (Agent tool, `subagent_type: "coder"`) with:
- the phase number and the full build file content
- the context file, if one exists
- explicit instruction to stay inside the build file's Boundaries, and to stop
  and report rather than touching anything outside them

The `coder` agent works red → green → refactor per step, writes tests alongside
the code, and does not commit or push.

## After implementation

1. Relay the coder's report: files changed, steps done, steps not done.
2. Do **not** mark the phase `Complete`, and do not write to the build log.
3. Tell the user to run `/verify $ARGUMENTS` next. Evidence comes from `qa`, not from
   the agent that wrote the code.

## Boundaries of this command

- No `git commit`, `git push`, or any publishing action. Those are separate and
  need the user's approval each time.
- No work belonging to the next phase, however small.
- No edits under `BE/eval/` beyond what the build file's Steps name — the rule
  is owned by `.claude/agents/coder.md`.
