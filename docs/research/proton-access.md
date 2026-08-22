# Proton Calendar Access Paths, Risk, and Recommendation

## Scope & date

Research date: **2026-08-22**. This document fills the gaps the existing survey
(`docs/research/existing-tools-survey.md`) left thin about **Proton Calendar** access, for the
self-hosted **Kotlin** hub (up 24/7) that aggregates 2 Gmail + 1 Proton + 1 Microsoft calendar and
exposes a booking link. It does **not** re-litigate the survey's established facts (Proton has no
CalDAV, no supported public Calendar API, and Bridge is mail-only). It adds four things: (1) the
unofficial/reverse-engineered access paths and their risk, (2) verified ICS share-link details that
the survey marked *(unverified)*, (3) a precise, sourced explanation of the E2E-encryption
constraint, and (4) a must/nice/drop recommendation. All non-obvious claims cite the primary source
that owns them; unconfirmable claims are marked **(unverified)**. No real credentials or share-link
tokens appear here.

## Executive answer

Proton stays exactly where the survey put it — **read-only, display + busy INPUT ONLY, never a
booking-write destination** — and the single realistic mechanism is the **outbound ICS share link
on a paid plan, set to Full view, treated as read-only**. Bookings are never routed to Proton;
getting an event *into* Proton from the hub remains a manual email-invite path. The unofficial
clients that *can* read/write Proton Calendar programmatically (best of them:
[`roman-16/proton-cli`](https://github.com/roman-16/proton-cli)) all share the same disqualifying
shape for a 24/7 unattended service: they need the account's **full password + 2FA** (there is **no
scoped token** — a leak exposes Mail, Drive, and Pass too, not just Calendar), they hit a **login
CAPTCHA / human-verification step** an unattended process cannot solve, and they run against an
**undocumented internal API Proton can change without notice**. Recommendation: **nice-to-have,
leaning drop-if-costly** — take the read-only ICS Full-view feed if the user already pays for
Proton; otherwise drop Proton to manual, and never build the hub on a reverse-engineered client.

## Access paths

Repo metadata checked via GitHub API on 2026-08-22 (pushed date / archived / stars / license).

| Path | Read? | Write? | Needs password/2FA? | ToS-ok? | Break risk | Maintenance |
|---|---|---|---|---|---|---|
| **Outbound ICS share link** (paid plan, "Share with anyone" → Create link) | Yes — Full view (title, description, location, participants) or Limited (busy only) | **No** (read-only: "Only you can create and edit events") | No — the secret URL is the only credential | **Yes** — Proton's own sanctioned feature | **Low** — a documented product feature | Proton-maintained |
| **Inbound ICS subscribe** ("Add calendar from URL", into Proton) | Into Proton, view-only | No programmatic write | No | Yes | Low | Proton-maintained |
| **`ProtonMail/go-proton-api`** (official Go lib, subset of internal REST API) | Calendar **read only** in the files read (`calendar.go`, `calendar_event.go`) — returns *encrypted* blobs | **No** calendar-event Create/Update/Delete in those files | Yes — SRP username+password + full PGP key hierarchy to decrypt; login `hv.go` = human-verification | **Gray** — internal API, unattended = "distinguishable from human users" (ToS §2.10) | **High** — internal API, no stability promise | Active (pushed 2026-08-14, 268★, MIT) but "not actively looking for contributors" |
| **`roman-16/proton-cli`** (unofficial Go client) | Calendar **read** yes | Calendar **write** yes — create/update/delete/RSVP *(README-claimed; not executed)* | Yes — SRP + 2FA + full local PGP key unlock (user→address→calendar) | **Gray** — reverse-engineered client, ToS §2.10 | **High** — undocumented internal API "may change" (its own disclaimer) | Active (pushed 2026-08-19, 52★, MIT); independent, "not endorsed by Proton AG" |
| **`Nojuza/proton-calendar-cli`** (unofficial Python CLI + MCP) | Calendar **read** yes | Calendar **write** yes — full CRUD + PGP round-trip *(README-claimed; not executed)* | Yes — SRP + 2FA + **CAPTCHA handling** + persistent session | **Gray** — "reverse-engineers the web client's endpoints" | **High** — "the API is undocumented and may change" | Weak (pushed 2026-05-25, **1★, 1 commit, NO license** = all-rights-reserved, not reusable) |
| **`emersion/hydroxide`** | **No Calendar at all** — Mail (IMAP/SMTP) + Contacts (CardDAV) only | n/a | Bridge password | n/a | n/a | **Archived 2026-08-02** (2177★, MIT); moved to Codeberg, "casual maintenance" |
| **`ProtonMail/proton-python-client`** | **No calendar** — SRP/auth wrapper only | n/a | SRP | n/a | n/a | Auth-only lib (GitHub API returned 404 on 2026-08-22 — possibly moved into the `proton-core` family that third-party tools now depend on) *(unverified)* |

## Unofficial / reverse-engineered libraries — findings

**None of these is a safe foundation for a 24/7 self-hosted service, and one (`go-proton-api`) that
looks official is read-only for calendar.**

- **`ProtonMail/go-proton-api`** — Official, MIT-licensed, actively pushed (2026-08-14), and it
  really does contain calendar code. But the files read expose **read only**: `calendar.go` has
  `GetCalendars`, `GetCalendar`, `GetCalendarKeys`, `GetCalendarMembers`, `GetCalendarPassphrase`;
  `calendar_event.go` has `CountCalendarEvents`, `GetCalendarEvents`, `GetAllCalendarEvents`,
  `GetCalendarEvent` — **no Create/Update/Delete for calendar events in those files**
  ([go-proton-api](https://github.com/ProtonMail/go-proton-api)). Even Proton's *own* Go library
  treats calendar as read-oriented, and the data it returns is ciphertext plus key/passphrase
  material you must decrypt with the user's PGP hierarchy. The README states it is "maintained by
  Proton AG, and is not actively looking for contributors" — it is the internal library its own
  apps (e.g. Bridge) build on, published, not a supported third-party SDK. Two separate legal
  instruments must not be conflated: **the MIT license grants rights to *use the code*; it grants
  nothing about *permission to access the Proton service*** (that is governed by the ToS below).

- **`roman-16/proton-cli`** — The most viable unofficial path, and the only one that plausibly does
  full read **and** write on Proton Calendar. Go, MIT, 52★, pushed 2026-08-19 (active). It does
  "SRP login and the full PGP key hierarchy, handled locally" with Proton's own `go-srp` and
  `gopenpgp` — "No bridge, no proxy, no browser in the middle"
  ([proton-cli](https://github.com/roman-16/proton-cli)). README examples claim list/create/update
  events, recurrence, delete-occurrence, and RSVP. **Not executed here** (would require the user's
  real credentials), so the write capability is *(unverified — README-claimed)*. It is explicitly
  "an independent, community-built project… not endorsed by, affiliated with, or supported by
  Proton AG," offered "use at your own risk."

- **`Nojuza/proton-calendar-cli`** — Python 3.11+ CLI + MCP server that "reverse-engineers the web
  client's endpoints and reproduces the client-side PGP key hierarchy," with full CRUD and a "full
  PGP encrypt/decrypt round-trip," SRP + 2FA + **CAPTCHA handling** + persistent session
  ([proton-calendar-cli](https://github.com/Nojuza/proton-calendar-cli)). Its own README states the
  reason no official path exists: "there's no official Proton Calendar API or CalDAV support because
  of the end-to-end encryption," and "Proton may change their internal API at any time." But it is
  **essentially unproven and unmaintainable as a dependency**: 1★, a single commit (pushed
  2026-05-25), and **no LICENSE file** — meaning default copyright / all-rights-reserved, so you
  cannot legally vendor it. Depends on `proton-core`, which is "not available on PyPI."

- **`emersion/hydroxide`** — **Does not touch Calendar.** Mail (IMAP/SMTP) and Contacts (CardDAV)
  only ([hydroxide](https://github.com/emersion/hydroxide)). It was **archived 2026-08-02** (now
  read-only on GitHub, migrated to Codeberg, "casual maintenance intended"). Irrelevant to this
  project regardless of maintenance state.

- **`ProtonMail/proton-python-client`** — Official but an **authentication/SRP wrapper only, no
  calendar functionality**; third-party calendar tools build *on top of* it (or its successor
  `proton-core`). The GitHub API returned 404 for it on 2026-08-22, suggesting it was
  moved/renamed/absorbed into the `proton-core` family *(unverified)*. `brandonland/proton-calendar`
  (surfaced in search as another Python calendar attempt) also returned 404 — gone or private.

**Do they need the user's password/2FA?** Yes, all of them — see the E2E section for why this is
structural, not incidental. **Do they violate the ToS?** Not by a blanket ban, but by behavior — see
next. **How likely to break?** High: every one runs against an undocumented internal REST API that
Proton changes at will, and each says so itself.

### Terms of Service — the precise verdict

Proton's Terms of Service **§2.10** prohibits "Accessing the Services through automated means
(including but not limited to bots, scripts, or similar technologies) **in a manner that is
distinguishable from the standard client behavior of human users**, that deviates significantly from
normal usage patterns, that exhibits characteristics of abuse, or that attempts to circumvent the
Company's security controls," with a carve-out immediately after: "automated access to the Services
**is permitted provided that the resulting traffic remains indistinguishable from the standard
client behavior of human users**" ([proton.me/legal/terms](https://proton.me/legal/terms)). No
explicit reverse-engineering clause appears in the Terms read.

**Consequence for this project:** the test is *behavioral indistinguishability*, not a flat ban. A
**24/7 unattended service** polling on a fixed schedule from a **datacenter IP**, holding a
persistent session, is plausibly "distinguishable from the standard client behavior of human users"
and "deviates from normal usage patterns" — i.e. **on the wrong side of §2.10**. So the ToS status of
the unofficial clients is best stated as **gray, trending prohibited for a server**, and users who
trip §2.10 are also "ineligible for support." The sanctioned share link (below) has **no** such
problem — it is a product feature.

## Verified ICS details (correcting the survey's *(unverified)* notes)

Sources: [share-calendar-via-link](https://proton.me/support/share-calendar-via-link),
[subscribe-to-external-calendar](https://proton.me/support/subscribe-to-external-calendar),
[how-to-import-calendar-to-proton-calendar](https://proton.me/support/how-to-import-calendar-to-proton-calendar).

- **Outbound share-link freshness — Proton sets the floor, not your poll interval.** Proton says
  when you change events, subscribers "may take up to **eight hours** before they see the changes,"
  *plus* the subscribing app's own refresh schedule on top
  ([share-calendar-via-link](https://proton.me/support/share-calendar-via-link)). So **polling the
  ICS URL every five minutes does not make Proton data fresher than ~8h.** There is no guaranteed
  tighter interval. (For the reverse direction, Proton's own inbound subscribe refreshes every
  **4–16 hours** — [subscribe-to-external-calendar](https://proton.me/support/subscribe-to-external-calendar).)

- **What "Full view" exposes.** Full view shows "all the event details": **Title, event
  description, Event participants, and Location** — not just title+time. Limited view "shows when
  you're busy but doesn't show any event details"
  ([share-calendar-via-link](https://proton.me/support/share-calendar-via-link)). So **Full view
  gives the unified view real, meaningful events**, not opaque busy blocks.

- **Share-link security.** The link is **read-only** ("Anyone with the shared calendar link can view
  the calendar. Only you can create and edit events"). It is **revocable** (Actions → Delete link;
  "other calendars using that link will no longer be able to sync"). You can create **up to five
  links per calendar** with different access levels/labels. **No password-protection option is
  documented**, and **no link expiry is documented**. Whether the token is cryptographically
  unguessable is **not stated** *(unverified)* — but operationally the URL is a **bearer credential**:
  possessing it grants full event details including attendees, with no password and no documented
  expiry. It must be handled as a secret in the hub's config.

- **Size limit — corrected, and the direction matters.** The survey's "~1 MB" is **not supported by
  any Proton page found; treat it as wrong.** The documented cap is **10 MB and 15,000 events**, and
  it is on the **import** page (one-time `.ics` upload):
  "the file needs to have a maximum size of 10 MB" and "you can import a maximum of 15,000 events at
  a time" ([how-to-import-calendar-to-proton-calendar](https://proton.me/support/how-to-import-calendar-to-proton-calendar)).
  The **subscribe-by-URL** page documents only a "too large to be imported" *failure* plus a
  workaround (append `?start-min=YYYY-MM-DDT00:00:00Z` to trim history) and gives **no numeric cap**
  *(subscribe-URL numeric cap unverified)*
  ([subscribe-to-external-calendar](https://proton.me/support/subscribe-to-external-calendar)).
  **Crucially, this cap governs `hub → Proton`** (the user subscribing the hub's feed *inside*
  Proton for display) — **it does not constrain `Proton → hub`**, the recommended mechanism, at all.

- **Which plans.** Sharing via link requires "**a paid Proton plan**"; free accounts cannot create
  share links ([share-calendar-via-link](https://proton.me/support/share-calendar-via-link)). The
  page gives no per-plan breakdown, so *which* paid tiers is **(unverified)** — verify against the
  user's actual subscription. **This is a hard precondition: on a free plan the sanctioned outbound
  path does not exist, and the recommendation collapses to drop-or-manual.**

## The E2E-encryption constraint — why there is no server-side API token

**Every field a calendar app cares about is end-to-end encrypted.** Event "title, description,
location, and attendees" are stored "with end-to-end encryption," using a per-calendar **ECC
Curve25519 PGP key** plus **two 32-byte session keys per event** (a shared session key and a
calendar session key), with address keys for signing invitations
([protoncalendar-security-model](https://proton.me/blog/protoncalendar-security-model)). Proton's
servers hold **ciphertext only** for normal events.

**This is why a Google-style server-side API token is impossible.** Google can mint a scoped OAuth
token because Google's servers can read your events and answer an API call server-side. Proton's
servers *cannot* — they never possess the plaintext or the calendar private key. The only way to
read the events programmatically is to **derive the PGP key hierarchy on the client**, which
requires the **user's password** (via SRP) to unlock user → address → calendar keys and decrypt
client-side. That is exactly why every unofficial client above needs the full password + 2FA and
reproduces "the key unlocking chain (user → address → calendar)" — it is structural, not a shortcut.
There is no narrower credential Proton could hand out even if it wanted to.

**The public ICS share link is the sanctioned escape hatch precisely because Proton deliberately
drops E2E for it.** Proton states the contrast in its own words: "Calendars shared with other Proton
users benefit from end-to-end encryption, **unlike those shared via a public link**," and "you can
share it with Google Calendar users… but your recipient's calendar provider **won't maintain our
end-to-end encryption**" ([keep-everyone-in-the-loop-with-shared-calendars](https://proton.me/blog/keep-everyone-in-the-loop-with-shared-calendars)).
So a **keyless third party (Google, or this hub) reads the feed with no Proton key** — which means
**plaintext of those events exists outside the user's client**, on the share-link path. That is the
whole reason the link works as plain ICS. Whether Proton decrypts server-side to serve it, or the
client uploads a plaintext feed when the link is created, is **not documented** *(unverified)* — but
either way the E2E guarantee does not cover that feed. (The security-model and `/calendar/security`
pages describe only Proton-to-Proton and email-invite flows and are silent on public links, so the
blog quotes above are the load-bearing primary source.)

## Recommendation for this project

**Verdict: nice-to-have, leaning drop-if-costly. Never a must-have via an unofficial client.** Frame
the three options as consequences, not preferences:

**Drop-in mechanism (the one realistic path):** In Proton (paid plan), publish the calendar as a
**read-only ICS share link**, **Full view**. In the hub, **subscribe to that URL** and treat those
events as **display + busy input only**. Compute booking availability from the union of all four
accounts' busy time with Proton included as busy-only input, and **write confirmed bookings to a
Gmail or Microsoft destination calendar — never Proton** (the read-many / write-one pattern from the
survey). The only way to get an event *into* Proton from the hub is out-of-band: **email an iCalendar
invite** that the user accepts inside Proton's own client. This is identical to the survey's
conclusion; nothing found here changes it.

**Concrete failure mode to design around:** because Proton's own freshness floor is ~8h, a Proton
event created 10 minutes ago is **invisible to the hub**, so **the booking link can double-book the
user against fresh Proton events.** Mitigations, in order of honesty: (a) accept it and treat Proton
as low-churn "committed" events only; (b) have the user manually mirror any urgent Proton block into
a writable Gmail/Microsoft calendar so the hub sees it immediately; (c) do not promise real-time
conflict-freedom for Proton. Tight polling does **not** fix this.

**Must-have** (full programmatic read+write in Proton) is satisfiable *only* through an unofficial
reverse-engineered client, so choosing it *means* accepting all of:
- the account **password + 2FA sitting in the service config**, with **no scoped token** — a leak
  compromises **Mail, Drive, and Pass too**, not just Calendar;
- a **login CAPTCHA / human-verification** step that **no unattended process can reliably solve**
  (Nojuza documents CAPTCHA handling; `go-proton-api` ships `hv.go`) — this, plus **silent death on
  session expiry**, is the **biggest risk**, sharper than the generic "the API may change";
- **ToS §2.10 exposure** for datacenter-based automated access distinguishable from human use;
- a dependency on a hobby repo (1–52★) with no stability promise and, for Nojuza, **no license**.
For a 24/7 self-hosted hub this is **not acceptable**. Do not build on it.

**Nice-to-have** (the recommended drop-in) buys **read-only Full view at ~8h staleness with zero
credential exposure** and only requires that the user **already has a paid Proton plan** and pastes a
**secret share URL** (a bearer credential — store it as a secret) into the hub.

**Drop** (omit Proton entirely, or reduce it to the manual email-invite path) saves the paid-plan
dependency and the secret-URL handling; it costs Proton visibility in the unified view and the
booking safety the coarse busy feed would have added. On a **free** Proton plan this is effectively
the only remaining option.

**Bottom line:** take the paid-plan **outbound ICS Full-view feed, read-only, busy+display input
only** if the user already pays for Proton; otherwise drop Proton to manual. Route no bookings to
Proton, and keep any reverse-engineered client out of a long-running server. This refines — does not
replace — the existing survey's conclusion.

---

*Primary sources cited inline. Fastest-rotting facts: account/plan support and the internal-API
surface of the unofficial clients — re-verify before committing. See
`docs/research/existing-tools-survey.md` for the broader aggregator/booking landscape this builds on.*
