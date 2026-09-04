# thesis-outlook-ooo-n8n-workflows — Automated Meeting Cancellation

Three n8n workflows that, for a defined out-of-office period, automatically detect
Outlook meetings and — after human approval — cancel, decline or remove them from
the calendar. The variants serve to compare a **purely rule-based (deterministic)**
approach with two increasingly **AI-assisted** approaches, in order to study *when
and how AI augmentation improves rule-based workflow automation*.

## The three workflows

### 1. `Outlook OOO Auto Cancel & Decline | Deterministic`
Purely rule-based, no AI. What happens to each meeting is decided by fixed rules:

| Role in meeting | Condition | Action |
|-----------------|-----------|--------|
| Organizer | meeting has attendees | **cancel** (notifies everyone) |
| Invited | response requested | **decline** (declines with a note) |
| Invited | no response needed, has attendees | **delete** (removes from own calendar only) |
| * | category "Whitelist" | **protected** (never changed) |

Every run is logged to a data table (`OOO Deterministic Decision Log`): for each
determined meeting it records the rule-based action, the human's overall decision
(Accept / Decline) and the resulting final action. This mirrors the logging of the
AI workflows so the datasets can be compared directly.

### 2. `Outlook OOO Auto Cancel & Decline | AI-Enhanced`
Meeting retrieval remains deterministic, and the whitelist remains a hard override 
that the AI cannot bypass. Same overall structure, but the **action per meeting 
is proposed by an AI model** (OpenAI, `gpt-5-mini`) instead of by fixed rules.
A confidence fallback keeps the workflow safe: any proposal with confidence < 0.5
(or an invalid action) is forced to **keep**. Every AI decision together with the
final human decision is logged to a data table (`OOO AI Decision Log`).

### 3. `Outlook OOO Auto Cancel & Decline | AI Intent-Protected`
Extends the AI-Enhanced approach with an **additional AI pre-stage** that decides,
before any action is proposed, whether a meeting should be *protected* because of
its **intent / importance** — even when the rules or the action model would
otherwise cancel or decline it. This models the case where the hard category whitelist 
is not flexible enough, or where maintaining it manually would be too much effort.

Two AI calls run per meeting:

1. **Intent protection** — classifies the meeting into one of
   `active_contribution` / `relationship_career` / `stay_informed`, or `none`.
   Protection is a **soft override**: it only holds when the model is confident
   (≥ 0.5); a weak signal is dropped and the event falls through to the normal
   action logic. The hard category whitelist still sits *above* this stage and
   cannot be overruled by any AI call.
2. **Action classification** — the same per-meeting action model as the
   AI-Enhanced workflow (cancel / decline / delete / keep, with the < 0.5 → keep
   fallback). If the meeting was intent-protected, the proposed action is forced
   to **keep**, but the raw action decision is still recorded so the effect of the
   intent layer stays auditable.

Every run is logged to its own richer data table (`OOO AI Intent Decision Log`),
which records both the intent decision and the action decision per meeting.

> **Research note (non-determinism):** because the AI variants rely on a language
> model, the same ambiguous meeting may occasionally be classified differently
> across runs. This is a deliberate finding of the thesis (flexibility bought at
> the cost of reproducibility/auditability), and is measured by running the same
> test dataset multiple times and logging the variance.

