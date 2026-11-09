# Data pulls

Run these through the Velixo connector's `execute_query` as written, with your
values in the `parameters` map. The query text goes in the `giql` argument whatever its fence says: a
` ```vql ` block goes there too, since `execute_query` has no `vql` argument. Do not call `prepare_query` for them and do not edit the text. If one
fails, report the error and stop rather than reformulating it. Parameter keys are the display labels
(the text after `AS` in each `DECLARE`); a key that matches no label is dropped silently and the
query returns nothing, which looks exactly like "no data". Pass every declared key, an unused optional
one as `null`: a key left out is not always treated as null. Always pass `top` — start at 250 and
raise it if the result comes back flagged `truncated`.
**Period values are `MMYYYY`** (`062025`), exactly as the user wrote them: never rewrite one into the
year-first `202506` that Acumatica stores internally. The connector translates, and a year-first
value matches no period and returns no rows.

| Alias | Query | Grain | Used for |
|---|---|---|---|
| K1 | `gl-branch-company-ledger-list` | one row per branch | branch codes, each branch's company and ledger |
| K2 | `financial-period-calendar-range` | one row per period | the as-of date, days in each period, which period is open |
| I1 | `in-item-cost-balance-by-period`, summary level | period × branch × account × subaccount | the total, the trend, the GL tie, DIO |
| I2 | `in-item-cost-balance-by-period`, detail level | period × item × branch × warehouse × account × subaccount | the valuation listing |
| I3 | `in-inventory-aging` | one row per item at a period end | aging, slow-moving stock, clean-up |
| G1 | `gl-account-balances-by-period-range`, detail level | period × account × subaccount | the GL side of the tie |
| C0 | `gl-chart-of-accounts-classification` | one row per account | proposing the cost of sales accounts |
| C1 | `gl-pl-actuals-by-period-and-account` | period × account | cost of sales for turnover and DIO |

## Which pulls for which use

- **Valuation:** I1 for the period, I2 for the period (one branch at a time on a large tenant), G1.
- **Total by period:** I1 over the window in one call, G1 over the same window.
- **Aging:** I3 for the period, I1 for the same period to foot.
- **Turnover and DIO:** I1 over the window plus the period before it (for the average), C0 once to
  propose accounts, C1 over the window, K2 for the days.
- **Reconciliation:** I1 and G1 over the window.
- **Clean-up:** I3 and I2 for the period.

## K1 — branches, companies and ledgers

```giql
FROM [PX.Objects.GL.Branch] As Branch
INNER JOIN [PX.Objects.GL.DAC.Organization] As Organization ON Branch.OrganizationID = Organization.OrganizationID
INNER JOIN [PX.Objects.GL.Ledger] As Ledger ON Branch.LedgerID = Ledger.LedgerID
ORDER BY Organization.OrganizationCD, Branch.BranchCD
SELECT Branch.BranchCD AS 'Branch ID', Branch.AcctName AS 'Branch Name', Organization.OrganizationCD AS 'Company ID', Organization.OrganizationName AS 'Company Name', Ledger.LedgerCD AS 'Ledger ID', Branch.BaseCuryID AS 'Base Currency ID', Branch.Active AS 'Active'
```

No parameters.
## K2 — financial period calendar

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
`End Date` is the next period's first day: the as-of date is `End Date` minus one day. `Status`
`Open` marks a period still open; label the output preliminary.

## I1 — valuation, summary level

```giql
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.FinPeriods.TableDefinition.FinPeriod].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.FinPeriods.TableDefinition.FinPeriod].FinPeriodID AS 'Period To' REQUIRED
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Branch].BranchCD AS 'Branch ID'
FROM FinPeriod
INNER JOIN INItemCostHist
  ON FinPeriod.FinPeriodID >= INItemCostHist.FinPeriodID
