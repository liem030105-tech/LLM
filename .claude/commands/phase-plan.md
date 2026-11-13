---
description: Plan a roadmap phase with the planner agent (read-only, no code)
argument-hint: <phase number, e.g. 01>
---

Plan phase **$ARGUMENTS** of the Shop Assistant roadmap. This is a read-only step: no
code is written and no file outside `harness/` is touched.

1. Read `PLANS.md` and confirm phase $ARGUMENTS exists and that the phases it depends on
   are `Complete` in `harness/build-log.md`. If a dependency is not complete,
   stop and say which one — do not plan ahead of the sequence.

2. Launch the `planner` agent (Agent tool, `subagent_type: "planner"`) with:
   - the phase number $ARGUMENTS and the path to its build file under `harness/build/`
   - instruction to follow its own Required reading and Output contract
   - any constraint the user stated in this conversation

3. Write the returned body to `harness/build/<NN>-*.md`, keeping
   `Status: Not started`. This is the only file this command may change.

4. Present the **approval gate** to the user, as a short summary:
   - what the phase delivers
   - the files that will be created or changed
   - the files that must **not** be touched (Boundaries)
   - the acceptance criteria, and how each one will be observed
   - the risk flags the planner surfaced (authorization, personal data,
     consequential actions, cost and latency)
   - open questions the user must decide

5. Stop there and wait. Do not create a context file, do not start
   implementing, and do not run `/phase-run`. The user approves first.
