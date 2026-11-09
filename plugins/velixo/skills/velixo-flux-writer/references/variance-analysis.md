# Variance analysis for narrative purposes

This file is about turning a column of differences into something worth reading. It is not a
computation method — the figures arrive already computed. It is the reasoning layer between the
numbers and the sentences. The materiality, selection, driver and footing rules below are shared
word for word with the variance analyzer, so findings and prose use one vocabulary.

## Name the baseline, and keep one spine

Every variance is a difference from something. State what, in the commentary itself, in the reader's
own vocabulary:

- **Versus prior period** — sequential movement. Best for operational commentary and cash.
- **Versus the same period last year** — removes most seasonality. Best for growth and trend claims.
- **Versus budget or plan** — accountability against a commitment. Requires the budget to be
  identified, since more than one can exist for a period.
- **Versus latest forecast** — how well the organization predicts itself. Distinct from budget
  variance and frequently confused with it.
- **Versus a prior version of the same forecast** — the right spine when the subject is a change in
  outlook rather than a change in results.

Pick one as the spine and hold it across the whole commentary. Secondary comparisons are useful but
must be marked as secondary; silently switching baselines between paragraphs is the most common way
a well-written commentary becomes unusable.

## Favourable and unfavourable, not positive and negative

Arithmetic sign does not carry meaning on its own. Revenue above baseline is favourable; expense
above baseline is unfavourable; both may be a positive number in the variance column. Cash movement
depends entirely on what caused it.

Use favourable and unfavourable, or plain language describing the effect. Reserve "positive" and
"negative" for cases where the direction of the number itself is what is being discussed. Pick a
presentation convention for the variance column — unfavourable shown as negative is the usual one —
state it once, and never deviate inside the same package.

## Materiality: set it, state it, size it

Do not carry in a generic threshold. A rule that suits one entity is noise at ten times the scale and
silence at a tenth of it, and a threshold inherited from a template is the fastest way to produce an
analysis that either misses the point or buries it.

| Basis | What it catches | Where it fails |
|---|---|---|
| Percentage of the line's own baseline | Large relative moves in small accounts, which is often where the interesting ones are | Long lists of trivial currency amounts with dramatic percentages; a near-zero baseline makes the percentage meaningless or undefined |
| Percentage of a controlling total (revenue, total spend, the gap being explained) | Keeps the analysis at the reader's altitude | Misses a complete collapse in a small line that nobody notices until it matters |
| An absolute currency floor | Stops trivia consuming a paragraph | Scale-dependent, so it needs resetting as the entity grows, and it silently stops working if nobody does |
| The organization's own reporting policy | Comparability across periods, which beats every theoretical argument | Only exists sometimes |

In practice a percentage combined with an absolute floor handles most cases — an item must clear
both, or either, depending on whether the reader wants a short list or a complete one. Say which
logic was used; "both" and "either" produce very different lists from the same two numbers. Use the
organization's policy where one exists, keep the rule stable period to period, state it in the
output in the reader's units, and apply it identically to every line.

**A threshold the user gives is that policy for the run.** Use it as given, state it as theirs, and
when it names an amount and a percentage without saying how they combine, a line must clear **both**.
"Size it for me" hands the choice back to the rule above, and the output says what was chosen and why.

**The percentage test does not apply to the spine below; the amount test does.** A large line
rarely moves by a large percentage of itself, so a percentage test on its own baseline can drop the
biggest movement in the review. A spine line is selected when it clears the amount floor, whatever
its percentage, and says so ("below the 10% test; within the coverage"). A spine line
below the amount floor is not large, and the rule drops it like any other. The four kinds of item
added back are selected whatever either test says, each with its reason. The rule decides among the
rest.

## Selection beyond size

**The spine is a count, not a judgement.** Add up every line's movement ignoring its sign: that is
the **gross movement**. Take lines largest first, by the size of their movement ignoring sign, until
together they reach the **coverage**; the line that crosses it is in. The coverage is the
organization's own, where its reporting policy sets one or the user gives one; otherwise it is
**60%**, and the basis says that 60% is the plugin's default, not their policy. Those lines are
the spine, and they are normally a minority of the lines; the materiality rule selects among the
lines outside it. Use the gross movement, never the net change in the result: offsetting movements
(revenue up, cost of sales up with it) make a net figure that one line can exceed on its own, and a
share of it means nothing. State the gross movement, the coverage and where it came from, the spine's
lines and their cumulative share in the basis. Then add back four
kinds of item that size alone misses:

1. **Reader-critical regardless of amount.** A covenant-relevant line, a metric the reader asked about
   last time, anything with governance or contractual implications, anything that will be noticed.
2. **Small but persistent.** Too small to rank on its own, in its third or fourth consecutive period
   in the same direction. The persistence is the finding, and a single period's table cannot show it.
