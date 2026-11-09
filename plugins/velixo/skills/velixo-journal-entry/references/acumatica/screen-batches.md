# Screen batches: command sets for any number of lines

A journal entry is written from **one build** that returns **command sets**: a header, one line, and
a finish. The server repeats the line set once per row, so the same build writes an entry of 2 lines or
2,000. A second build, **R**, releases the entry; it is made only when release was asked for and runs
only after the read-back ties.

| Set | Field in the build | Runs | Does |
|---|---|---|---|
| Header | `commands` + `variables` | once, in the first call | `Clear`, a new GL batch (`<NEW>`), the header fields. Does not save. |
| Line | `lineCommands` + `lineVariables` | once per row in `rows` | `NewRow` and one line's fields. No `Clear`, no `Save`. |
| Finish | `finishCommands` + `finishVariables` | once, in the last call | Put on hold, `Save`, read back the batch number, status and totals. |
| **R** — release | a separate build | once, after the read-back ties | Opens the batch by number, takes it off hold, releases it. |

The builder chooses the screen (Journal Transactions, `GL301000`); never name one in a request. Never
write, edit or reorder a set: pass each one exactly as the build returned it.

## Requests

Pass these to `prepare_screen_commands` as written. They name fields, never values, and say "any
number of lines", which is what makes the builder answer with command sets. Give an `intent` saying
what the user is booking.

**The entry**

> Create a new general ledger journal entry (journal transaction batch) with any number of lines.
> Header: branch {{BranchID}}, ledger {{LedgerID}}, transaction date {{TranDate}}, post period
> {{FinPeriod}}, currency {{CuryID}}, auto reversing {{AutoReverse}}, description {{Description}}.
> Each line: branch {{Branch}}, account {{Account}}, subaccount {{Subaccount}}, project {{Project}},
> project task {{ProjectTask}}, cost code {{CostCode}}, quantity {{Quantity}}, unit of measure {{UOM}},
> reference number {{RefNbr}}, debit {{DebitAmt}}, credit {{CreditAmt}}, transaction description
> {{TranDesc}}, non billable {{NonBillable}}. After the last line, put the batch on hold, save it, and
> read back the batch number, status, currency, debit total, credit total and control total.

Check the answer before using it: `lineCommands` and `finishCommands` are present, the header does not
save, and `lineVariables` names the thirteen line placeholders above. A build without command sets (one
fixed batch) is not usable for this skill: rebuild with the same request once, then stop and report.

**R — release** (build it only when release was asked for)

> Open the existing general ledger journal entry with module GL and batch number {{BatchNbr}}, take it
> off hold, and release it. Then read back the batch number, status, debit total and credit total.

Keep each returned artifact whole, its manifests included. If the builder answers with an `ERROR:`,
relay the reason in plain words and rebuild with the user's answer; never work around it.

## Values

Every placeholder gets a value on every call it appears in. A field the user left to Acumatica goes in
as `null` (the command is dropped and Acumatica defaults it); `""` writes a blank and is used only when
the user asked for the field to be cleared. A missing or unknown key is refused before anything runs.

- **`variables`** (header and finish): `BranchID`, `LedgerID`, `TranDate` (`YYYY-MM-DD`), `FinPeriod`
  (`MMYYYY`, no separator), `CuryID`, `AutoReverse` (`"True"` or `null`), `Description`.
- **`rows`**: one `{ "values": { … } }` per line, in the order of the preview, keyed by the
  `lineVariables` names: `Branch`, `Account`, `Subaccount` (no separators, or `null` for the account's
  default), `Project`, `ProjectTask`, `CostCode`, `Quantity`, `UOM`, `RefNbr` (each `null` when not
  given), `DebitAmt` and `CreditAmt` (plain decimals, `"0"` on the side the line is not on),
  `TranDesc`, `NonBillable` (`"True"` or `null`).
- **R:** `BatchNbr`.

## Running the entry

All of this happens after the user's yes to the preview. The preview is asked **after** the build (and
R, when release was asked for), so the confirmation sits beside what will run.

**Up to 50 lines: one call.** `commands`, `lineCommands`, `finishCommands`, `variables` and all the
`rows`. The whole entry is one batch with one `Save`, so it is written completely or not at all.

**More than 50 lines: groups of up to 50, in consecutive calls.** The server takes at most 50 rows per
call; 50 journal lines took about 37 seconds on gab-dev.

