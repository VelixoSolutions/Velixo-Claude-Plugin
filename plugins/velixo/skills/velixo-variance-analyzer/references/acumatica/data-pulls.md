# Data pulls from Acumatica

These queries supply the scope, the ledger activity, the plan and the transaction-level evidence a
variance decomposition and root-cause investigation needs. Run them through the Velixo connector's
`execute_query` as written, with your values in the `parameters` map. Do not call `prepare_query` for
them and do not edit the text. If one fails, report the error and stop rather than reformulating it.
When no pull below returns the grain the question needs, build that one query through `prepare_query`
(never by hand), and name it in the basis block as a query outside the library, with what it returned
and the tie-out it passed.

**Parameter keys are the display labels** — the text after `AS` in each `DECLARE` — and the table
under each pull lists them. The same concept has the same label on every query: `Period From` /
`Period To` for a range, `Fin Period` for a single period, `Branch ID`, `Ledger ID`, `Account CD`,
`Account Class`, `Subaccount CD`. A key that matches no label is dropped without an error and the
query returns nothing, which looks exactly like "no data". **Pass every key a pull declares**: an
optional one you are not using goes in as `null`. A key left out of the map is not the same as a null —
the unposted-batch lines below return nothing when `Batch Number` is missing and every batch's lines
when it is `null`.

**Period values** use the financial period format the organization's users see (typically `MMYYYY`,
for example `032026` for period 3 of 2026), passed exactly as the user wrote them. Never rewrite one
into the year-first `202603` that Acumatica stores internally — the connector translates, and a
year-first value matches no period, so the pull returns no rows. `BETWEEN` on a financial period is
chronological, not lexical, so ranges that cross a year end work correctly.

**Always set `top`** — start at 250, and raise it (up to 10,000) or page with `skip` if the result
comes back flagged `truncated`. Never analyse a truncated result as the whole population, and never
report a gap as a finding without first ruling out truncation.

**Sizing rule: pull at the grain you need.** When a per-period pull would return more than about 250
rows and the analysis only needs the window total, run the window-summed variant instead — pull 4b for
pull 4, pull 4c for two windows side by side, pull 8b for pull 8 over a range. Never page through a
per-period pull just to add it up: the sum costs one call at the right grain, and paging is where rows
go missing.

**One branch per call** for pulls 5 to 9, and for pull 4 whenever the figure has to tie to a branch.
For a company or group view, run per branch and combine only branches on the same fiscal calendar
and ledger.

**`Natural Net` is the column to present.** Every activity, balance and budget query here returns it:
the amount in the account's natural sign — credit less debit on income and liability (equity
included) accounts, debit less credit on asset and expense — so revenue, expense, an asset and a
liability all read positive when they grow, and budget subtracts from actual with no sign flip.
Keep the raw `Debit` / `Credit` columns for the pattern reads (credits netting invoices, allocations),
and the older debit-less-credit or credit-less-debit nets only for tie-outs between queries. Label a
variance by account type, never by sign: actual above budget on `Natural Net` is favourable on
income and unfavourable on expense. State the convention once, in the basis block.
| Pull | Returns | Query |
|---|---|---|
| 1 | Financial periods in a range, with dates, status and adjustment flag | `financial-period-calendar-range` |
| 2 | Every GL account with type, class, control module, cash flag, currency | `gl-chart-of-accounts-classification` |
| 3 | Every branch with its company, ledger, base currency and status | `gl-branch-company-ledger-list` |
| 4 | Income and expense movement per period at account grain, with type and class | `gl-pl-actuals-by-period-and-account` |
| 4b | Pull 4 summed over a window: one row per account | `gl-pl-actuals-by-account-for-window` |
| 4c | Two windows per account: current, baseline and change | `gl-pl-actuals-window-comparison` |
| 5 | GL activity by account, subaccount, ledger and period | `gl-account-balances-by-period-range` |
| 6 | Budget by account and period | `gl-budget-by-account-and-period` |
| 7 | Posted transaction lines on actual ledgers, every filter optional | `gltran-lines` |
| 8 | Unposted GL batches in a period, with totals and the debit-credit gap | `gl-unposted-batches-by-period` |
| 8b | Pull 8 over a period range, the period as a column | `gl-unposted-batches-by-period-range` |
| 9 | Invoiced revenue by customer and period | `ar-revenue-by-customer-and-period` |

The shared method files call pull 7 **the transaction-lines pull**, and pull 2 **the chart pull**:
on Acumatica it proposes the cost-of-sales and trading-revenue grouping from account class.

## Which pulls an analysis needs

| Entry point | Scope and structure | The gap | Cause and evidence |
|---|---|---|---|
| Income statement flux | 1, 2, 3 | 4c for the gap (or 4b per window), 4 for the shape, 5 for the detail | 7 per line, 5 at subaccount grain, 8 or 8b where a period is open |
| Budget-to-actual | 1, 2, 3 | 5 and 6 at the same grain | 7 per line, 5 at subaccount grain, 8 where the period is open |
| Forecast-to-actual | 1, 2, 3 | 5, against the forecast as supplied | 7 per line; 6 only if the forecast version was loaded as a budget |
| Single cost line | 1, 2, 3 | 5 narrowed with `Account CD` | 7 for that account, 5 at subaccount grain, 8 where the period is open |
| Revenue mix and customer effects | 1, 2, 3 | 5 on the revenue accounts | 9 alongside it |

Pull 1 runs for every window used, to label periods, get the remaining-period count right and flag
open ones. Pull 5 is run twice on most analyses: once wide and coarse for the gap, once narrow and
deep for the investigation. Where the account drill-down is in play (Step 5), it takes over pulls 5
and 7 at business-account, segment and branch grain.

**Activity and rate data has no pull here.** Unit volumes, hours, headcount, standard costs and
contracted rates are not in the general ledger. They come from an operational system, from a licensed
module where one is configured, or from the user. Where they are absent the volume, rate and
efficiency components are not produced — see the decomposition reference. Do not derive a volume from
spend and an assumed rate.

