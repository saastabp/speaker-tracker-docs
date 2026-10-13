# Speaker Tracker — Operations

Deploying, checking a deploy, and what to do when an alarm fires. Placeholders stand in for every
account-specific value: `<profile>`, `<env>` (`sandbox` or `prod`), `<app>` (`speaker-tracker`).
The real values — accounts, mailbox, profile, domain — are in `infra/cdk/lib/config.ts` and in the
untracked `docs/OPERATIONS.local.md`.

How the system is built is in [ARCHITECTURE.md](ARCHITECTURE.md).

**Contents** — [1. Deploying](#1-deploying) · [2. Checking a deploy](#2-checking-a-deploy) ·
[3. Alarms](#3-alarms) · [4. Common tasks](#4-common-tasks) · [5. New environment](#5-a-new-environment)
· [6. Known hazards](#6-known-hazards)

---

## 1. Deploying

Deploys are manual; CI never deploys.

1. **Run the checks locally** — backend `ruff check`, `ruff format --check` and `pytest`; frontend
   typecheck, vitest and build; infra typecheck and jest.
2. **Commit first.** The Frontend stack builds the SPA during synth and stamps the footer with the
   commit (`v<version> · <sha> · <env>`), adding `-dirty` if the working tree has *any* uncommitted
   change. A dirty stamp points at no reviewable commit.
3. **Sandbox, then production:**

   ```bash
   cd infra/cdk
   npx cdk deploy '<app>-sandbox-*' --profile <profile> --region us-west-2 --require-approval never
   npx cdk deploy '<app>-prod-*'    --profile <profile> --region us-west-2 --require-approval never
   ```

   Quote the wildcard, and **never use `--all`**. `Api` and `Frontend` must deploy together (the
   Frontend's `/api/*` origin is the Api stack's HTTP API). Stack ids are `<app>-<env>-<Role>`.

4. **Migrations run during the deploy.** The `migrate` function applies every pending migration
   through a CDK `Trigger`; a failed migration fails the deploy. When reasoning about what a
   production deploy will apply, compare what **production** is running with what is shipping — not
   just the last commit, which may not be the only undeployed one.

**Version bumps.** The version shown in the footer comes only from `frontend/package.json`:

```bash
cd frontend
npm version <x.y.z> --no-git-tag-version   # updates package.json and both lockfile entries
```

Commit, then deploy. Never edit the lockfile's version by hand.

---

## 2. Checking a deploy

All of these are read-only.

- **Footer** shows `v<version> · <sha> · <env>` with the expected commit and no `-dirty`.
- **Migrations** — the `migrate` log for the deploy, or `SELECT version, status FROM
  schema_migrations ORDER BY version DESC LIMIT 3;`.
- **Strict mode** — each function logs `Opened RDS connection sql_mode=ONLY_FULL_GROUP_BY,…` on a new
  connection; a WARNING containing "not the strict mode requested" means the server ignored it.
  CloudWatch search lags new events by a minute or two.

  ```bash
  aws logs filter-log-events --profile <profile> --region us-west-2 \
    --log-group-name /aws/lambda/<app>-<env>-api --start-time <epoch-ms> \
    --filter-pattern '"Opened RDS connection"'
  ```

- **Poller** (production only) — `Poll end … status=ok` every minute, no `poll_stuck`, no ERROR.
- **Alarms** — all three `OK`:

  ```bash
  aws cloudwatch describe-alarms --profile <profile> --region us-west-2 \
    --alarm-name-prefix <app>-<env>- --query 'MetricAlarms[].[AlarmName,StateValue]' --output text
  ```

- **Browser** — load a few pages; a failed load shows an error card with *Try again* and a request
  reference rather than an empty page.

To test the alarm email path end to end, force an alarm (it returns to its real state at the next
evaluation):

```bash
aws cloudwatch set-alarm-state --profile <profile> --region us-west-2 \
  --alarm-name <app>-sandbox-imap-poll-stuck --state-value ALARM \
  --state-reason "Manual test of the notification path"
```

---

## 3. Alarms

All three alarms email the `<app>-<env>-ops-alerts` topic. The email arrives once, when the alarm
fires; there is no "resolved" email. **Do not click the unsubscribe link** in an alert: it is one
click, with no confirmation, and mutes all three alarms for that environment.

### `imap-poll-failures` — the mailbox refused the credentials

Inbound mail is **not** being processed. Transient rejections never reach this alarm (they skip one
poll), so this means the credentials themselves were refused, or refused in a way the code does not
recognise.

1. In `/aws/lambda/<app>-<env>-imap-poll`, find `credentials rejected after refresh`.
2. Check the mailbox user is enabled and the password in the IMAP secret is current. The secret does
   **not** rotate; it changes only when someone changes it.
3. To update the secret, see `OPERATIONS.local.md`. The next poll picks the new value up (the poller
   re-fetches the secret after a rejection).

### `imap-poll-stuck` — a message or folder keeps failing

The poller stops at a message it cannot ingest, so that message is not skipped — which means **no
later mail in that folder is processed** until it is fixed. The alarm fires after about 15 minutes of
steady failure.

1. In the poller log, search for `poll_stuck`. Each line names the folder and UID (`"message"`) or
   the folder (`"folder"`), with the stack trace.
2. **Fix the cause** and do nothing else: the next poll retries the message and carries on.
3. **To give up on one message deliberately** (it can never be ingested), move the folder's cursor to
   its UID, so the next poll starts above it:

   ```sql
   UPDATE imap_folder_cursors SET last_seen_uid = <uid>
    WHERE user_id = <user-id> AND folder_name = '<folder>';
   ```

   The skipped message stays in the mailbox; it just never enters the app. If it was in
   `Speaker Tracker/Import`, move it out by hand.

### `followup-failures` — a reminder email gave up

A reminder exhausted its retries (5 attempts over 2 hours) and was dead-lettered.

1. **Nothing is lost.** The follow-up is still pending and on the Dashboard, now marked *reminder
   didn't send*. The user simply did not get the email.
2. Read `/aws/lambda/<app>-<env>-followup-notify` for the cause; the `followup-failed` log names the
   follow-up id.
3. **Fix the cause** — the alarm means the reminder path is broken, so tomorrow's reminders will fail
   too.
4. **Prefer editing the follow-up** (which clears the failure and schedules afresh) over resending
   the stale reminder — a morning nudge delivered at lunchtime is worse than none. To resend anyway:

   ```bash
   aws lambda invoke --profile <profile> --region us-west-2 \
     --function-name <app>-<env>-followup-notify --payload fileb://payload.json /dev/stdout
   # payload.json: {"follow_up_id": …, "to_address": …, "note": …, "due_date": "YYYY-MM-DD",
   #                "contact_name": …, "opportunity_title": …}
   ```

   The dead-letter message shape has not yet been observed from a real failure;
   `handlers/followup_failed.extract_follow_up_id` logs the raw body at ERROR if it finds no id —
   check that line before trusting the parser.

---

## 4. Common tasks

- **A migration failed.** The deploy fails and `schema_migrations` holds a `failed` row with the
  error and how many statements succeeded. Fix the SQL, check by hand that re-running the file is
  safe from where it stopped, delete that version's row, and redeploy. **Never edit a migration that
  has been applied** — the runner refuses a checksum mismatch.
- **Invite a user.** Cognito is invite-only. Create the user with the console or
  `admin-create-user`; the app's `users` row is created on their first signed-in request. A user who
  has never set a password cannot use *Forgot password* (Cognito sends nothing), so setting a
  temporary password directly is the reliable route.
- **Read a production value.** Production data is reached only through the shared instance's own
  access path (see `OPERATIONS.local.md`); nothing in this repository connects to it.

---

## 5. A new environment

CDK cannot do everything. For a new environment, in order:

1. Deploy `<app>-<env>-Messaging` **before** `<app>-<env>-Api` — Messaging creates the IMAP secret.
2. Write the secret value (it is created empty).
3. Deploy the rest; migrations run from `0001` on an empty schema.
4. **Confirm the SNS email subscription** — until the link is clicked, every alarm is silent.
5. Invite the user (§4). Until someone signs in, the poller logs `status=no_user` each minute; that is
   expected.
6. Decide which **one** environment polls the mailbox (`pollEnabled` in `config.ts`). Two pollers race
   over the Import folder.

---

## 6. Known hazards

- **Sandbox can email real people.** Polling is limited to one environment, but sending is not:
  sandbox sends from the same address, so a follow-up reminder or an email sent from sandbox reaches
  a real recipient. Use disposable contacts in sandbox, and when testing by hand, block writes in the
  browser rather than trusting yourself not to click Send. See [BACKLOG.md](BACKLOG.md).
- **The database instance is shared** with other applications. Never change its parameter group or
  size from this project; the app sets what it needs per session.
- **A CDK warning logged as ERROR** — CDK's own custom-resource handler (Node.js) emits a
  `url.parse()` deprecation warning (`DEP0169`) on every deploy. It is harmless and not this
  project's code.