# Data pulls

Run these through the Velixo connector's `execute_query` as written, with your
values in the `parameters` map. The query text goes in the `giql` argument whatever its fence says: a
` ```vql ` block goes there too, since `execute_query` has no `vql` argument. Do not call `prepare_query` for them and do not edit the text. If one
fails, report the error and stop rather than reformulating it. Parameter keys are the display labels
(the text after `AS` in each `DECLARE`); a key that matches no label is dropped silently and the
query returns nothing, which looks exactly like "no data". Pass every declared key, an unused optional
one as `null`: a key left out is not always treated as null. Always set `top` — start at 250 and
raise it if the result comes back flagged `truncated`.

**Dates are `YYYY-MM-DD`; every GL pull names one book and, except C1, E1 and B1, one entity.** The
GL pulls are capped at 25,000 journal lines a call (the connector groups them in memory): narrow the
window when one fails with "returned more than 25000 rows".

| Alias | Query | Grain | Used for |
|---|---|---|---|
| P1 | `gl-reporting-period-for-month` | reporting period | the Sage reporting period that matches a month, if any |
| E1 | `entities` | entity | names and base currencies |
| B1 | `gl-journals-and-books` | journal | the books, and which module a journal symbol is |
| C1 | `gl-chart-of-accounts` | account | normal balance for `Natural Net`, titles |
| T1 | `gl-activity-by-account-for-window` | account × debit or credit, one entity and window | the control every table ties to; the entity axis |
| D1 | `gl-activity-by-business-account` | customer, vendor, employee × debit or credit | the business-account axis |
| D2 | `gl-activity-by-dimension` | department, class, project × debit or credit | the dimension axes |
| D3 | `gl-lines` | journal line | the documents behind a cell |
| BU | `gl-budget-by-account-and-period` | account × reporting period | a budget column, when asked |

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
Keep the rows of the accounts in scope; the others are the rest of the entity's ledger. The entity
axis is T1 run for each entity, side by side, never totalled across base currencies.

## D1 — by business account

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
GROUP BY Line.dimensions.customer.id, Line.dimensions.customer.name, Line.dimensions.vendor.id, Line.dimensions.vendor.name, Line.dimensions.employee.id, Line.txnType
SELECT Line.dimensions.customer.id AS 'Customer ID', Line.dimensions.customer.name AS 'Customer Name', Line.dimensions.vendor.id AS 'Vendor ID', Line.dimensions.vendor.name AS 'Vendor Name', Line.dimensions.employee.id AS 'Employee ID', Line.txnType AS 'Transaction Type', SUM(Line.baseAmount) AS 'Base Amount', COUNT(*) AS 'Line Count'
```

| Key | Required | Binds to |
|---|---|---|
| `Account Number` | yes | `general-ledger/journal-entry-line.glAccount.id` |
| `Entity ID` | yes | `company-config/location.entity.id` |
| `Date From` | yes | `general-ledger/journal-entry-line.entryDate` |
| `Date To` | yes | `general-ledger/journal-entry-line.entryDate` |
| `Book ID` | yes | `general-ledger/journal.bookId` |
## D2 — by department, class and project

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
GROUP BY Line.dimensions.location.id, Line.dimensions.location.name, Line.dimensions.department.id, Line.dimensions.department.name, Line.dimensions.class.id, Line.dimensions.class.name, Line.dimensions.project.id, Line.dimensions.project.name, Line.txnType
SELECT Line.dimensions.location.id AS 'Location ID', Line.dimensions.location.name AS 'Location Name', Line.dimensions.department.id AS 'Department ID', Line.dimensions.department.name AS 'Department Name', Line.dimensions.class.id AS 'Class ID', Line.dimensions.class.name AS 'Class Name', Line.dimensions.project.id AS 'Project ID', Line.dimensions.project.name AS 'Project Name', Line.txnType AS 'Transaction Type', SUM(Line.baseAmount) AS 'Base Amount', COUNT(*) AS 'Line Count'
```

| Key | Required | Binds to |
|---|---|---|
| `Account Number` | yes | `general-ledger/journal-entry-line.glAccount.id` |
| `Entity ID` | yes | `company-config/location.entity.id` |
| `Date From` | yes | `general-ledger/journal-entry-line.entryDate` |
| `Date To` | yes | `general-ledger/journal-entry-line.entryDate` |
| `Book ID` | yes | `general-ledger/journal.bookId` |
D2's `Location ID` splits the entity into its own location and its child locations; the entity
axis itself is T1 run per entity. Another dimension (item, warehouse, a custom one) is a rebuild of D2 through `prepare_query` with
D2 as `baseQuery`, named in the basis block with the tie-out it passed.

## D3 — the lines behind a cell

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
The budget name comes from the user; budget amounts are already in the natural sign.

Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-09 from
50f3928. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