## Pull 1 — Financial period calendar for a range

```giql
DECLARE CompanyID SCHEMA [PX.Objects.GL.DAC.Organization].OrganizationCD AS 'Company ID' REQUIRED
DECLARE PeriodFrom SCHEMA [PX.Objects.GL.FinPeriods.TableDefinition.FinPeriod].FinPeriodID AS 'Period From' REQUIRED
DECLARE PeriodTo SCHEMA [PX.Objects.GL.FinPeriods.TableDefinition.FinPeriod].FinPeriodID AS 'Period To' REQUIRED
FROM [PX.Objects.GL.FinPeriods.TableDefinition.FinPeriod] As FinPeriod
INNER JOIN [PX.Objects.GL.DAC.Organization] As Organization ON FinPeriod.OrganizationID = Organization.OrganizationID
WHERE Organization.OrganizationCD = CompanyID AND FinPeriod.FinPeriodID BETWEEN PeriodFrom AND PeriodTo
ORDER BY FinPeriod.FinPeriodID
SELECT FinPeriod.FinPeriodID AS 'Financial Period ID', FinPeriod.StartDate AS 'Start Date', FinPeriod.EndDate AS 'End Date', FinPeriod.Status AS 'Status', FinPeriod.IsAdjustment AS 'Adjustment Period'
```

| Key | Required | Binds to |
|---|---|---|
| `Company ID` | yes | `GL.DAC.Organization.OrganizationCD` |
| `Period From` | yes | `GL.FinPeriods.TableDefinition.FinPeriod.FinPeriodID` |
| `Period To` | yes | `GL.FinPeriods.TableDefinition.FinPeriod.FinPeriodID` |
Reading it:

- **`Status`** — `Closed` or `Locked` means final; `Open` means the figures are preliminary and every
  variance computed on them is labelled so. `Inactive` or no row means the range is wrong or the
  calendar isn't generated that far.
- **`End Date` is the start of the next period.**
- **Period lengths come from here, not from a calendar assumption.** A run rate, a per-day figure or a
  remaining-period projection computed on "twelve even months" is wrong wherever the calendar isn't.
- **Adjustment periods** have zero days but real postings. Include them in full-year windows, and
  report a variance that lands in one as an adjustment rather than as operating movement.
- **Calendars are per company.** Run it for the company that owns each branch.

## Pull 2 — Chart of accounts with classification attributes

```giql
FROM [PX.Objects.GL.Account] As Account
LEFT JOIN [PX.Objects.GL.AccountClass] As AccountClass ON Account.AccountClassID = AccountClass.AccountClassID
ORDER BY Account.Type, Account.AccountCD
SELECT Account.AccountCD AS 'Account', Account.Description AS 'Description', Account.Type AS 'Type', Account.AccountClassID AS 'Account Class', AccountClass.Descr AS 'Account Class Description', Account.Active AS 'Active', Account.ControlAccountModule AS 'Control Account Module', Account.IsCashAccount AS 'Cash Account', Account.CuryID AS 'Currency'
```

No parameters.
Pass a `top` large enough for the whole chart. Reading it:

- **`Type`** is `A`, `L`, `I` or `E`. Only `I` and `E` enter a P&L variance.
- **This is where cost of sales gets identified, and it cannot be done from `Type`.** Acumatica has no
  cost-of-sales account type, so a COGS account is typed `E` exactly like rent. `Account Class` and its
  description are the organization's own grouping and often make the split; a `Control Account Module`
  of `IN` or `SO` on an Expense account is a further signal. Propose a grouping from these, then have
  the user confirm it before any gross margin or contribution figure is produced.
- **The Income type is not the revenue line.** It also carries freight income, interest income, other
  income and discount accounts that post negative. Confirm which accounts are trading revenue before
  decomposing a revenue variance.
- **Run this once per analysis** and keep it — every later pull returns account codes that have to be
  read against it.

## Pull 3 — Branches, companies and ledgers

```giql
FROM [PX.Objects.GL.Branch] As Branch
INNER JOIN [PX.Objects.GL.DAC.Organization] As Organization ON Branch.OrganizationID = Organization.OrganizationID
INNER JOIN [PX.Objects.GL.Ledger] As Ledger ON Branch.LedgerID = Ledger.LedgerID
ORDER BY Organization.OrganizationCD, Branch.BranchCD
SELECT Branch.BranchCD AS 'Branch ID', Branch.AcctName AS 'Branch Name', Organization.OrganizationCD AS 'Company ID', Organization.OrganizationName AS 'Company Name', Ledger.LedgerCD AS 'Ledger ID', Branch.BaseCuryID AS 'Base Currency ID', Branch.Active AS 'Active'
```

No parameters.
Reading it:

- **`Branch ID`** is the value to pass to pulls 4 to 9; **`Company ID`** is the value for pull 1.
- **`Ledger ID`** is the branch's posting ledger — the value to supply to pulls 4, 5 and 7. Where more
  than one actual ledger exists, leaving that filter empty sums them together silently and inflates
  every variance downstream with no error anywhere. The budget ledger for pull 6 is a different value,
  from the user.
- **Different base currencies** across branches mean the figures need translation before they are
  added, and translation movement is then one of the decomposition's components rather than part of
  "other".
- **This pull is how scope gets stated rather than assumed.** The company, branch and ledger named in
  the analysis's basis block come from here.

## Pull 4 — Income and expense movement per period, at account grain

The trend spine. Run it across a wide window before anything else — history is the first rung of the
evidence ladder and this is the least expensive way to get it. It returns one row per income or expense
account per period, each carrying its `Type` and `Account Class`, so the shape by type, by class or by
a grouping the user has confirmed is a client-side sum.

