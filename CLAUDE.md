# CLAUDE.md

Guidance for AI assistants working in this repository.

## Project overview

**계통수 랩 / Phylo Lab** is a Korean-language educational web app for building
**phylogenetic trees (cladograms)** from a trait matrix. Students enter traits
(형질) and species (종), mark which species have which traits in an analysis
table, and the app draws the tree and scores it. It also supports shareable
challenge modes (quiz / draw / fossil) sent between users via URL links.

The entire app is a **single self-contained file: `index.html`** (~1,400 lines)
with all HTML, CSS, and vanilla JavaScript inline. The UI text and code comments
are in **Korean** — match that voice when editing.

## Architecture

- **`index.html`** — the only source file. Everything lives here.
- **`.nojekyll`** — disables Jekyll so GitHub Pages serves the file as-is.
- **No build step, no package manager, no dependencies to install, no tests.**
- Three **optional** CDN libraries are loaded with `defer`, each guarded so the
  app degrades gracefully if the script fails to load — **do not add new
  external dependencies**:
  - `canvas-confetti` — success celebrations (`window.confetti` guarded)
  - `lz-string` — share-link compression (`window.LZString` guarded)
  - `qrcode-generator` — QR codes for share links (`window.qrcode` guarded)

### Script layout (numbered sections)

The `<script>` is organized into numbered comment-header sections 0–13. Keep
this structure when adding code:

| # | Section | Key items |
|---|---------|-----------|
| 0 | Global state | `traits`, `species`, `appMode`, `analysisModel`, `answerMatrix`, `LS_KEY`, `PHYLO_DATABASE`, `EXAMPLE`, `esc()` |
| 1 | Data mutation | `commit()`, `addTrait`/`addSpecies`, `toggleCell`, `buildMission` (random mission generator) |
| 2 | Persistence & share links | `serialize`/`deserialize`, `saveLocal`/`loadLocal`, `encodeState`/`decodeState` |
| 3 | Chips / matrix render | `renderMatrix`, `refreshLockUI`, neon conflict highlighting |
| 4 | Tree build + analysis | `buildNode` (parsimony), `buildUPGMATree` + `hammingDistance` (UPGMA), `parsimonyLength` (Fitch), `analyzeConflicts`, `analyzeHomoplasy` (수렴진화 진단: 형질별 독립 기원 수로 공유파생형질·수렴진화·형질 소실 분류) |
| 5 | Result render | `renderTree`, SVG drawing, celebration/fail effects |
| 6 | Quiz matching | `quizProgress`, `checkQuiz` |
| 7 | Draw builder | tap-to-merge clade builder (`drawTap`, `checkDraw`) |
| 8 | HUD + pass handling | `updateHUD`, `onSolved` |
| 9 | Submission | author/solver names stamped into links |
| 10 | Views / modals / share | `showView`, `openShare`, `toast` |
| 11 | Help / intro modals | `openHelp`, `quizIntro`, etc. |
| 12 | First-visit guided tour | `TOUR`, `startTour` |
| 13 | Init / URL routing | `init`, `bindEnter`, the `Object.assign(window, {...})` export block |

## Domain model

- **traits** (형질 / 공유파생형질 = synapomorphies): array of strings.
- **species** (종): `{ name, has: Set<trait>, unknown: Set<trait> }`. A species
  with an empty `has` set is the **outgroup (외군)**.
- Matrix cells cycle **O → X → ?** via `toggleCell` (`?` = `unknown`, used for
  fossil/missing data).
- **Two analysis models** (`analysisModel`), chosen in the table UI:
  - `parsimony` — Maximum Parsimony. `buildNode()` constructs the tree;
    `parsimonyLength()` scores it (Fitch algorithm), reported as "진화 비용".
  - `upgma` — UPGMA distance clustering. `buildUPGMATree()` with
    `hammingDistance()` (which skips `?` cells).

### App modes (`appMode`)

- `free` — normal free editing (default).
- `quiz` — 🎯 도전과제: answer matrix is hidden; user fills the blank table to match.
- `draw` — 🌳 계통수 그리기: table locked read-only; user builds the tree by tapping.
- `fossil` — 🦖 화석 추론: some cells blanked to `?`; user restores them.

Locking is enforced by `structureLocked()` (blocks trait/species edits) and
`cellsLocked()` (read-only cells). Quiz/draw/fossil lock until `solved`.

### Views (SPA)

Five views switched by `showView(n)`: `0` 시작 / `1` 형질 / `2` 종 / `3` 분석표 /
`4` 계통수. Bottom nav and step buttons call `showView`.

## Conventions — follow these when editing

1. **Inline `onclick` handlers need global functions.** Handlers are written as
   inline `onclick="fn()"` in the HTML/template strings. Any new handler
   function MUST be added to the `Object.assign(window, { ... })` export block
   in section 13, or it will be `undefined` at runtime.
2. **Escape user input with `esc()`** before injecting any user-derived string
   into HTML (XSS guard used consistently throughout).
3. **Korean UI & comments** — keep new strings and comments in Korean to match.
4. **Preserve the maker attribution.** Do not remove or alter the header comment
   credit, the `MAKER` / `MAKER_YEAR` constants, or the `.maker-credit`
   rendering. The credit is also stamped into every share link via
   `encodeState` (`mk:MAKER`).
5. **Keep the single-file, numbered-section structure** — do not split into
   modules or add a build system.
6. **CSS uses design tokens** declared in `:root` (Botanical Modern UI palette,
   `--primary`, `--secondary`, `--accent`, etc.). Reuse tokens rather than
   hardcoding colors.

## Persistence & share links

- **localStorage**: state under key `phylo_lab_v4` (`LS_KEY`); tour-completion
  flag under `phylo_lab_tour_done`. Locked challenge state is not persisted
  (so it can't overwrite the user's free data).
- **Share links**: state is encoded into the URL hash `#d=...`.
  `encodeState` prefers `LZString` compression (`L` prefix) and falls back to
  base64 (`B` prefix); `decodeState` reverses it and routes into the right mode.
  Link makers: `makeQuizLink`, `makeDrawLink`, `makeSubmitLink`.

## Development & deployment workflow

- **Edit** `index.html` directly.
- **Test** by opening `index.html` in a browser — no server is required. For
  clean hash routing you can serve locally with `python3 -m http.server` and
  open `http://localhost:8000/`. There is no automated test suite; verify
  behavior manually (add traits/species, toggle cells, view the tree, try a
  share link).
- **Deploy**: the site is published via **GitHub Pages** straight from the repo;
  pushing the updated `index.html` updates the live site.
- **Commit & push** to the active feature branch (do not push to `main`
  without explicit permission).
