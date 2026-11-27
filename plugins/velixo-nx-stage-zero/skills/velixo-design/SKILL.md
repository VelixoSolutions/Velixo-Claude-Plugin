---
name: velixo-design
description: >
  Velixo's palette, logos and layout for any dashboard, chart, styled report, document or
  presentation built from Velixo or ERP results: Excel, Word, PDF, PowerPoint or slide deck, HTML.
  Load it before building one, ahead of any generic chart, design, document, slide or spreadsheet
  skill, also when the request follows an analysis ("make it a dashboard", "put it in a deck",
  "export to Excel"). Not for plain Markdown answers, Teams cards or a customer's own template.
license: "Copyright © 2026 Velixo Solutions Inc. All rights reserved. Proprietary: for use only by a licensed Velixo user, and not to be redistributed, distilled or modified for any other purpose. By installing or using this plugin, you agree to be bound by the agreements at https://velixo.com/agreements. See the LICENSE file shipped with this plugin."
user-invocable: false
disable-model-invocation: false
metadata:
  author: Velixo
  display_name: "Velixo Design"
---

# Velixo design system

The brand assets here (palette, logos) are Velixo's and may be used only in output produced for Velixo or its users.

This skill exists so that every dashboard and file produced from Velixo results looks like it came
from the same company, without each skill carrying its own copy of the palette. Other skills say
"load the `velixo-design` skill" instead of listing colours — this is where those colours live.

**Load it first.** When the user asks for a dashboard, a chart, a styled report, a document, a
presentation or a file of the results (Excel, Word, PDF, PowerPoint or slide deck, HTML) — in the
first request or as a follow-up after an analysis — load this skill before writing any of it, and
before a generic chart, design, document, slide or spreadsheet skill (those bring a brand-neutral
palette). Its rules win over theirs; theirs still apply to mechanics this skill does not cover.

**Where the format has its own look** — a host's editable document or slide-deck type with a fixed
theme — apply the palette wherever the format takes it, and otherwise keep the Velixo logo (or the
text "Velixo"), the scope line and the "Prepared with Velixo" footer line (unless the user asked to
leave it out).

## When it applies, and when it must not

| Surface | Apply this skill? |
|---|---|
| Interactive HTML artifact, dashboard, chart, rendered report | **Yes** |
| Excel workbook, Word or other document, PDF, PowerPoint or other slide deck built from Velixo results | **Yes** — see "Files" below |
| Velixo connector workbook (`create_workbook_from_query`) | **Not restyled** — the connector builds it and sets its layout; never rebuild it as a local file to style it, which drops its live formulas. Offer a styled Excel snapshot of the same figures beside it ("Files" below) |
| Landing-page or marketing preview produced for Velixo | Yes |
| Microsoft Teams adaptive card | **No** — Teams themes its own cards |
| Embedded inside the ERP's UI | **No** — inherit the ERP's own styling |
| A customer's own `.xlsx` / `.pptx` / `.docx` template (board packs, management reports) | **No** — fill their template, never restyle it |
| Plain Markdown answer or commentary, prompts, query text | **No** |

The rule in one line: *style what Velixo produces; never restyle what a customer owns.*

## Velixo brand style — dashboards, HTML artefacts and files

Source: Damien, 2026-09-15 (logo URLs updated 2026-09-16). Applies when a task calls for something
meant to be viewed styled rather than read as source: a dashboard, an HTML artefact (a rendered
report, a landing-page preview, an email-ready page), or any file built from Velixo results (Excel,
Word, PDF, PowerPoint or another format). Prompts
themselves stay plain Markdown per the prompt library's own conventions — this does not change that.


### Logo

The logo always comes from its URL below: the plugin ships no logo file, so every user gets it from
the same place.

- **An HTML page** references the URL. Do **not** embed it as base64. Prefer the SVG where one is
  listed; use the PNG only where SVG isn't practical.
- **A file** (Excel, Word, PDF, PowerPoint) cannot show a remote image: download the **PNG** from its
  URL while building the file and insert it as an image. Files take the PNG, not the SVG.
- **Where the logo cannot load** — no network, a sandboxed preview, the download refused — leave the
  image out and write the wordmark as the text "Velixo" in `--ink` (white on a `--brand` or `--ink`
  band). A broken image is worse than no logo.

**Icon only** — the Velixo symbol alone, for compact placements (favicons, avatar-style badges, tight
corner marks). Transparent background, works on light or dark:

- SVG: `https://secure.velixo.com/logos/symbol-gradient.svg`
- PNG: `https://secure.velixo.com/logos/symbol-gradient.png`

**Full logo, navy text** — icon + "Velixo" wordmark in navy blue text, transparent background. Light
or neutral backgrounds only — the text is a fixed navy and won't read on dark ones:

- SVG: `https://secure.velixo.com/logos/logo-full-gradient-navy.svg`
- PNG: `https://secure.velixo.com/logos/logo-full-gradient-navy.png`

**Full logo, white text** — same icon + wordmark lockup, with the "Velixo" text in white, transparent
background. Dark or brand-color backgrounds only — the text is a fixed white and won't read on light
ones:

- SVG: `https://secure.velixo.com/logos/logo-full-gradient-white.svg`
- PNG: `https://secure.velixo.com/logos/logo-full-gradient-white.png`

### Palette (CSS custom properties — use these variable names)

```css
--brand:#345ad9; --brand-dark:#2e429b; --ink:#243261; --text:#242424; --muted:#6e747d;
--bg:#fbfcfe; --card:#ffffff; --panel:#f3f6fc; --line:#e5e8ee;
--good:#1f9d57; --alert:#f74245; --warn:#e08a2b;
```
## Layout rules for finance dashboards

