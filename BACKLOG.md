# Speaker Tracker — Backlog

What is not built, not finished, or knowingly imperfect. The single list of "what next"; nothing here
is promised. When an item ships, delete it here and describe it in the document it belongs to.

**Contents** — [Queued features](#queued-features) · [Hazards to fix](#hazards-to-fix) ·
[Deferred deliberately](#deferred-deliberately) · [Unverified](#unverified) · [Maintenance](#maintenance)

---

## Queued features

- **Rename and move materials.** Rename a material, and link it to a talk — or back to *standalone* —
  after upload. A material is legitimately standalone (a general one-sheet) or tied to one talk, and
  must be movable either way when it was filed wrong or its use changes. The API supports both
  (`PUT /materials/{id}` replaces name and `talk_id`; `talk_id` null is standalone) and a hook exists
  unwired (`useRenameMaterial`); the UI has neither.
- **Attach an existing contact from the import card.** Today an imported message only offers *Add
  contact*, so a known person writing from a new address cannot be cleared without creating a
  duplicate. The API and hook exist (`PUT /emails/threads/{id}/contact`, `useLinkThreadContact`);
  the button and its picker do not.
- **Email threads on contact and gig pages.** Threads are reached only from the inbox (search and the
  contact filter include closed ones). A thread list on a contact's page and on a gig's page.
- **A general reminder ("tickler") system.** *Needs attention* has five hard-coded reasons with fixed
  windows (14 days quiet, 7 days awaiting reply). A general system would let each kind of reminder
  carry its own timing, likely with its own table.
- **Mockup gaps confirmed still open:** a *Last touch* column and an *Activity* panel on venues;
  LinkedIn and Instagram fields on contacts.

## Hazards to fix

- **Sandbox can email real people.** Polling is limited to one environment; sending is not. Sandbox
  sends from the production address, so a reminder or an email sent from sandbox reaches a real
  recipient. Options: an environment-gated sender, or an allow-list of recipients in sandbox.
- **Harden the mailbox login.** The IMAP credential is a password with no second factor. IMAP itself
  cannot do MFA, so this means restricting or monitoring that login rather than changing the app.
- **The poller fetches a capped batch in one `FETCH`.** A batch that cannot transfer within the
  function's 120 s timeout would fail, never advance the cursor, and retry the same batch forever (the
  stuck alarm would now fire). Never hit; if it is, lower `MAX_UIDS_PER_POLL` or fetch in
  sub-batches — and do not point a `UIDVALIDITY` reset at a large INBOX casually.

## Deferred deliberately

Each has a reason in [DECISIONS.md](DECISIONS.md); revisit when the reason stops holding.

- **More than one user.** Its own project, with its own requirements. Known work: the sending
  address, the poller and reminders all assume one user's mailbox; shared templates are editable by
  anyone; sign-out does not end the Cognito session and the query cache is not cleared; S3 tenancy is
  enforced only by the application; one-off integrity checks across tenants.
- **More than one email address per contact** (`contact_email_addresses`). Today a second address
  arrives as an unknown sender.
- **Windowed money and funnel figures.** Key money off the event date over a quarter, and surface
  booked gigs with no event date rather than dropping them.
- **A money target** — a new target type, once activity targets stop being the point.
- **SES configuration sets** for bounce and complaint tracking, once outreach volume is real.
- **A custom MAIL FROM subdomain** for tighter DMARC alignment (DMARC is `p=none`, so not a blocker).
- **Seeding a gig's angle from the venue's *how to approach*,** and a research panel in the composer.

## Unverified

Built and covered by tests, but never observed end to end:

- **The broken-`References` fallback** (sender + normalized subject + time window) threading a real
  message.
- **The Import and Processed folders appearing in a mail client that filters by subscription** (the
  app creates *and subscribes* them; verified only in webmail, which ignores subscriptions).
- **Inbound mail never creates an outreach row** — true of all traffic so far, but never asserted as
  its own live check.
- **The dead-letter message shape** for a failed reminder — no reminder has failed yet; see
  [OPERATIONS.md](OPERATIONS.md#followup-failures--a-reminder-email-gave-up).
- **No production smoke test.** Production is checked by hand after each deploy.
- **Mockup items to re-check against the app** (from the archived UX reconciliation, whose DEFER
  tags predate several slices): reopening a closed gig, an invoice date and number, the payment
  method, template usage counts, attaching a venue while creating a contact.

## Maintenance

- **Large files** — over the 500-line review trigger ([CODING-GUIDELINES.md §3](CODING-GUIDELINES.md#3-functions-and-modules)):
  `infra/cdk/lib/api-stack.ts`, `repositories/opportunities.py`, `pages/Dashboard.tsx`,
  `pages/Pipeline.tsx`, `core/email_headers.py`, `repositories/dashboard.py`,
  `handlers/imap_poll.py`, `pages/OpportunityDetail.tsx`. Split when next substantially changed.
- **CDK's `url.parse()` deprecation warning** comes from `aws-cdk-lib`'s custom-resource handler;
  check whether it is gone at the next `aws-cdk-lib` upgrade.