# Changelog — `front-page-layout` vs `master`

Goal of this branch: make the docx2tex pipeline reproduce the Word/OnlyOffice rendering of
a styled Word report (reference document `HSS_REP.docx`, a Health Support Services template
with a cover section, classification marking, logo lockup and artwork footer). Measured
against the OnlyOffice PDF export, page 1 of the generated LaTeX PDF went from having seven
substantial differences to matching the reference: same page count (6), same content per
page, same fonts (Arial + Calibri), same colours, same marking position, correct logo
proportions and true A4 page geometry.

Verification artifacts and per-issue root-cause notes live in
`.zcode/plans/page1-diff-docx2tex-vs-onlyoffice.md`.

## Base repo (docx2tex) — 12 commits ahead of master

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

### Submodule wiring

- **baradhili forks** (`bb1dbca`, `237ac3d`, `7f65bb5`): `docx2hub`, `xml2tex` and
  `mml2tex` submodules point at the github.com/baradhili forks, track their
  `front-page-layout` branches (`branch = front-page-layout` in `.gitmodules`), and are
  pinned to the fork commits listed below.

## docx2hub fork — 4 commits ahead of master

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
