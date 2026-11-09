# Dashboard skeleton

A minimal, self-contained HTML page that already follows the Velixo design system. Copy it, replace
the `<!-- … -->` markers, keep the variable names. No external CSS, no web fonts, logo by URL.

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title><!-- Report title --></title>
<style>
  :root{
    --brand:#345ad9; --brand-dark:#2e429b; --ink:#243261; --text:#242424; --muted:#6e747d;
    --bg:#fbfcfe; --card:#ffffff; --panel:#f3f6fc; --line:#e5e8ee;
    --good:#1f9d57; --alert:#f74245; --warn:#e08a2b;
  }
  /* Dark surfaces: derived from the palette, same variable names. Brand lifts for contrast. */
  @media (prefers-color-scheme: dark){
    :root:not([data-theme="light"]){
      --brand:#5b7cf0; --brand-dark:#3d58c4; --ink:#243261; --text:#e8ebf2; --muted:#9aa3b5;
      --bg:#121725; --card:#1a2033; --panel:#222a40; --line:#2e3752;
      --good:#3fbf78; --alert:#ff6b6e; --warn:#f0a44a;
    }
  }
  :root[data-theme="dark"]{
    --brand:#5b7cf0; --brand-dark:#3d58c4; --ink:#243261; --text:#e8ebf2; --muted:#9aa3b5;
    --bg:#121725; --card:#1a2033; --panel:#222a40; --line:#2e3752;
    --good:#3fbf78; --alert:#ff6b6e; --warn:#f0a44a;
  }
  *{box-sizing:border-box}
  body{margin:0;background:var(--bg);color:var(--text);
       font:15px/1.5 -apple-system,"Segoe UI",Roboto,Helvetica,Arial,sans-serif}
  header{background:var(--ink);color:#fff;padding:20px 32px;display:flex;align-items:center;gap:24px}
  header img{height:28px}
  header .scope{font-size:13px;opacity:.85}
  main{max-width:1100px;margin:0 auto;padding:32px}
  .cards{display:grid;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));gap:16px}
  .card{background:var(--card);border:1px solid var(--line);border-radius:10px;padding:18px 20px}
  .card .label{color:var(--muted);font-size:13px}
  .card .value{font-size:28px;font-weight:600;margin:4px 0}
  .card .delta{font-size:13px}
  .good{color:var(--good)} .warn{color:var(--warn)} .alert{color:var(--alert)}
  /* variance cells: colour by meaning (favourable / unfavourable), figure stays in --text */
  td.fav{background:color-mix(in srgb,var(--good) 12%,transparent)}
  td.unfav{background:color-mix(in srgb,var(--alert) 12%,transparent)}
  table{width:100%;border-collapse:collapse;background:var(--card);border:1px solid var(--line);
        border-radius:10px;overflow:hidden;margin-top:28px}
  th{background:var(--panel);text-align:left;font-weight:600;padding:10px 14px;font-size:13px}
  td{padding:10px 14px;border-top:1px solid var(--line)}
  td.num,th.num{text-align:right;font-variant-numeric:tabular-nums}
  footer{color:var(--muted);font-size:12px;padding:24px 32px;border-top:1px solid var(--line)}
</style>
</head>
<body>
<header>
  <!-- Logo by URL. If the page must render offline, replace the <img> with: <strong>Velixo</strong> -->
  <img src="https://secure.velixo.com/logos/logo-full-gradient-white.svg" alt="Velixo">
  <div>
    <div style="font-size:18px;font-weight:600"><!-- Report title --></div>
    <div class="scope"><!-- Company · Branch · Ledger · Period · Currency · Run date · Preliminary? --></div>
  </div>
</header>
<main>
  <section class="cards">
    <div class="card">
      <div class="label"><!-- Revenue --></div>
      <div class="value"><!-- $4.2M --></div>
      <div class="delta good"><!-- +3.1% vs budget · On track --></div>
    </div>
    <!-- more cards -->
  </section>
  <table>
    <thead><tr><th>Line</th><th class="num">Actual</th><th class="num">Budget</th><th class="num">Var</th><th class="num">Var %</th><th>Status</th></tr></thead>
    <tbody>
      <tr><td><!-- Revenue --></td><td class="num"></td><td class="num"></td><td class="num"></td><td class="num"></td><td class="good">On track</td></tr>
    </tbody>
  </table>
</main>
<footer>Source: <!-- Acumatica via Velixo, connection scope --> · Generated <!-- timestamp --> · Draft for review</footer>
</body>
</html>
```

Favourable variances positive, unfavourable negative, stated once in the scope line; a `Var` cell
carries `fav` or `unfav` by meaning, so an expense overrun is `unfav` whatever its sign. Units in the
header, not repeated in every cell. The page renders in light and dark without changes; the dark
values are derived from the palette, not a second brand.
