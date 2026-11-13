# Phase 01 — context

Decisions that constrain this phase, recorded at the approval gate. Not a diary.

## Approval

- 2026-11-13 — the user approved the scope of `harness/build/01-shop-backend-streaming.md`
  as written that day, with the decisions below already applied to it.

## Decisions

- **LLM provider: fake first.** Steps 0–8 and 10–11 are built and verified with
  the fake provider only. Step 9 (the real provider plus `scripts/chat_smoke.py`)
  and acceptance criterion 12 stay blocked until the user picks a provider and
  has a key. Until then the phase cannot be `Complete`.
- **A provider needs its own API key.** A claude.ai subscription does not include
  API usage and may not power a third-party app; a Claude provider would need a
  Console account with API credits.
- **Refunds only for `delivered` orders.** Any other status returns 409. Phase 02's
  policy corpus must state the same rule.
- **No phone or address columns.** Users hold email and name only.
- **Browser check by the main session.** Criterion 13's manual part is done by the
  main session in the desktop app's built-in browser, with the fake provider, before
  it launches `qa`. The observation is passed to `qa` to record.
- **Step 0 is the user's.** The user installs Python 3.12, uv and Docker Desktop
  themselves; these need admin rights and possibly a reboot. `coder` starts at
  step 1 and stops if the tools in Verification section 1 are missing.

## Open

- LLM provider and dev model — blocks step 9 and criterion 12.
