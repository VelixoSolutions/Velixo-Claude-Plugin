# Inputs: every field a journal entry carries

The entry arrives in any shape: a sentence ("accrue June utilities, 1,250 to 63300 against 20150"),
debit and credit lines typed after the command, a pasted table, or a spreadsheet laid out like the
Journal Transaction Writeback template (journal fields at the top, one row per line below). Map what
the user gave onto the fields below by meaning, not by column position, and say which column you read
for which field when the headers were not the ones below.

Ask for everything missing in **one** message, with choices the user can tap where there are choices
(branches from J1, open periods from J2). Never fill a business value yourself.

## Header (journal fields)

| Field | Acumatica field | Required | When not given |
|---|---|---|---|
| Connection | — | when several are usable | Ask which one; never guess. One usable connection is used as is. |
| Branch | `BatchModule.BranchID` | yes | **Ask**, with the active branches from J1 as choices. Never default to the first one. |
| Ledger | `BatchModule.LedgerID` | yes | The branch's posting (actual) ledger from J1. A different ledger (budget, statistical, reporting) only when the user names it; say it was not checked against J1. |
| Transaction date | `BatchModule.DateEntered` | yes | **Ask.** Accruals and reclasses are dated at period end, not today, so today is never assumed. Pass as `YYYY-MM-DD`. A date typed `MM-DD-YYYY` (`10-15-2026`) is read month first; a date that could be either (`03-04-2026`) is asked about. |
| Post period | `BatchModule.FinPeriodID` | yes | The period from J2 whose start date ≤ the transaction date < its end date. A period the user gives wins, and must still be open; when it does not contain the date, say so in the preview (Acumatica allows it; reviewers notice it). Passed as `MMYYYY`, no separator. |
| Currency | `BatchModule.CuryID` | yes | The branch's base currency from J1. A foreign currency only when the user names it: amounts are then in that currency and Acumatica converts at its default rate type and the rate effective on the date; a missing rate is Acumatica's error to relay. A rate override is out of scope. |
| Auto reversing | `BatchModule.AutoReverse` | no | Off (`null`, Acumatica's default). On when the user asks for a reversing entry or an accrual that reverses next period; say the reversal is generated on release, into the next period. |
| Description | `BatchModule.Description` | yes | Ask, or propose one from the user's own words ("June 2025 utilities accrual") and show it in the preview for them to change. |

## Writeback settings

| Setting | Default | What it does |
|---|---|---|
| Automatically release | **No** | No: the batch is created **on hold** for a reviewer. Yes: after the read-back ties, the batch is taken off hold and released in a second batch. Release posts to the ledger and cannot be undone (only reversed by another entry), so the preview says so and the confirmation button names it ("Create and release the journal entry"). |
| Open after import | **Yes** | The reply leads with the batch number and where to open it (Journal Transactions, module GL, batch number). The connector cannot open a browser window; never invent a URL. No: the batch number is still reported, after the tie-out. |
| Last batch nbr | — | Output: the batch number Acumatica assigned, reported at the end of every run. Input, when the user supplies one from a previous run: check it with J6 before building. If that batch already carries these lines, the entry has been written; ask before writing it again. Editing an existing batch is out of scope. |

Tax settings (create tax lines, skip tax validation) are not supported. If the user asks for them,
say so and that tax lines can be added on the Journal Transactions screen.

## Lines

| Field | Acumatica field | Required | When not given |
|---|---|---|---|
| Branch | `GLTranModuleBatNbr.BranchID` | yes | The header branch. A line in another branch makes the entry inter-branch: Acumatica generates the due-to / due-from lines on release from the inter-branch account mapping, and refuses when none is set up. Say so in the preview. |
| Account | `GLTranModuleBatNbr.AccountID` | yes | **Ask.** Never infer one from a description. A name ("utilities") is resolved against J3 and shown with its code; more than one match is a question. |
| Subaccount | `GLTranModuleBatNbr.SubID` | yes, when the tenant has segments | Ask once for all lines missing one, offering "use each account's default subaccount" as a choice. Accepted with or without separators (`000-000`, `000000`); passed without. |
| Project | `GLTranModuleBatNbr.ProjectID` | no | `null`: Acumatica's non-project default (`X`, "Non-Project Code." on gab-dev). Say so in the preview. |
| Project task | `GLTranModuleBatNbr.TaskID` | with a project | `null` without a project. A project given without a task: ask, unless the user says to use the project's default task. |
| Cost code | `GLTranModuleBatNbr.CostCodeID` | no | `null`. Only meaningful with a project on a tenant with cost codes. |
| Quantity | `GLTranModuleBatNbr.Qty` | no | `null`. Statistical; it does not change the amount. |
| External UoM | `GLTranModuleBatNbr.UOM` | with a quantity | `null`. A quantity without a unit: ask. |
| Ref. number | `GLTranModuleBatNbr.RefNbr` | no | `null`. |
| Debit | `GLTranModuleBatNbr.CuryDebitAmt` | one of debit or credit | `0` when the line is a credit. In the batch currency. |
| Credit | `GLTranModuleBatNbr.CuryCreditAmt` | one of debit or credit | `0` when the line is a debit. |
| Transaction description | `GLTranModuleBatNbr.TranDesc` | no | The header description, shown in the preview as such. |
| Non billable | `GLTranModuleBatNbr.NonBillable` | no | `null` (off). Only meaningful on a project line. |
| Tax ID, Tax category | — | — | Not supported (see tax settings above). |

**Amounts.** Positive numbers with at most two decimals, thousands separators removed (`1,250.00` →
`1250.00`). A line written as a signed amount (`-1,250.00`) or "Dr/Cr" in one column is asked about,
not flipped: conventions differ. A line with both a debit and a credit, or neither, is asked about.

**Volume.** Any number of lines in one batch. Up to 50 go in one call; a longer entry goes in
consecutive calls of up to 50 and is still saved once, as one batch (`references/acumatica/screen-batches.md`).
Above 1,000 lines, say how many and ask whether an Acumatica import scenario would suit better.
