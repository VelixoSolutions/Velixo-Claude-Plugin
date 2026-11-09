---
name: velixo-flux-writer
description: >
  Variance and flux narrative writer: turns figures that already exist — actuals, budget or forecast
  comparisons, variance-analyzer findings, cash, KPIs — into finished commentary: an executive
  summary, board-style MD&A, flux bullets beside a variance table, or an account note. It explains
  results; it never computes them or invents a driver.
license: "Copyright © 2026 Velixo Solutions Inc. All rights reserved. Proprietary: for use only by a licensed Velixo user, and not to be redistributed, distilled or modified for any other purpose. By installing or using this plugin, you agree to be bound by the agreements at https://velixo.com/agreements. See the LICENSE file shipped with this plugin."
user-invocable: false
disable-model-invocation: false
metadata:
  author: Velixo
  display_name: "Variance/Flux Narrative Writer"
---

# Variance/Flux Narrative Writer

## What this skill does, and where it stops

A finance team's reporting package usually fails at the last mile. The numbers are right, the
comparison columns are populated, and then someone has to write two hundred words explaining why
operating expenses came in above plan — in language a board will accept, by Thursday.

This skill is that last mile. It takes results that have already been produced from an ERP and
produces the written commentary that goes around them. It deliberately does **not** pull data, build
queries or recompute figures: retrieval belongs to the Velixo connector's own guidance and, where one
is bundled, the variance analyzer for your ERP, whose findings this skill turns into prose.

**The rule that governs everything below:** every sentence must be traceable to a figure in the
input or to a fact the reader supplied. A variance that cannot be explained from what is on the
table is reported as unexplained — never filled in with a plausible-sounding reason.


## Step 1 — Settle the scope before writing a word

Five questions. If any answer is missing and cannot be inferred safely from the input, ask.

1. **What figures, at what scope?** Company, entity, branch or consolidation; ledger or book;
   financial period or range; accrual or cash basis; currency.
2. **What is the baseline?** Prior period, same period last year, original budget, current forecast,
   latest reforecast — name it, and use one spine throughout. Secondary comparisons are labelled.
3. **Who reads this, and what will they do with it?** A board approving a plan, a lender testing a
   covenant, an operating team deciding where to cut. This drives register, length and what is left
   out.
4. **Which format?** Executive summary, board MD&A prose, bulleted flux commentary, or an account
   note. See `references/output-formats.md`.
5. **What is off-limits or not yet firm?** Confidential items, anything pre-announcement, anything
   still moving because the period is open.

`references/inputs-and-scope.md` has the per-domain input checklist — what each kind of commentary
(P&L, budget, revenue outlook, cash, KPI) minimally needs, what it may and may not claim when a
piece is missing, and the structural facts about ERP data that change how a sentence is worded.

## Step 2 — Decide what is worth talking about

Rank by size of movement ignoring sign, take the spine (to the organization's coverage, else 60% of the gross movement), then add back the reader-critical,
the persistent, the unexpectedly signed and the suspiciously quiet. Set a materiality rule sized to
the entity, state it in the commentary, and apply it consistently. `references/variance-analysis.md`
has the bases and the selection passes.

## Step 3 — Explain the movement, not just the size

Work each material variance through the driver ladder — volume, rate, mix, efficiency, timing,
scope, one-time, classification — and say whether it is timing or permanent, the distinction readers
ask for first. Where the pieces are meant to add up, build a bridge whose components sum to the gap
and whose residual is named unexplained rather than absorbed. Full method, and the wrinkles specific
to budget, forecast and cash commentary, in `references/variance-analysis.md`.

## Step 4 — Write it

`references/output-formats.md` gives the deliverable shapes and how to calibrate them to the
audience. `references/narrative-craft.md` covers the sentence-level work: result → comparison →
driver → consequence, hedging matched to the evidence class (and to the analyzer's
established / indicated / open marks), the words that erode credibility, and worked weak-versus-strong
examples.

## Step 5 — Check it before it leaves

Run the self-check in `references/narrative-craft.md`: every figure matches the table it came from,
every percentage recomputes, every driver traces to a stated source, the adverse items are present
and not buried, the forward-looking language matches the evidence behind it, and the period and
scope labels are the ones the reader's own reports use.

## Guardrails

- **Never invent.** Not a figure, a percentage, a driver, a customer, vendor, project or employee
  name, an event, or a market statistic. This is the fastest way for a narrative layer to put a wrong
  number into a real filing.
- **Arithmetic is not causation.** A decomposition shows what moved; why it moved comes from a person
  or a document. Attribute it, or present it as unexplained and ask.
- **Do not smooth.** Adverse results get the same specificity as favourable ones, in the same
  register, and are not deferred to the end.
- **Hedge honestly.** Forward-looking language is calibrated to the evidence, not to mood.
  Contracted revenue and weighted pipeline do not get the same verb.
- **Preliminary means preliminary.** Say so where it applies.
- **Do not narrate the table.** If a sentence only restates a row the reader can see, cut it.
- **A human signs it off.** Anything leaving the organization is a draft for the responsible finance
  owner; flag forward-looking statements, risk characterizations, commitment language and anything
  naming a counterparty.
- **Respect the surface.** Plain Markdown is the default output. Where a rendered HTML deliverable is
  genuinely called for — a dashboard, a styled report — apply the `velixo-design` skill bundled
  alongside this one; it is the only place Velixo's palette and logo rules are authored. Never restyle
  a customer's own template, an Excel worksheet or a Teams card with it.
## Quick reference

```
SCOPE FIRST:     company/entity/branch, ledger or book, period, basis, currency — confirm, never infer
BASELINE:        name it explicitly; one spine per commentary, secondary comparisons labelled
SELECT:          spine = largest |movements| to the coverage (policy's, else 60%), exempt from
                 the % test only; add reader-critical, persistent, unexpectedly-signed and
                 suspiciously-quiet items; then the rule
MATERIALITY:     set it, state it, size it to the entity — no imported generic threshold
EXPLAIN:         volume / rate / mix / efficiency / timing / scope / one-time / classification
KEY SPLIT:       timing (reverses) vs permanent (does not) — say which, every time
BRIDGE:          components sum to the gap; residual named as unexplained, never absorbed
FORMATS:         executive summary | board MD&A prose | bulleted flux commentary | account note
HEDGING:         matched to evidence class; established → flat, indicated → hedged, open → unexplained
PERIODS:         fiscal calendar, per company; adjustment periods exist; open = preliminary
NEVER:           invent a figure, driver, name or event; assert cause without a source; bury bad news
ALWAYS:          hand off as a draft for finance review before it leaves the organization
```

Read a reference when the step that points to it is in front of you, not all four up front.
