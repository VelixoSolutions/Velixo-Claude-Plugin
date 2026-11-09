# Release or reverse an existing batch by its number

Two operations on a GL batch that already exists, named by its number alone ("release GL002519",
"reverse GL002539"). Both start with the same lookup and end with the same tie-out. The rest of the
skill (inputs, validation, preview, confirmation) applies as written; this file says what differs.

## Look the batch up first

Run **J7** (`gl-batch-by-number`) with `Batch Number` exactly as the user typed it (`GL002519`).
It returns the batch's branch, ledger, period, date, status, flags, description, currency, totals and
every line, whatever the status. Trim the padded codes (branch, project). No rows: no GL batch has
that number; say so and ask. Never guess a neighbouring number.

| `Batch Status` | Release | Reverse |
|---|---|---|
| `H` On Hold | yes — it is taken off hold first | no: nothing has posted; offer to delete it in Acumatica instead |
| `B` Balanced | yes, with RB (a reclassification batch is always Balanced) | no, as above |
| `U` Unposted | already released; offer to post it in Acumatica | no, as above |
| `P` Posted / `C` Completed | no: already released | **yes** |
| `D` Pending Approval / `J` Rejected | no: it waits on an approver; never approve it | no |
| `V` Voided, `S` Scheduled | no; say why | no |

Header totals repeat on every row: take them from the first row, and check that the line amounts sum
to them and that debits equal credits. A batch that does not balance cannot be released; stop and
show the difference.

## Release

1. **Checks.** The status allows it (table above). The period is still **Open** in J2 for the batch's
   company (J1 gives the company from the branch). The batch balances.
2. **Build the release batch for the batch's status.** Both take only `BatchNbr`:
   - **On hold** (`H`): **R** from `references/acumatica/screen-batches.md` — Remove Hold, Save, Release.
   - **Balanced** (`B`), which includes every reclassification batch: **RB**, built from
     > Open the existing general ledger journal entry with module GL and batch number {{BatchNbr}},
     > which is already balanced and not on hold, and release it. Then read back the batch number,
     > status, debit total and credit total.

     It is Release alone. R fails on a balanced batch ("The Remove Hold button is disabled.") without
     writing anything; RB on a held batch would be refused because Release is disabled on hold.
3. **Preview and confirm.** Headline: "**Release batch GL002519** (branch PRODWHOLE, period 06-2025,
   10.00), which posts it to the ledger." The lines as a table from J7, with names. Then this sentence,
   word for word, just above the question: "**Releasing cannot be undone: once posted, the batch can
   only be offset by a reversing entry.**" Button: **Release the batch** / **No**.
4. **Run R or RB** with `BatchNbr`. Read `processStatus` as `references/acumatica/screen-batches.md` says. Then run J7
   again: status `P` (or `U` where posting is separate) and `Released` true. Report both.

Release by number is how a held entry the skill wrote earlier gets released later, once reviewed.

## Reverse

A reversal is a new journal entry with the original's lines and **debit and credit swapped**, so
that the two net to nothing. The skill writes it with the same entry build and command sets as any
entry (`references/acumatica/screen-batches.md`), from J7's lines:

| Reversal field | Taken from J7 |
|---|---|
| Branch, ledger, currency | the original's (`Batch Branch Code`, `Ledger Code`, `Currency ID`) |
| Transaction date, post period | **ask**: the original's date and period (a correction in place) or the first day of the next open period (an accrual reversal). Never pick one. The period must be Open in J2. |
| Description | "Reversal of <batch>" followed by the original's description, unless the user gives one |
| Each line's branch, account, subaccount, project, task, cost code | the original line's, trimmed; project `X` passed as `X` |
| Each line's debit | the original's `Line Currency Credit Amount` |
| Each line's credit | the original's `Line Currency Debit Amount` |
| Each line's quantity, unit | the original's, `null` when `0` / empty |
| Each line's reference number | the original batch number, so every reversing line points back to it |
| Each line's description | "Reversal of <batch>: " followed by the original line's description |
| Non billable | the original's, `null` when false |

Use the **currency** amounts, not the base amounts: a reversal in the batch's own currency nets the
original exactly, whatever the rate.

Before the preview, check each reversing line against its J7 line: the original's debit is now the
credit, of the same amount, and the original's credit is now the debit. A line still on its original
side doubles the original instead of cancelling it.

Then the usual path: run the validation in `references/acumatica/validation.md` on the new lines (an account
since deactivated stops the reversal), preview with the original beside it, confirm (**Create the
reversing entry** / **No**), write on hold, read back with J6 and J7, and release only if the user
asked (`release` with `reverse`), with R on the new batch number. The report names both batches: the
original and its reversal, and that the two net to zero line by line.

**What this reversal does not do.** Acumatica's own Reverse action links the reversal to the
original (`Original Batch Number`). This skill cannot run it yet: the action opens the reversing batch
unsaved, and saving it needs a second call that the screen builder does not build today. The
reversal written here carries the link in each line's reference number and in its description
instead. Say so when the user asks for Acumatica's Reverse button by name.