LEFT JOIN [PX.Objects.IN.INItemCostHist] AS INItemCostHistPrev
  ON INItemCostHist.InventoryID = INItemCostHistPrev.InventoryID
  AND INItemCostHist.CostSubItemID = INItemCostHistPrev.CostSubItemID
  AND INItemCostHist.CostSiteID = INItemCostHistPrev.CostSiteID
  AND INItemCostHist.AccountID = INItemCostHistPrev.AccountID
  AND INItemCostHist.SubID = INItemCostHistPrev.SubID
  AND {=[FinPeriod.FinPeriodID]} >= INItemCostHistPrev.FinPeriodID
  AND INItemCostHist.FinPeriodID < INItemCostHistPrev.FinPeriodID
INNER JOIN INSite
  ON INItemCostHist.CostSiteID = INSite.SiteID
INNER JOIN [PX.Objects.GL.Branch] AS Branch
  ON INSite.BranchID = Branch.BranchID
INNER JOIN [PX.Objects.GL.Account] AS Account
  ON INItemCostHist.AccountID = Account.AccountID
INNER JOIN [PX.Objects.GL.Sub] AS Sub
  ON INItemCostHist.SubID = Sub.SubID
INNER JOIN InventoryItem
  ON INItemCostHist.InventoryID = InventoryItem.InventoryID
WHERE FinPeriod.OrganizationID = {=0}
  AND INItemCostHistPrev.FinPeriodID IS NULL
  AND INItemCostHist.FinYtdCost != {=0}
  AND (FinPeriod.FinPeriodID >= [PeriodFrom])
  AND (FinPeriod.FinPeriodID <= [PeriodTo])
  AND ((Branch.BranchCD = [BranchID] OR [BranchID] IS NULL))
GROUP BY FinPeriod.FinPeriodID, Branch.BranchID, Account.AccountID, Sub.SubID
SELECT FinPeriod.FinPeriodID AS 'FinPeriodID', Branch.BranchCD AS 'BranchID', Account.AccountCD AS 'AccountID', Sub.SubCD AS 'SubID', SUM(INItemCostHist.FinYtdCost) AS 'Balance', SUM(INItemCostHist.FinYtdQty) AS 'Qty'
```

| Key | Required | Binds to |
|---|---|---|
| `Period From` | yes | `GL.FinPeriods.TableDefinition.FinPeriod.FinPeriodID` |
| `Period To` | yes | `GL.FinPeriods.TableDefinition.FinPeriod.FinPeriodID` |
| `Branch ID` | no | `GL.Branch.BranchCD` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
- `Balance` is the inventory value at the period end, debit-positive like the GL asset account;
  `Qty` the quantity on hand.
- An item that did not move in a period still counts: its last cost history row carries forward.
- The branch is the warehouse's branch. Leave adjustment periods (`13YYYY`) out of a trend.

## I2 — valuation, detail level

```giql
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.FinPeriods.TableDefinition.FinPeriod].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.FinPeriods.TableDefinition.FinPeriod].FinPeriodID AS 'Period To' REQUIRED
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Branch].BranchCD AS 'Branch ID'
FROM FinPeriod
INNER JOIN INItemCostHist
  ON FinPeriod.FinPeriodID >= INItemCostHist.FinPeriodID
LEFT JOIN [PX.Objects.IN.INItemCostHist] AS INItemCostHistPrev
  ON INItemCostHist.InventoryID = INItemCostHistPrev.InventoryID
  AND INItemCostHist.CostSubItemID = INItemCostHistPrev.CostSubItemID
  AND INItemCostHist.CostSiteID = INItemCostHistPrev.CostSiteID
  AND INItemCostHist.AccountID = INItemCostHistPrev.AccountID
  AND INItemCostHist.SubID = INItemCostHistPrev.SubID
  AND {=[FinPeriod.FinPeriodID]} >= INItemCostHistPrev.FinPeriodID
  AND INItemCostHist.FinPeriodID < INItemCostHistPrev.FinPeriodID
INNER JOIN INSite
  ON INItemCostHist.CostSiteID = INSite.SiteID
INNER JOIN [PX.Objects.GL.Branch] AS Branch
  ON INSite.BranchID = Branch.BranchID
INNER JOIN [PX.Objects.GL.Account] AS Account
  ON INItemCostHist.AccountID = Account.AccountID
