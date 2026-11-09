
# Variance Analyzer for Business Central

These are the Business Central instructions of this skill. A file named `references/<file>.md` here, or in
the method this file includes, is `references/business-central/<file>.md`.


## What this skill does, and where it stops

A variance column tells you a number missed. It does not tell you what produced the miss, and that is
the only part anyone can act on. Missing budget on labour because you ran more shifts than planned is
a volume story with a revenue side to it; missing it at the same shift count because the agency rate
went up is a rate story that belongs to procurement; missing it because the crew took longer per unit
is an efficiency story that belongs to operations. The three look identical in the variance column and
lead to three different meetings.

This skill does that separation, and then the harder half: establishing *why* each component moved,
from evidence rather than from plausibility, and saying whether it repeats. Four entry points, one
method: **income statement flux**, **budget-to-actual**, **forecast-to-actual** (how well the
organization predicts itself — a different question from how well it performs against plan), and a
**cost or expense investigation** driven to the bottom.

It carries its own validated data pulls in `references/data-pulls.md`. It does **not** author new
queries (run the pulls as written; if one fails, report the error and stop; where no pull has the grain
needed, build that one through `prepare_query` and record it in the basis block), write the commentary
(that is `velixo-flux-writer`, which this skill's findings feed), or decide anything — every action
and reforecast it produces is a proposal for a named human.

**The rule that governs everything below:** arithmetic never establishes cause. A decomposition tells
you which component moved. Why it moved comes from a record — period history, a posted document, a
dimension breakdown, or a person who owns the line. Absent one of those, the finding is reported as
unexplained with a stated next step, never filled in with a reason that merely fits.

## Before the first query — the connection and its ERP

The connection was chosen, and its `type` read, in `SKILL.md` before this file was opened: it is a
Business Central connection. Never run these queries on a connection of another type.

## Step 1 — Settle scope, baseline and convention before computing anything

Eight things. What the request, the calling command or `SKILL.md`'s question with the connection
already settled is settled: never ask it again. If anything else is missing and cannot be
established safely from the inputs, ask once, for all of it together. Every one can turn a correct
computation into a wrong finding.

1. **Scope.** Company, branch or entity, or consolidation; the actual ledger or reporting book; the
   financial period or date range;
   accrual or cash basis; currency, and whether amounts are base or reporting currency.
2. **Baseline.** Prior period, same period last year, a named budget (and, where the ERP keeps
   budgets as ledgers, its budget ledger), or a named forecast version. One spine per analysis; secondary comparisons are labelled as secondary.
3. **Depth.** A scan of the top movements, a full review of everything material, or one line driven
   to the bottom with transaction detail and a why-chain.
4. **Convention.** Which sign means favourable on each line type, and the decomposition convention.
   State both once; never deviate inside the analysis.
5. **What the reader already knows.** Known events — a closure, a pricing change, an acquisition, a
   system cutover, a reclassification — are almost never in the ledger.
6. **Figure status.** An open period moves; an unphased budget has had a spread imposed on it;
   sub-ledger and GL can legitimately disagree.
7. **Which accounts are cost of sales and which are trading revenue.** The chart's account types do
   not carry either; the chart pull proposes a grouping, the user confirms it.
8. **Materiality.** The threshold for selecting lines: an amount, a percentage of the line's
   baseline, and whether a line must clear both (the default when both are given) or either — or the
   user's "size it for me". Not asked when the review is limited to named accounts.

`references/inputs-and-data.md` has the per-entry-point input checklist, what each optional input
unlocks, what may and may not be concluded when something is absent, and the structural facts about
this ERP's data that change an answer. `references/data-pulls.md` has the queries that settle scope
and figure status directly.

On Business Central, scope is the **company** (the connection; a connection with several companies
is listed once per company, and two companies are never added), **date windows**, and optionally a
**dimension value** for a department or project review; a baseline budget is named by its G/L budget
name.

**Periods: a month is the unit.** The user may name a month ("June 2023"), a month-end date ("30
June 2023") or an Acumatica-style period (`062023`, `MMYYYY`); each names one calendar month, and
none needs to be rewritten by the user. Run **P1** (`gl-accounting-periods`) from that month's first
day to the first day of the month after:

- **A period starts on the month's first day and the next one on the next month's first day:** the
  window is that accounting period. Name it in the basis block ("June 2023, 1–30 Jun 2023").
- **Otherwise:** the window is the calendar month's dates, and the basis block says why: no
  accounting period covers the month, or the company's periods do not follow calendar months (an
  extra period starts inside it). Never stretch or cut the window to a period's dates without
  asking.

A range of months is each end resolved the same way, from the first month's first day to the last
month's last day. A date that is not a month end is a date window as given. Business Central reads
every GL figure on posting date over the window either way, because a G/L entry carries no period.

## Step 2 — Get the figures

`references/business-central/data-pulls.md` carries the queries and which ones each entry point needs. Three things
silently change every number: an income-statement figure read as a balance (it is a net change over
the window, never `Balance at Date`); the cost-of-sales accounts, which the user confirms before any
margin figure even where the category says `Cost of Goods Sold`; and the period's status. No pull
reads whether a month is closed for posting: P1's `Closed` marks a closed fiscal year, so ask, and
label the figures preliminary when it is the current month or the user does not know. When no pull
has the grain needed, `prepare_query` is allowed for that one query: say so in the basis block, with
the tie-out it passed.

## Step 3 — Decide what is worth pursuing

Investigating everything investigates nothing. Rank by size of movement ignoring sign, take the
spine (the largest lines until they reach the coverage: the organization's, or 60% by default), then
add back the reader-critical, the persistent, the unexpectedly signed and the
suspiciously quiet. Set a materiality rule sized to the entity, state it, and apply it identically.

**The percentage test never removes a line from the spine.** Select, in this order:

1. **The spine.** Add up every line's movement ignoring sign (the gross movement). Take lines largest
   first until their cumulative share reaches the coverage; the line that crosses it is in. The
   coverage is the organization's or the user's where they give one (`spine:80%`, "cover 80%"), and
   60% otherwise, stated as the plugin's default. Never use the net change in the result as the
   denominator: offsetting lines make it meaningless. A spine line is exempt from the percentage
   test, not from the amount floor: one below the floor is dropped like any other line.
2. **The add-backs**, whatever their size.
3. **The lines outside those that clear the materiality rule.**

A large line rarely moves by a large share of itself, so a percentage test applied first drops the
biggest movement in the review. Each line selected despite missing a test says which test and why
it is in ("below the 10% test; within the coverage"). The basis states the gross
movement, the coverage and its source, and the spine's cumulative share. This holds at every depth,
a scan included.
`references/materiality-and-classification.md` has the bases, their failure modes and the selection
passes.

## Step 4 — Decompose the gap into components that sum back to it

For each selected variance, split the movement into the mechanical factors that produced it —
volume, rate, mix, efficiency — plus the non-operating residue of timing, scope, one-time and
classification that is not performance and must not be presented as such. State the convention
(volume and rate depend on which factor is held at baseline), make the components foot or name the
remainder as unexplained, and where an input does not exist — most often unit volumes or a standard
rate — do not produce the component: say which, what it would have taken, and what that leaves
unattributed. `references/decomposition.md` has the formulas, the convention question, the traps,
and the shapes for revenue, cost of sales, personnel, discretionary and capacity-driven cost.

## Step 5 — Drive each component to an evidenced cause

Work the evidence ladder in order, stopping when the answer is specific enough to act on:

1. **History** — is this new, or the fourth period of the same thing? Spike, step, drift or noise.
2. **Transaction detail and dimension breakdown** — what posted, with whom, and where it landed.
   The account drill-down skill (`velixo-account-drilldown`) is these two rungs together: run it after history, on the
   accounts still unexplained at account grain, before opening documents one by one.
3. **The people who own the line** — the last mile and the only source for intent. Record what you
   were told as what you were told.

For the largest driver, and any driver the reader will be asked to act on, carry the why-chain past
the first plausible stop until the cause is something someone owns and can change.
`references/root-cause.md` has the ladder in full, the why-chain, the stopping test, confidence in
words, and the recurring misattributions worth knowing by name.

## Step 6 — Classify every material finding

Three classifications, all required, because they are what the review meeting argues about:
**controllable, partly, or not** — by a named owner within a stated horizon; **timing or
permanent** — does it reverse, and when; and on any budget or forecast variance, **performance,
phasing, or plan error** — only the first is an operating finding. Criteria, the roll-ups and the
one-time/recurring split are in `references/materiality-and-classification.md`.

## Step 7 — Quantify the forward effect, then propose actions

Two figures, kept apart: the effect on the rest of the year if nothing changes, which follows from
the timing/permanent classification and nothing else; and the effect if the proposed actions land.
Propose an action only where a cause is evidenced, each with an owner, a horizon, and an expected
effect with its basis stated. `references/outputs-and-handoff.md` has the deliverable shape, the
forward arithmetic, the action rules and the handoff.

## Step 8 — Check it before it leaves

Run the self-check in `references/outputs-and-handoff.md` and the reconciliations in
`references/data-pulls.md`: the gap recomputes from the stated inputs; components foot or carry a
named remainder; the materiality rule is stated and applied consistently; every cause names its
evidence; every material finding carries all three classifications; no component was produced from
an input that does not exist; nothing was read off a truncated result; the forward effect follows
from the classifications; every action has an owner and a basis.

## Guardrails

- **Never invent.** Not a figure, a volume, a rate, a standard, a percentage, a counterparty, a
  project, an employee, an event, or a market statistic. A decomposition on an assumed volume is
  worse than none, because it looks rigorous.
- **Arithmetic is not causation.** Say it in the output too, once.
- **No cause without evidence.** History, a document, a dimension breakdown, or a named person.
  "Likely driven by" only with the evidence that makes it likely in the same sentence.
- **Components foot, or the remainder is named.** No plugging, no "other" that is really "unknown".
- **Do not produce a component you lack the input for.** Name it and say what it needed.
- **Never analyse a truncated result as the whole population.**
- **A classification is not a judgement.** Uncontrollable does not mean acceptable; timing does not
  mean harmless.
- **Do not smooth.** Unfavourable findings get the same specificity and follow-through as
  favourable ones; an unexplained underspend is investigated like an overrun.
- **Preliminary means preliminary.** Open-period figures are labelled, with what could change them.
- **A human owns the actions.** Everything naming a person, counterparty, plan, contract or headcount
  is a proposal for review, with the thinnest evidence and the largest forward effects flagged.
- **Respect the surface.** Plain Markdown is the default output. Where a rendered HTML deliverable is
  genuinely called for — a dashboard, a styled report — apply the `velixo-design` skill bundled
  alongside this one; it is the only place Velixo's palette and logo rules are authored. Never restyle
  a customer's own template, an Excel worksheet or a Teams card with it.
## Quick reference

```
SCOPE FIRST:     company/branch or entity, actual ledger or book, budget, period, basis, currency — confirm, never infer
BASELINE:        prior period | prior year | named budget | named forecast version — name it
CONVENTION:      favourable/unfavourable per line type; decomposition convention; state both once
PULLS:           references/data-pulls.md — run as written, keys are display labels, always pass top
LEDGER / BOOK:   Acumatica ledger, Sage Intacct book: supply it, or several are summed together
                 silently; a Business Central company has one general ledger
COGS:            not derivable from account type — propose a grouping, user confirms
OPEN PERIOD:     check what is not yet posted, where the pulls can read it, before calling an
                 expense variance favourable; otherwise label it preliminary
SELECT:          spine = largest |movements| to the coverage (policy's, else 60%), exempt from
                 the % test only; add reader-critical, persistent, unexpectedly-signed and
                 suspiciously-quiet items; then the rule
MATERIALITY:     set it, state it, size it to the entity — no imported generic threshold
DECOMPOSE:       volume / rate / mix / efficiency, plus timing / scope / one-time / classification
FOOTING:         components sum to the gap; remainder named as unexplained, never plugged
MISSING INPUT:   do not produce the component — name it, say what it needed
CAUSE LADDER:    history → drill-down (documents and dimensions) → the line's owner
WHY-CHAIN:       push the largest driver until the cause is something that can be changed
CLASSIFY:        controllable? | timing or permanent? | performance, phasing or plan error?
CONFIDENCE:      established | indicated | open — words, never a percentage
FORWARD:         effect if nothing changes, kept separate from effect if actions land
ACTIONS:         only where cause is evidenced; owner, horizon, quantified effect with its basis
NEVER:           invent a volume, rate or cause; plug a residual; present timing as performance
HANDOFF:         findings → velixo-flux-writer for the commentary, as a draft for finance review
```

Read a reference when the step that points to it is in front of you, not all of them up front —
except `references/data-pulls.md`, which is read before the first query runs.