# Data pulls

Run every query through `execute_query` exactly as written, with your values in the `parameters` map
keyed by the display labels in each table below. Don't call `prepare_query` for them and don't edit
the text; if one fails, report the error and stop. When no pull here has the grain the question needs,
build that one through `prepare_query` and name it in the basis block with the tie-out it passed.
Labels are the same on every query: `Period From` / `Period To`, `Ledger ID`, `Branch ID`, `Account 1`
… `Account 8` or `Account CD`, `Account Class`, `Subaccount CD`; a source document is always the pair
`Document Type` + `Reference Number`. A key that matches no label is dropped silently and the query
returns nothing.
**Pass every key a query declares**: an optional one you are not using goes in as `null` (the unused
`Account 2` … `Account 8` included), never left out of the map — the connector does not treat a
missing key as null on every query.
**Period values are `MMYYYY`** (`012025`), exactly as the user wrote them: never rewrite one into
the year-first `202501` that Acumatica stores internally — the connector translates, and a year-first
value matches no period, so every pull returns no rows.

| Alias | Query | Grain | Joins the business account |
|---|---|---|---|
| D0 | `subaccount-segments` | one row per segment value | — |
| D1 | `gltran-activity-by-reference` | period × branch × account × business account | yes |
| D2 | `gltran-activity-by-subaccount-reference` | period × branch × account × subaccount × business account | yes |
| D4 | `gl-budget-by-account-and-period`, detail level (the analyzer's pull 6 at subaccount grain) | account × subaccount × period | — |
| D5 | `gltran-activity-by-branch-and-account` | period × branch × account | no — the line-level control |
| D6 | `glhistory-activity-by-branch-subaccount` | period × branch × account × subaccount, from GL history | no |
| D6b | `glhistory-activity-by-subaccount-for-window` | account × subaccount summed over a window, one branch | no |
| D7 | `gltran-lines` (the analyzer's pull 7) | one row per posted line | yes |
| D8 | `gltran-source-routing` | one row per posted line, with module and source-document keys | no |
| D9 | `ar-document-lines-and-applications` | one AR document: its lines; its applications | the customer |
| D10 | `ap-document-lines-and-applications` | one AP document: its lines; its applications | no |
| D11 | `in-document-lines` | one inventory document's lines | no |
| D11-CA | `ca-transaction-splits` | one cash transaction's splits | no |
| D12 | `ar-invoice-lines-vs-inventory-issues` | one row per AR item line, matched to its inventory issues | no |
| pull 1, 3, 5, 6 | period calendar, branch list, account balances and budget (below, shared with the variance analyzer) | — | — |

There is no D3: the all-branches business account query was merged into D1.

**`Natural Net` is the column to present.** Every activity, balance and budget query here returns it:
the amount in the account's natural sign — credit less debit on income and liability (equity
included) accounts, debit less credit on asset and expense — so revenue, expense, an asset and a
liability all read positive when they grow, and budget subtracts from actual with no sign flip.
Keep the raw `Debit` / `Credit` columns for the pattern reads (credits netting invoices, allocations),
and the older debit-less-credit or credit-less-debit nets only for tie-outs between queries. Label a
variance by account type, never by sign: actual above budget on `Natural Net` is favourable on
income and unfavourable on expense. State the convention once, in the basis block.
## D1 — GL activity by branch, account, business account and period

```giql
DECLARE [LedgerID] SCHEMA [PX.Objects.GL.Ledger].LedgerCD AS 'Ledger ID' REQUIRED
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.GLTran].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.GLTran].FinPeriodID AS 'Period To' REQUIRED
DECLARE [Account1] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 1'
DECLARE [Account2] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 2'
DECLARE [Account3] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 3'
DECLARE [Account4] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 4'
DECLARE [Account5] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 5'
DECLARE [Account6] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 6'
DECLARE [Account7] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 7'
DECLARE [Account8] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 8'
DECLARE [AccountClass] SCHEMA [PX.Objects.GL.Account].AccountClassID AS 'Account Class'
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Branch].BranchCD AS 'Branch ID'
FROM [PX.Objects.GL.GLTran] As Tran
INNER JOIN [PX.Objects.GL.Ledger] As Ledger ON Tran.LedgerID = Ledger.LedgerID AND Ledger.BalanceType = {='A'}
INNER JOIN [PX.Objects.GL.Branch] As Branch ON Tran.BranchID = Branch.BranchID
INNER JOIN [PX.Objects.GL.Account] As Account ON Tran.AccountID = Account.AccountID
LEFT JOIN [PX.Objects.CR.BAccount] As BusinessAccount ON Tran.ReferenceID = BusinessAccount.BAccountID
WHERE Tran.Posted = 1 AND Ledger.LedgerCD = [LedgerID] AND Tran.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo] AND (Account.AccountCD = [Account1] OR Account.AccountCD = [Account2] OR Account.AccountCD = [Account3] OR Account.AccountCD = [Account4] OR Account.AccountCD = [Account5] OR Account.AccountCD = [Account6] OR Account.AccountCD = [Account7] OR Account.AccountCD = [Account8] OR ([Account1] IS NULL AND [Account2] IS NULL AND [Account3] IS NULL AND [Account4] IS NULL AND [Account5] IS NULL AND [Account6] IS NULL AND [Account7] IS NULL AND [Account8] IS NULL)) AND (Account.AccountClassID = [AccountClass] OR [AccountClass] IS NULL) AND (Branch.BranchCD = [BranchID] OR [BranchID] IS NULL)
GROUP BY Tran.FinPeriodID, Branch.BranchCD, Account.AccountCD, Tran.ReferenceID, BusinessAccount.Type
ORDER BY Account.AccountCD, Branch.BranchCD, BusinessAccount.AcctCD, Tran.FinPeriodID
SELECT Tran.FinPeriodID AS 'Period ID', Branch.BranchCD AS 'Branch ID', Account.AccountCD AS 'Account', BusinessAccount.AcctCD AS 'Business Account Code', BusinessAccount.AcctName AS 'Account Name', BusinessAccount.Type AS 'Business Account Type', SUM(Tran.DebitAmt) AS 'Debit Amount', SUM(Tran.CreditAmt) AS 'Credit Amount', SUM({=[Tran.DebitAmt]-[Tran.CreditAmt]}) SCHEMA [PX.Objects.GL.GLTran].DebitAmt AS 'Net Amount', COUNT(*) AS 'Line Count', SUM({=IIf([Account.Type]='I',[Tran.CreditAmt]-[Tran.DebitAmt],IIf([Account.Type]='L',[Tran.CreditAmt]-[Tran.DebitAmt],[Tran.DebitAmt]-[Tran.CreditAmt]))}) SCHEMA [PX.Objects.GL.GLTran].DebitAmt AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Ledger ID` | yes | `GL.Ledger.LedgerCD` |
| `Branch ID` | no | `GL.Branch.BranchCD` |
| `Period From` | yes | `GL.GLTran.FinPeriodID` |
| `Period To` | yes | `GL.GLTran.FinPeriodID` |
| `Account 1 … Account 8` | no | `GL.Account.AccountCD` |
| `Account Class` | no | `GL.Account.AccountClassID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
- `Ledger ID` is the actual ledger code from pull 3. `Branch ID` left out returns every branch on the
  ledger, one row per branch, the branch code in every row. `Period From` / `Period To` run from the
  first period of the earliest window to the last of the current one. Pass account codes in
  `Account 1` … `Account 8` (fewer is fine; pass each unused slot as `null`), an `Account Class` code, or both —
  a line must then match both; with neither, every account comes back, so always pass one.

### Reading it

- **One row per period × branch × account × business account.** Keep `Debit Amount` and
  `Credit Amount`: a business account whose invoices are routinely netted by credits is a pattern in
  itself.
- **`Business Account Type`:** `VE` vendor, `CU` customer, `VC` customer and vendor, `EP` employee,
  `PR` other business account, `CP` branch (usually intercompany). `EP` comes back even though the
  column's allowed values don't list it — don't treat it as invalid.
- **Null reference** means the line carried no business account: typically card transactions, bank
  feeds or manual journals. All business-account columns are null on those rows.
- **`Line Count`** separates one large document from many small ones, and shows a change in pattern
  (monthly to annual billing, one invoice to per-seat lines, more orders, more claims).
- **Period basis.** Grouped by `FinPeriodID`, so it ties to GL history. D7 filters by financial period
  too, so its lines sum to the cell; filtered by transaction date instead it won't tie exactly, which
  is expected.

### Sizing

Rows grow with accounts × active business accounts × periods, and with branches when `Branch ID` is
left out. Eight expense accounts over 30 periods returned 721 rows for one branch and 1,360 rows over
12 periods across five branches in testing; revenue or receivables accounts with many customers are
larger, so shorten the window or run fewer accounts per call. Always set `top` explicitly (start at
5,000) and check `truncated`. If the host stores large results to a file, parse it from there with a
script; if the result comes back inline, move it into a script without retyping amounts where
possible — the Step 3 tie-out to pull 5 is what proves the figures were carried correctly, so never
skip it. `Branch.AcctName` is a computed column and cannot be grouped on, so only the branch code is
returned; names, companies and the shared ledger and currency come from pull 3.

### Errors already met

| Error | Cause | What to do |
|---|---|---|
| `Failed to convert parameter value from a String to a Int32` | A parameter bound to an integer key (`GLTran.AccountID`, `LedgerID`, `BranchID`) received a code | Use this query, which binds to `AccountCD`, `LedgerCD`, `BranchCD`. If a rebuilt variant hits it, rebuild through `prepare_query`, quoting the error |
| Empty result | A parameter key that matches no label is dropped silently | Copy the keys from the table above |
| Business account table below pull 5 on one account while the other accounts tie | A posted line whose business account the query cannot read (soft-deleted, or restricted from the connector's user) is dropped by the `LEFT JOIN` to `CR.BAccount` instead of coming back with empty columns | Run D5. If it ties to pull 5, add a *Business account not visible* row for the difference. Seen on `gab-dev` 2026-10-06: 40000 Sales Revenue 2–20% short on every branch, every other account tied |
| `truncated: true` on a control pull | `top` lower than the row count; the tail of the ordered result is missing, so the last accounts in code order don't tie | Raise `top` or page with `skip`; a tie-out that fails only on the last accounts is this, not a data problem |

## D0 — Subaccount segment structure

```giql
FROM [PX.Objects.CS.Segment] As Segment
LEFT JOIN [PX.Objects.CS.SegmentValue] As SegmentValue ON Segment.DimensionID = SegmentValue.DimensionID AND Segment.SegmentID = SegmentValue.SegmentID
WHERE Segment.DimensionID = 'SUBACCOUNT'
ORDER BY Segment.SegmentID, SegmentValue.Value
SELECT Segment.SegmentID AS 'Segment ID', Segment.Descr AS 'Description', Segment.Length AS 'Length', Segment.Validate AS 'Validate', SegmentValue.Value AS 'Value', SegmentValue.Descr AS 'Value Description', SegmentValue.Active AS 'Active'
```

No parameters.
Run once per tenant and keep it for the run.

- **Splitting `SubCD`.** Segments are stored without separators. Take the segments in `Segment ID`
  order; each one's `Length` gives its characters inside `SubCD` (for example Department = characters
  1-2, Cost Center = characters 3-6). Check that the lengths add up to the `SubCD` length you see in
  pull 5. If they don't, the tenant uses a display mask or a different dimension: stop and ask.
- **Labels.** `Value Description` names each value. Values described as `<Uncategorized>`,
  `<Undefined>`, or all zeros are the **unassigned bucket**; treat them as such whatever their code.
- **`Validate` = false** means postings can carry values not in the list: show the raw code with
  "(not in list)".
- **Meaning is per tenant.** Confirm with the user what each segment represents before attributing a
  movement to its owner.

## D2 — GL activity by account, subaccount, business account and period

```giql
DECLARE [LedgerID] SCHEMA [PX.Objects.GL.Ledger].LedgerCD AS 'Ledger ID' REQUIRED
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Branch].BranchCD AS 'Branch ID'
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.GLTran].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.GLTran].FinPeriodID AS 'Period To' REQUIRED
DECLARE [Account1] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 1'
DECLARE [Account2] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 2'
DECLARE [Account3] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 3'
DECLARE [Account4] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 4'
DECLARE [Account5] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 5'
DECLARE [Account6] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 6'
DECLARE [Account7] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 7'
DECLARE [Account8] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 8'
DECLARE [AccountClass] SCHEMA [PX.Objects.GL.Account].AccountClassID AS 'Account Class'
FROM [PX.Objects.GL.GLTran] As Tran
INNER JOIN [PX.Objects.GL.Ledger] As Ledger ON Tran.LedgerID = Ledger.LedgerID AND Ledger.BalanceType = {='A'}
INNER JOIN [PX.Objects.GL.Branch] As Branch ON Tran.BranchID = Branch.BranchID
INNER JOIN [PX.Objects.GL.Account] As Account ON Tran.AccountID = Account.AccountID
INNER JOIN [PX.Objects.GL.Sub] As Sub ON Tran.SubID = Sub.SubID
LEFT JOIN [PX.Objects.CR.BAccount] As BusinessAccount ON Tran.ReferenceID = BusinessAccount.BAccountID
WHERE Tran.Posted = 1 AND Ledger.LedgerCD = [LedgerID] AND (Branch.BranchCD = [BranchID] OR [BranchID] IS NULL) AND Tran.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo] AND (Account.AccountCD = [Account1] OR Account.AccountCD = [Account2] OR Account.AccountCD = [Account3] OR Account.AccountCD = [Account4] OR Account.AccountCD = [Account5] OR Account.AccountCD = [Account6] OR Account.AccountCD = [Account7] OR Account.AccountCD = [Account8] OR ([Account1] IS NULL AND [Account2] IS NULL AND [Account3] IS NULL AND [Account4] IS NULL AND [Account5] IS NULL AND [Account6] IS NULL AND [Account7] IS NULL AND [Account8] IS NULL)) AND (Account.AccountClassID = [AccountClass] OR [AccountClass] IS NULL)
GROUP BY Tran.FinPeriodID, Account.AccountCD, Branch.BranchCD, Sub.SubCD, Tran.ReferenceID, BusinessAccount.Type
ORDER BY Account.AccountCD, Branch.BranchCD, Sub.SubCD, BusinessAccount.AcctCD, Tran.FinPeriodID
SELECT Tran.FinPeriodID AS 'Period ID', Account.AccountCD AS 'Account', Branch.BranchCD AS 'Branch ID', Sub.SubCD AS 'Subaccount', BusinessAccount.AcctCD AS 'Business Account Code', BusinessAccount.AcctName AS 'Account Name', BusinessAccount.Type AS 'Business Account Type', SUM(Tran.DebitAmt) AS 'Debit Amount', SUM(Tran.CreditAmt) AS 'Credit Amount', SUM({=[Tran.DebitAmt]-[Tran.CreditAmt]}) SCHEMA [PX.Objects.GL.GLTran].DebitAmt AS 'Net Amount', COUNT(*) AS 'Line Count', SUM({=IIf([Account.Type]='I',[Tran.CreditAmt]-[Tran.DebitAmt],IIf([Account.Type]='L',[Tran.CreditAmt]-[Tran.DebitAmt],[Tran.DebitAmt]-[Tran.CreditAmt]))}) SCHEMA [PX.Objects.GL.GLTran].DebitAmt AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Ledger ID` | yes | `GL.Ledger.LedgerCD` |
| `Branch ID` | no | `GL.Branch.BranchCD` |
| `Period From` | yes | `GL.GLTran.FinPeriodID` |
| `Period To` | yes | `GL.GLTran.FinPeriodID` |
| `Account 1 … Account 8` | no | `GL.Account.AccountCD` |
| `Account Class` | no | `GL.Account.AccountClassID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
Same parameters as D1, with the branch code in every row. It is the cross-tab: which business account
is coded to which segment value, and the business account table whenever a subaccount filter is set.
Larger than D1 (one row per subaccount × business account × period), so run it for the current
window, or the comparison window too when a business account's coding may have changed, not the whole
history. `Natural Net` summed per account must equal pull 5 for the same window.

