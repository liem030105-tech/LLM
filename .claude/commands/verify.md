---
description: Verify a phase with the qa agent and record evidence in the build log
argument-hint: <phase number, e.g. 01>
---

Verify phase **$ARGUMENTS** of the Shop Assistant roadmap and record the evidence.

1. Read `harness/build/<NN>-*.md` for phase $ARGUMENTS — its Acceptance criteria and
   Verification section are what you are checking against.

2. Launch the `qa` agent (Agent tool, `subagent_type: "qa"`) with:
   - the phase number and the full build file content
   - whatever `coder` reported, framed as **claims to test**, not facts
   - instruction to paste real command output and to mark anything it could not
     run as `unverified`

3. The `qa` agent appends its entry to `harness/build-log.md` and sets the
   phase status there. Confirm it did, and that it did not rewrite any earlier
   entry — the log is append-only.

4. Update `Status` in the build file and the row in `PLANS.md` to match what the
   build log now says. `Complete` requires every acceptance criterion to have a
   checked box with evidence behind it. Anything less is `In progress` or
   `Blocked`.

5. Report to the user:
   - the verdict per acceptance criterion
   - findings for `coder` to fix, worst first
   - **which criteria are unverified and why** — state this plainly rather than
     letting a clean-looking summary imply full coverage

## Rules

- A plan is not evidence. The existence of code is not evidence. Only observed
  command output is.
- Do not let `qa` fix product code. Findings in `BE/app/` go back to `coder`.
- Do not let `qa` change `BE/eval/` questions to improve a number. Adding new
  questions is fine; altering existing ones to move the measurement is not.
- If the phase cannot be verified because the environment is incomplete (no
  database, no API key, no dependencies), say exactly that and leave the status
  as `Blocked`. Do not approximate.
