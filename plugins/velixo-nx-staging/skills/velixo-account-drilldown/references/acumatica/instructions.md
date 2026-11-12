# Account drill-down for Acumatica

These are the Acumatica instructions of this skill. A file named `references/<file>.md` here, or in
the method this file includes, is `references/acumatica/<file>.md`.


## What it does, and where it stops

A flux tells you an account moved. This skill tells you **with whom**, **where it was coded** and
**in which branch**: every posted GL line in the account, grouped by the business account on the line
(`GLTran.ReferenceID`), by subaccount segment and by branch, and by financial period, laid out month
by month against the same months of prior years and, for segments and branches, against the budget.

**Business account** is the term used throughout, because the Reference ID can point at any business
account: a vendor on an expense account, a customer on a revenue or receivables account, an employee
on an expense-claim or advance account, a partner, a branch. Use the word that fits the account in
the output ("supplier", "customer", "employee"), and "business account" when the account mixes them.

It produces evidence for the variance analyzer's ladder — its second and third rungs, transaction
detail and dimension breakdown, in one pass. It does **not** decide causes: a business account whose
activity tripled is a finding; *why* it tripled is a hypothesis until a contract, an invoice, an order
or the line's owner confirms it.

## Before the first query — the connection and its ERP

The connection was chosen, and its `type` read, in `SKILL.md` before this file was opened: it is an
Acumatica connection. Never run these queries on a connection of another type.

## Step 1 — Inputs

**The grouping is the user's choice, never yours.** Invoked on its own, this skill asks which axis or
axes to break the figure out on, and in what order when there are several, in one message with
choices the user can tap, and runs no query until it has the answer. "Analyze account 60000" with no
axis named is that question, not a licence to pick one. The one exception: called from
`/variance` with no axes passed, the default is business account and segment (segment only when
the subaccount carries more than one value with activity), and the flux says so in its output.

Take the rest from the parent flux when called from it; otherwise ask for whatever is missing in the
same message:

1. **Connection, ledger and branches.** One actual ledger per run; never sum ledgers or currencies.
   Take the ledger from the connection when it has one actual ledger; ask only when it has several.
   **Branches: every branch on the ledger unless the user names some; never ask which.** When the
   baseline is a budget, also take the budget ledger.
