## Overview
Stand up the test and lint toolchain required by the constitution's Stack so that `pnpm vitest run` and `pnpm eslint .` pass cleanly before any gameplay code is written. This plan adds a Vitest configuration wired for a browser-like DOM environment, a flat ESLint config using `typescript-eslint`'s recommended-type-checked rules, and a single smoke test that proves the harness executes TypeScript tests. The result is a CI-ready foundation that enforces the "Fail loud in dev" principle from day one and leaves all gameplay decisions from the earlier project plan untouched.

## Architecture
Three cooperating config surfaces live at the repo root and are consumed by their respective CLIs when a contributor runs the commands declared in `CONSTITUTION.md`.

- `vitest.config.ts` is read by the Vitest runner. It selects a DOM-capable test environment (`jsdom`) so future game-logic tests that touch `HTMLCanvasElement`, `KeyboardEvent`, or `AudioContext` typing have the expected globals. It sets `include` to match `src/**/*.test.ts` and `tests/**/*.test.ts`, and enables TypeScript source imports without a build step.
- `eslint.config.js` is read by ESLint v9's flat config loader. It composes `@eslint/js` recommended rules with `typescript-eslint`'s `recommendedTypeChecked` preset, scoped to `**/*.ts` and `**/*.tsx`. It ignores build artifacts (`dist/`, `coverage/`, `node_modules/`) and config files that don't need type-aware linting. Parser options point at `tsconfig.json` so rules that need type information work.
- `tsconfig.json` continues to own strict TypeScript, and is referenced by both configs via `projectService` (ESLint) and Vitest's built-in TS loader. No duplicate type settings live in the lint or test config.

Data flow per contributor command:

1. `pnpm vitest run` → Vitest loads `vitest.config.ts` → discovers `src/smoke.test.ts` → runs under jsdom → reports pass/fail.
2. `pnpm eslint .` → ESLint loads `eslint.config.js` → walks `src/` and `tests/` → reports zero errors on the committed baseline.
3. `pnpm tsc --noEmit` continues to enforce strict typing across test and config files where applicable (test files are picked up via `include` in `tsconfig.json`).

## User experience
This proposal is consumed by contributors, not library end users. The "public surface" is the set of root-level config files plus the commands in `CONSTITUTION.md`. Gameplay library exports remain as defined in the broader project plan; this task does not add or remove any library exports.

Configuration-level types and signatures contributors interact with:

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "jsdom",
    include: ["src/**/*.test.ts", "tests/**/*.test.ts"],
    globals: false,
    reporters: "default",
  },
});
```

```js
// eslint.config.js (flat config)
import js from "@eslint/js";
import tseslint from "typescript-eslint";

export default tseslint.config(
  { ignores: ["dist/**", "coverage/**", "node_modules/**"] },
  js.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  {
    files: ["**/*.ts"],
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
  },
);
```

```ts
// src/smoke.test.ts
import { describe, it, expect } from "vitest";

describe("smoke", () => {
  it("runs arithmetic under the test harness", () => {
    expect(1 + 1).toBe(2);
  });
});
```

Realistic contributor usage — first-time setup:

```sh
pnpm install
pnpm vitest run    # expects: 1 passed (src/smoke.test.ts)
pnpm eslint .      # expects: zero errors, zero warnings
pnpm tsc --noEmit  # still passes once src/ has real code
```

Realistic contributor usage — adding a new game-logic test later:

```ts
// src/core/collision.test.ts
import { describe, it, expect } from "vitest";
import { collides } from "./collision";

describe("collides", () => {
  it("detects overlapping AABBs", () => {
    expect(collides({ x: 0, y: 0, w: 10, h: 10 }, { x: 5, y: 5, w: 10, h: 10 })).toBe(true);
  });
});
```

Realistic contributor usage — running a single test file:

```sh
pnpm vitest run src/smoke.test.ts
```

Error/result shape:

- `pnpm vitest run` exits non-zero on any failing `expect` or uncaught error, printing the failed assertion and stack trace.
- `pnpm eslint .` exits non-zero if any rule reports an error; warnings do not fail the command unless `--max-warnings 0` is added later.
- Invalid config shapes surface as Vitest or ESLint startup errors (module load failure) rather than silent misbehavior — matching "Fail loud in dev".

Internal vs public: none of the files added by this task are importable library APIs. They are toolchain configuration and an internal smoke test. They may change without a semver bump.

Side effects: ESLint and Vitest both read files from the working tree; neither touches the network, and no new runtime side effects are introduced into shipped code.

## File tree
Files this plan creates or modifies:

```text
.
├── package.json          (modify — add devDependencies + scripts if missing)
├── vitest.config.ts      (create)
├── eslint.config.js      (create)
├── tsconfig.json         (modify only if needed to include src/**/*.test.ts)
└── src/
    └── smoke.test.ts     (create)
