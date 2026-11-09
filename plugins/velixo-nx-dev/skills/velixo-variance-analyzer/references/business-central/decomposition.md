# Decomposing a variance

The computational core. A gap is one number; this file turns it into named components that sum back
to it. Read at Step 4. The eight drivers are the ones in `materiality-and-classification.md`'s ladder;
this file is their arithmetic.

## What a decomposition is, and is not

Splitting a gap tells you which mechanical factor produced it. It does not tell you why that factor
moved, and it is not evidence of anything. A volume variance says activity differed from baseline; it
says nothing about whether that was demand, capacity, a sales decision or a data error. Root cause is
the next step and a different kind of work.

The value of the split is that it points the investigation at the right question and the right person.
That is enough to justify the effort, and claiming more for it is where variance analysis goes wrong.

## Settle the convention before computing

Volume and rate variances are not uniquely determined. When both the quantity and the per-unit amount
change, the interaction between them has to be assigned somewhere, and different conventions assign
it differently. The same data produces different splits, all arithmetically valid.

Three conventions are in common use:

- **Volume at baseline rate, rate at actual volume.** The volume component uses the baseline per-unit
  amount; the rate component uses actual quantity and so absorbs the interaction. The most common
  convention in management reporting, and the default here unless the organization has its own.
- **Volume at actual rate, rate at baseline volume.** The mirror image; the volume component absorbs
  the interaction.
- **Three-way, with the interaction broken out.** Volume and rate both computed at baseline, and the
  cross term reported as its own component. The most honest and the least readable; worth it when the
  interaction is large enough that assigning it either way would mislead.

Pick one, **say which one in the output**, and apply it identically everywhere in the analysis.
Switching conventions between lines makes the components incomparable and makes the totals wrong in a
way nobody can see. Where the organization already has a convention in its reporting policy, use
theirs — comparability across periods beats any argument about which is theoretically better.

## The components

### Volume

The gap attributable to doing more or less of the same thing at the baseline per-unit amount.

```
Volume component = (Actual quantity − Baseline quantity) × Baseline rate
```

Needs a quantity on both sides. Where no quantity exists, this component does not exist — see the
missing-input rule below.

On cost lines, a volume variance usually is not a finding on its own: more units consumed more
material, which is what was supposed to happen. It becomes a finding when the volume moved and the
corresponding revenue or output did not, which is why cost and revenue volume components should be
read together rather than in separate sections.

### Rate

The gap attributable to the same activity at a different per-unit amount — a price, a wage, a
contract rate, a tariff, a landed cost.

```
Rate component = (Actual rate − Baseline rate) × Actual quantity
```

Where a rate is not directly available, it is derived as amount divided by quantity, which makes it
an average realised rate rather than a list rate. Say which one you are reporting. A movement in
average realised rate can be discounting, mix within the line, or a genuine price change, and those
are three different findings — which is exactly what the mix component is for.

### Mix

The gap attributable to the same total activity distributed differently across products, services,
customers, channels, grades, or entities that carry different rates.

```
Mix component = Σ over items of [ (Actual share − Baseline share) × (Item baseline rate − Overall baseline rate) ] × Actual total quantity
```

Mix requires item-level data. With only a total, a mix effect will appear inside the rate component
and be misread as a price change — the single most common misattribution in revenue analysis, and the
reason a "price decline" so often turns out to be a shift toward a lower-priced line at unchanged
prices.

Two working notes. **Rate and mix are routinely confused**: rate is the same thing at a different
price, mix is different things at their usual prices. And **mix is a residual by construction** —
computed after volume and rate, it collects whatever they did not, so a large mix component with no
item-level story behind it is a signal that something else is wrong rather than a finding.

### Efficiency

The gap attributable to consuming more or less input than the output should have required. Applies
wherever a standard, a norm or an expected consumption ratio exists — production, labour, freight per
shipment, cost per service call, spend per active customer.

```
Efficiency component = (Actual input used − Input that the actual output should have required) × Baseline rate
```

The "should have required" figure is the standard, and the component is only as good as it is. A
standard that has not been reviewed in years generates a permanent efficiency variance that is a
finding about the standard, not about the work — which is worth reporting as exactly that.

Efficiency and rate together make up the total spend variance on an input; reporting one without the
other lets a favourable rate hide an unfavourable efficiency and vice versa.

### The non-operating residue

Four categories that appear in the gap and are not performance. Separating them out is often the
single most useful thing the decomposition does, because each one tells the reader to stop
investigating.

