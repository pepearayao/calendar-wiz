# ADR 0004 — Booking link rules

- Status: Accepted
- Date: 2026-08-25
- Decides: [Decide: booking-link rules — bookable windows and what a booking may do](https://github.com/pepearayao/calendar-wiz/issues/9)
- Builds on: [ADR 0001](./0001-aggregation-architecture.md), [ADR 0002](./0002-event-data-model.md), [ADR 0003](./0003-busy-mirroring.md), and [`google-microsoft-auth.md`](../research/google-microsoft-auth.md) / [`proton-access.md`](../research/proton-access.md)

## Context

A shareable link lets other people book open time within windows the user predefines. It must offer
only genuinely-free slots, write the booking to a calendar the hub can reliably write, and stay
correct under the constraints already decided (Proton staleness, mirror exclusion).

## Decision

### 1. Model — generatable parametric links under the user's own domain

Booking links are **URLs the user generates by sharing**, not entries hand-created in a UI. The
scheme lives under the user's own domain:

```
https://<user-domain>/agenda/{handle}/{context}/{duration}
e.g. https://pepearayao.com/agenda/rondia/entremed/30
```

- **`{handle}`** is a fixed personal handle segment (e.g. `rondia`) — a stable, shareable identity in
  the path.
- **`{context}`** selects a named **profile / persona** — `personal`, `entremed`, `work`, … — each
  an editable availability profile (below).
- **`{duration}`** selects the meeting length (15, 30, 60, …) from the set the context allows.

Every `context × allowed-duration` combination is **automatically live**, so the user "generates" a
link just by constructing/sharing its path — no per-link creation step. A **profile** (`{context}`)
is the editable unit and holds:

- **bookable days** and **hours** (in the user's timezone)
- **allowed durations**, **buffer** before/after, **minimum lead time**, **booking horizon**
- **destination calendar**, **booker fields**, optional **max bookings/day** cap

A profile can change *display* and *destination*, but **not** which calendars gate conflicts — see §3.

The page is served under the **same domain the hub uses to send iMIP invites** (DKIM-aligned) — one
domain for the whole hub (booking pages out, invites out).

### 2. Defaults (editable per link)

Days **Mon–Fri**; hours **09:00–17:00** local; **15-min buffer**; **4-hour minimum lead time**;
horizon **4 weeks**. Duration is chosen per link.

### 3. Availability computation

For each candidate slot in the link's window (days × hours × duration, across the horizon, minus
lead-time and buffers), the slot is **offered only if free across all four calendars**:

- Subtract **busy** real events from **Gmail #1, Gmail #2, Teams, and Proton** (Q1: all four).
  **All four are always checked for conflicts, regardless of the link's context** — a profile can
  never opt out of a busy source, or a `work` link could book over a personal commitment (the exact
  failure this system exists to prevent). A context narrows what is *shown*, never what *blocks*.
- Treat the user's own **tentative** real events as busy too (conservative — don't get booked over a
  "maybe"). Free and declined events don't block. **This deliberately diverges from ADR 0003**, which
  does *not* mirror tentative events: mirroring omits maybes to avoid cluttering other calendars,
  while booking includes them to protect the user. Same fact, opposite direction, on purpose.
- **Exclude mirror blockers** (`is_mirror = true`, ADR 0003 §7) — they'd double-count the same
  commitment the real event already blocks.

### 4. Destination and confirmation

The confirmed booking is written to the link's **destination calendar** — a **writable Gmail** by
default (API write = **busy**), chosen **per link** (Q3). The booker is added as an **attendee** so
the calendar sends them the invite/confirmation. The new event then **mirrors** as blockers to the
other calendars via the mirror engine (ADR 0003). Teams is not a booking destination (its write may
be admin-blocked).

### 5. Booker fields

**Name, email, reason/agenda, and phone** (Q4). Email is required to send the invite.

### 6. Timezone

The booking page shows slots in the **booker's** timezone (auto-detected, overridable); the event is
stored per ADR 0002 (UTC instant + the user's IANA tz). Windows in the profile are defined in the
**user's** timezone.

### 7. Concurrency — no double-booking

Two people can hit the same slot at once. On confirmation the hub must **re-check availability and
atomically claim the slot** (a short-lived hold / unique constraint on the destination time range)
**before** writing the event — never trust the availability shown at page load. A lost race returns
"slot just taken, pick another."

### 8. Proton staleness — bounded, with a safety net

Availability reads real Proton events at **~8 h staleness** (ADR 0002 / #5), so a very recent Proton
event may not yet block. Mitigations: the **minimum lead time** shrinks the exposed window, and a
**post-sync reconcile** — when Proton's next ICS pull reveals a conflict with a just-made booking,
the hub flags it and offers to reschedule. Residual risk is accepted; Proton is treated as coarse.

### 9. Abuse (public page)

The page is public, so: **rate-limit** by IP, honour the per-link **max/day** cap, basic bot
mitigation, and require a valid email (confirmation to the address). Exact controls are a build-time
detail.

## Consequences

- One availability engine (windows minus busy-across-four, mirrors excluded) serves every link.
- Bookings land as **busy** on a Gmail and propagate everywhere via mirroring, so the user's whole
  system reflects them immediately.
- The event-type model scales from one link to many without a redesign.
- The one correctness hazard is the booking **race** (§7); it must be handled at write time, not by
  the page's stale view.

## Deferred / build-time

- The public booking page UI (feeds the [view prototype #7](https://github.com/pepearayao/calendar-wiz/issues/7) / the web-UI framework choice).
- The **context → profile** registry (how personas like `personal` / `entremed` / `work` are
  defined and edited) and the allowed-duration set per context.
- **Public domain + hosting** — the hub must be reachable at the user's own domain (e.g.
  `pepearayao.com`) to serve `/agenda/...` and to send DKIM-aligned invites. Ties to the deployment
  and email-sending-identity fog on the map.
- Exact abuse controls and the hold/lock mechanism.
- **Build vs vendor** the booking layer (cal.com/cal.diy is MIT but TypeScript — tension with the
  Kotlin learning goal); still open in the map fog.
