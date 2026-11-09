# Reclassify posted lines

Acumatica's own reclassification: lines of a posted batch moved to another account, subaccount or
branch, or split across several, through the Reclassify Transactions screen. Acumatica writes a
**reclassification batch** that credits the old coding and debits the new, and links it to each
original line (`ReclassBatchNbr`). That link is why this is not done as a journal entry.

Three forms of request:

| The user says | Form | Values |
|---|---|---|
| "on GL1234, move what is on 4000 to 4001" | **by account** | `BatchNbr`, `FromAccount`, `ToAccount` |
| "line 2 of GL1234 to subaccount 123-456" | **by line, new subaccount** | `BatchNbr`, `LineNbr`, `ToSub` (no separators: `123456`) |
| "split line 1 of GL1234: 3.00 to 63100, the rest to 63000" | **split** | `BatchNbr`, `LineNbr`, `FirstAccount` (the remainder), `SecondAccount`, `SplitAmount` |

"Move from subaccount 000-000 to 123-456" with no line named: look the batch up (J7), list the lines
on that subaccount, and run the by-line form for each, or ask which lines when it is not obvious. A
change the forms above do not cover (a new branch, date, period or description; a split in three or
more) is asked of the builder the same way, in the user's words with placeholders; say in the preview
that it has not been run live before.

## 1. Look the batch up and check it

Run **J7** with the batch number. The batch must be **Posted** (`P`) or Completed: an unposted batch is
corrected in place, not reclassified. Every line the request names must exist (by account: at least
one line on `FromAccount`; by line: that `Line Number`). Check the destination with J3 and J4 like any
entry line: the account exists and is active, is not a control account without the user's say, and
the subaccount's segment values are valid. A split amount is positive and below the line's amount.
The period of the batch must still be open for a reclassification dated in it (J2).

## 2. Build, then preview without processing

Build through `prepare_screen_commands` with the requests below, as written. The builder answers with
one batch on Journal Transactions that runs Reclassify and then `ContinueOn GL506000`, so every later
command runs on the Reclassify Transactions screen the action opened. **The lines Reclassify loads live
only inside that one call**: a batch with `ContinueOn` is refused with `leaveOpen` or `continuation`,
and a second call cannot reach them. So the flow is two complete calls with the same values:

1. **Preview call** — the edits and a read-back of every loaded row, **nothing processed**. Running it
   writes nothing, so it needs no confirmation; it is what the confirmation is about.
2. **Process call** — the same edits, ending with `ValidateAndProcess`. Only after a yes.

**Preview request** (by account):

> Preview reclassifying the lines of posted GL batch {{BatchNbr}} that are on account {{FromAccount}} to
> account {{ToAccount}}, using Acumatica's Reclassify Transactions screen. Do not process anything. Read
> back every loaded line: BatchNbr, LineNbr, AccountID, SubID, NewAccountID, NewSubID, NewBranchID,
> debit, credit and Selected.

For the by-line and split forms, ask the same way: "Preview reclassifying line {{LineNbr}} …" /
"Preview splitting line {{LineNbr}} …", with the read-back list above and "Do not process anything".
Check the built batch before running it: it contains no `ValidateAndProcess`. If it does, do not run
it; rebuild once, then stop and report.

Show the read-back as the preview: one row per loaded line — line, account and subaccount, debit,
credit, the new account, subaccount and branch, and whether it will be reclassified (`Selected`). Only
the selected rows move. Name accounts and subaccounts (J3, J4). State what the reclassification batch
will do: for each selected line, the old coding credited (or debited, for a credit line) and the new
coding charged, for the line's amount. Then the question, with the warning that a processed
reclassification is undone only by another reclassification: **Reclassify the lines** / **No**.

## 3. Process

After a yes, build and run the process request for the same form, with the **same values**:

> Reclassify the lines of posted GL batch {{BatchNbr}} that are on account {{FromAccount}} to account
> {{ToAccount}}, using Acumatica's Reclassify Transactions screen, and process the reclassification.
> Read back each line's batch number, line number, new account and reclassification batch number before
> positioning on any line. Leave every other field at its default.

> Reclassify line {{LineNbr}} of posted GL batch {{BatchNbr}} to subaccount {{ToSub}}, using Acumatica's
> Reclassify Transactions screen, and process the reclassification. Leave every other field at its
> default.

> Split line {{LineNbr}} of posted GL batch {{BatchNbr}} with Acumatica's Reclassify Transactions screen:
> the line goes to account {{FirstAccount}} for its remaining amount, and a split of {{SplitAmount}} goes
> to account {{SecondAccount}}. Process the reclassification. Leave every other field at its default.

Read the result:

- `processStatus` `completed` and a `ReclassBatchNbr` in the read-back: the reclassification batch
  number. Report it.
- `failed`, or no reclassification batch number: nothing was reclassified; relay each row's message in
  Acumatica's words. Before any retry, check the read-back's `ReclassBatchNbr`, and when in doubt look
  for a new Reclassification batch in J5 for the branch and period. Never run the process call twice
  blind: it would reclassify twice.

## 4. Read back, then release only when asked

Run **J7** on the reclassification batch number. Tie it to the preview: one credit of the old coding and
one debit of the new for each selected line, for the line's amount (a split: one pair per part), totals
equal. Its status is **Balanced** (`B`), and its `BatchType` is Reclassification: GL history has not
moved yet.

**Release** only when the user asked for it (`release` with `reclassify:`) or says so now, with the
**release build for a balanced batch** (`references/acumatica/release-and-reverse.md`): a reclassification batch is
never on hold, and the on-hold release build fails on it ("The Remove Hold button is disabled.").

## What the builder takes care of

The batches rely on rules the builder learned from the GL506000 screen note; they explain results:

- Reclassify shows an OK-only message when the batch holds a line already reclassified ("Some
  transactions of the batch cannot be reclassified"); the batch answers it before the action.
- Lines are positioned by `BatchNbr` and `LineNbr`, never by read order. Setting a new destination
  selects the row; `Selected` cannot be set directly.
- Every part of a split must go somewhere other than the original coding, or the whole group is
  refused ("not all lines of the splitting group have been selected"). The part a split adds has
  `LineNbr` `-2147483648`.
- `Replace` carries `commit: true` before `ValidateAndProcess`; without it the process finds nothing
  selected and reports success having done nothing. A "completed" process with no reclassification batch
  number is that case: report that nothing was reclassified.

