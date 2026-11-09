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
| A1 | `ar-aging` | one row per AR document open at the as-of point: now, a period end or a date | every AR aging; Current AR for CEI and best possible DSO |
| A2 | `ar-subledger-balance-by-period` | period × branch × account × subaccount | AR reconciliation, trend, retainage |
| A3 | `ar-subledger-activity-by-period` | period × branch × account × subaccount | the period's sales and payments, for DSO, CEI and turnover |
| P1 | `ap-aging` | one row per AP document open at the as-of point: now, a period end or a date | every AP aging, the payment plan, vendor credits |
| P2 | `ap-subledger-balance-by-period` | period × branch × account × subaccount | AP reconciliation, trend, retainage |
| P3 | `ap-subledger-activity-by-period` | period × branch × account × subaccount | the period's purchases and payments, for DPO and turnover |
| G1 | `gl-account-balances-by-period-range`, detail level | period × account × subaccount | the GL side of the tie |

The shared method files (`uses.md`, `buckets-and-ratios.md`) name pulls by their role: **the aging
pull** is A1 (AR) or P1 (AP); **the subledger balance by period** is A2 or P2; **the activity** for
ratios is A3 or P3; **the GL control account** is G1.

## Which pulls

An aging, a list or a figure is one pull. The rest run only when the user asks for them.

- **An aging, whatever the as-of:** A1 for AR, P1 for AP. Now: `Fin Period` and `As Of Date` both
  null. A period end: `Fin Period`. A date: `As Of Date`. Never both: that returns no rows.
- **The period and the as-of date:** period `MMYYYY` is calendar month `MM` of year `YYYY`; the
  as-of date is its last day, and today falls in this month's period. Run K1 then K2 only when the
  user says their periods are not calendar months, or when an aging pull for a period returns no
  rows.
- **Ratios** (asked): A2 or P2 and A3 or P3 over the window, one call each.
- **A trend** (asked): A2 or P2 over the whole window in one call; A1 or P1 once per period only when
  the past-due share or CEI is asked for each period.
- **Retainage and deposits** (asked): A2 or P2 for the as-of period.
- **The GL tie** (asked), **one period:** A1 or P1 with `Fin Period`, totalled per branch and
  `ARAccountID` / `APAccountID`, then G1 once per branch and control account, then the carry-forward
  calls the G1 section describes for any subaccount without a row. No A2 or P2: as of a period the
  aging equals the history's `Balance` (validated on `gab-dev` at three periods per side), so the
  history adds a third column that cannot differ from the first. A1 and P1 carry no subaccount, so
  compare the GL with them per account in total; when that differs, run A2 or P2 for the period and
  compare subaccount by subaccount. G1 needs the branch's ledger: K1.
- **The GL tie** (asked), **several periods:** A2 or P2 over the window in one call (in place of one
  aging per period), then G1 as above with the window's periods. Add A2 or P2 to a one-period tie
  only when the user asks about retainage, which is in the history (`RetainedBalance`) and not in the
  aging.

**One customer or vendor:** pass its code as `Business Account` to the aging (A1 or P1), and to
the history (A2 or P2) when it is asked for. Leave it null for everyone. The GL has no customer or
vendor dimension, so a one-customer or one-vendor run has no GL tie: say so if the user asks for one.

Run A1 or P1 once with `Branch ID` null for every branch (the rows carry the branch), or once per
branch named. A2/P2 likewise.

## K1 — branches, companies and ledgers

```giql
FROM [PX.Objects.GL.Branch] As Branch
INNER JOIN [PX.Objects.GL.DAC.Organization] As Organization ON Branch.OrganizationID = Organization.OrganizationID
INNER JOIN [PX.Objects.GL.Ledger] As Ledger ON Branch.LedgerID = Ledger.LedgerID
ORDER BY Organization.OrganizationCD, Branch.BranchCD
SELECT Branch.BranchCD AS 'Branch ID', Branch.AcctName AS 'Branch Name', Organization.OrganizationCD AS 'Company ID', Organization.OrganizationName AS 'Company Name', Ledger.LedgerCD AS 'Ledger ID', Branch.BaseCuryID AS 'Base Currency ID', Branch.Active AS 'Active'
```

No parameters.
Gives each branch's `Company ID` (for K2) and `Ledger ID` (for G1). Trim the padded codes.

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
- **`End Date` is the next period's first day.** The as-of date is `End Date` minus one day; days in
  the period are `End Date` − `Start Date`.
- **`Status`** `Open` marks a period still open: an aging as of an open period can still change, so
  label it preliminary.
- One call per company, from K1.

## A1 — AR aging (as of now, a period end or a date)

