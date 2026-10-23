# htmx

htmx (published as `htmx.org` on npm) is a dependency-free browser library. It lets HTML elements make AJAX requests and swap the response into the page via `hx-*` attributes. This checkout is htmx **2.x**. The library is one hand-written ES2015 file with JSDoc types and no build step for development.

## Layout

- `src/htmx.js`: the entire library. It is a single IIFE (`var htmx = (function() { ... })()`). The public `htmx` object at the top declares API slots as `null`, and the lines right after it assign each slot to an internal function (`htmx.process = processNode`). `htmx.config` (also at the top) holds every option, each with an `@default` JSDoc tag.
- Map of `src/htmx.js` (grep `function <name>(`; line numbers drift). The `//= ===` section banners are too coarse, since "Node processing" alone runs ~1,700 lines.
  - Processing: `processNode` → `initNode` → `getTriggerSpecs`/`parseAndCacheTrigger` (hx-trigger parsing) and `processVerbs` → `addTriggerHandler`; `boostElement` for hx-boost.
  - Request: `issueAjaxRequest` (`getTarget`, `getInputValues` for params, builds the XHR) → `handleAjaxResponse`, which is passed as the default `etc.handler`, so grepping for a direct call finds nothing.
  - Swap: `handleAjaxResponse` → `getSwapSpecification` (hx-swap parsing) → `swap` → `swapWithStyle`, plus `findAndSwapOobElements` → `oobSwap` for hx-swap-oob.
  - History: `saveCurrentPageToHistory`, `restoreHistory`. Events: `triggerEvent`. Extensions: `defineExtension`, `getExtensions`.
- `test/attributes/hx-<name>.js`: one Mocha file per attribute. `test/core/*.js` covers cross-cutting behavior: ajax, api, events, internals, regressions, security.
- `test/util/util.js`: the global test helpers `make`, `byId`, `clearWorkArea`, `makeServer`, `getParameters`.
- `test/index.html`: a manual Mocha page that lists every test file by hand. `test/manual/`: standalone HTML repros. Neither is run by CI.
- `www/`: the htmx.org site (Zola, deployed by Netlify from `netlify.toml`). Docs live in `www/content/`: `attributes/hx-*.md`, `reference.md` (attribute, event, and config tables), `events.md`, `api.md`, `docs.md`.

## Commands

```bash
npm ci                                  # Node 20+; Playwright downloads its own Chromium, so CHROME_PATH is not needed
npm test                                # lint + tsc type-check + headless Chromium suite (what CI runs)
npm test test/attributes/hx-target.js   # same chain, but only one test file
npm run test:ci                         # lint + types + Chromium/Firefox/WebKit; CI itself only runs Chromium
npm run lint                            # eslint check; `npm run format` auto-fixes
npm run types-check                     # tsc --checkJs over src/htmx.js; JSDoc types must stay valid
open test/index.html                    # in-browser Mocha runner for humans; agents can't read its results, so use `npm test <file>`
```

The runner prints every passing test plus browser logs. Filter it: `npm test 2>&1 | grep -E "test files|✗|❌|Error" | tail -20`. The `N/N test files | X passed, Y failed` line is the result. If the first file fails with "Browser tests did not start after 20000ms", it is a cold-start flake. Rerun before investigating.

## Conventions

- Style is eslint-config-standard with overrides in `package.json`: no space before function parens (`function(elt) {`), no semicolons, and `.substr` is banned (use `.slice`/`.substring`). Use `const`/`let` in `src/`. Tests use `var`.
- Keep `src/htmx.js` a plain ES2015 script. Do not add imports, exports, dependencies, or TypeScript syntax. Type everything with JSDoc so `npm run types-check` passes.
- Read attributes through `getAttributeValue`/`getClosestAttributeValue`/`hasAttribute`. Never call `elt.getAttribute('hx-…')` directly. These helpers make every attribute work with the `data-hx-*` prefix too.
- Fire events with `triggerEvent(elt, 'htmx:camelName', detail)`. It also dispatches the kebab-case form (`htmx:camel-name`).
- Test idiom: `this.server = makeServer()` (a sinon fake server) in `beforeEach`, plus `clearWorkArea()` in both hooks. Then `server.respondWith(...)`, build the DOM with `make('<button hx-get="/test">')`, call `btn.click()` and `this.server.respond()`, and assert with chai `should` (`div.innerHTML.should.equal('Clicked!')`). Reach internals in tests with `htmx._('functionName')`.
- A new test file must live in `test/attributes/` or `test/core/` (only those are globbed). Also add a `<script>` tag for it in `test/index.html`, or the browser runner silently skips it. `core/class-cleanup.js` is already missing from that list.
- A behavior change needs a test and a docs update. For a new attribute, add `www/content/attributes/hx-<name>.md` and a row in `www/content/reference.md`. For a new config option, add a row in both config tables, in `reference.md` and in `docs.md`.

## Guardrails

- **Do not edit or commit `dist/`.** Maintainers regenerate it with `npm run dist` at release time. That step also regenerates `src/htmx.esm.d.ts` (a copy of `dist/htmx.esm.d.ts`) and `editors/jetbrains/htmx.web-types.json` (built from the `www/content` docs). `dist/ext/` holds deprecated extension copies; extensions live in the separate `bigskysoftware/htmx-extensions` repo.
- Do not edit `www/static/src/`, `www/static/test/`, `www/static/node_modules/`, or `www/themes/htmx-theme/static/js/htmx.js`. `npm run www` (`scripts/www.sh`) overwrites them with copies.
- `www/reference.md` is an empty 2023 leftover. The real reference is `www/content/reference.md`.
- Do not bump `version` in `package.json` or in `src/htmx.js`. That is a release step.
- Branches: code PRs target `dev`, and docs-only (`www/`) PRs may target `master`. `four`/`four-dev` hold htmx 4 with a different layout, so do not port changes there.
- To iterate on one test, run its file with `npm test <file>` instead of adding `it.only`. Lint flags any `it.only` left behind.
- Upstream closes refactor-only and formatting-only PRs. Keep diffs minimal and scoped to the behavior change.
