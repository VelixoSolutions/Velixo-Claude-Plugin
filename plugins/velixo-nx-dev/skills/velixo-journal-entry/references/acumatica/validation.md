# Validation

Every check below runs before a batch is built. A **stop** is not overridable: the entry cannot be
written as given, and the preview is replaced by what is wrong and what would fix it. An **ask** is
the user's decision: put it to them with the choices laid out, in the same message as any other
question, and record their answer in the preview. A check that passes is listed in the preview's
basis line, not narrated.

## Before the write

| Check | Source | Result when it fails |
|---|---|---|
| Connection usable | `list_connections` | Stop; relay its status (the user fixes it in the Velixo Portal). |
| Branch exists and is active | J1 | Stop; offer the active branches. |
| Line branches exist and are active | J1 | Stop on that line. A line branch in another company: ask, and say Acumatica needs an inter-branch account mapping to release it. |
| Ledger is the branch's posting ledger | J1 | A named non-posting ledger is passed as given; say it was not checked. |
| Period exists | J2 | Stop; say the period is not generated in the calendar. |
| Period is Open | J2 | Stop; name the status (Closed, Locked, Inactive). Never suggest reopening it as a step the agent takes. |
| Period contains the date | J2 | Ask: Acumatica accepts a date outside its post period, and a reviewer will question it. |
| Adjustment period | J2 | Ask unless the user named period 13. |
| Accounts exist | J3 | Stop on each; never substitute a similar account. |
| Accounts are active | J3 | Stop on each. |
| Control account (`Control Account Module` set) | J3 | Ask, line by line: posting to an AR, AP, inventory, fixed asset, tax, deferral or purchasing control account from the GL puts the ledger out of step with the subledger, and Acumatica may refuse it unless the account allows manual entry. The usual right answer is a document in the subledger. |
| Cash account (`Cash Account` true) | J3 | Ask: a GL entry to a cash account bypasses Cash Management and shows up as a reconciling item. |
| Account currency (`Currency` set) | J3 | Stop when it differs from the entry's currency. |
| Subaccount splits and each segment value is valid and active | J4 | Stop on each line; show which segment and value, and the valid values when there are few. |
| Subaccount missing on a tenant with segments | J4 | Ask once for all such lines; "use each account's default subaccount" is one choice (`null`). |
| Project task given without a project, or a cost code without a project | — | Ask. |
| Quantity without a unit, or a unit without a quantity | — | Ask. |
| Every line has exactly one of debit or credit, positive | the entry | Ask about each line that does not. |
| At most two decimals (the entry's currency precision) | the entry | Ask; never round silently. |
| **Total debits = total credits** | the entry | **Stop.** State both totals and the difference ("debits 1,250.00, credits 1,200.00, out of balance by 50.00"). Never add a line, resize one or round to close it. |
| Possible duplicate | J5 | Ask, showing the batch that matches. |
| Last batch number given | J6 | That batch already holds these lines: ask before writing again. |
| More than 1,000 lines | the entry | Ask whether to write it here or use an import scenario. |

Balance is checked in the entry's own currency, line amounts as given. Report totals with thousands
separators and two decimals.

Projects, tasks and cost codes are not in the library: Acumatica validates them as each line is
written, and its message is relayed (`references/acumatica/screen-batches.md`).

## After the write

The read-back is the record of what was written. Tie it before reporting success:

| Check | Compare | Result when it fails |
|---|---|---|
| Batch exists, status On Hold | J5 row for the batch number | Report what J5 shows; nothing is released. |
| Line count | J6 rows against the preview | Report the missing or extra lines by number. |
| Each line | J6 `Account`, `Subaccount`, `Debit Amount`, `Credit Amount` against the preview line of the same number | Report each difference with both values. |
| Totals balanced | J5 `Debit Total` = `Credit Total`, and (base-currency entry) = the preview's total | Report both figures and the difference. |

In a foreign-currency entry, J5 and J6 carry base-currency amounts: tie the entry's own totals to the
finish set's read-back instead, and show the base-currency totals beside them as the converted
figures.

A difference is reported, with the batch left on hold for a human. It is never corrected by another
write without a new preview and confirmation, and release is not run.