```giql
DECLARE [FinPeriod] SCHEMA [PX.Objects.AR.ARRegister].FinPeriodID AS 'Fin Period'
DECLARE [AsOfDate] SCHEMA [PX.Objects.AR.ARRegister].DocDate AS 'As Of Date'
DECLARE [CompanyID] SCHEMA [PX.Objects.GL.DAC.Organization].OrganizationCD AS 'Company ID'
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Branch].BranchCD AS 'Branch ID'
DECLARE [CustomerCode] SCHEMA [PX.Objects.AR.ARRegister].CustomerID AS 'Business Account'
FROM [PX.Objects.AR.ARRegister] AS Document
CROSS JOIN [PX.Objects.AP.APAROrd] AS APAROrd
LEFT JOIN [PX.Objects.GL.Branch] AS Branch ON Document.BranchID = Branch.BranchID
LEFT JOIN [PX.Objects.GL.DAC.Organization] AS Organization ON Branch.OrganizationID = Organization.OrganizationID
LEFT JOIN [PX.Objects.AR.ARAdjust] AS ARAdjusted
  ON Document.DocType = ARAdjusted.AdjdDocType
  AND Document.RefNbr = ARAdjusted.AdjdRefNbr
  AND (ARAdjusted.AdjgFinPeriodID <= [FinPeriod] OR [FinPeriod] IS NULL)
  AND (ARAdjusted.AdjgDocDate <= [AsOfDate] OR [AsOfDate] IS NULL)
  AND ARAdjusted.Released = {=True}
  AND APAROrd.Ord = {=0}
LEFT JOIN [PX.Objects.AR.ARAdjust] AS ARAdjusting
  ON Document.DocType = ARAdjusting.AdjgDocType
  AND Document.RefNbr = ARAdjusting.AdjgRefNbr
  AND (ARAdjusting.AdjgFinPeriodID <= [FinPeriod] OR [FinPeriod] IS NULL)
  AND (ARAdjusting.AdjgDocDate <= [AsOfDate] OR [AsOfDate] IS NULL)
  AND ARAdjusting.Released = {=True}
  AND APAROrd.Ord = {=1}
LEFT JOIN [PX.Objects.GL.FinPeriods.MasterFinPeriod] AS MasterFinPeriod ON MasterFinPeriod.FinPeriodID = [FinPeriod]
WHERE Document.DocType != 'CSL'
  AND Document.DocType != 'RCS'
  AND Document.Released = True
  AND ([FinPeriod] IS NULL OR [AsOfDate] IS NULL)
  AND (([FinPeriod] IS NOT NULL AND Document.FinPeriodID <= [FinPeriod] AND (Document.ClosedFinPeriodID > [FinPeriod] OR Document.ClosedFinPeriodID IS NULL))
    OR ([AsOfDate] IS NOT NULL AND Document.DocDate <= [AsOfDate] AND (Document.ClosedDate > [AsOfDate] OR Document.ClosedDate IS NULL))
    OR ([FinPeriod] IS NULL AND [AsOfDate] IS NULL AND Document.ClosedFinPeriodID IS NULL))
  AND (Organization.OrganizationCD = [CompanyID] OR [CompanyID] IS NULL)
  AND (Branch.BranchCD = [BranchID] OR [BranchID] IS NULL)
  AND (Document.CustomerID = [CustomerCode] OR [CustomerCode] IS NULL)
  AND (([FinPeriod] IS NULL OR MasterFinPeriod.FinPeriodID IS NOT NULL))
GROUP BY
  Document.DocType,
  Document.RefNbr,
  Document.CustomerID,
  Document.ARAccountID
ORDER BY
  Document.DueDate,
  Document.RefNbr
SELECT Document.DocType AS 'DocType', Document.RefNbr AS 'RefNbr', Document.Status AS 'Status', Document.CustomerID AS 'CustomerCode', Document.CustomerID_description AS 'CustomerName', Organization.OrganizationCD AS 'CompanyCode', Branch.BranchCD AS 'BranchCode', Document.ARAccountID AS 'ARAccountID', Document.DocDate AS 'Date', Document.DueDate AS 'DueDate', Document.FinPeriodID AS 'Period', Document.CuryID AS 'Currency', {=IIF([Document.DocType]='CRM' Or [Document.DocType]='PMT' Or [Document.DocType]='PPM' Or [Document.DocType]='RPM' Or [Document.DocType]='SMB',-1,1)*[Document.OrigDocAmt]} SCHEMA [PX.Objects.AR.ARRegister].OrigDocAmt AS 'Amount', SUM({=ISNULL([ARAdjusting.AdjAmt]*IIF([ARAdjusting.AdjgDocType]=[ARAdjusting.AdjdDocType] And [ARAdjusting.AdjgRefNbr]=[ARAdjusting.AdjdRefNbr],-1*IIF([ARAdjusting.AdjgDocType]='CRM' Or [ARAdjusting.AdjgDocType]='PMT' Or [ARAdjusting.AdjgDocType]='PPM' Or [ARAdjusting.AdjgDocType]='RPM' Or [ARAdjusting.AdjgDocType]='SMB',-1,1),IIF([ARAdjusting.AdjdDocType]='CRM' Or [ARAdjusting.AdjdDocType]='PMT' Or [ARAdjusting.AdjdDocType]='PPM' Or [ARAdjusting.AdjdDocType]='RPM' Or [ARAdjusting.AdjdDocType]='SMB',-1,1)),0)}) SCHEMA [PX.Objects.AR.ARAdjust].AdjAmt AS 'Adjusting Amount', SUM({=ISNULL(([ARAdjusted.AdjAmt]+[ARAdjusted.AdjDiscAmt]+[ARAdjusted.AdjWOAmt]+[ARAdjusted.RGOLAmt])*IIF([ARAdjusted.AdjgDocType]=[ARAdjusted.AdjdDocType] And [ARAdjusted.AdjgRefNbr]=[ARAdjusted.AdjdRefNbr],0,IIF( [ARAdjusted.AdjdDocType] = 'CRM' And ([ARAdjusted.AdjgDocType] = 'PMT' Or [ARAdjusted.AdjgDocType] = 'RPM' Or [ARAdjusted.AdjgDocType] = 'PPM'),1,IIF([ARAdjusted.AdjgDocType]='CRM' Or [ARAdjusted.AdjgDocType]='PMT' Or [ARAdjusted.AdjgDocType]='PPM' Or [ARAdjusted.AdjgDocType]='RPM' Or [ARAdjusted.AdjgDocType]='SMB',-1,1))),0)}) SCHEMA [PX.Objects.AR.ARAdjust].AdjAmt AS 'Adjusted Amount', {=MAX(IIF([Document.DocType]='CRM' Or [Document.DocType]='PMT' Or [Document.DocType]='PPM' Or [Document.DocType]='RPM' Or [Document.DocType]='SMB',-1,1)*[Document.OrigDocAmt])+SUM(ISNULL([ARAdjusting.AdjAmt]*IIF([ARAdjusting.AdjgDocType]=[ARAdjusting.AdjdDocType] And [ARAdjusting.AdjgRefNbr]=[ARAdjusting.AdjdRefNbr],-1*IIF([ARAdjusting.AdjgDocType]='CRM' Or [ARAdjusting.AdjgDocType]='PMT' Or [ARAdjusting.AdjgDocType]='PPM' Or [ARAdjusting.AdjgDocType]='RPM' Or [ARAdjusting.AdjgDocType]='SMB',-1,1),IIF([ARAdjusting.AdjdDocType]='CRM' Or [ARAdjusting.AdjdDocType]='PMT' Or [ARAdjusting.AdjdDocType]='PPM' Or [ARAdjusting.AdjdDocType]='RPM' Or [ARAdjusting.AdjdDocType]='SMB',-1,1)),0))+SUM(ISNULL(([ARAdjusted.AdjAmt]+[ARAdjusted.AdjDiscAmt]+[ARAdjusted.AdjWOAmt]+[ARAdjusted.RGOLAmt])*IIF([ARAdjusted.AdjgDocType]=[ARAdjusted.AdjdDocType] And [ARAdjusted.AdjgRefNbr]=[ARAdjusted.AdjdRefNbr],0,IIF([ARAdjusted.AdjdDocType]='CRM' And ([ARAdjusted.AdjgDocType]='PMT' Or [ARAdjusted.AdjgDocType]='RPM' Or [ARAdjusted.AdjgDocType]='PPM'),1,IIF([ARAdjusted.AdjgDocType]='CRM' Or [ARAdjusted.AdjgDocType]='PMT' Or [ARAdjusted.AdjgDocType]='PPM' Or [ARAdjusted.AdjgDocType]='RPM' Or [ARAdjusted.AdjgDocType]='SMB',-1,1))),0))} SCHEMA [PX.Objects.AR.ARRegister].OrigDocAmt AS 'Open Balance'
```

