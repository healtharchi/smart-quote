# CLAUDE.md

Guidance for AI assistants (Claude Code etc.) working in this repository.

## What this is

**HEALTH ARCHI スマート見積システム** — a free, browser-based estimating tool
for Japanese wooden-house (木造) builders/工務店. The user enters basic building
parameters (坪数 / floor count / building & roof shape) and the app auto-generates
a full new-construction quote from a built-in 136-item unit-price master (単価マスタ),
then prints/PDFs it in three formats.

Key product facts:
- **Audience & language:** Japanese SMB construction firms. **All UI, domain
  terms, data, and comments are in Japanese.** Preserve this — do not anglicize
  labels or messages.
- **Privacy promise:** all data lives in the browser (`localStorage`). Nothing is
  sent to any server. Do not add network calls, analytics, or telemetry.
- **Deployment:** static files served from GitHub Pages. There is no backend.

## Repository layout

```
.
├── index.html   ← the entire application (single file, ~2,270 lines)
├── guide.html   ← standalone user manual (使い方ガイド), static HTML/Tailwind
└── README.md    ← short product README (Japanese)
```

That's it. **There is no other source.** `index.html` contains everything:
embedded data, styles, and the full React app.

## Tech stack & architecture (important)

This is **not** a typical bundled SPA. There is **no build step, no package.json,
no node_modules, no tests, no CI, no lint config.**

Everything is loaded from CDNs inside `index.html`:
- **React 18 + ReactDOM** (UMD, production builds via unpkg)
- **Babel Standalone** — JSX/`react` preset is transpiled **in the browser at
  runtime** (`<script type="text/babel" data-presets="react">`)
- **Tailwind CSS** via `cdn.tailwindcss.com` (utility classes used directly in JSX)

Implications for editing:
- You edit JSX directly inside the `<script type="text/babel">` block in
  `index.html`. No transpile/build is needed or possible here.
- Tailwind classes are resolved at runtime by the CDN script — any utility class
  works without config.
- Because Babel runs in-browser, **a syntax error breaks the whole app silently**
  (blank page, error only in console). Be careful with JSX.

### How `index.html` is organized (top to bottom)

1. `<head>`: meta, CDN script tags, `<style>` (fonts, `.num` tabular figures,
   `@media print` rules, scroll helpers).
2. `<script id="app-data" type="application/json">` (line ~35): the **seed unit-price
   master** (`rateMaster`) — 136 line items. This is the factory default data.
3. `<script type="text/babel">` (line ~37 onward): the React app. Structure:
   - Constants: `LS_KEY`, `LS_MASTER_KEY`, `APP_DATA_INITIAL`
   - **Utilities** (`uuid`, `fmt`, `fmtDate`, `cleanClientName`, category-code
     helpers `codeFromNumber` / `nextAvailableCategoryCode` / `compareCategoryCode`,
     `sectionRank` / `sortGroupsBySection`)
   - **Calculation engine** (`computeScaleParams`, `evalQuantityFormula`,
     `getEffectivePrice`, `computeEstimate`)
   - **Storage** (`loadAll` / `saveAll` / `loadMasters` / `saveMasters`,
     `defaultProject`)
   - **Components** (`App` and ~30 child components)
   - Mount: `ReactDOM.createRoot(...).render(<App/>)` at the very bottom.

### Component map (search by `function <Name>`)

- `App` — root; holds all state, view routing, CRUD for properties/projects.
- `Header` — top nav / view switcher.
- `PropertyList` → `ProjectList` → `ProjectEditor` — the main drill-down flow
  (物件 → 見積一覧 → 見積編集).
- `ProjectEditor` tabs: `TabScale` (規模入力), `TabDetails` (内訳), `TabResult`
  (結果) + `PrintPreview`.
- Print formats: `FormatStandard` (シンプル), `FormatDetailed` (内訳付き),
  `FormatFormal` (正式見積書) + `SummaryTable`, `FooterNotes`.
- `MasterManager` / `RateMasterEditor` — edit the shared unit-price master.
- `CategoryAddModal` — custom in-page modal for adding categories (replaces
  `prompt()` for that flow).
- `Settings` — company info, stamp image, JSON import/export, full reset.
- Small reusables: `Field`, `SelectField`, `Row`, `TextField`, `OptionPresets`.

## Data model & domain concepts

State is split across **two `localStorage` keys**:
- `kentiku_estimate_v1_0` (`LS_KEY`) → app state: `{ projects, properties, companyInfo }`
- `kentiku_master_v1_0` (`LS_MASTER_KEY`) → `{ rateMaster }` (the user's editable
  master, seeded from `APP_DATA_INITIAL`)

### 区分 (sections) and カテゴリ (category groups)

Every line item has:
- `category` — one of three top-level **sections**: `本体工事` / `付帯工事` / `諸経費`.
- `categoryGroup` — an Excel-style column code (`A`, `B`, … `Z`, `AA`, `AB`, …)
  identifying the category. New custom categories get the next free code via
  `nextAvailableCategoryCode`. Sections always render in the fixed order above
  (`sectionRank` / `sortGroupsBySection`); new categories append to the *end of
  their section*.
