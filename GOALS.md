# Goals

## Outcome

A deployed customer-support assistant for an online shop that both **answers
policy questions from documents** (retrieval with citations) and **performs real
actions on the shop system** (order lookup, refund requests, handoff to a human)
— with measured quality, measured cost, and a defensible security model.

The project exists to be shown and defended in interviews for AI Engineer and
LLM Application Engineer roles. That makes **evidence the deliverable**, not the
code. Three things carry the weight:

1. An evaluation table in the README showing retrieval quality before and after
   each improvement.
2. Authorization enforced in the tool layer, demonstrably not bypassable by
   prompt injection.
3. Cost and latency per request, with the optimizations that moved them.

## Success conditions

The project is done when all of these are observably true:

- [ ] A working demo link, and a two-minute demo video.
- [ ] `README.md` contains an architecture diagram, the decisions and trade-offs
      taken, and the remaining limitations.
- [ ] An evaluation table showing recall@k before and after each retrieval
      improvement, over a fixed question set of 50–100 items.
- [ ] A prompt-injection and authorization test suite, with its results table.
- [ ] Cost and latency per request, measured, with p50 and p95.
- [ ] Every phase in `PLANS.md` marked `Complete` with evidence in
      `harness/build-log.md`.
- [ ] CV updated with the project's numbers, and applications sent.

A claim without observed command output behind it does not count toward any of
these.

## Non-goals

Deliberately out of scope. Adding any of them is scope creep, not ambition:

- **A real shop.** The shop backend is a simulation: generated users, products,
  and orders. No payment integration, no real inventory, no real customers.
- **Real customer data.** The policy corpus and the order data are written and
  generated for this project. Personal data never enters the system.
- **A general-purpose agent.** The assistant does five things. It does not browse
  the web, write code, or hold open-ended conversations.
- **An agent framework.** The agent loop is written by hand through phase 05.
  LangChain and LangGraph are studied in phase 09, on a separate branch, to be
  able to say what a framework does underneath — not to build on.
- **Multi-tenancy, scale, or high availability.** One shop, one deployment.
  Retrieval is designed for thousands of chunks, not millions.
- **A production-grade frontend.** One chat page with streaming. No design
  system, no mobile app, no account management UI.
- **Model training or fine-tuning.** Hosted LLM APIs only.

## Constraints

- **Time:** roughly 10–12 hours per week, over 12 weeks. If the available time
  drops to 5–6 hours, stretch the schedule to 20 weeks rather than cutting
  content — the phases depend on each other.
- **Budget:** a few dollars of API spend total. Use small models during
  development; reserve the stronger model for the evaluation judge.
- **Language:** the policy corpus in `DB/docs/` is Vietnamese, because the
  assistant serves Vietnamese customers and retrieval quality in Vietnamese is
  part of what the project demonstrates. Code, prompts, and framework files are
  English.

## The rule that protects all of this

Phase 04 is the evaluation phase, and it is the least interesting one to build.
Every improvement in phases 05 through 08 depends on it to prove it was an
improvement. **It does not get skipped, shortened, or deferred.**
