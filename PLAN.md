## Overview
Scaffold the actual greenfield project skeleton described by proposal `#110`: `pnpm` + strict TypeScript + Vite + Vitest + ESLint + a bootable HTML canvas entrypoint. The deliverable is a repo where `pnpm install`, `pnpm vitest run`, `pnpm eslint .`, `pnpm tsc --noEmit`, and `pnpm build` all pass, and the built shell opens from `file://` or any static host without network access. This plan stops at the executable shell and toolchain; gameplay systems, assets, audio playback, and simulation APIs are intentionally deferred to later tasks.

## Architecture
The repo is greenfield, so the skeleton must create every root config file it depends on rather than assuming they already exist.

- `package.json` is created at the repo root with `private: true`, `type: "module"`, a pinned `packageManager`, `engines.node`, and scripts for `dev`, `build`, `test`, `lint`, and `typecheck`. `pnpm install` produces and commits `pnpm-lock.yaml` so clean-clone installs are reproducible.
- `tsconfig.json` is created with `strict: true`, DOM libraries, bundler-friendly module settings, and a hard `include` that covers `src/**/*.ts`, `src/**/*.test.ts`, `vite.config.ts`, and `vitest.config.ts`. This keeps `typescript-eslint`'s type-aware rules and `pnpm tsc --noEmit` aligned instead of treating test coverage as conditional.
- `vite.config.ts` is the build/dev entrypoint. It sets `base: "./"` so the emitted `dist/index.html` uses relative asset URLs and remains compatible with `file://`, which is required by the constitution.
- `index.html` provides the minimal browser shell: a mount element, a visible `<canvas>`, and placeholder status text so contributors can verify the app boots before any gameplay exists.
- `src/bootstrap.ts` owns DOM-only setup for that shell. It locates the required elements, validates that the canvas exists, and marks the shell as ready. It does not call `canvas.getContext()` or Web Audio APIs; jsdom is sufficient for DOM wiring tests, but not for real canvas or audio execution.
- `src/main.ts` is the Vite entrypoint and simply calls `bootstrapShell()` in the browser. Keeping bootstrap logic in its own module makes the initial jsdom test exercise a real DOM contract instead of a meaningless arithmetic check.
- `vitest.config.ts` uses `jsdom` and includes `src/**/*.test.ts`. The first test must touch `document` and `HTMLCanvasElement` so a broken DOM environment fails loudly.
- `eslint.config.js` uses ESLint v9 flat config plus `typescript-eslint`'s `recommendedTypeChecked` preset for TypeScript files. It ignores build artifacts and `eslint.config.js` itself, while linting the TypeScript source, tests, and TS config files. `tsconfigRootDir` is computed with `fileURLToPath(import.meta.url)` rather than `import.meta.dirname` so the plan remains compatible with Node `18.18+`.

Command flow after the skeleton lands:

1. `pnpm install` reads `package.json`, respects the pinned `packageManager` and `engines`, and writes the committed `pnpm-lock.yaml`.
2. `pnpm vitest run` loads `vitest.config.ts`, discovers `src/bootstrap.test.ts`, runs under jsdom, and fails if DOM globals or the canvas shell contract are missing.
3. `pnpm eslint .` loads `eslint.config.js`, type-checks the included TypeScript files through `projectService`, and ignores generated output.
4. `pnpm tsc --noEmit` verifies the strict TypeScript project, including the test file and TS config files.
5. `pnpm build` loads `vite.config.ts`, uses `index.html` as the entry document, and emits a static `dist/` bundle with relative asset paths.

## User experience
This task is for contributors rather than end users. The user-visible output is a bootable placeholder shell, not a playable game yet.

Contributor-facing contracts:

```ts
// src/bootstrap.ts
export interface AppShell {
  readonly canvas: HTMLCanvasElement;
  readonly statusElement: HTMLElement;
}

export function bootstrapShell(doc?: Document): AppShell;
```

```ts
// src/bootstrap.test.ts
import { describe, expect, it } from "vitest";
import { bootstrapShell } from "./bootstrap";

describe("bootstrapShell", () => {
  it("mounts a canvas shell in a DOM environment", () => {
    document.body.innerHTML = `
      <main id="app">
        <canvas id="game-canvas" width="960" height="540"></canvas>
        <p id="boot-status"></p>
      </main>
    `;

    const shell = bootstrapShell(document);

    expect(shell.canvas).toBeInstanceOf(HTMLCanvasElement);
    expect(shell.statusElement.textContent).toContain("ready");
  });
});
```

Expected contributor workflow:

```sh
pnpm install
pnpm vitest run
pnpm eslint .
pnpm tsc --noEmit
pnpm build
```

Opening `dist/index.html` after `pnpm build` should show a visible canvas shell and placeholder ready text. No gameplay exports are promised yet; `bootstrapShell` exists as an internal seam to keep the DOM bootstrap testable and may change once the real game surface is introduced.

