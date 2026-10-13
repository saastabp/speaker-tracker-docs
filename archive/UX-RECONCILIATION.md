# Speaker Tracker — UX Reconciliation (archived)

> **Archived 2026-10-12.** The record of reconciling each page with the approved mockup. Gaps that
> were still open when it was archived were checked against the app and moved to
> [BACKLOG.md](../BACKLOG.md); nothing here is a to-do.

> **Status: COMPLETE and merged (branch `ux-reconciliation`, 2026-07-23 → 07-25). This document is
> history, not a work list.** ⚠ The DEFER/PARTIAL tags below were written when slices 6–12 were
> still unbuilt, and most named a backend the app now has (the email composer, `follow_ups`, the
> summary fields the API lacked). **Do not read a DEFER tag as outstanding work** — check the
> running app first; the slice it was waiting on has almost certainly shipped. §1 Dashboard, §2 Pipeline (board + modals), §2b Opportunity detail, §3 Venues, §4
> Contacts, §5 History, §6 Templates, §7 Targets — all shipped & browser-verified; §8 Nav was
> reconciled early via `AppShell`. Per-section banners record what shipped, what was deferred (mostly
> backend-dependent: `follow_ups`/0009, slice-6a email composer, summary fields the API lacks), and
> the few WON'T-DO decisions. App-wide: mockup radius/brand-border theme + shared `FieldLabel` /
> `FilterBar` / `detailCards` / `opportunityChips` / `venueChips` / `contactChips`. Remaining open
> items are all tagged DEFER/PARTIAL and belong to later slices.

**Source of truth:** `samples/speaker-tracker-mockup.html` (approved by the user) + the mockup-render
screenshots. Line refs below (`mL###`) point into that HTML file.

**Dispositions:**
- **FIX** — build/change to match the approved mockup.
- **KEEP** — implementation deviates on purpose (recorded decision or reasonable real-app addition);
  do **not** revert. Confirm at triage.
- **DEFER** — needs backend/surfaces from a later slice; do not half-build now.

**Method:** one page at a time as a self-contained unit, per-mutation approval, `ruff`/tests where
backend is touched. Check items off as they land.

---

## Shared / cross-boundary items (build once, reused by several pages)

- [ ] **Filter-chip + search row** — a reusable top-of-list toolbar (search box + pill filters).
  Reused by Venues, Contacts, History (and Emails in 6a). Build with the first list page (Venues).
- [ ] **Follow-up scheduling** — **DEFER.** The mockup places it in the Log Outreach modal, New
  Opportunity modal, Opportunity detail, Venue detail, and Contact detail, but it writes to
  `follow_ups`, which ships in `0008_followups.sql` (**slice 7 — not yet migrated**). Tag every
  follow-up finding DEFER; do not half-build UI against a missing table.
- [ ] **Email surfaces on detail pages** — **DEFER** to slice 6a: "Compose email" buttons and
  per-contact/per-venue Emails cards, email-thread views.
- [ ] **Talks & Materials screen + one-step attach** — **DEFER** to the Talks & Materials slice.
- [ ] **Dashboard aggregate drill-down** — **DEFER** to DEV-PLAN slice 8 (click an aggregate → the
  filtered summary list). Not in the mockup; a new feature. Prerequisite is the list filter rows
  this pass builds, so don't wire click handlers during reconciliation.

---

## 1. Dashboard  (`pages/Dashboard.tsx` vs mL382–443)

> **✅ SHIPPED & browser-verified (2026-07-23).** All FIX items done (greeting, week/tz subtitle,
> funnel conversion %, Delivered row, money sub-counts + pro-bono tile, Coming-up card, two-column
> layout, Stale demoted). KEEP: "% of goal" tile subtitle; DROP: card hint sub-headers (developer).
> Added `research_incomplete` to Needs attention (any not-research-ready venue → links to venue).
> DEFERRED: "+ Log outreach" header button (needs contact-picker), funnel-leak caveat, richer
> needs-attention timing reasons (need `follow_ups`/0009 — see [[ticklers-reminders-future]]).
> Backend: `0007_target_labels` + dashboard repo/model additions (deployed to sandbox).

