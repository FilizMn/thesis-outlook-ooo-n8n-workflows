# Test data
## Proposed test set

Each meeting is designed to trigger a specific expected action. This makes it
possible to check whether the deterministic and the AI-enhanced workflows decide
identically or sensibly.

| ID | Day/Time | Title | Role | Attendees | Resp. req. | Recurring | Whitelisted | V1 (deterministic) | Acceptable outcome(s) |
|----|----------|-------|------|-----------|------------|-----------|-------------|--------------------|-----------------------|
| A1 | Mon 09:00 | Team Sync | Organizer | 3 | yes | no | no | cancel | cancel |
| A2 | Mon 14:00 | Project Kickoff | Organizer | 5 | yes | no | no | cancel | cancel |
| B1 | Tue 10:00 | Budget Review | Invitee | 5 | yes | no | no | decline | decline |
| B2 | Tue 15:00 | Cross-Team Planning | Invitee | 13 | yes | no | no | decline | decline |
| C1 | Wed 09:15 | Daily Standup | Invitee | 6 | yes | yes | yes | keep | keep |
| C2 | Wed 11:00 | Weekly All-Hands | Invitee | 40 | yes | yes | yes | keep | keep |
| D1 | Wed 16:00 | Weekly (tbd) | Invitee | 4 | yes | no | no | decline | decline / keep |
| D2 | Thu 09:30 | Quick Catch-up | Invitee | 3 | no | no | no | delete | delete / keep |
| E1 | Thu 11:00 | Review with Prof. (Supervisor) | Organizer | 2 | yes | no | no | cancel | keep |
| E2 | Thu 14:00 | Focus Time (self) | Organizer | 0 | no | no | no | keep | keep |
| E3 | Fri 09:00 | Company Town Hall | Invitee | 51 | no | no | no | delete | keep |
| E4 | Fri 11:00 | Client Demo (you present) | Invitee | 5 | yes | no | no | decline | keep |
| E5 | Fri 15:00 | Cancelled Workshop | Invitee | 3 | yes | no | no | keep (excluded) | keep (excluded) |
| E6 | Fri 16:00 | Board Preparation Meeting | Organizer | 6 | yes | no | no | cancel | cancel |

**Note on "Attendees":** Attendees is the value of `attendees.length` from Microsoft Graph. The organiser is never part of this array, so the organiser is not counted. For your own organised meetings the number therefore excludes you; for meetings you were invited to it includes you but not the organiser.

**Note on "Resp. req." for D2 and E3:** Response is deliberately not requested for these two invitee events. This is required to exercise the deterministic deletion path (invitee, no response requested, event has attendees); with a response requested, the deterministic variant would classify them as `decline` instead of `delete`.

**Note on the "Whitelist" category:** In Outlook, create a category named `Whitelist` and assign it to the relevant meeting. The category match is case-insensitive and ignores surrounding whitespace, but only the exact word `whitelist` is recognised (variants such as `whitelisted` do not match). Meetings carrying this category are excluded before classification and are therefore always left unchanged.

**Note on recurring meetings:** Outlook categories (e.g. "Whitelist") apply to the entire series. Within a period, either all occurrences are protected or none — a single occurrence can only be protected if it is first broken out of the series into a standalone event.

## Fictitious attendee addresses

Fictitious attendee addresses and meeting names for testing the workflows.
Only reserved addresses of the `example.com` domain (RFC 2606) are used, so that
**no real people** are contacted.

```
max.mustermann@example.com; erika.musterfrau@example.com; john.doe@example.com; jane.smith@example.com; hans.mueller@example.com; anna.schmidt@example.com; peter.fischer@example.com; sabine.weber@example.com; michael.meyer@example.com; julia.wagner@example.com; david.brown@example.com; sarah.miller@example.com; james.wilson@example.com; emily.taylor@example.com; robert.thomas@example.com; jessica.white@example.com; daniel.martin@example.com; laura.anderson@example.com; thomas.jackson@example.com; lisa.thompson@example.com; christian.becker@example.com; nicole.hoffmann@example.com; stefan.schaefer@example.com; melanie.koch@example.com; alexander.bauer@example.com; christina.richter@example.com; martin.klein@example.com; nadine.wolf@example.com; andreas.schroeder@example.com; tanja.neumann@example.com; matthias.schwarz@example.com; sandra.braun@example.com; jan.kruger@example.com; vanessa.hofmann@example.com; markus.hartmann@example.com; anita.lange@example.com; juergen.schmitt@example.com; petra.werner@example.com; frank.schmitz@example.com; sonja.krause@example.com; stefanie.meier@example.com; oliver.lehmann@example.com; kerstin.schmid@example.com; torsten.mueller@example.com; daniela.koenig@example.com; rene.germann@example.com; manuela.walter@example.com; patrick.kaiser@example.com; claudia.fuchs@example.com; sebastian.weber@example.com
```

# Automated seeding approach failed

The first idea was to create the test appointments automatically instead of
entering them by hand. Two "seeder" workflows were built for this:

- **Seed Test Calendar | Own Account (Organizer)** — creates events where you
  are the organizer (to test the cancel / keep / whitelist / series cases).
- **Seed Test Calendar | Test Account (Invites You)** — creates events from a
  second account that invites you (to test the decline / delete cases).

Both used the Microsoft Graph API (`POST /me/events`) via an HTTP Request node
and filled the events with the fictional test1@example.com … test80@example.com
attendee addresses.

**What went wrong:**

1. **Rate limiting (HTTP 429 `ApplicationThrottled`).** Creating many events in
   quick succession hit Microsoft's per-mailbox concurrency limit. Adding
   throttling (one event at a time, ~1.5 s pause between requests) fixed the 429.
2. **Account suspension (HTTP 403 `ErrorAccountSuspend`).** After the throttling
   fix, Microsoft suspended the mailboxes anyway. Rapidly creating a large
   number of events, each inviting many non-deliverable @example.com addresses,
   was treated as a spam signal. This affected both the main account and the
   second test account.

**Conclusion:** automated calendar seeding with many fake external invitees is
not viable against personal Microsoft accounts — it looks like spam and gets the
mailbox blocked. The test appointments must be created manually in Outlook
instead.
