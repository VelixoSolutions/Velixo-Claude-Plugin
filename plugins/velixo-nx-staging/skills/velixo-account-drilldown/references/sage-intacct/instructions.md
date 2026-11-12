
# Account drill-down for Sage Intacct

These are the Sage Intacct instructions of this skill. A file named `references/<file>.md` here, or in
the method this file includes, is `references/sage-intacct/<file>.md`.


## What it does, and where it stops

A flux tells you an account moved. This skill tells you **with whom**, **where it was coded** and
**in which entity**: every journal line on the account in one reporting book, grouped by the
business account on the line (its customer, vendor or employee dimension), by the other dimensions
(department, class, project) and by entity, window by window.

**Business account** is the term used throughout for the customer, vendor or employee on a line; use
the word that fits the account in the output. **Dimensions** are what subaccount segments are in
Acumatica, and the **entity** (the location dimension at the top level) plays the branch's role.

It produces evidence for the variance analyzer's ladder: transaction detail and dimension breakdown
in one pass. It does **not** decide causes.

## Step 1 — Inputs

**The grouping is the user's choice, never yours.** Invoked on its own, ask which axis or axes to
break the figure out on (business account, a dimension, entity), and in what order, in one message
with choices the user can tap, and run no pull until it has the answer. Called from a flux with no
axes passed, the default is business account and department, and the output says so.

Ask in the same message for whatever else is missing:

1. **Connection:** already chosen in `SKILL.md`.
2. **Book:** Accrual, unless the user names another reporting book (`references/sage-intacct/data-pulls.md`, B1).
   One book per run.
3. **Entities:** every entity with activity on the account, unless the user names some. Each entity
   is read and tied on its own; entities with different base currencies are never added together.
4. **Accounts:** one or more account numbers. Every pull is per account.
5. **Windows:** the current window, and each comparison window the user asked for; no comparison
   unless one is named. Every window goes in the same function call, one invocation each; pull the
   months in between only when comparing. Months resolve as *Periods* below says.
6. **Depth:** one table per axis chosen; the documents behind a cell only where it pays (Step 5).

### Periods

**Periods: a month is the unit.** The user may name a month ("June 2018"), a month-end date ("30
June 2018") or an Acumatica-style period (`062018`, `MMYYYY`); each names one calendar month, and
none needs to be rewritten by the user. Run **P1** (`gl-reporting-period-for-month`) with that
month's first and last day:

- **A row back:** the window is that Sage reporting period. Name it in the basis block ("Month Ended
  June 2018, 1–30 Jun 2018").
- **No row:** the window is the calendar month's dates, and the basis block says no reporting period
  matched.

A range of months is each end resolved the same way, from the first month's first day to the last
month's last day. A date that is not a month end is a date window as given. The GL is read on entry
date over the window either way, because a journal line carries no reporting period.

### Connection and ERP

The connection was chosen, and its `type` read, in `SKILL.md` before this file was opened: it is a
Sage Intacct connection. Never run these functions or queries on a connection of another type.

## Step 2 — Pull the data

Every pull is in `references/sage-intacct/data-pulls.md`, which also maps each axis to its dimension; run each as
written. Read the chart (C1) and the entities (E1) once per run. Then follow its sequence: the
members that moved (F4), then in one call the account's turnover for each window (F1), each member's
(F5) and their names (F6). Past about 60 invocations, switch the axis to its query (D1 or D2), one
per window, and say so. A run of more than about 40 calls in all is too large for one answer: say
how many it needs and offer to narrow it first.

## Step 3 — Tie out before analysing

- **Every table ties to F1** (T1 when F1 fell back): the members of an axis (F5, F7, or D1 and D2),
  summed per account, entity and window, equal the account's activity for the same three. D3 sums to
  the cell it was pulled for.
- **A short total is a finding, not a stop.** When an axis table is below F1, report both figures
  and show the difference as its own row, never spread across the others (the shared rules below).
  With F5 that row is the axis's unassigned lines, which F4 never lists. First rule out the
  mechanical causes: a failed item in the batch, a truncated result, a different book, the child flag
  left out, or the 25,000-line cap of a query; rerun before calling it a finding.
- **Sign:** the functions return debit-positive figures and the queries debit and credit rows.
  Compute `Natural Net` from C1's normal balance (the figure as returned, or debit less credit, on a
  `debit` account; its opposite on a `credit` one) and present it everywhere; state the convention
  once.

## Step 4 — Classify and read

| Class | How to recognise it | How to report it |
|---|---|---|
| Customer | an F5 member on `CUSTOMER`, or `Customer ID` set in D1 | By name (F6) |
| Vendor | an F5 member on `VENDOR`, or `Vendor ID` set in D1 | By name (F6) |
| Employee | an F5 member on `EMPLOYEE`, or in D1 `Employee ID` set and no customer or vendor | By name (F6), or the ID |
| No business account | the account's F1 less its members' sum, or none of the three in D1 | Its own line, **No business account**; never allocated. AR batch summaries ("… Batch Summary Entry") carry no customer |

A line with both a customer and an employee (time-and-materials revenue) is the customer's on a
revenue account: say so, and never add a customer axis and an employee axis together, since the same
line is in both. For a dimension axis, an empty value is the **unassigned** row of that axis.
Then read the patterns in `references/sage-intacct/patterns.md`.

## Step 5 — Go to documents only where it pays

For a cell that matters and is not self-explanatory, pull **D3** with the cell's account, entity and
window: the posted lines, with journal symbol, entry number, descriptions and document. The journal
symbol routes each line: ARJ and CRJ to AR, APJ and CDJ to AP, GJ to manual entries, RR to revenue
recognition. A batch-summary line stands for many documents: report it as summary-posted rather than
guessing which.

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

Every amount is in its entity's base currency (E1), named beside it. The basis block says whether
each entity includes its child locations (the default) or reads the location only, as the user asked,
so the user can ask to exclude them.

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