2. **Accounts.** Up to eight account codes, an **account class** (Acumatica's grouping of accounts), or
   both, in which case an account must be in the class and in the list. Resolve a class name or
   description to its code with the chart of accounts classification; more than eight codes without a
   class run in batches of eight and are combined. Called from the flux with a grouping of accounts,
   take its account codes.
3. **Windows.** The current window, and each comparison window the user asked for (for example
   periods 01–06 of 2026, 2025 and 2024). **No comparison unless one is named: never ask for one;
   show the window itself, month by month.** When comparing, also pull **the months in between**: they
   date when a business account first appeared or stopped, often the most useful fact in the report.
   **Period values are `MMYYYY`** (`012025`), passed to every pull exactly as the user wrote them;
   never convert one to Acumatica's internal year-first form (`202501`) — the connector translates,
   and a year-first value matches no period and returns no rows.
4. **Axes.** Business account, subaccount segment, branch: one, or any combination. The order the user
   gives is the nesting order for the combined view; with several axes and no order, branch, then
   segment, then business account.
5. **Subaccount filter** (optional). One exact subaccount code, or a pattern built from segment values
   with D0: the value at its segment's position and `_` elsewhere. It narrows every table to those
   lines, and the filtered figure ties to D6 filtered the same way, not to the account total; the
   basis block says so.
6. **Business account types in scope** (optional). All by default; or only vendors, customers or
   employees when the user asks ("which customers drove revenue", "expense claims by employee").
7. **Depth.** Default: one table per axis chosen, month by month for the material business accounts.
   Deeper: the cross-tabs (business account × segment, branch × business account), then document
   lines for a specific business-account-month (Step 5).

## Step 2 — Pull the data

Every pull is in `references/acumatica/data-pulls.md`: functions (`F`) through `execute_velixo_function`, queries
with their parameter tables through `execute_query`, each run as written, a query with `top` high
enough that the result is not `truncated`. A function that fails falls back to the query in the
control column's place (pull 5 for F1 and F3, D5 for F5, pull 6 for F2), named in the basis block. Which pull serves which table depends only on
the axis, whether branches were named, and whether a subaccount filter is set:

| Table | Branches named | Subaccount filter | Pull | Control it ties to |
|---|---|---|---|---|
| Business account | yes | no | D1 per named branch, with `Branch ID` | D5 per branch, then pull 5 per branch |
| Business account | no (whole ledger) | no | D1 once, without `Branch ID` | D5 once, per branch inside it |
| Business account | either | yes | D2, keeping the matching subaccounts | D6 filtered the same way |
| Segment | yes | either | F3 per account and window, one pattern per D0 segment value, every invocation in one call | F1 |
| Segment | no | either | F5 (`"#"`) for the branches with activity, then F3 per branch with activity, account and window; past about 60 invocations, D6 once, split with D0 | F1 per branch |
| Segment × business account cross-tab | either | either | D2 for the current window (`Branch ID` when named) | D6, or pull 5 |
| Branch | — | no | F5 per account and window, every branch in one invocation | F1 per branch; D5 when lines are needed |
| Branch | — | yes | D6, keeping the matching subaccounts | — |
| Branch × business account | — | no | D1 without `Branch ID` | D5 per branch |
| Budget beside segments or branches | per branch | — | F2 per branch; F3 on the budget ledger by segment; D4 for the current window | F2 |
| Margin by segment | per branch | either | F3 per window on the revenue and cost accounts together, one pattern per D0 value | F1 per account |
| Cost of sales by customer | per branch | no | D12 for the window, summed by customer | D8's `IN` lines with an AR link; the rest of the account is its own row |
| Documents behind one cell | the cell's | pattern allowed | D7 with the cell's own filters, or D8 and the pull its module routes to (Step 5) | the cell |

Run D1, D5 and D6 from the first period of the earliest window to the last period of the current one;
D2 and D4 for the current window (and the comparison window when a business account's coding may have
changed). D0 and the chart of accounts run once per tenant. Branch names, companies and ledgers come
from pull 3 (`references/acumatica/data-pulls.md`, shared with the variance analyzer). `references/acumatica/segments.md` and `references/acumatica/branches.md` carry the method
for those two axes.

**Two pulls run on every drill-down, whatever the axis, and are not optional,** both in `references/acumatica/data-pulls.md`:
**pull 1** (the period calendar — the only source of each period's status, so "preliminary" is a
finding from it, never an assumption) and **F1** for each account, branch and window in scope (GL
history — the balance every table ties to; pull 5 at account grain when F1 fails). D5 is posted lines, the same source as D1; agreeing with
it proves the business-account join dropped nothing, not that the table ties to the GL. A write-up
whose basis block says neither F1 nor pull 5 was run, or pull 1 was not, is incomplete.

## Step 3 — Tie out before analysing

Every table ties to the GL before anything is read from it:

- **Business account:** for every account and window, D1's `Natural Net` summed across all business
  accounts equals the account's activity from F1 (GL history; pull 5 when F1 fell back). D5 is the second check, for the
  join; it does not replace pull 5.
- **Branch:** F5's branches sum to the account's all-branch F1, and each branch's F5 row equals F1
  for that branch. With D5 in play, D5 equals F5 branch by branch.
- **Segment:** each F3 table sums to the account's F1 for every window (a shortfall is a segment value
  the pattern list missed); with D6 in play, D6 equals D5 per branch, account and window, and D2 sums
  to F3 (or D6) per account for the current window.
- **Budget:** D4 summed per account equals F2 for the same branch, ledger and window.
- **Business account join:** D1, branch by branch, equals D5 on every account. A shortfall on an
  account whose lines carry references, while D5 ties to pull 5, is posted lines whose business
  account the query cannot read — a deleted record, or one the connector's user is restricted from —
  dropped by the business account join rather than returned with empty columns. Add a **Business
  account not visible** row for the difference so the table foots, say what it is, and never allocate
  it. If D5 doesn't tie to pull 5 either, stop.

Report both figures and the difference. A difference above rounding stops the run: the usual causes
are a truncated result, a missing account in the list, or a different ledger. The tie-out also
catches any error made while moving rows from the tool result into an analysis script.

**Sign.** Present `Natural Net` everywhere — every pull returns it, actual and budget alike, in the
account's natural sign — and state the convention once. The older `Net Amount` (debit less credit) and
`Budgeted Amount` (credit less debit) columns are for tie-outs only.

## Step 4 — Classify and read

Classify every business account by `Business Account Type` before ranking anything:

| Class | How to recognise it | How to report it |
|---|---|---|
| Vendor | `VE` | By name |
| Customer | `CU` | By name |
| Customer and vendor | `VC` | By name; say on which side this account sits |
| Employee | `EP` (returned, though not in the column's allowed values) | By name |
| Other business account or branch | `PR`, `CP` | By name; a `CP` (branch) business account is usually intercompany: say so |
| Clearing or reimbursement channel | A business account whose name says clearing, reimbursement, payment processor, card issuer | Its own line; say it is a channel, not the real business account, and look through it with D7 if material |
| No business account | `Reference ID` null | Its own line, labelled **No business account** and nothing more; never allocated across the others |
| Business account not visible | D5 less D1 on the account | Its own line, **Business account not visible (deleted or restricted business account)**; never allocated. Branch and segment tables are unaffected: they don't join the business account |

For the segment axis, split each subaccount into its segments using D0's lengths and labels, and read
the unassigned share, new values, concentration, recoding and the budget variance as described in
`references/acumatica/segments.md`. When the question is margin, read revenue and cost paired on the same segment
value (margin by segment, same reference), decoding every value with D0 first. For the branch axis, read where the change sits, intercompany lines and
branches that opened or closed as described in `references/acumatica/branches.md`.

Then read the patterns in `references/acumatica/patterns.md`: first and last seen, run-rate, price and volume
steps, irregular or annual billing, credits netting invoices, spikes, concentration and the share with
no business account.

## Step 5 — Go to documents only where it pays

For a business-account-month that matters and is not self-explanatory (a spike, a credit, a one-off),
pull **D7** with the cell's own filters (branch, account, business account, subaccount or segment
pattern, and the period): the posted lines behind it, with document, description and module. Their
`Natural Net` sums to the cell. D7 filters by financial period as the other pulls do; say so if you
filter by date instead and the figure differs.

When the question needs the document itself — item, quantity, price, order, receipt, shipment —
**route by module** before opening documents one by one: run **D8** for the account, branch and window,
then open each material line's document with the pull its `Module` routes to (D9 for `AR`, D10 for `AP`,
D11 for `IN`, D11-CA for `CA`; the table is in `references/acumatica/data-pulls.md`), always with the
`Document Type` + `Reference Number` pair. For cost of sales by customer, or invoiced against issued
quantity, run **D12**. Two caveats: under **summary posting** one GL line covers many documents and the
reference can be blank — report that line as summary-posted rather than guessing its documents; and an
**SO-originated invoice** is an ordinary AR document, so the sales order and shipment are reached through
its lines (D9), not a separate document.

## Step 6 — Write it up

Report shape is in `references/output.md`. Minimum content: summary with the account or account
class figure and what explains it; month-by-month matrix by year; one table per axis chosen, in the
user's nesting order; the drivers month by month; the cross-tabs when asked for; the movements that
need a human decision; basis and tie-out. For the prose under the tables, the narrative writer's
**account note** format (`velixo-commentary-writer`, output formats) is the shape.

Tables by branch or entity, segment or dimension are **complete partitions**: their rows sum exactly
to the account figure, with an unassigned row where the ledger left one. The business account table is a long list: show the
top movers, a remainder row and the total that ties, and say that the rows shown are the top movers,
not the population. Decode every code to its label: segment or dimension values to their
descriptions, branch or entity codes to names.

## Rules that do not bend

- **The axis is the user's.** Never choose a grouping for them; ask, with the choices laid out. The
  flux path's default is the single exception, and it is announced. Evidence that sits on another axis
  (where an allocation went by branch or entity, which segment or dimension value a vendor is coded
  to) goes in a sentence with its amount, or into the next cut you offer — never into a table the user
  did not ask for.
- **Totals foot.** Every table ties to the GL. A remainder is named, never absorbed.
- **Partitions are complete.** A table by branch, entity, segment or dimension adds up to the figure
  exactly; a leaf list says it shows the top movers.
- **Branches and entities sit side by side** (Acumatica branches, Sage Intacct entities; a Business
  Central connection is one company and has neither). The business account and segment or dimension
  tables may total the branches on one ledger, or the entities in one currency, but the branch or
  entity table never carries a total column: a sum across them is a company figure that belongs to
  consolidated statements and their eliminations, and each one ties to its own GL balance.
- **A short total is a finding.** When the business account table is below the GL and the account's own
  total explains the difference, the gap is its own row, never spread and never absorbed into the remainder.
- **No business account stays unattributed.** Postings with no business account are reported as such and
  often become the main finding. What they are (an allocation, a journal, a card import) is known only
  from their batch or journal and source: when the row is material, read one month of its lines (the
  lines pull) and quote the batch or entry description. Until then, call them "the lines with no business account" in every
  sentence, never "the allocation" or "the journal"; a reading of them is a hypothesis and says so,
  and the row label never carries it; the fix is a process one (require a business account on card
  imports, bank feeds and journals to the account).
- **Unassigned is a finding, not noise.** Activity with no value on the axis (a segment's unassigned
  bucket, entries with no dimension value) is reported with its share; it is never spread across the
  other values.
- **A pattern is not a cause.** "Activity tripled from February" is evidence; the reason needs a record
  or the owner's confirmation. Name a pattern by its shape and amounts ("peaks in March, then falls
  18.7%"), never by a cause ("usage-shaped", "seasonal", "what the other branches picked up"). A
  possible reason goes in a sentence that calls it a hypothesis and names the record that would
  settle it.
- **Never truncate silently.** Check the `truncated` flag before reading first-seen or last-seen dates.
- **A human owns the actions.** Cancel, renegotiate, chase, reclassify, prepay: proposals for a named
  person.