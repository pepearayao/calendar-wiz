# Existing Tools Survey: Calendar Aggregation + Booking Links

## Scope & date

Survey date: **2026-08-22**. This document surveys how existing products (a) aggregate
multiple personal calendars into one unified view and (b) expose shareable booking/scheduling
links. It is a design input for a self-hosted Kotlin calendar hub that must aggregate
**2 Gmail + 1 Proton + 1 Microsoft/Teams** calendar, support **read + write**, and expose a
booking link. Findings come from primary sources (official product docs, help centers, API
docs, source repos, and RFC 4791 CalDAV / RFC 5545 iCalendar). Non-obvious claims are cited
inline; anything not confirmable from a primary source is marked **(unverified)**.

## Executive summary

No mainstream consumer tool aggregates **Proton Calendar** through an API or CalDAV — Proton
publishes no CalDAV endpoint and no supported public Calendar API, and Proton Mail Bridge
exposes mail only. The only interop is a **read-only ICS share link out** (paid plans) and a
**view-only ICS subscription in**; nothing writes events into Proton programmatically. So for a
DIY hub, Proton is **busy/display input only, never a booking-write destination**. Among
aggregators, **Morgen** is the most connective consumer app (Google, Microsoft 365, iCloud,
Fastmail, generic CalDAV, ICS) with full read-write and RSVP; **Cronofy** is the reference
*aggregation API* (one API over Google/Microsoft/Exchange/iCloud with an Availability engine);
**cal.com** is the reference *self-hostable booking layer*. The dominant booking pattern is
consistent across cal.com, Calendly, and Cronofy: **read free/busy across ALL connected
calendars to compute availability, but write the confirmed event to exactly ONE chosen
"destination" calendar.** Google sync tokens and Microsoft Graph delta queries make a **local
unified store** feasible for the 2 Gmail + 1 Microsoft accounts, rather than re-querying live.

## Comparison table