1. First call: `commands`, `lineCommands`, `variables`, rows 1–50, and `leaveOpen: true`. The result
   carries a `continuation`.
2. Each middle call: that `continuation`, `lineCommands`, the next 50 rows, `leaveOpen: true`, and no
   `commands`. Each result carries the next `continuation`; always pass the latest one.
3. Last call: the latest `continuation`, `lineCommands` with the remaining rows, `finishCommands` and
   `variables`. This call saves the entry.

Between those calls, make **no other screen call** on the connection (a read through `execute_query`
is fine), and send the next call **within 2 minutes**: the screen session closes after 2 minutes idle.
Nothing is saved until the last call, so an entry interrupted part way is not in Acumatica at all.

After the write:

1. Take the batch number and status from the finish set's read-back (`feedback.savedKey` holds the
   number Acumatica assigned) and relay every `feedback.issues` entry, warnings included.
2. **Read back** with J5 and J6 and tie out (`references/acumatica/validation.md`, "After the write").
3. **R**, only when release was asked for and the read-back tied. Read `processStatus`: `completed`
   with status Posted or Released is done; `failed` is not released, whatever the rest says — relay
   `processStatus.detail` as it is (a closed period, an inter-branch account missing, a posting error);
   `stillRunning` or `probeFailed` means re-check with J5 and say what it shows. "Pending Approval"
   means the batch went to an approver: report it, never approve it. A refused release leaves the batch
   on hold and complete; say so.

## What the live run showed (gab-dev, 2026-10-17)

| Test | Calls | Result |
|---|---|---|
| 2 lines, on hold | 1 | GL002519, On Hold, 10.00 / 10.00; J6 tied line by line |
| 60 lines, on hold | 2: rows 1–50 with `leaveOpen`, rows 51–60 with the `continuation` and the finish set | GL002526, On Hold, 30.00 / 30.00, lines 1–60 in order; J6 tied |
| Account 99999 on line 2 | 1 | Refused at line 2 ("The record has been deleted."), batch still `<NEW>`; J5 showed nothing saved |
| 2 lines, then R | 2 | GL002539 saved On Hold, J6 tied, then R: `processStatus` `completed`, status **Posted** |

Fields left `null` resolved to: project `X` ("Non-Project Code."), cost code `00000`, quantity `0`, non
billable and auto reversing `false`. Report them as such in the defaults list. On this tenant release
posts straight away (status Posted, not Released).

## Errors

| What comes back | What it means | What to do |
|---|---|---|
| A refusal naming a row ("line 7") | that row's values do not fit the line set (a missing key, a `<` or `>` in a value) | Nothing ran in that call. Fix the row from the preview; ask the user if the value is theirs to give. |
| `SCREEN_VARIABLE_MISSING` / `_UNKNOWN` | a placeholder without a value, or a key the set does not use | Nothing ran. Fix the map from the manifests and run again. |
| `SCREEN_SESSION_LOST` | more than 2 minutes passed between calls, or the session was reset | Nothing of this call ran, and the unsaved entry is gone: nothing was saved. Say so, and offer to write the whole entry again from the first call (the same confirmation covers it, since nothing was written). |
| `SCREEN_CONTINUATION_MISMATCH` | another screen call ran on the session between two calls | Same as above: nothing saved; start again from the first call, with no other screen call in between. |
| A field error in `feedback.issues` (account, subaccount, project, task, cost code, period) | Acumatica refused a value; the issue names the row | Relay it with the line number, in plain words: for an account code that does not exist Acumatica says "The record has been deleted." Correct and ask for a new confirmation. |
| A failed call whose error says the batch "partially ran" and that what it wrote "stands" | the server's generic wording for any failure part way through a batch | Read the feedback: a `BatchModule.BatchNbr` of `<NEW>` means the `Save` never ran and nothing was saved. Confirm with J5 (no batch with this description), then say so. Never re-run the call without that check. |
| An HTTP 408, 504 or 524, or any timeout on the last call | the request may have run and saved anyway | **Never retry before checking J5** for a batch with this description and total. Found: report it and tie it out. Not found: say so and ask before writing again. |
| "The period is closed" or similar | the period closed between the check and the write | Relay it; a closed period is the user's decision, never reopen it. |
