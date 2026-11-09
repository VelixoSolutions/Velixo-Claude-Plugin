# Outputs, forward effect and handoff

What the analysis delivers, how the forward numbers are built, and how it hands over. Read at Steps
7 and 8.

## The output is findings, not prose

This skill produces a structured set of findings that another step turns into commentary, a decision,
or a reforecast. Keep the writing functional: a reader should be able to scan the findings, disagree
with one of them specifically, and see exactly which evidence they are disagreeing with.

Where the deliverable is meant to be readable narrative, that is a different job —
`skills/velixo-flux-writer/` — and the handoff below is how to get there.

## Deliverable shape

In this order. Adjust depth to the depth agreed at Step 1; do not drop sections, because their absence
is itself information.

1. **Basis** — short, up front, and never omitted. Scope (company, branch or consolidation, ledger,
   period, status, basis, currency), the named baseline, the sign convention, the decomposition
   convention, and the materiality rule as applied — the user's, stated as theirs, or the one chosen
   and why. Five or six lines. Every disagreement about a finding starts here, and having it stated
   ends most of them quickly. Beneath it, the **sources**: per query whose figures appear, its name,
   the connection and every parameter value passed (`null` for an unused one), so any figure can be
   rerun exactly.
2. **The gap** — the total being explained, at the level the reader recognises, with the proportion
   covered by the selected items and the aggregate left unexamined. Where the request is a review of
   a statement, the full statement goes here: every line the pulls returned, current, baseline and
   change, grouped with subtotals, the selected lines marked. The materiality rule decides what is
   investigated, never what is shown.
3. **Ranked findings** — one row per selected variance: the line, the gap in currency and percentage,
   the components, the three classifications, the cause in a phrase, and the evidence class. Ranked by
   contribution to the gap.
4. **Decomposition detail** — per material finding, the components with the footing shown: they sum to
   the gap, or the named remainder is visible. Components that could not be computed are listed with
   what they would have needed.
5. **Cause record** — per material finding, the cause, the evidence behind it, the confidence in
   words, and for the largest drivers the why-chain. This is the section a reviewer will actually
   argue with, so make each claim separately attackable.
6. **Roll-ups** — three tables, each with favourable and unfavourable shown separately rather than
   netted: controllable / partly / not; timing / permanent / undetermined; one-time / recurring /
   timing. These three are what the review meeting is about.
7. **Forward effect** — remaining-period and full-year consequence, built as below.
8. **Proposed actions** — only where cause is evidenced. Owner, horizon, expected effect, and the
   basis for that effect.
9. **Open items** — what could not be determined, what would settle each one, and who would know.
   Ordered by how much of the gap each would unlock.

## Building the forward effect

Two separate figures. Keeping them apart is the whole discipline here.

**If nothing changes.** Follows mechanically from the timing/permanent classification and from nothing
else:

- Permanent components continue at their established per-period rate for the remaining periods.
- Timing components reverse — state in which period, and do not carry them forward.
- One-time components do not repeat.
- Undetermined components are excluded from the projection and stated as a range or an open exposure,
  not silently treated as either.

State the period count used and confirm it against the actual fiscal calendar rather than assuming
twelve even periods.

**If the proposed actions land.** The same arithmetic with the expected effect of each action applied
from its stated horizon, not from today. Present this as a scenario dependent on the actions being
taken, and never merge it into the first figure — a forecast that has already assumed its own
recommendations succeeded is the most expensive artefact this method can produce.

**A reforecast recommendation is a proposal.** Where the analysis concludes a plan needs changing, say
which lines, by how much, on what evidence, and hand it to whoever owns the plan. The recommendation
is an input to a forecasting process, not a substitute for one —
Velixo's Prompt Library has a Rolling Forecast Update entry where that work belongs.

## The action list

Four rules, each of which prevents a common failure:

1. **No action without an evidenced cause.** An action attached to an unexplained variance is activity,
   not management, and it will be reported as complete without the variance changing.
