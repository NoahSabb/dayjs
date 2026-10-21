# AGENTS.md

## What this is

Day.js: a ~2 KB immutable date library with a Moment.js-compatible API. The core is one class in `src/index.js`; everything else is an opt-in plugin or locale. Source is ES6 compiled by Babel. There is no TypeScript source, only hand-written `.d.ts` files.

## Layout

- `src/index.js` core `Dayjs` class. `src/utils.js` helpers, exposed to plugins as `this.$utils()`. `src/constant.js` unit names (`C.D`, `C.M`, `C.Y`) and the parse/format regexes; use these, not string literals.
- `src/plugin/<name>/index.js` one plugin per folder, shaped `export default (option, Dayjs, dayjs) => { Dayjs.prototype.x = function () { ... } }`. To change an existing method, save it first (`const oldStartOf = proto.startOf`) and call it inside the new one.
- `src/locale/<code>.js` one locale per file, lowercase code. Line 1 must be `// English Name [code]`; `build/index.js` parses that comment into `locale.json`. Locale files `import dayjs from 'dayjs'` and end with `dayjs.locale(locale, null, true)`; in jest that import resolves to `src/` through `test/__mocks__/dayjs.js`.
- `types/index.d.ts`, `types/plugin/<name>.d.ts`, `types/locale/` hand-written declarations. The build copies `types/` to the repo root.
- `test/*.test.js` core by topic (`parse`, `display`, `manipulate`, ...). `test/plugin/<name>.test.js`, `test/locale/<code>.test.js`, `test/issues/issueNNNN.*.test.js` for regressions.
- `build/` rollup scripts. `docs/` is stale; the real docs live at day.js.org in a separate repo.

## Commands

Run `npm ci` once. Node 24 and 25 both work for lint and test.

- Lint: `npm run lint` (eslint airbnb-base over `src test build`). Also runs as a pre-commit hook.
- One test file: `npx jest test/plugin/isToday.test.js --coverage=false` (about 1 s).
- Full suite, what CI runs: `TZ=UTC npm test`. It runs the timezone suites under 4 timezones, then everything with a 100% line-coverage gate. The `TZ=UTC` matters: on a Pacific-time Mac without it, 6 DST tests in `timezone`, `dayOfYear` and `localizedFormat` fail that pass in CI (ubuntu, UTC).
- Build: `NODE_OPTIONS=--openssl-legacy-provider npm run build`. Rollup 2 needs the flag on Node 17+; `release.yml` sets the same one. Writes `locale/`, `plugin/`, `dayjs.min.js`, `index.d.ts` and `locale.json` to the repo root and fails if `dayjs.min.js` exceeds the 2.99 KB `size-limit`. `npm run babel` builds `esm/`.

## Conventions

- Style: no semicolons, no trailing commas, single quotes, 2-space indent. ESLint 4 enforces it; `npm run prettier` only touches `docs/**/*.md`.
- Tests: jest 22. Import the source, `import dayjs from '../src'`, never the package name or build output. Freeze time with `MockDate.set(new Date())` in `beforeEach` and `MockDate.reset()` in `afterEach`. Moment is the oracle: `expect(dayjs(d).format()).toBe(moment(d).format())`.
- Coverage is 100% of lines in `src/**`. Every new branch needs a test or `npm test` fails at the end.
- A new plugin is three files: `src/plugin/<name>/index.js`, `types/plugin/<name>.d.ts` (`test/plugin/declarations.test.js` checks it exists) and `test/plugin/<name>.test.js`. A new locale is `src/locale/<code>.js` plus, ideally, `test/locale/<code>.test.js` comparing against moment's locale.
- Keep the core tiny. Feature code goes in a plugin, not `src/index.js`; the size limit is the gate.
- Commits are conventional: `fix(plugin): ...`, `fix: ...`, `feat: ...`, `test: ...`, `chore: ...`. semantic-release derives the version and `CHANGELOG.md` from them. Locale fixes are `fix:`.
- Pull requests target `dev`. `master` is release-only; a push there runs semantic-release and publishes a release.

## Guardrails

- Never edit `locale/`, `plugin/`, `esm/`, `dayjs.min.js`, `index.d.ts` or `locale.json` at the repo root. They are build output and gitignored. The sources are `src/` and `types/`.
- Do not touch `test/browser.spec.js` (Sauce Labs; the file says so), `CHANGELOG.md` (written by semantic-release) or `patches/`.
- Do not "fix" the 6 DST tests that fail without `TZ=UTC`. Set the variable.
- Do not add dependencies. The core has none, and the dev dependencies are old on purpose (jest 22, eslint 4, rollup 2); `npm install <pkg>` rewrites `package-lock.json`.
