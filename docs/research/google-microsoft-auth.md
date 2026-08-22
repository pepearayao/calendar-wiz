# Google Calendar API & Microsoft Graph: Auth and Capabilities

## Scope & date

Research date: **2026-08-22**. This document goes deep on **authentication** and
**capabilities** for the two API-native providers in the self-hosted **Kotlin** calendar hub
(up 24/7, run by **one** individual): **2 personal Gmail accounts** via the Google Calendar API,
and **1 Microsoft work/Teams account** via Microsoft Graph. It builds on — and does **not**
repeat — the incremental-sync mechanics and target architecture already established in
`docs/research/existing-tools-survey.md` (Google `nextSyncToken`; Graph `/me/calendarView/delta`;
webhooks carry no data and expire) and `docs/research/proton-access.md`. All non-obvious claims
cite the primary source that owns them; unconfirmable claims are marked **(unverified)**. No real
secrets or tokens appear here.

## Executive answer

**Both providers give full read + create/update/delete + RSVP + free/busy through documented,
first-class API methods** — the capability question is a clean "yes" for both (table below). The
**auth** question is where the risk lives, and it splits cleanly by provider.

- **Google (2 Gmail accounts):** one OAuth client in one Google Cloud project; each Gmail account
  runs its own consent. The single decision that matters for 24/7 operation is **publishing
  status**. In **"Testing"**, refresh tokens die after **7 days** — fatal for an unattended hub.
  The fix is to **publish the app to "Production" and simply leave it unverified**: Google
  explicitly supports a developer clicking through the "unverified app" screen for an app used only
  by themselves or people they know, the 7-day expiry disappears, and you stay under the 100-user
  cap forever with two accounts. You do **not** need to complete Google's app-verification for
  single-user personal use. Calendar scopes are **sensitive, not restricted** (no CASA/security
  assessment). In Production a refresh token then lives until 6-month inactivity, with a 100-token
  cap per account per client.
- **Microsoft (1 work account) — the load-bearing risk:** the model is **delegated permissions
  (`Calendars.ReadWrite` + `offline_access`) with a refresh token**, and by *default* a non-admin
  user can both register the app and self-consent (Calendars permissions do **not** require admin
  consent by default). **But the work-tenant admin holds three independent kill switches**:
  (1) turn off "Users can register applications"; (2) set user-consent to **"Do not allow user
  consent"**, which forces admin consent on **every** app including this one; (3) a **Conditional
  Access** policy (require compliant/managed device, approved client app, or sign-in frequency)
  that a headless datacenter server cannot satisfy. **Lever 2 has no workaround** — it is enforced
  in the work tenant against the work identity, so registering the app in your own tenant does not
  escape it. **Worst case is worse than Proton:** if the admin locks it down, the only degraded
  path is the read-only **Outlook published-ICS link** — and admins can (and on work tenants often
  do) disable that too, leaving **nothing programmatic at all** (manual only). So Microsoft is
  "full read/write if the admin permits, read-only-like-Proton if you're lucky, zero if you're
  not."

---

## Capabilities table (provider × operation, with the API method)

