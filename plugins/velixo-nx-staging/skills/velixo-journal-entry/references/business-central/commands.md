# Commands: building and running the entry

## Fields

| Field | API path | Required | When not given |
|---|---|---|---|
| Journal batch | `code` (header `Key`) | yes | **Ask**, with the batches from B1 that have no balancing account. |
| Account | `journalLines.accountNumber` | yes | **Ask.** A name is resolved with A1; several matches are a question. `accountType` is always `G/L Account`. |
| Posting date | `journalLines.postingDate` | yes | **Ask.** `YYYY-MM-DD`, the same on every line. Never assume today for an accrual or a reclass. |
| Document number | `journalLines.documentNumber` | yes | Propose one from the user's words (`ACCR-2026-10`) and show it; the same on every line. |
| Description | `journalLines.description` | yes | Propose one from the user's words, the same on every line unless the user gives line memos. |
| Amount | `journalLines.amount` | yes | Signed: a debit positive, a credit negative. Two decimals at most. |
| Dimensions | `journalLines.dimensionSetLines.code` and `.valueCode` | no | None, except a default dimension the account carries. On a reversal or a reclass, every dimension of the original from JE2. |

## The request

Build once, through `prepare_commands`, for any number of lines:

> Add general journal lines to the existing journal batch {{JournalCode}}, with any number of lines,
> each with G/L account number {{Account}}, posting date {{PostingDate}}, document number
> {{DocumentNo}}, description {{Description}}, amount {{Amount}} (positive for a debit, negative for a
> credit), and any number of dimensions per line, each with dimension code {{Dimension}} and dimension
> value code {{DimensionValue}}. Save the lines. Do not post.

When the user asked to post, end it with "Save the lines, then post the journal batch." instead.

The build returns a header (`Key` on the batch's `code`), a line set (account type, account, posting
date, document number, description, amount), a split set (dimension code and value) and a finish set
(`Save`, then `post` when asked). Run them in one call: one row per line, each line's dimensions as
its own `rows`, `[]` for a line without one. Business Central refuses `leaveOpen` and `continuation`
(`BC_RECORD_SESSION_UNSUPPORTED`), so every line goes in the one call.

**To post lines already waiting in a batch**, without adding any, run the header `Key` and an
`Action` `post`, after B2 shows the batch holds only the entry the user asked to post.

## What the live run showed (bc-dev, 2026-10-27)

- **The write works as built.** VXTEST-001 (debit 8640 10.00 with `DEPARTMENT` `ADM`, credit 8630
  10.00) went into `TEMPTEST` as two lines; B2 read them back with `ADM` as shortcut dimension 1, no
  balancing account, and dimension set 19 on the first line.
- **Posting posts the batch.** `Key` `TEMPTEST` and `post` posted VXTEST-001 as transaction 1430 and
  emptied the batch. On that day `DEFAULT` held 53 unposted lines and `HAMSTER-1` 6: posting either
  would have posted them too.
- **VAT is split off the line.** 8630 and 8640 carry general posting type `Purchase`, VAT groups
  `DOMESTIC` and `STANDARD` (25 %): each 10.00 line posted 8.00 to its account and 2.00 to 5630
  Purchase Tax, both with the line's dimensions.
- **A reversal is a new entry.** VXTEST-002 (the same lines negated, `ADM` copied, "Reversal of
  VXTEST-001") posted as 1431 and netted 1430 to zero on 8640, 8630 and 5630; `Reversed` stays false
  on both.
- **Clearing the tax code does not remove VAT.** VXTEST-003, written with the API's `taxCode` set to
  an empty value, read back with a blank VAT product posting group but VAT % 25 and VAT 2.00 still on
  each line, and (`DOMESTIC`, blank) has no VAT posting setup. It was not posted.
- **This skill never deletes a line.** The connector can, one line per call (`journalLines`, keyed on the
  batch's id and the line number; it refuses a delete by document number), but a line written by
  mistake is the user's to delete in Business Central, or with a yes for that one line.

## The preview

1. **Headline:** "**Add a journal entry** ACCR-2026-10 to batch TEMPTEST, dated 31 Oct 2026, and
   post it" — or "and leave it unposted in the batch".
2. **Lines** as a table: account and name, debit, credit, description, each dimension; a total row.
3. **VAT Business Central will post:** for each line on a VAT-carrying account, the net amount on the
   account, the VAT and its account, and the rate (`8640: 8.00, VAT 2.00 to 5630 at 25 %`). "No VAT"
   when no line carries any.
4. **Left to Business Central, and why:** dimensions left empty; default dimensions the accounts add.
5. **Checked:** batch has no balancing account and, when posting, holds no other line; accounts
   allow direct posting; dimension values valid; balanced; no entry with the same description.
6. **For a reversal or a reclass:** the entry it offsets, by document number, posting date and
   transaction number.
7. **The question:** **Add the journal entry** (or **Add and post the journal entry**) and **No**.

## Errors

A refused write is Business Central's message, relayed as it came, and nothing was written unless B2
or JE1 shows the entry. A timeout may have written: check B2 and JE1 before anything else, and never
re-run a write blind. `BC_RECORD_CHANGED` means someone changed the batch after the batch was read:
read it again and ask before writing.
