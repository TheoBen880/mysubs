# Build, Test, and Release

How the `mysubs` npm package is developed, verified, built, and published. All
commands and pipeline steps below come from `package.json`, `tsdown.config.ts`,
`vitest.config.mts`, the husky hook, and `.github/workflows/publish.yml`.

## Tooling

- **Language** — TypeScript with `strict` and `noUncheckedIndexedAccess`,
  targeting ES2022 (`tsconfig.json`). The published module is ESM.
- **Package manager** — pnpm (a workspace with `allowBuilds` for esbuild);
  CI enables corepack and installs with a frozen lockfile. Node 24 in CI.
- **Runtime dependencies** — only `@napi-rs/keyring` for the OS keyring;
  everything else (commander, chalk, zod, ms) is bundled at build time.
- **Quality gates** — ESLint (flat config with typescript-eslint, import,
  prettier, and check-file plugins) plus Prettier; a husky pre-commit hook runs
  `tsc --noEmit` and lint-staged, which fixes staged sources with ESLint
  (max 5 warnings) and formats staged JSON/Markdown/YAML.

## npm scripts

| Script | Command | Purpose |
| --- | --- | --- |
| `lint` / `lint:fix` | `eslint .` | Lint (and autofix) the repository. |
| `typecheck` / `typecheck:watch` | `tsc --noEmit` | Type-check without emitting. |
| `build` | `tsdown --config ./tsdown.config.ts` | Bundle `src/index.ts` to `dist/index.mjs` (ESM, target ES6, dead-code-eliminated, no declarations or sourcemaps). |
| `dev` | `tsdown --watch` | Rebuild on change. |
| `start` | `tsx ./src/index.ts` | Run the CLI from source. |
| `test` / `test:watch` | `vitest run` / `vitest` | Unit tests. |

The `bin` field maps the `mysubs` command to `dist/index.mjs`.

## Test coverage

Vitest includes every `src/**/*.test.ts` file and forces `NO_COLOR=1`. The
suites cover:

- Config loading and account key handling (`src/core/config.test.ts`), using a
  temporary `XDG_CONFIG_HOME`.
- Usage bar color computation (`src/lib/color.test.ts`) and secret reference
  resolution including env fallbacks (`src/lib/secret.test.ts`).
- Provider option defaults, especially detection toggles
  (`src/providers/config.test.ts`).
- Antigravity quota parsing and bucket ordering
  (`src/providers/antigravity/index.test.ts`).
- Copilot usage mapping against stubbed global fetch and env
  (`src/providers/copilot/index.test.ts`).
- OpenCode Go usage mapping against stubbed fetch
  (`src/providers/opencode/index.test.ts`).
- Cache TTL parsing (`src/utils/cache.test.ts`), money/percent/duration
  formatting (`src/utils/format.test.ts`), home expansion
  (`src/utils/path.test.ts`), and the progress spinner (`src/utils/progress.test.ts`).

## Release pipeline

1. A maintainer publishes a GitHub release; the `Publish` workflow triggers on
   `release: published`.
2. The workflow checks out the default branch, bumps `package.json` to the
   release tag with `version-patch`, commits it as
   `chore(release): v<version>`, runs `pnpm build`, publishes to npm with the
   registry token, and pushes the commit.
3. A follow-up `update-schema` job runs `pnpm exec tsx ./src/json.ts`, which
   generates `schema.json` — the JSON Schema for the config file including
   every provider's account and options schemas — and force-pushes it to the
   `schema` branch. That branch is what the `$schema` pointer in user configs
   references.
4. `schema.json` is git-ignored; it only ever exists on the `schema` branch.
