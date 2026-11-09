# Views and dashboards

Two views answer most visual requests. Build the one the user asks for; when they ask for "a
dashboard" without saying which, build the outstanding review and offer the second, which needs the
history and the GL.

## View 1 — Outstanding review

**For** the person who works the balances: the AR clerk chasing, the AP clerk paying, the controller
reviewing before close. **From** the aging (A1, or P1 for AP) alone.

- **Cards, in this order:** total open; past due and its share; over 90 days and its share; DSO (AR)
  or DPO (AP) when the history for the period was pulled. Each card shows the previous period's
  figure beneath it when the user asked for a trend.
- **Bucket bar:** one horizontal bar, 100% wide, split into Current, 1–30, 31–60, 61–90 and over 90,
  each labelled with its amount and share. Credits are not a segment: show them as a figure beside
  the bar. Series colours follow `velixo-design`'s order (Current `--brand`, then `--ink`, `--muted`,
  `--brand-dark`, `--brand` at 60%); status colours are never used for buckets.
- **Top balances:** the top 10 customers or vendors by over-90 balance (AR) or past-due balance (AP),
  as a horizontal bar chart, largest first, with the "all others" total in text.
- **The table:** customer or vendor, Current, 1–30, 31–60, 61–90, over 90, Credits, Total, largest
  total first; a total row; numbers right-aligned. A row expands to its
  documents (number, document date, due date, days past due, open balance) when the page is
  interactive.
- **The tie strip** at the foot, only when the user asked for the tie: "AR history 4,210,500.00 · GL
  12000 4,223,000.00 · differs by 12,500.00", with "Ties" or "Differs" in words beside the colour.
  Otherwise the foot says "Not tied to the GL".

## View 2 — Total by period against the GL (when asked)

**For** the controller and the reviewer of the close. **From** the history (A2 or P2) over the window,
and the GL control account (G1, detail level, carried forward by subaccount).

- **Chart:** the subledger balance at each period end as columns (`--brand`), the GL balance as a
  line over them (`--ink`). Leave adjustment periods (`13YYYY`) out.
- **Table:** period, branch, control account, subledger balance, GL balance, difference, status
  ("Ties" / "Differs" in words, with `--good` or `--alert`). One row per period; a total per period
  when there are several branches.
- **Under the table:** where the difference first appears or moves (a constant difference started
  before the window), and the next step (the account's GL lines for that period). Never a cause
  without those lines.
- **Ratios, when pulled:** a second small chart of DSO or DPO by period, with its basis in the
  caption.

## The page

An HTML page follows the `velixo-design` skill (its `references/dashboard-skeleton.md` is the
starting point): header with the scope line (company, branches, as-of period and date, currency,
"Preliminary" when the period is open), cards, then charts, then tables, then the footer. The footer
names the queries run, the run time, and the Acumatica report each figure ties out to (AR630500 or
AP630500 for the aging total, the GL control account for the history).

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
| Outstanding review | `ar-aging` or `ap-aging` (one query each, as-of optional) | `Fin Period` or `As Of Date` (both blank for now), `Branch ID` |
| Total by period | `ar-subledger-balance-by-period` or `ap-subledger-balance-by-period` | `Period From`, `Period To`, `Branch ID`, `Business Account` |
| The GL side | `gl-account-balances-by-period-range`, detail level | `Period From`, `Period To`, `Branch ID`, `Account CD` |

The aging workbook holds documents with their due dates, not buckets. Tell the user that a bucket
column is one formula on the due date and the as-of date, and give them that formula when they want
the buckets in the workbook.
