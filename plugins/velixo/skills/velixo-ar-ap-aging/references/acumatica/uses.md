# The uses

On Acumatica the aging pull is A1 or P1 and the subledger balance by period is A2 or P2's `Balance`
(`data-pulls.md`). A1 and P1 also return documents still flagged open with a zero balance: they
belong in the clean-up list. Deposit and prepayment rows sit on their own `ARAccountID` or
`APAccountID`. Customers or vendors the connection's user may not see are in the GL and in
Acumatica's own aged report (AR630500, AP630500), but in neither the aging nor the history.

Each use answers a different question. The aging report, the collections list, the payment plan,
cash timing and the reserve read the aging query alone, once; reuse its rows when the user moves from
one to the next ("now show me who to chase"). Ratios, the reconciliation and the clean-up run more
pulls, and only when the user asks for them.

## Aging report

**Question.** How much is open, and how old is it?

**Output.** One row per customer (AR) or vendor (AP) with a balance, columns for each bucket and the
total; a total row; the share of the total in each bucket. Lead
with the total, the amount past due and its share, and the amount over 90 days. Credits (credit
memos, unapplied payments, prepayments, debit adjustments) net within the customer or vendor and are
also totalled in their own column, so a large credit is not hidden inside a small net balance.

**As of a past period or date.** The aging pull rebuilds the balance at that period
end or date, so a document paid since appears with what was then open. Say "as it stood on 30 Jun
2025", not "today", and say whether the cut-off was the period or the date.

## Collections list (AR)

**Question.** Whom do we chase, for what, and in which order?

**Output.** Customers with a past-due balance, ranked by amount over 60 days, then over 30, then past
due. For each: past-due amount, the oldest open invoice (number, due date, days past due), the number
of past-due invoices, unapplied credits they hold (apply before chasing), and the share of total AR.
Top 10 by default; the rest on request. Never add a judgement of the customer's ability to pay.

Useful additions when asked: customers whose past-due share grew since the previous period (run the
aging for both periods), and customers with credit balances only (refund or apply).

## Payment plan (AP)

**Question.** What do we pay, when, and what is already overdue?

**Output.** Bills already past due first (by vendor, oldest first), then bills due in the next 7, 14
and 30 days from the as-of date, each with vendor, bill number, due date and amount; vendor credits
(debit adjustments, prepayments) beside the vendor so they are applied, not paid over. Totals per
window. The aging as of now is the right pull: a payment plan is about now.

The aging query carries no discount date, so early-payment discounts are not visible here; say so if
the user asks about them.

## Cash timing

**Question.** When should cash come in (AR) or go out (AP)?

**Output.** Open balances by the week or month in which they fall due, from the as-of date: overdue
(a separate line, never assumed collected in week one), then each week. This is contractual timing:
it assumes payment on the due date. When the user wants it adjusted, use the observed pattern from
the ratios (DSO against terms, or average days past due) and state the adjustment. Never present the
contractual schedule as a forecast.

## Ratios and trend

**Question.** How fast do we collect and pay, and is it getting better or worse?

**Output.** One row per period: balance, period sales or purchases, DSO or DPO, turnover, and, when
the aging is pulled for the period, the share past due and over 90 days and the CEI. Formulas
and traps are in `buckets-and-ratios.md`. Name the basis of every ratio (ending or average balance,
days in the period, what counts as sales), because the same name means different formulas in
different companies.

## Reconciliation to the GL

**Question.** Does the subledger equal the GL control account?

**Output.** Per branch and control account, for each period asked: GL balance, subledger total and
the difference. For one period the subledger total is the aging's; for several it is the subledger balance by
period (`references/data-pulls.md` says which pull), which equals the aging by construction where the
ERP keeps one. Retainage, where the ERP tracks it, is outside the subledger total: show it on its own
line when the user asks for it. Say whether the difference is constant or moves,
and name the periods where it moves, as an observation: it does not say where the difference comes
from. The usual sources are postings to the control account from outside the AR or AP module (opening
balances, manual journal entries), subledger documents posted to another account, documents dated and
posted on different sides of the cut-off, and customers or vendors the connection is not allowed to
see. `references/data-pulls.md` lists the ones this ERP is known for. Which one it is needs the GL
lines of the control account in those periods (the account drill-down skill for this ERP, by business
account) or the ERP's own aged report compared customer by customer: name both as next steps, and
none as the cause. Also show deposit and prepayment accounts on their own lines: they are often
expected on separate GL accounts.

## Clean-up

**Question.** What in the subledger needs attention, apart from collecting and paying?

**Output.** Lists, each with count and total: credit balances by customer or vendor; unapplied
payments, prepayments and credit memos; documents over a year past due; documents with a zero balance
still flagged open (an application or a status to fix); customer deposits and retainage outstanding,
where the ERP tracks them. Each list names who would act (AR clerk, AP
clerk, controller) without acting.

## Reserve (allowance for doubtful accounts)

**Question.** What allowance do these buckets imply?

**Output.** Bucket totals × the user's rate for each bucket = the implied reserve, against the
reserve balance on the GL when the user names that account. **The rates come from the user** (policy
or history): never supply typical percentages as if they were the company's. Specific reserves for
named customers are the user's call; list the largest over-90 balances to inform it.