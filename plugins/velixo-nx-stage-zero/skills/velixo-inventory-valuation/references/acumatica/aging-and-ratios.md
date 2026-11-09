# Aging and ratios

## Aging

- **Age** = months from `Last Activity Period` to the as-of period: `062025` against `122024` is 6
  months. Count calendar periods, leaving out `13YYYY` adjustment periods.
- **Bands** by default: 0–3, 4–6, 7–12, 13–24, over 24 months. The user may ask for other cut-offs.
- **What it measures:** the last period in which the item's cost history moved, for any reason —
  receipt, issue, transfer, adjustment. An item issued last month and received three years ago has
  aged one month. When the user asks for age since the last receipt, say this query does not give it
  and that the receipts are in the inventory document lines.
- **Across warehouses:** one row per item, so an item that moved in one warehouse is young in all.
  When the user needs the age per warehouse, say so and offer the detail level of the valuation,
  which has no age.
- **Foot:** the bands add to the aging total; the aging total equals the valuation summary for the
  branch and period, less items with cost and no quantity.

## Ratios

| Ratio | Default formula | Notes |
|---|---|---|
| **DIO** (days inventory outstanding) | average inventory ÷ cost of sales × days in period | Average inventory = (opening + ending) ÷ 2, opening being the previous period's I1 total. Cost of sales from C1, on the accounts the user confirmed. |
| **Inventory turnover** | cost of sales × (365 ÷ days in period) ÷ average inventory | Annualised. 365 ÷ turnover is DIO on an annual basis. |
| **Share unmoved over 12 months** | value in bands over 12 months ÷ total value | From the aging. |

**Traps.**

- **A period with no cost of sales** has no DIO: show "n/a", never a division by zero.
- **Cost of sales on the GL includes what the inventory subledger does not hold** (freight, landed
  cost variances, non-stock items, services) on many charts. Name the accounts used and say the ratio
  is on them.
- **Branches.** Compute per branch and for the total; the total is the sum of balances and of cost of
  sales, never an average of ratios. Cost of sales is posted by the document's branch and inventory
  is valued by the warehouse's branch: when they differ, the total is the reliable figure.
- **Rounding.** One decimal for days and turns; compute from unrounded figures.
