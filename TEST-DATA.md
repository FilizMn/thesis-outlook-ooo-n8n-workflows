# Test data
## Proposed test set

Each meeting is designed to trigger a specific expected action. This makes it
possible to check whether the deterministic and the AI-enhanced workflows decide
identically or sensibly.
| #  | Day / Time (CEST) | Title                          | Organizer / Attendee | Attendees | Response requested? | Recurring?   | Whitelisted? | V1 (deterministic, actual) | Acceptable outcome(s) | What it demonstrates                                              |
|----|-------------------|--------------------------------|----------------------|-----------|---------------------|--------------|--------------|----------------------------|-----------------------|------------------------------------------------------------------|
| A1 | Mon 09:00         | Weekly Team Sync               | Organizer            | 8         | –                   | Yes (weekly) | no           | cancel                     | cancel                | Control: clear organizer cancel                                  |
| A2 | Tue 10:00         | Sprint Planning                | Attendee             | 6         | Yes                 | No           | no           | decline                    | decline               | Control: clear invitee decline                                   |
| B1 | Wed 09:30         | Board Meeting                  | Attendee             | 6         | Yes                 | No           | yes          | keep                       | keep                  | Hard whitelist override (invitee), AI never reached              |
| B2 | Fri 09:00         | Client Kickoff (you host)      | Organizer            | 5         | –                   | No           | yes          | keep                       | keep                  | Whitelist override dominates even an organizer-cancel case       |
| C1 | Mon 14:00         | Focus Time (Deep Work)         | Organizer            | 0         | –                   | No           | no           | skipped (no proposal)      | keep                  | Coverage gap: V1 skips personal blockers, AI surfaces them       |
| C2 | Wed 17:00         | Dentist appointment            | Organizer            | 0         | –                   | No           | no           | skipped (no proposal)      | keep                  | Same as C1, second data point                                    |
| D1 | Tue 16:00         | Sync                           | Attendee             | 2         | –                   | No           | no           | delete                     | keep or delete        | Ambiguous case: both outcomes acceptable; tests run-to-run variance |
| D2 | Thu 14:00         | Catch-up                       | Organizer            | 1         | –                   | No           | no           | cancel                     | keep or cancel        | Ambiguous case for intent protection; may flip between runs      |
| E1 | Mon 11:00         | 1:1 with Sarah (Supervisor)    | Organizer            | 1         | –                   | No           | no           | cancel                     | keep                  | Flagship: career-relevant 1:1 should be protected (only V3 can)  |
| E2 | Wed 13:00         | Career Development / Mentoring | Attendee             | 2         | Yes                 | No           | no           | decline                    | keep                  | Relationship value should be preserved                          |
| E3 | Tue 15:00         | Quarterly All-Hands            | Attendee             | 50        | –                   | No           | no           | delete                     | keep                  | Info value should be preserved, not deleted                     |
| E4 | Thu 10:00         | Strategy Workshop (you present)| Organizer            | 12        | –                   | No           | no           | cancel                     | keep                  | Active contribution should be recognized                        |
| E5 | Fri 12:00         | Lunch & Learn                  | Attendee             | 20        | –                   | No           | no           | delete                     | keep or delete        | Borderline intent-confidence; protection may trigger or not      |


**Note on the "Whitelist" category:** In Outlook, create a category named
`Whitelist` and assign it to the relevant meeting. Such meetings are always
protected by both workflows.

**Note on recurring meetings:** Outlook categories (e.g. "Whitelist") apply to the
entire series. Within a period, either all occurrences are protected or none — a
single occurrence can only be protected if it is first broken out of the series
into a standalone event.

> **Note on event sensitivity (C2):** C2 (*Dentist appointment*) is deliberately
created with Outlook sensitivity set to `private`, while all other events use the
default `normal`. This is a fixed part of the test setup, documented here so the
related evaluation finding is reproducible: none of the three workflows read the
`sensitivity` flag, so a private event is classified by exactly the same
structural rules as any other, and in the AI variants its subject line is still
passed to the LLM.

## Fictitious attendee addresses

Fictitious attendee addresses and meeting names for testing the workflows.
Only reserved addresses of the `example.com` domain (RFC 2606) are used, so that
**no real people** are contacted.

```
max.mustermann@example.com, erika.musterfrau@example.com, john.doe@example.com, jane.smith@example.com, hans.mueller@example.com, anna.schmidt@example.com, peter.fischer@example.com, sabine.weber@example.com, michael.meyer@example.com, julia.wagner@example.com, david.brown@example.com, sarah.miller@example.com, james.wilson@example.com, emily.taylor@example.com, robert.thomas@example.com, jessica.white@example.com, daniel.martin@example.com, laura.anderson@example.com, thomas.jackson@example.com, lisa.thompson@example.com, christian.becker@example.com, nicole.hoffmann@example.com, stefan.schaefer@example.com, melanie.koch@example.com, alexander.bauer@example.com, christina.richter@example.com, martin.klein@example.com, nadine.wolf@example.com, andreas.schroeder@example.com, tanja.neumann@example.com, matthias.schwarz@example.com, sandra.braun@example.com, jan.kruger@example.com, vanessa.hofmann@example.com, markus.hartmann@example.com, anita.lange@example.com, juergen.schmitt@example.com, petra.werner@example.com, frank.schmitz@example.com, sonja.krause@example.com, stefanie.meier@example.com, oliver.lehmann@example.com, kerstin.schmid@example.com, torsten.mueller@example.com, daniela.koenig@example.com, rene.germann@example.com, manuela.walter@example.com, patrick.kaiser@example.com, claudia.fuchs@example.com, sebastian.weber@example.com
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