```

No other files are touched by this task. The gameplay source files listed in the broader project plan (`src/index.ts`, `src/main.ts`, `src/core/**`, etc.) are out of scope here and will be added by later tasks.

## Dependencies
All new additions go into `devDependencies` only. No runtime dependencies are introduced, preserving the constitution's Boundaries.

- `typescript` — required by ESLint type-aware rules and by `pnpm tsc --noEmit`.
- `vitest` — test runner matching the constitution's declared `pnpm vitest run` command.
- `jsdom` — DOM environment used by Vitest so tests can reference `HTMLCanvasElement`, `KeyboardEvent`, and similar browser types without importing a full browser.
- `eslint` (v9+) — flat-config-capable ESLint matching `pnpm eslint .`.
- `typescript-eslint` (meta package exposing `tseslint.config`, parser, and plugin) — provides `recommendedTypeChecked`.
- `@eslint/js` — provides `js.configs.recommended` for the flat config composition.
- `@types/node` — types for `import.meta.dirname` in `eslint.config.js` and for any Node-ish test helpers.

Scripts in `package.json` should mirror the constitution's commands so `pnpm test`, `pnpm lint`, and `pnpm typecheck` are idiomatic shortcuts to `vitest run`, `eslint .`, and `tsc --noEmit` respectively. The constitution's exact commands (`pnpm vitest run`, `pnpm eslint .`, `pnpm tsc --noEmit`, `pnpm build`) remain the canonical ones.

## Data model
This task introduces configuration shapes rather than runtime types, but the relevant interfaces are:

- `VitestUserConfig` (from `vitest/config`):
  - `test.environment: "jsdom"`
  - `test.include: readonly string[]` — `["src/**/*.test.ts", "tests/**/*.test.ts"]`
  - `test.globals: false` — tests must import `describe`/`it`/`expect` from `vitest` to keep surface explicit and typed.
  - `test.reporters: "default"`
- `FlatESLintConfig` (from `typescript-eslint`):
  - `ignores: readonly string[]` — `dist/**`, `coverage/**`, `node_modules/**`.
  - `files: readonly string[]` — `**/*.ts`.
  - `languageOptions.parserOptions`: `{ projectService: true, tsconfigRootDir: string }`.
  - Composition: `js.configs.recommended` ∘ `tseslint.configs.recommendedTypeChecked`.
- `SmokeTest`: a single `describe` block with one `it` asserting `1 + 1 === 2` via `expect().toBe()`.
- `PackageJsonDevDeps`: a `Record<string, string>` additions listed in Dependencies.

No new runtime types are added. Game-logic types (`GameState`, `PlayerState`, etc.) stay out of this task's scope.

## Implementation phases
Ordered smallest-to-largest; each phase leaves the repo in a runnable state.

1. **Add devDependencies and scripts.** Update `package.json` to declare `typescript`, `vitest`, `jsdom`, `eslint`, `typescript-eslint`, `@eslint/js`, and `@types/node` under `devDependencies`. Add `test`, `lint`, `typecheck` script aliases matching the constitution's commands. Run `pnpm install` to lock versions.
2. **Write the smoke test.** Create `src/smoke.test.ts` with a single `describe`/`it` that asserts `expect(1 + 1).toBe(2)`. Imports come from `vitest` directly so `globals: false` works.
3. **Author `vitest.config.ts`.** Use `defineConfig` from `vitest/config`, select `jsdom`, set `include` to `src/**/*.test.ts` and `tests/**/*.test.ts`, keep `globals: false`. Confirm `pnpm vitest run` finds and passes the smoke test.
4. **Author `eslint.config.js`.** Use the `typescript-eslint` flat-config helper to compose `@eslint/js` recommended with `tseslint.configs.recommendedTypeChecked`. Ignore `dist/`, `coverage/`, `node_modules/`. Point `parserOptions.projectService` at the repo's `tsconfig.json`. Confirm `pnpm eslint .` reports zero errors on the smoke test and the config files themselves.
5. **Ensure `tsconfig.json` covers tests.** Only modify if the current `include` doesn't already pick up `src/**/*.test.ts`; otherwise leave it alone. Confirm `pnpm tsc --noEmit` still passes.
6. **Verify the full gate locally.** Run `pnpm install`, `pnpm vitest run`, `pnpm eslint .`, and `pnpm tsc --noEmit` in sequence; all four must exit zero before the task is complete.

## Acceptance criteria
A reviewer can accept this MVP when all of the following hold:

- `pnpm install` succeeds from a clean clone using the committed `pnpm-lock.yaml`.
- `pnpm vitest run` exits zero and reports exactly one passing test from `src/smoke.test.ts` asserting `1 + 1 === 2`.
- `pnpm eslint .` exits zero with no errors and no warnings on the committed files.
- `pnpm tsc --noEmit` continues to exit zero with `strict: true`.
- `vitest.config.ts` exists at the repo root, uses `defineConfig` from `vitest/config`, and selects a DOM-capable environment (`jsdom`).
- `eslint.config.js` exists at the repo root as a flat config and composes `@eslint/js` recommended with `typescript-eslint`'s `recommendedTypeChecked` preset.
- `package.json` lists `vitest`, `eslint`, `typescript-eslint`, `@eslint/js`, `jsdom`, `@types/node`, and `typescript` under `devDependencies` only (no runtime deps added).
- No `any` types and no TypeScript or ESLint suppression comments are introduced anywhere in the added files.
- Gameplay source files and their public API are untouched by this task.

## Open questions
- Should the Vitest environment be `jsdom` or `happy-dom`? `jsdom` is more compatible with browser-grade DOM APIs the game will eventually exercise, but `happy-dom` is faster; the proposal allows either and the choice affects CI wall time.
- Should `pnpm lint` use `--max-warnings 0` immediately, or wait until after gameplay code lands and baseline noise is understood?
- Is `recommendedTypeChecked` the right starting preset, or should the project start with the non-type-checked `recommended` and layer type-aware rules later to minimize initial lint load?
- Should coverage (`@vitest/coverage-v8`) be wired up as part of this initial toolchain task, or deferred until gameplay tests exist and a meaningful coverage baseline can be set?
- Does the repo need a `tests/` top-level directory today, or should all tests co-locate next to source under `src/**` until a second test location is actually required?
- What minimum Node.js version should contributors target? ESLint v9 flat config and `typescript-eslint` v8 both require Node 18.18+, which should be pinned in `package.json` `engines` if adopted.
