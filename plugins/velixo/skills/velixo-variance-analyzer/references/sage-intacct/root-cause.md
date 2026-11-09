# Root-cause investigation

Getting from "the rate component moved" to something a person can act on. Read at Step 5.

## The standard of proof

A cause is a claim about something that happened. It needs a record: period history, a posted
document, a breakdown across a business dimension, or a person who owns the line and was asked.

Everything else is a hypothesis. Hypotheses are useful — they direct the next query — but they are
labelled as hypotheses, and an item with no evidence behind it is reported as unexplained together
with what would settle it. A plausible reason attached to an unexplained variance is worse than no
reason: it closes the investigation, it survives into the commentary, and it eventually gets repeated
in a board meeting by someone who believes it was established.

"Likely driven by" is acceptable only when the thing that makes it likely is stated in the same
sentence.

## The evidence ladder

Work it in order. Each rung is cheaper than the one below it and frequently ends the investigation.

### 1. History — is this new?

The first question and the cheapest answer. Put the line's last several periods side by side against
the same baseline.

Four shapes, four different investigations:

- **A spike** — one period out of line, others normal. Go straight to transaction detail; a spike is
  almost always one or a few documents.
- **A step** — the line moved to a new level and stayed. Look for a dated event at the step: a
  contract renewal, a hire, a site opening, a rate review, a policy change. The date of the step is
  the single most useful fact you can establish.
- **A drift** — a small movement in the same direction for several periods. Usually a rate, a mix or a
  volume trend, and usually the finding that matters most, because it is the one that compounds and
  the one nobody notices in a single period's review.
- **Noise** — movement in both directions of similar size. Often the correct answer is that there is
  nothing here and the materiality rule caught a naturally volatile line. Say so and move on; this is
  a legitimate outcome, not a failed investigation.

Where several periods of history are available, check the same shape on the baseline as well. A
variance can open because the actual moved, because the baseline moved, or because a prior-period
correction moved one of them retrospectively.

### 2. Transaction detail — what actually posted?

Where inference becomes evidence. Look at the documents behind the line for the period: date, type,
counterparty, description, dimension coding, amount.

**Rungs 2 and 3 are one skill.** After history, run the account drill-down skill for this ERP
(`velixo-account-drilldown`) on the accounts still unexplained
at account grain: it groups every posted line by business account, by subaccount segment or
dimension, and by branch or entity, month by month against prior years, ties each table to the GL,
and usually shows which few documents are worth opening. Open documents one by one (the
transaction-lines pull in `data-pulls.md`) only for the cells it flags — a spike, a credit, a
one-off.

What you are looking for, roughly in order of how often it turns out to be the answer:

- **Concentration.** A handful of documents accounting for most of the gap. Very common, and it
  usually names the cause by itself.
- **A date at a boundary.** Documents posted right at the start or end of the period, or dated in one
  period and posted in another — the signature of a timing variance.
- **Something absent.** A recurring charge that did not appear this period. A favourable variance is
  as often an omission as an efficiency, and an omission is a close finding rather than a business
  one.
- **A duplicate.** Same counterparty, same amount, near dates. Rare, material when it happens, and
  worth a specific look precisely because nobody expects it.
- **A catch-up.** One document covering several periods. It inflates this period and understates the
  earlier ones, and treating it as a run-rate change is a standard mistake.
- **Miscoding.** A document that belongs on another line, another dimension, or another period. Check
  whether the offsetting under-run exists somewhere else before concluding anything about this line.
- **A change in document pattern.** Same total, many more or many fewer documents — a supplier
  consolidation, a switch from accrual to invoice, a change in how a team raises requisitions.

### 3. Dimension breakdown — where did it land?

Split the movement across the axis that matters: department, project, branch, region, product,
customer, cost centre.

The pattern tells you the kind of cause:

- **Concentrated in one value** — a local cause with a named owner. Go and ask them.
- **Spread evenly across all values** — a rate, a policy, an allocation or an accounting change. Do
  not go and ask a department head about it; it is not theirs.
