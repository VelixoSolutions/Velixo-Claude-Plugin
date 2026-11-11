# Narrative craft

Sentence-level work. The analysis can be right and the commentary still fail here.

The figures in the examples below are illustrative only — they exist to show sentence shape and
carry no relationship to any real organization's results.

## The default sentence shape

Most commentary sentences do four things, in this order:

**Result → comparison → driver → consequence.**

> Operating costs of $4.2M were $310K above plan, largely from earlier-than-planned engineering
> hiring, which raises the full-year run rate by roughly $900K unless the remaining positions are
> deferred.

Result (`$4.2M`), comparison (`$310K above plan`), driver (`earlier-than-planned hiring`),
consequence (`run-rate effect and the lever available`). Drop the consequence where there genuinely
isn't one, but check first — a movement with no consequence usually did not deserve a sentence.

Two habits that follow from this shape. Put the figure in the sentence rather than in a trailing
parenthesis, because a reader skimming for numbers should find them in the flow of the sentence.
And make one sentence carry one driver; a sentence with three causes in it is a sentence the reader
has to re-read.

## Worked examples

### Operating cost flux

**Weak.** "Operating expenses were higher than budget this month, mainly due to timing and some
additional headcount costs. We expect this to normalize."

Nothing here is checkable. No amounts, "timing" names nothing, "some additional headcount" is not a
figure, and "normalize" is a hope with no date attached.

**Strong.** "Operating costs of $4.2M ran $310K (8%) above plan. Two items account for most of it:
$180K of engineering salary cost from four hires landing a period earlier than the plan assumed —
permanent, and lifting the full-year run rate by about $900K — and $95K of professional fees for
audit work that the plan had scheduled for the following period, which reverses next month. The
remaining $35K is spread across several lines, none individually above the $25K reporting
threshold."

### Cash position

**Weak.** "Cash decreased during the month but remains at a healthy level. Collections were a bit
slow but we expect them to improve."

"Healthy" is an assertion with no benchmark, "a bit slow" is not a quantity, and the improvement is
asserted rather than evidenced.

**Strong.** "Cash closed the period at $6.8M, down $420K. The decrease is almost entirely timing:
$650K of customer receipts originally expected before period end were received in the first week of
the following period, partly offset by a $230K deferral of a scheduled supplier payment run.
Underlying operating cash generation was positive at $180K. With the $3.0M revolving facility
undrawn, available liquidity is $9.8M."

### Revenue outlook

**Weak.** "Management remains confident of hitting the full-year revenue target, given a strong
pipeline and sustained demand across our core segments."

Confidence is stated, not supported; "strong pipeline" carries no coverage figure; "sustained
demand" is unattributed and unverifiable.

**Strong.** "The full-year outlook of $52M is unchanged. Of the $14M remaining in the final quarter,
$9.1M is contracted or recurring and already reflected in the order book. The balance of $4.9M
depends on new business, against a weighted pipeline of $7.2M — 1.5 times coverage, below the 2.0
times the same point last year carried. On that basis the target is achievable but requires a
better-than-recent close rate in the final six weeks."

Note what makes the third one credible: it names the part that is safe, sizes the part that is not,
supplies the coverage ratio, benchmarks it against a comparable point, and states the condition
under which the target holds. Confidence is demonstrated rather than claimed.

## Hedging against evidence, not against mood

Forward-looking language should be readable as a statement about the evidence behind it. Match the
verb to the evidence class:

| Evidence behind the statement | Language that fits |
|---|---|
| Already transacted and posted | State it flatly. No hedge belongs on a fact. |
| Contracted, committed, or in the order book | "is contracted", "is committed", "will deliver, absent cancellation" |
| Recurring with a stable retention history | "is expected to continue at", "on current retention" |
| Weighted pipeline or a probability-adjusted model | "the weighted pipeline supports", "on current conversion rates" |
| An assumption made by management | "assumes", "on the assumption that", with the assumption named |
| A judgement with limited visibility | "current indications suggest", stated together with what would change it |

**When the findings come from the variance analyzer**, each cause arrives marked with one of three
words, and the mapping is mechanical:

