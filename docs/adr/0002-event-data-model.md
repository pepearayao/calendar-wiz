# ADR 0002 — Unified event data model: recurrence, timezones, dedup

- Status: Accepted
- Date: 2026-08-23
- Decides: [Decide: unified event data model — recurrence, timezones, cross-source dedup](https://github.com/pepearayao/calendar-wiz/issues/13)
- Builds on: [ADR 0001 — aggregation architecture](./0001-aggregation-architecture.md)

## Context

The unified store ([ADR 0001](./0001-aggregation-architecture.md)) holds events from 2 Gmail, 1
Proton, and 1 Microsoft calendar. The view, booking free/busy, and busy-mirroring all read this
model, so its semantics must be pinned before the schema is built. Three problems dominate:
recurring events, timezones/all-day, and the same meeting appearing on more than one calendar.

## Decision

### 1. Recurrence — hybrid (instances within a rolling window + the master)

Store concrete **instances** for a rolling window (default **−1 month … +12 months**, re-expanded as
it rolls) *and* keep the recurring **master** (its `RRULE`) for whole-series edits and re-expansion.

- Google (`singleEvents=true`) and Microsoft (`calendarView`) **expand recurrences server-side**; the
  sync engine ingests their instances directly. Proton's raw ICS carries `RRULE`s, expanded
  **locally** with an iCal library (e.g. biweekly / ical4j).
- Per-instance **exceptions** are ordinary instance rows: an edited occurrence carries its
  `recurrence_id` (the `RECURRENCE-ID`) and its own fields; a cancelled occurrence (`EXDATE`) is kept
  as a **tombstone** row (`status = cancelled`), **not** silently dropped — so the mirror engine can
  tell a real cancellation (delete that blocker) from an occurrence merely outside the sync window.
- The view and booking read instances (fast, no per-query expansion). "Edit the whole series" edits
  the master and re-expands.

### 2. Timezones and all-day

- **Timed events:** store as a **UTC instant** (`start_utc`/`end_utc`) plus the original **IANA
  timezone** (`tz`); render in the viewer's local zone.
- **All-day events:** store as **floating dates** (`start_date`/`end_date`, no zone).
- **Busy semantics:** respect each event's own free/busy **transparency flag** (`busy`). An all-day
  event counts as busy only when the source marks it busy — an all-day OOO/offsite blocks; a birthday
  or banner does not. This flag drives both mirroring and booking availability.

### 3. Cross-source dedup — UID-first, conservative fallback, presence-tracked

The same meeting can live on several calendars. Detect duplicates and group them:

- **Primary key: the iCalendar `UID`** (`iCalUID` in Google, `iCalUId` in Graph, present in Proton
  ICS). Same UID ⇒ same meeting — **but every instance of a recurring series shares the master's
  UID**, so the clustering key for an instance is **`(ical_uid, recurrence_id)`** (falling back to
  `(ical_uid, start_utc)` when there is no `RECURRENCE-ID`), never `ical_uid` alone. Otherwise a
  52-week series collapses into one bogus cluster and mirroring writes a single year-long blocker.
- **Fallback:** only when there is no shared UID, a **conservative** title+time match, tuned to
  **under-merge** — never fuzzy-merge two same-time events into one, because a false merge could hide
  a real double-booking.
- **Presence tracking:** record which sources each logical event lives on, so **mirroring writes a
  blocker only to calendars that lack the real event** (never a duplicate next to it), and booking
  counts the busy time **once**.

### 4. Schema shape — raw per-source rows + `cluster_id`

Keep **one row per source event copy** — lossless, so each copy keeps its provider id, ETag, and
sync handle for incremental sync and write-back. Group duplicates with a derived **`cluster_id`**; a
**logical event is a cluster**. Do **not** merge copies into one row (that destroys the per-source
handles and gets messy when copies diverge).

```
events
  id                    -- internal PK
  source_id             -- which account/source (gmail-1, gmail-2, proton, ms)
  source_event_id       -- provider's id (null for mirror rows we authored)
  ical_uid              -- dedup key
  cluster_id            -- groups duplicate copies into one logical event
  start_utc, end_utc    -- instants (timed events)
  tz                    -- original IANA tz (null for all-day)
  all_day               -- bool
  start_date, end_date  -- floating dates (all-day; null for timed)
  title
  busy                  -- free/busy transparency -> drives mirroring + booking
  status                -- confirmed / tentative / cancelled
  is_recurring_master   -- bool; true on the stored master
  rrule                 -- recurrence rule (on the master)
  recurrence_master_id  -- FK to master (on an instance)
  recurrence_id         -- RECURRENCE-ID (on an exception instance)
  is_mirror             -- bool; true for blocker events the hub authored
  etag, updated_at      -- change detection
  raw                   -- optional original payload
```

- A **logical event** = the set of `events` rows sharing a `cluster_id`.
- **Mirror rows** (`is_mirror = true`) are the blocker events the hub writes out. They are **never
  clustered as real events** and never re-mirrored; their linkage to the real cluster lives in
  `mirror_map` (ADR 0001), not in `cluster_id`.
- **Change detection / identity:** the stable per-source key is `(source_id, source_event_id)`;
  `etag`/`updated_at` detect updates; sync tokens live in `sync_state` (ADR 0001).

## Consequences

- The view, booking, and mirror engines all operate on **clusters**, so a meeting on three calendars
  shows once, blocks once, and is never mirrored back onto a calendar that already has it.
- Recurrence expansion is bounded and mostly offloaded to the providers; only Proton needs local
  expansion.
- Instances are pre-materialized, so the view and booking are simple range queries — no expansion on
  the hot path.
- The rolling window means events far in the future aren't materialized until the window reaches
  them; the master is retained so nothing is lost.
- **Trailing edge:** when the window advances and old instances drop off the −1-month edge, their
  already-written mirror blockers are **left in place** — window eviction never deletes mirrors, so
  past blockers are not rewritten each month. A mirror is deleted only on a real cancellation
  (the tombstone in decision 1), not on falling out of the window.
- Mirroring correctness now has its data foundation ([#12](https://github.com/pepearayao/calendar-wiz/issues/12) is unblocked).

## Deferred

- The exact conservative-fallback matching rule (fields, tolerance) — tune during build.
- The rolling-window bounds are a default (−1 mo / +12 mo), adjustable.
- `mirror_map` columns and the mirror reconciliation algorithm: the mirroring ticket
  ([#12](https://github.com/pepearayao/calendar-wiz/issues/12)).