### Shared flow of all workflows
1. Open the form → enter the out-of-office period (first/last day) and a reason.
2. The workflow reads all Outlook meetings in the period (across all calendars).
3. Already-cancelled meetings and hard-whitelisted meetings are filtered out of the candidate list (they are never proposed for any action).
4. The remaining meetings are classified (deterministically, by AI, or by the two-stage AI) into a proposed action.
5. **Review page** (this is where the variants differ):
   - **Deterministic:** shows a list of all proposed meetings and offers a single
     overall decision — **Accept** (carry out all listed actions) or **Decline**
     (leave everything unchanged). Individual meetings **cannot** be changed here.
   - **AI-Enhanced:** shows **one dropdown per meeting**, pre-selected with the AI
     recommendation, which can be individually changed (cancel / decline / delete /
     keep) before confirming.
   - **AI Intent-Protected:** shows **one card per meeting** with two columns —
     *Current state* (role, attendee count, recurring, response requested, current
     status) on the left and *AI proposal* (proposed action, intent-protection
     label + confidence, or action confidence, plus the model's reason) on the
     right — followed by a per-meeting dropdown pre-selected with the proposal.
     Intent-protected meetings are pre-set to **keep**.
6. After confirmation, the actions are executed via the Microsoft Graph API.

## The action mapping (Microsoft Graph)

| Action | Graph call | Who |
|--------|-----------|-----|
| cancel | `POST /me/events/{id}/cancel` (with comment) | organizer |
| decline | `POST /me/events/{id}/decline` (with comment, `SendResponse: true`) | invitee |
| delete | `DELETE /me/events/{id}` | invitee, own calendar only |
| keep | no call | — |

## Requirements

- **An n8n account** is required (n8n Cloud or self-hosted). Without an n8n
  instance the workflows cannot be run. A free n8n Cloud trial is sufficient to
  try them out.
- **A Microsoft/Outlook account** with a calendar.
- For the AI workflows: access to OpenAI. During development and testing, the model
  **`gpt-5-mini`** was used (via n8n's free OpenAI credits). These free credits are
  limited but were sufficient for development and testing. For more extensive
  testing or productive use, a **dedicated OpenAI API key** would be required.

## Setup (step by step)

1. **Import the workflows**: In n8n, go to *Workflows → Import from File* and
   import the three `.json` files from this repository.
2. **Connect the Outlook credential (OAuth)**:
   - When you open a Microsoft Outlook node, you are prompted to connect a
     credential of type *Microsoft Outlook OAuth2 API*.
   - Click **Connect / Sign in with Microsoft** and complete the OAuth login with
     your Outlook account. n8n Cloud manages the OAuth app itself — there is **no**
     need to enter a Client ID / Client Secret manually.
   - Important: all Outlook/Graph nodes in a workflow must point to the same
     connected credential.
   - Note: your browser's popup blocker may prevent the login window — if no window
     appears, allow popups for n8n.
3. **OpenAI credential (AI workflows only)**: In the *OpenAI Chat Model* node(s),
   select an OpenAI credential or enter your own API key. The AI Intent-Protected
   workflow uses **two** model nodes (intent + action) — both point to the same
   OpenAI credential.
4. **Data tables (all workflows)**: Each workflow writes its decisions to its own
   data table. When imported on a new instance these tables do not exist and must be
   recreated and re-selected in the respective *Log to Table* node.
   - **Deterministic** → table `OOO Deterministic Decision Log` (7 columns):
     `run_id`, `logged_at`, `event_subject`, `event_day`, `rule_action`,
     `human_decision`, `final_action`.
   - **AI-Enhanced** → table `OOO AI Decision Log` (9 columns):
     `run_id`, `logged_at`, `event_subject`, `event_day`, `ai_action`,
     `ai_confidence`, `ai_reason`, `human_final_action`, `was_overridden`.
   - **AI Intent-Protected** → table `OOO AI Intent Decision Log` (14 columns):
     `run_id`, `logged_at`, `event_subject`, `event_day`, `intent_protected`,
     `intent_label`, `intent_confidence`, `intent_reason`, `ai_action`,
     `ai_confidence`, `ai_reason`, `proposed_action`, `human_final_action`,
     `was_overridden`.

   The different column sets are intentional and reflect the difference between the
   approaches: the deterministic workflow captures one overall human decision, the
   AI-Enhanced workflow captures a per-meeting AI proposal with confidence and
   whether the human overrode it, and the Intent-Protected workflow additionally
   captures the separate intent decision (protected flag, label, confidence, reason)
   alongside the raw action proposal (`ai_action`) and the final `proposed_action`.
   A crosswalk mapping comparable fields across the three tables is provided in the
   thesis appendix.
5. **Testing**: In the editor, click *Execute workflow* and fill in the opened
   test form URL in your browser (keep the tab open). For continuous use / a fixed
   URL, **publish (activate)** the workflow; the production form URL then shows no
   "This is a test version" banner.

## Human-in-the-loop

None of the workflows act autonomously. Nothing is cancelled, declined or deleted
until the user submits the review form. The workflows differ only in *how much
control the review gives*: a single global Accept/Decline (deterministic) vs. a
per-meeting override (both AI variants).

## Important note on OAuth login with university accounts

The OAuth login **could not be completed with my university email address**, because
I lack the **administrator rights in the university's Microsoft 365 tenant**. Many
university tenants require administrator approval (admin consent) before a
third-party app such as n8n is allowed to access the calendar.

**Recommended for testing:** use a **private Microsoft/Outlook account**
(e.g. `@hotmail.de`, `@outlook.com`). This allows the OAuth login to work without
administrator rights. Whether it works with a university account depends solely on
the policies of the respective tenant and can only be determined by attempting the
login (including any required admin approval).

## Test data

Since no real people should be invited for testing, fictitious attendee addresses
(the reserved `@example.com` domain) and realistic meeting names are used. These
are provided in [`TEST-DATA.md`](./TEST-DATA.md), which includes the full list of
test addresses (freely extendable), the expected action for each test meeting, and
a note on why an automated calendar-seeding approach was abandoned in favour of
creating the test appointments manually.

To keep the three workflows comparable, all variants are run against the **same
calendar state**. Because every meeting is left on **keep** at the review step, no
meeting is actually cancelled, declined or deleted during the evaluation runs, so
the same test events can be reused across all variants and all repeated runs
without re-creating them.

## Evaluation results (decision logs)

The recorded decision logs from the evaluation runs are provided as CSV files in
this repository:

- [`v1_deterministic_log.csv`](./v1_deterministic_log.csv) — 1 run, 10 rows
- [`v2_ai_enhanced_log.csv`](./v2_ai_enhanced_log.csv) — 5 runs, 55 rows
- [`v3_intent_protected_log.csv`](./v3_intent_protected_log.csv) — 10 runs, 110 rows

Each row corresponds to one candidate meeting in one run; runs are distinguished
by `run_id`. The deterministic variant is run once because its output is
reproducible, whereas the AI variants are run multiple times so that the
run-to-run variation of the model decisions can be observed. The AI
Intent-Protected variant is run more often than the AI-Enhanced variant because
its two AI stages introduce an additional source of variation.

Two points are important when reading the logs:

- **Excluded meetings do not appear.** Hard-whitelisted meetings and
  already-cancelled meetings are filtered out before classification, so each run
  logs the remaining candidate meetings rather than the full test set.
- **The final action is constant.** During the evaluation, every meeting was left
  on **keep** at the review step so that no calendar changes were executed and the
  same calendar state could be reused across all runs. The evaluation therefore
  compares the system's *proposed* action (`rule_action` / `ai_action` /
  `proposed_action`, together with the intent fields in the Intent-Protected
  variant) against the acceptable outcomes defined in
  [`TEST-DATA.md`](./TEST-DATA.md), not the logged `human_final_action`.

## Security note

All workflows start with a **form trigger without login**. Anyone who knows the
form URL triggers the workflow on the connected Outlook account. Therefore, **do
not share the URL publicly**. If needed, the form trigger can additionally be
protected with Basic Auth.