| Key | Required | Binds to |
|---|---|---|
| `Fin Period` | no | `AR.ARRegister.FinPeriodID` |
| `As Of Date` | no | `AR.ARRegister.DocDate` |
| `Company ID` | no | `GL.DAC.Organization.OrganizationCD` |
| `Branch ID` | no | `GL.Branch.BranchCD` |
| `Business Account` | no | `AR.ARRegister.CustomerID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
- **The as-of point:** both `Fin Period` and `As Of Date` null for now (every document not yet
  closed, every application whatever period it is posted to); `Fin Period` (`MMYYYY`) for a period
  end, which ties to the history, the GL and AR630500; `As Of Date` (`YYYY-MM-DD`) for a date, which
  ties to no GL balance. Pass at most one: both given returns no rows.
- **Period or date, when the user's words allow either.** "As of June month end" for a
  reconciliation, a trend or a comparison with the GL or AR630500 is the period. "As of 15 March", or
  any day that is not a month end, is the date. When a month-end date could mean either and the
  answer will be tied to the GL, use the period and say so in the basis line.
- `Open Balance` is signed (debits positive, credits negative) and is the column to age and total.
  `Amount`, `Adjusting Amount` and `Adjusted Amount` are its components. **Drop the rows whose
  `Open Balance` is 0**: documents still flagged open with nothing left on them.
- `DueDate` is the invoice's; credits have none. `CustomerCode` and `CustomerName` name the customer;
  `CompanyCode` and `BranchCode` come back padded, so trim them.
- `ARAccountID` is the control account each document sits on: total per account.

## A2 — AR subledger balance by period

```giql
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.FinPeriods.TableDefinition.FinPeriod].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.FinPeriods.TableDefinition.FinPeriod].FinPeriodID AS 'Period To' REQUIRED
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Branch].BranchCD AS 'Branch ID'
DECLARE [CustomerCode] SCHEMA [PX.Objects.AR.ARHistory].CustomerID AS 'Business Account'
FROM FinPeriod
INNER JOIN ARHistory
  ON FinPeriod.FinPeriodID >= ARHistory.FinPeriodID
