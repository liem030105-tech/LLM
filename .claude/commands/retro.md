---
description: Run an evidence-based retrospective on a finished phase and improve the framework
argument-hint: <phase number, e.g. 01>
---

Run a retrospective on phase **$ARGUMENTS**. This step is read-only until the user
approves the improvements. Its purpose is to make the next phase cheaper, not to
narrate the last one.

## 1. Gather evidence

Read, and base everything on these rather than on recollection:

- `harness/build/<NN>-*.md` — what the phase promised
- `harness/build-log.md` — what actually happened, including failed attempts
- `harness/context/phase-<NN>-*-context.md` — decisions made along the way
- `git log` for the phase's commits, and the reviewer verdicts it went through

## 2. Separate what worked from what caused friction

For each item, name the evidence. Specifically look for:

- Acceptance criteria that turned out not to be observable, and had to be
  reinterpreted during verification.
- Findings `reviewer` caught that `coder` or `planner` should have prevented.
- The same mistake appearing more than once.
- Scope that leaked across a phase boundary.
- Steps that were larger than one verifiable unit.

## 3. Assign every improvement exactly one owner

An improvement with no owner file does not exist. Pick one:

| Owner | Use when |
|---|---|
| `CLAUDE.md` | A convention every agent and session must follow |
| `.claude/agents/<name>.md` | One role kept making the same mistake |
| `.claude/commands/<name>.md` | The procedure itself was wrong or ambiguous |
| `harness/build/<NN>-*.md` | A later phase's plan needs correcting now |
| A regression test in `BE/tests/` | The failure is reproducible |

Rules:

- **Prefer tightening an existing agent or command over adding a new one.** A
  fifth agent is a last resort, justified only by work the existing four cannot
  express.
- Convert reproducible failures into regression tests rather than into prose
  warnings. A test holds; a warning erodes.
- One owner per improvement. If two files seem to own it, the instruction is
  not sharp enough yet — sharpen it.

## 4. Propose, then wait

Present the retrospective to the user:

- what worked, with evidence
- what caused friction, with evidence
- the proposed changes, each with its single owner file and the exact edit

Then stop. Apply nothing until the user approves. After applying, append a short
entry to `harness/build-log.md` recording which framework files changed and why,
so the next retrospective can tell whether the change helped.

## Rules

- No blame, and no praise either — both are noise. Report causes.
- Do not rewrite history in the build log. Failed attempts stay; they are the
  most useful part of it.
- Do not declare the framework improved on the strength of the edit alone.
  The next phase is the test.
