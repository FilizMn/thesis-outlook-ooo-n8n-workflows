# Test data

Fictitious attendee addresses and meeting names for testing the workflows.
Only reserved addresses of the `example.com` domain (RFC 2606) are used, so that
**no real people** are contacted.

## Proposed test set

Each meeting is designed to trigger a specific expected action. This makes it
possible to check whether the deterministic and the AI-enhanced workflows decide
identically or sensibly.

| # | Day / Time (CEST) | Title | **Organizer / Attendee** | Attendees | Response requested? | Recurring? | Whitelisted? | V1 Deterministic | V2 AI-Enhanced | V3 AI Intent-Protected | What it demonstrates |
|---|-------------------|-------|--------------------------|-----------|---------------------|------------|--------------|------------------|----------------|------------------------|----------------------|
| A1 | Mon 09:00 | Weekly Team Sync | **Organizer** | 8 | – | Yes (weekly) | not whitelisted | cancel | cancel | cancel | Control: all three agree (clear organizer cancel) |
| A2 | Tue 10:00 | Sprint Planning | **Attendee** | 6 | Yes | No | not whitelisted | decline | decline | decline | Control: clear invitee decline |
| B1 | Wed 09:30 | Board Meeting | **Attendee** | 6 | Yes | No | **whitelisted** | keep | keep | keep | Control: hard whitelist override on an invitee event, identical in all three (AI never reached) |
| B2 | Fri 09:00 | Client Kickoff (you host) | **Organizer** | 5 | – | No | **whitelisted** | keep | keep | keep | Second whitelist case: hard override wins even on an organizer event that would otherwise be cancelled — shows the override dominates a different role/action |
| C1 | Mon 14:00 | Focus Time (Deep Work) | **Organizer** | 0 | – | No | not whitelisted | skipped (no proposal) | surfaced → keep | surfaced → keep | Coverage gap: V1 silently ignores personal blockers, AI surfaces them |
| C2 | Wed 17:00 | Dentist appointment | **Organizer** | 0 | – | No | not whitelisted | skipped (no proposal) | surfaced → keep | surfaced → keep | Same as C1, second data point |
| E1 | Mon 11:00 | 1:1 with Sarah (Supervisor) | **Organizer** | 1 (Supervisor) | – | No | not whitelisted | cancel | cancel | **keep** (relationship_career / active_contribution) | Flagship discriminator: only V3 protects the career-relevant 1:1 |
| E2 | Wed 13:00 | Career Development / Mentoring | **Attendee** | 2 | Yes | No | not whitelisted | decline | decline | **keep** (relationship_career) | V3 preserves relationship value that V1/V2 decline |
| E3 | Tue 15:00 | Quarterly All-Hands | **Attendee** | 50 | – | No | not whitelisted | delete (from own calendar!) | keep / delete | **keep** (stay_informed) | V1 destroys info value (delete); V3 preserves it |
| E4 | Thu 10:00 | Strategy Workshop (you present) | **Organizer** | 12 | – | No | not whitelisted | cancel | cancel | **keep** (active_contribution) | V3 recognizes active contribution; V1/V2 cancel |
| D1 | Tue 16:00 | Sync | **Attendee** | 2 | – | No | not whitelisted | delete | keep *or* delete | keep *or* delete | Threshold case: confidence near 0.5 → <0.5→keep fallback; non-determinism across repeated runs |
| D2 | Thu 14:00 | Catch-up | **Organizer** | 1 | – | No | not whitelisted | cancel | cancel | keep *or* cancel | Threshold case for intent protection in V3 (may flip between runs) |
| E5 | Fri 12:00 | Lunch & Learn | **Attendee** | 20 | – | No | not whitelisted | delete | keep / delete | keep *or* delete (stay_informed borderline) | Intent-confidence threshold: just above/below 0.5 → protection triggers or not |

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

For meetings with many attendees, use as many addresses as needed from this list
(`test1@example.com` through `testn@example.com`):

```
test1@example.com, test2@example.com, test3@example.com, test4@example.com, test5@example.com, test6@example.com, test7@example.com, test8@example.com, test9@example.com, test10@example.com, test11@example.com, test12@example.com, test13@example.com, test14@example.com, test15@example.com, test16@example.com, test17@example.com, test18@example.com, test19@example.com, test20@example.com, test21@example.com, test22@example.com, test23@example.com, test24@example.com, test25@example.com, test26@example.com, test27@example.com, test28@example.com, test29@example.com, test30@example.com ... testn@example.com
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