LEFT JOIN [PX.Objects.AR.ARHistory] AS ARHistoryPrev
  ON ARHistory.BranchID = ARHistoryPrev.BranchID
  AND ARHistory.AccountID = ARHistoryPrev.AccountID
  AND ARHistory.SubID = ARHistoryPrev.SubID
  AND ARHistory.CustomerID = ARHistoryPrev.CustomerID
  AND ARHistory.FinPeriodID < ARHistoryPrev.FinPeriodID
  AND {=[FinPeriod.FinPeriodID]} >= ARHistoryPrev.FinPeriodID
INNER JOIN [PX.Objects.GL.Branch] AS Branch
  ON ARHistory.BranchID = Branch.BranchID
INNER JOIN [PX.Objects.GL.Account] AS Account
  ON ARHistory.AccountID = Account.AccountID
INNER JOIN [PX.Objects.GL.Sub] AS Sub
  ON ARHistory.SubID = Sub.SubID
WHERE FinPeriod.OrganizationID = {=0}
  AND ARHistoryPrev.FinPeriodID IS NULL
  AND (ARHistory.FinYtdBalance != {=0}
  OR ARHistory.FinYtdDeposits != {=0}
  OR ARHistory.FinPtdRevalued != {=0}
  OR ARHistory.FinYtdRetainageWithheld != {=0}
  OR ARHistory.FinYtdRetainageReleased != {=0})
  AND (FinPeriod.FinPeriodID >= [PeriodFrom])
  AND (FinPeriod.FinPeriodID <= [PeriodTo])
  AND ((Branch.BranchCD = [BranchID] OR [BranchID] IS NULL))
  AND ((ARHistory.CustomerID = [CustomerCode] OR [CustomerCode] IS NULL))
GROUP BY
  FinPeriod.FinPeriodID,
  ARHistory.BranchID,
  ARHistory.AccountID,
  ARHistory.SubID
