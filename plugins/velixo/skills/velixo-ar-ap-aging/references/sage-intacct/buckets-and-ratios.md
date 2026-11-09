# Buckets and ratios

## The open amount of a document

- **Open = `Total Base Amount` (S1 or S3) less the applied amounts (S2 or S4)** for the same
  `Invoice Record ID` or `Bill Record ID`. The applied amounts are every amount column of S2 or S4
  added together; an empty one is zero, and a document with no row had nothing applied.
- **Credit memos and debit memos** come back as documents with a negative total: they are credits.
- **Drop documents whose open amount is 0.** Count them only for the clean-up list.
- **Base currency only.** Never add `Total Transaction Amount` across documents.
- **Where the ratio inputs come from:** ending AR or AP is the aging total (or G1's subledger
  balance); period sales are ARJ debits less ARJ credits on the AR account in the period (sales net of
  credit memos), and collections CRJ credits less CRJ debits; purchases are APJ credits less APJ
  debits on the AP account. G1 is cumulative, so a period's figure is
  G1 at the period's last day less G1 at the day before its first. Days in the period are calendar
  days.

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