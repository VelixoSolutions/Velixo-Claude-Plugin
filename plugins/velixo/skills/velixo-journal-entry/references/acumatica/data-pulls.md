# Data pulls

Every check before and after the write reads the tenant through these library queries. Run each
through `execute_query` exactly as written, with your values in the `parameters` map keyed by the
labels in the table under it. Don't call `prepare_query` for them and don't edit the text; if one
fails, report the error and stop. A key that matches no label is dropped silently and the query
returns nothing. **Pass every key a query declares**, an unused optional one as `null`. **Period values
are `MMYYYY`** (`062025`), never Acumatica's internal year-first form (`202506`), which matches no
period. Always set `top` (250 is enough for all of these but the chart of accounts and the segments,
which take the whole list: 2,000) and check `truncated`.

| Alias | Query | Used for | When |
|---|---|---|---|
| J1 | `gl-branch-company-ledger-list` | branch exists and is active; its company, posting ledger and base currency | every entry |
| J2 | `financial-period-calendar-range` | the period for the date, and its status | every entry |
| J3 | `gl-chart-of-accounts-classification` | every account: exists, active, type, control module, cash flag, currency; names for the preview | every entry |
| J4 | `subaccount-segments` | every subaccount splits into valid, active segment values; descriptions for the preview | every entry on a tenant with subaccounts |
| J5 | `gl-unposted-batches-by-period`, summary level | duplicate check before; the batch's status and totals after | every entry |
| J6 | `gl-unposted-batches-by-period`, detail level | the batch's lines after the write; a last batch number given by the user | after every write |
| J7 | `gl-batch-by-number` | one batch by its number alone, any status: header, flags and every line | release or reverse by number; read-back after a release |

J1, J3 and J4 are tenant lists: run each once per session and reuse them for every entry.

## J1 — Branches, companies, ledgers

```giql
FROM [PX.Objects.GL.Branch] As Branch
INNER JOIN [PX.Objects.GL.DAC.Organization] As Organization ON Branch.OrganizationID = Organization.OrganizationID
INNER JOIN [PX.Objects.GL.Ledger] As Ledger ON Branch.LedgerID = Ledger.LedgerID
ORDER BY Organization.OrganizationCD, Branch.BranchCD
SELECT Branch.BranchCD AS 'Branch ID', Branch.AcctName AS 'Branch Name', Organization.OrganizationCD AS 'Company ID', Organization.OrganizationName AS 'Company Name', Ledger.LedgerCD AS 'Ledger ID', Branch.BaseCuryID AS 'Base Currency ID', Branch.Active AS 'Active'
```

No parameters.
- `Ledger ID` is the branch's posting (actual) ledger, the header's default. `Company ID` is the key
  J2 needs. `Base Currency ID` is the header currency's default.
- Codes come back padded with trailing spaces (`PRODWHOLE `); trim before comparing or passing.
- `Active` false: the branch takes no new entries; say so and ask.

## J2 — The period for the date

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
- `Company ID` is the branch's company from J1. When the user gave the period, pass it as both
  `Period From` and `Period To`. When only the date is known, use `01` of the date's year to `13` of
  the next year (`012025` – `132026` for a date in 2025): that covers a fiscal year that does not run
  January to December. Take the non-adjustment row whose `Start Date` ≤ the date < `End Date`.
- **`End Date` is the next period's start**: `062025` ends `2025-07-01`, so 30 June is in `062025`
  and 1 July is not.
- Only `Open` takes the entry. `Closed`, `Locked` or `Inactive`: stop and say which; reopening a period
  is the user's decision, made in Acumatica. No row: the period is not generated; say so.
- `Adjustment Period` true (period 13): only when the user named it.

## J3 — Chart of accounts

```giql
FROM [PX.Objects.GL.Account] As Account
LEFT JOIN [PX.Objects.GL.AccountClass] As AccountClass ON Account.AccountClassID = AccountClass.AccountClassID
ORDER BY Account.Type, Account.AccountCD
SELECT Account.AccountCD AS 'Account', Account.Description AS 'Description', Account.Type AS 'Type', Account.AccountClassID AS 'Account Class', AccountClass.Descr AS 'Account Class Description', Account.Active AS 'Active', Account.ControlAccountModule AS 'Control Account Module', Account.IsCashAccount AS 'Cash Account', Account.CuryID AS 'Currency'
```

No parameters.
- `Account` and `Description` decode each line in the preview. `Type` (A, L, I, E) tells the reader
  which side increases the account.
- `Active` false: refuse the line. `Control Account Module` set (`AR`, `AP`, `IN`, `FA`, `TX`, `DR`,
  `PO`, `SO`): the account is a subledger's control account; ask (`references/acumatica/validation.md`).
  `Cash Account` true: ask. `Currency` set and different from the entry's currency: refuse.

## J4 — Subaccount segments

