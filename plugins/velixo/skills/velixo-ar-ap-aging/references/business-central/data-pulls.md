# Data pulls

Run these through the Velixo connector's `execute_query` as written, with your
values in the `parameters` map. The query text goes in the `giql` argument whatever its fence says: a
` ```vql ` block goes there too, since `execute_query` has no `vql` argument. Do not call `prepare_query` for them and do not edit the text. If one
fails, report the error and stop rather than reformulating it. Parameter keys are the display labels
(the text after `AS` in each `DECLARE`); a key that matches no label is dropped silently and the
query returns nothing, which looks exactly like "no data". Pass every declared key, an unused optional
one as `null`: a key left out is not always treated as null. Always pass `top` — start at 250 and
raise it if the result comes back flagged `truncated`.

**Dates are `YYYY-MM-DD`.** Business Central has no period parameter here: a month end is its last
day. Every amount is in the company's local currency.

| Alias | Query | Grain | Used for |
|---|---|---|---|
| G0 | `gl-setup` | one row | the local currency's code |
| S1 | `ar-open-entries-as-of` | one row per customer document with something left on the date | the AR aging |
| S2 | `ap-open-entries-as-of` | one row per vendor document with something left on the date | the AP aging |
| C1 | `customer-posting-groups` | one row per customer posting group | the receivables account of each group |
| C2 | `vendor-posting-groups` | one row per vendor posting group | the payables account of each group |
| G1 | `gl-account-balance-at-date` | one control account at a date | the GL side of the tie; sales, receipts and purchases for ratios |

The shared method files name pulls by their role: **the aging pull** is S1 (AR) or S2 (AP); **the
subledger balance by period** is the aging pull run at each period end; **the activity** and **the
GL control account** come from G1, run at each date needed.

## Which pulls

- **An aging, whatever the date:** S1 for AR, S2 for AP, once each, with `As Of Date`. "Now" is
  today's date. Run G0 once to name the currency.
- **The GL tie** (asked): C1 or C2 once, then G1 once per control account they name, with the same
  `As Of Date`.
- **Ratios and a trend** (asked): G1 at each period end and at the day before the window's first
  day; the aging once per period only when the past-due share or CEI is asked for each period.
- **One customer or vendor:** the same pulls, keeping that customer's or vendor's rows.

## G0 — General Ledger Setup

```vql
FROM [General Ledger Setup] AS Setup
SELECT Setup.[LCY Code] AS 'LCY Code', Setup.[Global Dimension 1 Code] AS 'Global Dimension 1 Code', Setup.[Global Dimension 2 Code] AS 'Global Dimension 2 Code', Setup.[Shortcut Dimension 3 Code] AS 'Shortcut Dimension 3 Code', Setup.[Shortcut Dimension 4 Code] AS 'Shortcut Dimension 4 Code', Setup.[Shortcut Dimension 5 Code] AS 'Shortcut Dimension 5 Code', Setup.[Shortcut Dimension 6 Code] AS 'Shortcut Dimension 6 Code', Setup.[Shortcut Dimension 7 Code] AS 'Shortcut Dimension 7 Code', Setup.[Shortcut Dimension 8 Code] AS 'Shortcut Dimension 8 Code', Setup.[Allow Posting From] AS 'Allow Posting From', Setup.[Allow Posting To] AS 'Allow Posting To'
```

No parameters.
## S1 — Customer documents open at a date

```vql
DECLARE [AsOfDate] TYPE date AS 'As Of Date' REQUIRED
FROM [Cust. Ledger Entry] AS Entry
WHERE Entry.[Posting Date] <= [AsOfDate] AND Entry.[Date Filter] <= [AsOfDate] AND Entry.[Remaining Amt. (LCY)] <> 0
ORDER BY Entry.[Customer No.], Entry.[Due Date]
SELECT Entry.[Customer No.] AS 'Customer Number', Entry.[Customer No.].Name AS 'Customer Name', Entry.[Entry No.] AS 'Entry Number', Entry.[Document Type] AS 'Document Type', Entry.[Document No.] AS 'Document Number', Entry.[Posting Date] AS 'Posting Date', Entry.[Due Date] AS 'Due Date', Entry.[Currency Code] AS 'Currency Code', Entry.[Original Amt. (LCY)] AS 'Original Amount (LCY)', Entry.[Remaining Amt. (LCY)] AS 'Remaining Amount (LCY) As Of Date', Entry.[Customer Posting Group] AS 'Customer Posting Group'
```

| Key | Required | Binds to |
|---|---|---|
| `As Of Date` | yes | `Cust. Ledger Entry.Date Filter` |
- The date filter makes the remaining amount the one on the date: a document paid after it shows
  at what was left then. Never add a filter on `Open`, which is today's state.
- `Due Date` ages the document. `Customer Posting Group` is how it reaches the GL (C1).

## S2 — Vendor documents open at a date

```vql
DECLARE [AsOfDate] TYPE date AS 'As Of Date' REQUIRED
FROM [Vendor Ledger Entry] AS Entry
WHERE Entry.[Posting Date] <= [AsOfDate] AND Entry.[Date Filter] <= [AsOfDate] AND Entry.[Remaining Amt. (LCY)] <> 0
ORDER BY Entry.[Vendor No.], Entry.[Due Date]
SELECT Entry.[Vendor No.] AS 'Vendor Number', Entry.[Vendor No.].Name AS 'Vendor Name', Entry.[Entry No.] AS 'Entry Number', Entry.[Document Type] AS 'Document Type', Entry.[Document No.] AS 'Document Number', Entry.[External Document No.] AS 'External Document Number', Entry.[Posting Date] AS 'Posting Date', Entry.[Due Date] AS 'Due Date', Entry.[Currency Code] AS 'Currency Code', Entry.[Original Amt. (LCY)] AS 'Original Amount (LCY)', Entry.[Remaining Amt. (LCY)] AS 'Remaining Amount (LCY) As Of Date', Entry.[Vendor Posting Group] AS 'Vendor Posting Group'
```

| Key | Required | Binds to |
|---|---|---|
| `As Of Date` | yes | `Vendor Ledger Entry.Date Filter` |
- Read as S1, with every sign the other way round: an invoice is negative.
- `External Document Number` is the vendor's own invoice number: show it in a payment plan.

## C1 and C2 — Posting groups and their control accounts

```vql
FROM [Customer Posting Group] AS PostingGroup
SELECT PostingGroup.Code AS 'Code', PostingGroup.Description AS 'Description', PostingGroup.[Receivables Account] AS 'Receivables Account'
```

No parameters.
```vql
FROM [Vendor Posting Group] AS PostingGroup
SELECT PostingGroup.Code AS 'Code', PostingGroup.Description AS 'Description', PostingGroup.[Payables Account] AS 'Payables Account'
```

No parameters.
Several groups can post to one account (`EU` and `FOREIGN` both to 2320 on the CRONUS demo company):
tie per account, summing every group that posts to it.

## G1 — One control account at a date

```vql
DECLARE [AccountNumber] TYPE string AS 'Account Number' REQUIRED
DECLARE [AsOfDate] TYPE date AS 'As Of Date' REQUIRED
FROM [G/L Account] AS Account
WHERE Account.[No.] = [AccountNumber] AND Account.[Account Type] = 'Posting' AND Account.[Date Filter] <= [AsOfDate]
SELECT Account.[No.] AS 'Account Number', Account.Name AS 'Name', Account.[Balance at Date] AS 'Balance at Date', Account.[Debit Amount] AS 'Debit Amount', Account.[Credit Amount] AS 'Credit Amount'
```

| Key | Required | Binds to |
|---|---|---|
| `Account Number` | yes | `G/L Account.No.` |
| `As Of Date` | yes | `G/L Account.Date Filter` |
- `Balance at Date` is signed debit-positive: receivables positive, payables negative, as S1 and S2
  read.
- `Debit Amount` and `Credit Amount` are cumulative to the date: a period's activity is the
  difference between two dates.

## When the aging and the GL disagree

Report both figures per control account and the difference as unexplained, then the possibilities
seen on Business Central, as possibilities:

1. **A manual entry on the control account.** A general journal line posted straight to the
   receivables or payables account (source code `GENJNL`, no customer or vendor) moves the GL and no
   ledger entry. The account drill-down (`velixo-account-drilldown`), by source, shows it as entries
   with no customer or vendor; offer it as the next step.
2. **A posting group whose account changed.** C1 and C2 name today's account; a document posted
   before the change sits on the old one.
3. **Exchange-rate adjustments** on foreign-currency documents post to the control account and to
   the documents' detailed entries at their own date; a tie on a date between two adjustment runs can
   part on them.
4. **Direct posting allowed on the account.** When the account accepts direct posting, any journal
   can hit it.

Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-09 from
0dc8dcb. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
