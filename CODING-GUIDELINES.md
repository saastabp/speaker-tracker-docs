# Coding Guidelines

Conventions for the Python 3.12 backend, the React frontend and the CDK app. They describe how the
code *is* written — a change that breaks one should either follow it or change this file in the same
commit. Architecture is in [ARCHITECTURE.md](ARCHITECTURE.md); the schema is in
[DATABASE.md](DATABASE.md).

**Contents** — [1. Backend layering](#1-backend-layering) · [2. Data and persistence](#2-data-and-persistence)
· [3. Functions and modules](#3-functions-and-modules) · [4. Reuse](#4-reuse) ·
[5. Errors and logging](#5-errors-and-logging) · [6. Frontend](#6-frontend) ·
[7. Mirrored rules](#7-mirrored-rules) · [8. Testing](#8-testing) ·
[9. Documentation](#9-documentation)

---

## 1. Backend layering

Three layers plus an edge layer; **dependencies point inward**.

| Layer | Directory | Responsibility | May import | Must not contain |
|---|---|---|---|---|
| **Presentation** | `handlers/` | Parse and validate the request, check ownership, call repositories and core, map the result or exception to a response, boundary logging | core, repositories, models, common | Business rules |
| **Core** | `core/` | Pure domain logic | models, stdlib | SQL, AWS SDK, HTTP shapes, clock or environment reads |
| **Data** | `repositories/`, `models/`, `migrations/` | Persistence — the **only** place SQL lives; pydantic models | common, models, stdlib | Business rules, HTTP shapes |
| **Edge** | `common/` | Side effects: database, S3, SES, IMAP, Scheduler, secrets, logging, HTTP mapping | stdlib, third-party | Anything from the other layers |

- **The handler is the composition root.** It fetches what core needs from repositories, passes it
  to core as **plain data** (rows, primitives, small dataclasses), and writes core's decision back.
  Core never receives a repository, a connection or a client, which is what makes it testable with
  no database and no mocks.
- **Keep handlers thin.** If a handler starts making business decisions, move the decision to
  `core/`.
- **`common/` is a leaf.** It never imports `core`, `repositories`, `handlers`, `models` or
  `migrations`; translate between them at the handler (that is why `common/scheduler` takes an
  expression and a payload dict, not a core type).
- Both rules are **enforced by ruff** (`flake8-tidy-imports` banned-api in `core/.ruff.toml` and
  `common/.ruff.toml`), so a violation fails CI rather than starting a review argument.

---

## 2. Data and persistence

- **Pydantic v2** for request and response models — validate at the boundary. Keep API models and
  persistence shapes separable; they may coincide for simple CRUD.
- **Raw SQL over pymysql, no ORM.** Parameterized queries only; never format a value into SQL.
  Interpolating a *table name* is acceptable only from a fixed allow-list (as
  `_ownership.owns_row` does).
- **Owner-scope everything.** Every query filters on the caller's `user_id`, and every write that
  references another row checks it with `_ownership.owns_row` / the `validate_*` helpers first — a
  foreign key proves existence, not ownership. A foreign id must behave exactly like a missing one.
- **Request caps equal column widths.** A request-model string writing a `VARCHAR(n)` has
  `max_length=n`; one writing `TEXT` uses `models/limits.MAX_TEXT_CHARS`. `tests/test_model_widths.py`
  enforces it. Shared limits live in `models/limits.py`, not as literals.
- **Strict `sql_mode`** is set on every connection (`common/db._SQL_MODE`), so a write that does not
  fit is an error in production exactly as in tests. Do not work around a strict-mode error by
  loosening the mode; fix the value or the cap.
- **Migrations** are forward-only hand-written `.sql`, numbered above the highest applied version,
  never edited once applied, and re-runnable where at all possible (`IF NOT EXISTS`, `ON DUPLICATE
  KEY UPDATE`, guarded `ALTER`s). See [DATABASE.md §6](DATABASE.md#6-migrations).
- **Catalog tables over ENUMs**; repositories resolve `short_name ↔ id` at the boundary, so callers
  pass `short_name`. Metric SQL joins catalog flags (`counts_toward_target`, `is_settled`,
  `is_terminal`) rather than hardcoding short names.
- **Connections** come from `common/db.get_connection` (reused at module scope, probed by the
  per-request `SET time_zone`, reconnected **once** with a fresh IAM token).
  - 🚫 **Never `ping(reconnect=True)`.** It reconnects with the *expired* token stored on the
    connection — failing intermittently on any container older than 15 minutes — and silently drops
    session state. A CI grep enforces this; ruff cannot see the call.
  - `autocommit=True`; wrap multi-statement writes in `with transaction(conn):`, or a handler that
    raises mid-write leaves locks held into the next invocation.
  - Never connect at import time — a database outage should fail `/catalogs`, not `/health`.
  - TLS verifies both the certificate **and** the hostname (`ssl_verify_identity=True`).
- **"Now" comes from the database** (`common/db.db_now_local`) — the same clock that stamps every
  stored timestamp, already in the user's zone.
- **Commit to the database first, then the outside effect** (schedule, email). See
  [ARCHITECTURE.md §5.2](ARCHITECTURE.md#52-dual-writes-we-accept).

---

## 3. Functions and modules

- **Small, single-purpose functions**; separate pure computation from I/O, and pass dependencies in.
- **File size.** ~300 lines is the target, **over 500 is a review trigger**, 1000 is a hard limit.
  A large file usually means too many responsibilities; split by route group, aggregate or concept.
  Current source files over 500 lines: `infra/cdk/lib/api-stack.ts`, `repositories/opportunities.py`,
  `pages/Dashboard.tsx`, `pages/Pipeline.tsx`, `core/email_headers.py`, `repositories/dashboard.py`,
  `handlers/imap_poll.py`, `pages/OpportunityDetail.tsx` — candidates when next touched, not
  emergencies.
- **NumPy-style docstrings** on every public function, method and class: a one-line summary, then
  `Parameters`, `Returns` (or `Yields`), `Raises`, and `Examples` for non-obvious usage, with
  `name : type` headers. Private functions (leading underscore) need only the summary line.
- **Python style**: ruff `check` and `format`, line length 100. CI runs both.

---

## 4. Reuse

- **Use the maintained package or the existing helper; don't hand-roll.** A helper that duplicates
  a package or a sibling module needs a one-line justification in the code.
- **Before writing a helper, search for one.** Copying a helper between modules has already shipped
  a real bug here (a UTC day-shift). The shared ones to reach for:

  | Need | Use |
  |---|---|
  | Durations in logs | `common.logger.elapsed_ms` |
  | AWS region resolution | `common.aws.resolve_region` |
  | Address parsing, domains, sender identity | `core.email_headers` (`address_list`, `normalize_address`, `address_domain`, `sender_domain`, `sender_address`) |
  | Ownership checks | `repositories._ownership.owns_row` and the `validate_*` wrappers |
  | Upload keys | `common.storage.upload_key` |
  | Boolean / choice query parameters | `handlers.params.query_bool`, `query_choice`, `path_int` |
  | Duplicate-key detection | `pymysql.constants.ER.DUP_ENTRY` |
  | Frontend dates | `dates.ts` (`parseDateLocal`, `parseTimestamp`, `longDate`, …) — never `new Date('YYYY-MM-DD')`, which is UTC midnight |
  | Frontend money and fees | `format.ts` (`formatMoney`, `isValidFee`) |
  | Frontend catalog labels and flags | `api/catalogs.ts` (`catalogLabel`, `isSettled`) |
  | Frontend uploads | `api/uploads.ts` (`uploadFile`) |

- **Packages in use:** aws-lambda-powertools (logging, metrics, the HTTP resolver), boto3,
  pydantic v2, pymysql, sqlparse (splitting migration files — never `split(";")`), imapclient, the
  stdlib `email` package and `zoneinfo`. **uv** manages dependencies and builds the Lambda bundle for
  arm64 (`--only-binary=:all:`, so a compiled wheel can never be built for the wrong architecture).

---

## 5. Errors and logging

- **Typed domain exceptions** (`common/errors`: `NotFound`, `Conflict`, `InvalidInput`) carry no HTTP
  concerns; `common/http.response_for_exception` maps them (404, 409, 400; a pydantic
  `ValidationError` is 400; anything unrecognised is 500 after `logger.exception`).
- No bare `except`. Catch the specific exception; `logger.exception` then re-raise or map.
- **A swallowed error, a fallback or a default substituted for bad input logs at WARNING** — so
  monitoring sees it. Silence is reserved for genuinely expected outcomes.
- **Every handler for an external trigger** (HTTP route, scheduled job, queue consumer, deploy
  trigger) logs:
  - an **entry** line: correlation id (generated if absent), operation, key non-sensitive ids;
  - an **exit** line just before returning: status, `duration_ms` from entry, the same correlation
    id;
  - on an error response: the response body (secrets redacted), the stack trace, and the correlation
    id, at ERROR;
  - its **key operations**: external calls (target, status, latency), database writes (operation +
    entity id, not rows), state transitions (`from → to`), authorization decisions, queue
    operations.
- Powertools `Logger` (or stdlib `logging`), never `print`. **Lazy formatting** (`logger.info("x=%s",
  x)`), never f-strings, inside log calls. A float duration uses `%s`, not `%d`.
- **Never log** passwords, tokens, API keys, mailbox credentials, presigned URLs, raw request bodies
  or raw Lambda events (the event carries the JWT).
- A log line an alarm depends on carries a **stable machine key** (`extra={"poll_stuck": …}`), and
  the alarm matches the key, never the message text.

---

## 6. Frontend

- **TypeScript strict; `tsc` is the gate.** There is no ESLint or Prettier configuration — match the
  surrounding style.
- **Server state is TanStack Query.** Each resource module exports a **key factory**
  (`opportunityKeys`, `dashboardKeys`, …); invalidate through it, never with a string literal, so a
  renamed key cannot silently stop a refresh. Window-focus refetching is off, so **every write must
  invalidate everything it changes** — including other resources it touches (a send also changes
  outreaches, the timeline, the dashboard and follow-ups).
- **Mutations use `useAppMutation`**, which gives every failure a persistent notification with the
  request reference (or an inline `FormError` when the form shows it). Errors never auto-dismiss;
  only success notices do. **Failed loads render `QueryError`** in place, never an empty state.
- **Filters live in the URL** (`useFilterParams`), so every list is linkable and the back button
  works.
- **Server-owned vocabulary.** Stage names, order and labels come from `/catalogs` and `/funnel`;
  never hardcode a stage name or re-derive `sort_order`.
- **Untrusted HTML goes through `SafeHtml`.** Received mail and rendered Markdown (`markdown.ts`
  passes raw HTML straight through) reach the page only via `SafeHtml` / `emailHtml.ts`, never
  `dangerouslySetInnerHTML` directly; never hand-write a DOMPurify `ALLOWED_URI_REGEXP` (it applies
  to every attribute).
- **Request types mirror the backend models.** Each `api/*.ts` module says which backend model it
  mirrors; keep required fields required on both sides (the send's `idempotency_key` is required in
  both so the compiler finds every call site).

---

## 7. Mirrored rules

Where a rule must exist on two sides, both copies carry a comment naming the other, and a change to
one must change the other in the same commit.

| Rule | Owner | Mirror |
|---|---|---|
| Money received / collected: paid comp **and** payment status `paid` | `repositories/dashboard.money_rollup` | `pages/History.tsx` `collectedTotal` |
| Fee precision — `DECIMAL(10,2)` | `models/opportunities.py` `fee_amount` | `format.ts` `isValidFee` |
| Largest material upload (25 MB) | `common/storage.MAX_MATERIAL_BYTES` | `api/materials.ts` `MAX_MATERIAL_BYTES` (a courtesy; the server enforces) |
| Longest material description (500) | `models/materials.py` `MAX_DESCRIPTION_CHARS` | `api/materials.ts` `MAX_DESCRIPTION_CHARS` (the form's cap; the server enforces) |
| Outreach-kind default (*initial* vs *correspondence*) | `core/outreach.infer_outreach_kind` | `components/OutreachFormModal.tsx` (display only) |
| Initial payment status by comp type | `core/opportunities._INITIAL_PAYMENT_STATUS` | `components/OpportunityFormModal.tsx` |
| Target cadences | `core/periods.CADENCES` and the `targets.cadence` ENUM | `api/targets.ts` |
| Appointment scopes | `repositories/appointments._SCOPE_WHERE` | `api/appointments.ts` `AppointmentScope` |
| SES region | `common/mail.SES_REGION` | `infra/cdk/lib/config.ts` `SES_REGION` and the identity ARN |
| Inline image types — send vs display, **deliberately different** | `common/mail.py` (what is embedded) | `emailHtml.ts` `SAFE_INLINE_IMAGE` (what is shown: any image type except SVG) |
| Development principal | `common/auth.py` | `auth/DevSession.tsx` |
| Stuck-poll log key `poll_stuck` | `handlers/imap_poll.py` | `infra/cdk/lib/api-stack.ts` `ImapPollStuckFilter` |
| Request and response shapes | `models/*.py` | `api/*.ts` interfaces |

Not mirrored, on purpose: period maths (week starts, quarter edges) lives only in `core/periods.py`,
and the frontend uses the bounds the server sends.

---

## 8. Testing

- **Backend: pytest**, flat under `backend/tests/` as `test_<module>.py`. Core is tested as pure
  functions. Repository and handler tests run against a real MySQL 8.4 schema built by the migration
  runner, and skip when `TEST_DATABASE_URL` is unset.
- **Only one pytest run at a time** against the shared test database — concurrent runs drop each
  other's tables and produce convincing fake failures.
- **Frontend: vitest** for pure logic (helpers, rules exported from pages), run in
  `TZ=Pacific/Honolulu` so date bugs at a negative UTC offset show up.
- **Infra: jest** over the synthesized stacks. Assert a resource's **value** — its wiring, pattern,
  threshold — not merely that it exists; several real outages synthesized cleanly and deployed green.
- **A regression test must fail against the old code.** Prove it (revert the fix, or mutate the
  logic) before trusting it; a test that passes either way is worse than none.
- Aim for meaningful coverage of core, repositories and the rules above; do not chase 100%.

---

## 9. Documentation

- The documents in `docs/` describe the system **as built**: [docs/README.md](README.md) is the
  index. Plans and records of finished work live in `docs/archive/`.
- **Update the document in the same change** that alters the behaviour, schema or wiring it
  describes. Mermaid diagrams (ER diagram, flowcharts) live in the Markdown so they diff with the
  change; the architecture diagrams in `docs/diagrams/` have editable sources beside their exports.
- **Docs are published.** `.github/workflows/publish-docs.yml` copies `docs/` to a public,
  docs-only repository on every push to `main` that touches it. No account ids, resource ids, mailbox
  addresses, personal names (other than the author credit on the landing page) or credentials; use
  placeholders (`<account-id>`, `app.example.com`). The workflow scans for real values and refuses to
  publish if it finds one — a new kind of real value belongs in its pattern list. Keep every link
  inside `docs/` relative to `docs/`, so the copy works unchanged. Operational specifics belong in
  code (`infra/cdk/lib/config.ts`) or in the untracked `docs/OPERATIONS.local.md`.