**High**
- [ ] FIX — Restore personalized header "Good morning, <name>" (generic "Dashboard" title now) — mL384
- [ ] FIX — Restore week-of + timezone subtitle ("Week of Jul 13 – 19 · HST") — mL384
- [ ] FIX — Restore primary "+ Log outreach" action button in the page head — mL385
- [ ] FIX — Restore the "Coming up" schedule card (dated agenda list) — mL429–438
- [ ] FIX — Funnel: render stage-to-stage **conversion %** (e.g. 39/57/75%), not just raw counts — mL399–402
- [ ] FIX — Restore "Pro bono" as a money stat tile (w/ "visibility gigs" sub-label); currently demoted to a text line — mL412

**Medium**
- [ ] FIX — Money card title "Revenue & payments — Q3" (now "Money") — mL407
- [ ] FIX — Funnel card title "Pipeline funnel — Q3 to date" (now "Funnel — reached or beyond") — mL396
- [ ] FIX — Restore funnel-leak annotation ("1 booked gig cancelled — still counts as booked…") — mL403
- [x] DROP — Card hint sub-headers ("conversion between stages" / "paid gigs only" / "outreach + payments"): **not reproduced** (developer, 2026-07-23). Illustrative microcopy; and "paid gigs only" is wrong — the money card shows the pro-bono count. — mL396/407/418
- [ ] FIX — Stat tiles: guarantee the four signed-off metric labels (New venues researched, Outreach touches, Pitches sent, Gigs booked) — mL388–391
- [x] KEEP — Per-tile subtitle stays the neutral **"% of goal"**, not the mockup's interpretive sentences (developer, 2026-07-23): that detail belongs in Needs Attention / Coming Up, not duplicated per tile. — mL388–391
- [ ] FIX — Money tiles: restore supporting sub-counts (Outstanding "2 invoiced", Received "2 collected", Booked "3 paid gigs") — mL409–411
- [ ] KEEP? — "Stale — needs a nudge" card is an EXTRA (not in mockup); it occupies the mockup's "Coming up" slot. Decide keep-alongside vs replace at triage — impl only

**Low**
- [ ] FIX — "Needs attention" chips: restore the richer status vocabulary (Overdue pay / day-counts / Awaiting pay / "1/3") — mL421–425
- [ ] FIX — Right-column pairing: mockup pairs "Needs attention" with "Coming up" — mL416–439

---

## 2. Pipeline  (`pages/Pipeline.tsx` + `OpportunityFormModal` + `LogOutreachModal`)

> **Board: Tier 1 + Tier 2 SHIPPED & browser-verified (2026-07-23).** Tier 1: stage-color dots,
> subtitle, "+ New opportunity", footer note, kept Log-outreach button, approved payment-chip
> colors, warm-panel column background + uppercase headers, **Recently-closed column**, app-wide
> sentence-case badges. Tier 2: card leads with venue + "talk · format" line + type chip — needed
> `talk_title` + `organization_type` added to `_SUMMARY_SELECT`/model/api (deployed).
> **DEFERRED:** Tier 3 (due/overdue, research N/3, warm-intro — need `follow_ups`/0009), removing
> the free-text `title` field (revisit at the Opportunity detail page).
> **Modals: REWORKED to the mockup & browser-verified (2026-07-24).** Log Outreach: uppercase
> `FieldLabel`s throughout, Template + editable Message split (merge fills `[Name]`, "there"
> fallback, Copy-to-clipboard), picker always visible (un-gated), channel segmented (email
> excluded), Kind kept. New Opportunity: `FieldLabel`s + mockup order, **title derived from venue +
> talk** (free-text field dropped), **Starting stage / Lead contact / Payment status** now built —
> via a new backend `OpportunityCreateInput` + create-seed (atomic: one `status_events` row at the
> chosen stage, lead linked `is_primary`, non-terminal-stage guard; 59 backend tests green). Payment
> status shown only for paid comp. Shared `FieldLabel` + `BRAND_MUTED`/`BRAND_FAINT` added to
> `theme.ts`. **Still DEFERRED:** the follow-up blocks in both (need `follow_ups`/0009); event-date
> free-text coarse dates (kept the native picker — data-model).

