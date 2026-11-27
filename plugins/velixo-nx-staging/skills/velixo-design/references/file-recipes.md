# File recipes: Excel, Word, PowerPoint, PDF

Starting points that already follow the Velixo design system, for files built in Python with
`openpyxl`, `python-docx`, `python-pptx` and `reportlab`. Copy the **common block** first, then the
recipe for the format asked for, and fill it with the run's figures. Keep figures as numbers until the
last step, and keep the colours, the logo fallback and the footer line as they are.

Every recipe takes `flags`, one entry per row: `"fav"`, `"unfav"` or `None`. It colours the row's
variance cell by **meaning**, never by sign: a cost under budget is favourable although its variance
is negative. Decide each flag from the account's nature before calling the recipe.

## Common block

```python
import io
import urllib.request

VX = dict(brand="345AD9", brand_dark="2E429B", ink="243261", text="242424", muted="6E747D",
          bg="FBFCFE", card="FFFFFF", panel="F3F6FC", line="E5E8EE",
          good="1F9D57", alert="F74245", warn="E08A2B",
          good_tint="E4F3EB", alert_tint="FEE8E9")      # the 12% tints of good and alert
SERIES = [VX["brand"], VX["ink"], VX["muted"], VX["brand_dark"]]  # actual, comparison, prior year, fourth
LOGO = {"navy": "https://secure.velixo.com/logos/logo-full-gradient-navy.png",    # light backgrounds
        "white": "https://secure.velixo.com/logos/logo-full-gradient-white.png"}  # brand or ink bands
FOOTER = "Prepared with Velixo · {source} · run {run_date}"
# When the user asked to leave the "Prepared with Velixo" line out, use this instead: the source and run
# date stay, since they say what the figures are.
# FOOTER = "Source: {source} · run {run_date}"


def velixo_logo(variant="navy"):
    """The logo PNG from its URL, as bytes, or None when it cannot be downloaded (no network,
    refused, not a PNG). Every recipe then writes the text "Velixo" in its place."""
    try:
        with urllib.request.urlopen(LOGO[variant], timeout=10) as r:
            data = r.read()
        return data if data[:8] == b"\x89PNG\r\n\x1a\n" else None
    except Exception:
        return None


def fill_for(flag):
    """The fill hex for a variance cell: favourable, unfavourable, or none."""
    return {"fav": VX["good_tint"], "unfav": VX["alert_tint"]}.get(flag)
```

## Excel (`openpyxl`)

Row 1 the logo, row 2 the title, row 3 the scope line, then the table with its header on `--ink`,
totals bold over a `--line` rule, panes frozen under the header, and the footer line under the table.
`variance_col` is the 0-based index of the column the flags colour.

