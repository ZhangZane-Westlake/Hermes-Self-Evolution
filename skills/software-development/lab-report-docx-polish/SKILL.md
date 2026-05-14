---
name: lab-report-docx-polish
description: "Use when converting a rough physics/engineering lab report DOCX into publication-style Word formatting with native equations, journal captions, standardized tables, and polished layout."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [lab-report, docx, word, equations, formatting, journal-style]
    related_skills: [hermes-agent-skill-authoring]
---

# Lab Report DOCX Polishing

## Overview

Use this skill when the user already has a `.docx` lab report and wants it upgraded into a clean, submission-ready or publication-style manuscript while keeping everything editable in Microsoft Word.

The core requirement is: do not flatten content into images or PDFs. Keep formulas as native Word equations, keep captions editable, and preserve tables/figures as real Word objects.

This workflow was validated on a physics lab report where the user wanted:
- Introduction formulas converted to Word equation format
- journal-style layout and spacing
- equation numbering
- figure/table caption normalization
- variable symbols italicized in prose
- consistent table headers, units, and significant figures
- a cleaner table of contents

## When to Use

Use when:
- the user gives you an existing `.docx` lab report to polish
- formulas are typed as plain text and need to become Word equations
- the report needs to look like a manuscript rather than class notes
- the user wants all edits to remain directly editable in Word
- the document already contains figures/tables and needs consistent captions and numbering

Do not use when:
- the source file is PDF-only and needs full OCR/reconstruction first
- the user wants LaTeX output instead of Word output
- the report content itself is still missing and the task is primarily writing, not formatting

## Recommended Tooling

Preferred environment:
- `python3`
- `python-docx`
- direct XML/OMML insertion for native Word equations

Important constraint:
- `python-docx` cannot create all Word equation structures via high-level API alone. For native editable equations, insert OMML (`m:oMathPara`) into `word/document.xml` through paragraph XML operations.

## Workflow

### 1. Discover the target document

Search the working directory for `.docx` files first.

Example:
```bash
find . -maxdepth 2 -name '*.docx'
```

If there are multiple candidate files, inspect names and prefer the latest report-looking document.

### 2. Inspect the current DOCX structure

Use `python-docx` to enumerate paragraphs, styles, tables, and approximate locations of formulas/captions.

Checklist:
- title block paragraphs
- heading structure
- table of contents block
- formula paragraphs currently typed as plain text
- figure captions and table captions
- table headers and numeric formatting

Useful inspection pattern:
```python
from docx import Document

doc = Document(path)
for i, para in enumerate(doc.paragraphs):
    txt = para.text.strip()
    if txt:
        print(i, para.style.name, repr(txt))
```

Also inspect tables:
```python
for ti, table in enumerate(doc.tables):
    for r, row in enumerate(table.rows[:3]):
        print(ti, r, [c.text for c in row.cells])
```

### 3. Standardize page layout and text styles

A safe journal-like baseline for Word lab reports:
- font: Times New Roman
- body size: 12 pt
- Heading 1: 14 pt bold
- Heading 2: 12 pt bold
- margins: 1.0 in on all sides
- body line spacing: 1.5
- body alignment: justified
- first-line indent: 0.3 in
- displayed equations: centered, no first-line indent
- captions: smaller than body text

Typical paragraph formatting:
- body: 0 pt before, 6 pt after
- heading 1: ~12 pt before, 6 pt after
- heading 2: ~10 pt before, 4 pt after
- equations: 3 pt before/after, single-spaced

### 4. Convert plain-text formulas into native Word equations

This is the most important step.

For each formula paragraph typed like:
- `E = -N dΦ/dt`
- `E = -N A dB/dt`
- `ΔU = mgd[cos(...)-cos(...)]`

replace the paragraph contents with OMML.

Pattern:
1. clear the paragraph while preserving `w:pPr`
2. set centered equation formatting
3. append `m:oMathPara` XML
4. add a right-aligned equation number run such as `(1)`

Minimal helper pattern:
```python
from docx.oxml import parse_xml
from docx.oxml.ns import qn

def clear_paragraph_keep_ppr(paragraph):
    p = paragraph._element
    for child in list(p):
        if child.tag != qn('w:pPr'):
            p.remove(child)
```

Then append OMML like:
```xml
<m:oMathPara xmlns:m="http://schemas.openxmlformats.org/officeDocument/2006/math">
  <m:oMath>
    <m:r><m:t>Φ = BA</m:t></m:r>
  </m:oMath>
</m:oMathPara>
```

Important:
- Word equations may appear as empty paragraph text when re-read through `python-docx`; this is expected.
- Verify by checking `word/document.xml` for `<m:oMathPara` rather than relying on `.paragraphs[i].text`.

Verification example:
```python
import zipfile
with zipfile.ZipFile(path) as zf:
    xml = zf.read('word/document.xml').decode('utf-8', errors='ignore')
    print(xml.count('<m:oMathPara'))
```

### 5. Number equations consistently

Use manuscript-style right-side numbering:
- `(1)`, `(2)`, `(3)` ...
- numbering should be continuous through the full document
- keep displayed equations centered and the number visually right-aligned

