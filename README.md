# Wholesale enquiry Zap: built with Zapier Copilot, then verified branch by branch

A demonstration Zap that routes wholesale enquiries from a Google Form into a ledger sheet, a
quarantine sheet and two emails. Built on 16 and 17 August 2026 on a personal Google account and a
Zapier trial account. The business, Example Grocer, is a fictional Melbourne organic grocer. Every
business name and address in this repository is a dummy.

I designed the flow and the acceptance tests, had Zapier's AI Copilot build the Zap, then opened
every step, ran every branch and fixed what it got wrong. This README records what that found.

## Status

Demo only. Nothing is live.

- The Zap was switched off when the Zapier trial ended on 30 August 2026.
- On 3 November 2026 the demo form and ledger sheet were deleted. The Zap stays switched off.
- No real company system, customer or supplier was connected at any point.

## The problem

A wholesale buyer (a cafe, grocer or restaurant) needs a way to send an enquiry, get an
acknowledgement and reach the person who handles wholesale. Each enquiry should be recorded once,
a repeat enquiry from the same business should not get the same form letter again, and a malformed
email address should never receive mail sent in the business's name.

## The design

The full design is in [docs/design.md](docs/design.md). The rules that shaped the order of steps:

- Validate first. An enquiry with a malformed email goes to a `Needs check` sheet and no email is
  sent to anyone.
- Check for duplicates before replying. A business already in the ledger gets no second auto-reply;
  the owner is told it is a repeat.
