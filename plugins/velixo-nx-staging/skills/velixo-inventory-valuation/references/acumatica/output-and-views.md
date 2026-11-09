# Output and views

Plain Markdown unless the user asks for a dashboard, a chart or Excel.

## Every output

1. **The answer**, two to four sentences: the total value, the number of items, the one or two facts
   that matter most for the use (the largest items; the value unmoved for over a year; DIO up eight
   days; a difference to the GL on one subaccount). Then the tie in one sentence: "Equals GL account
   12100 on every subaccount (4.36M); compare with Acumatica's inventory valuation report IN617000 for
   the same branch and period."
2. **The table or tables** for the use (`uses.md`). Full numbers with thousands separators in
   tables, `$X.XM` / `$X.XK` in prose; quantities with the item's decimals; trimmed codes, with the
   description beside every item code.
3. **The basis line:** branch or branches; as-of period and date; the levels and pulls used; the
   footing (detail to summary, aging to valuation); the GL tie per subaccount, naming any GL row
   carried forward and its period; the report it ties out to; anything left out (zero-cost or
   zero-quantity items, adjustment periods). A reader must be able to rerun it from this line.

## View 1 — Detailed review of what is on hand

**For** the inventory controller and the reviewer at close. **From** I2 for the period (I3 when the
aging is asked for), footed to I1.

- **Cards:** total value; number of items with stock; value unmoved over 12 months and its share;
  DIO when cost of sales was pulled.
- **Value by warehouse:** horizontal bars, largest first.
- **Aging bands:** one 100% bar split into the bands, each labelled with value and share, series in
  the `velixo-design` order (0–3 `--brand`, then `--ink`, `--muted`, `--brand-dark`, `--brand` at
  60%). Status colours are never used for bands.
- **The table:** item, description, warehouse, account, subaccount, quantity, value, unit cost, last
  activity period, status; largest value first; a total row equal to I1. Filters by warehouse,
  subaccount and band when the page is interactive.

## View 2 — Total by period against the GL

**For** the controller and the close reviewer. **From** I1 over the window and G1 carried forward by
subaccount.

- **Chart:** inventory value at each period end as columns (`--brand`), the GL balance as a line
  (`--ink`). Leave `13YYYY` out.
- **Table:** period, branch, account, subaccount, subledger, GL (with the period of a carried-forward
  row), difference, status ("Ties" / "Differs" in words, with `--good` or `--alert`).
- **Under it:** where a difference appears or moves, and the next step. Never a cause without the GL
  lines.

## The page

An HTML dashboard follows the `velixo-design` skill (start from its
`references/dashboard-skeleton.md`): header with company, branches, as-of period and date, currency
and "Preliminary" when the period is open; cards; charts; tables; a footer naming the queries, the
run time, and what each figure ties out to (IN617000 for the valuation, the GL inventory account for
the total).

**A refreshable dashboard is a Velixo workbook.** When the user asks for a dashboard they can
refresh, an Excel file, or "something I can rerun each month", call `create_workbook_from_query`
once for each view they want, with the query exactly as this skill vendors it (`DECLARE` lines
included) and the same parameter values you ran it with. The workbook holds the query as a live
formula, not its rows: it fetches current figures each time it is opened or refreshed, and its
parameters are editable cells, so a new period is a new value, not a new workbook. Give the user each
link and name what it holds and which cell to change. Never create one unasked; offer it once, at the
end, as a choice.

- **Excel keeps the connector's own layout.** Do not apply the `velixo-design` palette to a workbook.
- **An HTML dashboard is a snapshot.** It shows the figures of this run and is styled with the
  `velixo-design` skill. Say that it does not refresh, and offer the workbooks beside it when the user
  wants to rerun it.
- **Each workbook runs one query.** The tie between two views (subledger against GL) is not
  recomputed in Excel: say which two workbooks and which columns to compare.
**The workbooks for these views:**

| View | Workbook query | Parameters the user changes |
|---|---|---|
| Detailed review | `in-item-cost-balance-by-period`, detail level | `Period From` = `Period To`, `Branch ID` |
| Aging | `in-inventory-aging` | `Fin Period`, `Branch ID` |
| Total by period | `in-item-cost-balance-by-period`, summary level | `Period From`, `Period To`, `Branch ID` |
| The GL side | `gl-account-balances-by-period-range`, detail level | `Period From`, `Period To`, `Branch ID`, `Account CD` |

The aging workbook holds `Last Activity Period`, not bands: give the user the formula for a band
column when they want it. The GL workbook holds rows only for periods with activity, so a
subaccount's balance in a quiet period is its last row above it: say so beside the link.

## Wording

- "Unmoved for 30 months", "no quantity", "negative": facts. "Obsolete", "dead", "excess": only when
  the user or a record says so.
- A difference to the GL is "unexplained" until the GL lines are read: name the next step.