| Tool | Type | Account types connected | Read / Write (incl. RSVP) | Unified view | Booking link (+ where event is written) | Self-host / OSS + stack | Proton? |
|---|---|---|---|---|---|---|---|
| **Morgen** | Consumer app | Google, Microsoft 365 / Exchange Online (Graph), Apple iCloud, Fastmail, generic CalDAV, ICS feeds | Read-write; RSVP yes (accept/decline/tentative, RSVP inbox) | Yes — single combined view, per-source colours, day/week, drag across accounts | Yes (Scheduling Links, Booking Page, Poll); event written to a designated connected calendar | Proprietary SaaS; not self-hostable; stack undisclosed | No native; read-only ICS only (unverified) |
| **Notion Calendar** (ex-Cron) | Consumer app | Google, Microsoft Outlook / 365, Apple iCloud. No CalDAV | Read-write; RSVP yes (set status + note) | Yes — sidebar accounts, per-calendar colours, auto-merge duplicates, day/week | Yes (scheduling links; daily windows, min/max booking window, expiration); destination account not stated (unverified) | Proprietary (Notion Labs); not self-hostable | No |
| **Amie** | Consumer app | Google, Apple iCloud, Microsoft Outlook / 365 (added Aug 2026). No CalDAV | Read-write; RSVP not documented (unverified) | Yes — combined multi-calendar view (colour details unverified) | Yes (scheduling links, co-hosts, `.me` domain; AI todo-scheduling); destination not stated (unverified) | Proprietary; not self-hostable; "zero server storage" claim | No |
| **Vimcal** | Consumer app | Google + Microsoft Outlook / 365 **only** (no iCloud/CalDAV/Proton documented) | Read-write; RSVP not documented (unverified) | Yes — Google+Outlook unified, auto-merge, "Time Travel" timezone overlay | Yes (Booking Links, Slots, Polls); event to connected Google/Outlook; per-link calendar selector unverified | Proprietary SaaS; not self-hostable | No native; only via ICS-into-Google (inference) |
| **Rise** | **DISCONTINUED** | n/a | n/a | n/a | n/a | Was SaaS; dead | n/a |
| **cal.com / cal.diy** | Scheduling layer over calendars | Google, Outlook/365 (Graph), Apple/iCloud (CalDAV), generic CalDAV (Beta), Exchange (EWS), Zoho, Lark, Vimcal, read-only ICS feed | Reads free/busy from ALL; writes booking to ONE calendar; booker added as attendee. RSVP read-back (unverified) | No true unified grid; read-only "Overlay my calendar" on booking page | Yes — public link per event type; conflicts across all connected calendars; event written to per-event **destination calendar** | **Self-host OSS, MIT** (repo `calcom/cal.com` renamed to `calcom/cal.diy`, relicensed from AGPL-3.0 in 2026 — confirmed via GitHub API, 47.8k★); Next.js/React/TS/tRPC/Prisma/Postgres | No |
| **Calendly** | Scheduling app (SaaS) | Google, Office 365 / Outlook.com, Microsoft Exchange; iCloud (existing connections only, closed to new since 2024-08-20) | Reads free/busy across 1–6 accounts (plan-dependent); writes booking to ONE "Add to calendar" account. RSVP behaviour unverified | No combined view | Yes — windows, buffers, min notice, per-day/week/month limits, date range, TZ auto/lock; event → the one main calendar | Proprietary SaaS; not self-hostable | Not listed |
| **Cronofy** | Aggregation **API** (SaaS) | Google, Microsoft 365, on-prem Exchange, Outlook.com, Apple iCloud (no generic CalDAV, no Proton) | Read events + free/busy + create/update/delete + RSVP (`change_participation_status`) | API-level unified model (not a visual UI) | Availability API + Real-Time Scheduling; RTS writes event into `target_calendars`, app notified via `callback_url` | No self-host (SaaS); OSS SDKs (Ruby/.NET/PHP/Node/Python) | Not listed |
| **Baikal** | CalDAV/CardDAV **server** (destination store) | Serves CalDAV / CardDAV / WebDAV-sync; connects to NO external provider | Full CalDAV read-write for clients that sync to it | n/a (storage server, not a viewer) | No | **Self-host OSS**; PHP + sabre/dav + SQLite/MySQL/Postgres; GPLv3 | n/a (aggregates nothing) |
| **Radicale** | CalDAV/CardDAV **server** (destination store) | Serves CalDAV / CardDAV / WebDAV; connects to NO external provider | Full CalDAV read-write for clients that sync to it | n/a (storage server) | No | **Self-host OSS**; Python 3.9+, files-on-disk; GPLv3 | n/a (aggregates nothing) |
| **Proton Calendar** | Consumer app / walled garden | Own account only; **no CalDAV, no public API**; ICS over HTTPS both directions | Display-only to outsiders: read-only ICS out, view-only ICS in; **no programmatic create/edit/RSVP in** | n/a | Cannot be a booking-write target (except emailed iCal invite a human accepts) | Not self-hostable; web clients OSS (GPLv3, React/TS) | **This IS the Proton answer** |

---

## Per-tool findings