INNER JOIN [PX.Objects.GL.Sub] AS Sub
  ON INItemCostHist.SubID = Sub.SubID
INNER JOIN InventoryItem
  ON INItemCostHist.InventoryID = InventoryItem.InventoryID
WHERE FinPeriod.OrganizationID = {=0}
  AND INItemCostHistPrev.FinPeriodID IS NULL
  AND INItemCostHist.FinYtdCost != {=0}
  AND (FinPeriod.FinPeriodID >= [PeriodFrom])
  AND (FinPeriod.FinPeriodID <= [PeriodTo])
  AND ((Branch.BranchCD = [BranchID] OR [BranchID] IS NULL))
GROUP BY
  FinPeriod.FinPeriodID,
  INItemCostHist.InventoryID,
  Branch.BranchID,
  INItemCostHist.CostSiteID,
  Account.AccountID,
  Sub.SubID
SELECT FinPeriod.FinPeriodID AS 'FinPeriodID', INItemCostHist.InventoryID AS 'InventoryID', InventoryItem.Descr AS 'Description', Branch.BranchCD AS 'BranchID', INSite.SiteCD AS 'SiteID', Account.AccountCD AS 'AccountID', Sub.SubCD AS 'SubID', SUM(INItemCostHist.FinYtdCost) AS 'Balance', SUM(INItemCostHist.FinYtdQty) AS 'Qty'
```

| Key | Required | Binds to |
|---|---|---|
| `Period From` | yes | `GL.FinPeriods.TableDefinition.FinPeriod.FinPeriodID` |
| `Period To` | yes | `GL.FinPeriods.TableDefinition.FinPeriod.FinPeriodID` |
| `Branch ID` | no | `GL.Branch.BranchCD` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
`Period From` = `Period To` = the as-of period. Its rows add to I1's for the same branch, account and
subaccount. Trim `InventoryID`, `SiteID` and the codes.

## I3 — inventory aging

```giql
DECLARE [FinPeriod] SCHEMA [PX.Objects.GL.FinPeriods.TableDefinition.FinPeriod].FinPeriodID AS 'Fin Period' REQUIRED
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Branch].BranchCD AS 'Branch ID'
FROM FinPeriod
INNER JOIN INItemCostHist
  ON FinPeriod.FinPeriodID >= INItemCostHist.FinPeriodID
LEFT JOIN [PX.Objects.IN.INItemCostHist] AS INItemCostHistPrev
  ON INItemCostHist.InventoryID = INItemCostHistPrev.InventoryID
  AND INItemCostHist.CostSubItemID = INItemCostHistPrev.CostSubItemID
  AND INItemCostHist.CostSiteID = INItemCostHistPrev.CostSiteID
  AND INItemCostHist.AccountID = INItemCostHistPrev.AccountID
  AND INItemCostHist.SubID = INItemCostHistPrev.SubID
  AND {=[FinPeriod.FinPeriodID]} >= INItemCostHistPrev.FinPeriodID
  AND INItemCostHist.FinPeriodID < INItemCostHistPrev.FinPeriodID
INNER JOIN InventoryItem
  ON INItemCostHist.InventoryID = InventoryItem.InventoryID
LEFT JOIN [PX.Objects.IN.INSite] As Site ON INItemCostHist.CostSiteID = Site.SiteID
LEFT JOIN [PX.Objects.GL.Branch] As Branch ON Site.BranchID = Branch.BranchID
WHERE FinPeriod.OrganizationID = {=0}
  AND INItemCostHistPrev.FinPeriodID IS NULL
  AND INItemCostHist.FinYtdQty != {=0}
  AND (FinPeriod.FinPeriodID = [FinPeriod])
  AND ((Branch.BranchCD = [BranchID] OR [BranchID] IS NULL))
GROUP BY
  FinPeriod.FinPeriodID,
  INItemCostHist.InventoryID
