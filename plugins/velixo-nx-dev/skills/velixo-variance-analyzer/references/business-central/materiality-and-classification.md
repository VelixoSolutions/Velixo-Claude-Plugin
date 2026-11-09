# Materiality, selection and classification

Which variances earn the work, and what labels every material finding must carry. Read at Steps 3
and 6. The materiality, selection, driver and footing rules are shared word for word with the
narrative writer, so findings and prose use one vocabulary.

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
## Classification axis 1 — Controllability

By whom, and within what horizon. Both qualifiers are required; without them this is an argument
rather than a finding, and it is the argument that consumes review meetings.

- **Controllable** — the named owner can change it within the horizon under discussion. Discretionary
  spend, hiring pace, supplier choice, overtime authorisation, scheduling, the timing of
  non-essential work.
- **Partly controllable** — the owner influences it but does not determine it. Volume-driven costs,
  costs shared with another function, prices set in a negotiation, anything governed by a contract
  that can be renegotiated but not this quarter.
- **Not controllable** — outside the owner's authority within the horizon. Statutory and regulatory
  costs, allocations set centrally, contracted prices inside their term, currency movement, insured
  events, and decisions taken above the line owner.

Three things this classification is not. It is **not a judgement of acceptability** — an
uncontrollable cost that has doubled still needs a plan. It is **not permanent** — much of what is
uncontrollable this quarter is controllable over a year, so state the horizon. And it is **not a
verdict on a person** — a finding that a line was uncontrollable protects the owner as often as it
excuses them, which is why it is worth getting right rather than leaving out.

**Roll it up.** Total controllable, partly controllable and uncontrollable, favourable and
unfavourable shown separately in each. This is the table the review meeting actually wants, because
it answers "how much of this miss was ours" in one line — and because netting favourable against
unfavourable inside a category hides exactly the thing the reader is trying to see.

## Classification axis 2 — Timing or permanent

Timing, permanent or undetermined, as defined above, on every material finding. **Roll it up too**, and keep the three totals visible. The permanent total is the one that drives the
forward projection; the timing total is the one that must be excluded from it. Merging them is how a
reforecast ends up extrapolating a late supplier invoice across the rest of the year.

## Classification axis 3 — Performance, phasing, or plan error

Required on every budget and forecast variance, and the axis most often skipped — which is why so many
budget reviews end in an argument about the budget rather than a decision about the business.

- **Performance** — the business did something different from what was planned. The only one of the
  three that is an operating finding.
- **Phasing** — the activity is happening in a different period than the plan assumed, but the annual
  position is intact. Especially common where an annual budget was spread evenly across periods; where
  that spread was imposed by whoever loaded the budget, or by this analysis, say so, because the
  variance is then partly an artefact of the analysis itself.
- **Plan error** — the budget or forecast was built on an assumption that was wrong at the time it was
  set. A rate that was never achievable, a headcount plan that did not reflect an agreed
  reorganisation, a volume assumption overtaken by events before the period started.

Plan-error findings are uncomfortable and are worth reporting anyway, plainly and without blame: they
are the only route to a better plan next cycle, and mislabelling one as performance sends an operating
team to fix something that is not broken.

## The one-time / recurring split

Related to timing but not the same, and both are worth carrying:

- **One-time** — a discrete event with no equivalent expected in future periods. Quantify it and state
  what it was.
- **Recurring** — it continues. State the ongoing per-period amount, which is the figure the forecast
  needs, not the period total.
- **Timing** — it reverses, so it is neither.

A single variance often splits across all three, and splitting it is usually more useful than labelling
the whole. An overrun that is half a one-off settlement, a third an ongoing rate increase and the
remainder a late invoice is three findings with three different consequences, and reporting it as one
number with one label loses all of them.

## Confidence, stated in words

Every classification is a judgement. Say how firm it is, in language, and tie it to what would change
it:

- **Established** — supported by a document, a contract, a dated event, or a direct confirmation from
  the line's owner.
- **Indicated** — consistent with the pattern in the data and with no contradicting evidence, but not
  directly confirmed. State what would confirm it.
- **Open** — not determined. State what would settle it and who would know.

Do not attach a percentage to a judgement. A number implies a measurement that was never taken, it
invites arithmetic that is meaningless, and it makes an "85% confident" cause read as more solid than
an "indicated" one that is identically supported.