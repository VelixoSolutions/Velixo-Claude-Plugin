# Drilling down by subaccount segment

The second axis. A business account breakdown says **who** the activity was with; a segment breakdown says **where the
organization coded it**: department, cost center, project, region, whatever each segment means in this
tenant. Together they say whether a movement has an owner.

## Build it

1. **Structure.** Run D0. Record each segment's order, length, description and value labels.
2. **Breakdown across windows.** F3, one invocation per account and window, the subaccount argument a
   list of patterns, one per value of the segment from D0 (`[["CON???"],["ELE???"]]`): each row is one
   value's total, already split. Every window of every account goes in one call (20 invocations a
   call). One table per segment: value, label, each window's total, change. `"#"` in place of the
   patterns keeps the full-subaccount table for combinations that matter. When F3 fails, D6 (every
   branch in scope) or pull 5 (a named branch) at subaccount grain, split client-side.
3. **Cross-tab with business accounts.** Run D2 (with `Branch ID` for one branch, or without it for
   the whole ledger) for the current window (and the comparison window when a business account's
   coding may have changed). Pivot business account × segment value for the material business
   accounts and the unassigned bucket.
4. **Tie out.** Each segment table sums to the account or account class total from F1 for every
   window (a shortfall is a segment value the pattern list missed), and D2 sums to F3 (or D6, or pull
   5) per account. A difference stops the run.

## Read it

- **Unassigned share first.** The share of activity, and of the change, in the unassigned bucket of each
  segment is a coding-coverage finding in its own right. When most of the movement is unassigned, the
  segment can't explain it: say so up front, and use the cross-tab to name the business accounts that carry
  no coding (they're the ones to fix, usually by setting a default subaccount on the vendor, customer
  or employee record, or a rule on the card or bank feed).
- **New values.** A segment value with no activity in the comparison window and activity now is usually a new
  initiative (an event, a compliance programme, a new office). Name it with its label; check whether the
  cross-tab shows one business account behind it.
- **Concentrated or spread.** From the root-cause evidence ladder: a change concentrated in one value has
  an owner to ask; a change spread evenly across values is a rate, a policy or an allocation and is not
  any one owner's. Report which it is.
- **Moved between values.** The same business account in a different value than last year is a recoding, not a
  change in activity: net it out before calling a department up or down.
- **Two segments, two stories.** A department that rose because of one cost center (an event, a
  project) is a different finding from a department whose run-rate rose. Read the combination table
  when both segments carry activity.

## Margin by segment

When the flux question is margin — gross margin moved, or revenue and cost of sales moved differently —
read revenue and cost on the same segment value instead of each account on its own.

1. **Decode first.** Run D0 and label every value from its `Value Description` before naming one in the
   output. Never infer what a value means from the item descriptions on its lines: a product-group code full
   of oven lines is still whatever D0 says it is, and an `<Uncategorized>` or all-zero value is the unassigned
   bucket whatever sits in it.
2. **Pull both sides in one call.** F3 for the branch on each revenue and cost-of-sales account (the
   accounts the user confirmed as trading revenue and cost of sales), one invocation per account and
   window, the segment's D0 values as patterns, every invocation in one call. D6b once per window
   when F3 fails.
3. **Pair on the segment value.** For each value of the segment that carries the product or line of
   business: revenue, cost, gross margin (revenue less cost, both `Natural Net`) and margin percent, per
   window, and the change. Keep the full-subaccount pairing when two segments both carry activity.
4. **Tie out.** Revenue and cost each sum, over the values, to F1 (or pull 5, or pull 4b) for the
   account and window. A value with revenue and no cost, or cost and no revenue, is its own row: cost coded to the
   unassigned bucket while revenue is coded to a product group is a coding finding, not a margin.
5. **Then go to the lines that move it.** Where one value's margin moved, D12 for that branch and window
   shows which invoice lines and items carry the change, with quantity invoiced against quantity issued.

## Against budget

When the scope carries a budget, run D4 for the current window (`Ledger ID` set to the budget
ledger) and split its `Subaccount` the same way. Each segment table gains a budget column and a
variance, actual `Natural Net` less budget `Natural Net`.

- **Label the variance by account type, never by sign alone:** more revenue than budget is
  favourable; more expense is not.
- **One side missing is a row, not a gap.** A value with budget and no actual is "budgeted, no
  activity"; a value with actual and no budget is "unbudgeted". Neither is dropped.
- **The unassigned bucket is compared like any other value.** Budget loaded against the unassigned
  code and spent in departments is a coding finding, not an overspend; say which before calling any
  department over budget.
- **Evenly spread budget lines** make a monthly variance measure the spread; compare the window
  cumulatively and say so.
- **Business accounts have no budget line.** The comparison stops at the segment and branch axes; the
  business account × segment cross-tab explains an actual, not a variance.

## Splitting in the query instead

GIQL can group on a segment directly: `{=Left([Sub.SubCD], n)}` or
`{=Substring([Sub.SubCD], offset, n)}` in `SELECT` and `GROUP BY`, with `SegmentValue` joined on
`SV.Value = {=Left([Sub.SubCD], n)}` (the function on the right-hand side of the comparison) for the
label. The lengths are tenant-specific, so such a query cannot be validated once and run as written;
it needs a `prepare_query` rebuild from D2 for each tenant. Reach for it only when a result is too
large to split client-side, and tie the result to F1 as usual. F3's patterns make this unnecessary in
most cases: the split happens in the function.

## Report it

Add to the drill-down output, in the user's nesting order:

- one table per segment (value, label, windows, change, and budget and variance where in scope),
  unassigned bucket first, with its share of the account or account class and of the change;
- the business account × segment cross-tab for material business accounts, flagging those coded to the
  unassigned bucket;
- a sentence per new segment value, with the business account behind it.