```giql
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period To' REQUIRED
DECLARE [BranchID] SCHEMA [PX.Objects.GL.GLHistory].BranchID AS 'Branch ID'
DECLARE [LedgerID] SCHEMA [PX.Objects.GL.GLHistory].LedgerID AS 'Ledger ID'
DECLARE [AccountClass] SCHEMA [PX.Objects.GL.Account].AccountClassID AS 'Account Class'
FROM [PX.Objects.GL.GLHistory] As Hist
INNER JOIN [PX.Objects.GL.Account] As Account ON Hist.AccountID = Account.AccountID
WHERE Hist.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo]
AND Hist.BalanceType = 'A'
AND (Account.Type = 'I' OR Account.Type = 'E')
AND (Hist.BranchID = [BranchID] OR [BranchID] IS NULL)
AND (Hist.LedgerID = [LedgerID] OR [LedgerID] IS NULL)
AND (Account.AccountClassID = [AccountClass] OR [AccountClass] IS NULL)
GROUP BY Hist.FinPeriodID, Account.AccountID
ORDER BY Account.AccountCD, Hist.FinPeriodID
SELECT Hist.FinPeriodID AS 'Financial Period', Account.AccountCD AS 'Account', Account.Description AS 'Description', Account.Type AS 'Type', Account.AccountClassID AS 'Account Class', SUM(Hist.FinPtdDebit) AS 'Fin. PTD Debit', SUM(Hist.FinPtdCredit) AS 'Fin. PTD Credit', {=SUM([Hist.FinPtdCredit])-SUM([Hist.FinPtdDebit])} SCHEMA [PX.Objects.GL.GLHistory].FinPtdCredit AS 'Net Period Movement', SUM({=IIf([Account.Type]='I' OR [Account.Type]='L',[Hist.FinPtdCredit]-[Hist.FinPtdDebit],[Hist.FinPtdDebit]-[Hist.FinPtdCredit])}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdCredit AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Period From` | yes | `GL.GLHistory.FinPeriodID` |
| `Period To` | yes | `GL.GLHistory.FinPeriodID` |
| `Branch ID` | no | `GL.GLHistory.BranchID` |
| `Ledger ID` | no | `GL.GLHistory.LedgerID` |
| `Account Class` | no | `GL.Account.AccountClassID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
Reading it:

- **Present `Natural Net`.** `Net Period Movement` (credit less debit) stays for tying to pull 6's
  `Budgeted Amount`, which uses the same convention.
- **Summed by `Type` it is two lines, never three.** That is total expense, never gross margin, for the
  reason in pull 2. Sum by `Account Class` instead, once the user has confirmed which classes make up
  cost of sales.
- **Supply `Branch ID` and `Ledger ID`.** Both are optional here, and omitted they sum every branch and
  every actual ledger together. Pull 5 ties to this only when both calls carry the same scope.
- **A missing period had no income or expense movement at all.** GL history carries rows only where
  there was activity, so confirm against pull 1 that the period exists and is open before reading a
  gap as a genuinely flat month.
- **This is the spike / step / drift / noise read.** Put the periods side by side before investigating
  anything: a spike goes straight to pull 7, a step wants the date of the step, and a drift is usually
  the finding that matters most.

## Pull 4b — Income and expense per account, summed over a window

Pull 4 without the period column: the window total per account, in one row. Use it for a full-year or
multi-period window whenever the monthly shape is not what is being read — a year is one row per
account here and thirteen in pull 4.

```giql
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period To' REQUIRED
DECLARE [BranchID] SCHEMA [PX.Objects.GL.GLHistory].BranchID AS 'Branch ID'
DECLARE [LedgerID] SCHEMA [PX.Objects.GL.GLHistory].LedgerID AS 'Ledger ID'
DECLARE [AccountClass] SCHEMA [PX.Objects.GL.Account].AccountClassID AS 'Account Class'
FROM [PX.Objects.GL.GLHistory] As Hist
INNER JOIN [PX.Objects.GL.Account] As Account ON Hist.AccountID = Account.AccountID
WHERE Hist.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo]
AND Hist.BalanceType = 'A'
AND (Account.Type = 'I' OR Account.Type = 'E')
AND (Hist.BranchID = [BranchID] OR [BranchID] IS NULL)
AND (Hist.LedgerID = [LedgerID] OR [LedgerID] IS NULL)
AND (Account.AccountClassID = [AccountClass] OR [AccountClass] IS NULL)
GROUP BY Account.AccountID
ORDER BY Account.AccountCD
SELECT Account.AccountCD AS 'Account', Account.Description AS 'Description', Account.Type AS 'Type', Account.AccountClassID AS 'Account Class', SUM(Hist.FinPtdDebit) AS 'Fin. PTD Debit', SUM(Hist.FinPtdCredit) AS 'Fin. PTD Credit', {=SUM([Hist.FinPtdCredit])-SUM([Hist.FinPtdDebit])} SCHEMA [PX.Objects.GL.GLHistory].FinPtdCredit AS 'Net Period Movement', SUM({=IIf([Account.Type]='I' OR [Account.Type]='L',[Hist.FinPtdCredit]-[Hist.FinPtdDebit],[Hist.FinPtdDebit]-[Hist.FinPtdCredit])}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdCredit AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Period From` | yes | `GL.GLHistory.FinPeriodID` |
| `Period To` | yes | `GL.GLHistory.FinPeriodID` |
| `Branch ID` | no | `GL.GLHistory.BranchID` |
| `Ledger ID` | no | `GL.GLHistory.LedgerID` |
| `Account Class` | no | `GL.Account.AccountClassID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
Reading it: as pull 4. A window ending in the closing adjustment period (`13YYYY`) includes its
postings; say whether it was in.

## Pull 4c — Two windows per account: current, baseline and change

The flux table's spine in one call: per account, `Current`, `Baseline` and `Change` (current less
baseline), all in the natural sign.

