# calendar-wiz

One place for all my calendars. A self-hosted **Kotlin** service that aggregates **2 Gmail, 1 Proton,
and 1 Microsoft/Teams** calendar into a single view, keeps them in sync with **busy-mirroring**, and
exposes shareable **booking links**. Built to learn Kotlin; a mobile app may follow.

**Status: design complete — ready to build.** Planned with `/wayfinder`; the design lives in
`docs/`.

## Start here

→ **[docs/BUILD.md](docs/BUILD.md)** — the build guide: what to build, in what order, and the
invariants not to get wrong.

## What it does

- **Aggregate** four calendars into one *"everything at once"* week view (colour-coded sources,
  weekends, a configurable multi-timezone ruler, hover-for-detail).
- **Write back** — create / edit / RSVP via each provider's API, or via an emailed iCalendar invite
  where the API can't write (Proton; an admin-blocked Teams).
- **Busy-mirroring** — a commitment on any calendar drops a private *"Busy"* blocker on the others,
  so every calendar reflects the full day.
- **Booking links** — `pepearayao.com/agenda/{handle}/{context}/{duration}`; people book only open
  slots, checked against all four calendars.

## The design

**Decisions (ADRs)**
- [ADR 0001 — Aggregation architecture](docs/adr/0001-aggregation-architecture.md): unified local store + two-way sync engine; SQLite/SQLDelight; one Ktor service.
- [ADR 0002 — Event data model](docs/adr/0002-event-data-model.md): recurrence, timezones, cross-source dedup.
- [ADR 0003 — Busy-mirroring](docs/adr/0003-busy-mirroring.md): direction, privacy, block strength, loop control.
- [ADR 0004 — Booking rules](docs/adr/0004-booking-rules.md): generatable links, availability, no double-booking.

**Research**
- [Existing tools survey](docs/research/existing-tools-survey.md)
- [Proton access](docs/research/proton-access.md)
- [Google + Microsoft auth](docs/research/google-microsoft-auth.md)
- [Email-invite auto-add](docs/research/email-invite-autoadd.md)

**Prototype** — [unified view mock](https://claude.ai/code/artifact/3751fb68-58d9-47ae-98c3-f5685191ae5e)

## Stack

Kotlin · Ktor · SQLite/SQLDelight · coroutines · Google Calendar API · Microsoft Graph · iCal
(biweekly/ical4j) · SMTP (iMIP invites).

## Open validations

Two account checks remain — neither blocks building:
[Proton reads](https://github.com/pepearayao/calendar-wiz/issues/10) ·
[Teams tenant](https://github.com/pepearayao/calendar-wiz/issues/11).

## Repo layout

- `docs/BUILD.md` — build guide (start here)
- `docs/adr/` — architecture decision records
- `docs/research/` — research summaries
- `docs/agents/` — agent/tooling config (issue tracker)