### Board (vs mL444–490)
**High**
- [ ] FIX — Card: add the talk/offer line (e.g. "Wellness Wheel for Women · guest workshop") — mL455–475
- [ ] FIX — Card: lead with organization, de-emphasize talk (emphasis currently inverted) — mL455
- [ ] FIX — Card: restore the category/venue-type chip (Association/Expo/Resort/…) — mL455–483
- [ ] FIX — Card: restore due/follow-up indicators ("follow up Jul 19", "no reply · 9d" overdue, …) — mL456–479

**Medium**
- [ ] FIX — Card: research-progress chip ("Research 1/3") — mL455
- [ ] FIX — Card: "★ warm intro" chip — mL465
- [ ] FIX — Dedicated "Recently closed" column when Show closed is on — mL481–485
- [ ] FIX — Page subtitle / open-count ("10 open opportunities · drag cards between stages") — mL446
- [ ] FIX — Primary button label "+ New opportunity" (now "Add opportunity") — mL449
- [x] KEEP — "Log outreach" button in board header (EXTRA vs mockup) stays (developer, 2026-07-23): a wanted action even though the mockup omits it. — impl only

**Low**
- [ ] FIX — Footer note about closed gigs living in History — mL487
- [ ] (Column set/order matches well — data-driven from funnel catalog)

### New / Edit Opportunity modal (vs mL1200–1239)
**High**
- [ ] DEFER — Follow-up block (switch + date + note) — needs `follow_ups` (0009) — mL1228–1233
- [x] FIX — "Payment status" segmented control (Unbilled/Invoiced/Partial/Paid) — create-only, paid comp — mL1224–1226
- [x] FIX — "Starting stage" select (default Researching) — mL1213
- [x] FIX — "Lead contact" select — mL1214
- [x] FIX/DISCUSS — Remove free-text "Title" field; derived from venue + talk on submit (developer: derive on frontend) — mL1204

**Medium**
- [x] FIX — "Compensation" section header + Pro-bono explanatory note — mL1217/1227
- [x] FIX (remove) — "Currency" field (app is USD-only) — impl only
- [x] FIX — Comp "Type" as segmented control (Paid/Pro bono/Trade) vs dropdown — mL1219–1221
- [x] FIX — Footer hint "Starts in Researching — drag it across the board…" — mL1235

**Low**
- [x] FIX — Labels: "Venue / organization", "Talk / offer", "Angle for this gig" — mL1204/1206/1216
- [x] FIX — Field pairing/order (Talk+Event date; Format standalone segmented) — mL1205–1211
- [ ] DEFER — Event date free-text "e.g. Oct 2026" (coarse dates) vs native picker — kept native picker (data-model) — mL1207

### Log Outreach modal (vs mL1106–1132)
**High**
- [ ] DEFER — Follow-up block — needs `follow_ups` (0009) — mL1122–1127

**Medium**
- [x] KEEP — Email channel removed (owned by composer; slice-4 decision) — mL1112
- [x] KEEP — "Kind" segmented control (initial/correspondence; contact-scoped inference; slice-4) — impl only
- [x] FIX — Submit button "Log touch" — mL1129
- [ ] FIX — Footer hint "Counts toward this week's 8-touch target" — kept generic "…this week's outreach target" (avoid hardcoding the count) — mL1129

**Low**
- [x] FIX — Labels "Opportunity" / "Date" — mL1119/1118
- [x] FIX — Channel as segmented control vs dropdown — mL1111–1113
- [x] TODO — `TemplatePicker` split into Template select + editable merged Message textarea + copy-to-clipboard — mL1114–1116

