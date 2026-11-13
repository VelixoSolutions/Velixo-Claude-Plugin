# Data pulls

**Dates are `YYYY-MM-DD`; every GL pull names one book and, except C1, E1 and B1, one entity.** The
figures come from Velixo functions (`F` aliases); the reference data, the documents and the fallback
come from VQL queries.

| Alias | Source | Grain | Used for |
|---|---|---|---|
| P1 | query `gl-reporting-period-for-month` | reporting period | the Sage reporting period that matches a month, if any |
| E1 | query `entities` | entity | names and base currencies |
| B1 | query `gl-journals-and-books` | journal | the books, and which module a journal symbol is |
| C1 | query `gl-chart-of-accounts` | account | normal balance for `Natural Net`, titles |
| F1 | function `gl-turnover-by-account` | account, one entity and window | the control every table ties to; the entity axis |
| F4 | function `gl-dimension-members-with-activity` | dimension member | the rows of an axis: which customers, departments… moved |
| F5 | function `gl-turnover-for-dimension-member` | account × one member, one window | the amount on each row of an axis |
| F6 | function `dimension-member-names` | customer or project | the names beside the IDs |
| F7 | function `gl-turnover-by-location` | account × location | the entity's own and child locations |
| F2 | function `gl-budget-turnover-by-account` | account, one window | a budget column, when asked |
| D3 | query `gl-lines` | journal line | the documents behind a cell |
| D1 | query `gl-activity-by-business-account` | customer, vendor, employee × debit or credit | fallback: many members, or every class at once |
| D2 | query `gl-activity-by-dimension` | location, department, class, project × debit or credit | fallback: many members |
| T1 | query `gl-activity-by-account-for-window` | account × debit or credit | fallback for F1 |
| BU | query `gl-budget-by-account-and-period` | account × reporting period | fallback for F2 |

## The axis, as dimensions

| Axis the user picks | `<DIMENSION>` in F4 and F5 | Names |
|---|---|---|
| Business account on a revenue or receivable account | `CUSTOMER` | F6, `SI.CUSTOMERNAME` |
| Business account on a cost, expense or payable account | `VENDOR` | D1's `Vendor Name`, run once for the current window |
| Employee | `EMPLOYEE` | the ID, or the name the user gives |
| Project | `PROJECT` | F6, `SI.PROJECTNAME` |
| Department, class | `DEPARTMENT`, `CLASS` | D2's names, run once for the current window |
| Item | `ITEM` | the ID |
| Location (inside one entity) | none: F7, with the entity's locations as the list | D2's `Location Name` |
| Entity | none: F1 per entity, side by side | E1 |

A business-account drill reads one class. When the user wants customers, vendors and employees on
the same account at once, or the class is unclear (a clearing account), run D1: it classifies every
line in one query.

## The sequence, in as few calls as it takes

1. **Reference:** C1, P1 for each month, E1 when more than one entity is in scope.
2. **Rows:** F4 for every account in scope, every window together (the windows as paired lists), in
   one call: one invocation per account and axis.
