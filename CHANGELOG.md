# Changelog — `front-page-layout` vs `master`

Goal of this branch: make the docx2tex pipeline reproduce the Word/OnlyOffice rendering of
a styled Word report (reference document `HSS_REP.docx`, a Health Support Services template
with a cover section, classification marking, logo lockup and artwork footer). Measured
against the OnlyOffice PDF export, page 1 of the generated LaTeX PDF went from having seven
substantial differences to matching the reference: same page count (6), same content per
page, same fonts (Arial + Calibri), same colours, same marking position, correct logo
proportions and true A4 page geometry.

A second pass (after tag `Changelog-1`) extended the match from the cover to the whole
document: Word Heading styles become styled, mid-page-flowing LaTeX sectioning commands
with a filled table of contents; every Word section renders its own header/footer with
roman/arabic page-number restarts; heading and footer spacing follows Word's max-collapse
rule; and per-section page geometry (including a closing page with a tall custom top
margin) is applied at the section breaks. All six pages now match the reference layout.

Verification artifacts and per-issue root-cause notes live in
`.zcode/plans/page1-diff-docx2tex-vs-onlyoffice.md` and
`.zcode/plans/full-diff.md`.

## Base repo (docx2tex) — 21 commits ahead of master

### Word page layout → LaTeX geometry

- **Map Word page setup to LaTeX geometry** (`80aa98a`, `1782c25`): `pgSz`/`pgMar` from the
  docx become `geometry` options; paragraph spacing (`w:spacing`) becomes
  `\vspace*`/`\vspace` and font sizes (half-points) become `\fontsize` selections at run and
  paragraph level. Word headers/footers are included in the hub by docx2hub
  (`include-header-and-footer`) and rendered from the generated preamble.
- **Standard paper sizes and correct units** (`bc275ae`): docx lengths (twips/20) are PDF
  points, but the emitted geometry used TeX `pt`, leaving every page ~0.37 % undersized
  (593.08 × 838.76 pt instead of A4). The geometry now emits the standard size name when
  the dimensions match one (portrait A4/A5/A3/Letter/Legal/Executive, ±0.5 bp) and numeric
  `paperwidth`/`paperheight` in `bp` otherwise; margins and derived lengths use `bp` too.
  Output is true A4, matching the reference exactly.
- **Headers/footers at absolute page positions** (`4293ca0`): Word header/footer content
  (classification marking, logo, vision text, artwork band) is drawn with `eso-pic`
  `\AddToShipoutPicture` at the Word anchor positions rather than flowing in the text.

### Page structure

- **Word section breaks become page breaks** (`5a73a8c`): the first docx section is a cover
  page ending in a `nextPage` section break. docx2hub already marks the (empty) `sectPr`
  paragraph with `css:page-break-after="always"` and conf already maps it to `\clearpage`,
  but the empty-paragraph cleanup in `docx2tex-preprocess.xsl` deleted the marker before
  xml2tex saw it. Paras carrying `@css:page-break-after` are now exempt from that removal.
  Result: one `\clearpage` per section break; the document is 6 pages (was 4) with a
  cover-only page 1, matching the reference.
- **Footer band reservation on cover pages** (`21a1139`): the vision text + artwork are
  absolute-positioned shipout pictures that LaTeX geometry knows nothing about, so a longer
  cover would flow text into the band. `\textheight` is now shrunk by (footer band height −
  bottom margin) for the pages before the first section break and restored right after the
  cover's `\clearpage`. `\newgeometry` was tried and rejected: it resets topskip/headheight
  to package defaults and shifted the cover text block ~18 pt; plain `\addtolength` changes
  nothing else. Verified with a 40-paragraph stress-test cover — content breaks above the
  band exactly as Word pushes text above a tall footer.

### Fonts

- **Real docx fonts under LuaLaTeX** (`4293ca0`): the generated preamble selects the
  document's real fonts via fontspec (Arial) instead of substituting.
- **Arial actually wins** (`f2cf01a`): `txfonts` (loaded for math) reset the text families
  to `txr`/`txss`, which have no TU-encoding definitions under LuaLaTeX, so every font
  shape silently fell back to Latin Modern — the whole document rendered serif. `txfonts`
  now loads before fontspec/tgheros, letting `\setmainfont{Arial}` have the final word on
  text while its math setup is unaffected. The PDF embeds
  Arial/Arial-Bold/Arial-Italic, matching the reference's font set.

### Colour

- **Classification marking: red, page-centred, Calibri** (`c2d898a`): the OFFICIAL marking
  is placed page-centred when its Word anchor declares `mso-position-horizontal:center`
  relative to page (`\put(\LenToUnit{0.5\paperwidth},…){\makebox[0pt][c]{…}}`), and
  header/footer phrases keep their Word font via an engine-guarded `\fontspec{Calibri}`
  (Calibri embeds alongside Arial, as in the reference).