These are the conventions the palette above was tuned for. `references/dashboard-skeleton.md` has a
copy-ready HTML starting point.

- **Page**: `--bg` background, content in a centred column, generous whitespace. Header carries the
  full logo (navy on light backgrounds, white on `--brand`/`--ink` bands) and the scope line —
  company, branch, ledger, period, currency, run date, and "Preliminary" when the period is open.
- **Cards** (`--card` on `--bg`, 1px `--line` border, soft radius): one headline figure each, label
  in `--muted`, the figure large in `--text`, the comparison (vs. budget, vs. prior) beneath it.
- **Status is words plus colour, never colour alone**: `--good` / `--warn` / `--alert` accompany
  "On track" / "Watch" / "Off track" (or the organization's own scheme).
- **Tables**: numbers right-aligned, consistent decimals, units in the header, negatives in the
  organization's convention; header row on `--panel`.
- **Charts**: series in this order — actual `--brand`; the comparison (budget, forecast or prior
  period) `--ink`; prior-year context `--muted`; a fourth series `--brand-dark`; a fifth and sixth
  `--brand` at 60% and 35% opacity. More than six series is a table, not a chart. Status colours are
  never series colours; `--alert` marks a genuine exception only.
- **Variance cells and heatmaps**: colour by meaning, never by arithmetic sign — favourable on a
  12% tint of `--good`, unfavourable on a 12% tint of `--alert`, the figure itself in `--text` with
  its sign or word visible. A diverging heatmap scales those two tints by magnitude with `--card` at
  zero, and carries a legend that says which direction is favourable.
- **Dark mode**: the artefact hosts render in light and dark. Redefine the same variable names under
  `@media (prefers-color-scheme: dark)` (guarded by `:root:not([data-theme="light"])`) and again
  under `:root[data-theme="dark"]`, and give `body` an explicit background. The dark values in
  `references/dashboard-skeleton.md` are derived from the palette for dark surfaces, not a second
  brand palette: `--brand` lifts to `#5b7cf0` for contrast on dark, status colours lift one step,
  surfaces step down from `--ink`.
- **Typography**: a system sans-serif stack (`-apple-system, Segoe UI, Roboto, Helvetica, Arial,
  sans-serif`). Don't load web fonts; artefacts must render offline and inside sandboxed previews.
- **Logos** from their URL, as the Logo section above says: a page references the URL (SVG first,
  never base64). Pick the lockup by background, not by taste. Where the artefact must render with no
  network — a sandboxed preview, a page opened offline — omit the image and keep the wordmark as the
  text "Velixo" in `--ink` (white on a `--brand` or `--ink` band): a broken image is worse than no logo.
- **Footer**: the line "Prepared with Velixo · <data source> · run <date>", and the draft/review
  notice when one applies. Every page, dashboard and file carries it by default, even where the format
  keeps its own look. When the user asks to leave it out ("no Velixo footer", "remove the branding
  line"), leave out the "Prepared with Velixo" line for that output and the rest of the conversation;
  keep the data source, run date and any draft notice, which say what the figures are.

## Files: Excel, PDF, Word, PowerPoint

A file has no CSS variables: use the palette's hex values. The dashboard rules above still hold
(status in words plus colour, colour by meaning, the series order, the footer line).
`references/file-recipes.md` has a copy-ready Python starting point for each format (`openpyxl`,
`python-docx`, `python-pptx`, `reportlab`), with the palette, the logo download and its fallback, and
the footer already in place: start from it rather than from a blank file.

**The logo in a file**: download the PNG from its URL (navy lockup on a light background, white on a
`--brand` or `--ink` band) and insert it as an image; a file cannot show a remote image. If the
download fails, write the text "Velixo" in `--ink` (white on a band) in its place, in every format.

- **Excel**: a title row with "Velixo", the report name and the scope line (company, branch, ledger,
  period, currency, run date); a header row filled `--ink` `#243261` with bold white text; figures
  right-aligned with a thousands separator and one number format per column; totals bold over a thin
  `--line` `#e5e8ee` top border; favourable cells filled `#e4f3eb` and unfavourable `#fee8e9` (the
  12% tints of `--good` and `--alert`), the figure in `--text` `#242424`. Freeze the panes under the
  header. Charts take the series order above. Keep the figures as numbers, never text.
- **PDF and Word**: the logo and the scope line in the header; headings in `--ink`; table headers on `--panel` `#f3f6fc` with a
  `--line` rule; the footer line and any draft notice in the page footer.
- **PowerPoint**: a title slide on a `--brand` `#345ad9` or `--ink` band with the white logo, then
  content slides on white with headings in `--ink` and one message per slide; charts and tables as
  above.

## Guardrails

- The palette is Velixo's. In output for a customer that carries *their* brand (their template, their
  deck), their brand wins and this skill is not applied.
- Never add colours outside the palette for emphasis; use weight, size and whitespace first.
- Status colours carry meaning (`--good`, `--warn`, `--alert`); don't use them decoratively.
- Accessibility: body text in `--text` on `--bg`/`--card`; `--muted` for labels only, never for
  figures a reader must act on.

`references/dashboard-skeleton.md` is a minimal HTML page wired to the variables above — header, KPI
cards, a variance table with favourable/unfavourable cells, light and dark, a footer — ready to fill.
`references/file-recipes.md` does the same for Excel, Word, PowerPoint and PDF files.

Built 2026-11-27 from `src/context/brand-style.md` at bf13e32.
