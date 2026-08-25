# calendar-wiz — build guide

The start-here document. The design is complete; this sequences it into buildable phases and lists
the invariants you must not get wrong. Every claim links to the ADR or research that owns the detail.

- Architecture: [ADR 0001](adr/0001-aggregation-architecture.md)
- Data model: [ADR 0002](adr/0002-event-data-model.md)
- Busy-mirroring: [ADR 0003](adr/0003-busy-mirroring.md)
- Booking rules: [ADR 0004](adr/0004-booking-rules.md)
- Research: [existing tools](research/existing-tools-survey.md) · [Proton](research/proton-access.md) · [Google/Microsoft](research/google-microsoft-auth.md) · [email-invite auto-add](research/email-invite-autoadd.md)
- View prototype: https://claude.ai/code/artifact/3751fb68-58d9-47ae-98c3-f5685191ae5e

## What it is

A self-hosted **Kotlin** service, up 24/7 at the user's own domain, that:

1. aggregates **2 Gmail + 1 Proton + 1 Microsoft/Teams** into one unified view;
2. writes create/edit/RSVP back (provider API, or emailed iMIP invite where the API can't write);
3. keeps all calendars in sync via **busy-mirroring**;
4. exposes **booking links** `pepearayao.com/agenda/{handle}/{context}/{duration}`.

## Stack

- **Kotlin** + **Ktor** (one service: HTTP/JSON API + web UI + background coroutine workers).
- **SQLite** via **SQLDelight** (embedded store; three core tables: `events`, `mirror_map`, `sync_state`).
- **Google Calendar API** + **Microsoft Graph** (direct provider APIs).
- An **iCal library** (biweekly / ical4j) for Proton ICS parsing + recurrence expansion.
- An **SMTP sending identity** on a DKIM-aligned domain (for iMIP invite writes).

The core exposes an HTTP/JSON API; the web UI consumes it now, a mobile app later reuses it. See
[ADR 0001](adr/0001-aggregation-architecture.md).

## Component map

```
                 ┌──────────────── one Ktor service ────────────────┐
 providers  ⇄    │  adapters (capability flags)                     │
 Google/MS API   │    Google · Microsoft · ProtonICS · (ProtonScrape?)│
 Proton ICS      │        │            ▲ write: API, else iMIP email │
 SMTP (iMIP)     │        ▼            │                             │
                 │   sync engine → [ SQLite store ] ← mirror engine  │
                 │                     │  events / mirror_map / sync │
                 │                     ▼                             │
                 │   HTTP/JSON API → web UI  ·  /agenda booking pages│
                 └───────────────────────────────────────────────────┘
```

## Build order

Each phase is independently demoable. Build in order; later phases assume earlier ones.

### Phase 0 — Foundations + one read-only source
- Ktor service skeleton; SQLite + SQLDelight; the `Event` schema and the three tables ([ADR 0002](adr/0002-event-data-model.md)).
- The `CalendarAdapter` interface with capability flags (`canWrite`, `canBeMirrorTarget`).
- **GoogleAdapter (read):** OAuth — publish the app to **Production, leave it unverified** (avoids the Testing 7-day token death); initial + incremental sync (`events.list` + `nextSyncToken`), store events. ([google-microsoft-auth.md](research/google-microsoft-auth.md))
- Minimal web week-view reading the store over the JSON API.
- **Done when:** one Gmail's real events render in the unified view.

### Phase 1 — All four sources, read, correct
- Second Gmail (same OAuth client, its own refresh token); **MicrosoftAdapter (read)** via Graph `calendarView/delta` (gated by [#11](https://github.com/pepearayao/calendar-wiz/issues/11)); **ProtonIcsAdapter** subscribing to the Full-view ICS share link (read-only, **~8 h stale**).
- **Dedup + clustering** ([ADR 0002](adr/0002-event-data-model.md)): cluster by iCal `UID` keyed `(uid, recurrence_id)`; conservative title+time fallback; per-source presence.
- **Recurrence:** consume provider-expanded instances; expand Proton's RRULEs locally; rolling window (−1mo … +12mo); cancelled occurrences as tombstones.
- **Timezones:** store UTC + IANA; render the **multi-timezone ruler** (home + 1–2 configurable) and hover-for-detail from the prototype.
- **Done when:** all four calendars appear in one deduped, recurrence-correct, multi-zone week view.

### Phase 2 — Writes (incl. the email-invite channel)
- Adapter writes: Google/Microsoft via API. Where the API can't write (**Proton always; Teams if admin-blocked**), the **iMIP email-invite writer**: hub is `ORGANIZER` on its **own DKIM-aligned domain**, emits `METHOD:REQUEST` (create/update via `SEQUENCE`), `METHOD:CANCEL` (delete). ([email-invite-autoadd.md](research/email-invite-autoadd.md))
- Stand up the sending domain (SPF/DKIM/DMARC).
- Create/edit/delete/RSVP from the unified view.
- **Done when:** creating an event in the hub lands it on the chosen calendar (API or emailed invite).

### Phase 3 — Busy-mirroring
- The **mirror engine** ([ADR 0003](adr/0003-busy-mirroring.md)): on any store change, reconcile a **generic "Busy · [source]"** blocker onto every *other* writable target that lacks it; **strongest write per target** (Teams busy via API when allowed); update/delete propagation.
- **Loop control:** record each blocker in `mirror_map`, stamp `X-CALWIZ-MIRROR-OF`, key on the **stable anchor** `(source_id, source_event_id)`; on ingest skip anything in `mirror_map` or carrying the marker.
- **Done when:** a commitment on any calendar produces blockers on the others, with no loops or duplicates.

### Phase 4 — Booking links
- **Availability engine** ([ADR 0004](adr/0004-booking-rules.md)): profile window minus busy across **all four** (always), tentative-blocks, **mirrors excluded**.
- Parametric routes `/agenda/{handle}/{context}/{duration}`; per-context editable profiles; the public booking page (booker: name/email/reason/phone).
- **Atomic slot-claim** at confirmation (re-check + claim before write — no double-booking); write to the per-link Gmail destination with the booker as attendee; the booking then mirrors.
- **Done when:** sharing a link lets someone book an open slot that lands and mirrors everywhere.

### Phase 5 — Run it 24/7 on Hermes
- **Deployment is decided in [ADR 0005](adr/0005-deployment.md):** right-sized for the shared
  `hermes-server` box — **plain Docker Compose** (no Swarm), image built in CI and pushed to
  **ghcr.io**, **Watchtower** auto-pulls on merge, **Caddy on 443** for TLS (port 80 stays munich's),
  `-Xmx256m` + a **2 GB swapfile** (1.9 GB box), reuse munich's `autoheal`. Do **not** replicate
  rondia's Swarm/Traefik/webhook stack.
- Secret storage (host `.env`): refresh tokens, Proton share URL, SMTP/DKIM creds, ghcr token.
- Webhook renewal (Google `watch`, Graph subscriptions) + periodic full re-sync fallback;
  **Proton post-sync reconcile** (flag a booking a later Proton pull reveals as conflicting); abuse
  controls on the public page.
- **Open items before first deploy** (ADR 0005): hostname/DNS (→ TLS challenge method), ghcr image
  visibility, swap added, email/DKIM domain.
- **Done when:** merge → CI → Watchtower → live, running unattended, without disturbing munich.

## Invariants — do not get these wrong

1. **Only real events (`is_mirror = false`) are mirror sources.** Skip anything in `mirror_map` or carrying the marker on ingest, or mirroring loops/explodes. ([ADR 0003](adr/0003-busy-mirroring.md))
2. **Booking always checks all four calendars.** A context profile narrows *display*, never conflict-checking — or a work link books over a personal commitment. ([ADR 0004](adr/0004-booking-rules.md))
3. **Booking is atomic:** re-check availability and claim the slot before writing; never trust the page's stale view.
4. **The hub is the `ORGANIZER`** of every emailed invite, on its own DKIM-aligned domain — never spoof the user's address (DMARC fails; updates/cancels break).
5. **Cluster recurring instances on `(uid, recurrence_id)`**, not `UID` alone, or a series collapses into one blocker.
6. **Cancelled occurrences are tombstones** (`status = cancelled`), so the mirror engine can delete the right blocker.
7. **Mirror key is the stable anchor** `(source_id, source_event_id)`, not the derived `cluster_id`.
8. **Proton is read-only + ~8 h stale**, never a booking destination; emailed writes land **pending/tentative** (visible to you, may read *free* to others).

## Open validations (don't block the build)

- **[#10 Proton reads](https://github.com/pepearayao/calendar-wiz/issues/10)** — is a scraping client worth it *only* for reads fresher than 8 h? Default: no; ICS-in + email-invite-out. Affects whether a `ProtonScrapeAdapter` exists (Phase 1/2).
- **[#11 Teams tenant](https://github.com/pepearayao/calendar-wiz/issues/11)** — does the work admin block the Graph API, and does an external invite auto-add? Decides whether Microsoft is API-writable (busy) or email-invite/read-only (Phase 2/3).

The design already handles every outcome; these refine the Proton/Teams specifics, not the plan.