- **Run and paragraph colours** (`c2d898a`): with docx2hub now mapping lowercase hex
  colours, cover text gets its Word colours — Title navy via the existing phrase
  `\textcolor` mapping, and a new paragraph-level `{\color{…}}` declaration handles
  style-derived colours on phrase-less paras (Subtitle grey). Bonus fidelity: table borders
  render their Word grey `a6a6a6` instead of black.

### Images

- **Word image crops (`srcRect`)** (`5ad5605`): cropped pictures render correctly. Crop
  fractions from the hub (`css:crop-*`, see docx2hub below) become graphicx
  `trim={\dimexpr…\relax} …, clip`. The fractions are relative to the natural image size,
  which the pipeline cannot know (the logo JPEG carries ~299 dpi metadata, so natural pt ≠
  pixel count), so the generated TeX measures it at compile time via
  `\sbox\@tempboxa{\includegraphics{…}}`. The page-1 logo was squeezed to 72 % of its
  correct width before; its aspect now matches the reference (3.93 vs 3.97).

### Regression fixes (cover and all pages)

- **Dedicated savebox for cropped images** (`2ae0613`): the crop-measuring code stored the
  image in `\@tempboxa`, and the shipout picture emitted `\@tempboxa` as literal text on
  every page ("tempboxa" visible on all pages). Crops now use a dedicated
  `\newsavebox{\docxcroppedimagebox}`, allocated in the preamble when any cropped image
  exists.
- **Zero `\fboxsep` around highlight colorboxes** (`292ea9b`): `\colorbox`'s default 3pt
  padding inserted gaps inside highlighted runs ("Month 20 YY", "DD.MM.20YY ABC"). Word
  highlight has no padding, so highlight boxes are now emitted inside a local
  `{\fboxsep0pt …}` group.
- **`amsmath` before `txfonts`** (`f22e7bc`): the txfonts load order broke latexmk — its
  `\iint` &co. clashed with amsmath's. txfonts now loads after amsmath (and still before
  fontspec/tgheros, preserving the "docx fonts win" rule from `f2cf01a`).

### Word Heading styles → LaTeX sectioning, and a filled TOC

- **Heading paragraphs map to sectioning commands** (`41cc90e`): numbered Word headings
  fell through to single-item `enumerate` lists, so `\tableofcontents` stayed empty.
  With the docx2hub styleId fix (see below) the heading roles match the conf templates,
  activating the pipeline's existing numbered-heading machinery (headline marking, list
  exemption, Word numbers replaced by LaTeX auto-numbering, anchors kept as
  `\label{mark-…}`). docx2tex-preprocess additionally retitles a heading by its effective
  outline level when Word numbers it at another level (Heading-1-styled "2.1 Sub heading"
  becomes `\section`) and strips literal numbers from title text that equal LaTeX's
  auto number ("2.2 Sub heading" stored its number as text). Word Heading 1 → `\chapter`,
  Heading 2 → `\section`.
- **Heading styles via titlesec; chapters flow like Word** (`27086c5`): one
  `\titleformat`/`\titlespacing` per level is generated from the docx Heading styles'
  css rules (font size, weight, colour — e.g. H1 14pt bold navy with a trailing dot after
  the number). `\titleclass{\chapter}{straight}` removes the chapter page break, so
  Heading 1 renders inline and flows mid-page as in the reference; the TOC gets an
  explicit `\clearpage`, restoring the reference's 6-page layout, and the "Contents"
  heading picks up the H1 style.

### Per-section headers, footers and page numbers

- **fancyhdr page styles per Word section** (`5334338`): each section's default
  header/footer part becomes a `\fancypagestyle{docx-section-N}` (header text by
  alignment, footer table cells as L/R entries, rules from the Word border widths),
  switched at the section breaks together with `\pagenumbering` (roman/arabic restarts
  from the sections' `pgNumType`). The classification marking stays on all pages via a
  `\docxmarking` macro; the cover artwork is cleared from the eso-pic shipout stack
  after page 1; the final section's full-bleed footer band is added at its break.
  Word's PAGE/NUMPAGES field results regenerate as live numbers (`\thepage`, and a
  physical page count via zref-abspage) so "Page i of 6" matches the real LaTeX
  pagination. `\KOMAoption{open}{any}` stops scrbook inserting blank verso pages between
  chapters. Header fonts resolve once in the preamble into `\docxhffont` macros — a
  missing font (Arial Narrow) must not trigger a luaotfload reload from inside the
  output routine.
