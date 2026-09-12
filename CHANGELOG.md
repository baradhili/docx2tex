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

## Base repo (docx2tex) — `front-page-layout` vs pre-merge master

(The `front-page-layout` branch was squash-merged into master as `0e00985`
"Handle more complex/messy word layout and styling(#1)"; the sections above
document that pass. The `further-fixes` pass below is 10 commits on top.)

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

CALS table row shading (`
owcolor`) and border rule colours (`rrayrulecolor`) in
`calstable2tabular`, so Word table styling (navy header rows, grey grids) renders as in
the reference. See [xml2tex/CHANGELOG.md](xml2tex/CHANGELOG.md).

## mml2tex fork

No changes on `front-page-layout` (pinned at master).

## `further-fixes` pass — second and third documents (example03, resume)

Driven by two more reference documents (`.zcode/example3` "Word Documents
Template", `.zcode/example4` one-page resume), each verified against its OnlyOffice
render. After every fix, all previously matching documents were rebuilt and their
known-good state re-checked (the guard caught one bad interaction before commit).

### example03 — whole-document fidelity

- **Unnumbered Word headings stay unnumbered** (`735ce58`): documents whose heading
  styles carry no list numbering got LaTeX numbers Word does not show. When no
  headline paragraph carries a Word number (detected via the `\label{mark-…}` PIs
  the preprocessor leaves behind — the identifier phrases themselves are stripped
  earlier), the preamble sets `secnumdepth` to -2 and titlesec drops the labels.
- **Image-only heading paragraphs render as in-flow images; body images take their
  Word size** (`b899107`): a screenshot dropped into a Heading-2 paragraph became a
  sectioning command with the `\includegraphics` as its title, and body images were
  stretched to `\textwidth` regardless of their Word display size — together a
  blank page (6 pages vs the reference's 5). Image-only headings lose their headline
  marking (alt text does not count as text) and images use the declared
  `css:width`/`css:height`.
- **Body font families switch** (`eb27692`): font macros collect families from all
  phrases (not just header/footer parts) and use `\newfontfamily` so bold/italic
  shapes are inherited — the Times New Roman print-size sample renders in Times
  inside the Arial document.
- **Word's default bullet is the middle dot** (`96ca068`): Symbol-font F0B7 renders
  as "·" (`\item[\textperiodcentered]`), matching the reference's Standard
  Symbols PS glyph.
- **Document-default line spacing** (`0d56230`): a "1.5 lines" Normal style
  (surfaced by docx2hub as `css:default-line-height`) applies a global
  `\linespread{1.5}`; single-spaced documents are unaffected.
- **Paragraph spacing from the Normal style's space-after** (`9c1f44f`): the
  hardcoded `\parskip` 8.5pt (an HSS_REP calibration) inflated every paragraph of
  documents whose Normal style declares no space-after (Word default 0); parskip is
  now driven by the surfaced value. Example3 renders 5 pages as the reference.

### example4 — floating layout, fonts, vector art

- **Font macros without header/footer parts** (`a082dc1`): the `\docxhffont`
  definitions moved out of the header/footer block (documents without any — the
  resume — referenced undefined macros; 44 errors).
- **Pipeline SVGs convert to PDF** (`caec009`): drawingml2svg shapes are written
  with document-order indices (svg shapes value-compare equal, so `index-of` gave
  them all one name), the tex references same-basename PDFs converted by
  `rsvg-convert` in the d2t wrapper after the pipeline, and zero-width degenerate
  conversions are skipped. The six vector icons render.
- **Absolutely positioned body text boxes** (`c97268b`): VML fallback shapes with
  `position:absolute` render as textpos textblocks at their Word coordinates
  (margin offsets + page margins, width from the shape style, 1pt modules). The
  resume's two-column layout renders; header/footer shapes and margin-less shapes
  are excluded (the classification marking would otherwise produce NaN
  coordinates — caught by the example2 guard).
- **VML length units are honoured** (base repo, conf): the text-box template read
  `margin-left/top` and `width` by stripping the unit and treating the number as
  pt, so Word's `margin-top:5in`/`8in` boxes landed at y≈77/80pt over the name
  and header (the visible "D i r e cCourse" garbling and stacked job titles). A
  `tr:css-length-to-pt` helper now converts in/cm/mm/pc/px to pt (unitless = pt,
  unparseable values fall back), also for the page-margin bases; the two boxes
  sit at 432/648pt as in the reference.
- **Lato-only fonts: no ArialMT, bullets restored** (base repo, conf): ArialMT
  was used by a single glyph — the class default footer page number, emitted
  because the `\pagestyle{empty}` suppression sat inside the
  `exists(docx-headers)` block. The reference (a docx without footer parts)
  prints no page number at all; the suppression now also emits for headerless
  documents (fancyhdr loaded just for the plain redefinition). Separately, all
  11 bullet glyphs were silently dropped — soul analyses `\so{}` arguments in
  a hardcoded 8-bit ectt1000 that lacks U+2022; the preamble redefines
  `\SOUL@tt` to lmmono10-regular.otf under LuaLaTeX (hyphen width re-measured),
  so bullets render from the document font. The PDF now embeds the five Lato
  faces only, 0 errors, 0 missing-character warnings.

### Submodule wiring

- **Three more baradhili forks** (`ca0ddc9`): `calabash`, `cascade` and `evolve-hub`
  now fetch from github.com/baradhili and advanced to the fork masters
  (fast-forwards: evolve-hub +5, cascade +3, calabash +1). The remaining submodules
  (xslt-util, xproc-util, htmlreports, mml-normalize, fontmaps) are unmodified
  upstream pins — fork when first changed.

Known open items for the resume (`.zcode/plans/futher-fixes.md`): the portrait
photo (a shape `blipFill` swallowed by drawingml2svg) and in-box spacing
fine-tuning.
