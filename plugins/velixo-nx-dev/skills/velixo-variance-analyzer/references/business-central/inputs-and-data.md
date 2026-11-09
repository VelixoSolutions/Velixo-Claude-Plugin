# Inputs, data requirements and scope

What each entry point needs before the method can run, what each optional input buys, and what the
analysis may and may not conclude when something is absent. Read this at Step 1.

Nothing here is a query. Retrieval belongs to the Velixo connector and `data-pulls.md`; this file says what to ask for and what to check once it
arrives.

## The scope facts that travel with every figure

Establish these once, for both sides of every comparison, and record them in the output. A variance
computed across a scope mismatch is the most common false finding in this whole method, and it never
announces itself.

| Fact | Why it changes the answer |
|---|---|
| Company, branch, or consolidation | Consolidated and single-branch figures are indistinguishable in a table |
| Ledger | Where reporting or statistical ledgers exist, an unrestricted figure can sum several together |
| Financial period or range | Periods follow each company's fiscal calendar; a "month" is not a given |
| Period status | An open period's figures move; unreleased and unposted work is not in them yet |
| Basis | Accrual and cash answer different questions and produce different variances |
| Currency, and base vs reporting | Translation movement is part of the variance, not a footnote |
| Which dimension carries which meaning | Department, project, region and product axes are per-environment conventions |

**Both sides, not one.** The single most useful check in this file: confirm that the baseline was
produced at the same scope as the actual. Different branch sets, different ledger treatment, a
consolidation on one side and an entity on the other, or a budget loaded at a different grain than
the actuals are recorded at — each produces a large, confident, entirely fictional variance.

## Structural facts about Business Central data

- **One company per connection.** Every amount is in its local currency (`LCY Code`); a group of
  companies is several runs side by side, never summed.
- **Totals are flow fields.** Business Central computes an account's `Net Change` over a window and
  its `Balance at Date` at a date from the G/L entries itself, so an account-level pull is one call
  per window, whatever the volume.
- **Every amount is debit-positive, and no account keeps a normal balance.** `Natural Net` comes
  from the account category: `Assets`, `Cost of Goods Sold`, `Expense` as read; `Liabilities`,
  `Equity`, `Income` negated.
- **An income-statement balance is the account's whole history** until the year is closed. A
  period's result is a net change over its window.
- **Dimensions are the business axes.** Two global dimensions sit on every entry (the company's
  `Global Dimension 1 Code` and `2 Code`, often department and project); every other dimension is in
  the entry's dimension set. Which one carries which meaning is the user's to confirm.
- **There is no draft in the ledger.** A G/L entry is posted by definition; unposted journals and
  documents are elsewhere and are not read here. Work not yet posted, and whether the month is
  closed, come from the user.
- **Budgets are named G/L budgets**, by account and date, normally in the GL's sign; a budget
  entered or imported by hand may hold income positive. Check before comparing.
- **Periods are dates.** Entries carry a posting date, not a period; an accounting period only names
  a window, and a company's periods need not be calendar months or cover old years.

## Entry point 1 — Income statement flux

**Minimum:** results by account (or by the reporting grouping the reader uses) for the period under
review and for the comparison period, at the same scope, with the account's type or grouping so that
favourable direction can be assigned.

**Strongly wanted:** several prior periods of the same lines. Without history, every variance looks
new, and "is this the fourth month of the same thing" is the cheapest and most decisive evidence
available. Twelve or more periods also lets a seasonal movement be recognised rather than
investigated.

**Unlocks further work:**

- Transaction detail behind a line → a cause instead of a category.
- The dimension breakdown → an owner instead of a company-level observation.
- Unit or activity volumes → a volume/rate split instead of a single net number.
- Known events from the user → the difference between explained and unexplained.

**Cannot conclude without:** volumes, so no volume-versus-rate attribution; cost-of-sales account
identification from the user, so no gross margin or contribution analysis; scope confirmation, so no
statement about the business at all.

## Entry point 2 — Budget-to-actual

**Minimum:** budget amounts and actual amounts for the same lines, scope and periods, plus the
identity of the budget — which version, which scenario, original or revised, and when it was
approved.

**Strongly wanted:**

- Year-to-date as well as period. A period gap that is the third of its kind reads differently in a
  year-to-date column, and the year-to-date position is what a reforecast starts from.
- How the budget was phased. An annual figure spread evenly across periods will generate period
  variances that are purely artefacts of that spread. If the spread was imposed by whoever loaded the
  budget, or by the analysis itself, say so and treat the resulting period variances as phasing until
  evidence says otherwise.
- The assumptions the budget was built on — planned headcount, planned volumes, assumed rates. These
  are what separate a performance finding from a plan-error finding, and they are never in the
  ledger.

