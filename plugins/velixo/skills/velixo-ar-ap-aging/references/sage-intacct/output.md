# Output

Plain Markdown unless the user asks for a document or a dashboard. Every output has the same three
parts in this order.

## 1. The answer

Two to four sentences: the total open, how much is past due and its share, the amount over 90 days,
and the one or two facts that matter most for the use (the largest overdue customer; bills overdue
already; DSO up five days; a gap to the GL). When the user asked for the tie, state it in one
sentence: "Differs from GL account 12000 by 12.5K (see reconciliation)." Otherwise say nothing about
a tie here: the basis line says the figures were not tied.

## 2. The table or tables

- **Aging report:** customer or vendor, Current, 1–30, 31–60, 61–90, over 90, Credits, Total; a
  total row; a share-of-total row. Largest total first. More than 25 rows: the top 25 and an "all
  others" row that keeps the total, and offer the rest.
- **Collections list:** customer, past due, over 60, over 90, oldest invoice (number, due date, days
  past due), credits to apply, share of AR.
- **Payment plan:** windows (overdue, next 7, 8–14, 15–30 days) with vendor, bill, due date, amount;
  vendor credits beside the vendor.
- **Cash timing:** overdue, then one column per week or month; one row per side.
- **Ratios and trend:** one row per period; the formula basis under the table.
- **Reconciliation:** branch or entity, account, period or date, GL, subledger, aging, difference to GL, difference to
  aging.
- **Clean-up:** one short table per list, with count and total.
- **Reserve:** bucket, balance, rate (the user's), reserve; total; and the GL reserve balance with the
  difference when the user named the account.

Full numbers with thousands separators in tables; `$X.XM` / `$X.XK` in prose. Dates as `30 Jun 2025`.
Names, never internal IDs; trim the padded codes the queries return. Amounts of two currencies never
share a total.

## 3. The basis line

One short block: side; branch or branches (entity or entities); the currency of the amounts; as-of period and date (or "now", with today's date); the
bucket basis (due date or document date, cut-offs); the query and level run; the tie to the GL with
its difference, or "not tied to the AR history or the GL" when it was not asked for; anything left
out (zero-balance documents, adjustment periods, cash sales in sales); when a period was given, that
the as-of date assumes calendar-month periods. A reader must be able to rerun the analysis from it.

Then the next steps, as choices the user can tap, in one line: the tie to the GL, the collections
list or payment plan, the ratios. Run none of them unasked.

## Wording

- "Past due", "over 90 days", "open", "credit": facts. "At risk", "doubtful", "disputed": only when
  the user or a record says so.
- A reconciliation difference is "unexplained" until the GL lines are read: name the next step.
- Ratios are named with their basis on first use: "DSO (ending AR, 30 days) 41.8 days".