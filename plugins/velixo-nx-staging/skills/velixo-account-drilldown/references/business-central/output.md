# Output

*Branch* below is the ERP's branch or entity, and *segment* a subaccount segment or a dimension
(department, class, project and the like).

## Default output

Inline Markdown in the conversation: a short summary, the account (or account class) matrix, one table per axis chosen
in the user's nesting order (branch and segment tables complete; the business account table top 15 by
absolute change, with the remainder grouped and the total), and the movements that need a human
decision. Offer a file only when the result is long or the user asks.

## Presentation rules

- **The order is the user's.** Tables appear in the nesting order they gave; with several axes and no
  order, branch, then segment, then business account.
- **Partitions foot.** A branch or segment table's rows add up to the account figure exactly, with an
  unassigned row where the ledger left one. Children always sum to their parent.
- **Leaf lists say what they are.** The business account table shows the top movers, a remainder row, a
  *Business account not visible* row where the join dropped lines, and the tying total, and states that
  the rows shown are the top movers rather than the population.
- **Codes become labels.** Segment values carry their descriptions, branches their names.
- **Budget columns** sit beside the actual in the same natural sign, with the variance labelled
  favourable or unfavourable by account type.

## Report file (PDF or HTML)

When the user asks for a document, keep this order:

1. **Summary** — account or account class figure in each window, the change, and the three or four drivers with
   amounts. If the drivers add up to more than the change because savings offset them, say so; never
   claim a driver share that doesn't add up.
2. **Month by month** — grouped bar chart (one bar per year per month) and the matrix with change rows.
3. **Axis tables**, in the user's nesting order — the branch table (branches as columns, no total
   across them), one table per segment (unassigned first), and the business account table (every
   business account, sorted by change, with a total row that ties to the GL).
4. **Drivers month by month** — line chart over the whole window (current, comparison and in-between
   months) for the driver lines, with their table.
5. **Cross-tabs**, when asked for — business account × segment, branch × business account.
6. **Findings needing a decision** — each with the evidence, the open question and who should answer.
7. **Basis and tie-out** — source, scope, sign, period basis, and the reconciliation result per table.

Apply the `velixo-design` palette for Velixo-produced files, and the narrative writer's **account
note** format for the summary paragraph. Render the pages and look at them before delivering. If the
PDF is produced with ReportLab and its built-in fonts, avoid characters outside WinAnsi (the minus
sign U+2212, Greek delta, arrows): use a hyphen, "Change" and "to", and escape `&` in table cells.