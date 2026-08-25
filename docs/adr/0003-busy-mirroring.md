# ADR 0003 — Cross-calendar busy-mirroring

- Status: Accepted
- Date: 2026-08-25
- Decides: [Decide: cross-calendar busy-mirroring (blockers across all calendars)](https://github.com/pepearayao/calendar-wiz/issues/12)
- Builds on: [ADR 0001 — architecture](./0001-aggregation-architecture.md), [ADR 0002 — data model](./0002-event-data-model.md), and the write-path findings in [`email-invite-autoadd.md`](../research/email-invite-autoadd.md)

## Context

A commitment on any calendar should appear as a **blocker** on the others, so every calendar — and
external free/busy lookups — reflect the user's whole day. The mirror engine (ADR 0001) writes these
blockers. This ADR fixes what it mirrors, where, how detailed, how strong, and how it avoids loops.

## Decision

### 1. Direction — all-to-all across all four calendars

Every real event is mirrored to the **other three** calendars, minus any that already hold it
(dedup presence, ADR 0002). All four are reachable: Gmail and Teams via API, Proton and
API-blocked-Teams via **emailed invite** (ADR 0001 / the auto-add research).

**Only real events (`is_mirror = false`) are inputs to the mirror engine.** A blocker is never
itself a mirror source. This bounds the fan-out to (real events × the other targets) — three
blockers per event — instead of compounding: blockers written by API reappear on the next Google/Graph
sync, and emailed blockers reappear in Proton's ICS read, but the ingest guard (§5) skips them, so
they never spawn further mirrors.

### 2. What is mirrored — confirmed + busy only

Mirror an event only when it is **confirmed** and **busy** (per the ADR 0002 free/busy flag):

- **timed busy** events, and **all-day busy** events (OOO, offsite, travel);
- **not free** events (birthdays, banners), **not tentative** events (a "maybe" must not close time),
  **not declined** events (declined ⇒ not busy for the user).

### 3. Blocker content — generic, no detail leak

A blocker shows **"Busy"**, optionally tagged with the source area (e.g. **"Busy · Personal"**) — no
title, description, location, or attendees. The real detail lives on the home calendar and in the
hub's unified view. This protects the **work** calendar (visible to admin/coworkers) from personal
detail. (Per-direction title pass-through for safe pairs is a future option, off by default.)

### 4. Block strength — strongest available write per target

A blocker written by **API** reads **busy** (blocks others); one written by **email invite** is
**tentative/pending** (visible to the user, may read free to others). Per target:

- **Gmail:** API → **busy**.
- **Teams:** prefer **Graph API → busy** (when the tenant allows, per [#11](https://github.com/pepearayao/calendar-wiz/issues/11)); else an admin **"Direct to Calendar"** rule → busy; else **email invite → tentative** as the floor. Use the strongest available.
- **Proton:** **email invite → pending**. Acceptable: Proton is not others-facing (nobody books the user via Proton), so "pending" is fine — it gives the user visibility.

### 5. Loop control — `mirror_map` is authoritative, marker is a hint

Every blocker the hub writes is recorded in **`mirror_map`** and stamped with a marker
`X-CALWIZ-MIRROR-OF: <source_id>:<source_event_id>` (Google `extendedProperties.private`, Graph
extension, and an `X-` property inside the emailed `VEVENT`). On ingest, an incoming event is treated
as a **mirror (and skipped — never read as real, never re-mirrored)** if it is **in `mirror_map`**
(matched by target + `UID`) **or** carries the marker. `mirror_map` is authoritative because a
provider may strip `X-` properties on export (notably a risk for Proton's ICS feed) — matching the
hub's own record of what it wrote where guarantees the loop is broken even then.

**The mirror key is the anchor copy's stable identity, not the cluster.** Clustering is *derived*
and can be re-computed differently (ADR 0002's conservative fallback), so keying mirrors on
`cluster_id` would orphan `mirror_map` rows whenever a cluster shifts. Instead, each logical event
picks a stable **anchor** — the `(source_id, source_event_id)` of one primary real copy (deterministic:
the min over the cluster's copies) — and `mirror_map` and the marker key on that. Re-clustering never
moves the anchor; only deletion of the anchor copy triggers a re-anchor to another copy in the cluster
(and a `mirror_map` migration), an edge case.

### 6. Linkage and propagation

`mirror_map` ties each real event's **anchor** `(source_id, source_event_id)` to its blocker on each
target: `target_source_id`, the mirror event's id/`UID`, `sequence`, and state. The cluster/presence
set (ADR 0002) decides *which* targets still need a blocker; the anchor identity is the durable key.
Propagation:

- **Create:** real busy event appears → write a blocker to each other target lacking it.
- **Update:** real event moves/changes busy state → update its blockers (API `patch`, or email
  `REQUEST` with `SEQUENCE`+1). If it becomes free/tentative/declined → remove its blockers.
- **Delete/cancel:** real event deleted (ADR 0002 tombstone) → delete the blockers (API `delete`, or
  email `METHOD:CANCEL`).

### 7. Booking correctness and organizer identity

- Mirror rows (`is_mirror = true`) are **excluded** from the hub's own booking free/busy and never
  clustered as real events — the booking link reads **real** events only, so nothing double-counts.
- The hub is the **`ORGANIZER`** of every emailed blocker, from its own **DKIM-aligned** domain
  (never the user's address — that would fail DMARC and break later updates/cancels).

## Consequences

- Every calendar shows the user's full day; coworkers checking Teams see **busy** whenever the API
  path is available, and at worst a visible-to-user tentative block otherwise.
- No detail leaks to the work calendar.
- The loop guard is robust to providers stripping the marker, because `mirror_map` records ground
  truth.
- Reconciliation is idempotent: the desired blocker set for each real event is recomputed and
  diffed against `mirror_map`, so a re-run converges rather than duplicating.
- **Booking staleness carries through.** Booking availability reads real events only (§7), and
  Proton's real events arrive ~8 h stale (ADR 0002 / Proton research); mirrors are excluded, so they
  can't compensate. A Proton event created minutes ago is invisible to the booking link — the #5
  double-booking risk resurfacing here. The booking-rules ticket ([#9](https://github.com/pepearayao/calendar-wiz/issues/9)) must account for it.

## Deferred / validate at build

- The exact reconciliation algorithm and its batching/rate-limit behaviour.
- Whether Proton's ICS feed preserves the `X-CALWIZ-MIRROR-OF` marker (if not, `mirror_map` matching
  by `UID` carries the loop guard) — confirm during the Proton write check.
- Whether to pursue an admin **"Direct to Calendar"** rule for true busy on Teams — depends on the
  [#11](https://github.com/pepearayao/calendar-wiz/issues/11) tenant result.
- The source-tag label format ("Busy · Personal" vs source calendar name).
