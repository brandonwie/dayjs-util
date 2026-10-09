---
tags: [personal, reference]
created: 2026-10-10
updated: 2026-10-10
status: active
---

# Review guide

The single review standard for this repository. Every reviewer applies it:
people, Claude (`@claude review`, wired in
`.github/workflows/claude-code-review.yml` through the shared workflow
`brandonwie/claude-review`), Codex and any other agent. Where a general best
practice disagrees with this file, this file wins.

## What to report

Report problems a maintainer would act on:

- **Bugs**: wrong instant or wall-clock time, timezone or DST mistakes,
  all-day dates that drift off midnight UTC, crashes on `null` or `undefined`
  input, mutated inputs.
- **API and packaging breaks**: anything in [Review areas](#review-areas)
  items 2 and 3.
- **Security**: workflow permissions, secret exposure, publish-path changes.
- **Missing tests**: a behavior change or new public method with no matching
  case in `src/DayjsUtil.spec.ts`.

Skip formatting and pure style, personal preference, speculative refactors,
anything in [Do not flag](#do-not-flag), and restating what the diff does.

For each finding give `path:line`, the failure (input and timezone, then the
wrong result) and a suggested fix. Tag a severity:

| Severity | Meaning                                                                     |
| -------- | --------------------------------------------------------------------------- |
| P0       | Wrong dates for all consumers, a broken published package, or a secret leak |
| P1       | Wrong output a consumer will hit, or an unflagged breaking change           |
| P2       | Edge case, or a risk to the next change                                     |

If nothing meets the bar, say so in one line rather than padding the review.

## Context

`@brandonwie/dayjs-util` is a timezone-safe wrapper around dayjs for calendar
applications, published to npm. Source lives in `src/`:

- `DayjsUtil.ts`: the static utility class, the main entry (`.`).
- `EventDateHandler.ts`: calendar event date normalizer (`toAllDayUTC`,
  `toTimedUTC`, `normalize`), also shipped as the `./event` entry point.
- `constants.ts` (`UTC`, `DATE_FORMAT`, `FORMAT_PATTERNS`, `RRULE_DAYS`),
  `types.ts` (`DateInput`, `TimezoneString`, `NormalizedEventDates`) and the
  barrel `index.ts`.

The central risk is **silent timezone error**: a date that parses, formats or
compares in the runtime's local zone, or in UTC when the caller passed a zone,
returns a plausible but wrong value. Every timezone-aware method takes the zone
as a parameter and applies it before arithmetic; `null` or an omitted zone
means UTC (`src/types.ts`). All-day events normalize to midnight UTC; timed
events keep their IANA zone.

## Review areas

1. **Timezone and DST correctness (primary).** The zone is applied before any
   arithmetic, boundary or comparison. No call relies on the host's local
   zone. DST gaps and overlaps behave as the README § How DST is Handled
   describes. Keep the `tz()` versus `tzParse()` distinction intact.
2. **Public API and semver.** Exports from `src/index.ts` and the `./event`
   entry are the contract. A removed or renamed export, a changed signature,
   return type or default zone is a breaking change: it needs a `!`
   Conventional Commit (`CONTRIBUTING.md` § Breaking Changes). v0.4.0 also
   documented its break in a `Breaking Changes in v0.4.0` section of
   `README.md` and `README.ko.md`; ask for the same when a break lands. New
   methods follow the suffix convention in `CONTRIBUTING.md` § Method Naming
   Conventions: `*Date` returns a JS `Date`, `*String` a string, and a bare
   name that returns a date returns a `Dayjs`. Predicates (`is*`) and numeric
   getters (`diff`, `toUnix*`) are bare by design; `epoch()` and
   `stripTimezoneToUTC()` are existing exceptions, not precedent.
3. **Build and types.** `exports` in `package.json` keeps both `import` and
   `require` conditions with matching `.d.ts` and `.d.cts` files for each
   entry in `tsup.config.ts`. dayjs stays a peer dependency and `external` in
   tsup; the package has no runtime dependencies. `sideEffects: false` and
   `splitting: false` stay. Plugins are registered with `dayjs.extend` once at
   import. Flag new plugins or dependencies that grow the bundle without need.
4. **Immutability and statics.** Methods return new values and never mutate
   their inputs; the class holds no instance or global state beyond plugin
   registration.
5. **Tests.** New cases go in `src/DayjsUtil.spec.ts`, in the `describe`
   block named after the method (add one for a new method), nested under
   `describe(DayjsUtil.name)` or `describe(EventDateHandler.name)`. Use
   explicit IANA zones and cover UTC midnight boundaries, DST transitions and
   `null` or `undefined` input. Tests run with `TZ=UTC` (`vitest.config.ts`).
6. **Documentation.** `README.md` and `README.ko.md` track any API change,
   including the API reference and supported formats. Public methods keep
   JSDoc with `@param`, `@returns` and `@example`.
7. **Security.** Workflow permissions, secret exposure and changes to the OIDC
   publish path in `.github/workflows/publish-npm.yml`.

## Do not flag

- The static-class design is intentional; do not suggest instances or
  free-function rewrites.
- The project has no external linter or formatter (`CONTRIBUTING.md` § Code
  Style; pre-commit only fixes whitespace and file endings); do not ask for
  one.
- Tests live in one spec file on purpose; do not ask to split it.
- Do not review the `EventDateHandler` API design afresh: changes to it should
  be discussed in an issue first (`CONTRIBUTING.md` § EventDateHandler, PR
  template). If the PR links no issue, say so; still review correctness.

## Verification

CI (`.github/workflows/ci.yml`, pull requests to `main`, Node 24, pnpm 10)
runs `pnpm install --frozen-lockfile`, `pnpm test` and `pnpm typecheck`;
`publish-npm.yml` repeats them on release before `pnpm build` and
`npm publish --access public --provenance`. The pre-commit hook runs
typecheck and test on `.ts` changes. Name the gates the change needs:

| Area changed                                        | Gate                                        |
| --------------------------------------------------- | ------------------------------------------- |
| `src/*.ts`                                          | `pnpm test` and `pnpm typecheck`            |
| `package.json` exports or `files`, `tsup.config.ts` | `pnpm build` (as `publish-npm.yml` runs it) |
| Dependencies or `pnpm-lock.yaml`                    | `pnpm install --frozen-lockfile`            |
| Coverage of a new path                              | `pnpm test:coverage`                        |
