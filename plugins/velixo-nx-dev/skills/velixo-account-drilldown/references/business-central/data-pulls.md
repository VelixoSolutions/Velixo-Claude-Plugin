# Data pulls

Run these through the Velixo connector's `execute_query` as written, with your
values in the `parameters` map. The query text goes in the `giql` argument whatever its fence says: a
` ```vql ` block goes there too, since `execute_query` has no `vql` argument. Do not call `prepare_query` for them and do not edit the text. If one
fails, report the error and stop rather than reformulating it. Parameter keys are the display labels
(the text after `AS` in each `DECLARE`); a key that matches no label is dropped silently and the
query returns nothing, which looks exactly like "no data". Pass every declared key, an unused optional
one as `null`: a key left out is not always treated as null. Always set `top` — start at 250 and
raise it if the result comes back flagged `truncated`.

**Dates are `YYYY-MM-DD`; every GL figure is read by posting date over a window.** The entry pulls
(D1, D2, D3) read at most 10,000 G/L entries a call: narrow the window when one fails or comes back
truncated.

| Alias | Query | Grain | Used for |
|---|---|---|---|
| P1 | `gl-accounting-periods` | accounting period | whether a month is an accounting period |
| G0 | `gl-setup` | one row | local currency, the global dimensions' codes |
| T1 | `gl-account-net-change-for-window` | one posting account, one window | the control every table ties to; the category for `Natural Net` |
| D1 | `gl-activity-by-source` | source type × source number | the business-account axis |
| D2 | `gl-activity-by-dimension` | the account's total (`@1`), and dimension value (`@2`) | a dimension axis |
| D3 | `gl-entries` | G/L entry | the entries behind a cell |
| BL | `gl-budget-names` | G/L budget | the budget names, when the user does not know one |
| BU | `gl-budget-by-account-and-date` | account × budget date | a budget column, when asked |

## P1 — accounting periods

```vql
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
FROM [Accounting Period] AS Period
WHERE Period.[Starting Date] BETWEEN [DateFrom] AND [DateTo]
ORDER BY Period.[Starting Date]
SELECT Period.[Starting Date] AS 'Starting Date', Period.Name AS 'Name', Period.[New Fiscal Year] AS 'New Fiscal Year', Period.Closed AS 'Closed', Period.[Date Locked] AS 'Date Locked'
```

| Key | Required | Binds to |
|---|---|---|
| `Date From` | yes | `Accounting Period.Starting Date` |
| `Date To` | yes | `Accounting Period.Starting Date` |
## G0 — General Ledger Setup

```vql
FROM [General Ledger Setup] AS Setup
SELECT Setup.[LCY Code] AS 'LCY Code', Setup.[Global Dimension 1 Code] AS 'Global Dimension 1 Code', Setup.[Global Dimension 2 Code] AS 'Global Dimension 2 Code', Setup.[Shortcut Dimension 3 Code] AS 'Shortcut Dimension 3 Code', Setup.[Shortcut Dimension 4 Code] AS 'Shortcut Dimension 4 Code', Setup.[Shortcut Dimension 5 Code] AS 'Shortcut Dimension 5 Code', Setup.[Shortcut Dimension 6 Code] AS 'Shortcut Dimension 6 Code', Setup.[Shortcut Dimension 7 Code] AS 'Shortcut Dimension 7 Code', Setup.[Shortcut Dimension 8 Code] AS 'Shortcut Dimension 8 Code', Setup.[Allow Posting From] AS 'Allow Posting From', Setup.[Allow Posting To] AS 'Allow Posting To'
```

## T1 — one account's net change in a window

```vql
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
DECLARE [AccountNumber] TYPE string AS 'Account Number' REQUIRED
FROM [G/L Account] AS Account
WHERE Account.[Account Type] = 'Posting' AND Account.[Date Filter] BETWEEN [DateFrom] AND [DateTo] AND Account.[No.] = [AccountNumber]
ORDER BY Account.[No.]
SELECT Account.[No.] AS 'Account Number', Account.Name AS 'Name', Account.[Income/Balance] AS 'Income Statement or Balance Sheet', Account.[Account Category] AS 'Account Category', Account.[Net Change] AS 'Net Change', Account.[Debit Amount] AS 'Debit Amount', Account.[Credit Amount] AS 'Credit Amount'
```

| Key | Required | Binds to |
|---|---|---|
| `Date From` | yes | `G/L Account.Date Filter` |
| `Date To` | yes | `G/L Account.Date Filter` |
| `Account Number` | yes | `G/L Account.No.` |
One call per account and window. No row means the number is not a posting account: ask, and never
pull the whole chart to look for it.

## D1 — by business account

```vql
DECLARE [AccountNumber] TYPE string AS 'Account Number' REQUIRED
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
FROM [G/L Entry] AS Entry
WHERE Entry.[G/L Account No.] = [AccountNumber] AND Entry.[Posting Date] BETWEEN [DateFrom] AND [DateTo]
GROUP BY Entry.[Source Type], Entry.[Source No.]
ORDER BY SUM(Entry.Amount) DESC
SELECT Entry.[Source Type] AS 'Source Type', Entry.[Source No.] AS 'Source Number', SUM(Entry.[Debit Amount]) AS 'Debit Amount Total', SUM(Entry.[Credit Amount]) AS 'Credit Amount Total', SUM(Entry.Amount) AS 'Net Amount Total', COUNT(*) AS 'Number of Entries'
```

| Key | Required | Binds to |
|---|---|---|
| `Account Number` | yes | `G/L Entry.G/L Account No.` |
| `Date From` | yes | `G/L Entry.Posting Date` |
| `Date To` | yes | `G/L Entry.Posting Date` |
`ORDER BY` sorts by the signed net: re-sort on `Natural Net`, largest first.

## D2 — by the value of one dimension

Run both levels with the same account and window. Level 1 less the sum of level 2's rows, field by
field, is the **no value** row: entries whose dimension set holds nothing for this dimension.

```vql
DECLARE [AccountNumber] TYPE string AS 'Account Number' REQUIRED
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
FROM [G/L Entry] AS Entry
WHERE Entry.[G/L Account No.] = [AccountNumber] AND Entry.[Posting Date] BETWEEN [DateFrom] AND [DateTo]
SELECT SUM(Entry.[Debit Amount]) AS 'Debit Amount Total', SUM(Entry.[Credit Amount]) AS 'Credit Amount Total', SUM(Entry.Amount) AS 'Net Amount Total', COUNT(*) AS 'Number of Entries'
```

| Key | Required | Binds to |
|---|---|---|
| `Account Number` | yes | `G/L Entry.G/L Account No.` |
| `Date From` | yes | `G/L Entry.Posting Date` |
| `Date To` | yes | `G/L Entry.Posting Date` |
```vql
DECLARE [AccountNumber] TYPE string AS 'Account Number' REQUIRED
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
DECLARE [DimensionCode] TYPE string AS 'Dimension Code' REQUIRED
FROM [G/L Entry] AS Entry
INNER JOIN [Dimension Set Entry] AS Dim ON Entry.[Dimension Set ID] = Dim.[Dimension Set ID]
WHERE Entry.[G/L Account No.] = [AccountNumber] AND Entry.[Posting Date] BETWEEN [DateFrom] AND [DateTo] AND Dim.[Dimension Code] = [DimensionCode]
GROUP BY Dim.[Dimension Value Code]
ORDER BY Dim.[Dimension Value Code]
SELECT Dim.[Dimension Value Code] AS 'Dimension Value Code', SUM(Entry.[Debit Amount]) AS 'Debit Amount Total', SUM(Entry.[Credit Amount]) AS 'Credit Amount Total', SUM(Entry.Amount) AS 'Net Amount Total', COUNT(*) AS 'Number of Entries'
```

| Key | Required | Binds to |
|---|---|---|
| `Account Number` | yes | `G/L Entry.G/L Account No.` |
| `Date From` | yes | `G/L Entry.Posting Date` |
| `Date To` | yes | `G/L Entry.Posting Date` |
| `Dimension Code` | yes | `Dimension Set Entry.Dimension Code` |
`Dimension Code` is the code (`DEPARTMENT`), not its caption. Level 1 is also the account's total in
the window from the entries themselves: it equals T1's `Net Change` for the same account and window.

## D3 — the entries behind a cell

```vql
DECLARE [AccountNumber] TYPE string AS 'Account Number' REQUIRED
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
FROM [G/L Entry] AS Entry
WHERE Entry.[G/L Account No.] = [AccountNumber] AND Entry.[Posting Date] BETWEEN [DateFrom] AND [DateTo]
ORDER BY Entry.[Posting Date], Entry.[Entry No.]
SELECT Entry.[Entry No.] AS 'Entry Number', Entry.[Posting Date] AS 'Posting Date', Entry.[Document Type] AS 'Document Type', Entry.[Document No.] AS 'Document Number', Entry.[External Document No.] AS 'External Document Number', Entry.Description AS 'Description', Entry.[Source Type] AS 'Source Type', Entry.[Source No.] AS 'Source Number', Entry.[Global Dimension 1 Code] AS 'Global Dimension 1 Code', Entry.[Global Dimension 2 Code] AS 'Global Dimension 2 Code', Entry.[Dimension Set ID] AS 'Dimension Set ID', Entry.[User ID] AS 'User ID', Entry.[Source Code] AS 'Source Code', Entry.[Debit Amount] AS 'Debit Amount', Entry.[Credit Amount] AS 'Credit Amount', Entry.Amount AS 'Amount'
```

| Key | Required | Binds to |
|---|---|---|
| `Account Number` | yes | `G/L Entry.G/L Account No.` |
| `Date From` | yes | `G/L Entry.Posting Date` |
| `Date To` | yes | `G/L Entry.Posting Date` |
## BL — budget names

```vql
FROM [G/L Budget Name] AS Budget
SELECT Budget.Name AS 'Budget Name', Budget.Description AS 'Description', Budget.Blocked AS 'Blocked'
```

## BU — budget by account and date

```vql
DECLARE [BudgetName] TYPE string AS 'Budget Name' REQUIRED
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
FROM [G/L Budget Entry] AS Entry
WHERE Entry.[Budget Name] = [BudgetName] AND Entry.Date BETWEEN [DateFrom] AND [DateTo]
GROUP BY Entry.[G/L Account No.], Entry.Date
ORDER BY Entry.[G/L Account No.], Entry.Date
SELECT Entry.[G/L Account No.] AS 'Account Number', Entry.Date AS 'Date', SUM(Entry.Amount) AS 'Amount Total'
```

| Key | Required | Binds to |
|---|---|---|
| `Budget Name` | yes | `G/L Budget Entry.Budget Name` |
| `Date From` | yes | `G/L Budget Entry.Date` |
| `Date To` | yes | `G/L Budget Entry.Date` |
The budget name comes from the user; BL lists the company's budgets when they do not know it. Sum
the dates inside each
window, and check the budget's sign against the actuals before comparing: the query's caveats say
how.

Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-13 from
d992d37. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