```python
from openpyxl import Workbook
from openpyxl.chart import BarChart, Reference
from openpyxl.drawing.image import Image as XLImage
from openpyxl.styles import Alignment, Border, Font, PatternFill, Side
from openpyxl.utils import get_column_letter


def velixo_sheet(ws, title, scope, headers, rows, total=None, flags=None, variance_col=None,
                 source="", run_date="", number_format="#,##0.00"):
    logo = velixo_logo("navy")
    if logo:
        img = XLImage(io.BytesIO(logo))
        img.width, img.height = img.width * 36 / img.height, 36
        ws.add_image(img, "A1")
        ws.row_dimensions[1].height = 30
    else:
        ws["A1"] = "Velixo"
        ws["A1"].font = Font(bold=True, size=16, color=VX["ink"])
    ws["A2"] = title
    ws["A2"].font = Font(bold=True, size=14, color=VX["ink"])
    ws["A3"] = scope
    ws["A3"].font = Font(size=9, color=VX["muted"])

    head = 5
    for c, h in enumerate(headers, 1):
        cell = ws.cell(head, c, h)
        cell.font = Font(bold=True, color="FFFFFF")
        cell.fill = PatternFill("solid", fgColor=VX["ink"])
        cell.alignment = Alignment(horizontal="left" if c == 1 else "right", vertical="center")
    for r, row in enumerate(rows, head + 1):
        for c, v in enumerate(row, 1):
            cell = ws.cell(r, c, v)
            cell.font = Font(color=VX["text"])
            if isinstance(v, (int, float)):
                cell.number_format = number_format
                cell.alignment = Alignment(horizontal="right")
        hexfill = fill_for(flags[r - head - 1]) if flags and variance_col is not None else None
        if hexfill:
            ws.cell(r, variance_col + 1).fill = PatternFill("solid", fgColor=hexfill)
    last = head + len(rows)
    if total:
        last += 1
        for c, v in enumerate(total, 1):
            cell = ws.cell(last, c, v)
            cell.font = Font(bold=True, color=VX["text"])
            cell.border = Border(top=Side(style="thin", color=VX["line"]))
            if isinstance(v, (int, float)):
                cell.number_format = number_format
                cell.alignment = Alignment(horizontal="right")
    foot = ws.cell(last + 2, 1, FOOTER.format(source=source, run_date=run_date))
    foot.font = Font(size=8, color=VX["muted"])

    for c, h in enumerate(headers, 1):
        width = max(len(str(h)), *(len(f"{row[c - 1]:,.2f}" if isinstance(row[c - 1], (int, float))
                                       else str(row[c - 1])) for row in rows))
        ws.column_dimensions[get_column_letter(c)].width = min(max(width + 2, 10), 50)
    ws.freeze_panes = ws.cell(head + 1, 1)
    return head, last  # the header row and the last table row, for a chart


def velixo_bar_chart(ws, head, last_row, value_cols, anchor, title=""):
    """A clustered bar chart of the table's rows (totals excluded), series in the Velixo order.
    value_cols: 1-based column numbers, actual first, then the comparison."""
    chart = BarChart()
    chart.title = title or None
    chart.style = 2
    for i, col in enumerate(value_cols):
        chart.add_data(Reference(ws, min_col=col, min_row=head, max_row=last_row), titles_from_data=True)
        chart.series[i].graphicalProperties.solidFill = SERIES[i % len(SERIES)]
        chart.series[i].graphicalProperties.line.solidFill = SERIES[i % len(SERIES)]
    chart.set_categories(Reference(ws, min_col=1, min_row=head + 1, max_row=last_row))
    chart.height, chart.width = 8, 16
    ws.add_chart(chart, anchor)


# wb = Workbook(); ws = wb.active; ws.title = "Variance"
# head, last = velixo_sheet(ws, "Revenue vs budget, March 2025", "Company … · Branch … · Ledger ACTUAL · USD",
#                           ["Account", "Actual", "Budget", "Variance"], rows, total=["Total", …],
#                           flags=flags, variance_col=3, source="Acumatica gab-dev", run_date="2025-04-02")
# velixo_bar_chart(ws, head, head + len(rows), [2, 3], f"A{last + 4}")  # under the table, so it prints with it
# wb.save("variance.xlsx")
```

## Word (`python-docx`)

The logo and the scope line in the page header, the footer line in the page footer, headings in
`--ink`, tables with their header on `--panel` and `--line` rules.