- Write the ledger row before sending email. If sending fails, the enquiry is still recorded.
- Give every ledger row an enquiry ID (`WS-` plus the Formatter step's output) so one case can be
  followed later.

## The Zap flow

12 steps:

```
1  Google Forms     New Form Response
2  Formatter        Date/Time, Melbourne time (used for the enquiry ID)
3  Google Sheets    Lookup Spreadsheet Row in Ledger, by Email
4  Paths
   Path A  Invalid   condition (5): Email does not contain "@"
                     6   Sheets: create row in Needs check
   Path B  Repeat    condition (7): Email contains "@" AND lookup Business name exists
                     8   Gmail: notify owner "Repeat enquiry"
   Path C  New       condition (9): Email contains "@" AND lookup Business name does not exist
                     10  Sheets: create row in Ledger (Status = New)
                     11  Gmail: auto-reply to the enquirer, quoting the enquiry ID
                     12  Gmail: notify owner "New enquiry"
```

| Case | Result |
|---|---|
| Email has no `@` | Row in `Needs check`. No email sent. |
| Business already in the ledger | Owner notified as a repeat. No auto-reply. |
| New business | Ledger row, then auto-reply, then owner notification. |

The ledger has three columns a person fills in (`Status`, `Owner`, `Next action due`) so it works
as a pipeline rather than an inbox.

## How it was built

1. A Google Apps Script created the demo form and the ledger sheet in one run.
2. I set up the trigger and the Formatter step by hand. The first trigger, Google Sheets
   "New Spreadsheet Row", kept returning "No spreadsheet row found" even though the form's rows
   were in the sheet. Switching to Google Forms "New Form Response" picked up the response straight
   away.
3. I gave Zapier Copilot a written specification of the remaining steps and let it build.

Copilot added roughly one step per instruction, confirmed over three rounds:

| Round | Asked for | Added |
|---|---|---|
| 1 | Lookup, three paths and five actions | The skeleton and Path A only |
| 2 | Build Path B and Path C | One Gmail action in Path B |
| 3 | Three actions in Path C | The ledger write only |

I stopped prompting after round 3 and added the remaining actions by hand. Copilot chose the right
apps and the right branch structure. The defects were inside the steps.

## How it was verified

I opened all 12 steps and compared each one with the design, using a checklist (trigger target,
path contents, step order, write targets, lookup settings, path conditions, field mapping and
recipients). Then I ran each branch with test enquiries and checked the sheets, the inbox and the
run history.

## The nine defects

1. Path B and Path C were left with empty action slots, so both branches would do nothing.
2. Path C's second condition was `Is in` with an empty value, so Path C would never run.
3. Path A's Sheets action had no spreadsheet or worksheet set, so invalid enquiries were recorded nowhere.
4. Asked to use `Does not exist`, Copilot instead built `ireversecontains on "."` to detect absence; I replaced it with the built-in operator.
5. The lookup searched the `Contact name` column with an email address, so it never matched, duplicate detection was dead and no error was raised.
6. An inline formula for the enquiry ID had a broken internal reference and failed only when run (`Expected 2 arguments, but found 1`).
7. Path B's Gmail action had every field blank, including To, Subject and Body, so the owner would never be notified.
8. Path A's condition was `Email (Text) Is in @`, which is reversed, so an invalid enquiry matched no branch, was recorded nowhere, and the run still reported `Successful`.
9. A saved draft had lost all three path branches; publishing it would have removed the branching from the live Zap, so I discarded it and made a new draft from the published version.

What these have in common:

- Defects 2, 3, 5 and 8 fail silently. The run history shows `Successful` and nothing lands
  anywhere. A single happy-path test would have passed.
- Quality varied between neighbouring steps. Path B's condition was correct while Path C's was
  broken; steps 11 and 12 were fully configured while step 8 was blank. Every step had to be opened.
- Defect 6 was the only one invisible in the settings. It appeared only when the step ran.
  Configuration review and test runs caught different defects.
- Defect 8 was found by running the invalid case. I had checked the Path B and Path C conditions
  but not Path A's.

## What was measured

Each case was submitted through the form to the published Zap and checked in the sheets, the inbox
and the run history.

| Case | Result | Tasks |
|---|---|---|
| New enquiry | Ledger row, auto-reply and owner notification all arrived | 4 |
| Same email again | No new ledger row, repeat notification sent, no auto-reply | 2 |
| Email without `@` | Row in `Needs check`, ledger unchanged, all three Gmail steps `Did not attempt` | 2 |
| Ledger tab renamed | Run `Errored` at the lookup, nothing written, nothing sent | 0 |

Before defect 8 was fixed, the same invalid enquiry was recorded nowhere and the run history showed
`Successful` (1 task). After the fix it went to `Needs check` as designed.

At 4 tasks per new enquiry, the free plan's 100 tasks a month covers about 25 new enquiries.

### Breaking it on purpose

I renamed the `Ledger` tab to `Ledger_OLD` and submitted a new enquiry.

- The run failed at step 3 (the lookup), not at step 10 (the write), because both point at the same
  sheet. Error: `Unable to parse range: Ledger!$E:$E`.
- The run was reported as `Errored`. Nothing after the lookup ran: no rows, no emails.
- The enquiry itself was still in the form's own response sheet.
- The failed run used 0 tasks.
- There was no automatic retry. Autoreplay was off, so the run stayed `Errored` until I pressed
  Replay. Nothing sent a notification; the only way to find it was to look at the run history.

### Replay and duplicates

Replay does not resume from the failed step. The dialog says: "Replaying this Zap will create a new
run on the latest published version." Every step runs again, including the writes.

- With the tab name restored, the first replay succeeded (4 tasks) and the ledger row appeared.
- A second replay of the same run succeeded with 2 tasks and added no ledger row.

The second replay did not duplicate the row because the lookup found the business and sent the run
down Path B. Zapier itself did not prevent the duplicate. Without the lookup step, each replay would
have added another ledger row and sent the enquirer another auto-reply.

## What I would change before running this for real

- Alert on failure. An `Errored` run is visible only to someone who checks the run history.
- Recovery is manual. Without Autoreplay, someone has to notice and press Replay.
- The lookup is the only duplicate guard and matches on email, so the same business writing from a
  second address is recorded twice.
- Paths and multi-step Zaps need a paid plan.

## Files

| File | Contents |
|---|---|
| `docs/design.md` | The design as approved before the build |

The Apps Script and the Copilot specification are kept private and are available to employers on request.

## Licence

Copyright (c) 2026 FMY74. All rights reserved. This repository is published for viewing and
evaluation only. No permission is granted to copy, modify, distribute, sublicense or use this code
or its documentation, in whole or in part, without prior written permission. See [LICENSE](LICENSE).
