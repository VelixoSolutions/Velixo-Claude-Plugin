# Commands: building and running the entry

## Fields

| Field | REST path | Required | When not given |
|---|---|---|---|
| Journal | `glJournal.id` | yes | **Ask**, with the active journals of the book from B1. |
| Posting date | `postingDate` | yes | **Ask.** `YYYY-MM-DD`. Never assume today for an accrual or a reclass. |
| Description | `description` | yes | Propose one from the user's words and show it in the preview. |
| Account | `lines.glAccount.id` | yes | **Ask.** A name is resolved with C1; several matches are a question. |
| Entity | `lines.dimensions.location.id` | yes | **Ask** when the company has several entities. |
| Debit or credit | `lines.txnType` | yes | `debit` or `credit`, from the column the amount is in. |
| Amount | `lines.txnAmount` | yes | Positive, two decimals at most. |
| Line memo | `lines.description` | no | The entry's description. |
| Department, class, project, task, customer, vendor, employee, item, warehouse, contract | `lines.dimensions.<name>.id` | no | `null` (left empty). On a reversal or a reclass, the original line's value from JE1. |

## The request

Build once, through `prepare_commands`, for any number of lines:

> Create a general ledger journal entry in journal {{JournalSymbol}} dated {{PostingDate}} with
> description {{Description}}, without setting its state so Sage Intacct applies its default, with
> any number of lines, each with GL account {{Account}}, location {{Location}}, transaction type
> {{TxnType}}, amount {{Amount}}, line description {{LineDescription}}, department {{Department}},
> class {{Class}}, project {{Project}}, task {{Task}}, customer {{Customer}}, vendor {{Vendor}},
> employee {{Employee}}, item {{Item}}, warehouse {{Warehouse}} and contract {{Contract}}. Read back
> id and state.

The build returns a header, a line set and a finish set (Save, then `id` and `state`); run them in
one call with one row per line. Every row binds every dimension: the value, or `null` for an empty
one, which drops that field from the write (the result lists it in `skippedFields`).
Ask for a draft only when the user wants one: say "saved as a draft".

## What the live run showed (skyline, 2026-10-25)

- `state: draft` on create is accepted (268419). `state: posted` is refused: "REST-1064: Payload
  contains errors Not a valid state".
- Without a state, Sage applies the journal's default: `GJ` returned `submitted` (268420, waiting for
  approval), `GL_T` returned `posted` (268421).
- The connector exposes no Post, Approve or Reverse workflow for journal entries: a draft or a
  submitted entry is finished in Sage, and a reversal is a new entry (268422 reversed 268421; the two
  net to zero on 60340 and 60350).
- Dimensions copy: 268424 (department 200 on both lines) was reversed by 268425 with the same
  department, read back by JE1. A class Sage does not accept is refused for the whole entry ("Class
  '100' specified is not valid") and nothing is written.
- A header that starts with `Clear` was refused: "REST-1064: Payload contains errors / is not a valid
  field". Use a build without it.
- Draft and submitted entries' lines are in `journal-entry-line`, so the GL pulls of the other skills
  filter on `Line.state = 'posted'`.

## The preview

1. **Headline:** "**Create a journal entry** in journal GL_T (Accrual book), dated 25 Oct 2026, entity
   100 — Sage posts it, or sends it for approval if the journal requires approval."
2. **Lines** as a table: account and title, entity, debit, credit, memo, each dimension given; a total
   row.
3. **Left to Sage, and why:** every dimension left empty, the line memos that take the description.
4. **Checked:** journal active, accounts active, balanced, no entry with the same description.
5. **For a reversal or a reclass:** the entry it offsets, by journal, transaction number and record id.
6. **The question:** **Create the journal entry** and **No**.

## Errors

A refused write is Sage's message, relayed as it came, and nothing was written unless JE2 shows the
entry. A timeout may have written: check JE2 before anything else, and never re-run a write blind.
