# Data pulls from Business Central

Run these through the Velixo connector's `execute_query` as written, with your
values in the `parameters` map. The query text goes in the `giql` argument whatever its fence says: a
` ```vql ` block goes there too, since `execute_query` has no `vql` argument. Do not call `prepare_query` for them and do not edit the text. If one
fails, report the error and stop rather than reformulating it. Parameter keys are the display labels
(the text after `AS` in each `DECLARE`); a key that matches no label is dropped silently and the
query returns nothing, which looks exactly like "no data". Pass every declared key, an unused optional
one as `null`: a key left out is not always treated as null. Always set `top` — start at 250 and
raise it if the result comes back flagged `truncated`.

**Dates are `YYYY-MM-DD`; every GL figure is read by posting date over a window.** The account-level
pulls (T1, T2) are flow fields, one call per window whatever the volume. The entry pull (L1) reads at
most 10,000 G/L entries a call: split a window that fails or comes back truncated.

| Alias | Query | Used for |
|---|---|---|
| P1 | `gl-accounting-periods` | whether a month is an accounting period; fiscal-year starts |
| G0 | `gl-setup` | local currency; the global dimensions' codes |
| C1 | `gl-chart-of-accounts` | account category (`Natural Net`), posting accounts; the cost-of-sales proposal |
| T1 | `gl-net-change-by-account-for-window` | the income statement, or any account's movement, for one window |
| T1a | `gl-account-net-change-for-window` | one named account's movement and category, for one window |
| T2 | `gl-balance-by-account-at-date` | balance-sheet accounts at a month end, for a balance-sheet flux |
| BL | `gl-budget-names` | the budget the user names |
| BU | `gl-budget-by-account-and-date` | budget by account and date |
| L1 | `gl-entries` | the transaction-lines pull: the entries behind one account and window |

## Which pulls each entry point needs

- **Every month the user names:** P1 once, to name the window (*Periods* in `instructions.md`).
- **Income statement flux:** G0 and C1, then T1 for the current window and T1 for the baseline
  window. History for the evidence ladder is T1 per month.
- **Balance sheet flux:** T2 at the current and the baseline month ends; the change on each
  balance-sheet account equals T1's net change for the months between, which is the tie-out.
- **Budget-to-actual:** C1, T1 for the window, BU for the same window (BL first when the budget's
  name is not known). Check the budget's sign per category against T1 before any variance.
- **Forecast-to-actual:** as budget-to-actual, with the forecast's budget name.
- **A department or project review:** the same pulls for the company, then the drill-down skill
  (`velixo-account-drilldown`) on that dimension for the accounts in scope. T1 reads the
  whole company; a dimension-filtered account list is a rebuild of T1 through `prepare_query`, named
  in the basis block with its tie-out.
- **One account, or a few the user names:** T1a per account and window, in place of T1 and C1; it
  carries the account's category for `Natural Net`. T1 is for the whole statement only.
- **Cost or expense investigation:** T1 per month over the history the user wants, then the
  drill-down skill for the accounts still unexplained, then L1 for the cells it flags.

**Reconcile before any finding:** T1's `Natural Net` summed over the income-statement accounts of
the window is the company's result for the window; a balance-sheet account's T2 change between two
dates equals T1 between them. A budget comparison uses accounts present on either side, and an
account on one side only is a finding.

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

## C1 — chart of accounts

```vql
FROM [G/L Account] AS Account
ORDER BY Account.[No.]
SELECT Account.[No.] AS 'Account Number', Account.Name AS 'Name', Account.[Account Type] AS 'Account Type', Account.[Income/Balance] AS 'Income Statement or Balance Sheet', Account.[Account Category] AS 'Account Category', Account.[Account Subcategory Descript.] AS 'Account Subcategory', Account.[Debit/Credit] AS 'Debit/Credit Setting', Account.Blocked AS 'Blocked'
```

## T1 — every account's net change in a window

```vql
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
FROM [G/L Account] AS Account
WHERE Account.[Account Type] = 'Posting' AND Account.[Date Filter] BETWEEN [DateFrom] AND [DateTo]
ORDER BY Account.[No.]
SELECT Account.[No.] AS 'Account Number', Account.Name AS 'Name', Account.[Income/Balance] AS 'Income Statement or Balance Sheet', Account.[Account Category] AS 'Account Category', Account.[Net Change] AS 'Net Change', Account.[Debit Amount] AS 'Debit Amount', Account.[Credit Amount] AS 'Credit Amount'
```

| Key | Required | Binds to |
|---|---|---|
| `Date From` | yes | `G/L Account.Date Filter` |
| `Date To` | yes | `G/L Account.Date Filter` |
## T1a — one account's net change in a window

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
## T2 — every account's balance at a date

```vql
DECLARE [AsOfDate] TYPE date AS 'As Of Date' REQUIRED
FROM [G/L Account] AS Account
WHERE Account.[Account Type] = 'Posting' AND Account.[Date Filter] <= [AsOfDate]
ORDER BY Account.[No.]
SELECT Account.[No.] AS 'Account Number', Account.Name AS 'Name', Account.[Income/Balance] AS 'Income Statement or Balance Sheet', Account.[Account Category] AS 'Account Category', Account.[Balance at Date] AS 'Balance at Date'
```

| Key | Required | Binds to |
|---|---|---|
| `As Of Date` | yes | `G/L Account.Date Filter` |
Read the balance-sheet rows only: an income-statement account's balance is its history.

## BL — G/L budgets

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
## L1 — G/L entries for one account and window

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
Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-11 from
e1c8e93. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