### Morgen
Most connective consumer app: connects Google, Microsoft 365 / Exchange Online (via Graph — not
on-prem Exchange), Apple iCloud, Fastmail, near-any generic CalDAV server, and ICS subscription
feeds ([integrations](https://www.morgen.so/integrations)). Full read-write with real two-way
sync and genuine RSVP — accept/decline/tentative plus an RSVP inbox on desktop and mobile
([changelog](https://changelog.morgen.so/rescheduling-assistant-rsvp-inbox-more-308543)). Presents
a single unified view where each account is a colour and you drag events between source calendars.
Mature scheduling suite — Scheduling Links, Open Invites, hosted Booking Page, Poll — with buffers
and automatic timezones; a booked event lands in a designated connected calendar
([scheduling links](https://www.morgen.so/scheduling-links),
[FAQ](https://www.morgen.so/faq)). Proprietary cloud product, not self-hostable, but ships
Win/Mac/Linux desktop, iOS/Android, web, and a public developer API
([docs](https://docs.morgen.so/)). No native Proton; only a read-only ICS subscription would work
(unverified).

### Notion Calendar (formerly Cron)
Connects Google, Microsoft Outlook / 365, and Apple iCloud — **no CalDAV, no Proton**
([connections](https://www.notion.com/help/notion-calendar-connections)). Freshness note:
Microsoft support, absent at the original Cron launch, is now live. Creates and edits events
(event, focus time, OOO, birthday) and supports RSVP with a note via right-click / mobile select
([manage events](https://www.notion.com/help/manage-your-calendars-and-events)). Unified view is a
colour-coded sidebar of all accounts with auto-merge of duplicate events. Scheduling links expose
daily availability windows, a min/max booking window, and expiration; a booked meeting
"automatically appears on your calendar" but the exact destination account is not stated
(unverified) ([availability](https://www.notion.com/help/availability-blocking-and-time-zones)).
Proprietary (Notion Labs), not self-hostable, on Mac/Windows/iOS/web.

### Amie
Live at [amie.so](https://amie.so/) but repositioned as an AI "note taker" / agent workday suite
with the calendar as a core feature. Connects Google, Apple iCloud, and — per its
[Aug 2026 changelog](https://amie.so/changelog) — Microsoft Outlook / 365 as first-class two-way
sync (the [pricing page](https://amie.so/pricing) is stale and lists only Google + Apple). No
CalDAV, no Proton. Read-write with two-way sync; RSVP support is not documented (unverified).
Booking links support multi-calendar availability, co-hosts (joint availability), a `.me` booking
domain, and auto-confirmations, plus AI scheduling that auto-places todos into gaps
([AI scheduling](https://amie.so/documentation/features/ai-scheduling)). Event-write destination
not stated (unverified). Proprietary; not self-hostable; claims calendar data is proxied, not
stored server-side.

### Vimcal
Live paid app (web, Mac/PC, iOS, Chrome extension). Its pricing, help center, and App Store
listing all state it connects **Google Calendar and Microsoft Outlook / 365 only** — no iCloud,
CalDAV, or Proton documented ([account setup](https://help.vimcal.com/getting-started/account-setup),
[pricing](https://www.vimcal.com/pricing)). This is absence of mention, not an explicit denial.
Full read-write client with a natural-language command bar, Slots, and Holds; RSVP accept/decline
is standard for such a client but not documented on pages read (unverified). Merges Google and
Outlook into one unified view with auto-merge of duplicate cross-account events and a "Time
Travel" multi-timezone overlay. Booking uses Personal Booking Links, Slots, and Group Polls;
confirmed bookings land in the connected Google/Outlook calendar (per-link calendar selector
unverified). Proprietary, not self-hostable, not open source. No native Proton — the only route is
subscribing to a Proton ICS link inside Google, then viewing it read-only in Vimcal (inference).

### Rise (risecalendar.com) — DISCONTINUED
Not a live product. [risecalendar.com](https://risecalendar.com/) serves a shutdown notice: the
founders announced the shutdown on **2025-01-27** with a sunset of **2025-03-31** (~17 months ago),
including deletion of user data. The help site `support.risecalendar.com` no longer resolves (DNS
NXDOMAIN). Historical third-party listings claim it once connected Google/Outlook/iCloud with
scheduling links and auto-scheduling "Flexible Events," but no live primary source remains to
verify, so its feature set is not reported as current. **Exclude from the design comparison.**

### cal.com (repo now cal.diy)
The reference self-hostable booking layer. Connects Google, Outlook/365 (Graph), Apple/iCloud
(CalDAV), generic CalDAV (Beta), on-prem Exchange (EWS), Zoho, Lark, Vimcal, plus a read-only ICS
feed ([apps/calendar](https://cal.com/apps/categories/calendar),
[caldav](https://cal.com/apps/caldav-calendar)). **The load-bearing mechanism is the split between
conflict-checking and the destination calendar.** It reads free/busy across ALL connected
calendars so "you will not be double-booked... no matter which event type or calendar it is tied
to" ([conflict checking](https://cal.com/help/event-types/eventtype-specific-checking-for-conflicts)),
but writes the confirmed event to exactly one **"destination calendar"** — a default set at
`/settings/my-account/calendars`, overridable per event type
([add-events](https://cal.com/help/event-types/add-events)). Availability uses reusable schedules
(per-schedule timezone), buffers before/after, minimum notice, booking-frequency and duration
limits, and future date-range limits. The booker is added as an attendee so the calendar sends the
invite. It is **not** a full unified calendar client — the closest is a read-only "Overlay my
calendar" on the booking page; a separate developer product ("Unified Calendar API",
[cal.com/unified](https://cal.com/unified)) is an API, not an end-user grid. Self-hostable OSS:
stack is Next.js / React / TypeScript / tRPC / Prisma / PostgreSQL / Tailwind, Docker or manual
([calcom/docker](https://github.com/calcom/docker)). **Licensing changed in 2026:** the repo
`github.com/calcom/cal.com` was renamed to
[`github.com/calcom/cal.diy`](https://github.com/calcom/cal.diy) and **relicensed from AGPL-3.0 to
MIT** (enterprise features removed). This is the same flagship codebase, not a small fork —
confirmed authoritatively via the GitHub API: `calcom/cal.com` resolves to `calcom/cal.diy`,
`license.spdx_id = MIT`, ~47.8k stars, actively pushed (2026-08-08). Historically cal.com was
AGPLv3. **Design consequence:** under MIT you may vendor/adapt this code into a proprietary hub
without AGPL's network-copyleft obligation; under the old AGPLv3 you could not. The hosted cal.com
product remains commercial. No Proton support (open requests
[#14884](https://github.com/calcom/cal.diy/issues/14884),
[#5756](https://github.com/calcom/cal.diy/issues/5756)).

### Calendly
Booking front-end, not a calendar. Connects Google, Office 365 / Outlook.com, and Microsoft
Exchange, plus grandfathered iCloud (Apple ID + app-specific password; **closed to new connections
since 2024-08-20**) ([calendar connections](https://calendly.com/help/calendar-connections),
[iCloud](https://calendly.com/help/icloud-overview)). Conflict-checking has two layers: 1–6
connected accounts depending on plan (Free = one), and per-calendar caveats — Exchange "can only
check your primary Exchange Calendar for conflicts... cannot check sub-calendars or shared
calendars" ([manage multiple](https://calendly.com/help/how-to-manage-multiple-calendars-and-email-addresses)).
It reads free/busy across connected calendars but **writes each confirmed booking to only ONE
designated "Add to calendar" account.** Booking links expose availability windows, buffers
before/after, minimum notice, meeting limits per day/week/month, a date range, and timezone
auto-detect or lock with DST handling
([availability](https://calendly.com/help/how-to-fine-tune-your-availability-settings),
[time zones](https://calendly.com/help/time-zones-overview)). No combined multi-calendar view.
Proprietary SaaS; not self-hostable; Proton not listed.

### Cronofy
The reference **calendar-aggregation API** — developers build on it instead of integrating each
provider directly. Unifies Google, Microsoft 365, on-prem Exchange, Outlook.com, and Apple iCloud
behind one API and one OAuth model ([calendar-users](https://docs.cronofy.com/calendar-users/)); no
generic CalDAV and no Proton. Reads events and free/busy, creates/updates/deletes events, and
changes RSVP status. The **Availability API** is the core engine: it aggregates free/busy across
multiple participants and multiple connected calendars and returns bookable slots, controlled by
`required_duration`, buffers, per-member `calendar_ids`, `available_periods`, and Availability
Rules; timezones use IANA `tzid`
([availability](https://docs.cronofy.com/developers/api/scheduling/availability/)). **Real-Time
Scheduling** builds a booking link on that engine and, when a slot is chosen, **writes the event
into `target_calendars` and notifies your app via `callback_url`** — with the raw Availability API
you instead get slots and write the event yourself
([RTS](https://docs.cronofy.com/developers/scheduling/real-time-scheduling/)). Push Notifications
(webhook channels) signal that a calendar changed but carry no event data, so you re-read; notably
Cronofy sends **no** push for changes caused by your own API calls
([push](https://docs.cronofy.com/developers/api/push-notifications/)). SaaS only (not
self-hostable); publishes OSS SDKs.

### Baikal
A lightweight **self-hosted CalDAV + CardDAV server** built on the sabre/dav PHP library, with a
web admin UI ([sabre.io/baikal](https://sabre.io/baikal/)). Critical role: it is a **destination
store, not an aggregator** — it does not log into or pull from Google, Microsoft, or Proton;
standards-compliant clients (iOS, macOS, DAVx5, Thunderbird) sync *to* it over CalDAV/CardDAV. Via
sabre/dav it speaks WebDAV, CalDAV (RFC 4791), CardDAV, iCalendar (RFC 5545), vCard, and WebDAV-sync
(RFC numbers not printed on the sabre pages — unverified). Full read-write for connected clients.
Stack: any PHP server plus SQLite/MySQL/PostgreSQL; GPLv3
([github](https://github.com/sabre-io/Baikal)). No booking link. Relevant to a DIY hub as the
**local CalDAV store or front** that a separate aggregation/sync layer writes into.

### Radicale
A small **self-hosted CalDAV + CardDAV server** in Python (3.9+) that "works out-of-the-box"
([radicale.org/v3](https://radicale.org/v3.html)). Like Baikal it is a **destination store, not an
aggregator** — clients sync into it; it connects to no external provider. Supports events, todos,
journals, and contacts with full read-write for clients; storage is plain files on disk (no
database); access limited by htpasswd / LDAP / IMAP / PAM / OAuth2 and secured with TLS; GPLv3
([github](https://github.com/Kozea/Radicale)). No booking link, no scheduling logic. Relevant as
the lightest self-hosted local CalDAV store/front behind a separate aggregation layer.

### Proton Calendar (the walled garden — the load-bearing answer)
Proton is effectively a **one-way, read-only participant** for external tools.

- **CalDAV: explicitly NOT supported.** Proton states plainly: "Proton Calendar doesn't support
  CalDAV, so you can't set up a direct two-way sync with an external calendar"
  ([subscribe-to-external-calendar](https://proton.me/support/subscribe-to-external-calendar)).
  This is a stated fact, not an inference from silence.
- **No supported public Calendar API.** The only public artifacts are the open-sourced React web
  clients ([github.com/ProtonMail](https://github.com/ProtonMail/proton-calendar)) and
  `go-proton-api`, an official Go library implementing "a subset of the Proton REST API" (contains
  `calendar.go`/`calendar_event.go`), maintained by Proton AG but not for third-party use
  ([go-proton-api](https://github.com/ProtonMail/go-proton-api)). The internal REST API is
  E2E-encrypted; programmatic access requires reverse-engineering.
- **Proton Bridge is mail only** (IMAP/SMTP/POP3) — no Calendar or Contacts
  ([mail/bridge](https://proton.me/mail/bridge)).
- **Direction is the constraint.** OUT: a **paid** plan can publish a per-calendar share link
  ("Share with anyone" → "Create link", Limited/busy or Full view) that serves the calendar as ICS
  over HTTPS, **read-only** ("Only you can create and edit events")
  ([share-calendar-via-link](https://proton.me/support/share-calendar-via-link)). IN: subscribing
  to an external URL ("Add calendar from URL") is **view-only** and syncs every 4–16 hours. WRITE-IN:
  nothing external can create/edit events programmatically; the sole inbound paths are a one-time
  ICS import or **emailing an iCalendar invite that a human accepts** inside Proton
  ([add-event-from-inbox](https://proton.me/support/add-event-from-inbox)).

Conclusion: for an aggregator, Proton is **read-only ICS/busy out, view-only ICS in, no
programmatic write in.** It can never be a booking-write destination via API or CalDAV. (A
possibly-official `api.proton.ai` page surfaced in search but could not be confirmed as a Proton
property or as documenting Calendar endpoints — treat as **not** a source (unverified). The ~1 MB
subscribe-size limit is also unverified.)

---

## Synthesis

### Recurring architectural patterns worth borrowing

**1. Unified store (sync) vs live query.** Two ways to build "everything at once":

- *Live query* — fetch free/busy from each provider on demand (Cronofy's Availability API model).
  Simplest; always fresh; but latency scales with account count and you re-hit provider rate limits.
- *Unified local store* — pull events once, then keep in sync incrementally, and serve all views
  and availability from the local copy. This is feasible for the target stack because both major
  providers support **incremental sync**:
  - **Google Calendar API**: `events.list` returns a `nextSyncToken`; pass it as `syncToken` to
    get only changes (including deletions). `410 GONE` means the token expired → wipe and full
    re-sync. Push via `events.watch` channels (webhook carries no event data, does not auto-renew)
    ([sync](https://developers.google.com/workspace/calendar/api/guides/sync),
    [push](https://developers.google.com/workspace/calendar/api/guides/push)).
  - **Microsoft Graph**: `/me/calendarView/delta` pages with `@odata.nextLink` then returns
    `@odata.deltaLink`; reuse it next round for created/updated/deleted events. On v1.0 the
    calendarView delta is **per-calendar and bound to a fixed date range**; change-notification
    subscriptions for `event` expire in under 7 days and need renewal
    ([delta](https://learn.microsoft.com/en-us/graph/delta-query-events),
    [notifications](https://learn.microsoft.com/en-us/graph/change-notifications-overview)).
  - Consequence for 2 Gmail + 1 Microsoft: **three independent token/delta streams and three
    independently-expiring webhook channels.** A local store (with a background sync worker) is the
    realistic choice, with periodic full re-sync as the safety net.

**2. Aggregation API (Cronofy) vs direct provider APIs vs CalDAV as lingua franca.**

- *Aggregation API (Cronofy)*: one API and one OAuth model over Google/Microsoft/Exchange/iCloud,
  plus a ready Availability + booking engine. Fastest path; but SaaS dependency, per-connection
  cost, and **no Proton**.
- *Direct provider APIs (Google Calendar API + Microsoft Graph)*: maximum control, best incremental
  sync, no third-party dependency; but you implement OAuth, sync, and free/busy per provider.
- *CalDAV (RFC 4791) as a common protocol*: attractive as a single client abstraction, and it is
  what Baikal/Radicale and Apple/Fastmail speak. **But the two biggest target providers are not
  first-class CalDAV**: Google and Microsoft are best served by their native OAuth REST APIs, and
  **Proton has no CalDAV at all.** So CalDAV is a lingua franca only for the *self-hosted store
  side* (exposing your hub as CalDAV to clients like DAVx5/Thunderbird), not for ingesting the
  cloud providers.

**3. Booking = read many, write one.** cal.com, Calendly, and Cronofy independently converge on the
same shape: **aggregate free/busy across ALL connected calendars to prevent double-booking, then
write the confirmed event to exactly ONE "destination calendar."** cal.com names it the
"destination calendar" (per-event-type overridable); Calendly calls it the one "Add to calendar"
account; Cronofy writes to `target_calendars`. Availability is computed from working-hours rules +
buffers + minimum notice + frequency/date-range limits, with IANA timezones. Borrow this directly:
the hub computes availability from the union of all four accounts' busy time (Proton included as
busy-only input) and writes bookings to a single writable destination (a Gmail or Microsoft
calendar — never Proton).

**4. Self-hosted CalDAV store as the "front."** Baikal/Radicale are not aggregators, but they show
the pattern for the *outbound* face of the hub: expose the unified store over CalDAV so standard
clients can read/write it, while a separate sync layer reconciles with the cloud providers.

### Trade-offs

| Pattern | Pros | Cons |
|---|---|---|
| Live query (per request) | Always fresh; no store to maintain; simplest | Latency ×N accounts; rate limits; no offline; recomputes every time |
| Unified local store + incremental sync | Fast unified views; one place for availability; resilient to provider hiccups | Must manage sync tokens/delta, webhook renewal, and re-sync on 410/expiry; eventual-consistency lag |
| Aggregation API (Cronofy) | One integration; booking engine included; least code | SaaS cost + dependency; data leaves your infra; **no Proton**; no self-host |
| Direct provider APIs | Full control; best incremental sync; no middleman | Per-provider OAuth + sync work; provider-specific quirks (Graph per-calendar delta) |
| CalDAV as ingest protocol | Single client abstraction; standard | Google/Microsoft better via native API; **Proton has no CalDAV** — dead end for the key case |
| CalDAV as outbound store (Baikal/Radicale-style) | Standard clients can sync to the hub; self-hostable | Still need a separate aggregation/sync layer; not a solution to ingest |

### Does any mainstream tool solve Proton Calendar aggregation?

**No.** None of Morgen, Notion Calendar, Amie, Vimcal, cal.com, Calendly, or Cronofy connects
Proton Calendar via API or CalDAV, because **Proton exposes neither** — no CalDAV endpoint, no
supported public Calendar API, and Proton Bridge is mail-only. The **realistic workaround** is the
only one Proton itself supports:

1. In Proton (paid plan), publish the calendar as a **read-only ICS share link** ("Share with
   anyone" → "Create link"). Choose **Full view**, not Limited/busy view: Full view exposes real
   event titles and details to the subscriber, so the hub shows meaningful events in the unified
   view instead of opaque "busy" blocks. It is read-only either way.
2. In the hub, **subscribe to that ICS URL** and treat those events as **display + busy input
   only** (never writable).

**How this constrains a DIY build:**

- Proton is **input-only**: it contributes busy time to availability and appears in the unified
  view, but **the hub cannot create, edit, RSVP, or delete Proton events programmatically.**
- Proton must **never be the booking-write destination.** Route confirmed bookings to a Gmail or
  Microsoft calendar (which have full read-write APIs), and let Proton pick them up only if the
  user also subscribes the other way.
- Freshness is limited by ICS polling (Proton's own inbound subscribe refreshes every 4–16 hours;
  an outbound share link has no documented guaranteed refresh interval). Treat Proton busy data as
  **coarse and slightly stale**, not real-time.
- The only way to get an event *into* Proton from the hub is out-of-band: **email an iCalendar
  invite** to the Proton address, which the user (or Proton's auto-add) accepts inside Proton's own
  client. This is a human-in-the-loop path, not automation.

### Bottom line for the Kotlin hub

Use **direct provider APIs** (Google Calendar API + Microsoft Graph) with **incremental sync**
(sync tokens / delta queries + webhooks) into a **local unified store** — this handles the 2 Gmail
+ 1 Microsoft accounts with full read-write, colour-by-source views, and a store to compute
availability from. Add **Proton as a read-only ICS subscription** for busy/display only. Implement
booking with the **read-many / write-one** pattern: availability from the union of all four
accounts' busy time; confirmed events written to a single writable Gmail/Microsoft destination
calendar (never Proton). Publish the Proton share link as **Full view** so those events carry real
titles in the unified view rather than opaque busy blocks. Optionally expose the unified store over
**CalDAV** (Baikal/Radicale-style) so standard clients can read/write it. cal.com/cal.diy
(**MIT-licensed as of 2026** — vendorable without AGPL's network-copyleft; Next.js/Prisma/Postgres)
is the closest open-source reference for the booking layer, and Cronofy's Availability + Real-Time
Scheduling docs are the reference for the availability-computation and event-write mechanics.

---

*Primary specs referenced: [RFC 4791 (CalDAV)](https://www.rfc-editor.org/rfc/rfc4791),
[RFC 5545 (iCalendar)](https://www.rfc-editor.org/rfc/rfc5545). Account-type support is the
fastest-rotting fact here; re-verify against current help/pricing pages before committing.*