```giql
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period To' REQUIRED
DECLARE [BaselinePeriodFrom] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Baseline Period From' REQUIRED
DECLARE [BaselinePeriodTo] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Baseline Period To' REQUIRED
DECLARE [BranchID] SCHEMA [PX.Objects.GL.GLHistory].BranchID AS 'Branch ID'
DECLARE [LedgerID] SCHEMA [PX.Objects.GL.GLHistory].LedgerID AS 'Ledger ID'
DECLARE [AccountClass] SCHEMA [PX.Objects.GL.Account].AccountClassID AS 'Account Class'
FROM [PX.Objects.GL.GLHistory] As Hist
INNER JOIN [PX.Objects.GL.Account] As Account ON Hist.AccountID = Account.AccountID
WHERE ((Hist.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo]) OR (Hist.FinPeriodID BETWEEN [BaselinePeriodFrom] AND [BaselinePeriodTo]))
AND Hist.BalanceType = 'A'
AND (Account.Type = 'I' OR Account.Type = 'E')
AND (Hist.BranchID = [BranchID] OR [BranchID] IS NULL)
AND (Hist.LedgerID = [LedgerID] OR [LedgerID] IS NULL)
AND (Account.AccountClassID = [AccountClass] OR [AccountClass] IS NULL)
GROUP BY Account.AccountID
ORDER BY Account.AccountCD
SELECT Account.AccountCD AS 'Account', Account.Description AS 'Description', Account.Type AS 'Type', Account.AccountClassID AS 'Account Class', SUM({=IIf([Hist.FinPeriodID]>=[PeriodFrom] AND [Hist.FinPeriodID]<=[PeriodTo],IIf([Account.Type]='I',[Hist.FinPtdCredit]-[Hist.FinPtdDebit],[Hist.FinPtdDebit]-[Hist.FinPtdCredit]),0)}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdCredit AS 'Current', SUM({=IIf([Hist.FinPeriodID]>=[BaselinePeriodFrom] AND [Hist.FinPeriodID]<=[BaselinePeriodTo],IIf([Account.Type]='I',[Hist.FinPtdCredit]-[Hist.FinPtdDebit],[Hist.FinPtdDebit]-[Hist.FinPtdCredit]),0)}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdCredit AS 'Baseline', SUM({=IIf([Hist.FinPeriodID]>=[PeriodFrom] AND [Hist.FinPeriodID]<=[PeriodTo],IIf([Account.Type]='I',[Hist.FinPtdCredit]-[Hist.FinPtdDebit],[Hist.FinPtdDebit]-[Hist.FinPtdCredit]),0)-IIf([Hist.FinPeriodID]>=[BaselinePeriodFrom] AND [Hist.FinPeriodID]<=[BaselinePeriodTo],IIf([Account.Type]='I',[Hist.FinPtdCredit]-[Hist.FinPtdDebit],[Hist.FinPtdDebit]-[Hist.FinPtdCredit]),0)}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdCredit AS 'Change'
```

| Key | Required | Binds to |
|---|---|---|
| `Period From` | yes | `GL.GLHistory.FinPeriodID` |
| `Period To` | yes | `GL.GLHistory.FinPeriodID` |
| `Baseline Period From` | yes | `GL.GLHistory.FinPeriodID` |
| `Baseline Period To` | yes | `GL.GLHistory.FinPeriodID` |
| `Branch ID` | no | `GL.GLHistory.BranchID` |
| `Ledger ID` | no | `GL.GLHistory.LedgerID` |
| `Account Class` | no | `GL.Account.AccountClassID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
Reading it:

- **Keep the windows disjoint.** An overlapping month is counted on both sides of `Change`.
- **One side zero is a finding** — a new or discontinued line, reported as such.
- **If an instance refuses the formula,** run pull 4b once per window and subtract; the figures are the
  same by construction.

## Pull 5 — GL actuals by account, subaccount, ledger and period

The core pull, and the one that serves two different jobs: run wide and coarse for the gap, then
narrow and deep — with `Account CD`, or at subaccount grain — for the investigation.

```giql
DECLARE [BranchID] SCHEMA [PX.Objects.GL.GLHistory].BranchID AS 'Branch ID' REQUIRED
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period To' REQUIRED
DECLARE [AccountCD] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account CD'
DECLARE [SelectedLedgerID] SCHEMA [PX.Objects.GL.GLHistory].LedgerID AS 'Ledger ID'
DECLARE [SubaccountCD] SCHEMA [PX.Objects.GL.GLHistory].SubID AS 'Subaccount CD'
FROM [PX.Objects.GL.GLHistory] AS History
INNER JOIN [PX.Objects.GL.Account] AS Account
  ON History.AccountID = Account.AccountID
INNER JOIN [PX.Objects.GL.Sub] AS Sub
  ON History.SubID = Sub.SubID
INNER JOIN [PX.Objects.GL.Ledger] AS Ledger
  ON History.LedgerID = Ledger.LedgerID
WHERE History.BranchID = [BranchID]
  AND History.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo]
  AND History.BalanceType = 'A'
  AND (History.AccountID = [AccountCD]
  OR [AccountCD] IS NULL)
  AND (History.LedgerID = [SelectedLedgerID]
  OR [SelectedLedgerID] IS NULL)
  AND (History.SubID = [SubaccountCD]
  OR [SubaccountCD] IS NULL)
GROUP BY
  History.FinPeriodID,
  Account.AccountCD,
  Sub.SubCD,
  Ledger.LedgerCD
ORDER BY
  Account.AccountCD,
  Sub.SubCD,
  Ledger.LedgerCD,
  History.FinPeriodID
