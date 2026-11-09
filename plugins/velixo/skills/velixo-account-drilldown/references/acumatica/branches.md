# Drilling down by branch

The third axis. A branch breakdown says **which part of the organization** carried the activity and
the change, before asking who it was with or how it was coded. It is the cheapest partition there is:
branches are few, every posted line carries one, and each branch has its own GL history to tie to.

## Build it

1. **Run D5** once, ledger-wide, from the first period of the earliest window to the last of the
   current one: branch × account × period with no business account join, so it ties to GL history by
   construction. Resolve branch codes to names, and to companies, with pull 3 in
   `references/acumatica/data-pulls.md`; D5 carries codes only, because `Branch.AcctName` cannot be grouped on.
2. **Branch table.** Roll D5 up to branch × account (or account class) × window: each branch's total per
   window and the change, branches side by side as columns.
3. **Tie out per branch.** Run pull 5 for each branch in scope (with `Ledger ID`) and match it per
   account and window. A branch that doesn't tie stops the run for that branch.
4. **Nest when asked.** Branch → business account comes from D1 without `Branch ID`, the same lines by business account;
   where D1 falls short of D5 on a branch, the difference is a *Business account not visible* row (the
   skill's Step 3). Branch → segment comes from pull 5 at subaccount grain, run per branch and split
   with D0.
5. **Against budget.** When the scope carries a budget, pull 6 per branch (with `Ledger ID` set to the
   budget ledger) adds a budget column and a variance to the branch table.

## Read it

- **Where the change sits.** A change concentrated in one branch has an owner: go and ask. A change
  of similar size in every branch is a rate, a policy or a central posting, and is not any one
  branch's.
- **The branch under review, in context.** Called from the flux, the branch being reviewed is the
  subject; the others show whether its movement is its own or the company's.
- **A branch that appears or disappears** inside the window (opened, closed, merged) explains a
  movement by itself; name the period.
- **Recoding between branches.** The same business account moving from one branch to another is a
  recoding, not a change in activity; net it before calling a branch up or down.
- **Intercompany.** A `CP` business account is another branch or company; say so rather than reading it
  as a supplier or customer, and expect a mirror line in the other branch.

## Report it

- One table: branches as columns (code and name), the account or account class figure per window
  (`Natural Net`), the change, and the budget and variance where in scope, with favourable or
  unfavourable stated by account type.
- **No total column across branches.** Each branch ties to its own GL history; a sum across branches
  is a company figure that belongs to consolidated statements and their eliminations, not here. (The
  business account and segment tables may total the branches on one ledger; this table does not.)
- A branch with no activity in a window shows as a dash, not a zero that was posted.
