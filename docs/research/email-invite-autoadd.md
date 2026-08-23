# Emailed iCalendar Invites: Do Receiving Calendars Auto-Add?

## Scope & date

Research date: **2026-08-23**. This document answers one operational question for the self-hosted
**Kotlin** calendar hub: when the hub **emails an iCalendar invite** (iMIP `METHOD:REQUEST` /
`METHOD:CANCEL`) from a dedicated hub identity, does the **receiving** calendar **auto-add** the
event, or must the user click "accept" every time? Auto-add is the precondition for an unattended
hub write path — it is the fallback for **Proton** (no write API, per
`docs/research/proton-access.md`) and for **Microsoft/Teams** when the tenant admin blocks Graph
(per `docs/research/google-microsoft-auth.md`). Targets: **Proton Calendar** (paid account) and the
**Outlook / Exchange Online user mailbox** (the work/Teams account). All non-obvious claims cite the
primary source that owns them; unconfirmable claims are marked **(unverified)**. No secrets appear.

## Executive verdict

**Both targets auto-add an emailed invite without any user click — but each adds it in a
non-committed state, not as an accepted event.** Proton adds it **"pending"**; Exchange (user
mailbox default `AutomateProcessing=AutoUpdate`) adds it **"tentative."** So the event **appears on
the calendar automatically**; what still requires a human is turning pending/tentative into
*accepted* (which the hub does not need for display/busy purposes). The important asymmetry is the
kill switch. Proton's is a **user** setting — auto-add of invitations is **on by default** and the
user can toggle it off, so the hub's only precondition is "the user did not disable it." Exchange's
tentative auto-processing is **not admin-disableable on a user mailbox** per the cmdlet doc's own
sentence ("you can't change the value on a user mailbox"), but two adjacent facts still gate the
hub: (1) the hub is an **external** sender, and the Exchange doc says external meeting requests are
**rejected by default** (`ProcessExternalMeetingMessages $false`) — whether that applies to a
*user* mailbox is a genuine documentary gap; and (2) an admin **can** grant true auto-**accept** to
a designated sender via a **Direct to Calendar** mail flow rule. Net: documentation settles that
auto-add-as-tentative/pending is the default design on both platforms; the one thing that genuinely
needs a **live test** is whether an *external* organizer's invite lands on the specific work
mailbox's calendar, and whether Proton auto-adds an invite whose organizer is the hub.

## At-a-glance table

