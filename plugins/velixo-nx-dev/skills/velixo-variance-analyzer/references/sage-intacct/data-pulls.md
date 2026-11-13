# Data pulls from Sage Intacct

**Dates are `YYYY-MM-DD`; every GL pull names one book and one entity.** The figures (activity,
budget, balances) come from Velixo functions (`F` aliases); the reference data, the documents and the
fallback come from VQL queries.

| Alias | Source | Used for |
|---|---|---|
| P1 | query `gl-reporting-period-for-month` | the Sage reporting period that matches a month, if any |
| E1 | query `entities` | entity names and base currencies |
| B1 | query `gl-journals-and-books` | the reporting books; which module a journal symbol is |
| C1 | query `gl-chart-of-accounts` | account type and normal balance (`Natural Net`); the cost-of-sales proposal |
| BL | query `gl-budgets` | the budget the user names, or the default one |
| F0 | function `gl-accounts-with-activity` | the accounts that moved in the windows compared, or carry a budget |
| F1 | function `gl-turnover-by-account` | each account's activity for one window: the income statement, any account's movement |
| F2 | function `gl-budget-turnover-by-account` | budget by account for one window |
| F3 | function `gl-balance-by-account` | opening and closing balances, for a balance-sheet flux |
| L1 | query `gl-lines` | the transaction-lines pull: the documents behind one account and window |
| T1 | query `gl-activity-by-account-for-window` | fallback for F1 |
| BU | query `gl-budget-by-account-and-period` | fallback for F2 |

## Which pulls each entry point needs

- **Every month the user names:** P1 once, to name the window (*Periods* in `instructions.md`).
- **Reference data, once per run:** C1, and E1 when more than one entity is in scope. B1 only when
  the user names a book other than Accrual.
- **Income statement flux:** F0 for the two windows, then F1 for each window in **one call** (one
  invocation per window, the F0 accounts as the list). Materiality on that summary. History for the
  evidence ladder is F1 per month, for the material accounts only, every month in one call.
- **Budget-to-actual:** F0 for the window, actual and budgeted (two invocations, one call), then F1
  and F2 for the window in one call, over the accounts on either list (BL first when the budget's ID
  is not known).
- **Forecast-to-actual:** as budget-to-actual, with the forecast's budget ID.
- **Balance-sheet flux:** F3 for the window, in one call; closing less opening ties to F1.
- **Cost or expense investigation:** F1 per month over the history the user wants, for the accounts
  in scope; then the drill-down skill (`velixo-account-drilldown`) for the accounts still unexplained,
  then L1 for the cells it flags.

A month-end income statement flux is therefore C1, P1 for each month, F0, and one F1 call for both
windows. Never pull every account (`"#"`) by every month in one go.

**Reconcile before any finding:** F1's `Natural Net` summed over the income-statement accounts of the
window is the entity's result for the window; the same figure from two windows differs only by the
activity between them. A budget comparison uses accounts present on either side, and an account on
one side only is a finding. **F1 and F2 are debit-positive; BU is already natural-signed.** Convert
each by C1 before comparing, and never compare an F2 figure with a BU one.

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
### F0 — accounts with activity

The accounts with activity in one entity and book over one or more windows, or with a budget amount
in a window: the account list the turnover and budget pulls take, so nothing is pulled for an
account that did not move. One row per account, no amounts.

```json
[{"functionName":"SI.EXPANDGLHISTORY","arguments":[null,"*",[["<Date From, window 1>"],["<Date From, window 2>"]],[["<Date To, window 1>"],["<Date To, window 2>"]],"<Entity ID>","<Book ID>",null,null,"TRUE","ACCOUNT",null,"turnover"]}]
```

**Caveats**

- **Windows are paired lists**: the n-th `FromDate` goes with the n-th `ToDate`; one window is two
  plain strings. An account counts when it moved in any window.
- **Budget side:** the accounts that carry an amount in a budget are the same call with
  `BalanceTypesToLookAt` `"budgeted"` and the budget ID last:
  `[null,"*","<Date From>","<Date To>","<Entity ID>","<Book ID>",null,null,"TRUE","ACCOUNT",null,"budgeted","<Budget ID>"]`.
  A budget-to-actual covers the accounts on either list.
- **Statistical accounts come back too** (Skyline: `CUSCO`, `HRSBU`, `HRSNN`, `HRSNU`). Keep only the
  accounts the chart of accounts (C1) lists with the statement in scope.
- **"No balances match the specified arguments" is an empty result**, not a failure: nothing moved.
- `IncludeChildDimensionBalances` is `"TRUE"`: without it the entity reads its own location's lines
  only and misses its child locations.

For the budget side, the same invocation with `"budgeted"` and the budget ID:
`[null,"*","<Date From>","<Date To>","<Entity ID>","<Book ID>",null,null,"TRUE","ACCOUNT",null,"budgeted","<Budget ID>"]`.
Keep the accounts C1 lists with the statement in scope (it drops statistical accounts).

### F1 — turnover by account for a window

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

One invocation per window, all of them in one call. The result is one row per account,
`[account, amount]`, debit-positive.

### F2 — budget by account for a window

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

Debit-positive, like F1: a revenue budget reads negative.

### F3 — opening and closing balances

Each listed account's balance at the start and at the end of a window, in one entity and book: the
balance-sheet side of a flux. Both invocations go in one call.

```json
[{"functionName":"SI.OPENINGBALANCE","arguments":[null,[["<account>"],["<account>"]],"<Date From>","<Date To>","<Entity ID>","<Book ID>",null,null,"TRUE"]},
 {"functionName":"SI.CLOSINGBALANCE","arguments":[null,[["<account>"],["<account>"]],"<Date From>","<Date To>","<Entity ID>","<Book ID>",null,null,"TRUE"]}]
```

**Caveats**

- **Closing less opening equals the window's turnover** (`gl-turnover-by-account`) for the same
  account, entity and book: the tie-out for every balance read.
- **The sign is debit-positive**: a liability reads negative. Flip a credit-normal account by C1.
- **It does not split a control account by journal.** The AR and AP aging tie needs the subledger
  (ARJ and CRJ, APJ and CDJ) apart from opening balances and manual entries, which only
  `gl-account-balance-by-journal` gives: the aging skill keeps that query.
- `IncludeChildDimensionBalances` is `"TRUE"`.

## Queries

Run these through the Velixo connector's `execute_query` as written, with your
values in the `parameters` map. The query text goes in the `giql` argument whatever its fence says: a
` ```vql ` block goes there too, since `execute_query` has no `vql` argument. Do not call `prepare_query` for them and do not edit the text. If one
fails, report the error and stop rather than reformulating it. Parameter keys are the display labels
(the text after `AS` in each `DECLARE`); a key that matches no label is dropped silently and the
query returns nothing, which looks exactly like "no data". Pass every declared key, an unused optional
one as `null`: a key left out is not always treated as null. Always set `top` — start at 250 and
raise it if the result comes back flagged `truncated`.
A GL query is capped at 25,000 journal lines (the connector groups it in memory): a window that fails
with "returned more than 25000 rows" is split in two and the halves added.

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

### BL — budgets

```vql
FROM [general-ledger/budget] AS Budget
SELECT Budget.id AS 'Budget ID', Budget.description AS 'Description', Budget.isDefault AS 'Default Budget', Budget.status AS 'Status'
```

### L1 — journal lines for one account and window

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
Debit and credit come back as separate rows, both positive; `Natural Net` from C1.

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
Already in the natural sign: never flip it, and never compare it with F2 unconverted.

Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-13 from
d992d37. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