SELECT FinPeriod.FinPeriodID AS 'FinPeriodID', Branch.BranchCD AS 'BranchID', Account.AccountCD AS 'AccountiD', Sub.SubCD AS 'SubID', SUM({=[ARHistory.FinYtdBalance]-[ARHistory.FinYtdDeposits]}) SCHEMA [PX.Objects.AR.ARHistory].FinYtdBalance AS 'Balance', SUM({=[ARHistory.FinYtdRetainageWithheld]-[ARHistory.FinYtdRetainageReleased]}) SCHEMA [PX.Objects.AR.ARHistory].FinYtdRetainageWithheld AS 'RetainedBalance', SUM({=IIF([ARHistory.FinPeriodID]=[FinPeriod.FinPeriodID], [ARHistory.FinPtdRevalued], 0)}) SCHEMA [PX.Objects.AR.ARHistory].FinPtdRevalued AS 'UnrealizedGainLoss'
```

| Key | Required | Binds to |
|---|---|---|
| `Period From` | yes | `GL.FinPeriods.TableDefinition.FinPeriod.FinPeriodID` |
| `Period To` | yes | `GL.FinPeriods.TableDefinition.FinPeriod.FinPeriodID` |
| `Branch ID` | no | `GL.Branch.BranchCD` |
| `Business Account` | no | `AR.ARHistory.CustomerID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
- `Balance` on the control account (the account A1's documents sit on) is the total A1 is compared
  with in a reconciliation, for the same branch and period.
- Other accounts (customer deposits, others) come back on their own rows: show them, do not add them
  to the aging.
- `RetainedBalance` is outside `Balance`. Leave out `13YYYY` adjustment periods in a trend.
- The account column is captioned `AccountiD` (the supplied query's caption).

## A3 — AR activity by period

```giql
DECLARE [PeriodFrom] SCHEMA [PX.Objects.AR.ARHistory].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.AR.ARHistory].FinPeriodID AS 'Period To' REQUIRED
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Branch].BranchCD AS 'Branch ID'
DECLARE [CustomerCode] SCHEMA [PX.Objects.AR.ARHistory].CustomerID AS 'Business Account'
FROM [PX.Objects.AR.ARHistory] As Hist
INNER JOIN [PX.Objects.GL.Branch] As Branch ON Hist.BranchID = Branch.BranchID
INNER JOIN [PX.Objects.GL.Account] As Account ON Hist.AccountID = Account.AccountID
INNER JOIN [PX.Objects.GL.Sub] As Sub ON Hist.SubID = Sub.SubID
WHERE Hist.FinPeriodID >= [PeriodFrom]
AND Hist.FinPeriodID <= [PeriodTo]
AND (Branch.BranchCD = [BranchID] OR [BranchID] IS NULL)
AND (Hist.CustomerID = [CustomerCode] OR [CustomerCode] IS NULL)
AND (Hist.FinPtdSales != 0 OR Hist.FinPtdPayments != 0)
GROUP BY Hist.FinPeriodID, Branch.BranchCD, Account.AccountCD, Sub.SubCD
ORDER BY Hist.FinPeriodID, Branch.BranchCD, Account.AccountCD, Sub.SubCD
SELECT Hist.FinPeriodID AS 'FinPeriodID', Branch.BranchCD AS 'BranchID', Account.AccountCD AS 'AccountID', Sub.SubCD AS 'SubID', SUM(Hist.FinPtdSales) AS 'Period Sales', SUM(Hist.FinPtdPayments) AS 'Period Payments'
```

| Key | Required | Binds to |
|---|---|---|
| `Period From` | yes | `AR.ARHistory.FinPeriodID` |
| `Period To` | yes | `AR.ARHistory.FinPeriodID` |
| `Branch ID` | no | `GL.Branch.BranchCD` |
| `Business Account` | no | `AR.ARHistory.CustomerID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
- Each period's own sales and payments; a period with no row had none. Drop a row whose two totals
  are both zero.
- `Period Sales` include cash sales, which are also in `Period Payments`.

## P1 — AP aging (as of now, a period end or a date)

```giql
DECLARE [FinPeriod] SCHEMA [PX.Objects.AP.APRegister].FinPeriodID AS 'Fin Period'
DECLARE [AsOfDate] SCHEMA [PX.Objects.AP.APRegister].DocDate AS 'As Of Date'
DECLARE [CompanyID] SCHEMA [PX.Objects.GL.DAC.Organization].OrganizationCD AS 'Company ID'
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Branch].BranchCD AS 'Branch ID'
DECLARE [VendorCode] SCHEMA [PX.Objects.AP.APRegister].VendorID AS 'Business Account'
FROM [PX.Objects.AP.APRegister] AS Document
CROSS JOIN [PX.Objects.AP.APAROrd] AS APAROrd
LEFT JOIN [PX.Objects.GL.Branch] AS Branch ON Document.BranchID = Branch.BranchID
LEFT JOIN [PX.Objects.GL.DAC.Organization] AS Organization ON Branch.OrganizationID = Organization.OrganizationID
LEFT JOIN [PX.Objects.AP.APAdjust] AS APAdjusted
  ON Document.DocType = APAdjusted.AdjdDocType
  AND Document.RefNbr = APAdjusted.AdjdRefNbr
  AND (APAdjusted.AdjgFinPeriodID <= [FinPeriod] OR [FinPeriod] IS NULL)
  AND (APAdjusted.AdjgDocDate <= [AsOfDate] OR [AsOfDate] IS NULL)
  AND APAdjusted.Released = {=True}
  AND APAROrd.Ord = {=0}
LEFT JOIN [PX.Objects.AP.APAdjust] AS APAdjusting
  ON Document.DocType = APAdjusting.AdjgDocType
  AND Document.RefNbr = APAdjusting.AdjgRefNbr
  AND (APAdjusting.AdjgFinPeriodID <= [FinPeriod] OR [FinPeriod] IS NULL)
  AND (APAdjusting.AdjgDocDate <= [AsOfDate] OR [AsOfDate] IS NULL)
  AND APAdjusting.Released = {=True}
  AND APAROrd.Ord = {=1}
LEFT JOIN [PX.Objects.AP.APInvoice] AS APInvoice
  ON Document.DocType = APInvoice.DocType
  AND Document.RefNbr = APInvoice.RefNbr
LEFT JOIN [PX.Objects.GL.FinPeriods.MasterFinPeriod] AS MasterFinPeriod ON MasterFinPeriod.FinPeriodID = [FinPeriod]
WHERE Document.DocType != 'QCK'
  AND Document.DocType != 'VQC'
  AND (Document.Voided = False
  OR Document.Prebooked = False
  OR Document.DocType = 'QCK')
  AND (Document.Released = True
  OR Document.Prebooked = True)
  AND (([FinPeriod] IS NOT NULL AND [AsOfDate] IS NULL
    AND Document.FinPeriodID <= [FinPeriod]
    AND (Document.ClosedFinPeriodID > [FinPeriod] OR Document.ClosedFinPeriodID IS NULL))
    OR ([FinPeriod] IS NULL AND [AsOfDate] IS NOT NULL
    AND Document.DocDate <= [AsOfDate]
    AND (Document.ClosedDate > [AsOfDate] OR Document.ClosedDate IS NULL))
    OR ([FinPeriod] IS NULL AND [AsOfDate] IS NULL AND Document.ClosedFinPeriodID IS NULL))
  AND (Organization.OrganizationCD = [CompanyID] OR [CompanyID] IS NULL)
  AND (Branch.BranchCD = [BranchID] OR [BranchID] IS NULL)
  AND (Document.VendorID = [VendorCode] OR [VendorCode] IS NULL)
  AND (([FinPeriod] IS NULL OR MasterFinPeriod.FinPeriodID IS NOT NULL))
GROUP BY
  Document.DocType,
  Document.RefNbr
ORDER BY
  APInvoice.DueDate,
  Document.RefNbr
SELECT Document.DocType AS 'DocType', Document.RefNbr AS 'RefNbr', Document.Status AS 'Status', MAX(Document.VendorID) AS 'VendorCode', Document.VendorID_description AS 'VendorName', Organization.OrganizationCD AS 'Company Code', Branch.BranchCD AS 'Branch Code', MAX(Document.APAccountID) AS 'APAccountID', Document.DocDate AS 'Date', APInvoice.DueDate AS 'DueDate', Document.FinPeriodID AS 'Period', Document.CuryID AS 'Currency', {=SWITCH([Document.DocType]='REF',1, [Document.DocType]='VRF',1,[Document.DocType]='INV',1,[Document.DocType]='ACR',1,[Document.DocType]='ADR',-1,[Document.DocType]='CHK',-1,[Document.DocType]='VCK',-1,[Document.DocType]='PPM',-1,[Document.DocType]='QCK',0,[Document.DocType]='VQC',0,1)*IIF([Document.DocType]<>'PPM' Or [APInvoice.RefNbr]=Null,[Document.OrigDocAmt], 0)} SCHEMA [PX.Objects.AP.APRegister].OrigDocAmt AS 'Amount', SUM({=ISNULL([APAdjusting.AdjAmt]*IIF([APAdjusting.AdjgDocType]=[APAdjusting.AdjdDocType] And [APAdjusting.AdjgRefNbr]=[APAdjusting.AdjdRefNbr],-1*SWITCH([APAdjusting.AdjgDocType]='REF',1,[APAdjusting.AdjgDocType]='VRF',1,[APAdjusting.AdjgDocType]='INV',1,[APAdjusting.AdjgDocType]='ACR',1,[APAdjusting.AdjgDocType]='ADR',-1,[APAdjusting.AdjgDocType]='CHK',-1,[APAdjusting.AdjgDocType]='VCK',-1,[APAdjusting.AdjgDocType]='PPM',-1,[APAdjusting.AdjgDocType]='QCK',0,[APAdjusting.AdjgDocType]='VQC',0,1),IIF([APAdjusting.AdjdDocType]='PPM' AND ([APAdjusting.AdjgDocType]='CHK' OR [APAdjusting.AdjgDocType]='VCK' OR [APAdjusting.AdjgDocType]='PPM'), 1, SWITCH([APAdjusting.AdjdDocType]='REF',1, [APAdjusting.AdjdDocType]='VRF',1,[APAdjusting.AdjdDocType]='INV',1,[APAdjusting.AdjdDocType]='ACR',1,[APAdjusting.AdjdDocType]='ADR',-1,[APAdjusting.AdjdDocType]='CHK',-1,[APAdjusting.AdjdDocType]='VCK',-1,[APAdjusting.AdjdDocType]='PPM',-1,[APAdjusting.AdjdDocType]='QCK',0,[APAdjusting.AdjdDocType]='VQC',0,1))),0)}) SCHEMA [PX.Objects.AP.APAdjust].AdjAmt AS 'Adjusting Amount', SUM({=ISNULL(([APAdjusted.AdjAmt]+[APAdjusted.AdjDiscAmt]+[APAdjusted.AdjWhTaxAmt]+[APAdjusted.RGOLAmt])*IIF([APAdjusted.AdjgDocType]=[APAdjusted.AdjdDocType] And [APAdjusted.AdjgRefNbr]=[APAdjusted.AdjdRefNbr],0,IIF([APAdjusted.AdjgDocType]='CHK' AND [APAdjusted.AdjdDocType]='ADR' OR [APAdjusted.AdjgDocType]='VCK' AND [APAdjusted.AdjdDocType]='ADR', 1, SWITCH([APAdjusted.AdjgDocType]='REF',1,[APAdjusted.AdjgDocType]='VRF',1,[APAdjusted.AdjgDocType]='INV',1,[APAdjusted.AdjgDocType]='ACR',1,[APAdjusted.AdjgDocType]='ADR',-1,[APAdjusted.AdjgDocType]='CHK',-1,[APAdjusted.AdjgDocType]='VCK',-1,[APAdjusted.AdjgDocType]='PPM',-1,[APAdjusted.AdjgDocType]='QCK',0,[APAdjusted.AdjgDocType]='VQC',0,1))),0)}) SCHEMA [PX.Objects.AP.APAdjust].AdjAmt AS 'Adjusted Amount', {=MAX(SWITCH([Document.DocType]='REF',1, [Document.DocType]='VRF',1,[Document.DocType]='INV',1,[Document.DocType]='ACR',1,[Document.DocType]='ADR',-1,[Document.DocType]='CHK',-1,[Document.DocType]='VCK',-1,[Document.DocType]='PPM',-1,[Document.DocType]='QCK',0,[Document.DocType]='VQC',0,1)*IIF([Document.DocType]<>'PPM' Or [APInvoice.RefNbr]=Null,[Document.OrigDocAmt], 0))+SUM(ISNULL([APAdjusting.AdjAmt]*IIF([APAdjusting.AdjgDocType]=[APAdjusting.AdjdDocType] And [APAdjusting.AdjgRefNbr]=[APAdjusting.AdjdRefNbr],-1*SWITCH([APAdjusting.AdjgDocType]='REF',1,[APAdjusting.AdjgDocType]='VRF',1,[APAdjusting.AdjgDocType]='INV',1,[APAdjusting.AdjgDocType]='ACR',1,[APAdjusting.AdjgDocType]='ADR',-1,[APAdjusting.AdjgDocType]='CHK',-1,[APAdjusting.AdjgDocType]='VCK',-1,[APAdjusting.AdjgDocType]='PPM',-1,[APAdjusting.AdjgDocType]='QCK',0,[APAdjusting.AdjgDocType]='VQC',0,1),IIF([APAdjusting.AdjdDocType]='PPM' AND ([APAdjusting.AdjgDocType]='CHK' OR [APAdjusting.AdjgDocType]='VCK' OR [APAdjusting.AdjgDocType]='PPM'), 1, SWITCH([APAdjusting.AdjdDocType]='REF',1, [APAdjusting.AdjdDocType]='VRF',1,[APAdjusting.AdjdDocType]='INV',1,[APAdjusting.AdjdDocType]='ACR',1,[APAdjusting.AdjdDocType]='ADR',-1,[APAdjusting.AdjdDocType]='CHK',-1,[APAdjusting.AdjdDocType]='VCK',-1,[APAdjusting.AdjdDocType]='PPM',-1,[APAdjusting.AdjdDocType]='QCK',0,[APAdjusting.AdjdDocType]='VQC',0,1))),0))+SUM(ISNULL(([APAdjusted.AdjAmt]+[APAdjusted.AdjDiscAmt]+[APAdjusted.AdjWhTaxAmt]+[APAdjusted.RGOLAmt])*IIF([APAdjusted.AdjgDocType]=[APAdjusted.AdjdDocType] And [APAdjusted.AdjgRefNbr]=[APAdjusted.AdjdRefNbr],0,IIF([APAdjusted.AdjgDocType]='CHK' AND [APAdjusted.AdjdDocType]='ADR' OR [APAdjusted.AdjgDocType]='VCK' AND [APAdjusted.AdjdDocType]='ADR', 1, SWITCH([APAdjusted.AdjgDocType]='REF',1,[APAdjusted.AdjgDocType]='VRF',1,[APAdjusted.AdjgDocType]='INV',1,[APAdjusted.AdjgDocType]='ACR',1,[APAdjusted.AdjgDocType]='ADR',-1,[APAdjusted.AdjgDocType]='CHK',-1,[APAdjusted.AdjgDocType]='VCK',-1,[APAdjusted.AdjgDocType]='PPM',-1,[APAdjusted.AdjgDocType]='QCK',0,[APAdjusted.AdjgDocType]='VQC',0,1))),0))} SCHEMA [PX.Objects.AP.APRegister].OrigDocAmt AS 'Open Balance'
```

| Key | Required | Binds to |
|---|---|---|
| `Fin Period` | no | `AP.APRegister.FinPeriodID` |
| `As Of Date` | no | `AP.APRegister.DocDate` |
| `Company ID` | no | `GL.DAC.Organization.OrganizationCD` |
| `Branch ID` | no | `GL.Branch.BranchCD` |
| `Business Account` | no | `AP.APRegister.VendorID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
- **The as-of point** works as A1's: both null for now, `Fin Period` for a period end (ties to the
  history, the GL and AP630500), `As Of Date` for a date; never both.