SELECT
  History.FinPeriodID AS 'Financial Period',
  Account.AccountCD AS 'Account',
  Account.Description AS 'Description',
  Account.Type AS 'Account Type',
  Sub.SubCD AS 'Subaccount',
  Ledger.LedgerCD AS 'Ledger',
  SUM(History.FinBegBalance) AS 'Fin. Begining Balance',
  SUM(History.FinPtdDebit) AS 'Fin. PTD Debit',
  SUM(History.FinPtdCredit) AS 'Fin. PTD Credit',
  SUM(History.FinYtdBalance) AS 'Fin. YTD Balance',
  SUM({=IIf(InStr('IL',[Account.Type])>0,[History.FinPtdCredit]-[History.FinPtdDebit],[History.FinPtdDebit]-[History.FinPtdCredit])}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdDebit AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Branch ID` | yes | `GL.GLHistory.BranchID` |
| `Period From` | yes | `GL.GLHistory.FinPeriodID` |
| `Period To` | yes | `GL.GLHistory.FinPeriodID` |
| `Account CD` | no | `GL.GLHistory.AccountID` |
| `Ledger ID` | no | `GL.GLHistory.LedgerID` |
| `Subaccount CD` | no | `GL.GLHistory.SubID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
Reading it:

- **Period activity is `Natural Net`.** Sum it across the window; don't use `Fin. YTD Balance` for a
  window that crosses a year end.
- **Ignore `A` and `L` rows** — balance sheet accounts come back too and play no part in a P&L
  variance.
- **This is also the dimension breakdown.** The `Subaccount` column is the department, project or
  region axis; which segment carries which meaning is a per-environment convention, so confirm it with
  the user before attributing a gap to one. A gap concentrated in one value has an owner; a gap spread
  evenly across all of them is a rate, a policy or an accounting change and is not that owner's.
- **Supply `Ledger ID`** wherever more than one actual ledger exists.
- **An account that nets to zero here is not necessarily quiet.** Offsetting movements at subaccount
  grain net away at account grain. Where a material line shows an unexpectedly small variance, re-run
  at subaccount grain before concluding nothing happened.
- **Sizing is the constraint.** Accounts × subaccounts × periods grows fast. Keep the deep window
  short, raise `top` deliberately, page with `skip` (the query is ordered), or narrow with `Account CD`
  to one account at a time when only part of the P&L is under investigation.
- **No row means no activity** for that combination in that period — not a zero that was posted.
- The `Fin. Begining Balance` caption is spelled as the connector returns it; key off it as-is.

## Pull 6 — Budget by account and period

```giql
DECLARE [BranchID] SCHEMA [PX.Objects.GL.GLHistory].BranchID AS 'Branch ID' REQUIRED
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period To' REQUIRED
DECLARE [AccountCD] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account CD'
DECLARE [LedgerID] SCHEMA [PX.Objects.GL.GLHistory].LedgerID AS 'Ledger ID'
DECLARE [AccountClass] SCHEMA [PX.Objects.GL.Account].AccountClassID AS 'Account Class'
FROM [PX.Objects.GL.GLHistory] As History
INNER JOIN [PX.Objects.GL.Account] As Account ON History.AccountID = Account.AccountID
WHERE History.BranchID = [BranchID] AND History.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo] AND History.BalanceType = 'B' AND (Account.AccountCD = [AccountCD] OR [AccountCD] IS NULL) AND (History.LedgerID = [LedgerID] OR [LedgerID] IS NULL) AND (Account.AccountClassID = [AccountClass] OR [AccountClass] IS NULL)
GROUP BY Account.AccountCD, Account.Description, Account.Type, History.FinPeriodID
ORDER BY Account.AccountCD, History.FinPeriodID
SELECT Account.AccountCD AS 'Account', Account.Description AS 'Description', Account.Type AS 'Account Type', History.FinPeriodID AS 'Financial Period', SUM({=[History.FinPtdCredit]-[History.FinPtdDebit]}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdCredit AS 'Budgeted Amount', SUM({=IIf([Account.Type]='I',[History.FinPtdCredit]-[History.FinPtdDebit],IIf([Account.Type]='L',[History.FinPtdCredit]-[History.FinPtdDebit],[History.FinPtdDebit]-[History.FinPtdCredit]))}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdCredit AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Branch ID` | yes | `GL.GLHistory.BranchID` |
| `Period From` | yes | `GL.GLHistory.FinPeriodID` |
| `Period To` | yes | `GL.GLHistory.FinPeriodID` |
| `Account CD` | no | `GL.Account.AccountCD` |
| `Ledger ID` | no | `GL.GLHistory.LedgerID` |
| `Account Class` | no | `GL.Account.AccountClassID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
Reading it:

- **Budget and actual are the same record shape, separated by balance type** — `B` here, `A` in
  pull 5 — at identical branch, account and period grain. A budget variance is therefore a straight
  subtraction: pull 5's `Natural Net` less this pull's `Natural Net`. If the two sides look
  incompatible, a scope filter differs between the calls; it is not a data problem.
- **Supply `Ledger ID`** — the budget ledger, from the user, not pull 3's actual ledger. Where the
  tenant keeps more than one budget ledger (a budget and a forecast, or versions kept as separate
  ledgers), omitting it sums them invisibly. The scope's `budget:<name>` is this value.
- **Account grain here, not subaccount.** The same file's detail level adds the subaccount; the
  account drill-down uses it to compare a segment's actuals with its budget.
- **No rows means no budget is loaded** for that branch and window. Say so rather than treating zero
  as the plan.
- **Budget versions are not filtered here.** Where more than one exists for the scope, confirm which
  one with the user and name it in the output — a favourable variance against a revised-down budget
  and one against the original are different claims.
- **Lines on one side only are findings.** An account with budget and no actual is a programme that
  did not start; an account with actual and no budget is unbudgeted spend. Compare the full
  populations rather than only where the two meet.

## Pull 7 — Posted transaction lines

Where the investigation stops being inference. Run it per line under investigation, never across the
whole chart: always set `Ledger ID`, `Branch ID`, `Account CD` and a period or date range, and set
`top`.

```giql
DECLARE [Ledger] SCHEMA [PX.Objects.GL.Ledger].LedgerCD AS 'Ledger ID'
DECLARE [Branch] SCHEMA [PX.Objects.GL.GLTran].BranchID AS 'Branch ID'
DECLARE [Account] SCHEMA [PX.Objects.GL.GLTran].AccountID AS 'Account CD'
DECLARE [BusinessAccount] SCHEMA [PX.Objects.CR.BAccount].AcctCD AS 'Business Account'
DECLARE [Subaccount] SCHEMA [PX.Objects.GL.GLTran].SubID AS 'Subaccount CD'
DECLARE [SubaccountPattern] SCHEMA [PX.Objects.GL.Sub].SubCD AS 'Subaccount Pattern'
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.GLTran].FinPeriodID AS 'Period From'
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.GLTran].FinPeriodID AS 'Period To'
DECLARE [DateFrom] SCHEMA [PX.Objects.GL.GLTran].TranDate AS 'Date From'
DECLARE [DateTo] SCHEMA [PX.Objects.GL.GLTran].TranDate AS 'Date To'
DECLARE [BatchNbr] SCHEMA [PX.Objects.GL.GLTran].BatchNbr AS 'Batch Number'
DECLARE [AccountClass] SCHEMA [PX.Objects.GL.Account].AccountClassID AS 'Account Class'
FROM [PX.Objects.GL.GLTran] As Tran
INNER JOIN [PX.Objects.GL.Ledger] As Ledger ON Tran.LedgerID = Ledger.LedgerID AND Ledger.BalanceType = {='A'}
LEFT JOIN [PX.Objects.CR.BAccount] As BAccount ON Tran.ReferenceID = BAccount.BAccountID
INNER JOIN [PX.Objects.GL.Sub] As Sub ON Tran.SubID = Sub.SubID
INNER JOIN [PX.Objects.GL.Account] As Account ON Tran.AccountID = Account.AccountID
WHERE Tran.Posted = 1
AND (Ledger.LedgerCD = [Ledger] OR [Ledger] IS NULL)
AND (Tran.BranchID = [Branch] OR [Branch] IS NULL)
AND (Tran.AccountID = [Account] OR [Account] IS NULL)
AND (BAccount.AcctCD = [BusinessAccount] OR [BusinessAccount] IS NULL)
AND (Tran.SubID = [Subaccount] OR [Subaccount] IS NULL)
AND (Sub.SubCD LIKE [SubaccountPattern] OR [SubaccountPattern] IS NULL)
AND (Tran.FinPeriodID >= [PeriodFrom] OR [PeriodFrom] IS NULL)
AND (Tran.FinPeriodID <= [PeriodTo] OR [PeriodTo] IS NULL)
AND (Tran.TranDate >= [DateFrom] OR [DateFrom] IS NULL)
AND (Tran.TranDate <= [DateTo] OR [DateTo] IS NULL)
AND (Tran.BatchNbr = [BatchNbr] OR [BatchNbr] IS NULL)
AND (Account.AccountClassID = [AccountClass] OR [AccountClass] IS NULL)
ORDER BY Tran.FinPeriodID, Tran.TranDate, Tran.BatchNbr, Tran.LineNbr
SELECT Tran.FinPeriodID AS 'Period ID', Tran.TranDate AS 'Transaction Date', Tran.BranchID AS 'Branch', Tran.AccountID AS 'Account', Tran.SubID AS 'Subaccount', BAccount.AcctCD AS 'Business Account', BAccount.AcctName AS 'Account Name', BAccount.Type AS 'Type', Tran.Module AS 'Module', Tran.BatchNbr AS 'Batch Number', Tran.LineNbr AS 'Line Nbr.', Tran.RefNbr AS 'Ref. Number', Tran.TranDesc AS 'Transaction Description', Tran.DebitAmt AS 'Debit Amount', Tran.CreditAmt AS 'Credit Amount', {=[Tran.DebitAmt]-[Tran.CreditAmt]} SCHEMA [PX.Objects.GL.GLTran].DebitAmt AS 'Net Amount', {=IIf([Account.Type]='I',[Tran.CreditAmt]-[Tran.DebitAmt],IIf([Account.Type]='L',[Tran.CreditAmt]-[Tran.DebitAmt],[Tran.DebitAmt]-[Tran.CreditAmt]))} SCHEMA [PX.Objects.GL.GLTran].DebitAmt AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Ledger ID` | no | `GL.Ledger.LedgerCD` |
| `Branch ID` | no | `GL.GLTran.BranchID` |
| `Account CD` | no | `GL.GLTran.AccountID` |
| `Business Account` | no | `CR.BAccount.AcctCD` |
| `Subaccount CD` | no | `GL.GLTran.SubID` |
| `Subaccount Pattern` | no | `GL.Sub.SubCD` |
| `Period From` | no | `GL.GLTran.FinPeriodID` |
| `Period To` | no | `GL.GLTran.FinPeriodID` |
| `Date From` | no | `GL.GLTran.TranDate` |
| `Date To` | no | `GL.GLTran.TranDate` |
| `Batch Number` | no | `GL.GLTran.BatchNbr` |
| `Account Class` | no | `GL.Account.AccountClassID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
Reading it:

- **Filter by financial period to tie to the balances, by date to read what happened.** Filtered by the
  same ledger, branch, account and period range, its `Natural Net` sums to pull 5's for that account.
  A date range is transaction-date based and will not agree wherever a document's date and its posting
  period differ; say which basis each figure came from.
- **Ordered by date, not by amount.** Rank client-side to find the concentration.
- **What to look for**, roughly in order of how often it turns out to be the answer: a handful of
  documents accounting for most of the gap; dates sitting right at a period boundary, which is the
  signature of a timing variance; a recurring charge that is *absent* this period, which is as often
  the cause of a favourable variance as efficiency is; the same counterparty and amount on near dates,
  which is a duplicate; one document covering several periods, which is a catch-up and must not be
  read as a run-rate change; and a document that belongs on another line, in which case look for the
  offsetting under-run elsewhere before concluding anything about this one.
- **`Posted = 1` only.** Anything not yet posted is in pull 8, not here.
- **Actual ledgers only, and always name one.** It joins to ledgers of type Actual, so reporting and
  statistical lines never come back; left without `Ledger ID` it returns every actual ledger's lines
  together. (The query this replaced had no ledger filter: 30 lines against 24 on `gab-dev`, the extra
  six on a reporting ledger.)
- **`Module`** says which subledger the line came from, which is often enough to name the cause
  without opening the document.
