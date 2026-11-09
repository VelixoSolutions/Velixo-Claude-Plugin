# Data pulls

Run these through the Velixo connector's `execute_query` as written, with your
values in the `parameters` map. The query text goes in the `giql` argument whatever its fence says: a
` ```vql ` block goes there too, since `execute_query` has no `vql` argument. Do not call `prepare_query` for them and do not edit the text. If one
fails, report the error and stop rather than reformulating it. Parameter keys are the display labels
(the text after `AS` in each `DECLARE`); a key that matches no label is dropped silently and the
query returns nothing, which looks exactly like "no data". Pass every declared key, an unused optional
one as `null`: a key left out is not always treated as null. Always pass `top` — start at 250 and
raise it if the result comes back flagged `truncated`.

**Dates are `YYYY-MM-DD`.** Sage Intacct has no period parameter here: a month end is its last day.

| Alias | Query | Grain | Used for |
|---|---|---|---|
| E1 | `entities` | one row per entity | entity names and base currencies |
| S1 | `ar-invoices-as-of` | one row per AR invoice or credit memo dated on or before the date | the AR aging, with S2 |
| S2 | `ar-payment-applications-as-of` | one row per invoice with something applied by the date | the AR aging, with S1 |
| S3 | `ap-bills-as-of` | one row per posted AP bill or debit memo posted on or before the date | the AP aging, with S4 |
| S4 | `ap-payment-applications-as-of` | one row per bill with something applied by the date | the AP aging, with S3 |
| G1 | `gl-account-balance-by-journal` | entity × journal × debit or credit | the GL side of the tie; sales, receipts and purchases for ratios |

The shared method files name pulls by their role: **the aging pull** is S1 with S2 (AR) or S3 with
S4 (AP); **the subledger balance by period** and **the activity** come from G1, run at each date
needed (journals ARJ and CRJ for AR, APJ and CDJ for AP); **the GL control account** is G1.

## Which pulls

- **An aging, whatever the date:** S1 and S2 for AR, S3 and S4 for AP, each once, with the same
  `As Of Date`. "Now" is today's date. Run E1 once when the answer names entities or currencies.
- **The GL tie** (asked): G1 once per control account, with the same `As Of Date` and the book.
  The control account is the one the user names, or the chart's "Accounts Receivable" or "Accounts
  Payable" account (12100 and 20100 on Sage's sample companies): confirm it in the basis line.
- **Ratios and a trend** (asked): G1 at each period end and at the day before the window's first
  day; the aging once per period only when the past-due share or CEI is asked for each period.
- **One customer or vendor:** the same pulls, keeping that customer's or vendor's rows.

## S1 — AR invoices as of a date

```vql
DECLARE [AsOfDate] TYPE date AS 'As Of Date' REQUIRED
DECLARE [EntityID] TYPE string AS 'Entity ID'
FROM [accounts-receivable/invoice] AS Invoice
LEFT JOIN [accounts-receivable/invoice-line] AS Line ON Line.invoice.id = Invoice.id
LEFT JOIN [company-config/location] AS Location ON Line.dimensions.location.id = Location.id
WHERE Invoice.invoiceDate <= [AsOfDate] AND Invoice.state <> 'draft' AND ([EntityID] IS NULL OR Invoice.entity.id = [EntityID] OR Invoice.entity.id IS NULL)
GROUP BY Invoice.id, Invoice.invoiceNumber, Invoice.customer.id, Invoice.customer.name, Invoice.invoiceDate, Invoice.dueDate, Invoice.state, Invoice.totalBaseAmount, Invoice.currency.baseCurrency, Invoice.totalTxnAmount, Invoice.currency.txnCurrency, Invoice.entity.id
SELECT Invoice.id AS 'Invoice Record ID', Invoice.invoiceNumber AS 'Invoice Number', Invoice.customer.id AS 'Customer ID', Invoice.customer.name AS 'Customer Name', Invoice.invoiceDate AS 'Invoice Date', Invoice.dueDate AS 'Due Date', Invoice.state AS 'State', Invoice.totalBaseAmount AS 'Total Base Amount', Invoice.currency.baseCurrency AS 'Base Currency', Invoice.totalTxnAmount AS 'Total Transaction Amount', Invoice.currency.txnCurrency AS 'Transaction Currency', Invoice.entity.id AS 'Entity', MAX(Location.entity.id) AS 'Line Entity'
```

| Key | Required | Binds to |
|---|---|---|
| `As Of Date` | yes | `accounts-receivable/invoice.invoiceDate` |
| `Entity ID` | no | `company-config/location.entity.id` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
- **For one entity, pass it as `Entity ID`**; null pulls every entity. The result also holds every
  invoice entered at the top level (no `Entity`): keep one only when its `Line Entity` is the entity.
- `Entity` is the entity the invoice was entered in; when it is empty (an invoice entered at the top
  level), use `Line Entity`. Age and total each entity on its own, in its `Base Currency`.
- `State` is today's state: never filter on it for an aging as of a past date.
- `Due Date` ages the invoice; a credit memo (negative total) is a credit.

## S2 — AR applications as of a date

```vql
DECLARE [AsOfDate] TYPE date AS 'As Of Date' REQUIRED
FROM [accounts-receivable/payment-detail] AS Detail
WHERE Detail.paymentDate <= [AsOfDate] AND Detail.arPayment.state <> 'draft'
GROUP BY Detail.arInvoice.id
SELECT Detail.arInvoice.id AS 'Invoice Record ID', SUM(Detail.baseCurrency.paymentAmount) AS 'Base Payment Amount', SUM(Detail.baseCurrency.inlineAmount) AS 'Base Inline Amount', SUM(Detail.baseCurrency.adjustmentAmount) AS 'Base Adjustment Amount', SUM(Detail.baseCurrency.postedAdvanceAmount) AS 'Base Posted Advance Amount', SUM(Detail.baseCurrency.postedOverPaymentAmount) AS 'Base Posted Overpayment Amount', SUM(Detail.baseCurrency.negativeInvoiceAmount) AS 'Base Negative Invoice Amount', COUNT(*) AS 'Detail Count'
```

| Key | Required | Binds to |
|---|---|---|
| `As Of Date` | yes | `accounts-receivable/payment-detail.paymentDate` |
Applied = the six amount columns added together, matched to S1 on `Invoice Record ID`. An
application whose invoice is not in S1 should not occur; if one does, report it.

## S3 — AP bills as of a date

```vql
DECLARE [AsOfDate] TYPE date AS 'As Of Date' REQUIRED
DECLARE [EntityID] TYPE string AS 'Entity ID'
FROM [accounts-payable/bill] AS Bill
LEFT JOIN [accounts-payable/bill-line] AS Line ON Line.bill.id = Bill.id
LEFT JOIN [company-config/location] AS Location ON Line.dimensions.location.id = Location.id
WHERE Bill.postingDate <= [AsOfDate] AND Bill.state NOT IN ('draft','submitted','partiallyApproved','declined','analyzing') AND ([EntityID] IS NULL OR Bill.entity.id = [EntityID] OR Bill.entity.id IS NULL)
GROUP BY Bill.id, Bill.billNumber, Bill.vendor.id, Bill.vendor.name, Bill.createdDate, Bill.postingDate, Bill.dueDate, Bill.state, Bill.totalBaseAmount, Bill.currency.baseCurrency, Bill.totalTxnAmount, Bill.currency.txnCurrency, Bill.entity.id
SELECT Bill.id AS 'Bill Record ID', Bill.billNumber AS 'Bill Number', Bill.vendor.id AS 'Vendor ID', Bill.vendor.name AS 'Vendor Name', Bill.createdDate AS 'Bill Date', Bill.postingDate AS 'GL Posting Date', Bill.dueDate AS 'Due Date', Bill.state AS 'State', Bill.totalBaseAmount AS 'Total Base Amount', Bill.currency.baseCurrency AS 'Base Currency', Bill.totalTxnAmount AS 'Total Transaction Amount', Bill.currency.txnCurrency AS 'Transaction Currency', Bill.entity.id AS 'Entity', MAX(Location.entity.id) AS 'Line Entity'
```

| Key | Required | Binds to |
|---|---|---|
| `As Of Date` | yes | `accounts-payable/bill.postingDate` |
| `Entity ID` | no | `company-config/location.entity.id` |

Pass every key in this table. An optional key you are not using goes in as `null`; never leave it out of the map.
- **For one entity, pass it as `Entity ID`**; null pulls every entity. The result also holds every
  bill entered at the top level (no `Entity`): keep one only when its `Line Entity` is the entity.
- `Entity` is the entity the bill was entered in; when it is empty, use `Line Entity`.
- Cut off on the **GL posting date**, so the aging ties to the GL; `Bill Date` is beside it and is
  the date to age by document date.
- Bills not yet posted (draft, submitted, partially approved, declined, analyzing) are left out:
  count them for the payment plan only when the user asks for bills awaiting approval.

## S4 — AP applications as of a date

```vql
DECLARE [AsOfDate] TYPE date AS 'As Of Date' REQUIRED
FROM [accounts-payable/payment-detail] AS Detail
WHERE Detail.paymentDate <= [AsOfDate] AND Detail.apPayment.state <> 'draft'
GROUP BY Detail.bill.id
SELECT Detail.bill.id AS 'Bill Record ID', SUM(Detail.baseCurrency.paymentAmount) AS 'Base Payment Amount', SUM(Detail.baseCurrency.discountAmount) AS 'Base Discount Amount', SUM(Detail.baseCurrency.adjustmentAmount) AS 'Base Adjustment Amount', SUM(Detail.baseCurrency.inlineAmount) AS 'Base Inline Amount', SUM(Detail.baseCurrency.postedAdvanceAmount) AS 'Base Posted Advance Amount', COUNT(*) AS 'Detail Count'
```

| Key | Required | Binds to |
|---|---|---|
| `As Of Date` | yes | `accounts-payable/payment-detail.paymentDate` |
Applied = the five amount columns added together, matched to S3 on `Bill Record ID`. Draft payments
are left out.

## G1 — GL control account by journal

```vql
DECLARE [AccountNumber] TYPE string AS 'Account Number' REQUIRED
DECLARE [AsOfDate] TYPE date AS 'As Of Date' REQUIRED
DECLARE [BookId] TYPE string AS 'Book ID' REQUIRED
FROM [general-ledger/journal] AS Journal
INNER JOIN [general-ledger/journal-entry] AS Entry ON Entry.glJournal.id = Journal.id
INNER JOIN [general-ledger/journal-entry-line] AS Line ON Line.journalEntry.id = Entry.id
INNER JOIN [company-config/location] AS Location ON Line.dimensions.location.id = Location.id
WHERE Journal.bookId = [BookId] AND Line.glAccount.id = [AccountNumber] AND Line.state = 'posted' AND Line.entryDate <= [AsOfDate]
GROUP BY Location.entity.id, Journal.id, Line.txnType
SELECT Location.entity.id AS 'Entity ID', Journal.id AS 'Journal ID', Line.txnType AS 'Debit or Credit', SUM(Line.baseAmount) AS 'Base Amount', COUNT(*) AS 'Line Count'
```

| Key | Required | Binds to |
|---|---|---|
| `Account Number` | yes | `general-ledger/journal-entry-line.glAccount.id` |
| `As Of Date` | yes | `general-ledger/journal-entry-line.entryDate` |
| `Book ID` | yes | `general-ledger/journal.bookId` |
- **Book:** `Accrual` unless the user named another. Never leave the book out: the line table holds
  every book, and an unscoped read counts the main book's postings once per book.
- **Subledger, both sides of each journal:** AR is (ARJ debits − ARJ credits) + (CRJ debits − CRJ
  credits), so credit memos and voided receipts count; AP is (APJ credits − APJ debits) + (CDJ credits
  − CDJ debits), so debit memos and voided payments count. Every other
  journal on the account (OBJ opening balances, GJ manual entries, others) is its own line.
- `Entity ID` is the entity of each line's location, so child locations count in their entity.
- `Base Amount` is in the entity's base currency on every row; `Currency` names that base
  currency.

## When the aging and the GL disagree

Report both figures per entity and the difference as unexplained, then the possibilities seen on
Sage Intacct, as possibilities:

1. **Receipts or credits posted outside the control account.** A cash receipt credited to another
   account (Skyline's CRJ 56 credits 500.00 of customer receipts to 20800 Other Payables) is a
   customer credit in the aging and never reaches the AR account.
2. **Backdated entries.** The aging dates a receipt by its receipt date, the GL by its posting date:
   a receipt keyed later with an earlier date sits on different sides of the cut-off.
3. **AR adjustments.** An adjustment posted to the GL is not an invoice, so it is in neither the
   rebuild nor Sage's own aging.
4. **Opening balances and manual entries** (OBJ, GJ) on the control account: shown on their own lines,
   never inside the subledger.
5. **Open advances** (AP) are not bills and are not on the AP account.

Every query block above (`giql` for Acumatica, `vql` for Sage Intacct and Business Central) was
inserted at build time from Velixo's validated query library, built 2026-11-09 from
80e40ff. The notes beside each block are the caveats that matter when running it. When a
result looks wrong, say so with the query's name and the figures, and never edit the query to make
it fit: a changed query is no longer the validated one.
