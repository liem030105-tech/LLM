# Phase 01 — Shop backend and streaming chat

**Status:** Not started
**Weeks:** 1–2

## Purpose

This phase builds the simulated shop that every later phase acts on: the schema, deterministic fake data, and an order and refund API whose authorization is enforced in code from day one. It also builds a chat endpoint that streams model output to a browser page. It comes first because retrieval (02) needs the database and the streaming endpoint to land on, the agent (05) wraps this phase's order and refund functions as tools, and operations (07) measures the `llm_calls` record started here. Nothing intelligent happens yet: no retrieval and no tools.

## Inputs

- **Completed phases:** none. `harness/build-log.md` holds only the framework-setup entry (2026-11-13). No `BE/`, `DB/`, `FE/` or `DB/docs/` exists. Decisions taken at the approval gate are in `harness/context/phase-01-shop-backend-streaming-context.md`.
- **Source material:** `KE-HOACH-CHI-TIET.md`, sections "0. Chuẩn bị môi trường" and "Tuần 1–2" (schema, seed sizes, the six order statuses, the done-condition).
- **Project rules:** the hard rules and conventions in `CLAUDE.md`. The existing root `.gitignore` already covers `.env`, `.venv/`, tool caches and `pgdata/`. Reuse it and do not rewrite it.
- **Machine:** Windows 11 with Node 20 installed. Python 3.12, uv and Docker Desktop (WSL2 backend) get installed in step 0. Verification has to run in Windows PowerShell 5.1, which has no `&&`, as well as PowerShell 7 and Git Bash.
- **Credentials:** an API key for the chosen LLM provider, needed only for step 9 and acceptance criterion 12. Every automated test runs without a key. No Langfuse key is needed (that is phase 07).
- **External facts this plan relies on** (checked 2026-11-13; coder re-checks when pinning):
  - FastAPI 0.135 and later has built-in SSE (`fastapi.sse`, `EventSourceResponse`). It works with POST, sends keep-alive pings, and sets `Cache-Control: no-cache` and `X-Accel-Buffering: no`.
  - httpx `ASGITransport` buffers the whole response body, so it cannot prove streaming.
  - The official `pgvector/pgvector` image publishes version-pinned pg16 tags, for example `0.8.x-pg16`.
- **Open decision:** the LLM provider and dev model. This blocks step 9 only.

## Steps

Every step ends with what proves it. Tests are written with the step (red, green, refactor).

Every Python command goes through `uv run --directory BE`, run from the repo root. That way it works in all three Windows shells and in agent shells whose working directory resets between calls.

0. **Environment.**
   - Install Python 3.12 (winget id `Python.Python.3.12`, added to PATH), uv (`astral-sh.uv`) and Docker Desktop (`Docker.DockerDesktop`, WSL2 backend). If WSL is missing, run `wsl --install` and reboot.
   - Set the user environment variable `PYTHONUTF8=1`. This keeps Vietnamese text from going through Windows' default cp1252 encoding, which matters from phase 02 onward.
   - Do not get an LLM key until the provider is decided.
   - Proven by: the version commands in Verification section 1.

1. **Backend project and settings.**
   - Create `BE/pyproject.toml` with Python pinned to 3.12.
     - Runtime dependencies: FastAPI 0.135 or later, uvicorn, SQLAlchemy 2.x (async), asyncpg, Alembic, pydantic-settings, Faker.
     - Dev dependencies: pytest, pytest-asyncio, httpx, ruff, mypy.
     - Include a build backend so that `uv sync` installs `app` into the venv in editable mode. The seed script and the Alembic environment import `app` from outside `BE/`.
   - Create `BE/uv.lock` and `BE/.python-version`.
   - Configure in `pyproject.toml`:
     - ruff, for both lint and format;
     - mypy in strict mode for `app/`;
     - pytest in asyncio mode, with `pythonpath` including `../DB/seed`.
   - Create `BE/app/__init__.py` and `BE/app/config.py`.
     - `config.py` holds one pydantic-settings class that reads the single repo-root `.env`.
     - It finds `.env` relative to the module file, not the working directory.
     - API keys are typed as secret strings so they never print.
   - Create a root `.env.example` with placeholders for every setting this phase reads. At minimum:
     - Postgres user, password, database and host port (defaults `shop`, a placeholder, `shop`, `5432`);
     - `DATABASE_URL`, and `TEST_DATABASE_URL` pointing at database `shop_test`;
     - `LLM_PROVIDER`, defaulting to `fake` so the stack runs with zero spend;
     - `LLM_MODEL` and the provider's key variable;
     - `LLM_MAX_OUTPUT_TOKENS`, `LLM_TIMEOUT_SECONDS`, `CHAT_MAX_INPUT_CHARS`, `FAKE_LLM_CHUNK_DELAY_MS`, `LOG_LEVEL`, and the FE directory path.
   - Proven by: `BE/tests/test_settings.py` checks that every settings field appears in `.env.example` and that every key in `.env.example` is a settings field.