- **The description is the strongest free evidence in the whole method** — and it is what a person
  typed, so it is a claim, not a fact. Record it as what the document says.

## Pull 8 — Unposted GL batches in a period

Only needed where the period is open — and then it is not optional, because unposted work is invisible
to every balance comparison above. A favourable expense variance in an open period is usually just
work that has not landed.

**Summary level** — one row per unposted batch:

```giql
DECLARE BranchID SCHEMA [PX.Objects.GL.Batch].BranchID AS 'Branch ID' REQUIRED
DECLARE FinPeriod SCHEMA [PX.Objects.GL.Batch].FinPeriodID AS 'Fin Period' REQUIRED
FROM [PX.Objects.GL.Batch] AS Batch
WHERE Batch.BranchID = BranchID
  AND Batch.FinPeriodID = FinPeriod
  AND Batch.Posted = 0
ORDER BY
  Batch.Module,
  Batch.BatchNbr
SELECT
  Batch.Module AS 'Module',
  Batch.BatchNbr AS 'Batch Number',
  Batch.DateEntered AS 'Transaction Date',
  Batch.Status AS 'Batch Status',
  Batch.DebitTotal AS 'Debit Total',
  Batch.CreditTotal AS 'Credit Total',
  {=[Batch.DebitTotal]-[Batch.CreditTotal]} SCHEMA [PX.Objects.GL.Batch].DebitTotal AS 'Debit-Credit Difference',
  Batch.Description AS 'Description'
```

| Key | Required | Binds to |
|---|---|---|
| `Branch ID` | yes | `GL.Batch.BranchID` |
| `Fin Period` | yes | `GL.Batch.FinPeriodID` |
**Detail level** — the lines inside those batches, so you can see which accounts a large unposted
batch would move. Supply `Batch Number` to scope it to one batch, or pass it as `null` for every
unposted batch in the period — left out of the map, it returns no rows:

```giql
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Batch].BranchID AS 'Branch ID' REQUIRED
DECLARE [FinPeriod] SCHEMA [PX.Objects.GL.Batch].FinPeriodID AS 'Fin Period' REQUIRED
DECLARE [BatchNbr] SCHEMA [PX.Objects.GL.GLTran].BatchNbr AS 'Batch Number'
FROM [PX.Objects.GL.Batch] AS Batch
INNER JOIN [PX.Objects.GL.GLTran] AS Tran
  ON Batch.Module = Tran.Module
  AND Batch.BatchNbr = Tran.BatchNbr
INNER JOIN [PX.Objects.GL.Account] AS Account
  ON Tran.AccountID = Account.AccountID
INNER JOIN [PX.Objects.GL.Sub] AS Sub
  ON Tran.SubID = Sub.SubID
WHERE Batch.BranchID = [BranchID]
  AND Batch.FinPeriodID = [FinPeriod]
  AND Batch.Posted = 0
  AND (Batch.BatchNbr = [BatchNbr]
  OR [BatchNbr] IS NULL)
ORDER BY Batch.Module, Batch.BatchNbr, Tran.LineNbr
SELECT Batch.Module AS 'Module', Batch.BatchNbr AS 'Batch Number', Batch.Status AS 'Batch Status', Tran.LineNbr AS 'Line Number', Account.AccountCD AS 'Account', Account.Description AS 'Account Description', Sub.SubCD AS 'Subaccount', Tran.TranDesc AS 'Line Description', Tran.RefNbr AS 'Ref. Number', Tran.DebitAmt AS 'Debit Amount', Tran.CreditAmt AS 'Credit Amount'
```

| Key | Required | Binds to |
|---|---|---|
| `Branch ID` | yes | `GL.Batch.BranchID` |
| `Fin Period` | yes | `GL.Batch.FinPeriodID` |
| `Batch Number` | no | `GL.GLTran.BatchNbr` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
Reading it:

- **Summary is the default.** Reach for detail when a batch's difference isn't zero, or to judge which
  accounts a large unposted batch would move.
- **`Batch Status` is a code**: `H` On Hold, `B` Balanced, `U` Unposted, `P` Posted, `C` Completed,
  `V` Voided, `R` Released, `Q` Partially Released, `D` Pending Approval, `J` Rejected, `S` Scheduled.
  Translate it before showing it to anyone. `B` and `D` are both genuinely unposted — a batch can be
  arithmetically fine and still not be in the ledger.
- **A non-zero `Debit-Credit Difference` means the batch cannot post at all**, whatever its size. Treat
  it as blocking rather than scoring it against a materiality threshold.
- **A zero difference does not mean a batch is substantive.** An empty batch scores 0.00 on the same
  test while containing nothing that could ever land. Report zero-total batches separately from real
  ones.
- **These amounts are not in any balance query.** Do not add them to pull 5 to "correct" it — report
  them as what could still land, and label the period's variances preliminary.
- **The two levels cannot be reconciled by row count.** A batch with no lines disappears from the
  detail level entirely, because of the inner join.
- **`Transaction Date` is the date entered and need not fall inside the financial period.** Filter on
  the period, report the date, and don't infer one from the other.
- **The branch filter is the batch's, not the line's**, so the detail level's totals are not a
  per-branch figure the way the summary level's are.

**Over a range** — pull 8b, the summary level for every period from `Period From` to `Period To`, with
the period in the first column. Use it for a quarter or a year instead of one call per period, then run
the detail level for the batches that matter, with their own period and `Batch Number`:

```giql
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Batch].BranchID AS 'Branch ID' REQUIRED
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.Batch].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.Batch].FinPeriodID AS 'Period To' REQUIRED
FROM [PX.Objects.GL.Batch] As Batch
WHERE Batch.BranchID = [BranchID]
  AND Batch.FinPeriodID >= [PeriodFrom]
  AND Batch.FinPeriodID <= [PeriodTo]
  AND Batch.Posted = 0
ORDER BY Batch.FinPeriodID, Batch.Module, Batch.BatchNbr
SELECT Batch.FinPeriodID AS 'Financial Period', Batch.Module AS 'Module', Batch.BatchNbr AS 'Batch Number', Batch.DateEntered AS 'Transaction Date', Batch.Status AS 'Batch Status', Batch.DebitTotal AS 'Debit Total', Batch.CreditTotal AS 'Credit Total', {=[Batch.DebitTotal]-[Batch.CreditTotal]} SCHEMA [PX.Objects.GL.Batch].DebitTotal AS 'Debit-Credit Difference', Batch.Description AS 'Description'
```

| Key | Required | Binds to |
|---|---|---|
| `Branch ID` | yes | `GL.Batch.BranchID` |
| `Period From` | yes | `GL.Batch.FinPeriodID` |
| `Period To` | yes | `GL.Batch.FinPeriodID` |
## Pull 9 — Invoiced revenue by customer and period

Only where revenue mix, concentration or customer effects are in scope.

```giql
DECLARE [BranchID] SCHEMA [PX.Objects.AR.ARInvoice].BranchID AS 'Branch ID' REQUIRED
DECLARE [PeriodFrom] SCHEMA [PX.Objects.AR.ARInvoice].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.AR.ARInvoice].FinPeriodID AS 'Period To' REQUIRED
FROM [PX.Objects.AR.ARInvoice] As Invoice
INNER JOIN [PX.Objects.AR.Customer] As Customer ON Invoice.CustomerID = Customer.BAccountID
WHERE Invoice.BranchID = [BranchID] AND Invoice.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo] AND Invoice.Released = 1 AND (Invoice.DocType = 'INV' OR Invoice.DocType = 'DRM' OR Invoice.DocType = 'CRM' OR Invoice.DocType = 'CSL' OR Invoice.DocType = 'RCS')
GROUP BY Customer.AcctCD, Customer.AcctName, Invoice.FinPeriodID
ORDER BY Customer.AcctCD, Invoice.FinPeriodID
SELECT Customer.AcctCD AS 'Customer ID', Customer.AcctName AS 'Customer Name', Invoice.FinPeriodID AS 'Post Period', SUM({=IIf([Invoice.DocType]='CRM' OR [Invoice.DocType]='RCS',-1,1)}) SCHEMA [PX.Objects.AR.ARInvoice].LineCntr AS 'Net Document Count', SUM({=IIf([Invoice.DocType]='CRM' OR [Invoice.DocType]='RCS',-[Invoice.LineTotal],[Invoice.LineTotal])}) SCHEMA [PX.Objects.AR.ARInvoice].LineTotal AS 'Total Sales'
```

| Key | Required | Binds to |
|---|---|---|
| `Branch ID` | yes | `AR.ARInvoice.BranchID` |
| `Period From` | yes | `AR.ARInvoice.FinPeriodID` |
| `Period To` | yes | `AR.ARInvoice.FinPeriodID` |
Reading it:

- **This is invoiced AR, not the GL revenue line.** Journal entries, deferrals and non-AR income sit
  outside it, so it will not tie to pull 5. Use it for mix, concentration and customer effects, and
  keep pull 5 as the control total. Show the difference as unattributed revenue rather than hiding it.
- **Report customer effects as four figures, not one net** — new, lost, expanded and contracted. A net
  revenue increase combining a large loss and a larger gain is two findings, and netting them is how
  the loss goes unreported.
- **Credit memos subtract from both amount and count.** A customer with a net count of zero took a
  credit that cancelled their invoicing that period; investigate rather than drop.
- **Amounts are in each document's own currency.** Confirm the population is single-currency before
  totalling.
- **No sort** — order by amount client-side.
- **Sizing is the real constraint here**, more than on any other pull. Shorten the window or raise
  `top` deliberately, and check the `truncated` flag before reading concentration off the result.
- **Customer names are sensitive.** Name a customer in a deliverable only if the user approves it.

## Reconciliations

Run for every branch and window used. A failure is an open item and holds any figure that depends on
it — it is not something to note and work around.

| Check | Pass condition | Most common cause of failure |
|---|---|---|
| Shape ↔ detail | Pull 4's `Natural Net` per account and period = pull 5's summed over subaccounts, same branch, ledger and window | A ledger supplied to one call and not the other |
| Window ↔ shape | Pull 4b's `Natural Net` per account = pull 4's summed over the same window, branch, ledger and class | A window bound that includes or excludes the adjustment period on one call only |
| Comparison ↔ windows | Pull 4c's `Current` and `Baseline` per account = pull 4b for each window | Overlapping windows; a class filter on one call only |
| Unposted range ↔ period | Pull 8b's rows for a period = pull 8 run for that period | A branch other than the one the variances are reported for |
| Query outside the library | A `prepare_query` build used because no pull had the grain ties to pull 4, 4b or 5 for the same scope, and the basis block names it | A grain or scope filter that differs from the control |
| Account ↔ subaccount | Pull 5 at account grain = pull 5 at subaccount grain, account by account | A subaccount filter left in place on one call |
| Budget ↔ actual population | Every account in pull 6 appears in pull 5 and vice versa, or the one-sided lines are reported | Comparing only where the two meet |
| Detail ↔ balance | Pull 7's `Natural Net` for an account, branch and period range equals pull 5's | A ledger, subaccount or account-class filter differs between the calls; with a date range instead, the transaction-date versus period basis, which is reported rather than reconciled away |
| Components ↔ gap | Each decomposition's components sum to its gap, or the remainder is named | A component produced from an input that does not exist |
| Revenue mix ↔ ledger | Pull 9's total vs the revenue accounts in pull 5, with the difference shown as unattributed | Journal entries, deferrals, non-AR income |
| Open period | Pull 8 returns nothing, or what it returns is reported alongside the variances | Reporting a favourable expense variance in an open period as performance |

Record each result in the basis block with both figures and the difference.

## Where these come from

Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-11 from
1d635db. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