---

## 2b. Opportunity detail  (`pages/OpportunityDetail.tsx` + `CloseOpportunityModal`)

> **✅ SHIPPED & browser-verified (2026-07-24).** Rebuilt to the mockup (mL490–560): breadcrumb,
> header chip row (stage · comp · payment · date), two-column grid — left: Details (key-value card
> with stage chip) / Payment (editable, kept) / Notes (add-at-top) / Lifecycle; right: Venue card
> (avatar + type·location via `useOrganization`, no backend change) / On-this-gig (avatars + Lead
> chip). Close modal restyled (FieldLabels, defaults-by-stage explainer, footer hint). Shared
> `opportunityChips.ts` (stage/payment colours + money) extracted from Pipeline.
> **DEFERRED:** Next-follow-up card (needs `follow_ups`/0009); Close modal reason-catalog + date
> (backend). **KEEP:** editable Payment card + Delete button (real-app needs beyond the mockup).

## App-wide (this session)

> **Theme radii + brand borders (`theme.ts`):** the app was near-square (Mantine ~4px default). Added
> the mockup radius scale — modal 14px, cards 12px, inputs/segmented 9px, chips/avatars pill — plus
> warm brand-line (`#E7DCC9`) input borders. Applies to every modal/card/input.
> **Optional Title reinstated:** the mockup dropped the free-text title, which left board cards
> showing only the venue. Re-added an optional Title (validation: title *or* talk required; blank →
> talk name); card sub-line is now `<format>: <title>`. Frontend-only (title column stays NOT NULL).

---

## 3. Venues  (`pages/Venues.tsx` + `VenueDetail.tsx` + `VenueFormModal`)

> **✅ List + detail + modal SHIPPED & browser-verified (2026-07-24), frontend-only.** Shared
> `FilterBar` (search + pill filters — reused by Contacts/History). List: "Venues & Organizations"
> heading + count, search + type/Ready pills, colour type chips, why-it-fits, Ready dot, Location
> demoted to org subtitle. Detail: breadcrumb, header (type chip + website + ready), two-column grid
> — Research–Kindling / Opportunities (client-side filter of `useOpportunities` by org) / Contacts
> (kept `AffiliationRow`) / Details (Type/Location/Added); Log-touch opens the outreach modal.
> Modal: FieldLabels, mockup field order, Research–Kindling divider, kind-hint callout, "Save,
> finish research later". Promoted shared `components/detailCards.tsx` + `venueChips.ts`.
> **DEFERRED (backend):** Last-touch + research fraction (venue summary lacks both); Activity panel
> (no org-scoped timeline); Source field (column exists in 0002, not in model/API); warmth chips +
> on-this-gig contacts + Compose email (6a) + Next-follow-up (0009). **KEEP:** editable
> `AffiliationRow`, Delete, Log-outreach + Email-domain + Notes (impl extras).

### List (vs mL652–674)
**High**
- [x] FIX — Search box ("Search venues…") — mL658  *(shared `FilterBar`)*
- [x] FIX — Filter pills (data-driven type pills present in the list + Ready only) — mL659  *(shared)*
- [ ] DEFER — "Last touch" column — venue summary has no last-outreach timestamp (backend) — mL662

**Medium**
- [ ] DEFER — Research column partial fraction (2/3) — summary has only `research_ready` bool (backend) — mL668–671
- [x] FIX — Organization cell subtitle line (location) — mL664
- [x] FIX — Type as color-coded chip vs plain text — mL664–671

**Low**
- [x] FIX — Column header "Organization" (was "Name") — mL662
- [x] FIX — Page heading "Venues & Organizations" + count subhead — mL654
- [x] FIX — Column order (Organization, Type, Why it fits, Research, Contacts) — mL662
- [x] FIX (remove) — "Location" column (now the org subtitle) — impl only
- [x] KEEP — "Log outreach" button in list header — impl only
- [x] FIX — Add button "+ Add venue" — mL655