```python
from docx import Document
from docx.enum.table import WD_TABLE_ALIGNMENT
from docx.enum.text import WD_ALIGN_PARAGRAPH
from docx.oxml import OxmlElement
from docx.oxml.ns import qn
from docx.shared import Cm, Pt, RGBColor


def _shade(cell, hexfill):
    shd = OxmlElement("w:shd")
    shd.set(qn("w:val"), "clear")
    shd.set(qn("w:color"), "auto")
    shd.set(qn("w:fill"), hexfill)
    cell._tc.get_or_add_tcPr().append(shd)


def _rules(table):
    borders = OxmlElement("w:tblBorders")
    for edge in ("top", "bottom", "insideH"):
        el = OxmlElement(f"w:{edge}")
        el.set(qn("w:val"), "single")
        el.set(qn("w:sz"), "4")
        el.set(qn("w:color"), VX["line"])
        borders.append(el)
    table._tbl.tblPr.append(borders)


def velixo_document(title, scope, source="", run_date=""):
    doc = Document()
    normal = doc.styles["Normal"]
    normal.font.name, normal.font.size = "Calibri", Pt(10)
    normal.font.color.rgb = RGBColor.from_string(VX["text"])
    for level in (1, 2, 3):
        doc.styles[f"Heading {level}"].font.color.rgb = RGBColor.from_string(VX["ink"])

    section = doc.sections[0]
    header = section.header.paragraphs[0]
    logo = velixo_logo("navy")
    if logo:
        header.add_run().add_picture(io.BytesIO(logo), height=Cm(0.9))
    else:
        run = header.add_run("Velixo")
        run.bold, run.font.size = True, Pt(14)
        run.font.color.rgb = RGBColor.from_string(VX["ink"])
    scope_para = section.header.add_paragraph()
    scope_para.paragraph_format.space_before = Pt(4)
    scope_line = scope_para.add_run(scope)
    scope_line.font.size = Pt(8)
    scope_line.font.color.rgb = RGBColor.from_string(VX["muted"])
    footer = section.footer.paragraphs[0].add_run(FOOTER.format(source=source, run_date=run_date))
    footer.font.size = Pt(8)
    footer.font.color.rgb = RGBColor.from_string(VX["muted"])

    doc.add_heading(title, level=1)
    return doc


def velixo_table(doc, headers, rows, total=None, flags=None, variance_col=None):
    body = rows + ([total] if total else [])
    table = doc.add_table(rows=1 + len(body), cols=len(headers))
    table.alignment = WD_TABLE_ALIGNMENT.LEFT
    _rules(table)
    for c, h in enumerate(headers):
        cell = table.rows[0].cells[c]
        _shade(cell, VX["panel"])
        cell.paragraphs[0].paragraph_format.space_after = Pt(0)
        run = cell.paragraphs[0].add_run(str(h))
        run.bold = True
        run.font.color.rgb = RGBColor.from_string(VX["ink"])
        if c:
            cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.RIGHT
    for r, row in enumerate(body, 1):
        for c, v in enumerate(row):
            cell = table.rows[r].cells[c]
            p = cell.paragraphs[0]
            p.paragraph_format.space_after = Pt(0)
            run = p.add_run(f"{v:,.2f}" if isinstance(v, (int, float)) else str(v))
            run.bold = bool(total) and r == len(body)
            if isinstance(v, (int, float)):
                p.alignment = WD_ALIGN_PARAGRAPH.RIGHT
        hexfill = fill_for(flags[r - 1]) if flags and r <= len(rows) and variance_col is not None else None
        if hexfill:
            _shade(table.rows[r].cells[variance_col], hexfill)
    return table


# doc = velixo_document("Revenue vs budget, March 2025", "Company … · Branch … · USD",
#                       source="Acumatica gab-dev", run_date="2025-04-02")
# doc.add_paragraph("Revenue finished …")
# velixo_table(doc, ["Account", "Actual", "Budget", "Variance"], rows, total=[…], flags=flags, variance_col=3)
# doc.save("variance.docx")
```

## PowerPoint (`python-pptx`)

A 16:9 deck: the title slide on an `--ink` band with the white logo, content slides on white with the
heading in `--ink`, the footer line on every slide. One message per slide: the heading says it.