- **Concentrated in new values** — a new site, project, product or customer. A scope movement, not
  performance.
- **Concentrated in values that disappeared** — a closure, a completion or a lost customer. Also
  scope, and the other half of the same story.

Confirm what each axis means in this environment before attributing anything to it. Which segment or
dimension carries department, project or region is a per-environment convention, and a finding
attributed to the wrong axis is confidently wrong.

### 4. The people who own the line

The last mile, and the only source for intent. A document shows that a payment was made; only a person
can say it was a deliberate acceleration to secure a discount.

Ask about the specific finding, not the line. "Spend on this line is up" invites a defence; "these
three documents in the second half of the period account for most of the movement — what were they"
invites an answer.

Record what you are told as what you were told, with who said it and when. A cause sourced from a
conversation is good evidence and is a different kind of evidence from a document, and the difference
matters when someone revisits the finding in three months.

## The why-chain

For the largest driver, and for anything the reader will be asked to act on, keep asking why until the
answer is something that can be changed. A chain of four or five steps is typical; the number is not
the point and should not be forced.

The discipline is that each step answers the previous one with a fact, not a restatement. The failure
mode is a chain that descends into abstraction — each step vaguer than the last, ending at something
like "insufficient oversight", which is true of everything and actionable against nothing.

**Stop when you reach a cause with three properties:** someone owns it, it can be changed, and
changing it would have prevented this variance. That is the level at which a corrective action is
worth writing.

**Do not stop at the first plausible step.** "Spend exceeded plan because volumes were higher" restates
the decomposition. Why were volumes higher; was that a demand change, a forecasting miss or a pull-
forward; and what does that imply about the next three periods — that is where the analysis earns its
place.

**Do not run the chain on everything.** It is expensive and it dilutes. One or two per analysis,
on the drivers that carry the consequence.

**Branch where the evidence branches.** A variance with two genuinely separate causes is two chains,
not one chain with a compromise at each step.

## Recurring misattributions

Worth knowing by name, because each one produces a confident and wrong finding, and each is common.

- **Mix read as price.** Revenue per unit fell, so prices were cut — when in fact the blend shifted
  toward a lower-priced line at unchanged prices. Needs item-level data to distinguish, and the two
  conclusions lead to opposite actions.
- **Timing read as performance.** A late supplier invoice becomes a favourable variance this period
  and an unfavourable one next. Extrapolating either is a standard forecasting error, and it is why
  the timing/permanent classification exists.
- **Correlation read as cause.** Two lines moved together, so one caused the other. Often both were
  caused by a third thing, and often the direction is the reverse of the assumed one.
- **The single document read as a trend.** One large item in an otherwise normal period. History
  answers this in one look, which is why history is the first rung.
- **A budget artefact read as a business event.** A period variance created entirely by how an annual
  budget was spread. Check the phasing basis before investigating a phasing artefact.
- **An efficiency variance read as a work problem.** A stale standard produces a permanent, meaningless
  efficiency variance. If the standard has not been reviewed in a long time, the finding is about the
  standard.
- **A favourable variance not investigated at all.** The most common omission in the whole method.
  Underspend is as likely to be an omission, a delay, a missed accrual or work that is not happening
  as it is to be good management.
- **An allocation change read as a cost change.** The pool did not move; this line's share of it did.
  Different finding, different owner, no operating action.

## What "unexplained" should look like

An honest unexplained item is specific, not a shrug. State the amount, what was ruled out, what the
next step is, and who would know:

> Of the gap on this line, an amount remains unattributed. History shows no comparable movement in
> the prior periods reviewed, and the transaction detail shows no concentration — the movement is
> spread across many small documents. The dimension breakdown has not been obtained; that is the next
> step, and the line's owner would be able to confirm whether a policy or pricing change took effect
> in the period.

Compare with "remaining variance: other". The first tells the reader what to do next and can be picked
up by someone else; the second tells them nothing and will be carried forward unexamined for as long
as anyone keeps producing it.