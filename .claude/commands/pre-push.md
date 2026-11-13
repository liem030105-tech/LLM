---
description: Gate the unpushed diff with the reviewer agent before anything reaches GitHub
---

Run the pre-push gate. Nothing goes to GitHub without this.

1. Establish what would be published, with read-only git commands:

```
git status --short --branch
git diff
git diff --staged
git diff origin/main...HEAD
git log origin/main..HEAD --oneline
```

   If there is nothing uncommitted and nothing unpushed, say so and stop.

2. Launch the `reviewer` agent (Agent tool, `subagent_type: "reviewer"`) with:
   - the branch state and the diff scope it should review
   - the phase this diff belongs to, plus that phase's build file and build log
     entry, so it can check the evidence requirement
   - instruction to work its five-section checklist and return a verdict

3. Relay the verdict in full:
   - `PASS` or `FIX-REQUIRED` on its own line
   - every finding, worst first, with file, line, problem, risk, and fix
   - what the reviewer could not check, stated plainly

4. Then:
   - **`FIX-REQUIRED`** → do not commit, do not push. Hand the findings to
     `coder` via `/phase-run`, then run `/pre-push` again.
   - **`PASS`** → report it and **ask the user whether to commit and push**.
     Do not commit. Do not push. Do not stage anything on your own initiative.
     `PASS` means nothing blocking was found, not that publishing is approved.

## Rules

- The reviewer is read-only. It must not commit, push, stage, or edit a file. If
  it reports having done any of those, treat that as a defect in the agent and
  tell the user.
- Publishing is always the user's decision, taken after reading the verdict.
  An earlier approval in this conversation does not carry to a later push.
- Never bypass the gate because the change looks trivial. Documentation-only
  diffs are cheap to review — the reviewer returns `PASS` in a line.
