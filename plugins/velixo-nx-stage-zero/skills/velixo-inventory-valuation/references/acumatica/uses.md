# The uses

Each use answers a different question from the same pulls. Run the pulls once and reuse them when the
user moves from one use to the next.

## Valuation

**Question.** What is on hand at the period end, and what is it worth?

**Output.** The total per branch, then by inventory account and subaccount (the grain the GL ties
at), then the items: item, description, warehouse, quantity, value, unit cost (value ÷ quantity),
largest value first. Lead with the total, the number of items, and the share of value in the top 10
items. More than 25 items: the top 25 and an "all others" row that keeps the footing; offer the rest
or a workbook.

## Total by period

**Question.** How has the inventory balance moved, and does it equal the GL each month?

**Output.** One row per period: subledger value, GL value, difference, and the month's change. Per
branch, and a total. A difference that is constant across the window started before it; one that
moves started in the period where it moves.

## Aging

**Question.** How long has stock sat since it last moved?

**Output.** Value and item count per band of months since last activity (0–3, 4–6, 7–12, 13–24, over
24 by default), with the share of total value; then the items in the oldest bands, largest value
first, with their status. The bands come from `Last Activity Period`: say that it is the last
movement of any kind, not the last receipt or the last sale.

## Turnover and DIO

**Question.** How fast does stock turn, and is it getting faster or slower?

**Output.** One row per period: cost of sales, average inventory, turnover (annualised) and DIO.
Formulas and traps are in `aging-and-ratios.md`. Name the cost of sales accounts used.

## Reconciliation to the GL

**Question.** Does the inventory subledger equal the GL inventory account?

**Output.** Per branch, account and subaccount, for each period asked: GL balance (with the period of
the row it came from when carried forward), subledger balance, difference. The next step for a
difference is the GL lines of that subaccount in the period where it appears
(`velixo-account-drilldown`): journal entries posted straight to the inventory account, or
inventory documents posted to another account, are the usual sources, but which one needs the lines.

## Clean-up

**Question.** What in the subledger needs attention?

**Output.** Lists, each with count and value: items with quantity and no cost; items with cost and no
quantity; negative quantities or values; items marked inactive or for deletion that still hold
stock; items unmoved for more than 24 months. Each list names who would act (inventory controller,
cost accountant) without acting.