SELECT FinPeriod.FinPeriodID AS 'FinPeriodID', InventoryItem.InventoryCD AS 'InventoryID', InventoryItem.ItemStatus AS 'Status', InventoryItem.Descr AS 'Description', SUM(INItemCostHist.FinYtdCost) AS 'Cost', SUM(INItemCostHist.FinYtdQty) AS 'Qty', MAX(INItemCostHist.FinPeriodID) AS 'Last Activity Period'
```

| Key | Required | Binds to |
|---|---|---|
| `Fin Period` | yes | `GL.FinPeriods.TableDefinition.FinPeriod.FinPeriodID` |
| `Branch ID` | no | `GL.Branch.BranchCD` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
- One row per item; `Last Activity Period` is the last period the item's cost history moved, on or
  before `Fin Period`, across its warehouses.
- Rows with zero quantity are left out, where I1 leaves out zero cost: the two totals can differ by
  items with cost and no quantity. List those for the clean-up.
- `Status` is today's item status, not the period's.

## G1 — GL inventory account

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
`Branch ID` from I1, `Account CD` each inventory account I1 returns, `Ledger ID` the branch's
ledger from K1.

**Tie to the GL one subaccount at a time.** GL history has a row only for a period in which the
account and subaccount had activity, and that row's `Fin. YTD Balance` is the balance of that
subaccount alone. So the account's balance at a period end is the sum, over its subaccounts, of each
subaccount's **latest** row on or before that period. Summing only the rows of the period itself
leaves out every subaccount that was quiet in it, and the account looks short by their balances.

1. Run the GL balances query's **detail level** once per branch and control account: `Account CD` the
   control account, `Ledger ID` the branch's ledger, `Subaccount CD` null, `Period From` and
   `Period To` the window (or the as-of period alone).
2. For each period and each subaccount with a row, the GL balance is that row's `Fin. YTD Balance`.
3. For each subaccount the subledger carries in a period but step 1 returned no row for, run the same
   level again with that `Subaccount CD`, `Period From` `012000` and `Period To` the period, with
   `top` 1. The result's `totalCount` says how many rows there are; run it once more with `skip`
   `totalCount` − 1 to read the last one. Its `Fin. YTD Balance` is the subaccount's balance carried
   into the period. No row at all means the GL holds nothing on that subaccount.
4. Compare subaccount by subaccount, then in total, when the other side of the tie carries
   subaccounts; when it does not, compare the total of step 2 and 3's balances with it. A difference
   on one subaccount names where to look; a total alone does not.

A subaccount with a GL balance that the subledger never uses and that had no activity in the window
is invisible to this method. When the totals differ and the subaccount comparison does not explain
it, say so and offer the account's whole history at the detail level (`Period From` `012000`) as
the next pull.
## C0 — chart of accounts classification

```giql
FROM [PX.Objects.GL.Account] As Account
LEFT JOIN [PX.Objects.GL.AccountClass] As AccountClass ON Account.AccountClassID = AccountClass.AccountClassID
ORDER BY Account.Type, Account.AccountCD
SELECT Account.AccountCD AS 'Account', Account.Description AS 'Description', Account.Type AS 'Type', Account.AccountClassID AS 'Account Class', AccountClass.Descr AS 'Account Class Description', Account.Active AS 'Active', Account.ControlAccountModule AS 'Control Account Module', Account.IsCashAccount AS 'Cash Account', Account.CuryID AS 'Currency'
```

No parameters.
Propose the expense accounts whose class or description names cost of sales or cost of goods sold,
and ask the user to confirm the list.

## C1 — cost of sales

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
Keep the confirmed accounts; `Natural Net` is the period's cost of sales (debit less credit).

## When a footing or a tie fails

The detail does not add to the summary, or the aging to the valuation, for the same branch and
period. In order:

1. **Something was posted between the pulls.** Rerun both back to back.
2. **Different scope:** a branch or period mismatch.
3. **The filters differ:** items with cost and no quantity (in I1, not I3), or quantity and no cost
   (in I3, not I1).

The subledger differs from the GL on a subaccount: check the carried-forward GL row was read (step 3
of the tie), then report both figures and the difference, and stop there.

Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-09 from
5ed670d. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