- **Footer refinements** (`316336a`): footer text takes its paragraph style's 9pt bold
  (the "Page n of m" paragraph keeps its explicit normal weight, as in the docx) —
  previously the un-highlighted runs rendered at 12pt regular next to 9pt highlighted
  ones; `\thepage{}` keeps the space in "Page i of 6" (the control word gobbled it);
  the cover's `\textheight` reservation is restored *before* the break's `\clearpage`
  (the page-1 shipout re-syncs the page-builder column height — restoring after it left
  page 2's footer rule 67.8pt too high).
- **Heading spacing collapses like Word** (`a285001`): Word keeps inter-paragraph
  spacing at max(space-after, space-before); the output summed the explicit `\vspace`
  values on top of `\parskip` (8.5 + 12 + 12 = 32.5pt where Word uses 12pt). Headline
  paragraphs now emit only the excess over `\parskip`, and a heading's space-before is
  reduced by the effective space-after it follows. Measured H1→H2 baseline gap 28.8pt
  vs 29.2pt in the reference.

### Per-section page geometry

- **Word `w:pgMar` at section breaks** (`eaf81d1`): sections 2–3 use a 70.9pt top margin
  (the pipeline had section 1's 99.25pt everywhere) and the closing section a tall
  custom 537.5pt top margin; `\newgeometry` at each section break applies the section's
  own margins (landing-style pages with a ≥300pt top margin additionally shift by the
  section's header distance, matching the reference: last-page body starts 582.6pt from
  the top vs 582.5pt). The header text is re-anchored to the reference position — Word's
  header paragraph is tall (it anchors the marking text box), so its bottom-border rule
  sits ~19pt below the text; modeled with fancyhdr's `\headruleskip` (rule 62.4pt,
  text baseline ~43.5pt).

### Submodule wiring

- **baradhili forks** (`bb1dbca`, `237ac3d`, `7f65bb5`): `docx2hub`, `xml2tex` and
  `mml2tex` submodules point at the github.com/baradhili forks, track their
  `front-page-layout` branches (`branch = front-page-layout` in `.gitmodules`), and are
  pinned to the fork commits listed below.

## docx2hub fork — 7 commits ahead of master

- **Accept capitalized built-in heading names when rewriting styleIds** (`86db888`):
  Word built-in styles are stored as lowercase `heading 1`, but some authoring tools
  write `Heading 1` (capitalized) with numeric styleIds (e.g. 1278). The case-sensitive
  regex never matched, so paragraphs kept `_1278`-style roles that no conf template
  recognizes and numbered headings fell through to enumerate lists. The name match and
  the `heading ` strip in the `Heading{N}` rewrite are now case-insensitive.
- **Surface section page-number format/restart and section boundaries** (`9d58a9d`): the
  add-props pass captures each `sectPr`'s `w:pgNumType` as `css:page-number-*` on the
  sectPr marker paras; wml-to-dbk flags the paragraph that ends a section
  (`docx2hub:section-end`) and copies the page-number attributes of the section that
  begins after the break onto it, so downstream converters can switch numbering at Word
  section boundaries.
- **Surface per-section page margins and header distance** (`21d7839`): each `sectPr`'s
  `w:pgMar` (top/bottom/left/right, twips → pt) and `w:header` distance are captured as
  `css:page-margin-*`/`css:page-header-distance` on the marker paras and copied onto the
  section-break paragraphs (final section via the body-level sectPr), enabling
  per-section `\newgeometry` downstream.

- **Surface page geometry, header/footer part rels and section page breaks** (`305578a`):
  the first section's `pgSz`/`pgMar` are emitted as `css:*` attributes on the hub root;
  image relationships in header/footer parts resolve against the part's own rels
  (`@xml:base`-based) instead of the main document rels; the docx2hub:header/footer divs
  keep their `@xml:base` and are ordered by section-reference position (first section
  first); the paragraph before a removed `sectPr` pseudo-paragraph is marked
  `css:page-break-after` so Word section breaks survive into the hub.
- **Keep embedded pictures out of the SVG renderer** (`b2cd89b`): AlternateContent drawings
  that contain embedded pictures are no longer converted to SVG, which used to swallow the
  logo pictures.
- **Fix lowercase hex colours** (`ef5606a`): `docx2hub:color()` matched hash-less hex only
  in uppercase (`[0-9A-F]{6}`), but Word writes colours lowercase — `ff0000`, `1b2546`,
  `3d3935`, `a6a6a6` matched no branch and were silently dropped (digit-only colours like
  `152147` worked, masking the bug). Now accepts both cases and normalises to uppercase.
- **Surface Word image crops** (`29662fa`): `a:srcRect` values ≤ 100000 are
  ST_Percentage in 1000ths of a percent (ISO 29500-1), not EMU — the old `css:clip`
  formula divided by 12700 and produced nonsense, and nothing consumed it. Crops are now
  emitted as `css:crop-top/right/bottom/left` percentages on `imagedata`, from both the
  DrawingML `a:srcRect` and the VML `crop*` (65536th-fraction) paths.

## xml2tex fork — 1 commit ahead of master

- **Row shading and border rule colours in `calstable2tabular`** (`13336a5`): CALS table
  row shading (`\rowcolor`) and per-rule border colours (`\arrayrulecolor`) are carried
  into the generated `tabular`, so the navy header row and grey grid of Word tables render
  as in the reference.

## mml2tex fork

No changes on `front-page-layout` (pinned at master).