```giql
FROM [PX.Objects.CS.Segment] As Segment
LEFT JOIN [PX.Objects.CS.SegmentValue] As SegmentValue ON Segment.DimensionID = SegmentValue.DimensionID AND Segment.SegmentID = SegmentValue.SegmentID
WHERE Segment.DimensionID = 'SUBACCOUNT'
ORDER BY Segment.SegmentID, SegmentValue.Value
SELECT Segment.SegmentID AS 'Segment ID', Segment.Descr AS 'Description', Segment.Length AS 'Length', Segment.Validate AS 'Validate', SegmentValue.Value AS 'Value', SegmentValue.Descr AS 'Value Description', SegmentValue.Active AS 'Active'
```

No parameters.
- Segments in `Segment ID` order; each `Length` is its characters in the subaccount code. The lengths
  add up to the subaccount's length; split each line's subaccount with them.
- `Validate` true: the value must be one of the segment's values and `Active`. False: any value of the
  right length is accepted; show it as "(not in list)".
- No rows: the tenant has no subaccount segments; use `null` for every line's subaccount.

## J5 — Unposted batches in the period (summary)

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
- **Before the write:** a GL batch in the same branch and period with the same description, or the
  same debit total, is a possible duplicate of the entry being booked. Show it (number, status, total,
  description) and ask before building. Nothing similar: say the check found none.
- **After the write:** the new batch's row gives its `Batch Status` (`H` On Hold, `B` Balanced, `D`
  Pending Approval) and its `Debit Total` and `Credit Total` in base currency. Once released and posted
  the batch leaves this list.
- `Branch ID` is the **header** branch: a batch is listed under the branch on its header, whatever
  branches its lines carry.

## J6 — The batch's lines (detail)

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
- Always set `Batch Number`: the batch Acumatica assigned, or the last batch number the user gave.
  Never leave it out of the map — this level returns nothing when it is missing.
- One row per line, in line order, with `Account`, `Subaccount`, `Debit Amount` and `Credit Amount`
  in base currency. In a foreign-currency entry these are the converted amounts; the entry's own
  amounts are the totals the finish set reads back.
- Inter-branch lines that Acumatica generates on release are not here before release.

## J7 — One batch by its number

```giql
DECLARE [Batch Number] SCHEMA [PX.Objects.GL.Batch].BatchNbr AS 'Batch Number' REQUIRED
FROM [PX.Objects.GL.Batch] As Batch
INNER JOIN [PX.Objects.GL.GLTran] As Tran ON Batch.Module = Tran.Module AND Batch.BatchNbr = Tran.BatchNbr
LEFT JOIN [PX.Objects.GL.Account] As Account ON Tran.AccountID = Account.AccountID
WHERE Batch.Module = 'GL' AND Batch.BatchNbr = [Batch Number]
ORDER BY Tran.LineNbr
SELECT Batch.BatchNbr AS 'Batch Number', Batch.BranchID AS 'Batch Branch Code', Batch.LedgerID AS 'Ledger Code', Batch.FinPeriodID AS 'Financial Period', Batch.DateEntered AS 'Transaction Date', Batch.Status AS 'Batch Status', Batch.Released AS 'Released', Batch.Posted AS 'Posted', Batch.Hold AS 'Hold', Batch.Description AS 'Batch Description', Batch.OrigBatchNbr AS 'Original Batch Number', Batch.AutoReverse AS 'Auto Reverse', Batch.DebitTotal AS 'Batch Debit Total', Batch.CreditTotal AS 'Batch Credit Total', Tran.LineNbr AS 'Line Number', Tran.BranchID AS 'Line Branch Code', Tran.AccountID AS 'Account Code', Account.Description AS 'Account Description', Tran.SubID AS 'Subaccount Code', Tran.TranDesc AS 'Line Description', Tran.RefNbr AS 'Reference Number', Tran.DebitAmt AS 'Line Debit Amount', Tran.CreditAmt AS 'Line Credit Amount', Batch.CuryID AS 'Currency ID', Tran.CuryDebitAmt AS 'Line Currency Debit Amount', Tran.CuryCreditAmt AS 'Line Currency Credit Amount', Tran.ProjectID AS 'Project', Tran.TaskID AS 'Project Task', Tran.CostCodeID AS 'Cost Code', Tran.Qty AS 'Quantity', Tran.UOM AS 'Unit of Measure', Tran.NonBillable AS 'Non Billable'
```

| Key | Required | Binds to |
|---|---|---|
| `Batch Number` | yes | `GL.Batch.BatchNbr` |
- `Batch Number` exactly as the user gave it (`GL002519`); module GL only.
- The header columns repeat on every row: read them from the first row, never sum them. Sum the line
  amounts and compare with `Batch Debit Total` / `Batch Credit Total`.
- `Batch Branch Code` and `Project` come back padded with trailing spaces; trim them.
- Two sets of amounts: base currency (`Line Debit Amount`, `Line Credit Amount`) and the batch's
  currency (`Line Currency Debit Amount`, `Line Currency Credit Amount`). A reversal copies the
  currency amounts.
- `Batch Status` uses the same codes as J5 (`H`, `B`, `U`, `P`, `C`, `D`, `J`, `V`, `S`), with
  `Released`, `Posted` and `Hold` beside it.

Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-13 from
d992d37. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
