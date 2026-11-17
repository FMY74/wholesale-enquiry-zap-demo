# Wholesale enquiry automation (Zapier): design

_15 August 2026. Status: approved before the build. Where the build differed, the README says so._

## Why this exists

1. **Hands-on experience with Zapier.** Build one real automation end to end, then break it on
   purpose, so that questions about how the platform behaves can be answered from experience.
2. **A worked example of a common small-business task.** Example Grocer is a fictional Melbourne
   organic grocer that wants wholesale enquiries recorded, acknowledged and passed to the right
   person without anyone copying them by hand.

Nothing here touches a real business's data or website. All data is dummy data.

## Scope

**In scope:** one Zapier automation, triggered by a Google Form, that validates the enquiry,
records it in a Google Sheet acting as the CRM ledger, replies to the enquirer, and notifies the
person handling wholesale. Plus the deliberate failure testing described below.

**Out of scope, and why:**

| Not doing | Reason |
|---|---|
| GoHighLevel | Its trial bills automatically unless cancelled. Zapier needs no card. |
| Real supplier or customer records | Dummy data only (see below). |

## Success criteria

Each one is checked by doing it, not by reading the configuration:

1. Submitting the form once produces **all three** effects: the enquirer receives an
   auto-reply, the ledger gains one row, and the handler receives a notification.
2. Submitting a second enquiry from the same business does **not** create a second new-enquiry
   row; it is recorded as a repeat and the handler is told so.
3. An enquiry with a malformed email address receives **no** auto-reply and lands in a
   "needs check" area instead.
4. A deliberately broken run (for example, the ledger sheet renamed mid-flight) is observed end
   to end: what Zapier retries, what the Zap History shows, and whether replaying it duplicates
   the ledger row.

Criterion 4 matters most. It answers how monitoring, retries and idempotency differ between code
and a platform like Zapier.

## Flow

```
Google Form: wholesale enquiry
      |  trigger: new form response
      v
[1] Validate the input
      +- malformed email or missing business name
             -> write to the "Needs check" sheet, send nothing, stop
      v
[2] Look for an existing enquiry from this business (search the ledger by email, then by business name)
      +- found -> record as a repeat enquiry, notify the handler as a repeat, stop
      v
[3] Append to the ledger (Google Sheets)
      v
[4] Auto-reply to the enquirer, quoting the enquiry ID
      v
[5] Notify the handler (subject carries the business name and trade type)
```

### Why this order

- **Validation first is fail-closed.** An auto-reply to a forged address sends mail in the
  business's name to someone who did not ask for it.
- **Duplicate check before the reply** stops the same business receiving the same form letter
  repeatedly.
- **Ledger write before the reply** means a failed send loses a notification, not an enquiry.
  The record is the source of truth; the messages are secondary.
- **An enquiry ID on every row** lets a person follow one case later.

### Ledger columns

`Received at` | `Enquiry ID` | `Business name` | `Contact name` | `Email` | `Phone` |
`Trade type` (cafe / grocer / restaurant / other) | `Products of interest` | `Estimated volume` |
`Message` | `Status` (New / Contacted / Sample sent / Trading / Closed) | `Owner` |
`Next action due`

`Status`, `Owner` and `Next action due` are filled in by a person. They exist so the sheet works
as a pipeline rather than an inbox.

## Failure testing (do these on purpose)

1. **Rename the ledger sheet mid-run** -> observe Zapier's automatic retries: how many, how far
   apart, and what the Zap History records.
2. **Replay a run that half-succeeded** -> does the ledger gain a duplicate row? Record the answer;
   this is the idempotency question in concrete form.
3. **Submit a malformed email** -> confirm no auto-reply leaves.
4. **Count task consumption** per enquiry, against the free plan's 100 tasks per month.

## Dummy data

Fictional businesses only: `Test Cafe One`, `Test Wholefoods Two`, `Test Provisions Three`.
No real supplier, staff member or customer appears anywhere. Every notification goes to the
author's own test mailbox (shown in this repository as `example.com` addresses). Nothing is
published.

## Deliverables

- `README.md`: the diagram, the build steps in order, and the measured results of the four
  failure tests
- Screenshots of the Zap editor, the Zap History (including a failure), the ledger, and a
  received auto-reply (kept privately, not in this repository)

**The written record is the deliverable, not the running Zap.** A new Zapier account gets 14 days
of the Professional tier; after that the account falls back to the free plan, where a Zap may
have only one trigger and one action, so a multi-step Zap stops running. Capture everything
before day 15.

## Risks

| Risk | Handling |
|---|---|
| Free plan drops multi-step Zaps on day 15 | Finish and screenshot inside the trial. The README is what survives. |
| Screenshots leak something | Only dummy businesses and the author's own test address appear. Check each image before it is saved. |

## Open questions

None. Decisions taken 15 August 2026: Zapier only (no GoHighLevel), Google Forms as the trigger,
build depth "basic plus the operational practices", and dummy data only.
