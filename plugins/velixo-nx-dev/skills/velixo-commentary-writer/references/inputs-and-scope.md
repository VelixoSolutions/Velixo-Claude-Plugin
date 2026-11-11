# Inputs and scope

What the commentary needs before it can be written, what it may and may not claim when something is
missing, and how to hand back when the figures are not there at all.

## The scope facts that travel with every number

A figure without these is not usable. Confirm each one, and carry them into the commentary's own
header or opening line so the reader never has to guess.

| Scope fact | Why it changes the words |
|---|---|
| Company / entity / branch / consolidation | A consolidated total and a branch total read identically in prose. Mixing them inside one commentary invites a reconciliation dispute at the meeting. |
| Ledger or book | Actual, budget, statistical and reporting ledgers (or books) answer different questions. A "budget variance" is only meaningful once the budget ledger, book or scenario is named. |
| Financial period or range | Periods follow the company's fiscal calendar, not the calendar year, and a calendar may include an adjustment period outside the normal reporting cycle. Use the reader's own period labels. |
| Period status | An open period's figures can still move. A closed period's generally cannot. This determines whether the commentary is preliminary. |
| Basis | Accrual and cash basis produce different variances from the same underlying activity. |
| Currency | State it. Where amounts were translated, translation movement is part of the variance and has to be named, not folded into "other". |
| Source of each figure | GL, sub-ledger, a spreadsheet model, a forecast file. Sub-ledger detail can legitimately disagree with the GL while documents are unreleased or batches unposted. |
| Which budget or scenario | More than one can exist for the same period and ledger or book. "Versus budget" without naming which one invites a reconciliation argument at the meeting. |
| What each segment means | Segments, dimensions and subaccounts are the by-department, by-region, by-product axis, and the meaning is per environment. Describe them by what they mean to the business; a reader outside finance should not have to decode a code. |
| Which capabilities are configured | Project, construction, multi-currency and similar detail exists only where it is configured. Commentary that leans on it states that as an assumption the reader can confirm. |

## Minimum inputs by commentary type

Each block below lists what must be present, what improves the result, and — the part that matters
most — what the commentary is not allowed to say without it.

### Income statement / flux commentary

**Required:** results for the period being discussed, the same lines for the baseline period, and
the account or line-item grouping the reader recognizes.

**Improves it:** a second comparison (prior year alongside prior period), year-to-date alongside the
period, volume or headcount statistics behind the lines, and a short list of known events in the
period.

**Without it:**

- No baseline → no variance commentary at all. Describe the period, or ask for the comparison.
- No line detail, only totals → describe the movement at the total level and say the drivers were
  not available at line level. Do not attribute a total movement to a line you cannot see.
- No events or operational context → the commentary can state what moved and by how much, and can
  decompose it arithmetically, but cannot assert a business cause. Say so plainly and ask.

### Budget commentary

**Required:** budget figures and actuals at the same line grouping and the same scope, and the name
or identifier of the budget being compared against.

**Improves it:** the prior year's actuals for the same lines, the assumptions the budget was built
on, phasing across periods, and whether the budget has been revised since approval.

**Without it:**

- Budget not identified → do not write "versus budget". Write "versus the figures supplied" and ask
  which budget or scenario they came from.
- No phasing → a year-to-date variance can be discussed; a statement about whether the year is on
  track cannot, because an unphased budget spread evenly is an assumption, not a fact. Do not make
  it silently.
- No assumptions → report the variance and its arithmetic decomposition; do not assess whether the
  budget was reasonable.

### Revenue outlook / forecast narrative

**Required:** the forecast figures with their periods, what they are being compared to (plan,
previous forecast, or prior-year actuals), and enough history to establish the trend being extended.

**Improves it:** the split between contracted or recurring revenue and everything else, pipeline or
backlog with its weighting basis, forecast accuracy over recent periods, and the assumptions the
forecast rests on.

**Without it:**

- No composition split → describe the total outlook; do not characterize its quality, predictability
  or risk profile, all of which depend on the mix.
- No pipeline or backlog → do not imply visibility that is not evidenced. Hedging language has to
  drop accordingly — see the evidence ladder in `narrative-craft.md`.
- No accuracy history → do not claim the forecast is reliable. Absence of a track record is not a
  track record.

### Cash position summary

**Required:** cash balances at the period end with their accounts or account grouping, the movement
over the period, and the comparison point.

**Improves it:** the split of movement into operating, investing and financing activity; committed
but undrawn facilities; restricted balances identified separately; known large receipts and payments
ahead; and any covenant the reader is tracking.

**Without it:**

- No movement detail, only balances → report the change and say the composition was not supplied. Do
  not infer that a cash increase was operating.
- No facility or restriction detail → report cash, not liquidity. They are different numbers and
  conflating them overstates the position.
- No forward information → do not produce a runway figure. Runway is a projection; it requires a
  stated burn assumption and a stated basis, and without them it is a fabricated number wearing a
  unit.

### KPI and operating metric commentary

**Required:** the metric, its definition, its value, the baseline or target, and the period it
covers.

**Improves it:** several periods of history, the components behind a ratio, and what decision the
metric informs.

**Without it:**

- No definition → do not interpret the metric. The same label means different things in different
  organizations, and interpreting the wrong definition is worse than omitting the metric.
- No history → report the value against target; do not describe it as improving or deteriorating.

## When the numbers are not there

This skill does not retrieve data. Two paths:

**Ask.** Usually correct and usually faster. Name precisely what is missing, at what grouping and
scope, and say what the commentary will be able to say once it arrives.

**Hand off to retrieval.** Where a Velixo connector session is available, retrieval follows the
connector's own tool ladder, scoping rules and guardrails, not anything invented here. Before a new
query is built, the data pulls in the variance analyzer skill for your ERP, where there is one, are the first place
to look; several of the shapes this skill's commentary types need are already parameterized there. Express the need in business terms — the
lines, the grouping, the periods, the scope — and let the query layer resolve how.

Do not write commentary against a figure you have neither been given nor seen retrieved. A number
that appeared without a source is not an input; it is a guess with formatting.

## Handling partial and inconsistent inputs

- **Figures that do not tie.** Where two supplied sources disagree, say so, quantify the difference,
  and ask which governs. Do not pick one silently, and do not average them.
- **Mixed scope.** Where part of the input is consolidated and part is a single branch, either
  restate everything at one scope or label each figure. Never let the reader assume consistency that
  is not there.
- **Mixed periods.** A year-to-date figure sitting beside a single-period figure is a common
  packaging accident. Label both.
- **Rounding drift.** Where components round to something other than the stated total, present the
  total as given and note the rounding rather than adjusting a component to force agreement.
- **A gap you cannot close.** Carry it into the output as an explicit note near the relevant section,
  not as a footnote at the end that a reader skimming the first page will never reach.