| Analyzer mark | Meaning | Language that fits |
|---|---|---|
| **Established** | A document, a contract, a dated event, or the line owner's confirmation | State it flatly: "reflects", "is", "follows" |
| **Indicated** | Consistent with the data, no contradicting evidence, not directly confirmed | "indicates", "is consistent with", "appears to reflect" — and name what would confirm it |
| **Open** | Not determined | "remains unexplained" — with the amount, what was ruled out and the next step, never a filler reason |

Two failure modes to avoid. Over-hedging a fact — a posted result does not need "we believe" in
front of it and reads as evasive when it has one. And under-hedging a projection — a hedge is not
decoration to be trimmed for concision; removing it changes what the sentence claims and, in an
external document, what it commits to.

Where a forward statement appears in anything leaving the organization, the assumption it rests on
appears with it. Not in a general disclaimer at the end — beside the statement.

## Language that erodes credibility

- **Vague intensifiers.** "Significantly", "substantially", "materially" used without the figure
  they are standing in for. If the number is available, the number is the adjective.
- **"Due to timing"** with no further content. Timing of what, moving from where to where, reversing
  when.
- **Unattributed external causes.** "Market conditions", "macroeconomic headwinds", "industry-wide
  pressure" — these are only usable with a source and a quantified effect. Otherwise they read as an
  excuse, and a board will hear them that way.
- **Superlatives.** "Record", "exceptional", "outstanding" — defensible only against a stated
  benchmark, and cheap without one.
- **Euphemism for bad news.** "Challenging", "softer than anticipated", "a headwind" used in place
  of the number. Give the number and the reason; the tone takes care of itself.
- **Passive constructions that hide the actor.** "A decision was taken to defer" tells the reader
  less than "we deferred", and boards notice the difference.
- **Hedging a fact.** Covered above, and worth repeating because it is the most common tic in
  drafted commentary.
- **Restating the table.** "Revenue was $9.4M, cost of sales was $3.6M, and gross profit was $5.8M"
  adds nothing to a reader who is looking at those three numbers.

## Balance

Adverse and favourable items get the same treatment: the same level of specificity, the same
register, the same position in the document. Where the largest movement in the period is adverse, it
is the first thing discussed. Ordering by size does this automatically, which is one of the reasons
to order by size.

A commentary that reads as uniformly positive against mixed results is not diplomatic, it is
unreliable, and once a reader notices that they discount everything else in it.

## Consistency

- A figure appears with the same value, rounding and label everywhere it appears.
- A line item is called the same thing in the prose as in the table beside it.
- Periods are labelled the way the reader's own reports label them.
- The comparison baseline is the same one from the first paragraph to the last, unless a change is
  explicitly flagged.
- Tense: past for results, present for position, future with hedging for outlook. Do not drift into
  the present tense for results because it sounds more immediate.
- Voice: first person plural for management actions, third person for the entity's results. Pick one
  convention for the package and keep it.

## Self-check before delivery

Work down this list. It takes a couple of minutes and catches most of what a reviewer would catch.

1. Does every figure in the prose match the source table exactly, including rounding?
2. Does every percentage recompute from the two figures it sits between?
3. Does every stated driver trace to something supplied — a person, a document, an event — rather
   than to inference? Anything that does not is either removed or re-labelled as unexplained.
4. Are the largest adverse items present, specific, and not deferred to the end?
5. Is each material variance identified as timing or permanent?
6. Does every forward-looking statement carry hedging matched to its evidence class, and name its
   assumption?
7. Is the baseline the same throughout, and named?
8. Are the scope facts — company, entity or branch, ledger or book, period, basis, currency — stated once and
   accurate?
9. If the period is still open, is the commentary labelled preliminary?
10. Does any sentence merely restate a row of the table? Cut it.
11. Is the materiality rule stated, and applied consistently?
12. Would the responsible finance owner be comfortable signing this as written? Anything that would
     give them pause is flagged in the handoff rather than smoothed over.

Deliver as a draft for finance review, and name the passages most needing that review — forward
statements, risk characterizations, commitment language, and anything naming a counterparty.
