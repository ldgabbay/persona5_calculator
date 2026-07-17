# Codebase Map

## System overview

A **Persona 5 / Persona 5 Royal fusion calculator** — a static, client-side web
app (originally by chinhodado, deployed to GitHub Pages). Given a persona, it
computes every 2-persona fusion recipe that produces it ("fusion to") and every
recipe it can be an ingredient in ("fusion from"), plus browsable persona stats,
elemental affinities, skills, itemization, and inheritance charts.

There are two parallel versions sharing one code path:
- **Vanilla P5** — entry point `index.html`, data in `data/*5.js` (non-Royal).
- **P5 Royal** — entry point `indexRoyal.html`, data in `data/*Royal.js`.

This repo is a **fork** (`git@github.com:ldgabbay/persona5_calculator.git`,
upstream `chinhodado/persona5_calculator`). Working branch is `develop`;
canonical trunk is `master`. As of this writing `develop` is 0 commits ahead of
`master`.

## Architecture

Plain AngularJS 1.5 SPA. No build tooling beyond `tsc`. Everything is loaded as
ordered `<script>` tags in the two HTML entry points — there is no module
bundler. TypeScript compiles 1:1 to ES5 `.js` next to each `.ts` (both the `.ts`
and generated `.js` are committed).

Load order (see `index.html`): third-party libs (jQuery slim, sticky table
headers, Angular + angular-route) → data files → `GLOBAL_IS_ROYAL` flag →
`src/*.js` (DataUtil, FusionCalculator, controllers, App).

**Layers:**

1. **Data layer** (`data/`) — large hand-maintained object literals declared as
   top-level `const`s (global in browser scope). Keyed maps: `personaMap`,
   `skillMap`, `itemMap`; plus fusion rule tables `rarePersonae`, `rareCombos`,
   `arcana2Combos`, `specialCombos`, `dlcPersona`, `inheritanceChart`. The
   `Data5.ts` file holds the rule tables; `PersonaData.ts` / `SkillData.ts` /
   `ItemData.ts` hold the big content maps and the TypeScript `interface`s
   describing each record.

2. **Derivation layer** (`src/DataUtil.ts`) — runs at load time (IIFEs) to
   transform the raw maps into the structures the app uses:
   - `fullPersonaList` — every persona as an array (name injected from key).
   - `customPersonaList` — `fullPersonaList` minus DLC personas the user hasn't
     marked as owned (per `localStorage["dlcPersona"]`).
   - `customPersonaeByArcana` — `customPersonaList` grouped by arcana, each group
     sorted ascending by level. **This is the structure fusion runs against.**
   - `skillList`, `arcanaMap` (fast arcana→arcana→result lookup),
     `special2Combos` (the 2-ingredient subset of `specialCombos`).
   - Plus display helpers (element name expansion, skill cost formatting,
     persona-link HTML, etc.).

3. **Fusion engine** (`src/FusionCalculator.ts`) — the core. A `FusionCalculator`
   is constructed with a `personaeByArcana` map. Pure logic, no DOM/Angular.
   Key methods:
   - `fuse(p1, p2)` — dispatches to special → rare → normal fusion.
   - `fuseNormal` — average-level + arcana-table lookup; handles same-arcana
     down-rank vs different-arcana up-rank; skips special/rare results.
   - `fuseRare` — rare (Treasure Demon) fusion via `rareCombos` offset table.
   - `getRecipes(persona)` — all recipes that PRODUCE the persona (special,
     normal, and rare), filtered/deduped.
   - `getAllResultingRecipesFrom(persona)` — all recipes the persona is an
     ingredient IN.
   - `getApproxCost` — the quadratic cost estimate (`27L² + 126L + 2147` per
     ingredient). README notes cost is an estimate, not exact.

4. **Controllers** (`src/*Controller.ts`) — thin AngularJS controllers, one per
   route. They call the engine and shape data onto `$scope` for the templates.

5. **App wiring** (`src/App.ts`) — declares the `myModule` Angular module, the
   `stickyTable` directive, and `$routeProvider` routes.

6. **Views** (`view/*.html`) — Angular templates loaded by route
   (`templateUrl`). The HTML entry points contain only the shell + `<ng-view>`.

**Routing** (`src/App.ts`):
- `/list` → `view/list.html` / `PersonaListController` (default route)
- `/skill` → `view/skill.html` / `SkillListController`
- `/persona/:persona_name` → `view/persona.html` / `PersonaController`
- `/setting` → `view/setting.html` / `SettingController`

**Royal reuse mechanism** (`indexRoyal.html`, self-described "dirty quick hack"):
Royal data files declare `*Royal`-suffixed globals (`personaMapRoyal`,
`rareCombosRoyal`, …). `indexRoyal.html` then aliases the unsuffixed names the
`src/` code expects to the Royal versions (`var personaMap = personaMapRoyal;`
etc.) before loading `src/*.js`. So **one copy of `src/` logic serves both
versions**; only the data and the alias block differ. `GLOBAL_IS_ROYAL` (set in
each entry point) tells `SettingController.save()` which page to redirect back
to.

