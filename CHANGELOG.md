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

## Base repo (docx2tex) — 22 commits ahead of master

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

### Cover paragraph spacing and vision weight

- **Word style space-after, line-box compensation, bold "Our vision"** (`0b0e113`): the
  cover gaps were Title→Subtitle 66/74pt, Subtitle→Authoring 23/110pt (the "Sub
  headlines" style's 84pt space-after was never emitted — no margin-bottom-from-style
  template existed), Authoring→Month 29/20pt. Word stacks line boxes while LaTeX adds
  the previous line's depth plus the next line's ascent, so raw space-after values
  under/overshoot after large fonts. Cover paragraphs (before the first Word section
  break) now emit glue = space-after + 0.775 × (own − next line-height excess) with
  `\parskip` netted out (a negative `space` when the Word space is smaller), a new
  margin-bottom-from-style template applies style-defined spacing, and a page-top
  space-before sheds topskip/ascent excess. All four cover anchors match the reference
  within 2pt (Title baseline 286.1 vs 285.7pt). Header/footer paragraph content also
  takes its style's font size and weight: "Our vision:" renders bold from the Word
  Footer style (the run-level normal override on the rest is preserved).

### Submodule wiring

- **baradhili forks** (`bb1dbca`, `237ac3d`, `7f65bb5`): `docx2hub`, `xml2tex` and
  `mml2tex` submodules point at the github.com/baradhili forks, track their
  `front-page-layout` branches (`branch = front-page-layout` in `.gitmodules`), and are
  pinned to the fork commits listed below.

## docx2hub fork — 7 commits ahead of master

The fork surfaces the Word features this pipeline consumes: page geometry and
header/footer part rels, embedded pictures kept out of the SVG renderer, lowercase hex
colour mapping, Word image crops (`srcRect`), case-insensitive built-in heading names
for the `Heading{N}` styleId rewrite, and per-section page numbering (`w:pgNumType`),
page margins (`w:pgMar`) and header distances on the section-break paragraphs.
See [docx2hub/CHANGELOG.md](docx2hub/CHANGELOG.md) for the per-commit details.

## xml2tex fork — 1 commit ahead of master

CALS table row shading (`owcolor`) and border rule colours (`rrayrulecolor`) in
`calstable2tabular`, so Word table styling (navy header rows, grey grids) renders as in
the reference. See [xml2tex/CHANGELOG.md](xml2tex/CHANGELOG.md).

## mml2tex fork

No changes on `front-page-layout` (pinned at master).