## File tree
Files this plan creates:

```text
.
├── package.json          (create — package metadata, scripts, pinned packageManager, engines)
├── pnpm-lock.yaml        (create/commit — locked dependency graph)
├── tsconfig.json         (create — strict TypeScript + DOM libs + includes for tests/configs)
├── vite.config.ts        (create — Vite config with relative base)
├── vitest.config.ts      (create — jsdom test config)
├── eslint.config.js      (create — flat ESLint config)
├── index.html            (create — visible canvas shell entry document)
└── src/
    ├── bootstrap.ts      (create — DOM shell bootstrap helper)
    ├── main.ts           (create — Vite browser entrypoint)
    └── bootstrap.test.ts (create — DOM-aware smoke test)
```

No gameplay modules, asset loaders, audio adapters, or simulation types are introduced by this task.

## Dependencies
All packages land in `devDependencies`. No runtime dependency is needed for the placeholder shell, preserving the constitution's boundary against unnecessary runtime additions.

- `typescript` — compiler for `pnpm tsc --noEmit` and source typing.
- `vite` — dev server and static bundler that implements `pnpm build`.
- `vitest` — test runner matching the constitution's `pnpm vitest run` command.
- `jsdom` — DOM environment for the bootstrap test; used for document/canvas element wiring only, not real canvas or audio execution.
- `eslint` — flat-config-capable ESLint matching `pnpm eslint .`.
- `typescript-eslint` — parser/plugin/config helper for type-aware lint rules.
- `@eslint/js` — base JavaScript recommended rules for ESLint flat config.
- `@types/node` — Node types for config files and `fileURLToPath`-based path resolution.

`package.json` scripts should map to:

- `dev`: `vite`
- `build`: `vite build`
- `test`: `vitest run`
- `lint`: `eslint .`
- `typecheck`: `tsc --noEmit`

The constitution's canonical commands remain `pnpm install`, `pnpm vitest run`, `pnpm eslint .`, `pnpm tsc --noEmit`, and `pnpm build`.

## Data model
This skeleton introduces config and bootstrap shapes, not gameplay state. The relevant contracts are:

- `PackageJson`:
  - `private: true`
  - `type: "module"`
  - `packageManager: "pnpm@<exact-version-used-to-create-lockfile>"`
  - `engines.node: ">=18.18.0"`
  - `scripts.dev | build | test | lint | typecheck`
- `TsConfig`:
  - `compilerOptions.strict: true`
  - `compilerOptions.lib: ["ES2022", "DOM", "DOM.Iterable"]`
  - `include: ["src/**/*.ts", "src/**/*.test.ts", "vite.config.ts", "vitest.config.ts"]`
- `ViteUserConfig`:
  - `base: "./"`
- `VitestUserConfig`:
  - `test.environment: "jsdom"`
  - `test.include: ["src/**/*.test.ts"]`
  - `test.globals: false`
- `AppShell`:
  - `canvas: HTMLCanvasElement`
  - `statusElement: HTMLElement`
- `FlatESLintConfig`:
  - `ignores: ["dist/**", "coverage/**", "node_modules/**", "eslint.config.js"]`
  - `files: ["**/*.ts"]`
  - `languageOptions.parserOptions.projectService: true`
  - `languageOptions.parserOptions.tsconfigRootDir: string` derived from `fileURLToPath(import.meta.url)`

No `GameState`, seeded RNG contract, resize API, asset-failure mode, or audio/rendering adapter API is part of this task's data model.

## Implementation phases
Ordered smallest-to-largest; each phase leaves the repo closer to the constitution's full command surface.

1. **Create package metadata and lockfile.** Create `package.json` from scratch with `private`, `type`, pinned `packageManager`, `engines.node >=18.18.0`, and `dev`/`build`/`test`/`lint`/`typecheck` scripts. Add the devDependencies listed above, run `pnpm install`, and commit the resulting `pnpm-lock.yaml`.
2. **Create strict TypeScript config.** Create `tsconfig.json` with `strict: true`, DOM libs, bundler module settings, and a non-optional include list covering `src/**/*.ts`, `src/**/*.test.ts`, `vite.config.ts`, and `vitest.config.ts`.
3. **Add the browser shell and build config.** Create `index.html`, `src/bootstrap.ts`, `src/main.ts`, and `vite.config.ts`. `vite.config.ts` must set `base: "./"`. The browser shell should show a visible canvas and placeholder ready text without depending on gameplay code or network-loaded assets.
4. **Add a DOM-aware test harness.** Create `vitest.config.ts` with `environment: "jsdom"` and `include: ["src/**/*.test.ts"]`. Create `src/bootstrap.test.ts` that interacts with `document`, validates `HTMLCanvasElement` usage, and asserts that `bootstrapShell()` marks the shell ready. Do not use a pure arithmetic smoke test.
5. **Add linting that matches the TypeScript project.** Create `eslint.config.js` with `@eslint/js` plus `typescript-eslint`'s `recommendedTypeChecked` preset. Ignore generated output and `eslint.config.js`, use `fileURLToPath(import.meta.url)` for `tsconfigRootDir`, and ensure the included TypeScript files lint cleanly under `projectService`.
6. **Verify the full skeleton locally.** Run `pnpm install`, `pnpm vitest run`, `pnpm eslint .`, `pnpm tsc --noEmit`, and `pnpm build`. Confirm `dist/index.html` references assets relatively and perform a manual `file://` smoke test that shows the placeholder canvas shell.

