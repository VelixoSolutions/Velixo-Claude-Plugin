# Data pulls from Sage Intacct

Run these through the Velixo connector's `execute_query` as written, with your
values in the `parameters` map. The query text goes in the `giql` argument whatever its fence says: a
` ```vql ` block goes there too, since `execute_query` has no `vql` argument. Do not call `prepare_query` for them and do not edit the text. If one
fails, report the error and stop rather than reformulating it. Parameter keys are the display labels
(the text after `AS` in each `DECLARE`); a key that matches no label is dropped silently and the
query returns nothing, which looks exactly like "no data". Pass every declared key, an unused optional
one as `null`: a key left out is not always treated as null. Always pass `top` — start at 250 and
raise it if the result comes back flagged `truncated`.

**Dates are `YYYY-MM-DD`; every GL pull names one book and one entity.** A GL call is capped at
25,000 journal lines (the connector groups it in memory): a window that fails with "returned more
than 25000 rows" is split in two and the halves added.

| Alias | Query | Used for |
|---|---|---|
| P1 | `gl-reporting-period-for-month` | the Sage reporting period that matches a month, if any |
| E1 | `entities` | entity names and base currencies |
| B1 | `gl-journals-and-books` | the reporting books; which module a journal symbol is |
| C1 | `gl-chart-of-accounts` | account type and normal balance (`Natural Net`); the cost-of-sales proposal |
| T1 | `gl-activity-by-account-for-window` | the income statement, or any account's movement, for one window |
| BL | `gl-budgets` | the budget the user names, or the default one |
| BU | `gl-budget-by-account-and-period` | budget by account and period |
| L1 | `gl-lines` | the transaction-lines pull: the documents behind one account and window |

## Which pulls each entry point needs

- **Every month the user names:** P1 once, to name the window (*Periods* in `instructions.md`).
- **Income statement flux:** C1, then T1 for the current window and T1 for the baseline window, per
  entity. History for the evidence ladder is T1 per month.
- **Budget-to-actual:** C1, T1 for the window, BU for the same window (BL first when the budget's ID
  is not known).
- **Forecast-to-actual:** as budget-to-actual, with the forecast's budget ID.
- **Cost or expense investigation:** T1 per month over the history the user wants, then the drill-down
  skill (`velixo-account-drilldown`) for the accounts still unexplained, then L1 for the cells
  it flags.

**Reconcile before any finding:** T1's `Natural Net` summed over the income-statement accounts of the
window is the entity's result for the window; the same figure from two windows' T1 differs only by
the activity between them. A budget comparison uses accounts present on either side, and an account
on one side only is a finding.

## P1 — the reporting period for a month

```vql
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
FROM [general-ledger/reporting-period] AS Period
WHERE Period.startDate = [DateFrom] AND Period.endDate = [DateTo]
SELECT Period.id AS 'Period Name', Period.startDate AS 'Start Date', Period.endDate AS 'End Date', Period.reportingPeriodType AS 'Period Type', Period.isBudgetable AS 'Available for Budgeting', Period.status AS 'Status'
```

| Key | Required | Binds to |
|---|---|---|
| `Date From` | yes | `general-ledger/reporting-period.startDate` |
| `Date To` | yes | `general-ledger/reporting-period.endDate` |
## E1 — entities

```vql
FROM [company-config/location] AS Location
WHERE Location.locationType = 'entity'
SELECT Location.id AS 'Location ID', Location.name AS 'Name', Location.baseCurrency AS 'Base Currency', Location.status AS 'Status'
```

## B1 — journals and books

```vql
FROM [general-ledger/journal] AS Journal
SELECT Journal.id AS 'Journal Symbol', Journal.name AS 'Title', Journal.bookId AS 'Book ID', Journal.bookType AS 'Book Type', Journal.status AS 'Status'
```

## C1 — chart of accounts

```vql
FROM [general-ledger/account] AS Account
ORDER BY Account.id
SELECT Account.id AS 'Account Number', Account.name AS 'Title', Account.accountType AS 'Account Type', Account.normalBalance AS 'Normal Balance', Account.closingType AS 'Closing Type', Account.status AS 'Status'
```

## T1 — account activity for a window, one entity

```vql
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
DECLARE [BookId] TYPE string AS 'Book ID' REQUIRED
DECLARE [EntityId] TYPE string AS 'Entity ID' REQUIRED
FROM [general-ledger/journal] AS Journal
INNER JOIN [general-ledger/journal-entry] AS Entry ON Entry.glJournal.id = Journal.id
INNER JOIN [general-ledger/journal-entry-line] AS Line ON Line.journalEntry.id = Entry.id
INNER JOIN [company-config/location] AS Location ON Line.dimensions.location.id = Location.id
WHERE Journal.bookId = [BookId] AND Line.state = 'posted' AND Line.entryDate >= [DateFrom] AND Line.entryDate <= [DateTo] AND Location.entity.id = [EntityId]
GROUP BY Line.glAccount.id, Location.entity.id, Line.txnType
SELECT Line.glAccount.id AS 'GL Account', Location.entity.id AS 'Entity ID', Line.txnType AS 'Transaction Type', SUM(Line.baseAmount) AS 'Base Amount', COUNT(*) AS 'Line Count'
```

| Key | Required | Binds to |
|---|---|---|
| `Date From` | yes | `general-ledger/journal-entry-line.entryDate` |
| `Date To` | yes | `general-ledger/journal-entry-line.entryDate` |
| `Book ID` | yes | `general-ledger/journal.bookId` |
| `Entity ID` | yes | `company-config/location.entity.id` |
## BL — budgets

```vql
FROM [general-ledger/budget] AS Budget
SELECT Budget.id AS 'Budget ID', Budget.description AS 'Description', Budget.isDefault AS 'Default Budget', Budget.status AS 'Status'
```

## BU — budget by account and period

```vql
DECLARE [BudgetId] TYPE string AS 'Budget ID' REQUIRED
DECLARE [EntityId] TYPE string AS 'Entity ID' REQUIRED
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
FROM [general-ledger/budget-detail] AS Detail
INNER JOIN [company-config/location] AS Location ON Detail.dimensions.location.id = Location.id
WHERE Detail.budget.id = [BudgetId] AND Location.entity.id = [EntityId] AND Detail.reportingPeriod.startDate >= [DateFrom] AND Detail.reportingPeriod.startDate <= [DateTo]
GROUP BY Detail.glAccount.id, Detail.reportingPeriod.id, Detail.reportingPeriod.startDate
SELECT Detail.glAccount.id AS 'GL Account ID', Detail.reportingPeriod.id AS 'Reporting Period ID', Detail.reportingPeriod.startDate AS 'Reporting Period Start Date', SUM(Detail.amount) AS 'Budget Amount'
```

| Key | Required | Binds to |
|---|---|---|
| `Budget ID` | yes | `general-ledger/budget-detail.budget.id` |
| `Entity ID` | yes | `company-config/location.entity.id` |
| `Date From` | yes | `general-ledger/budget-detail.reportingPeriod.startDate` |
| `Date To` | yes | `general-ledger/budget-detail.reportingPeriod.startDate` |
## L1 — journal lines for one account and window

```vql
DECLARE [AccountNumber] TYPE string AS 'Account Number' REQUIRED
DECLARE [EntityId] TYPE string AS 'Entity ID' REQUIRED
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
DECLARE [BookId] TYPE string AS 'Book ID' REQUIRED
FROM [general-ledger/journal] AS Journal
INNER JOIN [general-ledger/journal-entry] AS Entry ON Entry.glJournal.id = Journal.id
INNER JOIN [general-ledger/journal-entry-line] AS Line ON Line.journalEntry.id = Entry.id
INNER JOIN [company-config/location] AS Location ON Line.dimensions.location.id = Location.id
WHERE Journal.bookId = [BookId] AND Location.entity.id = [EntityId] AND Line.glAccount.id = [AccountNumber] AND Line.state = 'posted' AND Line.entryDate >= [DateFrom] AND Line.entryDate <= [DateTo]
ORDER BY Line.entryDate
SELECT Line.entryDate AS 'Entry Date', Journal.id AS 'Journal Symbol', Entry.txnNumber AS 'Entry Transaction Number', Entry.description AS 'Entry Description', Entry.id AS 'Entry Record ID', Line.documentId AS 'Line Document ID', Line.description AS 'Line Description', Location.id AS 'Location ID', Line.dimensions.customer.id AS 'Customer ID', Line.dimensions.customer.name AS 'Customer Name', Line.dimensions.vendor.id AS 'Vendor ID', Line.dimensions.vendor.name AS 'Vendor Name', Line.dimensions.department.id AS 'Department ID', Line.txnType AS 'Transaction Type', Line.baseAmount AS 'Base Amount', Line.txnAmount AS 'Transaction Amount'
```

| Key | Required | Binds to |
|---|---|---|
| `Account Number` | yes | `general-ledger/journal-entry-line.glAccount.id` |
| `Entity ID` | yes | `company-config/location.entity.id` |
| `Date From` | yes | `general-ledger/journal-entry-line.entryDate` |
| `Date To` | yes | `general-ledger/journal-entry-line.entryDate` |
| `Book ID` | yes | `general-ledger/journal.bookId` |
Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-09 from
5ed670d. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
