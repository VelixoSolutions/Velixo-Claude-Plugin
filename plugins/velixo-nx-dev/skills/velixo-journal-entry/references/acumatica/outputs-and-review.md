# Outputs: the preview, the confirmation, the report

## The preview (before anything is written)

Shown once the checks pass and the batches are built. In this order:

1. **Headline:** what will happen and where. "**Create a journal entry** in branch PRODWHOLE (Products
   Wholesale), ledger ACTUAL, dated 30 Jun 2025, period 06-2025 — **left on hold** for review." With
   release asked for: "… **and release it** (posts to the ledger)."
2. **Header:** description, currency, auto reversing, and the connection.
3. **Lines** as a table, in the order they will be written:

   | # | Branch | Account | Subaccount | Debit | Credit | Description | Project / task / cost code | Qty / UoM | Ref. | Non billable |
   |---|---|---|---|---|---|---|---|---|---|---|
   | 1 | PRODWHOLE | 63300 Utilities | 000-000 All others / Corporate | 1,250.00 | | June utilities accrual | — | — | — | — |
   | 2 | PRODWHOLE | 20150 Expense Purchase Accrual | 000-000 All others / Corporate | | 1,250.00 | June utilities accrual | — | — | — | — |
   | | | | **Total** | **1,250.00** | **1,250.00** | | | | | |

   Drop a column that is empty on every line and say which fields were left to Acumatica instead.
   Accounts with their names from J3; subaccounts split into segments with their descriptions from
   J4.
4. **What was derived or defaulted, and from where:** the period from the date (J2: 06-2025, Open),
   the ledger and currency from the branch (J1), a line description taken from the header, a project
   left to Acumatica's non-project default, a subaccount left to the account's default. A value the
   user did not state is never left out of this list.
5. **What was checked:** in one line — branch active, period open, accounts and subaccounts valid,
   balanced, no similar unposted batch in the period.
6. **Warnings:** when release was asked for, this sentence word for word: "**Releasing cannot be
   undone: once posted, the batch can only be offset by a reversing entry.**" Also an inter-branch
   entry; a control or cash account the user approved; a date outside its period.
7. **The question:** two options, the affirmative labelled with what it does — **Create the journal
   entry** (or **Create and release the journal entry**) — and **No**. Say the exact screen commands
   are available on request; show them only when asked.

A single-turn run, or any run where the user has not answered, ends here.

## The report (after the write)

Lead with the result:

> **Created GL batch 002371** in branch PRODWHOLE, period 06-2025 — **On Hold**, 2 lines, debits and
> credits 1,250.00. Open it on the Journal Transactions screen (module GL, batch 002371).

Then:

- **Tie-out:** the read-back against the approved lines — line count, each line, totals — "ties" with
  the figure, or each difference.
- **What Acumatica resolved:** every field left to its default, with the value it took when the read
  shows it.
- **Issues:** every warning or error Acumatica returned, as it returned it, in plain words.
- **Release:** when asked for, its outcome from `processStatus` — released and posted, pending
  approval, or not released and why.
- **Next step for a human:** review and release the held batch in Acumatica, or the decision a
  difference needs.

"Open after import: No" moves the batch-number line to the end; it is never omitted. A run that
stopped part way says what is saved: nothing, unless the saving call ran (check J5 before saying so).

## Checks before the reply leaves

- Every figure in the preview and the report comes from the entry or the read-back, never retyped from
  memory: totals are summed from the lines.
- Every value the user did not give is named with its source.
- Nothing is described as posted or released unless `processStatus` says so.
- No internal record ID, connection detail or raw tool error is shown.
