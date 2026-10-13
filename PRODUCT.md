# Speaker Tracker — Product

What the app does and the rules it follows, as built. For how it is built see
[ARCHITECTURE.md](ARCHITECTURE.md); for the schema, [DATABASE.md](DATABASE.md); for why it is this
way, [DECISIONS.md](DECISIONS.md); for what is not built yet, [BACKLOG.md](BACKLOG.md).

**Contents** — [1. Purpose](#1-purpose) · [2. Features](#2-features) ·
[3. Rules worth knowing](#3-rules-worth-knowing) · [4. Not built](#4-not-built) ·
[5. Look and feel](#5-look-and-feel)

---

## 1. Purpose

A behind-sign-in pipeline CRM for **getting a speaker booked on stages and podcasts** — workshops,
keynotes, retreats, women's networks, expos, podcast guest spots. One user (the speaker), with a
developer who administers it. The workflow it supports:

> build a target list of venues → find a contact at each → warm introduction or templated outreach
> → follow up on a cadence → book → deliver → review the funnel.

**Scope boundary.** This app is the *gig-acquisition* side of the business. A separate system owns
*listener conversion* — the conversations, calls and downloads an audience member goes on to have.
Speaker Tracker only **counts** what each delivered gig generated (response counters); who responded
and when lives in that other system.

**Goals and measurement are the point.** Targets and the funnel exist because tracking outreach
effort and conversion was the weakest part of the speaker's process before the app.

---

## 2. Features

### Venues and contacts

- **Venues** (organizations) — retreat venues, resorts, studios, spas, women's networks, podcasts,
  expos, corporate. Type, location, website, email domain, notes, and three **research** fields:
  *what it is*, *why it fits*, *how to approach*. The venue list shows the first line of *why it
  fits* as a scan column.
- **Research-readiness.** A venue is *research-ready* when all three research fields are filled and
  it has at least one contact. That is the quality bar behind the *venues researched* target, which
  counts venues the month they first became ready.
- **Contacts** — the person, with warmth tier, source, how you know them and notes. **A person can
  be affiliated with several venues**, with a title, a *primary contact* flag and a *power partner*
  flag **per venue**. Adding someone who already exists offers to add an affiliation instead of a
  duplicate.
- A contact page shows affiliations, an **activity timeline** (outreach touches, plus notes and
  stage changes on the contact's gigs), follow-ups and appointments.

### The pipeline

- **Kanban board** — one card per gig: `Researching → Outreach Sent → In Conversation → Pitched →
  Booked → Delivered`, moved by drag and drop. Every move is journaled; the journal drives the funnel
  and the targets.
- **Opportunity detail** — venue, talk, format, event date, money, the people on the gig (with a role
  such as introducer or coordinator, and a *lead on this gig* flag), dated notes, the lifecycle, and
  response counters.
- **Closing.** A gig closes as **Delivered** (and paid — below), **Cancelled** (was booked, then fell
  through — still counts as a booking) or **Lost / Passed** (did not convert). Closing asks for a
  reason. Closed gigs leave the board for **History**; *Show closed* peeks at recently closed cards.
- **Money.** Each gig records a comp type (**Paid / Pro bono / Trade**), a fee and currency, a payment
  status (Unbilled → Invoiced → Partial → Paid, or N/A) and a paid-on date. **Pro bono is
  first-class** and counts as a booking. **A delivered gig stays on the board until it is paid**, so
  it is not lost before it is collected.
- **History** — closed gigs in a filterable table (outcome, comp, year), a collected total, and CSV
  export.
- **Response counters** — on a delivered gig, how many audience members took each call to action
  (for example a discovery call or a booklet request): a `+`/`−` per type and a total. They form the
  funnel's bottom row. **Counts, not records**, and **not a target**.

### Outreach

- **Outreach journal** — an append-only log of **outbound** touches per contact (email, DM, call, in
  person, text), optionally tied to a gig and independent of its stage. Touches can be edited or
  deleted.
- **Outreach kind** — *initial* (first outbound touch to that contact), *follow-up* (a later
  prospecting touch) or *correspondence* (logistics on an existing conversation). The app infers
  *initial* or *correspondence* and shows it as an editable chip. **Only prospecting kinds count
  toward targets.**
- **DMs are logged, not sent.** There is no practical send API for social DMs, so the flow is: pick a
  template, its merge fields fill from the contact, copy to the clipboard, paste into the DM, log the
  touch. **Email is the only channel the app sends.**
- **Message templates** — shared templates (editable in place) and personal copies (*Duplicate*),
  each with a channel (DM or email) and a purpose (cold pitch, power-partner introduction), and merge
  fields such as `[Name]`.

### Email

The app sends and receives through the speaker's own business mailbox, which they also use from an
ordinary mail client; both see the same mail.

- **Composer** — rich text, attachments (a new file, or a copy of a library material), the default
  signature added to new drafts, To/Cc up to 50 recipients. A send is logged as an outreach touch
  with its inferred kind, and the sent message appears in the mailbox's Sent folder.
- **Inbox** — threads with who wrote last (sent / received) and unread state, searchable, filterable
  by contact. It lists **open** threads; *Show closed* includes closed ones, and searching or
  filtering by contact always does.
- **Thread view** — the whole conversation with attachments, and an **inline reply** carrying the
  same attachments and follow-up option as the composer.
- **Which mail the app sees.** Only (a) correspondence with tracked contacts, including replies that
  continue a thread the app already has, and (b) messages the speaker moves into the mailbox folder
  `Speaker Tracker/Import`. Nothing else in the mailbox is read into the app.
- **Importing a new sender.** Moving a message into `Speaker Tracker/Import` brings it in; the app
  shows *"N emails awaiting import"*, and *Add contact* opens prefilled from the sender. The message
  then moves itself to `Speaker Tracker/Processed`. The app creates both folders.
- **Received mail is shown safely.** Remote images are held back — loading them tells the sender the
  message was opened — until allowed for that message, or always for that sender.
- **Signatures** — styled HTML, several allowed, one default.

### Follow-ups and appointments

- **Follow-ups** — a reminder with an explicit **calendar date** and a free-form note, on a contact, a
  gig or both. Created on their own, or as an **opt-in rider** when logging outreach, sending email,
  replying or creating a gig (off by default, so sending never silently schedules anything).
- **Reminder email** — optional per follow-up: **one email on the due date**, in the morning. It does
  not complete the follow-up, which stays on the Dashboard until marked done. If the email fails after
  its retries, the follow-up shows *reminder didn't send*.
- **Appointments** — a logged meeting with a contact: title, date and time, details. **Not
  calendaring**: nothing syncs, invites or emails. Upcoming ones join the Dashboard's *Coming up*.

### Targets and the Dashboard

- **Targets** — a goal per type and cadence (weekly, monthly or quarterly): **venues researched,
  outreaches, pitches, bookings**. Activity-based on purpose; there is no money target.
- **Dashboard**:
  - **Target tiles** — actual against goal for the period, with a week picker that moves the tiles
    only. Each tile links to the list behind its number (except outreaches).
  - **Funnel** — for each stage, how many gigs ever reached it ("reached") and how many are there now,
    plus the response row.
  - **Money** — Booked, Received and Outstanding over paid gigs, and the pro bono count.
  - **Needs attention** — gigs awaiting payment, past-event gigs never booked, gigs gone quiet (no
    stage change or outreach for 14 days), venues with research incomplete, and threads awaiting a
    reply for 7 days.
  - **Coming up** — dated gigs and appointments. **Follow-ups due** — due and overdue follow-ups.
- Every count on the Dashboard links to the filtered list it counts, and that list shows the same
  number.

### Talks and materials

- **Talks** — the menu of talks offered: title, duration (free text), one-liner.
- **Materials** — uploaded files (one-sheets, handouts) up to 25 MB, previewable (PDF, images,
  Markdown), downloadable, and replaceable in place. Attaching one to an email copies it, so replacing
  a material never changes mail already sent.

---

## 3. Rules worth knowing

- **A gig is closed** when it is Delivered *and* paid (or N/A), or Cancelled, or Lost.
- **Money received is paid gigs whose payment status is Paid.** The fee field is also filled in on pro
  bono and trade gigs, on purpose — what a gig was worth matters for tax reporting — but that value is
  **never** counted as money booked, received or collected.
- **Cancelled counts as a booking** in the funnel and targets; it is the leak between Booked and
  Delivered.
- **Targets count entries, not states.** *Pitches* and *bookings* count gigs that **entered** that
  stage during the period; *venues researched* counts venues that **became** research-ready; only
  prospecting outreach counts.
- **The funnel counts "reached or beyond"**, so a gig that skipped a stage still counts for it.
- **Received mail never counts as outreach.** Outreach is outbound; a long logistics thread with a
  booked venue does not inflate the outreach target.
- **Threads close explicitly.** Nothing assumes a sent email is owed a reply. A thread closes when the
  speaker closes it or when its gig closes; **a reply reopens it**.
- **Nothing auto-schedules.** Follow-up riders are off by default; a reminder email is one email.
- **One email address per contact.** Mail from a contact's second address arrives as an unknown
  sender ([BACKLOG.md](BACKLOG.md)).
- **"Primary" means two different things**, labelled differently: a venue's *primary contact*, and a
  gig's *lead*.

---

## 4. Not built

Ideas that appear in early planning but are **not** in the app — listed so nobody goes looking for
them. Queued work is in [BACKLOG.md](BACKLOG.md).

- Email threads listed on a contact's or a gig's page (threads are reached from the inbox).
- A research side panel in the composer.
- Seeding a gig's *angle* from the venue's *how to approach*.
- Linking a material to a talk from the UI, renaming a material, or moving it between talks (the
  schema supports the link; materials are standalone in practice).
- Attaching an existing contact from the import card (today it only offers *Add contact*).
- A dark theme.

---

## 5. Look and feel

- A plain business app — "not ugly" — with **no gamification** (no confetti, no streaks).
- **Light theme only**, sans-serif type. The brand is carried by colour:

  | Deep navy `#1F3B4D` | Terracotta `#C2483A` | Gold `#D9A02C` | Warm cream `#FBF8F2` | Warm gray `#555555` |
  |---|---|---|---|---|
  | nav rail, headings | primary buttons, emphasis | accents, power-partner ★, warmth | page background | body text |

- The board uses the full browser width — it is worked on a large monitor.
- The original clickable mockup is in `samples/` (see `samples/README.md`). It predates the build, and
  the app differs from it in places.