3. **Amounts:** in one call (20 invocations a call, more calls only past that): F1 for each window,
   the accounts as the list; F5 for each member and window; F6 for customer or project names (the
   other axes' names as the table above says).
4. **Tie:** each account's members sum to its F1 figure per window; the difference is the axis's
   **unassigned** row (lines with no value on that dimension), its own line.
5. **Documents:** D3 only for the cells that need it.

An account by its three customers over two months is two calls after the reference pulls: F4, then
F1 ×2, F5 ×6 and F6 together. **More than about 60 invocations** (members × windows, three calls) is
the point to switch the axis to D1 or D2, one query per window, and to say so in the basis block.

## Functions

The figures come from Velixo functions first, through the connector's `execute_velixo_function`; the
queries are the fallback. Load the function tools' rules once:
`tool_guidance(["get_started", "list_velixo_functions", "describe_velixo_function", "execute_velixo_function"])`,
then `describe_velixo_function` once, naming every function this file uses: the argument order below is
the one it reports. Every call carries `connectionName` and `intent`.

- **Pass each invocation as written**, its `<placeholders>` replaced by your values, in the
  `invocations` argument as a JSON array. Do not reorder, drop or add arguments: they have no names on
  the wire, and an argument in the wrong place is read as something else.
- **Summary first, detail for what matters.** Pull the accounts with activity, then their turnover for
  the windows being compared, then compute materiality. Monthly history, budget detail and dimension
  breakdowns are pulled only for the accounts that cross it. Fewer, wider calls are the aim, not one
  call of every account by every month: a 900-row answer costs context and hides the few rows that
  matter among hundreds of zeros.
- **Batch:** up to 20 invocations a call, all windows and members of one step together; a result holds
  at most 1,000 rows. Read every item's `status`: an item can fail while its siblings return figures.
- **Lists:** an argument given a list returns one labelled row per member; the same members as one
  range string return their sum. Two lists pair row by row and never cross; a `#` range takes no
  other list. So accounts × windows is one invocation per window.
- **Decimals:** a function returns a plain number with the trailing zeros dropped (`-268080`,
  `-30421`). Present every amount with its two decimals (268,080.00), in tables, tie-outs and prose
  alike, as the queries' figures read.
- **Fall back to the query** when a function fails twice for the same arguments, or a breakdown needs
  more than about 60 invocations (three calls): run the query named beside the function, as written,
  and say in the basis block which pulls came from a query and why.

Every ```` ```json ```` block here was inserted at build time from Velixo's validated function library,
built 2026-11-13 from d992d37; the notes beside each block are the caveats that matter.
When a figure looks wrong, say so with the function's name and the figures, and never change an
invocation's shape to make it fit.
On Sage Intacct:

- **`Dimensions` never takes a list**: a list inside it is read as its first member, with no error.
  One member is one invocation. `No balances match the specified arguments` means nothing moved, not
  a failure.
- **Sign:** every amount comes back debit-positive, budgets included. `Natural Net` flips a
  credit-normal account by C1's normal balance.
- **Entity and child locations:** `LocationId` is the entity ID and `IncludeChildDimensionBalances`
  is `"TRUE"` by default, so the lines on the entity's child locations count in it, as they do in the
  GL and in the queries below. **Say so in every answer's basis block**, in words the user can act on:
  "Entity 100 includes its child locations; ask for location 100 only to exclude them." When the user
  asks to exclude them, set the flag to `"FALSE"` in every invocation of the run and say in the basis block that
  child locations are excluded. The queries cannot exclude them (they read the location's entity), so
  with children excluded a figure never falls back to a query: report the function's error instead.
  The location split (F7) sets it to `"FALSE"` by design, to read each location on its own.
### F1 — the account's turnover for a window

Each listed account's activity in one entity and book over one window: one invocation per window,
one labelled row per account. The income statement for a window, and the movement of any balance
sheet account, come from it. Up to 20 windows (13 months and a baseline) go in one
`execute_velixo_function` call.

```json
[{"functionName":"SI.TURNOVER","arguments":[null,[["<account>"],["<account>"]],"<Date From>","<Date To>","<Entity ID>","<Book ID>",null,null,"TRUE"]}]
```

**Caveats**

- **The sign is debit-positive**: a credit-normal account (revenue, a liability) reads negative.
  `Natural Net` is the figure as returned on a debit-normal account and the figure with its sign
  flipped on a credit-normal one, by C1's normal balance.
- **`IncludeChildDimensionBalances` is `"TRUE"`**: without it, entity 100 reads its own location only
  and drops the lines on its child locations (Skyline: 150 USA - Texas, 160 NY).
- **One account list per invocation**, from `gl-accounts-with-activity`. A list returns one row per
  account (`["40100",-268080]`); the same accounts as one range string (`"40000:49999"`) return their
  sum. `"#"` lists every account in the chart, zeros included (206 on Skyline): use the accounts with
  activity instead.
- **A list and a `#` range never go in the same invocation**, and two lists pair row by row: they
  never combine every account with every window. Accounts × windows is one invocation per window.
- **A result is capped at 1,000 rows**; an invocation of more accounts is split in two.
- No 25,000-line cap: the function reads Sage's balances, not journal lines.

The entity axis is F1 run per entity, side by side, never totalled across base currencies.

### F4 — the members of an axis that moved

The members of one dimension that moved on one account (or a range of accounts) in one entity and
book, over one or more windows: the rows of a drill-down, before any amount is pulled. With a range
and `"ACCOUNT,<DIMENSION>"` as the expansion order it returns account and member pairs.

```json
[{"functionName":"SI.EXPANDGLHISTORY","arguments":[null,"<account>",[["<Date From, window 1>"],["<Date From, window 2>"]],[["<Date To, window 1>"],["<Date To, window 2>"]],"<Entity ID>","<Book ID>",[["<DIMENSION>","*"]],null,"TRUE","<DIMENSION>",null,"turnover"]}]
```

**Caveats**

- **The dimension is named twice**: in `Dimensions` as `[["<DIMENSION>","*"]]` and in the expansion
  order. Named in the expansion order alone, the call fails with
  `UnexpectedDimensionInExpansionOrderError`. Several dimensions are comma-separated there
  (`"DEPARTMENT,PROJECT"`), each with its own `*` pair in `Dimensions`.
- **Validated dimension names:** `CUSTOMER`, `VENDOR`, `EMPLOYEE`, `DEPARTMENT`, `CLASS`, `PROJECT`,
  `ITEM`. Location is not one of them: it is the `LocationId` argument, and `LOCATION` here fails
  with `DuplicateEntriesForPredefinedDimensionError`. A user-defined dimension goes in
  `UserDefinedDimensions` instead; none was validated.
- **It returns IDs only.** Names come from `dimension-member-names`.
- **"No balances match the specified arguments" is an empty result**: no line on the account carries
  that dimension in the window (Skyline: no vendor on 20100 in June 2018).
- **A line with no value on the dimension is not listed.** The members' turnovers then sum to less
  than the account's: the difference is the **unassigned** row, shown as its own line.

### F5 — one member's turnover

The activity of one dimension member on each listed account, in one entity, book and window: one
invocation per member and window, one labelled row per account. A drill-down by customer over two
months is (customers × 2) invocations, 20 to a call.

```json
[{"functionName":"SI.TURNOVER","arguments":[null,[["<account>"]],"<Date From>","<Date To>","<Entity ID>","<Book ID>",[["<DIMENSION>","<member ID>"]],null,"TRUE"]}]
```

**Caveats**

- **One member per invocation.** `Dimensions` takes one value per dimension and never a list. A
  second pair for the same dimension fails (`DuplicateEntriesForPredefinedDimensionError`), `"*"`
  sums every member into one figure, and **a list of members in the value is read as its first member
  with no error**: `[["PROJECT",[["10001"],["10002"]]]]` returned project 10001's −66,840.00 alone.
  Never put a list in `Dimensions`.
- **Two dimensions at once** are two pairs: `[["DEPARTMENT","200"],["PROJECT","10001"]]` is that
  department's lines on that project.
- **The members' sum ties to the account's turnover** (`gl-turnover-by-account`); a shortfall is the
  unassigned row of the axis, its own line.
- The sign is debit-positive; `IncludeChildDimensionBalances` is `"TRUE"`.
- **Many members** (more than about 60 invocations, three calls): run `gl-activity-by-dimension` or
  `gl-activity-by-business-account` for the window instead, one query per window, and say so.

### F6 — member names

The names of customers or projects, one labelled row per ID, in one invocation: `SI.CUSTOMERNAME` or
`SI.PROJECTNAME` with the IDs as a list. It goes in the same call as the turnovers. These two are the
only name lookups validated; the other dimensions' names come from the queries (the business-account
query's `Vendor Name`, the dimension query's location, department and class names), and an employee
or item is shown by its ID unless the user names it.

