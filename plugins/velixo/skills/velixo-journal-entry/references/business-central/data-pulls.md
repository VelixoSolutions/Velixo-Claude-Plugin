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
| G0 | `gl-setup` | local currency; which dimensions are global |
| B1 | `gl-journal-batches` | the batch exists and has no balancing account |
| B2 | `gl-journal-lines-in-batch` | what the batch already holds; the read-back of unposted lines with their VAT |
| A1 | `gl-journal-accounts` | each account is a posting account, not blocked, allows direct posting; its VAT groups; names for the preview |
| V1 | `vat-posting-setup` | the VAT rate and account of each VAT-carrying line |
| D1 | `dimension-values` | each dimension value exists and is postable |
| JE1 | `gl-entries-by-document` | the read-back after posting; the lookup before a reversal or reclass |
| JE2 | `gl-entry-dimensions-by-document` | the dimensions a reversal or reclass copies |
| JE3 | `gl-entries-by-description` | the duplicate check; the already-reversed check (`Reversal of <document number>`) |

## G0 — General Ledger Setup

```vql
FROM [General Ledger Setup] AS Setup
SELECT Setup.[LCY Code] AS 'LCY Code', Setup.[Global Dimension 1 Code] AS 'Global Dimension 1 Code', Setup.[Global Dimension 2 Code] AS 'Global Dimension 2 Code', Setup.[Shortcut Dimension 3 Code] AS 'Shortcut Dimension 3 Code', Setup.[Shortcut Dimension 4 Code] AS 'Shortcut Dimension 4 Code', Setup.[Shortcut Dimension 5 Code] AS 'Shortcut Dimension 5 Code', Setup.[Shortcut Dimension 6 Code] AS 'Shortcut Dimension 6 Code', Setup.[Shortcut Dimension 7 Code] AS 'Shortcut Dimension 7 Code', Setup.[Shortcut Dimension 8 Code] AS 'Shortcut Dimension 8 Code', Setup.[Allow Posting From] AS 'Allow Posting From', Setup.[Allow Posting To] AS 'Allow Posting To'
```

## B1 — journal batches

```vql
DECLARE [TemplateName] TYPE string AS 'Journal Template' REQUIRED
FROM [Gen. Journal Batch] AS Batch
WHERE Batch.[Journal Template Name] = [TemplateName]
ORDER BY Batch.Name
SELECT Batch.[Journal Template Name] AS 'Template Name', Batch.Name AS 'Batch Name', Batch.Description AS 'Description', Batch.[No. Series] AS 'Number Series', Batch.[Posting No. Series] AS 'Posting Number Series', Batch.[Reason Code] AS 'Reason Code', Batch.[Bal. Account Type] AS 'Balancing Account Type', Batch.[Bal. Account No.] AS 'Balancing Account Number'
```

| Key | Required | Binds to |
|---|---|---|
| `Journal Template` | yes | `Gen. Journal Batch.Journal Template Name` |
## B2 — lines waiting in a batch

```vql
DECLARE [JournalTemplate] TYPE string AS 'Journal Template' REQUIRED
DECLARE [JournalBatch] TYPE string AS 'Journal Batch' REQUIRED
FROM [Gen. Journal Line] AS Line
WHERE Line.[Journal Template Name] = [JournalTemplate] AND Line.[Journal Batch Name] = [JournalBatch]
ORDER BY Line.[Line No.]
SELECT Line.[Line No.] AS 'Line Number', Line.[Document No.] AS 'Document Number', Line.[Account No.] AS 'Account Number', Line.Amount AS 'Amount', Line.[Gen. Posting Type] AS 'General Posting Type', Line.[Gen. Bus. Posting Group] AS 'General Business Posting Group', Line.[Gen. Prod. Posting Group] AS 'General Product Posting Group', Line.[VAT Bus. Posting Group] AS 'VAT Business Posting Group', Line.[VAT Prod. Posting Group] AS 'VAT Product Posting Group', Line.[VAT %] AS 'VAT Percent', Line.[VAT Amount] AS 'VAT Amount', Line.[VAT Base Amount] AS 'VAT Base Amount'
```

| Key | Required | Binds to |
|---|---|---|
| `Journal Template` | yes | `Gen. Journal Line.Journal Template Name` |
| `Journal Batch` | yes | `Gen. Journal Line.Journal Batch Name` |
## A1 — posting accounts for a journal

```vql
FROM [G/L Account] AS Account
WHERE Account.[Account Type] = 'Posting'
ORDER BY Account.[No.]
SELECT Account.[No.] AS 'Number', Account.Name AS 'Name', Account.[Account Category] AS 'Account Category', Account.Blocked AS 'Blocked', Account.[Direct Posting] AS 'Direct Posting', Account.[Gen. Posting Type] AS 'General Posting Type', Account.[VAT Bus. Posting Group] AS 'VAT Business Posting Group', Account.[VAT Prod. Posting Group] AS 'VAT Product Posting Group'
```

## V1 — VAT posting setup