- `no` — stable internal id (e.g. `A-1`). **`no` is the key** used in
  `excludedItems`, `manualPrices`, `manualQuantities`. Never reuse/renumber it
  for existing items. `displayNo` is the *recomputed* sequential number shown to
  users (excludes disabled/excluded items).

### rateMaster item shape

```
{ category, categoryGroup, categoryName, no, name, spec,
  quantity, unit, price, amount, remark, quantityFormula }
```

`quantityFormula` is a small DSL evaluated by `evalQuantityFormula`:
- Variables (Japanese): `延床坪数`, `階数`, `軒高`, `建築面積`, `外周長`,
  `延床面積`, `外壁面積`, `屋根面積`, `内装床面積`.
- `×` is treated as `*`; supports `IF(cond, then, else)`.
- ⚠️ Evaluated via `new Function(...)`. This is acceptable **only** because
  formulas come from the bundled master / the user's own machine (no remote
  input). Do not feed untrusted data into it.
- ⚠️ Subtle: the variable `軒高` in formulas maps to **`wallHeight` = 軒高 + 2.0m**
  (外壁高さ), not the raw eaves height. See the comment in `evalQuantityFormula`.

### Project (見積) shape — see `defaultProject`

A project belongs to a property (`propertyName`) and carries:
- inputs: `scale` (規模: 坪数/階数/軒高/建物形状/屋根形状 + optional manual
  overrides for 建築面積/各階面積/外周長/外壁面積/屋根面積)
- overrides: `excludedItems` (no[]), `manualPrices` (no→price),
  `manualQuantities` (no→qty), `customItems` (user-added lines), `options`
- **`rateMasterSnapshot`** — a deep copy of the master taken when the project is
  created. `computeEstimate` prefers the snapshot over the live master, so editing
  the master does **not** retroactively change existing quotes. `refreshSnapshot`
  re-syncs a project to the latest master on demand (preserving manual edits &
  custom items).

### Calculation flow (`computeEstimate`)

1. `computeScaleParams(project.scale)` derives geometry (建築面積 priority:
   direct input > 1階面積 > 延床÷階数; shape factors for 不整形/屋根).
2. For each master line: resolve enabled state, quantity (manual override → else
   formula), price (manual override → else master), `amount = round(qty × price)`.
3. Append `customItems`, assign `displayNo`, sum per section, add options,
   **tax = 10%** (hard-coded `0.1`), compute `total` and 坪単価 (`tsuboCost`).

## Conventions to follow

- **Match the surrounding style.** 2-space indent, single quotes, Japanese
  identifiers/strings where the codebase already uses them, Tailwind utilities
  inline. Comments are written in Japanese — keep new comments Japanese too.
- **Keep it single-file and dependency-free.** Do not introduce npm packages,
  bundlers, frameworks, or new CDN dependencies without explicit instruction.
- **Numbers:** format displayed amounts with `fmt()` (rounds + `ja-JP` locale).
  Use the `.num` class for tabular figures in tables.
- **Currency/tax:** consumption tax is 10% and hard-coded in `computeEstimate`.
  If asked to change it, that is the single source of truth.
- **IDs:** create ids with `uuid()`. Category codes via `nextAvailableCategoryCode`.
- **Backward compatibility matters** — users have real data in `localStorage`.
  Migrations live in `loadAll` / `loadMasters` (e.g. defaulting missing
  `properties`). Reading old project shapes must keep working; prefer
  `project.rateMasterSnapshot || masters.rateMaster` style fallbacks.
- **No server / no tracking.** Preserve the privacy guarantee. Stamp/image uploads
  are read locally as data URLs (≤500KB) — keep that pattern.
- The footer string and `README.md` mention a version (`v1.0`) and the seed-data
  description ("2026年5月 首都圏中級32坪"). Update those if the seed data or version
  changes.

## Running, testing & deploying

- **Run locally:** open `index.html` directly in a browser, or serve the folder
  (e.g. `python3 -m http.server`) and visit it. No install/build.
  - Note: the page needs **internet access** to fetch React/Babel/Tailwind from
    the CDNs; it will not render fully offline.
- **Testing:** there is no automated test suite. Verify changes **manually in a
  browser**: check the browser console for Babel/React errors, exercise the flow
  (add property → create quote → edit scale/details → view result → print
  preview → export/import JSON). Test with existing `localStorage` data to catch
  migration regressions.
- **Deploy:** GitHub Pages serves the repo's static files. Merging to the
  published branch is the deploy. There is no CI/CD pipeline.

## Git workflow

- Work on the designated feature branch; commit with clear, descriptive messages
  (the existing history uses concise, imperative summaries, often in Japanese or
  English — match that). Push with `git push -u origin <branch>`.
- **Do not open a pull request unless explicitly asked.**
- Do not commit anything secret — there are no env files or credentials in this
  repo, and none should be added.