```json
[{"functionName":"SI.CUSTOMERNAME","arguments":[[["<member ID>"],["<member ID>"]]]}]
```

### F7 — by location inside one entity

One account's activity on each listed location, one book and window: an entity split into its own
location and its child locations. The child flag is `"FALSE"` here, and only here, so each location
reads its own lines.

```json
[{"functionName":"SI.TURNOVER","arguments":[null,"<account>","<Date From>","<Date To>",[["<location>"],["<location>"]],"<Book ID>",null,null,"FALSE"]}]
```

**Caveats**

- **The locations sum to the entity** read with the flag `"TRUE"`: the tie-out.
- **Never sum across entities**: entities have their own base currencies.
- The sign is debit-positive.

The location IDs come from the user. When they are not known, the location axis is D2, whose
`Location ID` column lists them. The location sum ties to F1 for the entity.

### F2 — budget for a window

One budget's amount for each listed account, one entity and one window: one invocation per window,
one labelled row per account.

```json
[{"functionName":"SI.BUDGETTURNOVER","arguments":[null,[["<account>"],["<account>"]],"<Budget ID>","<Date From>","<Date To>","<Entity ID>","<Book ID>",null,null,"TRUE"]}]
```

**Caveats**

- **The sign is debit-positive, like the actuals**: a revenue budget reads negative. This is the
  opposite of `gl-budget-by-account-and-period`, whose amounts are already in the natural sign. Flip
  a credit-normal account by C1 as for the actuals, and never mix the two pulls' figures unconverted:
  a variance computed on one of each doubles instead of cancelling.