3. **Unexpectedly signed.** A favourable variance on a line that should not have one. An underspend
   on a programme that was supposed to be running is a programme that is not running.
4. **Suspiciously quiet.** A material line with almost no net movement when the business changed
   around it. Offsetting movements underneath a flat total are common and invisible from above.

Everything not selected gets one closing line: the remaining variances were individually below the
stated rule and, in aggregate, account for the balance — with that aggregate quantified.

## The driver ladder

Every material variance is explained as one or more of these, and the same eight words are used
from the findings through to the prose. Most real variances are a combination; saying which
combination is the value added.

| Driver | What it means | What it sounds like in a sentence |
|---|---|---|
| **Volume** | More or fewer units, transactions, customers, hours, headcount, at the baseline rate | "…on twelve percent higher shipped volume" |
| **Rate** | The same activity at a different price, cost or wage per unit | "…at an average realised price four percent below the prior period" |
| **Mix** | The same total activity distributed differently across items that carry different rates | "…as lower-margin service work took a larger share of the total" |
| **Efficiency** | More or less input consumed than the output should have required, against a standard | "…at 6% more labour hours per unit than standard" |
| **Timing** | The activity belongs to a different period than the one it landed in; it reverses | "…reflecting a supplier invoice received a period later than planned" |
| **Scope** | The comparison is not like-for-like: an acquisition, a disposal, a reorganization, a chart change | "…including a full period of the acquired operation, absent from the baseline" |
| **One-time** | A discrete item with no equivalent expected in future periods | "…including a one-time settlement, with no equivalent in the baseline" |
| **Classification** | The same economics reported on a different line than before; not an economic movement | "…following the reclassification of support costs into cost of sales" |

Rate and mix are routinely confused: rate is the same thing at a different price, mix is different
things at their usual prices. Timing, scope and classification are not performance and must be
named as such, or the reader acts on a movement that did not happen.

## Timing or permanent

The distinction readers ask for first and analyses most often omit. For every material variance:

- **Timing** — it reverses. Say roughly when, in which direction, and whether the reversal is already
  in the forecast. A timing item has no run-rate consequence, but it can still have a cash
  consequence and still breach something in the period it lands in. Timing does not mean harmless.
- **Permanent** — it does not reverse; the run rate has changed and the forward effect follows.
- **Undetermined** — sometimes the honest answer, and better said than guessed. Say what would
  settle it and who would know.

## Footing and bridges

```
Gap = Volume + Rate + Mix + Efficiency + Timing + Scope + One-time + Classification + Unexplained remainder
```

- The components sum to the gap. Compute the sum and show it.
- The remainder is named "unexplained" and left visible. Never resize a component to close the
  arithmetic, and never let a residual hide inside a component or an "other" line that reads as
  though it were understood. A 60% explained gap reported honestly beats a 100% plugged one.
- A component that nets two large opposing movements gets both stated. A small net on top of two
  large opposing movements is misinformation by omission.
- Where a reader needs to see the pieces add up, build a bridge: open with the baseline figure, close
  with the actual, both matching the supporting statements exactly; components in business terms,
  ordered by size or by a narrative logic the reader can follow.
## Domain-specific wrinkles

### Budget commentary

Separate three things a single budget variance can be hiding: performance against the commitment,
an error or optimism in the budget itself, and phasing — the activity happening in a different
period than the budget assumed. Only the first is performance. Where the budget was never phased and
has been spread evenly to produce a period comparison, say so; that spread is an assumption
introduced by the analysis, not a fact about the plan.

Where the budget has been revised since approval, be explicit about which version the variance is
measured against. A favourable variance against a revised-down budget and one against the original
are different claims.

### Forecast narrative

Three comparisons routinely get conflated: forecast versus plan (are we going to hit the
commitment), forecast versus prior forecast (has our view changed), and forecast versus prior-year
actual (are we growing). State which is being discussed. A change in outlook is itself the subject
in most forecast commentary, and it deserves its own decomposition: what part of the change is
results already in the bag, what part is a changed assumption, and what part is a changed
probability on something that has not happened.

Where composition data exists, use it — a forecast weighted toward contracted and recurring revenue
carries different risk than one leaning on unclosed pipeline, and the hedging language should differ
accordingly.

### Cash commentary

Separate the movement into operating, investing and financing before explaining any of it; a net
cash movement that nets an operating deterioration against a financing draw is the single most
misleading figure in a cash summary. Distinguish cash from total liquidity, and identify restricted
balances separately.

Where a forward view is included, state the assumptions that drive it and the horizon it covers, and
present it as a projection rather than a position. Where a runway or coverage figure is requested,
it requires a stated burn basis and a stated treatment of committed facilities; without both, say
what is missing rather than producing a figure that looks precise and is not.
