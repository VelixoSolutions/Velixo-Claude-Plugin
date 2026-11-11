
# Account drill-down for Business Central

These are the Business Central instructions of this skill. A file named `references/<file>.md` here, or in
the method this file includes, is `references/business-central/<file>.md`.


## What it does, and where it stops

A flux tells you an account moved. This skill tells you **with whom** and **where it was coded**:
every G/L entry on the account, grouped by the source it came from (the customer, vendor, bank
account, fixed asset or employee) and by the value of any dimension (department, project, customer
group, a custom one), window by window.

**Business account** is the term used throughout for an entry's source; use the word that fits the
account in the output. **Dimensions** are what subaccount segments are in Acumatica. Business
Central has no branch: a company is its own connection, so the **company** is the whole scope of a
run, never an axis inside it.

It produces evidence for the variance analyzer's ladder: transaction detail and dimension breakdown
in one pass. It does **not** decide causes.

## Step 1 — Inputs

**The grouping is the user's choice, never yours.** Invoked on its own, ask which axis or axes to
break the figure out on (business account, a dimension), and in what order, in one message with
choices the user can tap, and run no query until it has the answer. Offer the company's global
dimensions by name (G0) and say any other dimension code works too. Called from a flux with no axes
passed, the default is business account and global dimension 1, and the output says so.

Ask in the same message for whatever else is missing:

1. **Connection:** already chosen in `SKILL.md`. A connection with several companies is
   listed once per company: the company is the connection.
2. **Accounts:** one or more G/L account numbers, posting accounts only. Every pull is per account.
3. **Windows:** the current window, and each comparison window the user asked for; no comparison
   unless one is named. Month by month is one call per month (Business Central cannot group by part
   of a date), so pull the months in between only when comparing. Months resolve as *Periods* below
   says.
4. **Depth:** one table per axis chosen; the entries behind a cell only where it pays (Step 5).

### Periods

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

### Connection and ERP

The connection was chosen, and its `type` read, in `SKILL.md` before this file was opened: it is a
Business Central connection. Never run these queries on a connection of another type.

## Step 2 — Pull the data

Every query is in `references/business-central/data-pulls.md` with its parameter table; run each as written. Read the
setup (G0) once per run. Then, per account and window: T1 for the account's total and category, and
D1 (business account) or D2 (a dimension, both of its levels) for the axis chosen. A run
of more than about 40 calls (accounts × axes × months) is too large for one answer: say how many it
needs and offer to narrow it first.

## Step 3 — Tie out before analysing

- **Every table ties to T1:** D1, and D2's values plus its "no value" row, summed per account and
  window, equal the account's `Net Change` from T1 for the same window. D3 sums to the cell it was
  pulled for.
- **A short total is a finding, not a stop.** When an axis table is below T1, report both figures
  and show the difference as its own row, never spread across the others (the shared rules below).
  First rule out the mechanical causes: a truncated result, or the 10,000-entry cap reached; rerun
  with a shorter window before calling it a finding.
- **Sign:** every amount comes back debit-positive. Compute `Natural Net` from T1's `Account Category`
  (`Assets`, `Cost of Goods Sold`, `Expense`: as read; `Liabilities`, `Equity`, `Income`: negated)
  and present it everywhere; state the convention once.

## Step 4 — Classify and read

| Class | How to recognise it | How to report it |
|---|---|---|
| Customer | `Source Type` = `Customer` | By number, and by name when D3's descriptions or the user give it |
| Vendor | `Source Type` = `Vendor` | The same |
| Bank account, fixed asset, employee | that `Source Type` | By type and number |
| No business account | `Source Type` blank | Its own line, **No business account**; never allocated. General journal lines usually carry no source |

For a dimension axis, the "no value" row from D2 is the **unassigned** row of that axis. Then read
the patterns in `references/business-central/patterns.md`.

## Step 5 — Go to entries only where it pays

For a cell that matters and is not self-explanatory, pull **D3** with the cell's account and
window: the entries, with document type and number, description, source, global dimensions and
source code. The source code routes each entry: `SALES` and `CASHRECJNL` to receivables,
`PURCHASES` and `PAYMENTJNL` to payables, `GENJNL` to manual journals, `INVTPCOST` to inventory
costing. For one dimension value's entries, D3's `Dimension Set ID` groups them; the global
dimensions are on the row itself.

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

Every amount is in the company's local currency (G0), named beside it.

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