```vql
FROM [VAT Posting Setup] AS Setup
ORDER BY Setup.[VAT Bus. Posting Group], Setup.[VAT Prod. Posting Group]
SELECT Setup.[VAT Bus. Posting Group] AS 'VAT Business Posting Group', Setup.[VAT Prod. Posting Group] AS 'VAT Product Posting Group', Setup.[VAT Calculation Type] AS 'VAT Calculation Type', Setup.[VAT %] AS 'VAT Percent', Setup.[Purchase VAT Account] AS 'Purchase VAT Account', Setup.[Sales VAT Account] AS 'Sales VAT Account', Setup.Blocked AS 'Blocked'
```

## D1 — dimension values

```vql
DECLARE [DimensionCode] TYPE string AS 'Dimension Code' REQUIRED
FROM [Dimension Value] AS DimValue
WHERE DimValue.[Dimension Code] = [DimensionCode]
ORDER BY DimValue.Code
SELECT DimValue.[Dimension Code] AS 'Dimension Code', DimValue.Code AS 'Value Code', DimValue.Name AS 'Name', DimValue.[Dimension Value Type] AS 'Dimension Value Type', DimValue.Blocked AS 'Blocked'
```

| Key | Required | Binds to |
|---|---|---|
| `Dimension Code` | yes | `Dimension Value.Dimension Code` |
## JE1 — posted entries of one document

```vql
DECLARE [DocumentNumber] TYPE string AS 'Document Number' REQUIRED
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
FROM [G/L Entry] AS Entry
WHERE Entry.[Document No.] = [DocumentNumber] AND Entry.[Posting Date] BETWEEN [DateFrom] AND [DateTo]
ORDER BY Entry.[Entry No.]
SELECT Entry.[Entry No.] AS 'Entry Number', Entry.[Transaction No.] AS 'Transaction Number', Entry.[Posting Date] AS 'Posting Date', Entry.[Document Type] AS 'Document Type', Entry.[Document No.] AS 'Document Number', Entry.[G/L Account No.] AS 'G/L Account Number', Entry.[G/L Account No.].Name AS 'Account Name', Entry.Description AS 'Description', Entry.[Debit Amount] AS 'Debit Amount', Entry.[Credit Amount] AS 'Credit Amount', Entry.Amount AS 'Amount', Entry.[Source Code] AS 'Source Code', Entry.[Journal Batch Name] AS 'Journal Batch Name', Entry.[Global Dimension 1 Code] AS 'Global Dimension 1 Code', Entry.[Global Dimension 2 Code] AS 'Global Dimension 2 Code', Entry.[Dimension Set ID] AS 'Dimension Set ID', Entry.[User ID] AS 'User ID', Entry.Reversed AS 'Reversed', Entry.[Reversed by Entry No.] AS 'Reversed By Entry Number', Entry.[Reversed Entry No.] AS 'Reversed Entry Number'
```

| Key | Required | Binds to |
|---|---|---|
| `Document Number` | yes | `G/L Entry.Document No.` |
| `Date From` | yes | `G/L Entry.Posting Date` |
| `Date To` | yes | `G/L Entry.Posting Date` |
## JE2 — their dimensions

```vql
DECLARE [DocumentNumber] TYPE string AS 'Document Number' REQUIRED
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
FROM [G/L Entry] AS Entry
INNER JOIN [Dimension Set Entry] AS Dim ON Entry.[Dimension Set ID] = Dim.[Dimension Set ID]
WHERE Entry.[Document No.] = [DocumentNumber] AND Entry.[Posting Date] BETWEEN [DateFrom] AND [DateTo]
ORDER BY Entry.[Entry No.], Dim.[Dimension Code]
SELECT Entry.[Entry No.] AS 'Entry Number', Entry.[Dimension Set ID] AS 'Dimension Set ID', Dim.[Dimension Code] AS 'Dimension Code', Dim.[Dimension Value Code] AS 'Dimension Value Code'
```

| Key | Required | Binds to |
|---|---|---|
| `Document Number` | yes | `G/L Entry.Document No.` |
| `Date From` | yes | `G/L Entry.Posting Date` |
| `Date To` | yes | `G/L Entry.Posting Date` |
## JE3 — entries by description

```vql
DECLARE [DateFrom] TYPE date AS 'Date From' REQUIRED
DECLARE [DateTo] TYPE date AS 'Date To' REQUIRED
DECLARE [DescriptionText] TYPE string AS 'Description Text' REQUIRED
FROM [G/L Entry] AS Entry
WHERE Entry.[Posting Date] BETWEEN [DateFrom] AND [DateTo] AND Entry.Description LIKE [DescriptionText]
ORDER BY Entry.[Entry No.]
SELECT Entry.[Entry No.] AS 'Entry Number', Entry.[Transaction No.] AS 'Transaction Number', Entry.[Posting Date] AS 'Posting Date', Entry.[Document No.] AS 'Document Number', Entry.[G/L Account No.] AS 'G/L Account Number', Entry.Description AS 'Description', Entry.Amount AS 'Amount', Entry.[Journal Batch Name] AS 'Journal Batch Name'
```

| Key | Required | Binds to |
|---|---|---|
| `Date From` | yes | `G/L Entry.Posting Date` |
| `Date To` | yes | `G/L Entry.Posting Date` |
| `Description Text` | yes | `G/L Entry.Description` |
Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-12 from
bbe59f0. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