2. **Every action has one named owner.** Not a function, not a committee — a person or a defined role.
   An action owned by "Finance" is owned by nobody.
3. **Every action states an expected effect and its basis.** "Renegotiating this contract at the rate
   achieved on the comparable one would recover an amount of this order per period" is a basis.
   A number with no basis is a guess with a decimal point, and it will be held against whoever wrote
   it.
4. **Where the honest expected effect is unknown, say so.** An action worth taking with an unquantified
   benefit is still worth taking; pretending to quantify it is what damages the rest of the analysis.

Group by horizon — immediate, this period, this quarter — and mark which ones need approval before
anyone acts.

## Self-check before handing over

Run every line. Each one catches a real class of error.

**Arithmetic**

- [ ] Every gap recomputes from the stated inputs, as currency and as percentage.
- [ ] Every decomposition's components sum to its gap, or carry a visible named remainder.
- [ ] Percentages are computed on a stated denominator, and near-zero baselines are handled rather
      than printed as absurd percentages or dividing by zero.
- [ ] Period counts in the forward projection match the actual fiscal calendar.
- [ ] Favourable and unfavourable are never netted inside a roll-up category.

**Method**

- [ ] The materiality rule is stated and was applied identically to every line.
- [ ] The spine was taken on the gross movement to the stated coverage (the organization's, or 60%
      named as the default); no spine line was dropped by the percentage test, every one cleared the
      amount floor, and each line selected despite missing a test says which test and why.
- [ ] Selection includes the reader-critical, persistent and unexpectedly-signed items, not only the
      largest.
- [ ] One decomposition convention throughout, stated once.
- [ ] No component was produced from an input that does not exist.
- [ ] Both sides of every comparison are at the same scope.
- [ ] Favourable variances on committed or contracted lines were investigated, not waved through.

**Evidence**

- [ ] Every stated cause names the evidence behind it and its confidence in words.
- [ ] No cause was asserted from arithmetic alone.
- [ ] Unexplained amounts are labelled as unexplained, with a next step, and are not absorbed into a
      named component.
- [ ] What was reported by a person is distinguishable from what was found in the data.

**Classification**

- [ ] Every material finding carries all three classifications.
- [ ] Controllability states the owner and the horizon.
- [ ] Timing items state when they reverse.
- [ ] Budget and forecast variances are split between performance, phasing and plan error.

**Honesty**

- [ ] Open-period figures are labelled preliminary, with what could change them.
- [ ] The two forward figures are separate.
- [ ] No invented volume, rate, standard, counterparty, project, person, event or market statistic
      appears anywhere.
- [ ] Assumptions the analysis introduced — a budget spread, a proxy rate, an average headcount basis
      — are stated as the analysis's own, not as facts about the business.

## Handoff

**To the narrative writer.** `skills/velixo-flux-writer/` starts where this
skill stops. Hand it the basis block, the ranked findings with their causes and evidence, the three
roll-ups, the forward effect, and the open items — and mark every cause **established**, **indicated**
or **open**, because the writer maps those three words straight onto its hedging ladder (established
is stated flatly, indicated is hedged with what would confirm it, open stays unexplained with its next
step) and the distinction is invisible once the findings become sentences. Use the same eight driver
words the writer uses — volume, rate, mix, efficiency, timing, scope, one-time, classification.

Also pass what the narrative writer cannot infer: who the commentary is for, what decision it
supports, and anything off-limits or not yet announced.

**To a human, always.** This analysis produces proposals, not decisions. State in the handoff that it
is findings for review, and flag specifically:

- Findings whose evidence is thinnest, and what would settle them.
- The largest forward-effect figures, since those propagate furthest.
- Anything naming a person, a counterparty, a supplier or a customer.
- Any recommendation that would change a plan, a contract, a headcount or a commitment.

**Where the figures were never tied out**, say so in the handoff rather than at the end. A variance
analysis on untied figures is a finding about the close, and the reader needs that before they read
anything else.