## D5 — GL activity by branch, account and period, no business account

```giql
DECLARE [LedgerID] SCHEMA [PX.Objects.GL.Ledger].LedgerCD AS 'Ledger ID' REQUIRED
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.GLTran].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.GLTran].FinPeriodID AS 'Period To' REQUIRED
DECLARE [Account1] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 1'
DECLARE [Account2] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 2'
DECLARE [Account3] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 3'
DECLARE [Account4] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 4'
DECLARE [Account5] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 5'
DECLARE [Account6] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 6'
DECLARE [Account7] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 7'
DECLARE [Account8] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account 8'
DECLARE [AccountClass] SCHEMA [PX.Objects.GL.Account].AccountClassID AS 'Account Class'
FROM [PX.Objects.GL.GLTran] As Tran
INNER JOIN [PX.Objects.GL.Ledger] As Ledger ON Tran.LedgerID = Ledger.LedgerID AND Ledger.BalanceType = {='A'}
INNER JOIN [PX.Objects.GL.Branch] As Branch ON Tran.BranchID = Branch.BranchID
INNER JOIN [PX.Objects.GL.Account] As Account ON Tran.AccountID = Account.AccountID
WHERE Tran.Posted = 1 AND Ledger.LedgerCD = [LedgerID] AND Tran.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo] AND (Account.AccountClassID = [AccountClass] OR [AccountClass] IS NULL) AND ((Account.AccountCD = [Account1] OR Account.AccountCD = [Account2] OR Account.AccountCD = [Account3] OR Account.AccountCD = [Account4] OR Account.AccountCD = [Account5] OR Account.AccountCD = [Account6] OR Account.AccountCD = [Account7] OR Account.AccountCD = [Account8]) OR ([Account1] IS NULL AND [Account2] IS NULL AND [Account3] IS NULL AND [Account4] IS NULL AND [Account5] IS NULL AND [Account6] IS NULL AND [Account7] IS NULL AND [Account8] IS NULL))
GROUP BY Tran.FinPeriodID, Branch.BranchCD, Account.AccountCD
ORDER BY Account.AccountCD, Branch.BranchCD, Tran.FinPeriodID
SELECT Tran.FinPeriodID AS 'Period ID', Branch.BranchCD AS 'Branch ID', Account.AccountCD AS 'Account', SUM(Tran.DebitAmt) AS 'Debit Amount', SUM(Tran.CreditAmt) AS 'Credit Amount', SUM({=[Tran.DebitAmt]-[Tran.CreditAmt]}) SCHEMA [PX.Objects.GL.GLTran].DebitAmt AS 'Net Amount', COUNT(*) AS 'Line Count', SUM({=IIf([Account.Type]='I' OR [Account.Type]='L',[Tran.CreditAmt]-[Tran.DebitAmt],[Tran.DebitAmt]-[Tran.CreditAmt])}) SCHEMA [PX.Objects.GL.GLTran].DebitAmt AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Ledger ID` | yes | `GL.Ledger.LedgerCD` |
| `Period From` | yes | `GL.GLTran.FinPeriodID` |
| `Period To` | yes | `GL.GLTran.FinPeriodID` |
| `Account 1 … Account 8` | no | `GL.Account.AccountCD` |
| `Account Class` | no | `GL.Account.AccountClassID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
Same parameters as D1 without `Branch ID`. The same posted lines with no business-account join, so it
ties to GL history by construction: one row per period × branch × account. It is the source of the
branch table and the line-level control for D1.

- **Where D1 falls short of it** on an account, the difference is lines whose business account the
  query cannot read; report it as **Business account not visible**, never as a remainder.
- **Sizing** is small: branches × accounts × periods.

## D6 — GL activity by branch, account, subaccount and period, no business account

```giql
DECLARE [Ledger] SCHEMA [PX.Objects.GL.GLHistory].LedgerID AS 'Ledger ID' REQUIRED
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period To' REQUIRED
DECLARE [Account1] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 1'
DECLARE [Account2] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 2'
DECLARE [Account3] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 3'
DECLARE [Account4] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 4'
DECLARE [Account5] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 5'
DECLARE [Account6] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 6'
DECLARE [Account7] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 7'
DECLARE [Account8] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 8'
DECLARE [AccountClass] SCHEMA [PX.Objects.GL.Account].AccountClassID AS 'Account Class'
DECLARE [BranchID] SCHEMA [PX.Objects.GL.GLHistory].BranchID AS 'Branch ID'
FROM [PX.Objects.GL.GLHistory] As Hist
INNER JOIN [PX.Objects.GL.Account] As Account ON Hist.AccountID = Account.AccountID
WHERE Hist.LedgerID = [Ledger]
AND Hist.BalanceType = 'A'
AND Hist.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo]
AND (Hist.AccountID = [Account1] OR Hist.AccountID = [Account2] OR Hist.AccountID = [Account3] OR Hist.AccountID = [Account4] OR Hist.AccountID = [Account5] OR Hist.AccountID = [Account6] OR Hist.AccountID = [Account7] OR Hist.AccountID = [Account8] OR ([Account1] IS NULL AND [Account2] IS NULL AND [Account3] IS NULL AND [Account4] IS NULL AND [Account5] IS NULL AND [Account6] IS NULL AND [Account7] IS NULL AND [Account8] IS NULL))
AND (Account.AccountClassID = [AccountClass] OR [AccountClass] IS NULL)
AND (Hist.BranchID = [BranchID] OR [BranchID] IS NULL)
GROUP BY Hist.FinPeriodID, Hist.BranchID, Hist.AccountID, Hist.SubID
ORDER BY Hist.AccountID, Hist.BranchID, Hist.SubID, Hist.FinPeriodID
SELECT Hist.FinPeriodID AS 'Financial Period', Hist.BranchID AS 'Branch', Hist.AccountID AS 'Account', Hist.SubID AS 'Subaccount', SUM(Hist.FinPtdDebit) AS 'Fin. PTD Debit', SUM(Hist.FinPtdCredit) AS 'Fin. PTD Credit', SUM({=[Hist.FinPtdDebit]-[Hist.FinPtdCredit]}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdDebit AS 'Net Debit Less Credit', SUM({=IIf([Account.Type]='I',[Hist.FinPtdCredit]-[Hist.FinPtdDebit],IIf([Account.Type]='L',[Hist.FinPtdCredit]-[Hist.FinPtdDebit],[Hist.FinPtdDebit]-[Hist.FinPtdCredit]))}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdDebit AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Ledger ID` | yes | `GL.GLHistory.LedgerID` |
| `Period From` | yes | `GL.GLHistory.FinPeriodID` |
| `Period To` | yes | `GL.GLHistory.FinPeriodID` |
| `Account 1 … Account 8` | no | `GL.GLHistory.AccountID` |
| `Account Class` | no | `GL.Account.AccountClassID` |
| `Branch ID` | no | `GL.GLHistory.BranchID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
The all-branches version of pull 5 at subaccount grain: one row per period × branch × account ×
subaccount, read from GL history, so it ties to pull 5 by construction and has no business account
join. It is the source of the segment tables when no branch is named, and of the subaccount-filtered
branch and segment tables; split each `Subaccount` into its segments with D0, then total per segment,
summing the branches on the one ledger.

- **Tie-out:** per branch and account, `Natural Net` summed over a window equals pull 5 for that
  branch, and equals D5 for the same branch, account and window.
- **`Branch ID` is optional:** `null` returns every branch; a branch code makes it pull 5 at subaccount
  grain for several accounts in one call.
- **Sizing:** branches × accounts × subaccounts × periods; set `top` explicitly and check `truncated`.
  When only the window total per subaccount is needed, run D6b instead of paging through D6.

## D6b — GL activity by account and subaccount, summed over a window, one branch

```giql
DECLARE [Ledger] SCHEMA [PX.Objects.GL.GLHistory].LedgerID AS 'Ledger ID' REQUIRED
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period To' REQUIRED
DECLARE [Account1] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 1'
DECLARE [Account2] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 2'
DECLARE [Account3] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 3'
DECLARE [Account4] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 4'
DECLARE [Account5] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 5'
DECLARE [Account6] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 6'
DECLARE [Account7] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 7'
DECLARE [Account8] SCHEMA [PX.Objects.GL.GLHistory].AccountID AS 'Account 8'
DECLARE [AccountClass] SCHEMA [PX.Objects.GL.Account].AccountClassID AS 'Account Class'
DECLARE [BranchID] SCHEMA [PX.Objects.GL.GLHistory].BranchID AS 'Branch ID' REQUIRED
FROM [PX.Objects.GL.GLHistory] As Hist
INNER JOIN [PX.Objects.GL.Account] As Account ON Hist.AccountID = Account.AccountID
WHERE Hist.LedgerID = [Ledger]
AND Hist.BalanceType = 'A'
AND Hist.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo]
AND (Hist.AccountID = [Account1] OR Hist.AccountID = [Account2] OR Hist.AccountID = [Account3] OR Hist.AccountID = [Account4] OR Hist.AccountID = [Account5] OR Hist.AccountID = [Account6] OR Hist.AccountID = [Account7] OR Hist.AccountID = [Account8] OR ([Account1] IS NULL AND [Account2] IS NULL AND [Account3] IS NULL AND [Account4] IS NULL AND [Account5] IS NULL AND [Account6] IS NULL AND [Account7] IS NULL AND [Account8] IS NULL))
AND (Account.AccountClassID = [AccountClass] OR [AccountClass] IS NULL)
AND Hist.BranchID = [BranchID]
GROUP BY Hist.AccountID, Hist.SubID
ORDER BY Hist.AccountID, Hist.SubID
SELECT Hist.AccountID AS 'Account', Hist.SubID AS 'Subaccount', SUM(Hist.FinPtdDebit) AS 'Fin. PTD Debit', SUM(Hist.FinPtdCredit) AS 'Fin. PTD Credit', SUM({=[Hist.FinPtdDebit]-[Hist.FinPtdCredit]}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdDebit AS 'Net Debit Less Credit', SUM({=IIf([Account.Type]='I',[Hist.FinPtdCredit]-[Hist.FinPtdDebit],IIf([Account.Type]='L',[Hist.FinPtdCredit]-[Hist.FinPtdDebit],[Hist.FinPtdDebit]-[Hist.FinPtdCredit]))}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdDebit AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Ledger ID` | yes | `GL.GLHistory.LedgerID` |
| `Period From` | yes | `GL.GLHistory.FinPeriodID` |
| `Period To` | yes | `GL.GLHistory.FinPeriodID` |
| `Account 1 … Account 8` | no | `GL.GLHistory.AccountID` |
| `Account Class` | no | `GL.Account.AccountClassID` |
| `Branch ID` | yes | `GL.GLHistory.BranchID` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
D6 without the period and branch columns: one row per account × subaccount for one branch, summed over the
window. The source of a segment table for a window and of the margin-by-segment read
(`references/acumatica/segments.md`): run it once per branch and window with the revenue and cost accounts in the
same call. **Tie-out:** per account, the subaccounts sum to pull 5 for the same branch and window.

## D7 — Posted transaction lines, every filter optional

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
The document level: the lines behind a cell of any other table. Pass only the filters that narrow the
question: `Ledger ID` (always), `Branch ID`, `Account CD`, `Business Account`, `Subaccount CD` (exact
code) or `Subaccount Pattern` (a segment value), a period range, a date range, `Batch Number`.

- **From a cell to its lines:** pass the cell's own filters, for example branch, account, business
  account and a single period, and the sum of `Natural Net` over the returned lines must equal that
  cell. Run it for a cell that is a spike, a credit or a one-off, and say what the lines show: the
  document, the description, whether it is an allocation (a `GL` batch with source and destination
  lines).
- **Segment filter:** build `Subaccount Pattern` from the layout in D0, with the value at its position
  and `_` elsewhere. Several segments at once are one pattern.
- **Always bound it:** set `top`, at least one of account, period or date, and page with `skip`.

## D8 — Posted lines with their source-document keys (the router)

```giql
DECLARE [LedgerID] SCHEMA [PX.Objects.GL.Ledger].LedgerCD AS 'Ledger ID' REQUIRED
DECLARE [BranchID] SCHEMA [PX.Objects.GL.Branch].BranchCD AS 'Branch ID' REQUIRED
DECLARE [AccountCD] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account CD' REQUIRED
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.GLTran].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.GLTran].FinPeriodID AS 'Period To' REQUIRED
FROM [PX.Objects.GL.GLTran] As Tran
INNER JOIN [PX.Objects.GL.Ledger] As Ledger ON Tran.LedgerID = Ledger.LedgerID
INNER JOIN [PX.Objects.GL.Branch] As Branch ON Tran.BranchID = Branch.BranchID
INNER JOIN [PX.Objects.GL.Account] As Account ON Tran.AccountID = Account.AccountID
WHERE Ledger.LedgerCD = [LedgerID] AND Ledger.BalanceType = 'A' AND Branch.BranchCD = [BranchID] AND Account.AccountCD = [AccountCD] AND Tran.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo] AND Tran.Posted = 1
ORDER BY Tran.FinPeriodID, Tran.Module, Tran.RefNbr, Tran.LineNbr
SELECT Tran.FinPeriodID AS 'Period ID', Tran.TranDate AS 'Transaction Date', Tran.Module AS 'Module', Tran.BatchNbr AS 'Batch Number', Tran.LineNbr AS 'Line Nbr.', Tran.TranType AS 'Transaction Type', Tran.RefNbr AS 'Ref. Number', Tran.TranLineNbr AS 'Tran. Line Nbr.', Tran.InventoryID AS 'Inventory ID', Tran.Qty AS 'Quantity', Tran.UOM AS 'UOM', Tran.ReferenceID AS 'Customer/Vendor', Tran.TranDesc AS 'Transaction Description', Tran.DebitAmt AS 'Debit Amount', Tran.CreditAmt AS 'Credit Amount', {=IIf([Account.Type]='I',[Tran.CreditAmt]-[Tran.DebitAmt],IIf([Account.Type]='L',[Tran.CreditAmt]-[Tran.DebitAmt],[Tran.DebitAmt]-[Tran.CreditAmt]))} SCHEMA [PX.Objects.GL.GLTran].DebitAmt AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Ledger ID` | yes | `GL.Ledger.LedgerCD` |
| `Branch ID` | yes | `GL.Branch.BranchCD` |
| `Account CD` | yes | `GL.Account.AccountCD` |
| `Period From` | yes | `GL.GLTran.FinPeriodID` |
| `Period To` | yes | `GL.GLTran.FinPeriodID` |
One row per posted line on one account, branch and window, carrying `Module`, `Transaction Type`,
`Ref. Number` and `Tran. Line Nbr.` — the keys of the document that posted it. Run it for the cell under
investigation, then open the document with the pull its module routes to:

| `Module` | Pull | `Document Type` to pass | `Reference Number` |
|---|---|---|---|
| `AR` | D9 | D8's `Transaction Type` (`INV`, `CRM`, `DRM`, `CSL`, `PMT` …) | `Ref. Number` |
| `AP` | D10 | D8's `Transaction Type` (`INV`, `ACR`, `ADR`, `CHK` …) | `Ref. Number` |
| `IN` | D11 | the IN register's type, **not** D8's transaction type: an `INV` line (an issue for a sales shipment) belongs to `I` (Issue), seen on `gab-dev`; for other transaction types try the likely register (`RCP` → `R`, `ADJ` → `A`, `TRX` → `T`), unconfirmed, and read the `Document Type` column D11 returns — an empty result means the type was wrong | `Ref. Number` |
| `CA` | D11-CA | D8's `Transaction Type` (`CAE`) | `Ref. Number` |
| `GL` | none — an allocation or a manual journal; the batch (D7 with `Batch Number`) is the evidence | — | — |
| `FA`, `DR`, `PM`, `CM`, `PR` | none yet — report the line with its batch and description | — | — |

- **Always pass the pair.** A reference number repeats across document types; one without its type can open
  the wrong document.
- **One source line can post several GL lines** (an inventory line issued from several cost layers): sum D8 by
  `Ref. Number` and `Tran. Line Nbr.` before comparing with the document.
- **Summary posting.** Where a module posts in summary, one GL line covers many documents and `Ref. Number` is
  blank or names only the batch. Report that line as summary-posted; do not guess which documents it covers.
- **Cost of goods sold carries no business account.** `IN` lines have no `Customer/Vendor`, so the customer
  axis for a cost account is D12, not D1.
- **Tie-out:** D8's lines sum to D5 for the same ledger, branch, account and window, and their count equals
  D5's `Line Count`.

## D9 — One AR document: lines and applications

```giql
DECLARE [DocumentType] SCHEMA [PX.Objects.AR.ARInvoice].DocType AS 'Document Type' REQUIRED
DECLARE [ReferenceNumber] SCHEMA [PX.Objects.AR.ARInvoice].RefNbr AS 'Reference Number' REQUIRED
FROM [PX.Objects.AR.ARInvoice] As Document
INNER JOIN [PX.Objects.AR.ARTran] As Line ON Line.TranType = Document.DocType AND Line.RefNbr = Document.RefNbr
INNER JOIN [PX.Objects.GL.Account] As Account ON Line.AccountID = Account.AccountID
WHERE Document.DocType = [DocumentType] AND Document.RefNbr = [ReferenceNumber]
ORDER BY Line.LineNbr
SELECT Document.DocType AS 'Document Type', Document.RefNbr AS 'Reference Nbr.', Document.DocDate AS 'Date', Document.FinPeriodID AS 'Post Period', Line.CustomerID AS 'Customer', Document.InvoiceNbr AS 'Customer Order Nbr.', Line.LineNbr AS 'Line Nbr.', Line.InventoryID AS 'Inventory ID', Line.TranDesc AS 'Transaction Descr.', Line.Qty AS 'Quantity', Line.UOM AS 'UOM', Line.UnitPrice AS 'Unit Price', Line.DiscPct AS 'Discount Percent', Line.DiscAmt AS 'Discount Amount', Line.TranAmt AS 'Amount', Line.SOOrderType AS 'Order Type', Line.SOOrderNbr AS 'Order Nbr.', Line.SOShipmentType AS 'Shipment Type', Line.SOShipmentNbr AS 'Shipment Nbr.', Account.AccountCD AS 'Account', Line.SubID AS 'Subaccount', Line.DrCr AS 'Dr/Cr', {=IIf([Account.Type]='I',IIf([Line.DrCr]='C',[Line.TranAmt],-[Line.TranAmt]),IIf([Account.Type]='L',IIf([Line.DrCr]='C',[Line.TranAmt],-[Line.TranAmt]),IIf([Line.DrCr]='D',[Line.TranAmt],-[Line.TranAmt])))} SCHEMA [PX.Objects.AR.ARTran].TranAmt AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Document Type` | yes | `AR.ARInvoice.DocType` |
| `Reference Number` | yes | `AR.ARInvoice.RefNbr` |
The lines: item, quantity, unit price, discount, the sales order and shipment, the account and the line's
`Natural Net` (taken from the line's posting side, so a document discount reads negative on its income
account). **An SO-originated invoice is an ordinary AR document**: the sales order and the shipment are on its
lines, and the inventory issue behind a line is D12 or D11. **Tie-out:** the line with D8's
`Tran. Line Nbr.` has D8's `Natural Net` on the same account.

```giql
DECLARE [DocumentType] SCHEMA [PX.Objects.AR.ARAdjust].AdjdDocType AS 'Document Type' REQUIRED
DECLARE [ReferenceNumber] SCHEMA [PX.Objects.AR.ARAdjust].AdjdRefNbr AS 'Reference Number' REQUIRED
FROM [PX.Objects.AR.ARAdjust] As Adjust
WHERE Adjust.AdjdDocType = [DocumentType] AND Adjust.AdjdRefNbr = [ReferenceNumber]
ORDER BY Adjust.AdjgDocDate
SELECT Adjust.AdjgDocType AS 'Adjusting Document Type', Adjust.AdjgRefNbr AS 'Adjusting Reference Number', Adjust.AdjgDocDate AS 'Adjustment Date', Adjust.AdjgFinPeriodID AS 'Application Period', Adjust.AdjAmt AS 'Amount Applied', Adjust.AdjDiscAmt AS 'Cash Discount Taken', Adjust.AdjWOAmt AS 'Write-Off Amount', Adjust.Released AS 'Released', Adjust.Voided AS 'Voided'
```

| Key | Required | Binds to |
|---|---|---|
| `Document Type` | yes | `AR.ARAdjust.AdjdDocType` |
| `Reference Number` | yes | `AR.ARAdjust.AdjdRefNbr` |
The applications: payments, credit memos and write-offs against the document, with the cash discount taken.
No `Natural Net` — an application is not a P&L amount; the discount posts through the payment's own GL lines.

## D10 — One AP document: lines and applications

```giql
DECLARE [DocumentType] SCHEMA [PX.Objects.AP.APInvoice].DocType AS 'Document Type' REQUIRED
DECLARE [ReferenceNumber] SCHEMA [PX.Objects.AP.APInvoice].RefNbr AS 'Reference Number' REQUIRED
FROM [PX.Objects.AP.APInvoice] As Document
INNER JOIN [PX.Objects.AP.APTran] As Line ON Line.TranType = Document.DocType AND Line.RefNbr = Document.RefNbr
INNER JOIN [PX.Objects.GL.Account] As Account ON Line.AccountID = Account.AccountID
WHERE Document.DocType = [DocumentType] AND Document.RefNbr = [ReferenceNumber]
ORDER BY Line.LineNbr
SELECT Document.DocType AS 'Document Type', Document.RefNbr AS 'Reference Nbr.', Document.DocDate AS 'Date', Document.FinPeriodID AS 'Post Period', Line.VendorID AS 'Vendor', Document.InvoiceNbr AS 'Vendor Ref.', Line.LineNbr AS 'Line Nbr.', Line.InventoryID AS 'Inventory ID', Line.TranDesc AS 'Transaction Descr.', Line.Qty AS 'Quantity', Line.UOM AS 'UOM', Line.UnitCost AS 'Unit Cost', Line.DiscPct AS 'Discount Percent', Line.DiscAmt AS 'Discount Amount', Line.TranAmt AS 'Amount', Line.POOrderType AS 'PO Type', Line.PONbr AS 'PO Number', Line.ReceiptType AS 'PO Receipt Type', Line.ReceiptNbr AS 'PO Receipt Nbr.', Account.AccountCD AS 'Account', Line.SubID AS 'Subaccount', Line.DrCr AS 'Dr/Cr', {=IIf([Account.Type]='I',IIf([Line.DrCr]='C',[Line.TranAmt],-[Line.TranAmt]),IIf([Account.Type]='L',IIf([Line.DrCr]='C',[Line.TranAmt],-[Line.TranAmt]),IIf([Line.DrCr]='D',[Line.TranAmt],-[Line.TranAmt])))} SCHEMA [PX.Objects.AP.APTran].TranAmt AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Document Type` | yes | `AP.APInvoice.DocType` |
| `Reference Number` | yes | `AP.APInvoice.RefNbr` |
The lines: item, quantity, unit cost, discount, the purchase order and receipt, the account and `Natural Net`
from the posting side. A stock line on a bill usually posts to the purchase accrual, not to cost; read
`Account` before treating it as an expense. **Tie-out:** as D9.

```giql
DECLARE [DocumentType] SCHEMA [PX.Objects.AP.APAdjust].AdjdDocType AS 'Document Type' REQUIRED
DECLARE [ReferenceNumber] SCHEMA [PX.Objects.AP.APAdjust].AdjdRefNbr AS 'Reference Number' REQUIRED
FROM [PX.Objects.AP.APAdjust] As Adjust
WHERE Adjust.AdjdDocType = [DocumentType] AND Adjust.AdjdRefNbr = [ReferenceNumber]
ORDER BY Adjust.AdjgDocDate
SELECT Adjust.AdjgDocType AS 'Adjusting Document Type', Adjust.AdjgRefNbr AS 'Adjusting Reference Number', Adjust.AdjgDocDate AS 'Transaction Date', Adjust.AdjgFinPeriodID AS 'Application Period', Adjust.AdjAmt AS 'Amount', Adjust.AdjDiscAmt AS 'Cash Discount Taken', Adjust.AdjWhTaxAmt AS 'Withholding Tax Amount', Adjust.Released AS 'Released', Adjust.Voided AS 'Voided'
```

| Key | Required | Binds to |
|---|---|---|
| `Document Type` | yes | `AP.APAdjust.AdjdDocType` |
| `Reference Number` | yes | `AP.APAdjust.AdjdRefNbr` |
The applications: the checks and debit adjustments that paid the bill, with the discount taken.

## D11 — One inventory document's lines

```giql
DECLARE [DocumentType] SCHEMA [PX.Objects.IN.INTran].DocType AS 'Document Type' REQUIRED
DECLARE [ReferenceNumber] SCHEMA [PX.Objects.IN.INTran].RefNbr AS 'Reference Number' REQUIRED
FROM [PX.Objects.IN.INTran] As Tran
INNER JOIN [PX.Objects.IN.INRegister] As Register ON Tran.DocType = Register.DocType AND Tran.RefNbr = Register.RefNbr
LEFT JOIN [PX.Objects.IN.InventoryItem] As Item ON Tran.InventoryID = Item.InventoryID
LEFT JOIN [PX.Objects.IN.INSite] As Warehouse ON Tran.SiteID = Warehouse.SiteID
LEFT JOIN [PX.Objects.GL.Account] As InventoryAccount ON Tran.InvtAcctID = InventoryAccount.AccountID
LEFT JOIN [PX.Objects.GL.Account] As COGSAccount ON Tran.COGSAcctID = COGSAccount.AccountID
WHERE Tran.DocType = [DocumentType] AND Tran.RefNbr = [ReferenceNumber]
ORDER BY Tran.LineNbr
SELECT Tran.DocType AS 'Document Type', Tran.TranType AS 'Tran. Type', Tran.RefNbr AS 'Reference Nbr.', Register.TranDate AS 'Date', Register.FinPeriodID AS 'Post Period', Tran.LineNbr AS 'Line Number', Item.InventoryCD AS 'Inventory ID', Item.Descr AS 'Description', Warehouse.SiteCD AS 'Warehouse ID', Tran.Qty AS 'Quantity', Tran.UOM AS 'UOM', Tran.UnitCost AS 'Unit Cost', Tran.TranCost AS 'Ext. Cost', Tran.SOShipmentType AS 'SO Shipment Type', Tran.SOShipmentNbr AS 'SO Shipment Nbr.', Tran.SOOrderType AS 'SO Order Type', Tran.SOOrderNbr AS 'SO Order Nbr.', Tran.ARDocType AS 'AR Document Type', Tran.ARRefNbr AS 'AR Reference Nbr.', Tran.ARLineNbr AS 'AR Line Nbr.', InventoryAccount.AccountCD AS 'Inventory Account', COGSAccount.AccountCD AS 'COGS Account', {=IIf([Tran.InvtMult]=-1,[Tran.TranCost],-[Tran.TranCost])} SCHEMA [PX.Objects.IN.INTran].TranCost AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Document Type` | yes | `IN.INTran.DocType` |
| `Reference Number` | yes | `IN.INTran.RefNbr` |
Item, warehouse, quantity, unit and extended cost, the shipment, the sales order, the AR invoice line it was
shipped for, and the inventory and COGS accounts. `Natural Net` is the line's effect on cost of goods sold
(positive for an issue). **Tie-out:** a line's `Natural Net` equals D8's lines on the COGS account with the
same `Ref. Number` and `Tran. Line Nbr.`, summed.

## D11-CA — One cash transaction's splits

```giql
DECLARE [DocumentType] SCHEMA [PX.Objects.CA.CASplit].AdjTranType AS 'Document Type' REQUIRED
DECLARE [ReferenceNumber] SCHEMA [PX.Objects.CA.CASplit].AdjRefNbr AS 'Reference Number' REQUIRED
FROM [PX.Objects.CA.CASplit] As Split
INNER JOIN [PX.Objects.CA.CAAdj] As Adj ON Split.AdjTranType = Adj.AdjTranType AND Split.AdjRefNbr = Adj.AdjRefNbr
INNER JOIN [PX.Objects.GL.Account] As Account ON Split.AccountID = Account.AccountID
WHERE Split.AdjTranType = [DocumentType] AND Split.AdjRefNbr = [ReferenceNumber]
ORDER BY Split.LineNbr
SELECT Split.AdjTranType AS 'Transaction Type', Split.AdjRefNbr AS 'Reference Nbr.', Adj.TranDate AS 'Tran. Date', Adj.FinPeriodID AS 'Fin. Period', Adj.CashAccountID AS 'Cash Account', Adj.EntryTypeID AS 'Entry Type', Adj.TranDesc AS 'Description', Adj.ExtRefNbr AS 'Document Ref.', Adj.DrCr AS 'Disbursement/Receipt', Split.LineNbr AS 'Line Nbr.', Split.InventoryID AS 'Item ID', Split.TranDesc AS 'Description', Split.Qty AS 'Quantity', Split.UnitPrice AS 'Price', Split.TranAmt AS 'Tran. Amount', Split.AccountID AS 'Offset Account', Split.SubID AS 'Offset Subaccount', {=IIf([Account.Type]='I',IIf([Adj.DrCr]='D',[Split.TranAmt],-[Split.TranAmt]),IIf([Account.Type]='L',IIf([Adj.DrCr]='D',[Split.TranAmt],-[Split.TranAmt]),IIf([Adj.DrCr]='C',[Split.TranAmt],-[Split.TranAmt])))} SCHEMA [PX.Objects.CA.CASplit].TranAmt AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Document Type` | yes | `CA.CASplit.AdjTranType` |
| `Reference Number` | yes | `CA.CASplit.AdjRefNbr` |
The bank or cash entry behind a `CA` line: cash account, entry type, external reference and each split's
offset account and amount. A correction can be a negative amount rather than a flipped side; `Natural Net`
follows it. **Tie-out:** a split's `Natural Net` equals D8's line on its offset account.

## D12 — AR invoice lines matched to their inventory issues

```giql
DECLARE [BranchID] SCHEMA [PX.Objects.AR.ARTran].BranchID AS 'Branch ID' REQUIRED
DECLARE [PeriodFrom] SCHEMA [PX.Objects.AR.ARTran].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.AR.ARTran].FinPeriodID AS 'Period To' REQUIRED
DECLARE [ReferenceNumber] SCHEMA [PX.Objects.AR.ARTran].RefNbr AS 'Reference Number'
FROM [PX.Objects.AR.ARTran] As ARTran
LEFT JOIN [PX.Objects.IN.INTran] As INTran ON INTran.ARDocType = ARTran.TranType AND INTran.ARRefNbr = ARTran.RefNbr AND INTran.ARLineNbr = ARTran.LineNbr
WHERE ARTran.BranchID = [BranchID] AND ARTran.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo] AND (ARTran.RefNbr = [ReferenceNumber] OR [ReferenceNumber] IS NULL) AND ARTran.InventoryID IS NOT NULL
GROUP BY ARTran.TranType, ARTran.RefNbr, ARTran.LineNbr, ARTran.FinPeriodID, ARTran.CustomerID, ARTran.InventoryID, ARTran.TranDesc, ARTran.Qty, ARTran.TranAmt, ARTran.DrCr
ORDER BY ARTran.FinPeriodID, ARTran.RefNbr, ARTran.LineNbr
SELECT ARTran.TranType AS 'AR Document Type', ARTran.RefNbr AS 'Reference Number', ARTran.LineNbr AS 'Line Nbr.', ARTran.FinPeriodID AS 'Financial Period', ARTran.CustomerID AS 'Customer', ARTran.InventoryID AS 'Inventory ID', ARTran.TranDesc AS 'Transaction Descr.', ARTran.Qty AS 'Qty Invoiced', SUM(INTran.Qty) AS 'Qty Issued', COUNT(INTran.LineNbr) AS 'Issue Lines', {=IIf([ARTran.DrCr]='C',[ARTran.TranAmt],-[ARTran.TranAmt])} SCHEMA [PX.Objects.AR.ARTran].TranAmt AS 'Revenue', SUM({=IIf([INTran.InvtMult]=-1,[INTran.TranCost],-[INTran.TranCost])}) SCHEMA [PX.Objects.IN.INTran].TranCost AS 'Issued Cost', {=[ARTran.Qty]-IsNull(SUM([INTran.Qty]),0)} SCHEMA [PX.Objects.AR.ARTran].Qty AS 'Qty Difference', {=IIf([ARTran.DrCr]='C',[ARTran.TranAmt],-[ARTran.TranAmt])-IsNull(SUM(IIf([INTran.InvtMult]=-1,[INTran.TranCost],-[INTran.TranCost])),0)} SCHEMA [PX.Objects.AR.ARTran].TranAmt AS 'Natural Net'
```

| Key | Required | Binds to |
|---|---|---|
| `Branch ID` | yes | `AR.ARTran.BranchID` |
| `Period From` | yes | `AR.ARTran.FinPeriodID` |
| `Period To` | yes | `AR.ARTran.FinPeriodID` |
| `Reference Number` | no | `AR.ARTran.RefNbr` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
One row per AR item line in a branch and window, with the inventory issued for it: `Qty Invoiced`,
`Qty Issued`, `Issue Lines`, `Revenue`, `Issued Cost` and the line's margin as `Natural Net`.

- **The customer axis for cost.** Sum `Issued Cost` by `Customer` for the cost-of-goods-sold drill-down by
  customer; D1 returns nothing there because inventory lines carry no business account.
- **`Qty Difference` on a stock item is the finding**: invoiced and issued quantities disagree, so cost (or
  revenue) is understated. Non-stock items are never issued — `Issue Lines` 0 — and are not a finding.
- **A negative `Issued Cost`** means the linked inventory line added stock; open it with D11 and say what it
  is rather than reading it as a margin.
- **Tie-out:** `Issued Cost` summed equals D8's `IN` lines on the COGS account that carry an AR link; the rest
  of the COGS account (unlinked issues, adjustments, standard-cost, landed-cost and AP discount lines) is its
  own row, never spread across customers. `Revenue` summed equals D8's `AR` item lines on the revenue
  account.

## D4 — Budget by account, subaccount and period

The detail level of the analyzer's pull 6 file. Run it for the current window only.

```giql
DECLARE [BranchID] SCHEMA [PX.Objects.GL.GLHistory].BranchID AS 'Branch ID' REQUIRED
DECLARE [PeriodFrom] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period From' REQUIRED
DECLARE [PeriodTo] SCHEMA [PX.Objects.GL.GLHistory].FinPeriodID AS 'Period To' REQUIRED
DECLARE [AccountCD] SCHEMA [PX.Objects.GL.Account].AccountCD AS 'Account CD'
DECLARE [LedgerID] SCHEMA [PX.Objects.GL.GLHistory].LedgerID AS 'Ledger ID'
DECLARE [AccountClass] SCHEMA [PX.Objects.GL.Account].AccountClassID AS 'Account Class'
FROM [PX.Objects.GL.GLHistory] As History
INNER JOIN [PX.Objects.GL.Account] As Account ON History.AccountID = Account.AccountID
INNER JOIN [PX.Objects.GL.Sub] As Sub ON History.SubID = Sub.SubID
WHERE History.BranchID = [BranchID] AND History.FinPeriodID BETWEEN [PeriodFrom] AND [PeriodTo] AND History.BalanceType = 'B' AND (Account.AccountCD = [AccountCD] OR [AccountCD] IS NULL) AND (History.LedgerID = [LedgerID] OR [LedgerID] IS NULL) AND (Account.AccountClassID = [AccountClass] OR [AccountClass] IS NULL)
GROUP BY Account.AccountCD, Account.Description, Account.Type, Sub.SubCD, History.FinPeriodID
ORDER BY Account.AccountCD, Sub.SubCD, History.FinPeriodID
SELECT Account.AccountCD AS 'Account', Account.Description AS 'Description', Account.Type AS 'Account Type', Sub.SubCD AS 'Subaccount', History.FinPeriodID AS 'Financial Period', SUM({=[History.FinPtdCredit]-[History.FinPtdDebit]}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdCredit AS 'Budgeted Amount', SUM({=IIf([Account.Type]='I',[History.FinPtdCredit]-[History.FinPtdDebit],IIf([Account.Type]='L',[History.FinPtdCredit]-[History.FinPtdDebit],[History.FinPtdDebit]-[History.FinPtdCredit]))}) SCHEMA [PX.Objects.GL.GLHistory].FinPtdCredit AS 'Natural Net'
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
- `Ledger ID` is the **budget** ledger's code (the scope's `budget:<name>`), from the user, not the
  actual ledger. Supply it wherever the tenant keeps more than one budget ledger, or they are summed
  invisibly. `Account CD` or `Account Class` narrow it; omit both for the whole chart and filter
  client-side.
- **Zero rows means no budget is loaded** for that branch, ledger and window, not a budget of zero.
- **A line with the same amount every period was spread evenly at load**; compare cumulatively there
  and say so.
- **Tie-out.** Summed per account and period, D4 must equal pull 6 for the same scope. They read the
  same table, so the check catches a truncated result or a different ledger, nothing else.

## Pulls shared with the variance analyzer

- **Chart of accounts classification** (`gl-chart-of-accounts-classification`, no parameters): every
  account with its account class code and description. Resolve a class the user named, by code or
  description, to its code for the `Account Class` parameter of D1 to D7, and list the accounts in it
  for the basis block. Pull 5 has no class filter: to tie a class to pull 5, run it per account in the
  class (or without `Account CD` and keep the class's accounts); D5 and D6 filtered by the same class
  are the control that needs no mapping.
- **Pull 1** — period calendar: confirms every period's status. Label open-period figures preliminary.
- **Pull 3** — branches, companies and ledgers: branch names for the branch axis, and the ledger and
  currency each branch posts to.
- **Pull 5** — GL actuals by account, subaccount and period for one branch: the control total for the
  Step 3 tie-out, per branch and window, and the segment source when a branch is named (D6 is its
  all-branches version).
- **Pull 6** — budget by account and period: the account-level budget per branch, and the control for
  D4.
- **Pull 7** — is D7: the same query serves the analyzer and the drill-down.

The same queries the variance analyzer runs, so the drill-down ties to the same figures:

### Chart of accounts classification

```giql
FROM [PX.Objects.GL.Account] As Account
LEFT JOIN [PX.Objects.GL.AccountClass] As AccountClass ON Account.AccountClassID = AccountClass.AccountClassID
ORDER BY Account.Type, Account.AccountCD
SELECT Account.AccountCD AS 'Account', Account.Description AS 'Description', Account.Type AS 'Type', Account.AccountClassID AS 'Account Class', AccountClass.Descr AS 'Account Class Description', Account.Active AS 'Active', Account.ControlAccountModule AS 'Control Account Module', Account.IsCashAccount AS 'Cash Account', Account.CuryID AS 'Currency'
```

No parameters.
### Pull 1 — period calendar

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
### Pull 3 — branches, companies and ledgers

```giql
FROM [PX.Objects.GL.Branch] As Branch
INNER JOIN [PX.Objects.GL.DAC.Organization] As Organization ON Branch.OrganizationID = Organization.OrganizationID
INNER JOIN [PX.Objects.GL.Ledger] As Ledger ON Branch.LedgerID = Ledger.LedgerID
ORDER BY Organization.OrganizationCD, Branch.BranchCD
SELECT Branch.BranchCD AS 'Branch ID', Branch.AcctName AS 'Branch Name', Organization.OrganizationCD AS 'Company ID', Organization.OrganizationName AS 'Company Name', Ledger.LedgerCD AS 'Ledger ID', Branch.BaseCuryID AS 'Base Currency ID', Branch.Active AS 'Active'
```

No parameters.
### Pull 5 — GL actuals by account, subaccount and period

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
### Pull 6 — budget by account and period

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
## Where these come from

Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-09 from
50f3928. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