- `Open Balance` is signed: bills and credit adjustments positive, debit adjustments, prepayments and
  payments negative. **Drop the rows whose `Open Balance` is 0.**
- `DueDate` comes from the bill; payments and prepayments have none. `VendorCode` and `VendorName`
  name the vendor; `Company Code` and `Branch Code` come back padded.
- `APAccountID`: a prepayment sits on the prepayment account, not the AP control account. Total per
  account.

## P2 — AP subledger balance by period

```giql
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.FinPeriods.TableDefinition.FinPeriod].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.FinPeriods.TableDefinition.FinPeriod].FinPeriodID AS 'Period To' REQUIRED
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Branch].BranchCD AS 'Branch ID'
DECLARE [VendorCode] SCHEMA [PX.Objects.AP.APHistory].VendorID AS 'Business Account'
FROM FinPeriod
INNER JOIN APHistory
  ON FinPeriod.FinPeriodID >= APHistory.FinPeriodID
LEFT JOIN [PX.Objects.AP.APHistory] AS APHistoryPrev
  ON APHistory.BranchID = APHistoryPrev.BranchID
  AND APHistory.AccountID = APHistoryPrev.AccountID
  AND APHistory.SubID = APHistoryPrev.SubID
  AND APHistory.VendorID = APHistoryPrev.VendorID
  AND APHistory.FinPeriodID < APHistoryPrev.FinPeriodID
  AND {=[FinPeriod.FinPeriodID]} >= APHistoryPrev.FinPeriodID
INNER JOIN [PX.Objects.GL.Branch] AS Branch
  ON APHistory.BranchID = Branch.BranchID
INNER JOIN [PX.Objects.GL.Account] AS Account
  ON APHistory.AccountID = Account.AccountID
INNER JOIN [PX.Objects.GL.Sub] AS Sub
  ON APHistory.SubID = Sub.SubID
WHERE FinPeriod.OrganizationID = {=0}
  AND APHistoryPrev.FinPeriodID IS NULL
  AND (APHistory.FinYtdBalance != {=0}
  OR APHistory.FinYtdDeposits != {=0}
  OR APHistory.FinPtdRevalued != {=0}
  OR APHistory.FinYtdRetainageWithheld != {=0}
  OR APHistory.FinYtdRetainageReleased != {=0})
  AND (FinPeriod.FinPeriodID >= [PeriodFrom])
  AND (FinPeriod.FinPeriodID <= [PeriodTo])
  AND ((Branch.BranchCD = [BranchID] OR [BranchID] IS NULL))
  AND ((APHistory.VendorID = [VendorCode] OR [VendorCode] IS NULL))
GROUP BY
  FinPeriod.FinPeriodID,
  APHistory.BranchID,
  APHistory.AccountID,
  APHistory.SubID
SELECT FinPeriod.FinPeriodID AS 'FinPeriodID', Branch.BranchCD AS 'BranchID', Account.AccountCD AS 'AccountiD', Sub.SubCD AS 'SubID', SUM({=[APHistory.FinYtdBalance]-[APHistory.FinYtdDeposits]}) SCHEMA [PX.Objects.AP.APHistory].FinYtdBalance AS 'Balance', SUM({=[APHistory.FinYtdRetainageWithheld]-[APHistory.FinYtdRetainageReleased]}) SCHEMA [PX.Objects.AP.APHistory].FinYtdRetainageWithheld AS 'RetainedBalance', SUM({=IIF([APHistory.FinPeriodID]=[FinPeriod.FinPeriodID], [APHistory.FinPtdRevalued], 0)}) SCHEMA [PX.Objects.AP.APHistory].FinPtdRevalued AS 'UnrealizedGainLoss'
```

