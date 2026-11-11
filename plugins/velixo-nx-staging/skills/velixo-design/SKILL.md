---
name: velixo-design
description: >
  Velixo's visual identity: palette, logos and finance-dashboard layout. Load before producing any
  HTML or visual artefact for Velixo or a Velixo user — a dashboard, styled report, chart or
  email-ready HTML. Not for plain Markdown, Excel worksheets, Teams cards or a customer's own
  template.
license: "Copyright © 2026 Velixo Solutions Inc. All rights reserved. Proprietary: for use only by a licensed Velixo user, and not to be redistributed, distilled or modified for any other purpose. By installing or using this plugin, you agree to be bound by the agreements at https://velixo.com/agreements. See the LICENSE file shipped with this plugin."
user-invocable: false
disable-model-invocation: false
metadata:
  author: Velixo
  display_name: "Velixo Design"
---

# Velixo design system

The brand assets here (palette, logos) are Velixo's and may be used only in output produced for Velixo or its users.

This skill exists so that every HTML artefact produced by a Velixo plugin looks like it came from the
same company, without each skill carrying its own copy of the palette. Other skills in this plugin
say "apply the `velixo-design` skill" instead of listing colours — this is where those colours live.

## When it applies, and when it must not

| Surface | Apply this skill? |
|---|---|
| Claude / ChatGPT interactive HTML artifact, dashboard, rendered report | **Yes** — this is the only surface it was written for |
| Landing-page or marketing preview produced for Velixo | Yes |
| Excel worksheet produced through Velixo | **No** — reuse the connector's `tool_guidance` layout rules |
| Microsoft Teams adaptive card | **No** — Teams themes its own cards |
| Embedded inside the ERP's UI | **No** — inherit the ERP's own styling |
| A customer's own `.pptx` / `.docx` template (board packs, management reports) | **No** — fill their template, never restyle it |
| Plain Markdown commentary, prompts, query text | **No** |

The rule in one line: *style what Velixo renders; never restyle what a customer owns.*

## Velixo brand style — HTML artefacts only

Source: Damien, 2026-09-15 (logo URLs updated 2026-09-16). Applies only when a task calls for an HTML
artefact — a rendered report, a dashboard, a landing-page preview, or anything else meant to be
viewed styled rather than read as source. Prompts themselves stay plain Markdown per the prompt
library's own conventions — this does not change that.


### Logo

Reference the logo by URL. Do **not** embed it as base64. Prefer the SVG where one is listed; use the
PNG only where SVG isn't practical.

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
- **Logos** by URL only, never base64; SVG first. Pick the lockup by background, not by taste. Where
  the artefact must render with no network — a sandboxed preview, a file opened offline — omit the
  image rather than embed it, and keep the wordmark as the text "Velixo" in `--ink` (white on a
  `--brand` or `--ink` band): a broken image is worse than no logo.
- **Footer**: data source and run timestamp, and the draft/review notice when one applies.

## Guardrails

- The palette is Velixo's. In output for a customer that carries *their* brand (their template, their
  deck), their brand wins and this skill is not applied.
- Never add colours outside the palette for emphasis; use weight, size and whitespace first.
- Status colours carry meaning (`--good`, `--warn`, `--alert`); don't use them decoratively.
- Accessibility: body text in `--text` on `--bg`/`--card`; `--muted` for labels only, never for
  figures a reader must act on.

`references/dashboard-skeleton.md` is a minimal HTML page wired to the variables above — header, KPI
cards, a variance table with favourable/unfavourable cells, light and dark, a footer — ready to fill.

Built 2026-11-11 from `src/context/brand-style.md` at 1d635db.