- **One-time.** A discrete item — a settlement, a write-off, a recruitment fee — with no equivalent
  expected in future periods. Quantify it and say what it was; it leaves the run rate untouched.

- **Timing.** The activity belongs to a different period than the one it landed in — an invoice
  received late, an accrual released, a payment run moved, a catch-up posting, a revenue recognition
  boundary. It reverses. Quantify it, name when it reverses, and keep it out of any run-rate
  conclusion.
- **Scope.** The two sides are not like-for-like — an acquisition, a disposal, a reorganisation, a
  new site, a discontinued line. Real movement, zero information about performance until the scope
  effect is removed. Where possible, also report the gap excluding scope, since that is the number
  the reader actually wants.
- **Classification.** The same economics reported on a different line than before — a
  reclassification, a chart change, an allocation method change. Not an economic movement at all.
  Identify it as presentational, or the reader will act on a change that did not happen.

## Footing: the discipline that makes it trustworthy

The footing rules are in `materiality-and-classification.md` (shared with the narrative writer):
the components sum to the gap, the remainder is named "unexplained" and left visible, and a
component that nets two large opposing movements gets both stated. If the sum does not tie,
something is wrong with an input, not with the concept.

## When an input does not exist

Do not produce the component. State which one could not be computed, what input it would have
required, and how much of the gap is left unattributed as a result.

The tempting workarounds all produce circular results: deriving volume from the spend and an assumed
rate makes the volume component a restatement of the total; carrying a rate from another period makes
the rate component a measure of that other period; treating a budget's implied per-unit figure as a
standard when the budget was built top-down makes the efficiency component a measure of the budgeting
process. Any of these can be used deliberately if the assumption is stated and every dependent figure
is marked — none of them can be used silently.

## Shapes by line type

### Revenue

Volume, rate (average realised price) and mix, plus customer effects where customer-level data
exists — new, lost, expanded and contracted customers, reported as four figures rather than one net.
A net revenue increase that combines a large loss and a larger gain is two findings.

Watch the revenue-side trap that mirrors the cost-of-sales one: a ledger's income classification
typically carries more than trading revenue — freight recovered, interest, other income, and
discount accounts that post against it. Establish which accounts are the revenue line with the user
before decomposing anything.

### Cost of sales and gross margin

Requires the user to identify which accounts are cost of sales, because the ledger's account typing
does not distinguish it from operating expense. Once identified, decompose into input rate, usage or
efficiency, mix of products sold, freight and duty, and — where a standard costing approach is in
use — purchase price and usage variances against those standards.

Report gross margin percentage movement alongside the absolute, and decompose the percentage
separately: a margin percentage can fall while margin currency rises, and which one the reader cares
about depends entirely on what decision they are making.

### Personnel

Headcount (volume), average cost per head (rate), and role or grade mix, plus the three items that
routinely sit outside all three and are worth naming individually: overtime and premium time,
contractor and agency spend, and one-off items such as severance, bonus true-ups and recruitment
fees. A headcount variance computed on period-end headcount will not reconcile to cost when joiners
and leavers moved mid-period; use average headcount over the period, and say which you used.

### Discretionary spend

Marketing, travel, professional fees, training. Often no meaningful volume or rate exists, and
forcing a decomposition onto them produces nonsense. The useful split here is different: planned
versus unplanned, committed versus discretionary, and one-off versus ongoing — with transaction
detail doing most of the work, because a handful of documents usually accounts for most of the gap.

### Capacity and facility cost

Largely fixed within a period, so a variance is usually a step change (a new site, a lease revision,
a rate review) or a timing artefact, not a gradual movement. Check for the step first; if there is
one, the decomposition is a single named event and the rest of the ladder is unnecessary.

### Allocated and shared cost

Decompose into the movement in the pool and the movement in this line's share of it, separately.
These are different findings with different owners — a department whose allocated cost rose because
the pool grew has a different conversation ahead of it than one whose share of an unchanged pool
rose. Where the allocation basis itself changed, that is a classification movement, not performance.

## Grain: where to compute

Compute the gap at two levels, not one.

**The level the reader recognises** — the reporting groupings their own statements use — is where the
variance is reported and ranked.

**The level where it is actionable** — account, dimension value, project, cost centre — is where it
is decomposed and investigated, because a gap at a summary line is almost always several movements in
different directions, and the net is the least informative number available.

Go one grain further only when the line is under real investigation. Decomposing everything to
transaction level is not thoroughness; it is a way of producing a document nobody reads.