| Target | Auto-add on create (no click)? | Update (SEQUENCE bump)? | Cancel? | Admin/user-disableable? | Primary source |
|---|---|---|---|---|---|
| **Proton Calendar** (paid) | **Yes — added automatically, marked "pending"** ("Event invites that arrive in your Proton Mail inbox are now automatically added to your calendar") | **(unverified)** — support pages document the *organizer* sending updates, not recipient-side auto-update | **(unverified)** — organizer-side cancel documented; recipient-side removal/marking not documented | **User** toggle, **on by default**: Settings → All settings → General → **Invitations** | [auto-add-invites](https://proton.me/support/auto-add-invites), [add-event-from-inbox](https://proton.me/support/add-event-from-inbox) |
| **Outlook / Exchange Online user mailbox** | **Yes — added automatically as "tentative"** (default `AutomateProcessing=AutoUpdate`: "Meeting requests are tentative in the calendar until approval") | **Yes** — the Calendar Attendant updates the tentative item; iTIP: highest `SEQUENCE` obsoletes lower ones | **(unverified)** for ordinary iMIP CANCEL on a user mailbox; for Direct-to-Calendar the doc says it stays until manual removal | **Not** via `AutomateProcessing` on a user mailbox ("you can't change the value on a user mailbox"); user-side Outlook "Automatically process meeting requests" toggle and mail-flow rules are the levers | [Set-CalendarProcessing](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-calendarprocessing?view=exchange-ps) |

Caveat on the external hub sender is broken out in the Exchange section — it is the load-bearing
residual and does not fit a table cell.

---

## Proton Calendar

**Auto-add on create — yes, by default, marked "pending."** Proton's support page states directly:
*"Event invites that arrive in your Proton Mail inbox are now automatically added to your
calendar."* The feature is *"activated by default but can be disabled on the Proton Mail web app and
Android app,"* via **Settings → All settings → General (left sidebar) → Invitations**
([auto-add-invites](https://proton.me/support/auto-add-invites)). The same page describes a second,
finer control — users can uncheck **"Add to the calendar and mark as pending"** to stop pending
invitations from being added automatically — which confirms the auto-added event lands in a
**pending** (non-accepted) state, the direct analogue of Exchange's "tentative."

**No manual click is required for the event to appear; the click only sets your RSVP.** The inbox
UI offers **Yes / Maybe / No**, and *"Event invitations can be accepted or denied directly from
Proton Mail and are automatically added to your calendar"*
([add-event-from-inbox](https://proton.me/support/add-event-from-inbox)). Responding sends your
answer to the organizer; it is not what puts the event on the calendar.

**Hub design constraint — the invite must be a real REQUEST, not a bare .ics.** The same page draws
a fork: true **invitations** get the Yes/Maybe/No buttons **and** auto-add, whereas non-invitation
calendar mail (appointment reminders, booking confirmations) shows only a manual **"Add to Proton
Calendar"** button ([add-event-from-inbox](https://proton.me/support/add-event-from-inbox)). To land
on the **auto-add** path, the hub must emit a genuine iMIP `METHOD:REQUEST` with the recipient's
address in an `ATTENDEE` property and the hub identity as `ORGANIZER` — not merely attach an `.ics`
file. This is the difference between auto-add and a manual click.

**External (non-Proton) organizer.** Proton's invitation feature is designed to interoperate with
other providers — the (dated, **January 2021 beta**) announcement says it *"works with Proton
Calendar as well as other popular calendars"*
([blog](https://proton.me/blog/proton-calendar-event-invitations)). More usefully, the **current**
support pages impose **no organizer-address restriction** on auto-add; the only documented limit is
that *"You cannot respond to invitations forwarded to your Proton Mail inbox from another email
address"* ([add-event-from-inbox](https://proton.me/support/add-event-from-inbox)) — and that
limit is about **responding**, not about the event being **added**. So the docs do not contradict
auto-add for a hub-organized invite, but they also never explicitly confirm it for a third-party
organizer → treat "Proton auto-adds an invite organized by the hub" as **needs a live check**.

**Must arrive in Proton Mail.** Auto-add is scoped to invites *"that arrive in your Proton Mail
inbox"* ([auto-add-invites](https://proton.me/support/auto-add-invites)) — the hub must email the
user's Proton Mail address, not reach Proton Calendar directly (Proton has no inbound write API).

**Update / cancel on the receiving side — undocumented.** The page
[update-an-invitation-to-an-event](https://proton.me/support/update-an-invitation-to-an-event)
covers the **organizer** modifying/deleting an event ("All participants will immediately receive an
email notification with the new event details"; "all participants will receive an email to inform
them of the cancellation"), **not** what the recipient's calendar does with an incoming update or
CANCEL. The general [manage-events](https://proton.me/support/calendar/using-calendar/manage-events)
page is also silent on recipient-side auto-update/removal. So whether a same-`UID`/higher-`SEQUENCE`
REQUEST updates the pending event in place, and whether a CANCEL removes or marks it, is
**(unverified)** from Proton's documentation.

---

## Outlook / Exchange Online (work / Teams user mailbox)

**Auto-add on create — yes, as "tentative," by server default.** The authoritative cmdlet reference
defines the `AutomateProcessing` values and the user-mailbox default
([Set-CalendarProcessing](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-calendarprocessing?view=exchange-ps)):

- *"None: Calendar processing is disabled on the mailbox. Both the resource booking attendant and
  the Calendar Attendant are disabled…"*
- *"AutoUpdate: Only the Calendar Attendant processes meeting requests and responses. **Meeting
  requests are tentative in the calendar until approval** by a delegate…"*
- *"AutoAccept: Both the Calendar Attendant and resource booking attendant are enabled…"*
- **The decisive sentence:** *"The default value for user mailboxes is AutoUpdate, but **you can't
  change the value on a user mailbox**."*

So a normal user mailbox runs the **Calendar Attendant**, which puts an incoming meeting REQUEST on
the calendar as **tentative** before the user does anything. That is the auto-add the hub needs for
display/busy. **Room/resource** mailboxes differ: in Exchange Online, resource mailboxes created in
the EAC (or in PowerShell after 2018-11-15) default to **AutoAccept** — this is the well-known
"rooms auto-accept" default, and it does **not** apply to user mailboxes.

**Not admin-disableable via `AutomateProcessing` on a user mailbox.** Because the cmdlet doc says
the value can't be changed on a user mailbox, an admin cannot simply set `None` there to stop the
tentative auto-add. (Field reports of `Set-CalendarProcessing -AutomateProcessing None` succeeding on
user mailboxes exist but conflict with the doc — **(unverified)**; do not rely on them.) The real
levers are elsewhere: the **user's own** Outlook client toggle *"Automatically process meeting
requests and responses"* (File → Options → Mail → Tracking) suppresses client-side processing, and
admins can shape behavior with **mail flow (transport) rules**. This is a *different kind* of control
than Proton's single user toggle, but it sits in the same column: something can still turn the
behavior off.

**The load-bearing residual — the hub is an EXTERNAL sender.** The same cmdlet page documents
`ProcessExternalMeetingMessages`: *"$false: Meeting requests from external senders are rejected.
This value is the default."* If that default governs user mailboxes, an invite from the hub's
outside domain could be **rejected rather than auto-added** — even though internal invites are
processed. The documentation is genuinely ambiguous, and I mark it as a gap rather than adjudicate
it: many parameters on that page are explicitly qualified *"used only on resource mailboxes where
the AutomateProcessing parameter is set to AutoAccept,"* and `ProcessExternalMeetingMessages`
carries **no such qualifier** in the text — yet user mailboxes are locked to `AutoUpdate`, which by
the doc's own definition runs *only* the Calendar Attendant (not the resource booking attendant that
most external-processing logic is described around). Whether an external organizer's REQUEST
auto-adds tentatively to *this* user mailbox is therefore **the single thing that most needs a live
test**. (A Microsoft Q&A thread corroborates the "user mailboxes are stuck on AutoUpdate, can't
change it" point, but it is an Independent Advisor plus an AI-generated answer — **non-authoritative**
corroboration only:
[Q&A](https://learn.microsoft.com/en-us/answers/questions/5613252/clarification-on-automateprocessing-behavior-for-u).)

**True auto-ACCEPT is available with a cooperative admin — "Direct to Calendar."** Exchange Online
has a documented admin feature that upgrades a **designated sender's** invites from tentative to
fully added: *"The event is automatically added to the recipient's calendar without any action from
them. If the user received the meeting invitation, it's on their calendar."* It is configured with
two mail flow rules keyed on the sender, setting
`X-MS-Exchange-Organization-CalendarBooking-Response = Accept` (and optionally moving the message out
of the Inbox)
([use-rules-to-add-meetings](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/use-rules-to-add-meetings)).
If the work admin will designate the hub's mailbox, the hub gets **real** auto-accept on the work
calendar — a much stronger outcome than tentative, and independent of the Graph kill switches. The
same doc notes a **cancel** caveat scoped to Direct-to-Calendar meetings: *"the canceled meeting
remains in the calendars of attendees until they manually remove it."*

**Update / cancel (general).** For an ordinary tentative item, the Calendar Attendant updates it
when a newer REQUEST for the same event arrives; iTIP's rule is that the highest `SEQUENCE` for a
given `UID` obsoletes lower revisions (see grounding below). Ordinary **iMIP CANCEL** handling on a
user mailbox is not covered by any page fetched here — **(unverified)**; do not borrow the
Direct-to-Calendar cancel behavior for it.

**Do not confuse this with "Events from email."** Outlook's *"Automatically add events from your
email to your calendar"* feature parses booking/flight/reservation confirmations out of message
**bodies** and is controlled per event type
([support.microsoft.com](https://support.microsoft.com/en-us/office/automatically-add-events-from-your-email-to-your-calendar-32e5cf0c-3e65-4870-9ff9-df3683d3fc97)).
It is **not** the iMIP meeting-request path the hub uses and is irrelevant to the hub's REQUEST/CANCEL.

**Tentative may not block time the way accepted does.** A tentative event shows on the owner's
calendar but can present as tentative/free (not busy) in other people's scheduling views, so "the
hub wrote the event" is weaker than an accepted booking if the intent is to reserve the slot.

---

## iTIP / iMIP grounding

A meeting invite over email is **iMIP** ([RFC 6047](https://www.rfc-editor.org/rfc/rfc6047))
carrying an **iTIP** ([RFC 5546](https://www.rfc-editor.org/rfc/rfc5546)) object. `METHOD:REQUEST`
is *"an explicit invitation to one or more Attendees… also used to update or change an existing
event"*: if the `UID` is new to the recipient it creates the event, and if the `UID` already exists
it is *"a rescheduling, an update, or a reconfirmation"* — with the rule that *"the component with
the highest numeric value for the SEQUENCE property obsoletes all other revisions… with lower
values"* ([RFC 5546](https://www.rfc-editor.org/rfc/rfc5546)). `METHOD:CANCEL` *"is used to send a
cancellation notice of an existing event request to the affected Attendees… sent by the Organizer."*
The MIME shape the hub must emit: a body part with *"Content-Type value of 'text/calendar'"* that
*"MUST also include the MIME parameter 'method'"*, e.g.
`Content-Type: text/calendar; method=REQUEST; charset=UTF-8; component=vevent`, and the `method`
parameter must match the `METHOD` inside the object ([RFC 6047](https://www.rfc-editor.org/rfc/rfc6047)).

**Deliverability & auth for a self-sent invite.** The hub sends its **own** message, so ordinary
email authentication applies to the **hub's sending domain**: publish SPF, DKIM-sign, and align
DMARC on that domain, or the invite is liable to spam-filtering or rejection before any calendar
ever processes it — this is a *separate* gate from whether the target auto-processes an invite it
did receive. iMIP's security model reinforces one rule: *"only the 'Organizer' is authorized to
modify or cancel calendar entries she organizes"* and implementations *"SHOULD verify the
authenticity of the creator of an iCalendar object"* ([RFC 6047](https://www.rfc-editor.org/rfc/rfc6047)).
Design consequence: the hub must set `ORGANIZER` to a **hub-controlled, DKIM-aligned address** and
send `From:` that same address. The tempting shortcut — setting `ORGANIZER` to the **user's own**
address so the event looks self-created — is a **DMARC failure and a spoof**, and later
updates/cancels (which must come from the same organizer) would break. The hub owns the event as
organizer; the user is an attendee.

---

## Settled vs still-needs-a-live-check

**Documentation settles:**

- **Proton auto-adds emailed invites by default**, in a **pending** state, no click required; it is
  a **user** toggle (Settings → General → Invitations) that is **on by default**. Requires the
  invite to reach the Proton **Mail** inbox. ([auto-add-invites](https://proton.me/support/auto-add-invites),
  [add-event-from-inbox](https://proton.me/support/add-event-from-inbox))
- **Exchange Online user mailboxes default to `AutomateProcessing=AutoUpdate`**, so a REQUEST
  auto-adds as **tentative** with no click, and the value **can't be changed on a user mailbox**.
  Rooms (AutoAccept) are the exception, not the rule.
  ([Set-CalendarProcessing](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-calendarprocessing?view=exchange-ps))
- A cooperative admin can grant **true auto-accept** to the hub as a designated sender via
  **Direct to Calendar** mail flow rules.
  ([use-rules-to-add-meetings](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/use-rules-to-add-meetings))
- The **wire format and organizer/auth rules** the hub must satisfy (text/calendar; method=REQUEST;
  UID+SEQUENCE; organizer authority). ([RFC 5546](https://www.rfc-editor.org/rfc/rfc5546),
  [RFC 6047](https://www.rfc-editor.org/rfc/rfc6047))

**Still needs a live check (the residual manual spike, now small and specific):**

1. **Does an EXTERNAL organizer's invite auto-add to the work user mailbox?** The doc's
   `ProcessExternalMeetingMessages $false` default may block external senders; its scope for user
   mailboxes is a documentary gap. **The user does not administer the work tenant**
   (`google-microsoft-auth.md`), so `Get-CalendarProcessing` is likely unavailable — resolve it by
   **sending one test REQUEST from the hub's domain to the work address and looking at the
   calendar** (tentative appears = auto-add works; nothing appears = external processing is off and
   the hub needs an admin Direct-to-Calendar rule or another path).
2. **Does Proton auto-add an invite whose ORGANIZER is the hub (a third party)?** Docs impose no
   organizer restriction but never explicitly confirm it. Resolve with **one test invite from the
   hub to the Proton Mail address** — confirm it auto-adds as pending, then send a higher-`SEQUENCE`
   update and a CANCEL to observe recipient-side **update/cancel** behavior (both undocumented).
3. **Hub email deliverability** — confirm the hub domain's SPF/DKIM/DMARC pass to both Proton and
   Exchange so the invite is not filtered before processing. This is independent of auto-add.

Two small, well-defined tests (one invite per target, plus an update+cancel follow-up) collapse the
entire residual. Everything else is settled by the sources above.

---

*Builds on `docs/research/proton-access.md` (Proton has no write API; email-invite is the only
inbound path) and `docs/research/google-microsoft-auth.md` (the work tenant is admin-controlled and
the user is not the admin). Primary sources cited inline. Fastest-rotting facts: the Exchange
external-processing default and Proton's invitation-setting defaults — re-verify before building.*
