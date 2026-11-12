# Reading a business-account-by-month matrix

The business-account matrix is F4 then F5, every month in one call (D1 once per month when it falls back), and the lines pull is D3 (`data-pulls.md`).

Build three views from the business-account pull (`Natural Net`, with the debits, credits and line
count kept beside it) before writing anything:

1. **Account (or account class) matrix:** total by month (columns) and year (rows), plus change rows against each
   comparison year.
2. **Business account table:** one row per business account with each window's total and the change, sorted by
   change against the main comparison year. Add first-seen and last-seen periods from the full window.
3. **Driver lines:** month-by-month across the whole window for the three to five business accounts that
   explain most of the change, plus the no-business-account line.

Then go through the patterns below. Each one says what it usually means and what would confirm it.
Most apply to any business account type; where the reading differs by account type, it says so.

## Composition

- **No-business-account share.** The share of the change on lines with no business account. When it is large,
  it is the headline: the ledger can't attribute it, so no business-account-level conclusion covers it.
  Recommend that card imports, bank feeds and journals to the account require a business account, then
  re-run.
- **Clearing and reimbursement channels.** A clearing account, a reimbursement account, a payment
  processor or a card issuer as the business account is a channel, not the real party. If material, look
  through it with the lines pull to the underlying descriptions before drawing conclusions.
- **Concentration.** If one or two business accounts make up most of the change, the investigation is
  about them and the rest can be summarised. On revenue, concentration is also a risk finding in its
  own right: say what share of the account the top one, three and ten business accounts hold.

## Business account life cycle

- **New.** First seen inside or between the windows. Report the first-seen period and the monthly
  amount since. On expense, a cluster of new suppliers in one quarter is usually one initiative; on
  revenue, new customers are the "new" component of the customer bridge.
- **Lost or ended.** Last seen before the current window. On expense, a saving, or a supplier that moved
  to a channel with no reference (check the no-business-account line for a matching amount from the same
  month). On revenue, a lost customer: report it on its own line, never netted against new ones.
- **Stopped mid-window.** Activity that ends partway through the current window: cancelled, moved to
  another channel, churned, or an accrual or invoice not yet booked. Ask before calling it final.
- **Revenue bridge.** For customer accounts, report four figures, not one net: new, lost, expanded,
  contracted.

## Amount patterns

- **Step.** A new monthly level that holds: price, tier, volume, seats, headcount, a new contract.
  Date the step; that date is the most useful fact you can give the owner.
- **Same count, higher amount** (from the line count): price or size per document. **More lines at a
  similar amount:** more subscriptions, more orders, more claims, more people.
- **Irregular or annual billing.** Large amounts in one or two months and nothing else. On expense, if
  the service covers a year it should usually be prepaid and amortised; on revenue, an annual invoice
  usually needs deferral. Flag it as a recognition question, not a spike.
- **Credits netting invoices.** Recurring credits beside debits (discounts, partner credits, refunds,
  credit memos). When the credits stop, the net jumps with no change in the invoice; when they grow,
  check returns, disputes or concessions.
- **One-off spike.** A single month far above the business account's level. Open the documents (the lines pull) and
  look for catch-up billing, a duplicate, a one-time purchase or sale.
- **Per-person activity** (employee accounts). Claims or advances concentrated on a few people, or a
  person whose monthly level stepped up, are review points for the expense policy.
- **Contract escalation.** Rent and service contracts stepping at the same month each year by a fixed
  percent. Expected; call it in line with the contract only if the contract has been seen.

## Wording

State patterns as observations with their dates and amounts ("rose from about 1.3K to about 6.6K a
month from June 2025"). Put the cause in a separate sentence and label it as a hypothesis until a
document or the owner confirms it.