Recommended numbering categories:
- theory equations in Introduction
- circuit correction equations in Procedure/Methods
- parameter-definition equations if displayed separately
- worked numeric evaluation equations in Analysis/Discussion

Do not skip numbers or mix styles like `Eq. 1`, `(Eq. 1)`, and `(1)` inside the displayed equation itself.

### 6. Normalize figure and table captions

Preferred journal-style convention:
- figures: `Fig. 1. ...`
- tables: `Table 1. ...`

Styling:
- figure captions: centered, 10 pt, italic
- table titles: left-aligned, 10 pt, bold
- image paragraphs: centered with compact spacing
- table contents: centered, smaller font (e.g. 9 pt)

Common cleanup:
- convert `Figure 1:` → `Fig. 1.`
- convert `Table 1:` → `Table 1.`
- make punctuation consistent across all captions

### 7. Polish table headers, units, and significant figures

Typical improvements:
- replace vague headers like `Diff.` with `Difference (%)`
- include units in header, not in every body cell when appropriate
- standardize naming like `Max. Voltage (V)` and `Mean EMF (V)`
- make row labels consistent, e.g. `2.2 cm Strong, 25°`
- align decimal precision within a column where sensible

Good practice:
- preserve original measured values unless the user asks for scientific reinterpretation
- only normalize formatting/precision presentation, not the underlying data meaning
- if percent is in the header, body cells can be numeric without `%`

### 8. Italicize mathematical variables in prose

For manuscript polish, variable symbols in running text should be italic, while units and ordinary words stay roman.

Examples to italicize in prose:
- `N`, `B`, `R`, `r`, `m`, `d`, `r_av`, `ΔU`, `θ_f,loss`

Examples not to italicize:
- units like `m`, `kg`, `V`, `T`, `rad` when they are units
- acronyms or prose unless the user explicitly wants them styled as math

Pitfall:
- naïve global replacement will incorrectly italicize ordinary letters inside units or words.
- reconstruct only targeted paragraphs/runs, or use carefully constrained token matching.

### 9. Clean the table of contents block

If the report has a manually typed TOC, improve it without rebuilding the whole document.

Typical refinements:
- rename `Contents` → `Table of Contents`
- center the TOC title
- compress spacing between entries
- add right-aligned tab stops with dot leaders
- normalize entry text spacing
- indent subsections slightly more than top-level sections

If the document contains a true Word TOC field, avoid breaking it. If it is plain text, manual refinement is fine.

### 10. Save in place and verify aggressively

After every major formatting pass:
- save the `.docx`
- reopen it with `python-docx`
- verify paragraph/table counts still make sense
- verify OMML equation count in XML
- inspect key paragraphs around equations/captions/tables

Minimum verification checklist:
- document opens successfully
- expected equation count is present
- equation numbering is continuous
- captions read correctly
- table headers/values remain intact
- no accidental duplication of paragraphs after insertions

## Common Pitfalls

1. Re-reading equations through `paragraph.text`

Native Word equations often show up as empty text in `python-docx`. This does NOT mean the equation is missing. Verify in document XML.

2. Accidentally duplicating content while inserting new paragraphs

When splitting a sentence around a new equation, it is easy to leave the old paragraph in place and also insert the rewritten paragraph. Always re-inspect surrounding indices after insertion.

3. Over-italicizing prose

Global search/replace for `m`, `B`, or `r` will corrupt units and normal words. Apply italics only to intended variable tokens in controlled paragraphs.

4. Breaking the TOC or heading flow

Manual TOC cleanup is fine for a plain-text contents page, but do not destroy a field-based TOC unless the user explicitly wants a manual one.

5. Changing scientific meaning while “fixing” significant figures

Formatting cleanup should not invent precision or alter conclusions. Standardize presentation carefully and only within the interpretation already present in the report.

6. Assuming captions are already consistent

Lab reports often mix `Figure`, `Fig`, `Table:`, `Table.` and inconsistent punctuation. Normalize the full set in one pass.

## Verification Checklist

- [ ] Target `.docx` identified correctly
- [ ] Body font, size, spacing, margins standardized
- [ ] All displayed formulas converted to native Word equations
- [ ] Equation numbering is continuous and right-aligned
- [ ] Figure captions use `Fig. n.` style
- [ ] Table titles use `Table n.` style
- [ ] Table headers include clear units and manuscript-friendly names
- [ ] Numeric precision is consistent within columns where appropriate
- [ ] Variable symbols in prose are italicized without damaging units
- [ ] TOC title/indentation/tab leaders are cleaned up
- [ ] Reopened DOCX successfully after edits
- [ ] Verified equation count via `word/document.xml`

## One-Shot Execution Pattern

1. inspect paragraphs/tables
2. identify plain-text formulas and captions
3. apply global page/layout styling
4. replace formulas with OMML equations
5. add/normalize equation numbering
6. restyle captions and tables
7. italicize variables in prose carefully
8. clean TOC block
9. save and verify XML + visible structure

## Output Expectation

The final deliverable should be the same `.docx` file, polished in place, fully editable in Word, and visually close to a journal-style manuscript rather than a raw course draft.
