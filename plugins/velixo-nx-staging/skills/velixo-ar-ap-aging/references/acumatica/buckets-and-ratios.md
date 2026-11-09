# Buckets and ratios

## The open balance of a document

- **`Open Balance`** (A1, P1) is already signed; add it as it comes. AR: invoices, debit memos and
  overdue charges positive; credit memos, payments, prepayments, voided payments and balance
  write-offs negative. AP: bills and credit adjustments positive; debit adjustments, prepayments and
  payments negative. It is `Amount` + `Adjusting Amount` + `Adjusted Amount`, the three columns
  beside it.
- **Drop rows whose open balance is 0.** A1 and P1 return documents still flagged open with nothing
  left on them; count them for the clean-up list, keep them out of the aging.
- **Base currency only.** Never add `Balance` (document currency) across documents.
- **As-of date when the period calendar was read** (periods that are not calendar months): the
  calendar's `End Date` minus one day, because `End Date` is the next period's first day.
- **Where the ratio inputs come from:** ending AR or AP is the history's `Balance` (A2, P2); period
  sales and purchases are `Period Sales` (A3) and `Period Purchases` (P3); days in the period come
  from the calendar.

- **Acumatica specifics for the traps below:** adjustment periods are `13YYYY`; retainage is
  `RetainedBalance`, outside `Balance`; deposits are netted out of `Balance`, and a customer deposit
  or vendor prepayment account comes back on its own row of A2 or P2.

## Buckets

- **As-of date:** the date the aging was run as of: the last day of the as-of period (June 2025
  ends on 30 Jun 2025), the date the user gave, or today for an aging "as of now".
- **Days past due** = as-of date − due date. Zero or less is **Current** (not yet due, or due on the
  as-of date). Then 1–30, 31–60, 61–90, over 90 by default, or the user's cut-offs.
- **Documents with no due date** (AR credit memos, payments and prepayments; AP payments and
  prepayments) go in a **Credits** column, not in Current: they are amounts to apply, not amounts that
  fall due.
- **Aging by document date** instead, when the user asks: days = as-of date − document date, and
  credits are aged too. Say which basis the table uses.
- **Check:** buckets + credits = the net total. In a reconciliation, that total is compared with the
  subledger balance and the GL control account for the same branch or entity and date.

## Ratios

State each ratio's basis in the output. The defaults below are what the skill uses unless the user
asks for another.

| Ratio | Default formula | Notes |
|---|---|---|
| **DSO** (days sales outstanding) | ending AR ÷ period sales × days in period | Ending AR and period sales from the pulls `references/data-pulls.md` names for ratios. Offer the average-balance form (average of opening and ending AR) and a three-period form (sum of three periods' sales over their days) when sales are seasonal. |
| **Best possible DSO** | Current AR ÷ period sales × days in period | Current = not yet due, from the aging's buckets. Needs the aging for the period. |
| **Average days delinquent** | DSO − best possible DSO | The part of DSO caused by late payment rather than terms. |
| **CEI** (collection effectiveness index) | (opening AR + period sales − ending AR) ÷ (opening AR + period sales − ending Current AR) × 100 | Opening AR is the previous period's ending AR. 100 means every collectible amount was collected; below 80 is usually a concern, but say the user's own benchmark governs. Needs the aging for the period's Current AR. |
| **AR turnover** | period sales × (365 ÷ days in period) ÷ average AR | Annualised. 365 ÷ turnover is the average-balance DSO. |
| **% past due**, **% over 90** | past-due (or over-90) balance ÷ total debit balances | Credits are left out of the denominator, so a large unapplied payment does not flatter the share. |
| **Concentration** | top 10 customers' (vendors') balances ÷ total | Also the largest single balance and its share. |
| **DPO** (days payable outstanding) | ending AP ÷ period purchases × days in period | Purchases are the period's AP bills. Many companies compute DPO on cost of goods sold instead: offer it from the GL (cost of sales accounts the user names) and label it. |
| **AP turnover** | period purchases × (365 ÷ days in period) ÷ average AP | Annualised. |
| **% AP past due** | past-due bills ÷ total bills | From the AP aging. |

**What these ratios need that the subledger does not give.** Credit sales: the subledger's period sales
include cash sales (paid at once, also in the period's payments), so they slightly overstate credit
sales and understate DSO; say so, and when cash sales are material, take them out using the AR
register.
The cash conversion cycle (DSO + days inventory outstanding − DPO) needs inventory and cost of sales
from the GL; this skill does not compute days inventory.

**Traps.**

- **A period with no sales or purchases** has no DSO or DPO: show "n/a", never a division by zero
  or an infinite figure.
- **Adjustment periods** carry the year-end balance and no activity: leave them out of a monthly
  trend.
- **Retainage**, where the ERP tracks it, is not in the subledger balance. Exclude it from DSO (it is
  not collectible on terms) and show it on its own line.
- **Deposits and prepayments** sit on their own accounts. Keep the control account's figures for the
  ratios.
- **Branches or entities.** Compute per branch or entity and for the total; the total is the sum of
  balances and of sales, never an average of the ratios. Never add entities with different base
  currencies.
- **Rounding.** One decimal for days, one for percentages; compute from unrounded figures.