### Detail (vs mL677–742)
**High**
- [x] FIX — "Opportunities" panel (opp row + stage chip; on-this-gig contact chips DEFERRED — need opp detail) — mL691–705
- [ ] DEFER — "Activity" panel — no org-scoped timeline endpoint (backend) — mL706–714
- [ ] DEFER — "Compose email" action (6a) — mL681
- [x] FIX — "Log touch" action (opens `LogOutreachModal`) — mL681

**Medium**
- [ ] DEFER — "Next follow-up" panel (needs `follow_ups`/0009) — mL724–730
- [x] FIX — "Details" panel (Type / Location / Added; Source DEFERRED — backend) — mL731–739
- [x] FIX — Research card header "Edit" affordance — mL686
- [x] FIX — Research card title "Research — Kindling" — mL686
- [ ] PARTIAL — Contacts panel: power-partner star kept (`AffiliationRow`); warmth chips DEFERRED (affiliation has no warmth — backend) — mL720–721

**Low**
- [x] FIX — Header: colored type chip + website + Outreach-ready inline (prose description omitted — no field) — mL680
- [x] FIX — Ready badge label "Outreach-ready" — mL680
- [x] FIX — Breadcrumb "Venues & Orgs › <name>" — mL678
- [x] KEEP — "Delete" button (reasonable real-app addition) — impl only

### Modal (vs mL1134–1155)
**High**
- [ ] DEFER — "Source" field — column exists (0002) but not in model/API; needs backend + deploy — mL1145
- [x] FIX — Research-readiness hint callout (static copy) — mL1151

**Medium**
- [x] FIX — "Research — Kindling" section divider before the three fields — mL1147
- [x] KEEP — "Notes" field (EXTRA) — impl only
- [x] KEEP — "Email domain" field (EXTRA) — impl only

**Low**
- [x] FIX — Name label "Organization name" — mL1138
- [x] FIX — Website "optional" tag — mL1144
- [x] FIX — Field order (Name; Type/Location; Website/Email-domain — Source deferred) — mL1138–1146
- [x] FIX — Secondary footer action "Save, finish research later" — mL1153

*(Kindling fields What it is / Why it fits / How to approach are present in both — OK.)*

---

## 4. Contacts  (`pages/Contacts.tsx` + `ContactDetail.tsx` + `ContactFormModal`)

> **✅ List + detail + modal SHIPPED & browser-verified (2026-07-24), frontend-only.** List:
> "Contacts" + "N people · M power partners", `FilterBar` (search + Everyone/Power-partners), ★ +
> "Power partner" sub-label, colour warmth chips, Venues count, Email column dropped, row-click.
> Detail: breadcrumb, header (★ + power-partner/warmth chips), two-column grid — Affiliations (kept
> `AffiliationRow`, "+ Add affiliation" toggles the form) / Activity (existing contact timeline) /
> Notes + Reach (email/phone) / Relationship (power-partner + warmth explanations) / Details (Warm
> intro / Source / Added). Modal: FieldLabels, Warmth → segmented, how-you-know relabeled "Warm
> intro / mutual connection", kept dedupe + Source. New `contactChips.ts` (`warmthColor`).
> **DEFERRED:** Role·Org + Source + Last-touch columns + Needs-follow-up pill (summary/`follow_ups`);
> Opportunities-across-orgs (no contact→opps); Emails + Compose (6a); LinkedIn/Instagram (no
> columns); modal org-attach + power-partner toggle + dedicated dedupe-search. **KEEP:** editable
> `AffiliationRow`, Delete, Source field, name-triggered dedupe hints.

### List (vs mL745–767)
**High**
- [x] FIX — Search box — mL751  *(shared `FilterBar`)*
- [x] FIX — Filter pills (Everyone / Power partners; Needs-follow-up DEFERRED — `follow_ups`) — mL752
- [ ] DEFER — "Role · Organization" column w/ "+N orgs" overflow — summary has only org count (backend) — mL755/759