**Watch for:** budget lines with no actual and actual lines with no budget. Both are findings in
themselves — an unbudgeted cost centre or a planned programme that never started — and both are
routinely dropped by a join that only reports where the two sides meet. Ask for the full population
on both sides, and report the one-sided lines explicitly.

**Cannot conclude without:** the budget's identity, so no defensible variance statement at all — a
favourable variance against a revised-down budget and one against the original are different claims
and must not be reported as the same one.

## Entry point 3 — Forecast-to-actual

**Minimum:** the forecast as it stood at a stated point in time, the actual for the same lines and
scope, and the date the forecast was fixed. A forecast compared against actuals after it was updated
to reflect those actuals measures nothing.

**Strongly wanted:** the same comparison for several prior periods, because the subject here is
prediction quality rather than performance, and one period of it is an anecdote. Also the forecast's
own composition where it exists — what portion was contracted or committed versus estimated — since
a miss on the committed portion and a miss on the estimated portion have different causes and
different fixes.

**Keep three comparisons apart**, because they are routinely conflated and the conclusions differ:
forecast against plan (will the commitment be met), forecast against the prior forecast (has the view
changed), and actual against forecast (how well does the organization predict itself). This entry
point is the third; state which one the reader asked for before computing anything.

## Entry point 4 — Cost or expense investigation

The narrow, deep entry point: one account, category, department or project.

**Minimum:** the amount under investigation, its baseline, and the scope facts.

**Strongly wanted, and unusually important here:**

- **Period history for the same line**, long enough to see whether this is a step change, a drift, or
  a spike. This is the first thing to look at and it frequently ends the investigation.
- **Transaction detail** — document, date, counterparty, description, dimension coding, amount. This
  is where an investigation stops being inference. Without it, the analysis can categorise the
  movement but cannot name it.
- **The activity the cost is supposed to scale with.** Units produced, hours worked, shipments,
  headcount, service calls, square footage, licence seats. Almost never in the ledger and almost
  always required for a decomposition worth the name.
- **A rate reference** — standard cost, contract rate, budgeted per-unit assumption, prior-period
  average. Without one, a rate variance cannot be separated from a volume variance and the
  decomposition collapses to a single net figure.

**The honest fallback when volume and rate data do not exist:** say that the variance could not be
decomposed, report the total with whatever transaction-level evidence does exist, and name what would
have been needed. An analysis that reports one net number with a clear explanation of a large
document that landed in the period is more useful than a three-way split built on an assumed volume.

## Activity and volume data: the recurring gap

Worth its own section because it is the most common reason a decomposition cannot be produced, and
because the instinct to work around it is strong and wrong.

The ledger records what was spent, not what was done. Unit counts, hours, headcount and throughput
live in operational systems, in other modules where those capabilities are configured and licensed,
or in a spreadsheet someone maintains. Where they exist, ask for them by name and confirm they are
measured on the same period boundaries as the financial figures — an activity count for a calendar
month against a financial period that is not a calendar month produces a rate variance out of
nothing.

Where they do not exist:

- Do not derive a volume from the spend itself. Dividing the cost by an assumed rate and calling the
  result a volume produces a decomposition in which the volume variance is a restatement of the
  total, dressed up.
- Do not carry a volume from a different period, entity or line as a proxy without saying so
  explicitly and marking every figure that depends on it.
- Do state, once, which components could not be computed and what the remaining unattributed amount
  is.

## Data quality checks before the analysis runs

Five checks. Each takes a minute and each prevents a class of wrong finding.

1. **Do the figures tie?** Variances computed on statements that do not tie are variances in the
   close, not in the business. Where a tie-out has not been done, say that the analysis assumes the
   figures are final and name the risk. Velixo's Prompt Library has a Financial Statement
   Tie-Out Review that does this properly.
2. **Is the period open?** If so, label everything preliminary and check whether unreleased documents
   or unposted entries explain a favourable expense variance before reporting it as one.
3. **Does the population match on both sides?** Lines present in one side and absent in the other are
   findings, not rounding.
4. **Is the comparison like-for-like?** An acquisition, a disposal, a reorganisation, a chart-of-
   accounts change or a reclassification makes the two sides structurally different. These produce
   scope variances that are real movements in the table and not performance at all.
5. **Does an unexpectedly small variance hide offsetting movements?** A material line showing almost
   no net change is worth one look at the next grain down before concluding nothing happened.

## When the analysis cannot proceed

Hand back rather than guess. Say precisely which input is missing, which part of the method it
blocks, and where it comes from:

- Figures, history, transaction detail or dimension breakdowns → retrieval, via the
  Velixo connector and `data-pulls.md`.
- Budget identity, phasing basis, planning assumptions, known events, volume data, standard rates,
  or which dimension carries which meaning → the user. None of these are discoverable from the
  ledger, and all of them are quick to answer.

Partial progress is fine and usually right: run the analysis on what is available, mark clearly what
could not be computed, and list the missing inputs in priority order so the next round is one round
rather than three.