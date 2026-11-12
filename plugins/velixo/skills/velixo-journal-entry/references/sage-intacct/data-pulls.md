# Data pulls

Run these through the Velixo connector's `execute_query` as written, with your
values in the `parameters` map. The query text goes in the `giql` argument whatever its fence says: a
` ```vql ` block goes there too, since `execute_query` has no `vql` argument. Do not call `prepare_query` for them and do not edit the text. If one
fails, report the error and stop rather than reformulating it. Parameter keys are the display labels
(the text after `AS` in each `DECLARE`); a key that matches no label is dropped silently and the
query returns nothing, which looks exactly like "no data". Pass every declared key, an unused optional
one as `null`: a key left out is not always treated as null. Always set `top` — start at 250 and
raise it if the result comes back flagged `truncated`.

| Alias | Query | Used for |
|---|---|---|
| E1 | `entities` | the entity on each line, and its base currency |
| B1 | `gl-journals-and-books` | the journal exists, is active, and which book it posts to |
| C1 | `gl-chart-of-accounts` | each account exists and is active; names for the preview |
| JE1 | `gl-journal-entry-by-id` | the read-back after a write; the lookup before a reversal or reclass |
| JE2 | `gl-journal-entries-by-description` | the duplicate check; the already-offset check (`%of entry <id>%`: `LIKE` is case-sensitive) |

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

## JE1 — one entry by record id

```vql
DECLARE [EntryId] TYPE string AS 'Entry ID' REQUIRED
FROM [general-ledger/journal-entry] AS Entry
INNER JOIN [general-ledger/journal-entry-line] AS Line ON Line.journalEntry.id = Entry.id
WHERE Entry.id = [EntryId]
ORDER BY Line.lineNumber
SELECT Entry.id AS 'Entry Record ID', Entry.txnNumber AS 'Transaction Number', Entry.glJournal.id AS 'Journal Symbol', Entry.postingDate AS 'Posting Date', Entry.state AS 'State', Entry.description AS 'Entry Description', Line.lineNumber AS 'Line Number', Line.glAccount.id AS 'GL Account ID', Line.dimensions.location.id AS 'Location ID', Line.txnType AS 'Transaction Type', Line.txnAmount AS 'Transaction Amount', Line.baseAmount AS 'Base Amount', Line.description AS 'Line Description', Line.dimensions.department.id AS 'Department ID', Line.dimensions.class.id AS 'Class ID', Line.dimensions.project.id AS 'Project ID', Line.dimensions.task.id AS 'Task ID', Line.dimensions.customer.id AS 'Customer ID', Line.dimensions.vendor.id AS 'Vendor ID', Line.dimensions.employee.id AS 'Employee ID', Line.dimensions.item.id AS 'Item ID', Line.dimensions.warehouse.id AS 'Warehouse ID', Line.dimensions.contract.id AS 'Contract ID'
```

| Key | Required | Binds to |
|---|---|---|
| `Entry ID` | yes | `general-ledger/journal-entry.id` |
## JE2 — entries by description

```vql
DECLARE [DescriptionText] TYPE string AS 'Description Text' REQUIRED
FROM [general-ledger/journal-entry] AS Entry
WHERE Entry.description LIKE [DescriptionText]
SELECT Entry.id AS 'Record ID', Entry.txnNumber AS 'Transaction Number', Entry.glJournal.id AS 'Journal Symbol', Entry.postingDate AS 'Posting Date', Entry.state AS 'State', Entry.description AS 'Description', Entry.reversedBy.id AS 'Reversing Entry ID', Entry.reversedFromDate AS 'Reversed From Date'
```

| Key | Required | Binds to |
|---|---|---|
| `Description Text` | yes | `general-ledger/journal-entry.description` |
Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-12 from
aea6566. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