2. **Database container.**
   - Create a root `docker-compose.yml` with a single `db` service:
     - the `pgvector/pgvector` image pinned to a pgvector version tag for pg16 (look up the current tag on Docker Hub; do not use the floating `pg16`);
     - credentials and host port from `.env`;
     - a named volume;
     - a `pg_isready` healthcheck, so `docker compose up -d --wait db` blocks until the database is ready.
   - No `api`, Langfuse or any other service.
   - Proven by: `docker compose up -d --wait db` exits 0.

3. **Schema: DB layer, models, Alembic, test database.**
   - **`BE/app/db.py`:**
     - an async engine on asyncpg, with `echo` off and bound parameters hidden from logs and exception messages (SQLAlchemy's `hide_parameters` option);
     - a session factory;
     - a request-scoped session dependency.
   - **`BE/app/models/`:** a declarative base, plus one module per entity using typed SQLAlchemy 2.0 declarative models.
     - `users`: email (unique), name, role (`customer` or `staff`), created_at. There are no phone or address columns; this is data minimization.
     - `products`: sku (unique), name (Vietnamese), category, price in integer VND, warranty_months.
       - Categories are fixed English slugs: `apparel`, `footwear`, `electronics`, `home`, `beauty`.
       - Phase 02's policies will differ by category.
     - `orders`:
       - code: unique and short. It is the natural key for seeding and what a customer would quote.
       - user_id, created_at, shipped_at, delivered_at, tracking_code.
       - status: `pending`, `paid`, `shipping`, `delivered`, `cancelled` or `refunded`.
       - total in integer VND.
       - An index on (user_id, created_at).
     - `order_items`: order_id, product_id, qty, unit price in VND; unique on (order_id, product_id).
     - `refund_requests`: order_id, reason, status (`pending`, `approved` or `rejected`), created_at.
       - A partial unique index on order_id, limited to rows whose status is `pending` or `approved`.
       - This lets the database guarantee at most one active request per order.
     - `handoffs`: user_id, conversation_id, reason, created_at.
       - conversation_id is nullable and has no foreign key until phase 05 creates conversations.
       - Table only; there is no endpoint.
     - `llm_calls`:
       - created_at, request_id, and user_id (nullable).
       - purpose: `chat` for now; later phases add values such as `embedding` and `judge`.
       - provider and model.
       - prompt_tokens, completion_tokens, latency_ms, ttft_ms.
       - cost_usd: numeric, at least 8 decimal places.
       - status (`ok`, `error` or `cancelled`) and error_type.
       - There are no columns for prompt or completion text.
     - Enumerated values are text columns with CHECK constraints, not native Postgres enums.
     - All timestamps are timezone-aware UTC.
   - **`BE/alembic.ini`:**
     - `script_location` points at `DB/migrations`, resolved from the ini file's own directory, not the working directory;
     - there is no database URL in this file.
   - **`DB/migrations/env.py`:** start from Alembic's async template.
     - Take the URL from settings, unless the caller overrides it on the Alembic config (tests do).
     - Take `target_metadata` from `app.models`.
   - **`DB/migrations/versions/0001_initial_schema.py`:**
     - creates the `vector` extension if it does not exist, then every table, constraint and index above;
     - its downgrade drops them in reverse order, then the extension.
     - No `create_all` anywhere, including tests.
   - **`BE/tests/conftest.py`:**
     - A synchronous, session-scoped fixture drops and recreates `shop_test`, then runs `upgrade head` programmatically. It must be synchronous because the async env calls `asyncio.run`, which cannot run inside a running event loop. As a result, every pytest run proves the migrations on an empty database.
     - Clean up between tests by truncating the app tables. Rollback isolation cannot see step 8's real-server writes.
     - Use a non-pooling engine in tests, so asyncpg connections do not cross event loops.
   - **Personal data flag:** `hide_parameters` is what keeps seeded emails out of SQL error logs. Reviewer must check that it is set.
   - Proven by: `upgrade head`, then `downgrade base`, then `upgrade head`, all exiting 0; then `alembic check` reports no drift.

4. **Deterministic seed**, in `DB/seed/`.
   - **`catalog.py`:** hand-written Vietnamese product-name parts for each category. Faker does not produce Vietnamese product names.
   - **`generate.py`:** a pure function with no database access or I/O.
     - It uses a fixed RNG seed and Faker `vi_VN` for names.
     - It computes every date from a fixed reference-date constant, never from the current time.
     - It produces:
       - about 200 users, of which a small fixed number are `staff`;
       - about 80 products;
       - 300 to 500 orders covering all six statuses, each with 1 to 4 items.
     - Each order's total equals the sum of its items.
     - Timestamps and tracking codes match the status: delivered_at is set only for delivered and refunded orders, and shipped_at is never set on orders that have not shipped.
     - Every `refunded` order gets an `approved` refund request, and a few delivered orders get a `pending` one.
     - Emails are synthetic, ASCII, and only on reserved example domains (RFC 2606).
   - **`seed.py`:** the CLI.
     - It reads `DATABASE_URL` through `app.config`, and refuses with a clear message if the schema is not at head.
     - It inserts everything in one transaction, using insert-or-ignore on the natural keys: email, sku, order code, (order_id, product_id), and the active refund per order.
     - It prints only per-table row counts, per-status order counts, and a checksum of the table contents ordered by natural key.
     - It never prints rows, names or emails.
   - **Personal data flag:** Faker output must never reach stdout, logs or test snapshots.
   - **Forward note for phase 05:** because the dates are pinned, refund-window checks will need a clock that can be pinned to the seed's reference date.
   - Proven by:
     - `BE/tests/test_seed_generation.py`, which checks that two generations are identical, every invariant above, that every email is on a reserved domain, and that user records have no phone or address field;
     - two CLI runs that print identical output.

5. **Identity and the shop domain layer.**
   - **`BE/app/auth.py`:** a dependency that reads `X-User-Id`, requires a positive integer, and loads that user.
     - If the header is missing, malformed or unknown, it returns 401 with a stable error code.
     - This is the "session" every later phase reuses.
   - **`BE/app/shop/orders.py`, `BE/app/shop/refunds.py`, `BE/app/shop/products.py`:** the only modules that query shop tables.
     - Every order and refund function takes the session `user_id` as a required keyword argument and puts it in the WHERE clause. Order items are filtered through their owning order.
     - Routes never query the tables directly.
     - Phase 05's tools will call these same functions.
   - There is no staff bypass in this phase: a staff user sees only their own orders.
   - **Authorization flag:** reviewer must read this layer line by line. An order that belongs to someone else must be indistinguishable from an order that does not exist.
   - Proven by: `BE/tests/shop/test_domain_authz.py`, which calls the domain functions directly, below HTTP. Looking up another user's order returns "not found", and creating a refund on another user's order writes no row.

6. **Shop API.**
   - **`BE/app/schemas/shop.py`:** response models.
     - Orders include code, status, totals, timestamps, tracking code, and items (sku, name, qty, unit price).
     - No response contains a user's email or name.
   - **`BE/app/routers/shop.py`:**
     - `GET /orders/{id}`: the owner gets 200. Anyone else gets 404, with a body identical to the one for an id that does not exist.
     - `GET /users/me/orders`: only the session user's orders, newest first, with `limit` defaulting to 20 and capped at 100. There is no `/users/{id}/orders`, so there is no user-id parameter to tamper with.
     - `GET /products/{sku}`: the public catalog, with no user scoping; 404 for an unknown sku.
     - `POST /refunds`: the body is an order id and a reason of 1 to 1000 characters.
       - Only the owner can create one. Anyone else gets 404, and no row is created.
       - The order must be `delivered`, otherwise 409 with a stable error code.
       - A second active request for the same order returns 409 and creates nothing. The partial unique index guarantees this, not a read-then-write check.
       - Success returns 201 with the pending request.
       - The reason text is never logged.
   - **`BE/app/main.py`:** the app factory, which mounts the routers.
   - **Consequential-action flag:**
     - In this phase, `POST /refunds` is a direct REST action by the identified user, not something the model initiates. Its replay guard is the database constraint.
     - The dry-run, confirm, execute flow with a single-use token becomes required in phase 05, when the agent gets a refund tool.
     - That tool must wrap `app/shop/refunds.py`. It must never call this endpoint unguarded.
     - Reviewer should confirm that this split is stated and that the constraint exists in migration 0001.
   - Proven by:
     - `BE/tests/shop/test_orders_authz.py`, using two customers and one staff user: the owner gets 200; the other customer gets 404; staff gets 404; an id that does not exist gets 404 with an identical body; `/users/me/orders` contains only the caller's own orders; a missing, non-integer or unknown header gets 401;
     - `BE/tests/shop/test_refunds.py`: the 201 path; a cross-user request gets 404 and writes no row; an ineligible status gets 409; a duplicate gets 409 and exactly one active row remains; an oversize reason gets 422;
     - `BE/tests/shop/test_products.py`.

7. **LLM interface, fake provider, cost record.**
   - **`BE/app/llm/base.py`:** a provider interface that streams text deltas, then yields final token usage taken from the provider's own stream.
     - Usage is never estimated.
     - Each provider reports streaming usage differently, and some only when explicitly asked.
   - **`BE/app/llm/fake.py`:** a deterministic provider for tests and for keyless development. It has:
     - configurable chunks;
     - a per-chunk delay (`FAKE_LLM_CHUNK_DELAY_MS`);
     - fixed usage numbers;
     - a record of the messages it received, for the personal-data test;
     - an injectable failure after N chunks.
   - **`BE/app/llm/factory.py`:** picks the provider from `LLM_PROVIDER`. At startup it fails fast, without printing the key, if the key is missing or the model has no price entry.
   - **`BE/app/obs/pricing.py`:** for each model, the input and output price in USD per million tokens, with the vendor URL and an as-of date.
     - Cost is computed with exact decimal arithmetic.
     - The fake model costs 0.
   - **`BE/app/obs/llm_calls.py`:** writes one `llm_calls` row using its own short-lived session, not the request-scoped one. How long a request-scoped dependency lives around a streaming response depends on the FastAPI version.
   - **Cost flag:** the pricing table is the source of every cost number in the README. It needs a dated vendor source for each entry.
   - Proven by: `BE/tests/chat/test_llm_calls.py`. Cost is exact for a known usage, and an unknown model is rejected at startup.

8. **Streaming chat endpoint.**
   - **`BE/app/schemas/chat.py` and `BE/app/routers/chat.py`:** `POST /chat`.
     - It requires the identity from step 5.
     - The body is a single message of 1 to `CHAT_MAX_INPUT_CHARS` characters.
   - The response uses FastAPI's built-in `EventSourceResponse`, with three event types:
     - `delta`: a piece of text;
     - `done`: the llm_call id and the usage;
     - `error`: a stable code and a generic message. It never contains the provider's exception text or a stack trace.
   - It is single-turn. The prompt is a short English system prompt plus the user's message. It contains no shop data, no user name or email, and no history supplied by the client.
   - Time to first token and total latency are measured on a monotonic clock.
   - The `llm_calls` row is written before `done` is emitted, with purpose `chat`, the session user id, and status `ok`.
   - A provider failure or timeout sends an `error` event and writes a row with status `error`.
   - When the client disconnects:
     - the provider stream is closed, which stops generation and billing;
     - a row with status `cancelled` is still written, so that write must survive task cancellation.
   - Never log the message text or the completion.
   - No GZip middleware, because it buffers streams.
   - **Cost and latency flag:** each `/chat` request makes one LLM call, bounded by `CHAT_MAX_INPUT_CHARS` and `LLM_MAX_OUTPUT_TOKENS`. Automated tests make zero paid calls.
   - **Personal data flag:** the prompt holds only the system prompt and the user's own message, and nothing from the user's message reaches logs or the database.
   - **Authorization flag:** the identity comes from the session dependency, never from the body.
   - Proven by:
     - **`BE/tests/chat/test_chat_stream.py`**, run against a real uvicorn server that a fixture starts on a free port, with the fake provider:
       - the content type is `text/event-stream`;
       - at least 3 `delta` events arrive, followed by exactly one `done`;
       - the gap between the first and last `delta` is at least 80% of (number of deltas − 1) × the configured delay, which shows delivery was incremental;
       - disconnecting after the first `delta` produces a `cancelled` row within 5 seconds.
     - **`BE/tests/chat/test_llm_calls.py`**, extended:
       - each call writes one row with non-null tokens, latency, ttft and cost;
       - the row's user_id equals the session user;
       - an injected failure produces an `error` event and an `error` row;
       - a request without identity gets 401;
       - an oversize message gets 422 and no provider call is made.
     - **`BE/tests/test_pii_logging.py`**, with every logger captured at DEBUG while exercising every endpoint and `/chat` with fixture users:
       - their emails never appear in the logs;
       - a marker string placed in a chat message never appears in the logs;
       - the fake provider's received messages contain neither the user's email nor name;
       - the engine hides bound parameters.

9. **Real provider and live smoke (blocked until the provider is decided).**
   - **`BE/app/llm/<provider>.py`,** built on the chosen provider's official SDK.
     - It streams deltas, takes usage from the stream, and respects the output-token cap and the timeout.
     - No retries; that is phase 07.
     - Add its price entry to `pricing.py` from the vendor's pricing page, with the date.
   - **`BE/scripts/chat_smoke.py`:**
     - starts the app in-process on a free port, against the dev database, with the configured provider;
     - sends one short Vietnamese question as the lowest-id customer, or as the user given by `--user-id`;
     - prints each event's arrival offset in milliseconds, and the resulting `llm_calls` row (token counts, latency, cost, status). It never prints content or email.
     - It exits non-zero if fewer than two `delta` events arrive or the row has null tokens.
   - Any live pytest tests carry a marker that is skipped by default.
   - **Cost flag:** one paid call per run. Expected spend is well under one US cent with a small model and the output cap.
   - Proven by: one run, with its output pasted into the build log.

10. **Chat page.**
    - **Files:** `FE/index.html`, `FE/chat.js`, `FE/sse.js`, `FE/sse.test.mjs`. Vanilla JavaScript, no build step, no npm dependencies.
    - **`sse.js`:** a pure module that turns arbitrary network chunks into SSE events. It handles frames split across chunks, several frames in one chunk, and ignores comment and ping lines.
    - **`chat.js`:**
      - posts with `fetch` and reads the response stream (`EventSource` cannot POST or send headers);
      - sends `X-User-Id` from a user-id field on the page, which has no default;
      - appends `delta` text with `textContent`, never `innerHTML`, because model output is untrusted;
      - shows `error` events plainly.
    - **`BE/app/main.py`:** mounts `FE/` as static files at `/`, after the API routes, with the path taken from a setting so phase 07's image can relocate it. Page and API share an origin, so no CORS middleware is needed.
    - Proven by:
      - `node --test FE/sse.test.mjs`;
      - `BE/tests/chat/test_chat_page.py`, which checks that `GET /` returns 200 `text/html` and references `chat.js`;
      - the week's done-condition, observed by the main session in the desktop app's built-in browser (criterion 13).

11. **README note.**
    - A minimal root `README.md` covering:
      - how to run locally on Windows (the commands from Verification);
      - a "Limitations" line saying `X-User-Id` is a placeholder identity, not authentication, while the authorization filtering behind it is real.
    - The full README is phase 08.
    - Proven by: `Select-String`.

## Acceptance criteria

1. **Environment.** `uv run --directory BE python --version` prints `Python 3.12.x`, and `uv --version`, `docker --version` and `docker compose version` all exit 0. Observed: their output.
2. **Database is up and pgvector is enabled.** `docker compose up -d --wait db` exits 0. After migration, the psql query in Verification section 3 returns exactly one row for `vector`. Observed: output.
3. **Migrations.** `upgrade head`, then `downgrade base`, then `upgrade head` all exit 0. `alembic check` prints "No new upgrade operations detected." `git grep --untracked -n create_all -- BE DB` prints nothing. Observed: output.
4. **Seed is idempotent.** Two consecutive runs print identical per-table counts and an identical checksum. The counts fall in range:
   - users between 180 and 220, with at least one staff;
   - products between 70 and 90;
   - orders between 300 and 500;
   - every one of the six statuses non-zero.

   Observed: both outputs pasted side by side.
5. **Seed leaks no personal data.** Piping the seed output through `Select-String '@'` prints nothing, and `tests/test_seed_generation.py` passes. Observed: output.
6. **Cross-user authorization.** Every test in `tests/shop/test_domain_authz.py` and `tests/shop/test_orders_authz.py` passes, including other-customer 404, staff 404, identical 404 bodies, and 401 for bad identity. Observed: the `pytest -v` lines listing each test by name.
7. **Refund rules.** `tests/shop/test_refunds.py` passes, including duplicate 409 with exactly one active row, and cross-user 404 with no row created. Observed: `pytest -v` lines.
8. **Streaming is incremental.** `tests/chat/test_chat_stream.py` passes against the real server: at least 3 deltas, then one `done`, with the timing-gap assertion holding. Observed: `pytest -v` lines.
9. **Every call is recorded.** `tests/chat/test_llm_calls.py` passes:
   - `ok` rows have non-null tokens, latency, ttft and cost, and the session user's id;
   - injected failures produce `error` rows;
   - disconnects produce `cancelled` rows.

   Observed: `pytest -v` lines.
10. **No personal data in logs or prompts.** `tests/test_pii_logging.py` passes. Observed: `pytest -v` lines.
11. **Settings are mirrored and secrets are ignored.** `tests/test_settings.py` passes, and `git check-ignore .env` prints `.env`. Observed: output.
12. **A real provider streams.** `scripts/chat_smoke.py` exits 0. It prints at least two `delta` events with increasing offsets, the first-delta offset is below the `done` offset, and the row shows non-null tokens and `cost_usd` above 0. Observed: output pasted to the build log. Until the provider is decided this stays unverified, so the phase cannot be `Complete` without it.
13. **Chat page.** `node --test FE/sse.test.mjs` passes and `tests/chat/test_chat_page.py` passes. The main session also opens `http://localhost:8000/` in the desktop app's built-in browser, sends one Vietnamese question, and reads the page at least twice during one response, showing the answer growing between reads. The fake provider with `FAKE_LLM_CHUNK_DELAY_MS` set is enough for this; it does not need the real provider. Observed: test output, plus the main session's dated observation, which it passes to qa to record in the build log. qa cannot open a browser itself; until that observation is recorded, this criterion is unverified.
14. **Quality gate.** `ruff check .`, `ruff format --check .`, `mypy app` (strict) and `pytest -q` all exit 0, with zero failures and no skips other than live-marked tests. Observed: output.
15. **The README states the limitation.** `Select-String -Path README.md -Pattern 'X-User-Id'` prints at least one line. Observed: output.

## Boundaries

**Files and areas this phase must not touch:**

- `CLAUDE.md`, `GOALS.md`, `PLANS.md`, `KE-HOACH-CHI-TIET.md`, `.claude/`, and every other file in `harness/build/`.
- `harness/build-log.md`: only qa writes to it.
- `BE/eval/`: do not create it. It is phase 04's measuring instrument.
- `DB/docs/`: do not create it. Phase 02 writes the corpus.
- `.gitignore`: do not rewrite it. Add entries only if a new artifact needs ignoring.

**Next-phase work that must not leak in:**

- **Phase 02:** no embeddings, no `doc_chunks`, no vector columns, no ingestion, no `RAG_MODE` setting, no citations. Creating the `vector` extension is the only pgvector work in this phase.
- **Phase 05:**
  - no tool definitions, no agent loop, and no use of the provider's tool-calling API;
  - no `conversations` or `messages` tables, and no multi-turn history;
  - no dry-run or confirmation tokens;
  - no handoff endpoint;
  - no refund-window policy logic;
  - no "calling a tool…" state in the UI.
- **Phase 06:** no injection filter, no output filter, no masking module. Phase 01 keeps personal data out by not collecting, printing or logging it.
- **Phase 07:**
  - no Langfuse;
  - no `/admin/metrics` or any other staff-only endpoint;
  - no retry or backoff, no rate limiting, no caching, no prompt-caching columns;
  - no API Dockerfile and no compose `api` service;
  - no deployment.
- **Phase 08:** no markdown rendering, and nothing in the README beyond step 11.

**Also out of scope:**

- No real authentication: no JWT, sessions or passwords.
- No LangChain, LangGraph, LiteLLM, or any other LLM framework or gateway. Use the official provider SDK only.
- No commits, pushes or deployments. Each one needs the user's approval in the main session.

**Risk-flagged files for reviewer:**

- **Authorization:** `BE/app/auth.py`, `BE/app/shop/*`, `BE/app/routers/shop.py`, and identity on `BE/app/routers/chat.py`.
- **Personal data:** `BE/app/db.py` (`hide_parameters`), `DB/seed/*`, `BE/app/schemas/shop.py`, `BE/app/routers/chat.py`, `BE/app/obs/llm_calls.py`.
- **Consequential actions:** `POST /refunds`, and the partial unique index in migration 0001.
- **Cost and latency:** `BE/app/llm/*`, `BE/app/obs/pricing.py`, `BE/scripts/chat_smoke.py`.

## Verification

**Before you start:**

- Docker Desktop is running.
- `.env` exists, created with `Copy-Item .env.example .env`, and has a local Postgres password set.
- Run from the repo root in PowerShell, one command per line. Windows PowerShell 5.1 has no `&&`.
- In Git Bash, use `grep '@'` in place of `Select-String '@'`, and `grep -n 'X-User-Id' README.md` in place of the README check.

```
# 1. Environment
uv --version
docker --version
docker compose version
node --version

# 2. Database and dependencies
docker compose up -d --wait db
uv sync --directory BE
uv run --directory BE python --version

# 3. Migrations: round trip from an empty schema, then drift check
uv run --directory BE alembic upgrade head
uv run --directory BE alembic downgrade base
uv run --directory BE alembic upgrade head
uv run --directory BE alembic check
docker compose exec -T db psql -U shop -d shop -c "select extname, extversion from pg_extension where extname = 'vector';"
git grep --untracked -n create_all -- BE DB

# 4. Seed: idempotency, and no personal data in its output
uv run --directory BE python ../DB/seed/seed.py
uv run --directory BE python ../DB/seed/seed.py
uv run --directory BE python ../DB/seed/seed.py | Select-String '@'

# 5. Secrets and README
git check-ignore .env
Select-String -Path README.md -Pattern 'X-User-Id'

# 6. Static checks and tests
uv run --directory BE ruff check .
uv run --directory BE ruff format --check .
uv run --directory BE mypy app
uv run --directory BE pytest -q
uv run --directory BE pytest -v tests/shop tests/chat tests/test_pii_logging.py tests/test_seed_generation.py tests/test_settings.py
node --test FE/sse.test.mjs

# 7. Live provider smoke: needs the provider decision and a key in .env; one paid call
uv run --directory BE python scripts/chat_smoke.py

# 8. Browser check, by the main session before it launches qa
uv run --directory BE uvicorn app.main:app --port 8000
#    in the app's built-in browser: open http://localhost:8000/, enter a customer id,
#    ask a question in Vietnamese, read the page twice while the answer streams
```

**What counts as passing, by section:**

- **Section 1:** every command exits 0, and `uv run --directory BE python --version` prints `Python 3.12.x`.
- **Section 2:** compose reports the `db` container healthy.
- **Section 3:**
  - all three Alembic commands exit 0;
  - `alembic check` prints "No new upgrade operations detected.";
  - psql returns one `vector` row;
  - `git grep` prints nothing. Exit code 1 with no output is the pass condition.
- **Section 4:**
  - the first two runs print byte-identical counts and checksum, with counts inside the ranges of criterion 4;
  - the third command prints nothing.
- **Section 5:** `git check-ignore` prints `.env`, and `Select-String` prints at least one README line.
- **Section 6:**
  - ruff, format check and mypy exit 0;
  - `pytest -q` shows zero failures and no skips other than live-marked tests;
  - the `-v` run lists the authorization, refund, streaming, llm_calls, personal-data, seed and settings tests as PASSED;
  - the node test reports 0 failures.
- **Section 7:** the script exits 0, prints at least two `delta` offsets that increase and come before the `done` offset, and prints a row with non-null tokens and `cost_usd` above 0.
- **Section 8:** the main session reports two reads during one response, the second showing more answer text than the first, and passes that observation to qa in its prompt. qa records it with its date; without it, criterion 13 stays unverified.