```python
from pptx import Presentation
from pptx.chart.data import CategoryChartData
from pptx.dml.color import RGBColor as PptRGB
from pptx.enum.chart import XL_CHART_TYPE, XL_LEGEND_POSITION
from pptx.util import Inches, Pt


def _text(slide, text, left, top, width, height, size, hexcolor, bold=False):
    box = slide.shapes.add_textbox(left, top, width, height)
    frame = box.text_frame
    frame.word_wrap = True
    run = frame.paragraphs[0].add_run()
    run.text = text
    run.font.size, run.font.bold = Pt(size), bold
    run.font.color.rgb = PptRGB.from_string(hexcolor)
    return box


def _footer(prs, slide, source, run_date, hexcolor):
    _text(slide, FOOTER.format(source=source, run_date=run_date), Inches(0.6),
          prs.slide_height - Inches(0.55), Inches(10), Inches(0.35), 9, hexcolor)


def velixo_deck(title, scope, source="", run_date=""):
    prs = Presentation()
    prs.slide_width, prs.slide_height = Inches(13.333), Inches(7.5)
    prs.velixo = dict(source=source, run_date=run_date)
    slide = prs.slides.add_slide(prs.slide_layouts[6])  # blank
    slide.background.fill.solid()
    slide.background.fill.fore_color.rgb = PptRGB.from_string(VX["ink"])
    logo = velixo_logo("white")
    if logo:
        slide.shapes.add_picture(io.BytesIO(logo), Inches(0.6), Inches(0.6), height=Inches(0.6))
    else:
        _text(slide, "Velixo", Inches(0.6), Inches(0.5), Inches(4), Inches(0.8), 28, "FFFFFF", bold=True)
    _text(slide, title, Inches(0.6), Inches(2.6), Inches(12), Inches(1.6), 40, "FFFFFF", bold=True)
    _text(slide, scope, Inches(0.6), Inches(4.3), Inches(12), Inches(0.6), 16, "FFFFFF")
    _footer(prs, slide, source, run_date, "FFFFFF")
    return prs


def velixo_slide(prs, heading):
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    _text(slide, heading, Inches(0.6), Inches(0.4), Inches(12.1), Inches(0.9), 28, VX["ink"], bold=True)
    _footer(prs, slide, prs.velixo["source"], prs.velixo["run_date"], VX["muted"])
    return slide


def velixo_slide_table(slide, headers, rows, flags=None, variance_col=None):
    shape = slide.shapes.add_table(len(rows) + 1, len(headers), Inches(0.6), Inches(1.5),
                                   Inches(12.1), Inches(0.4) * (len(rows) + 1))
    table = shape.table
    table.first_row = True
    for r, row in enumerate([headers] + rows):
        for c, v in enumerate(row):
            cell = table.cell(r, c)
            cell.text = f"{v:,.2f}" if isinstance(v, (int, float)) else str(v)
            run = cell.text_frame.paragraphs[0].runs[0]
            run.font.size = Pt(12)
            run.font.bold = r == 0
            run.font.color.rgb = PptRGB.from_string("FFFFFF" if r == 0 else VX["text"])
            if c:
                cell.text_frame.paragraphs[0].alignment = 3  # right
            cell.fill.solid()
            hexfill = VX["ink"] if r == 0 else (fill_for(flags[r - 1]) if flags and c == variance_col else None)
            cell.fill.fore_color.rgb = PptRGB.from_string(hexfill or VX["card"])
    return table


def velixo_slide_chart(slide, categories, series):
    """series: {name: values}, actual first, then the comparison; coloured in the Velixo order."""
    data = CategoryChartData()
    data.categories = categories
    for name, values in series.items():
        data.add_series(name, values)
    chart = slide.shapes.add_chart(XL_CHART_TYPE.COLUMN_CLUSTERED, Inches(0.6), Inches(1.5),
                                   Inches(12.1), Inches(5.2), data).chart
    chart.has_legend, chart.legend.position, chart.legend.include_in_layout = True, XL_LEGEND_POSITION.BOTTOM, False
    axis = chart.value_axis
    axis.tick_labels.number_format, axis.tick_labels.number_format_is_linked = "#,##0", False
    axis.major_gridlines.format.line.color.rgb = PptRGB.from_string(VX["line"])
    for i, s in enumerate(chart.series):
        s.format.fill.solid()
        s.format.fill.fore_color.rgb = PptRGB.from_string(SERIES[i % len(SERIES)])
    return chart


# prs = velixo_deck("March 2025 variance review", "Company … · Branch … · USD",
#                   source="Acumatica gab-dev", run_date="2025-04-02")
# velixo_slide_chart(velixo_slide(prs, "Revenue missed budget by 4%, on two customers"), cats, {"Actual": a, "Budget": b})
# velixo_slide_table(velixo_slide(prs, "The three accounts behind it"), headers, rows, flags=flags, variance_col=3)
# prs.save("variance.pptx")
```

## PDF (`reportlab`)