## Where things live

- **Fusion rules / mechanics** → `data/Data5.ts` (and `data/Data5Royal.ts`):
  `rareCombos` (per-arcana ±1/±2 offset table for the 8 rare personas),
  `arcana2Combos` (arcana + arcana → result arcana), `specialCombos`
  (named multi-ingredient recipes), `rarePersonae` (the 8 Treasure Demon names),
  `dlcPersona` (pairs of `[base, Picaro]` names), `inheritanceChart`
  (affinity-inheritance ✓/✘ grid by skill category).
- **Persona records** (stats, elems, skills, arcana, level, item, inherits,
  mementos area/floor, dlc/rare/special/max flags) → `data/PersonaData.ts` /
  `PersonaDataRoyal.ts`. Type: `interface PersonaData` at top of that file.
- **Skill records** (effect, element, cost, which personas learn it + level,
  talk/fuse sources) → `data/SkillData.ts` / `SkillDataRoyal.ts`. Type:
  `interface SkillData`.
- **Equipment / item records** (from itemization) → `data/ItemData.ts` /
  `ItemDataRoyal.ts`. Type: `interface ItemData`.
- **The fusion algorithm** → `src/FusionCalculator.ts`.
- **Load-time data derivation & display helpers** → `src/DataUtil.ts`.
- **DLC ownership toggle + persistence** → `src/SettingController.ts` (reads/
  writes `localStorage["dlcPersona"]`, a JSON map of name→bool). Read side:
  `isDlcPersonaOwned()` in `DataUtil.ts`.
- **Persona detail page logic** (fusion-to/from tables, pagination, filtering,
  stats/elems/skills/inheritance assembly) → `src/PersonaController.ts`.
- **Persona list & skill list pages** → `src/PersonaListController.ts`,
  `src/SkillListController.ts`.
- **Element-affinity cell coloring** → `style.css` (classes like `.wk`, `.rs`,
  `.nu`, `.rp`, `.ab` applied in `view/persona.html`).
- **Tests** → `test/FusionTest.ts` (+ `FusionTestRoyal.js`), `test/TestUtil.ts`
  (helpers + data-integrity checks), run via `test/test.js` (a Mocha harness).
- **CI** → `.github/workflows/nodejs.yml`: on push, `npm install` → `tsc` →
  `node test/test.js` (Node 20).

## Design goals and constraints

- **Zero-backend, static hosting.** The whole app is HTML + committed JS,
  designed to run from GitHub Pages by opening an HTML file. No server, no API.
- **Fusion correctness is the point.** The mechanics are subtle (rare/Treasure-
  Demon fusion where the result depends on the ingredient's *current level*,
  down-rank same-arcana fusion, special recipes, DLC skips, and the real-game
  quirk that Judgement + Justice/Strength/Chariot/Death is impossible despite
  the official guide). These rules are documented in the README and encoded in
  `FusionCalculator` + the `data/Data5.ts` tables, and guarded by
  `test/FusionTest.ts`.
- **Assumptions baked into results** (documented in README & surfaced in
  `view/persona.html` notes): user owns *no* DLC persona by default; rare
  fusions assume the ingredient is at its *base* level.
- **Committed build output.** Both `.ts` and generated `.js` are committed. The
  browser loads the `.js`; `tsc` regenerates it. After editing any `.ts` you
  must re-run `npm run compile` (`tsc`) and refresh.

## Working preferences

- **`master` is the trunk** and the base for PRs (user's global rule; also the
  fork's default branch). Do not treat `main` as base.
- Match the existing plain-AngularJS-1.5 / global-`const` / hand-maintained-data
  style; there is no framework upgrade or bundler in play.
- When changing data or fusion logic, keep the `.ts` and committed `.js` in sync
  by running `tsc`, and run `node test/test.js` — CI does exactly this.

## Known limitations / deliberate debt

- **The Royal "dirty quick hack"** (author's own words in `indexRoyal.html`):
  Royal support is implemented by aliasing `*Royal` globals onto the unsuffixed
  names rather than parameterizing the code. It works but means the two versions
  share mutable global names; load order and the alias block are load-bearing.
- **Committed compiled `.js` alongside `.ts`** — easy to get out of sync if you
  edit one and forget `tsc`.
- **`test/FusionTestRoyal.js` has no committed `.ts` counterpart** (only
  `FusionTest.ts` exists). The Mocha harness (`test/test.js`) picks up every
  `.js` in `test/`, so this Royal test still runs; just note there's no source
  `.ts` for it in the tree.
- **Cost is an approximation** (README explicitly says so); `getApproxCost` is a
  fitted quadratic, not the game's real formula.
- Recipe-count assertions in `test/FusionTest.ts` under "normal fusion" are
  self-described as possibly-not-authoritative ("count the number of recipes and
  may not be correct") — they're regression guards, not ground truth.
- Third-party libs (jQuery, Angular, Semantic-UI CSS) load from CDNs; opening
  the file locally may need CORS relaxed (README note).

## Open design questions

None recorded yet.