## Acceptance criteria
A reviewer can accept this skeleton when all of the following hold:

- `package.json`, `pnpm-lock.yaml`, `tsconfig.json`, `vite.config.ts`, `vitest.config.ts`, `eslint.config.js`, `index.html`, `src/bootstrap.ts`, `src/main.ts`, and `src/bootstrap.test.ts` exist.
- `package.json` is created with `private: true`, `type: "module"`, a pinned `packageManager`, `engines.node >=18.18.0`, and scripts for `dev`, `build`, `test`, `lint`, and `typecheck`.
- `pnpm install` succeeds from a clean clone using the committed `pnpm-lock.yaml`.
- `pnpm vitest run` exits zero and the initial test fails if DOM globals are missing because it asserts against `document` and `HTMLCanvasElement`, not pure arithmetic.
- `pnpm eslint .` exits zero with no errors or warnings on the committed files.
- `pnpm tsc --noEmit` exits zero with `strict: true`.
- `pnpm build` exits zero and emits `dist/index.html` with relative asset URLs suitable for `file://`.
- Opening `dist/index.html` from `file://` or a static host shows a visible canvas shell and placeholder ready text without requiring network access.
- The initial bootstrap logic and test do not rely on jsdom implementing real `CanvasRenderingContext2D`, WebGL, or Web Audio APIs; those concerns remain deferred behind later rendering/audio work.
- No runtime dependencies, `any` types, or TypeScript/ESLint suppression comments are introduced.

## Open questions
No blocking questions remain for this task. Follow-up work may later decide:

- when to introduce a top-level `tests/` directory instead of colocated `src/**/*.test.ts` files
- whether to add coverage reporting once there is meaningful gameplay logic to measure
- what browser-level strategy to use for future canvas/audio integration tests beyond this DOM-only shell

## Revision notes for round 1
- Addressed the repeated scope mismatch comments (`4284793430`, `4284793992`, `4284794536`, `4284801920`, `4284802482`, `4284802922`, `4284811933`, `4284812544`, `4284813026`, `4284814500`, `4284819876`, `4284820525`, `4284820981`, `4284823191`, `4284830088`, `4284830849`, `4284840002`, `4284840622`, `4284842133`, `4284842814`) by expanding the plan from a toolchain-only draft into the full greenfield skeleton: Vite, `pnpm build`, `index.html`, `src/main.ts`, and a bootable canvas shell are now explicitly in scope.
- Addressed the greenfield/bootstrap comments (`4284795149`, `4284795787`, `4284803625`, `4284805070`, `4284813942`, `4284821722`, `4284824378`, `4284833192`, `4284841430`) by making `package.json`, `pnpm-lock.yaml`, and `tsconfig.json` create-and-commit artifacts with explicit `packageManager`, `engines`, strict TypeScript settings, and a non-optional test/config include list.
- Addressed the DOM/test-harness comments (`4284730964`, `4284804315`, `4284822561`, `4284834802`) by keeping `jsdom`, replacing the arithmetic smoke test with a real DOM/canvas bootstrap test, hard-including `src/**/*.test.ts` in `tsconfig.json`, and clarifying that jsdom is only used for DOM shell wiring, not real canvas/audio execution.
- Addressed the config-consistency comments (`4284743313`, `4284805716`, `4284823853`, `4284831599`, `4284832377`, `4284834109`) by resolving the environment and lint-preset choices in the plan body, requiring `base: "./"` for Vite, switching away from the Node 20-only `import.meta.dirname` assumption, and reconciling the ESLint ignore/lint scope.
- Did not incorporate the gameplay-API comments (`4284731584`, `4284732113`, `4284733115`, `4284733721`, `4284734395`, `4284744081`, `4284744674`, `4284745468`, `4284746101`) into the body because they target an older draft centered on gameplay/public APIs (`resize`, RNG/public seed contract, asset-failure mode, status machine, Web Audio adapter behavior, dev HUD, fixed-timestep tests). This skeleton plan intentionally stops before those modules exist, so the feedback is recorded here as deferred rather than being silently dropped.