The logo and the scope line in the page header, the footer line and page number in the page footer,
the title in `--ink`, the table header on `--panel` with `--line` rules. Its built-in fonts cover
WinAnsi only: write a hyphen for the minus sign (Python's own formatting does), and words for arrows
and deltas.

```python
from reportlab.lib import colors
from reportlab.lib.pagesizes import A4, landscape
from reportlab.lib.styles import ParagraphStyle, getSampleStyleSheet
from reportlab.lib.units import cm
from reportlab.lib.utils import ImageReader
from reportlab.platypus import Paragraph, SimpleDocTemplate, Spacer, Table, TableStyle


def hexc(key):
    return colors.HexColor("#" + VX[key])


def velixo_pdf(path, title, scope, headers, rows, total=None, flags=None, variance_col=None,
               paragraphs=(), source="", run_date=""):
    logo = velixo_logo("navy")
    image = ImageReader(io.BytesIO(logo)) if logo else None
    page_w, page_h = landscape(A4)

    def frame(canvas, doc):
        canvas.saveState()
        if image:
            w, h = image.getSize()
            canvas.drawImage(image, doc.leftMargin, page_h - 1.7 * cm, width=0.9 * cm * w / h,
                             height=0.9 * cm, mask="auto")
        else:
            canvas.setFont("Helvetica-Bold", 14)
            canvas.setFillColor(hexc("ink"))
            canvas.drawString(doc.leftMargin, page_h - 1.5 * cm, "Velixo")
        canvas.setFont("Helvetica", 8)
        canvas.setFillColor(hexc("muted"))
        canvas.drawRightString(page_w - doc.rightMargin, page_h - 1.4 * cm, scope)
        canvas.drawString(doc.leftMargin, 1 * cm, FOOTER.format(source=source, run_date=run_date))
        canvas.drawRightString(page_w - doc.rightMargin, 1 * cm, f"Page {doc.page}")
        canvas.restoreState()

    styles = getSampleStyleSheet()
    h1 = ParagraphStyle("vxh1", parent=styles["Heading1"], textColor=hexc("ink"), fontSize=18)
    body = ParagraphStyle("vxbody", parent=styles["BodyText"], textColor=hexc("text"), fontSize=10)
    story = [Paragraph(title, h1), Spacer(1, 0.2 * cm)]
    story += [Paragraph(p.replace("&", "&amp;"), body) for p in paragraphs]
    story.append(Spacer(1, 0.4 * cm))

    fmt = lambda v: f"{v:,.2f}" if isinstance(v, (int, float)) else str(v)
    data = [list(map(str, headers))] + [list(map(fmt, r)) for r in rows] + ([list(map(fmt, total))] if total else [])
    style = [
        ("BACKGROUND", (0, 0), (-1, 0), hexc("panel")),
        ("TEXTCOLOR", (0, 0), (-1, 0), hexc("ink")),
        ("FONTNAME", (0, 0), (-1, 0), "Helvetica-Bold"),
        ("TEXTCOLOR", (0, 1), (-1, -1), hexc("text")),
        ("FONTSIZE", (0, 0), (-1, -1), 9),
        ("ALIGN", (1, 0), (-1, -1), "RIGHT"),
        ("LINEBELOW", (0, 0), (-1, -1), 0.5, hexc("line")),
        ("TOPPADDING", (0, 0), (-1, -1), 4),
        ("BOTTOMPADDING", (0, 0), (-1, -1), 4),
    ]
    if total:
        style.append(("FONTNAME", (0, -1), (-1, -1), "Helvetica-Bold"))
    for r, flag in enumerate(flags or [], 1):
        if fill_for(flag) and variance_col is not None:
            style.append(("BACKGROUND", (variance_col, r), (variance_col, r), colors.HexColor("#" + fill_for(flag))))
    table = Table(data, repeatRows=1, hAlign="LEFT")
    table.setStyle(TableStyle(style))
    story.append(table)

    doc = SimpleDocTemplate(path, pagesize=(page_w, page_h), topMargin=2.4 * cm, bottomMargin=1.8 * cm,
                            leftMargin=1.6 * cm, rightMargin=1.6 * cm, title=title, author="Velixo")
    doc.build(story, onFirstPage=frame, onLaterPages=frame)


# velixo_pdf("variance.pdf", "Revenue vs budget, March 2025", "Company … · Branch … · USD",
#            ["Account", "Actual", "Budget", "Variance"], rows, total=[…], flags=flags, variance_col=3,
#            paragraphs=["Revenue finished …"], source="Acumatica gab-dev", run_date="2025-04-02")
```
