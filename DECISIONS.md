# Speaker Tracker — Decisions

The choices that shaped the app, with the reason for each and what it costs. Many of these look like
something to "fix" until you know why they are there. Each entry says what was decided, why, and
what follows from it. Dates are when the decision was made.

What the app does is in [PRODUCT.md](PRODUCT.md); how it is built is in
[ARCHITECTURE.md](ARCHITECTURE.md). The original slice-by-slice plan, with its acceptance criteria,
is archived in [archive/DEV-PLAN.md](archive/DEV-PLAN.md).

**Contents** — [Platform](#platform) · [Data](#data) · [Email](#email) · [Pipeline and money](#pipeline-and-money)
· [Outreach and targets](#outreach-and-targets) · [Follow-ups and appointments](#follow-ups-and-appointments)
· [Frontend](#frontend) · [Process](#process)

---

## Platform

**One API function, not one per route group** *(2026-07)*. Powertools' resolver with a `Router` per
route group. *Why:* each function cold-starts on its own and pays a 2–6 s TLS handshake to the
database; with one occasional user, twenty functions would mean a cold start on nearly every action.
*Follows:* the database connection can be reused at module scope; background work (poller,
migrations, reminders) stays in separate functions because their triggers, timeouts and permissions
differ.

**Functions outside any VPC, database over TLS with IAM auth** *(2026-07)*. *Why:* no NAT gateway or
ENI cold-start cost, no password in transit. *Cost:* a public database endpoint gated by IAM auth and
TLS (hostname verified), and a 2–6 s cold-start handshake.

**Share an existing RDS instance, new schema** *(2026-07)*. *Why:* a new schema on a running instance
costs nothing, and the connection and migration code already existed. *Follows:* the parameter group
belongs to another project — hence the per-session strict `sql_mode` below — and connections are
bounded by reserved concurrency.

**Same-origin CloudFront** *(2026-07)*. The SPA and `/api/*` share one distribution, with a CloudFront
Function stripping `/api`. *Why:* no CORS and no environment-specific API URL in the build.

**CDK in TypeScript, explicit routes** *(2026-07)*. Every route is declared (84 today) rather than
`ANY /{proxy+}`. *Why:* `/health` can stay unauthenticated while everything else carries the
authorizer, and the gateway rejects unknown paths itself.

**Strong cross-stack references** *(2026-07)*. `cdk.json` sets `defaultCrossStackReferences` to
`strong`, against the recommended default. *Why:* with weak references a recreated API left
CloudFront pointing at a deleted gateway while every deploy reported success; strong references make
CloudFormation refuse instead.

**The poller, the scheduler and the reminder functions live in the Api stack** *(2026-07)*, not in
Messaging as first planned. *Why:* they need the content bucket and the shared database, which Api
already has; putting them in Messaging would have needed exactly the cross-stack references that
broke the CloudFront origin. Messaging holds only the IMAP secret and imports and exports nothing.

**Cognito: invite-only, Managed Login, ID token, 90-day sessions** *(2026-07)*. No self sign-up, user
pool retained on teardown; Managed Login rather than the classic Hosted UI (in maintenance, no real
branding); the SPA sends the ID token because it carries the `aud` the JWT authorizer validates; a
90-day refresh token with silent renewal and no splash screen. *Why the long session:* one user on a
fixed desktop signs in about quarterly. *Revisit* for more users or mobile.

**The API creates the `users` row** *(2026-07)*, lazily on the first authenticated request. *Why:* a
Cognito post-confirmation trigger has a 5 s limit against a 2–6 s cold handshake, and an
administrator-created user is already confirmed, so the trigger may never fire.

**Sandbox has no authentication** *(2026-07)*. *Why:* half the infrastructure, no second certificate.
*Guard:* the API refuses to start with dev auth outside sandbox. *Cost:* sandbox must hold only
disposable data.

**Exactly one environment polls the mailbox** *(2026-08)*. *Why:* two pollers race over the Import
folder, and a dragged message lands in one database at random. Production polls; a test pins it.
*Cost:* sandbox cannot exercise inbound mail without a mailbox of its own.

**Strict `sql_mode` per session** *(2026-10)*. *Why:* the shared instance is non-strict, so
production silently truncated values the tests rejected (that is how one review bug hid). The
parameter group is not ours to change; a session setting is. Each connection logs the mode the server
applied, because performance_schema is off on the instance and there is no other way to see it.

## Data

**MySQL, raw SQL, no ORM; catalog tables, not ENUMs** *(2026-07)*. Two ENUMs are deliberate
(cadence, message direction) — see [DATABASE.md §5](DATABASE.md#5-deliberate-deviations).

**Foreign keys never cascade** *(2026-07)*. *Why:* deleting a parent should fail loudly, not take
children with it. Soft delete is the normal path.

**A person is not owned by a venue** *(2026-07)*. Contacts and venues are many-to-many, with title,
primary and power-partner **per affiliation**. *Why:* in a small market one person is the contact for
several venues.

**`closed_at` is stored, not computed** *(2026-07)*. *Why:* its predicate depends on flags in two
catalog tables, which a generated column cannot read; and the board/History split is on nearly every
query.

**`follow_ups` has no status column** *(2026-07)*. `completed_at` is the state. *Why:* one fact in two
columns eventually disagrees.

**`research_ready_at` is stamped once and never cleared** *(2026-08)*. *Why:* "venues researched this
month" is a flow and needs a date; re-stamping a venue that lost and regained its contact would move
it into a later month and change a number already reported.

**Appointments use `DATETIME`** *(2026-08)*. *Why:* a wall-clock commitment must not move with the
session time zone.

**Response counters are counters, with no date and no delete** *(2026-08)*. *Why:* who responded and
when is tracked elsewhere; here only the count matters, zero is the empty state, and the write sets a
value rather than adding one, so a repeated click is harmless.

**Request caps equal column widths, and TEXT is capped at 16,000 characters** *(2026-10)*. *Why:* an
overlong value should be a clear 400 before any SQL runs. `TEXT` is 65,535 bytes and utf8mb4 uses up
to 4 per character; production held nothing near the cap when it was introduced.

**Ownership checks are one helper over an allow-list** *(2026-10)* (`_ownership.owns_row`). *Why:*
thirteen hand-copied "does this row belong to the caller?" queries had already started to differ. A
schema test keeps the list complete.

## Email

**WorkMail + SES + IMAP, not a mail API** *(2026-07)*. The user's mailbox is WorkMail, read in an
ordinary mail client. The app sends with SES `SendRawEmail` (full header control, IAM-authorized)
and appends a copy to the Sent folder over IMAP, rather than sending through authenticated SMTP.

**No cross-account access** *(2026-07)*. The mailbox is in another AWS account; sending uses a domain
identity in the app's account (domain verification covers every address), and IMAP is
username/password. A cross-account sending authorization was rejected as more moving parts.

**The app reads two consented surfaces, never the whole mailbox** *(2026-07)*: correspondence with
tracked contacts (or continuing a known thread), and messages the user moves into an Import folder.

**Import by folder move, not forward** *(2026-07)*. *Why:* a move keeps the original message, so its
`Message-ID` survives and the reply threads at the other end; forwarding rewrites it.

**Store both Message-IDs** *(2026-07)*. SES replaces the header the app mints, so the minted id (the
idempotency key) and the one the recipient sees are both stored and matched. Matching only the minted
one silently dropped every reply from someone not already a contact.

**Intent-then-confirm sends with a client idempotency key** *(2026-07)*. A pending row before SES,
discarded only on a clean rejection; an ambiguous failure keeps it. The Message-ID derives from a key
the SPA holds for the life of a draft, so a retry returns 409 instead of sending twice. The key is
required, not optional, so the compiler finds every caller.

**Threads close explicitly; a reply reopens them** *(2026-07, 2026-10)*. Nothing infers that a sent
message is owed a reply. A gig's threads close only when the gig moves from open to closed, so a
thread the user reopened is not re-closed by a later edit. An inbound reply that is the thread's
newest message reopens it, so a reply is never buried.

**An IMAP auth failure alarms; a transient rejection does not; an unrecognised one does**
*(2026-07, 2026-08)*. *Why:* the worst failure is a poller that runs, finds nothing, and says
nothing. A transient `[UNAVAILABLE]` once paged on a healthy mailbox; the response code now decides,
failing toward alarming.

**A stuck message jams its folder, and alarms** *(2026-10)*. The poller does not skip a message it
cannot ingest. *Why:* skipping loses mail silently; a jam that pages someone does not. Automatic
skipping after N failures was considered and declined.

**Remote images are held back; trust is per address** *(2026-10)*. *Why:* loading a remote image tells
the sender the message was opened, when and from where. Trusting a domain would trust everyone at a
large mail provider.

**An attachment is a copy, never a link** *(2026-08)*. *Why:* mail already sent must not change when a
library file is replaced.

## Pipeline and money

**Pro bono is first-class, and Cancelled counts as a booking** *(2026-07)*. Pro bono builds
visibility and is expected; a cancellation is the leak between Booked and Delivered, not a gig that
never booked.

**A delivered gig stays on the board until it is paid** *(2026-07)*. *Why:* closing on delivery would
hide money still owed.

**The fee field records value on unpaid gigs; only paid gigs count as money** *(2026-10)*. *Why:* the
value of a trade or pro bono engagement matters for tax reporting, but it was never received.
Received and History's collected total both require comp type *paid* and payment status *paid*.

**The week picker moves the target tiles only** *(2026-08)*. *Why:* Outstanding is a balance, not a
flow; the funnel is a snapshot of where gigs sit; a week of Booked is usually zero. If money is ever
windowed, key it off the event date over a quarter, and surface booked gigs with no event date rather
than dropping them.

## Outreach and targets

**Outreach is outbound only, and only prospecting counts** *(2026-07)*. Received mail never creates an
outreach row, and `counts_toward_target` gates the metric. *Why:* a logistics thread with a booked
venue would otherwise turn *outreaches per week* into a measure of email volume. All outreach counting
originates inside the app.

**The outreach kind is inferred once, shown, and never re-inferred** *(2026-07, 2026-08)*. *Why:*
right by default and always correctable, with no friction on a one-line reply; re-inferring on an edit
would let a typo fix change whether a touch counts. A touch cannot move to another contact — that is
delete and re-log.

**DMs are logged, not sent** *(2026-07)*. There is no practical send API for social DMs; templates
merge and copy to the clipboard instead.

**Targets are activity-based; no money target** *(2026-07)*. The model is generic, so a money target
would be a new target type when it is wanted.

**Responses are not a target** *(2026-08)*. No goal, no tile; they measure audience growth in
aggregate.

## Follow-ups and appointments

**Reminders: one email, local time, no database in the sender** *(2026-07)*. EventBridge Scheduler
with a deterministic schedule name, the user's IANA zone in the expression, and a payload carrying
everything the email needs. *Why:* no state read-back, no UTC arithmetic, and the sender needs no
database access. *Follows:* editing a follow-up replaces its schedule; marking it done cancels it.

**A small retry budget, a visible failure** *(2026-07)*. Five attempts over two hours, then a
dead-letter queue whose consumer marks the follow-up and alarms. *Why:* a morning nudge delivered
after lunch has missed its purpose, and 185 retries at a bad address damage sending reputation.

**Riders are opt-in** *(2026-07)*. Logging or sending never silently schedules a follow-up.

**Accepted dual writes** *(2026-07)*. Follow-up row → schedule, and send → rider row, are not atomic.
A general outbox was judged not worth its machinery for one user; database first, outside effect
second, everywhere. See [ARCHITECTURE.md §5.2](ARCHITECTURE.md#52-dual-writes-we-accept).

**Appointments are a log, not a calendar** *(2026-08)*. No sync, no invites, no completion state; the
page keeps past ones reachable so a mistyped date can be corrected.

## Frontend

**Mantine + TanStack Query + dnd-kit + Tiptap** *(2026-07)*. TanStack Query because the optimistic
kanban drag needs rollback; Mantine for forms, notifications and the rich-text editor.

**Errors persist until dismissed** *(2026-10)*. Every failed save says what failed, why, and a request
reference with a copy button; only success notices auto-dismiss. Failed loads show in place with
*Try again* rather than as an empty state.

**Filters live in the URL** *(2026-08)*. Every list is linkable, the Dashboard's drill-down links are
just URLs, and the back button works.

**Light theme, sans-serif, brand by colour** *(2026-07)*. The user dislikes dark themes; the brand's
display fonts are for the public website, not a working tool.

## Process

**CI from the first slice, no deploy step** *(2026-07)*. Deploys stay manual.

**The test schema is built by the real migration runner** *(2026-07)*. The test schema cannot drift
from production, and the runner is exercised on every run.

**No sample data in any environment** *(2026-07)*. Reference data (catalogs, the shared templates) is
the only seed.

**Prove a regression test against the old code** *(2026-07)*. A test that passes either way has
shipped bugs here more than once; mutation-checking is routine.