| Operation | Google Calendar API (2 Gmail) | Microsoft Graph (1 work account) |
|---|---|---|
| **Read events** | `events.list` — `GET /calendars/{id}/events`; returns `nextSyncToken` for incremental sync ([events.list](https://developers.google.com/workspace/calendar/api/v3/reference/events/list), [sync guide](https://developers.google.com/workspace/calendar/api/guides/sync)) | `GET /me/events` / `GET /me/calendarView`; delta for sync ([list events](https://learn.microsoft.com/en-us/graph/api/user-list-events), [delta](https://learn.microsoft.com/en-us/graph/delta-query-events)) |
| **Create** | `events.insert` — `POST /calendars/{id}/events` ([events.insert](https://developers.google.com/workspace/calendar/api/v3/reference/events/insert)) | `POST /me/events` ([create event](https://learn.microsoft.com/en-us/graph/api/user-post-events)) |
| **Update** | `events.update` / `events.patch` — `PUT` / `PATCH /calendars/{id}/events/{id}` ([events.patch](https://developers.google.com/workspace/calendar/api/v3/reference/events/patch)) | `PATCH /me/events/{id}` ([update event](https://learn.microsoft.com/en-us/graph/api/event-update)) |
| **Delete** | `events.delete` — `DELETE /calendars/{id}/events/{id}` ([events.delete](https://developers.google.com/workspace/calendar/api/v3/reference/events/delete)) | `DELETE /me/events/{id}` ([delete event](https://learn.microsoft.com/en-us/graph/api/event-delete)) |
| **RSVP (own attendance)** | `events.patch` writing the user's `attendees[].responseStatus` (`accepted` / `declined` / `tentative` / `needsAction`; the user's own entry is flagged `attendees[].self`, read-only) ([events resource](https://developers.google.com/workspace/calendar/api/v3/reference/events)) | dedicated actions `POST /me/events/{id}/accept` \| `/decline` \| `/tentativelyAccept` ([accept](https://learn.microsoft.com/en-us/graph/api/event-accept), [decline](https://learn.microsoft.com/en-us/graph/api/event-decline), [tentativelyAccept](https://learn.microsoft.com/en-us/graph/api/event-tentativelyaccept)) |
| **Free/busy** | `freebusy.query` — `POST /freeBusy` ([freebusy.query](https://developers.google.com/workspace/calendar/api/v3/reference/freebusy/query)) | `POST /me/calendar/getSchedule` ([getSchedule](https://learn.microsoft.com/en-us/graph/api/calendar-getschedule)) |

**Verification notes.** All Google and Microsoft method pages above were fetched directly.
Microsoft `decline` / `tentativelyAccept` are the sibling actions of `accept` (verified directly)
and carry the identical `Calendars.ReadWrite` requirement. Google RSVP has a documented caveat:
setting `responseStatus` to `accepted`/`declined`/`tentative` when *adding* an event can be reset
to `needsAction` for attendees with certain email settings ([events resource](https://developers.google.com/workspace/calendar/api/v3/reference/events)).
Microsoft `getSchedule` is **"Not supported" for personal Microsoft accounts** — irrelevant here
because the Teams account is a work account ([getSchedule](https://learn.microsoft.com/en-us/graph/api/calendar-getschedule)).

---

## Google Calendar API — auth (for the 2 Gmail accounts)

### Scopes

For **full read + write + RSVP + free/busy**, the single broad scope
`https://www.googleapis.com/auth/calendar` ("See, edit, share, and permanently delete all the
calendars you can access") covers every operation above. A narrower combination that still does
everything the hub needs is `.../auth/calendar.events` (read/write events, including RSVP) plus
`.../auth/calendar.freebusy` (free/busy). Read-only variants exist
(`calendar.readonly`, `calendar.events.readonly`). Exact scope strings are enumerated on the
[Calendar API auth page](https://developers.google.com/workspace/calendar/api/auth); each write
method (`insert`/`patch`/`delete`) accepts `calendar` or `calendar.events`, and `freebusy.query`
accepts `calendar`, `calendar.readonly`, `calendar.freebusy`, or `calendar.events.freebusy`.

**Sensitive vs restricted.** Calendar scopes are **"sensitive," not "restricted."** Google's own
sensitive-scope page names "reading events stored in Google Calendar" as its example of a
*sensitive* scope ([sensitive-scope-verification](https://developers.google.com/identity/protocols/oauth2/production-readiness/sensitive-scope-verification)).
The **restricted** category is Gmail/Drive-class data and additionally requires an annual
**CASA** third-party security assessment ([restricted-scope-verification](https://developers.google.com/identity/protocols/oauth2/production-readiness/restricted-scope-verification));
no Calendar scope is in it. (One third-party article claimed Calendar write is "restricted" — that
is **wrong**; treat the [OAuth API Verification FAQ](https://support.google.com/cloud/answer/9110914)
as the definitive sensitive/restricted list to re-verify.)

### Unattended 24/7 auth — refresh tokens and publishing status

**Obtaining the refresh token.** Request `access_type=offline` in the authorization request; the
refresh token is returned only when this is set. Add `prompt=consent` to force a fresh refresh
token to be issued ([web-server OAuth](https://developers.google.com/identity/protocols/oauth2/web-server)).

**The 7-day trap (Testing status).** Google states plainly: *"A Google Cloud Platform project with
an OAuth consent screen configured for an external user type and a publishing status of 'Testing'
is issued a refresh token expiring in 7 days"* ([OAuth2 expiration](https://developers.google.com/identity/protocols/oauth2)).
The Audience page repeats it: *"Authorizations by a test user will expire seven days from the time
of consent. If your OAuth client requests an `offline` access type and receives a refresh token,
that token will also expire"* ([Manage App Audience](https://support.google.com/cloud/answer/15549945)).
A 7-day-expiring token is **incompatible with a 24/7 hub** — the service would silently die every
week and demand a manual re-consent.

**The fix — publish to Production and stay unverified.** Because these are **personal Gmail**
accounts, the app's user type is "External" (the "Internal" exemption is Google-Workspace-org only,
so it does not apply). The correct path is:

1. **Publish the app to "Production."** This removes the 7-day expiry for normal users: *"When
   transitioning to production, the 7-day token expiration for test users no longer applies"*
   ([Manage App Audience](https://support.google.com/cloud/answer/15549945)).
2. **Do not complete verification.** Because the app requests a sensitive scope and is unverified,
   each account sees the **"unverified app" warning screen once**, and clicks **Advanced → "Go to
   {app} (unsafe)"** to proceed. This click-through **still exists in 2026**, and Google explicitly
   blesses it for exactly this case: *"If you are the only user of your app or if your app is used
   by only a few users, all of whom are known personally to you, you … might be comfortable with
   advancing through the unverified app screen"* ([Unverified apps](https://support.google.com/cloud/answer/7454865)).
   An unverified Production app is capped at **100 new users total** (permanent) — irrelevant with
   two accounts.

So the one-person hub **avoids Google's formal app-verification entirely**; verification is only
needed to remove the warning screen, show branding, or exceed 100 users
([when verification is not needed](https://support.google.com/cloud/answer/13464323)).

**Refresh-token lifetime in Production.** Once in Production the refresh token persists, subject to
two documented limits ([OAuth2 expiration](https://developers.google.com/identity/protocols/oauth2)):

- **6-month inactivity:** a refresh token unused for six months expires. A 24/7 hub polling every
  few minutes never trips this.
- **100 tokens per Google Account per OAuth client ID:** *"creating a new refresh token
  automatically invalidates the oldest refresh token without warning."* The hub should store **one**
  refresh token per account and reuse it, not mint new ones each restart.
- Other invalidations: user revokes access, or the account password changes (password change
  invalidates Gmail-scope tokens; treat any 400 `invalid_grant` as "re-consent needed").

### Per-account setup (2 separate Gmail accounts)

Both Gmail accounts use **the same Cloud project and the same OAuth client ID**; each account
simply runs its own consent flow and yields its **own independent refresh token**. The 100-token
cap is per-account-per-client, so two accounts never interfere. **Gotcha:** while the app is in
Testing, every account must be on the **test-users list** (max 100) — and each such authorization
still expires in 7 days. Publishing to Production removes both the test-user-list requirement and
the 7-day expiry, which is the second reason to publish.

---

## Microsoft Graph — auth (for the 1 work/Teams account), with admin-blocking analysis

### Permission model: delegated + refresh token

For a personal tool acting **as** the user 24/7, the model is **delegated permissions** (act on
behalf of the signed-in user), **not** application permissions (application permissions are
app-only, admin-consent-only, and meant for daemons accessing many users — the wrong shape here)
([permissions-consent-overview](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview)).

Scopes needed:

- **`Calendars.ReadWrite`** (delegated) — covers create/update/delete and the accept/decline/
  tentativelyAccept RSVP actions. `getSchedule` free/busy needs only `Calendars.ReadBasic`, which
  `ReadWrite` subsumes. Verified in each method's permission table (create/update/delete/accept all
  list `Calendars.ReadWrite` as least-privileged for a work/school delegated call).
- **`offline_access`** — **required to receive a refresh token.** *"On the Microsoft identity
  platform … your app must explicitly request the `offline_access` scope, to receive refresh
  tokens"* ([scopes-oidc](https://learn.microsoft.com/en-us/entra/identity-platform/scopes-oidc)).
- Plus `openid`/`profile` for sign-in as needed.

Access tokens last ~1 hour; the refresh token is *"typically valid for 90 days"* and rolls forward
as it is used, so an actively-polling hub stays alive — **unless** a tenant policy shortens it
(see Conditional Access below) ([scopes-oidc](https://learn.microsoft.com/en-us/entra/identity-platform/scopes-oidc)).

**By default this all works for a non-admin user.** Delegated `Calendars.Read`, `Calendars.ReadBasic`,
and `Calendars.ReadWrite` each have **"Admin consent required = No"** ([permissions reference](https://learn.microsoft.com/en-us/graph/permissions-reference)),
and member users can register applications by default ([default user permissions](https://learn.microsoft.com/en-us/entra/fundamentals/users-default-permissions)).
The risk is entirely that **the admin has changed those defaults.**

### The three admin kill switches (the load-bearing risk)

The Microsoft account is a **work tenant the user does not administer.** Any **one** of these three
admin settings, if turned on, blocks the project. They are independent — clearing one does not
clear the others.

| # | Lever | Default | What the admin does | Real workaround? |
|---|---|---|---|---|
| **1** | **App registration** | Users **can** register apps | Set **"Users can register applications" = No**; then only the Application Developer role or an admin can register ([default user permissions](https://learn.microsoft.com/en-us/entra/fundamentals/users-default-permissions)) | **Partial.** Register the app in **a tenant you control** (a free personal Entra tenant) as a **multitenant** app; the work user then signs in to it. This solves registration only. |
| **2** | **User consent** | Calendars perms need **no** admin consent → user self-consents | Set user-consent to **"Do not allow user consent"** → *"Users can't grant permissions to applications … Only users who are granted a directory role … can consent"* — this forces admin consent for **every** app, including this one ([user & admin consent](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/user-admin-consent-overview)) | **None.** Consent is evaluated **in the work tenant against the work identity**: *"User consent by nonadministrators is possible only in organizations where user consent is allowed."* Registering elsewhere does not help. The user can only file an **admin-consent request** and wait for approval ([admin consent workflow](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-admin-consent-workflow)). |
| **3** | **Conditional Access** (P1) | none | Require a **compliant/hybrid-joined device**, an **approved client app**, block by **IP/location**, or enforce **sign-in frequency** ([Conditional Access overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview)) | **Mostly none.** A headless datacenter server has no managed device, no interactive MFA path, and a fixed datacenter IP — it fails "require compliant device"/"approved app", and a short sign-in-frequency forces interactive re-auth that breaks unattended refresh. |

**Why lever 2 is the decisive one.** Even the standard fallback — registering the app in your own
tenant so *you* control registration (lever 1) — does not escape lever 2, because when the work
user signs in and consents, that consent is governed by **the work tenant's** user-consent policy,
not your tenant's. If the work admin has disabled user consent, the work user cannot self-consent
to *any* third-party app regardless of where it was registered. This is the single fact the
build/no-build decision turns on.

### Fallbacks if the tenant is locked down

- **Register in a tenant you control (multitenant app).** Fixes lever 1 only. Does **not** bypass
  lever 2 or 3. The audience "Accounts in any Microsoft Entra directory" lets a work user sign in
  to an app you registered ([single vs multitenant](https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps)),
  but the work tenant's consent and Conditional-Access policies still apply to that sign-in.
- **B2B guest into your own tenant — does not work for this.** Inviting the work email as a **guest**
  in your personal tenant gives that guest identity access to **your** tenant's resources, not to
  the **work** mailbox/calendar. Delegated Graph calls (`/me/events`, `/me/calendar/getSchedule`)
  resolve to the user's **own mailbox**, which lives in the **work tenant** under the work tenant's
  control; a guest has no mailbox in your tenant. So this does not read the work calendar.
  *(Inferred from the delegated-access model — [permissions-consent-overview](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview); not a single owning doc.)*
- **Outlook published-ICS link (the Proton-style degraded path).** The work Outlook calendar can be
  *published* as a read-only ICS/HTML URL that any app can subscribe to ([share calendar in Outlook
  on the web](https://support.microsoft.com/en-us/office/share-your-calendar-in-outlook-on-the-web-7ecef8ae-139c-40d9-bae2-a23977ee58d5)).
  This gives **read-only** event/busy data **only** — no create, no edit, no RSVP, no `getSchedule`.
  **And it is admin-controllable:** internet calendar publishing is governed by the Exchange Online
  sharing policy and can be disabled tenant-wide ([disable internet calendar publishing](https://learn.microsoft.com/en-us/exchange/disable-internet-calendar-publishing-exchange-2013-help)),
  and it is *"commonly disabled on work and school accounts"* (search-sourced, treat the specific
  "commonly" as **(unverified)**). So this fallback may itself be blocked.

### Personal vs work/school accounts

Microsoft **personal** accounts (outlook.com/Xbox/Skype) have **no tenant admin** — none of the
three levers exist, so a personal account is unblockable by an admin. But `getSchedule` free/busy
is **"Not supported"** for personal accounts ([getSchedule](https://learn.microsoft.com/en-us/graph/api/calendar-getschedule)),
so you would compute busy from listed events instead. This project's account is a **work/Teams**
account, so `getSchedule` works but the admin levers apply. ("Teams" ⇒ work/school tenant.)

---

## Per-provider setup checklist (for this one user)

### Google (repeat for each of the 2 Gmail accounts)

1. Create **one** Google Cloud project; enable the **Google Calendar API**.
2. Configure the **OAuth consent screen** (External user type); add scopes `.../auth/calendar`
   (or `calendar.events` + `calendar.freebusy`).
3. Create **one OAuth client ID** (Desktop or Web, per your redirect strategy).
4. **Publish the app to "Production"** (Audience page). Do **not** submit for verification.
5. For **each** Gmail account: run the consent flow with `access_type=offline` (+ `prompt=consent`);
   at the **"unverified app"** screen click **Advanced → Go to {app} (unsafe)**; store the returned
   **refresh token** as a secret (one per account).
6. Nothing here needs an administrator — personal Gmail accounts are self-administered.

### Microsoft (the 1 work account)

1. **Try to register an app** in the work tenant (Entra ID → App registrations). *If blocked* →
   lever 1 is on; register a **multitenant** app in a **personal Entra tenant you create** instead.
2. Add **delegated** Microsoft Graph permissions **`Calendars.ReadWrite`** + **`offline_access`**
   (+ `openid`/`profile`). Add a redirect URI; create a client secret if using a confidential client.
3. Run the auth-code flow as the work user. **If the consent prompt says approval is required / you
   cannot consent** → lever 2 is on (admin disabled user consent); submit an **admin-consent
   request** and wait — there is no self-service path.
4. **If sign-in is blocked or repeatedly forces interactive re-auth** on the server → lever 3
   (Conditional Access) is on; likely unsolvable from a headless host.
5. If all three pass, store the **refresh token** as a secret; it rolls forward with use (~90-day
   window).

**What could require an admin the user does not have:** app registration (lever 1), consent
(lever 2), and Conditional-Access exceptions (lever 3) — **any one** can require the work admin.

## Worst case for Microsoft

If the admin has locked down consent (lever 2) and there is no cooperative admin to approve a
request, **the API path is dead** — no amount of re-registering in another tenant recovers it. The
only remaining option is the **read-only Outlook published-ICS link**, which drops Microsoft to a
**Proton-like degraded input**: busy/display in, **no write, no RSVP, no free/busy API**. And
because that ICS publishing is itself an admin-toggled Exchange setting, the true worst case is
**nothing at all** — the work calendar becomes a manual, out-of-band input, exactly like the
Proton "email an invite a human accepts" path. So route **all booking writes to a Gmail
destination** and treat Microsoft as potentially read-only-or-less, not as a guaranteed writable
calendar.

---

## Fastest-rotting facts — re-verify immediately before building

These change often; confirm each against its cited page at build time:

1. **Google publishing-status / token behavior (HIGHEST volatility).** The "Testing = 7-day
   refresh token" rule and the "Production-unverified click-through + 100-user cap" behavior have
   changed more than once. Re-verify [OAuth2 expiration](https://developers.google.com/identity/protocols/oauth2),
   [Manage App Audience](https://support.google.com/cloud/answer/15549945), and
   [Unverified apps](https://support.google.com/cloud/answer/7454865) before relying on the
   no-verification path.
2. **Google refresh-token limits** — the 6-month inactivity window and the 100-token-per-account
   cap ([OAuth2 expiration](https://developers.google.com/identity/protocols/oauth2)).
3. **Google sensitive/restricted scope list** — confirm Calendar is still *sensitive* via the
   [OAuth API Verification FAQ](https://support.google.com/cloud/answer/9110914).
4. **Entra defaults** — "Users can register applications" and the default user-consent setting can
   be changed by Microsoft or by the tenant at any time ([default user permissions](https://learn.microsoft.com/en-us/entra/fundamentals/users-default-permissions),
   [user & admin consent](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/user-admin-consent-overview)).
5. **Graph refresh-token / Conditional-Access interaction** — the ~90-day refresh window and how
   sign-in-frequency policies shorten it ([scopes-oidc](https://learn.microsoft.com/en-us/entra/identity-platform/scopes-oidc)).
6. **Graph permission requirements per method** — re-check each method's permission table; Microsoft
   occasionally re-tiers permissions ([permissions reference](https://learn.microsoft.com/en-us/graph/permissions-reference)).

---

*Builds on `docs/research/existing-tools-survey.md` (sync mechanics, read-many/write-one booking)
and `docs/research/proton-access.md` (Proton degraded path). Primary sources cited inline; the
Google verification/publishing policy is the single most volatile input and must be re-verified
before implementation.*