**Medium**
- [ ] DEFER — "Source" column — not in the contact summary (backend) — mL755
- [ ] DEFER — "Last touch" column — not in the contact summary (backend) — mL755
- [ ] DEFER — "Next follow-up" column (needs `follow_ups`/0009) — mL755/761/763

**Low**
- [x] FIX (remove) — "Email" column (not in mockup) — impl only
- [x] FIX — Header subtitle "N people · M power partners" — mL747
- [x] FIX — Power-partner "Power partner" sub-label under name — mL757/759
- [x] FIX — Add button "+ Add contact" — mL748

### Detail (vs mL769–847)
**High**
- [ ] DEFER — "Opportunities across orgs" card — no contact→opps list (backend) — mL785–797
- [x] FIX — "Relationship" card (power-partner marker + warmth explanation) — mL823–829
- [ ] DEFER — "Compose email" primary action (6a) — mL773

**Medium**
- [ ] DEFER — "Emails" card (per-contact thread list) (6a) — mL798–803
- [ ] DEFER — "Next follow-up" card (needs `follow_ups`/0009) — mL830–836
- [x] FIX — Power-partner marker (★) on header + power-partner/warmth chips — mL772
- [ ] PARTIAL — "Reach" block: Email + Phone done; LinkedIn + Instagram DEFERRED (no columns — backend) — mL816–821

**Low**
- [x] FIX — Details card "Added" date (+ Source, Warm intro) — mL838–843
- [x] FIX — Action "Log outreach" (opens `LogOutreachModal`, contact preselected) — mL773
- [x] KEEP — Edit/Delete header buttons (reasonable) — impl only
- [x] FIX — Affiliations "+ Add affiliation" button (progressive disclosure) — mL778

### Modal (vs mL1158–1197)
**High**
- [ ] DEFER — Power-partner toggle (★) + description — power-partner is per-affiliation, set on detail — mL1188–1191
- [ ] DEFER — Multi-hat org attach ("Add this org to her") — affiliations added on the detail page — mL1163–1170
- [ ] DEFER — "Organization" select + "Role / title at this org" — org-attach on create (backend/flow) — mL1174/1176

**Medium**
- [x] FIX — "Warm intro / mutual connection" field (relabeled `how_you_know`) — mL1192
- [x] FIX — Warmth as Cold/Lukewarm/Warm segmented control vs dropdown — mL1185–1187
- [ ] KEEP — Dedupe as name-triggered hint (dedicated "find first" search DEFERRED — UX rework) — mL1162

**Low**
- [ ] DEFER — LinkedIn + Instagram fields — no columns (backend) — mL1182–1183
- [x] KEEP — "Source" text field (EXTRA in modal; mockup has it on detail) — impl only
- [ ] N/A — "…or create a new person" divider — moot without the find-first/create split — mL1171
- [x] FIX — Submit button "Add contact" — mL1195

---

## 5. History  (`pages/History.tsx` + history-detail)

> **✅ List + read-only detail SHIPPED & browser-verified (2026-07-25), frontend-only.** List
> (`History.tsx`): "History" + stat line (closed/delivered/cancelled/lost · $ collected · pro bono),
> `FilterBar` (search + outcome + comp + dynamic-year pills), columns (Gig w/ talk·format sub /
> Outcome / Date=event / Format / Comp / Amount / Payment — coloured chips), client-side Export CSV,
> row-click → detail. Venue column dropped. Detail = **read-only mode of `OpportunityDetail`** when
> `closed_at` set: hides Edit/Close/Delete + the editable Payment card, adds "· read-only record" +
> read-only Payment/Paid-on rows in Details. **DEFERRED (backend):** Duplicate + Reopen actions,
> Invoiced #/date fields; dedicated read-only record component (chose read-only mode instead). NOTE:
> Notes + Link-contact stay editable on closed gigs (scope was Edit/Delete/Payment only).