| Key | Required | Binds to |
|---|---|---|
| `Period From` | yes | `GL.FinPeriods.TableDefinition.FinPeriod.FinPeriodID` |
| `Period To` | yes | `GL.FinPeriods.TableDefinition.FinPeriod.FinPeriodID` |
| `Branch ID` | no | `GL.Branch.BranchCD` |
| `Business Account` | no | `AP.APHistory.VendorID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
`Balance` on the AP control account is the total P1 is compared with in a reconciliation.
`RetainedBalance` is outside `Balance`. The account column is captioned `AccountiD`.

## P3 — AP activity by period

```giql
DECLARE [PeriodFrom] SCHEMA [PX.Objects.AP.APHistory].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.AP.APHistory].FinPeriodID AS 'Period To' REQUIRED
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Branch].BranchCD AS 'Branch ID'
DECLARE [VendorCode] SCHEMA [PX.Objects.AP.APHistory].VendorID AS 'Business Account'
FROM [PX.Objects.AP.APHistory] As Hist
INNER JOIN [PX.Objects.GL.Branch] As Branch ON Hist.BranchID = Branch.BranchID
INNER JOIN [PX.Objects.GL.Account] As Account ON Hist.AccountID = Account.AccountID
INNER JOIN [PX.Objects.GL.Sub] As Sub ON Hist.SubID = Sub.SubID
WHERE Hist.FinPeriodID >= [PeriodFrom] AND Hist.FinPeriodID <= [PeriodTo]
AND (Branch.BranchCD = [BranchID] OR [BranchID] IS NULL)
AND (Hist.VendorID = [VendorCode] OR [VendorCode] IS NULL)
AND (Hist.FinPtdPurchases != 0 OR Hist.FinPtdPayments != 0)
GROUP BY Hist.FinPeriodID, Branch.BranchCD, Account.AccountCD, Sub.SubCD
ORDER BY Hist.FinPeriodID, Branch.BranchCD, Account.AccountCD, Sub.SubCD
SELECT Hist.FinPeriodID AS 'FinPeriodID', Branch.BranchCD AS 'BranchID', Account.AccountCD AS 'AccountID', Sub.SubCD AS 'SubID', SUM(Hist.FinPtdPurchases) AS 'Period Purchases', SUM(Hist.FinPtdPayments) AS 'Period Payments'
```

| Key | Required | Binds to |
|---|---|---|
| `Period From` | yes | `AP.APHistory.FinPeriodID` |
| `Period To` | yes | `AP.APHistory.FinPeriodID` |
| `Branch ID` | no | `GL.Branch.BranchCD` |
| `Business Account` | no | `AP.APHistory.VendorID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
Each period's own purchases (released bills) and payments; a period with no row had none.

## G1 — GL control account balance

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
`Branch ID` from the aging, `Account CD` the control account, `Ledger ID` the branch's ledger from
K1.

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
For AR (an asset) `Fin. YTD Balance` is debit-positive and compares with A2 directly; for AP (a
liability) it is credit-positive and compares with P2 directly.

## When the aging and the history disagree

This only comes up when both are pulled (a several-period reconciliation, or a check the user asks
for). In order:

1. **Something was posted between the two pulls** (an aging run now, the history an hour later). Rerun
   both back to back.
2. **Different scope:** a branch or period mismatch, or a row on another account added in.
3. **A zero-balance or open-flag oddity:** recount without the zero rows.

If none explains it, report both figures and the difference as unexplained.

**When the subledger agrees with itself but not with the GL or the Acumatica report,** the
connection may not see every customer or vendor. The queries return only what the connection's
Acumatica user is allowed to see (customer and vendor restriction groups apply to the history too), so
aging and history agree with each other while both fall short. Acumatica's aged period-sensitive
report, run by a user who sees everything, shows which: compare it customer by customer, and a
customer present there and absent here is a visibility gap. Say so as a possibility to check with the
administrator, never as established, until that comparison is made.

Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-09 from
5ed670d. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
