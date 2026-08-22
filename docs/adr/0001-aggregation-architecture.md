# ADR 0001 — Aggregation architecture: unified store + two-way sync engine

- Status: Accepted
- Date: 2026-08-23
- Decides: [Decide: aggregation architecture — unified store vs live-query](https://github.com/pepearayao/calendar-wiz/issues/8)
- Builds on research: [existing-tools survey](../research/existing-tools-survey.md), [Proton access](../research/proton-access.md), [Google + Microsoft auth](../research/google-microsoft-auth.md)

## Context

calendar-wiz aggregates four calendars — 2 Gmail, 1 Proton, 1 Microsoft/Teams (work) — for one
person on a self-hosted machine that is up 24/7. It must present one unified view, write events
back to the writable sources, expose a booking link, and — the decisive requirement — keep all
calendars **in sync via busy-mirroring** (a commitment on any calendar becomes a blocker on the
others). It is built in Kotlin as a learning project, with a possible mobile app later.

The research established: no tool aggregates Proton (read-only ICS only, ~8 h stale); Google and
Microsoft have full read/write plus cheap incremental sync (`syncToken` / Graph `delta`); the
Microsoft work tenant may block writes; booking should write to Gmail. Mirroring requires writing
tracked blocker events back into each source and healing them across updates/deletes.

## Decision

### 1. Unified local store + two-way sync engine (not live-query)

A local store is the source of truth. Background workers sync each provider **in** and mirror
blockers **out**. A stateless live-query aggregator is ruled out: it cannot hold mirror-mapping
state, cannot dedup, cannot compute booking availability without per-request latency across four
providers, and cannot survive a provider outage. Proton is stale ICS and must be cached regardless.

### 2. Store: SQLite via SQLDelight

Embedded, zero-ops, one file to back up. SQLDelight gives type-safe Kotlin queries. The load is
tiny and single-writer (the sync engine), so SQLite is not a bottleneck. Postgres was rejected as
operational overhead unjustified at this scale.

### 3. Runtime: one Ktor service

A single process hosts the HTTP/JSON API, serves the web UI, and runs the sync and mirror workers
as background coroutines. One unit to start, watch, and restart. Split daemons were rejected as
needless IPC/coordination at this scale.

### 4. Client contract: HTTP/JSON API + web UI first

The core exposes a clean HTTP/JSON API. The web UI consumes it now; a future iOS/Android app reuses
the same API — this keeps the mobile door open cheaply. A CalDAV-outbound server is **deferred**:
busy-mirroring already fills every native calendar, making CalDAV largely redundant. Revisit only if
a native client should edit the unified store directly.

### 5. Internal decomposition

**Canonical `Event` model** — one normalized event the whole app speaks, so provider quirks do not
leak upward:

- identity: `sourceId`, `sourceEventId`, internal `id`
- time: `start`, `end`, `timezone`, `allDay`, recurrence
- content: `title`, `busy`/`free`, `status`
- mirror bookkeeping: `isMirror` (a denormalized read-side flag) plus sync timestamps. **The
  `mirror_map` table below is the authority for mirror linkage**, not a field on `Event`: a real
  event has N blocker copies across N targets, which a single `mirrorOf` field cannot express.

**Components (all inside the one Ktor service):**

- **Provider adapters** — one per source behind a common interface (`pullChanges`,
  `create`/`update`/`delete`, `freeBusy`) with **capability flags** (`canWrite`,
  `canBeMirrorTarget`). `GoogleAdapter`, `MicrosoftAdapter`, `ProtonIcsAdapter` (read-only); a
  `ProtonScrapeAdapter` slots in later if the spike approves it. Proton's and Microsoft's
  write-uncertainty stays contained as a flag, never a rewrite.
- **Store (SQLite)** — three load-bearing tables: `events`; `mirror_map` (a real event → its blocker
  copies per target); `sync_state` (sync token / deltaLink / last-poll per source).
- **Sync engine** — per-source workers: webhooks where available (Google `watch`, Graph
  subscriptions) + poll timers + full re-sync on token expiry; Proton polls the ICS every ~8 h.
- **Mirror engine** — separate from the sync engine. On any store change it reconciles blocker
  events into each writable target and tags them so a mirror is never re-mirrored. Its detailed
  rules live in the mirroring ticket.
- **API layer** — range queries for the view, availability for booking, write operations.

### 6. Write mechanism: provider API first, iMIP email-invite as a universal fallback

An adapter's `create`/`update`/`delete` is implemented by the **provider API where the API can
write** (Google always; Microsoft if the tenant permits). Where the API cannot write — **Proton
always; Microsoft if the admin blocks it** — the adapter writes by **emailing an iCalendar invite**
(iMIP / iTIP: `METHOD:REQUEST` to create/update with a bumped `SEQUENCE`, `METHOD:CANCEL` to delete)
from a dedicated hub sending identity. This is the only sanctioned way to write into Proton and a
robust fallback for a locked-down work tenant, and it writes near-instantly. It hangs on one
empirical question — do Proton and Outlook **auto-add** an emailed invite, or require a manual accept
each time — decided by the spike ([#14](https://github.com/pepearayao/calendar-wiz/issues/14)). The
mirror engine treats "write a blocker" uniformly; the adapter chooses API or email underneath. If the
email path works, the Proton scraping client is not needed for writes.

## Consequences

- Mirroring, booking, and the view all read/write one consistent local model — fast and offline-safe.
- Write-uncertainty (Microsoft admin, Proton) is absorbed by adapter capability flags; a source that
  cannot be written simply is not a mirror/booking target.
- The store must **distinguish real events from mirror blockers** everywhere (a marker), or booking
  free/busy will double-count and mirrors will loop.
- Sync is eventually-consistent; Proton lags ~8 h, so it is treated as low-churn "committed" busy.
- A mobile app later is an API client — no core rewrite.

## Deferred to other tickets / fog

- Mirroring internals — direction matrix, blocker privacy, loop/dedup marker, propagation:
  [#12](https://github.com/pepearayao/calendar-wiz/issues/12).
- Booking rules and booking-link implementation: [#9](https://github.com/pepearayao/calendar-wiz/issues/9).
- Proton access strategy (does `ProtonScrapeAdapter` exist?): [#10](https://github.com/pepearayao/calendar-wiz/issues/10).
- Microsoft tenant reality (is `MicrosoftAdapter` writable?): [#11](https://github.com/pepearayao/calendar-wiz/issues/11).
- Write via iMIP email invites (the universal fallback in decision 6): [#14](https://github.com/pepearayao/calendar-wiz/issues/14).
- Detailed data model — **recurrence** (store series vs expand instances; per-instance
  exceptions differ across providers; mirroring a recurring event is categorically harder than a
  single one), timezone normalization, and cross-source dedup: [#13](https://github.com/pepearayao/calendar-wiz/issues/13).
- Auth/token storage, deployment shape, and web-UI framework choice.