### List (vs mL562–585)
**High**
- [x] FIX — Summary stat line (closed · delivered · cancelled · lost · $ collected · pro bono) — mL564
- [x] FIX — Outcome/comp/year filter pills (All outcomes/Delivered/Cancelled/Lost/Paid/Pro bono + dynamic years) — mL569  *(shared `FilterBar`)*
- [x] FIX — Search box ("Search closed gigs…") — mL568

**Medium**
- [x] FIX — "Export CSV" button (client-side) — mL565
- [x] FIX — "Format" column — mL572
- [x] FIX — "Comp" column (Paid/Pro bono/Trade chip) — mL572
- [x] FIX — "Date" column = event date — mL572/574

**Low**
- [x] FIX — Column "Gig" + two-line cell w/ "Talk · format" sub-line — mL572/574
- [x] FIX — Column "Amount" — mL572
- [x] FIX — Column "Payment" (chip) — mL572
- [x] FIX (remove) — "Venue" column (not in mockup) — impl only

### Detail (read-only closed-gig record — vs mL588–649)
**High**
- [x] FIX/DISCUSS — Read-only record via a **read-only mode of `OpportunityDetail`** (developer's call) — hides Edit/Close/Delete + editable Payment when `closed_at` set — mL591–592

**Medium**
- [x] FIX — Header status/payment chip row (Delivered / comp / payment) + "· read-only record" — mL591
- [ ] DEFER — "Duplicate" and "Reopen" buttons (Reopen needs backend) — mL592
- [ ] PARTIAL — "Record": Closed-date shown (Payment/Paid-on read-only in Details); Source DEFERRED (backend) — mL640–646
- [ ] DEFER — "Invoiced" field (invoice date + number) — no columns (backend) — mL612

**Low**
- [x] KEEP — Money heading (read-only Payment rows in Details; editable "Payment" card only for open gigs) — mL607
- [ ] DEFER — "Paid on" payment method ("· check") — no column (backend) — mL613
- [x] KEEP — "Venue" + "On this gig" cards (from the Opportunity-detail rebuild) vs mockup's combined "Venue & contact" — mL628–632
- [ ] PARTIAL — "Outcome notes" — `outcome` shows in Details when present; no dedicated card — mL635–638

---

## 6. Templates  (`pages/Templates.tsx` + `TemplateFormModal`)

> **✅ Page + modal SHIPPED & browser-verified (2026-07-25), frontend-only.** Page: rebuilt from a
> data table to a responsive **card grid** (`SimpleGrid` 1→2→3 cols) — each card = name + scope chip
> (Shared / "Your copy"), a `kind · channel` meta line, a warm body-preview box (line-clamped), and
> Edit / Duplicate / Delete (Delete on personal only). "Message Templates" heading + mockup subhead +
> "+ New template". Modal: FieldLabels, kind-hint callout (`[Name]` note + shared-template warning),
> "Duplicate as my copy" footer action (shared only, wired to `useDuplicateTemplate`), "Save
> template", "Shared template · editable in place" hint. **DEFERRED (no API fields):** per-card
> audience/description paragraph + "Used N× · last …" usage metadata. **KEEP:** split Purpose+Channel
> selects + Subject field.

### Page (vs mL989–1023)
**High**
- [x] FIX — Rebuild as **card grid** (rich per-template card) vs data table — mL994–1022
- [x] FIX — Per-template body / merge-field preview — mL999/1008/1017
- [ ] DEFER — Per-template audience/description paragraph — no field (backend); show `kind · channel` meta instead — mL998/1007/1016

**Medium**
- [ ] DEFER — Per-template usage metadata ("Used 7 times · last …") — no field (backend) — mL1000/1009/1018
- [x] KEEP — Delete action (personal templates only) — reasonable — impl only
- [x] FIX — Scope chip "Your copy" (was "Personal") — mL1005

**Low**
- [x] FIX — Heading "Message Templates" — mL991
- [x] FIX — Button "+ New template" — mL992
- [x] FIX — Subhead wording — mL991
- [x] FIX — Edit/Duplicate as labeled text buttons vs icon-only — mL1001
- [x] KEEP — Explicit Purpose/Channel (now the card meta line; splits kept in the modal) — impl only

### Modal (vs mL1280–1291)
**High**
- [x] FIX — "Duplicate as my copy" action in footer (shared templates) — mL1291
- [x] FIX — `kind-hint` block (merge-field note + shared-template warning) — mL1289

**Medium**
- [x] KEEP — Split Purpose + Channel selects vs mockup's single "Use for" (matches the channel/purpose data model) — mL1286
- [x] KEEP — "Subject" field (EXTRA; needed for email templates) — impl only

**Low**
- [x] FIX — Footer status hint "Shared template · editable in place" — mL1291
- [x] FIX — Save button "Save template" — mL1291
- [x] FIX — Body helper wording — mL1288
- [x] KEEP — Footer "Cancel" button — impl only

---

## 7. Targets  (`pages/Targets.tsx` vs mL1026–1068)

> **✅ Visual polish SHIPPED & browser-verified (2026-07-25), frontend-only.** The mockup's
> progress-list rebuild was **REJECTED by the developer** (2026-07-25): Targets is the *setting* surface only
> — actual-vs-goal **progress already lives on the Dashboard**, so a per-row meter would duplicate it;
> and **"+ New target" isn't supportable** (targets are a fixed catalog, each tied to a specific
> Dashboard aggregation — you can't add an arbitrary one without building the backend rollup). Kept
> the target×cadence editable matrix (functionally fine); polished it to the app standard: wrapped in
> a rounded `Card`, uppercase/tracked column headers, navy target labels, setting-focused subtitle
> ("…Progress is tracked on the Dashboard"). Inputs inherit the brand theme.

**High**
- [x] WON'T-DO — Progress-tracking-list rebuild — duplicates the Dashboard; Targets is setting-only (developer) — mL1031–1068
- [x] WON'T-DO — Actual-vs-target progress meter per row — on the Dashboard already — mL1036–1064
- [x] WON'T-DO — "Actual / Goal" numeric column — on the Dashboard already — mL1037–1065

**Medium**
- [x] WON'T-DO — "+ New target" button — targets are a fixed catalog tied to backend aggregations — mL1029
- [x] KEEP — Inline-editable cells (save on blur) instead of a per-row Edit button — impl only
- [x] KEEP — Three editable cadence columns (the setting matrix) vs a single cadence chip — mL1035–1063
- [x] KEEP — Setting-focused subtitle instead of the mockup's two-hours-a-week framing — mL1028

**Low**
- [ ] PARTIAL — Two-line target labels — names wrap; no separate helper copy (no field) — mL1034–1062
- [x] FIX — Column headers polished (Target / Weekly / Monthly / Quarterly, uppercase) — mL1032

---

## 8. Nav / chrome  (`components/AppShell.tsx` + `main.tsx` vs mL340–376)

**High**
- [ ] FIX — Group/section headings: Relationships / Outreach / Growth (now flat) — mL356/361/366
- [ ] DEFER — "Compose" nav item + `/composer` route (6a) — mL364

**Medium**
- [ ] FIX — Nav label "Venues & Orgs" (now "Venues") — mL357
- [ ] DEFER — Nav label "Talks & Materials" (now "Talks") — becomes real with the Talks slice — mL368
- [ ] FIX — Nav order (Dashboard/Pipeline/History cluster; then Relationships; then Outreach; then Growth) — mL351–368
- [ ] FIX — 360 Balanced Living logo in the sidebar (text-only now) — mL346–348
- [ ] DEFER — Emails/Talks currently route to Placeholder — resolve as those slices land — main.tsx wildcard

**Low**
- [ ] KEEP — Header auth controls (email + Sign in/out) — real-world chrome, not in the static mockup — impl only