- The budget ID is the one the user names, or the default from `gl-budgets` (BL).
- `IncludeChildDimensionBalances` is `"TRUE"`, as for the actuals.

The budget name comes from the user; F2 is debit-positive like F1.

## Queries

Run these through the Velixo connector's `execute_query` as written, with your
values in the `parameters` map. The query text goes in the `giql` argument whatever its fence says: a
` ```vql ` block goes there too, since `execute_query` has no `vql` argument. Do not call `prepare_query` for them and do not edit the text. If one
fails, report the error and stop rather than reformulating it. Parameter keys are the display labels
(the text after `AS` in each `DECLARE`); a key that matches no label is dropped silently and the
query returns nothing, which looks exactly like "no data". Pass every declared key, an unused optional
one as `null`: a key left out is not always treated as null. Always set `top` — start at 250 and
raise it if the result comes back flagged `truncated`.
The GL queries are capped at 25,000 journal lines a call (the connector groups them in memory):
narrow the window when one fails with "returned more than 25000 rows".

### P1 — the reporting period for a month

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
### E1 — entities

```vql
FROM [company-config/location] AS Location
WHERE Location.locationType = 'entity'
SELECT Location.id AS 'Location ID', Location.name AS 'Name', Location.baseCurrency AS 'Base Currency', Location.status AS 'Status'
```

### B1 — journals and books

```vql
FROM [general-ledger/journal] AS Journal
SELECT Journal.id AS 'Journal Symbol', Journal.name AS 'Title', Journal.bookId AS 'Book ID', Journal.bookType AS 'Book Type', Journal.status AS 'Status'
```

### C1 — chart of accounts

```vql
FROM [general-ledger/account] AS Account
ORDER BY Account.id
SELECT Account.id AS 'Account Number', Account.name AS 'Title', Account.accountType AS 'Account Type', Account.normalBalance AS 'Normal Balance', Account.closingType AS 'Closing Type', Account.status AS 'Status'
```

### D3 — the lines behind a cell

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
### D1 — by business account (fallback)

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
### D2 — by location, department, class and project (fallback)

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
D2's `Location ID` splits the entity into its own location and its child locations. Another dimension
(item, warehouse, a custom one) on many members is a rebuild of D2 through `prepare_query` with D2 as
`baseQuery`, named in the basis block with the tie-out it passed.

### T1 — account activity for a window (fallback for F1)

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
### BU — budget by account and period (fallback for F2)

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
BU is already in the natural sign: never compare it with F2 unconverted.

Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-13 from
d992d37. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
