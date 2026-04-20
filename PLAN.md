## Overview
Build a strict TypeScript Space Invaders MVP on the constitution stack: `pnpm` for package management, Vite for the browser bundle, Vitest for game-logic tests, ESLint for static checks, and a raw HTML canvas entrypoint for play. The implementation should produce a static bundle that runs from `file://` or any static host, expose a small public TypeScript API for embedding the game into a page, and keep gameplay logic deterministic and testable by separating the pure simulation from rendering, audio, input, and asset-loading adapters.

## Architecture
The project should be split into a thin browser shell and a pure core. `src/main.ts` mounts the default browser experience into `index.html`, while `src/index.ts` exports the public library API. The public factory creates a game instance around a provided canvas element, validates configuration, loads bundled assets, wires keyboard input, then starts a fixed-timestep simulation loop backed by `requestAnimationFrame`.

Core data should flow in one direction each frame: keyboard state is sampled into an input snapshot, the simulation updates the immutable-ish game state using a fixed delta, the simulation emits domain events, and adapter layers consume that state to render sprites, update HUD text, and trigger Web Audio playback. Rendering stays on raw Canvas 2D for the first MVP cut, with renderer boundaries left clean enough to swap or extend later if a WebGL path is needed. Asset loading should use Vite-bundled local URLs rather than remote fetches so the built game remains compatible with `file://`.

## User experience
Public API exports should be limited to `src/index.ts`; everything else is internal and may change without a major version bump.

Proposed public exports:

```ts
export type SpaceInvadersStatus =
  | "idle"
  | "loading"
  | "running"
  | "game-over"
  | "destroyed";

export type AssetId =
  | "player"
  | "invader-a"
  | "invader-b"
  | "invader-c"
  | "projectile"
  | "shoot"
  | "hit"
  | "explosion"
  | "bg-loop";

export interface SpaceInvadersAssets {
  readonly sprites?: Partial<Record<Exclude<AssetId, "shoot" | "hit" | "explosion" | "bg-loop">, string>>;
  readonly audio?: Partial<Record<Extract<AssetId, "shoot" | "hit" | "explosion" | "bg-loop">, string>>;
}

export interface SpaceInvadersOptions {
  readonly canvas: HTMLCanvasElement;
  readonly width?: number;
  readonly height?: number;
  readonly pixelRatio?: number;
  readonly autoStart?: boolean;
  readonly muted?: boolean;
  readonly assetBaseUrl?: string;
  readonly assets?: SpaceInvadersAssets;
  readonly onEvent?: (event: SpaceInvadersEvent) => void;
}

export interface SpaceInvadersHandle {
  start(): Promise<void>;
  restart(): void;
  resize(width: number, height: number, pixelRatio?: number): void;
  setMuted(muted: boolean): void;
  getSnapshot(): Readonly<GameSnapshot>;
  destroy(): void;
}

export interface GameSnapshot {
  readonly status: SpaceInvadersStatus;
  readonly score: number;
  readonly wave: number;
  readonly lives: number;
  readonly enemiesRemaining: number;
}

export type SpaceInvadersEvent =
  | { readonly type: "ready" }
  | { readonly type: "score-changed"; readonly score: number }
  | { readonly type: "wave-started"; readonly wave: number }
  | { readonly type: "player-hit"; readonly livesRemaining: number }
  | { readonly type: "game-over"; readonly finalScore: number }
  | { readonly type: "warning"; readonly error: SpaceInvadersError };

export type SpaceInvadersError =
  | {
      readonly kind: "invalid-config";
      readonly message: string;
      readonly field: keyof SpaceInvadersOptions;
    }
  | {
      readonly kind: "unsupported-browser";
      readonly message: string;
      readonly feature: "canvas-2d" | "webaudio";
    }
  | {
      readonly kind: "asset-load-failed";
      readonly message: string;
      readonly assetId: AssetId;
    };

export declare function createSpaceInvaders(
  options: SpaceInvadersOptions,
): Promise<SpaceInvadersHandle>;

export declare function isSpaceInvadersError(
  value: unknown,
): value is SpaceInvadersError;
```

Usage snippet: basic browser mount.

```ts
import { createSpaceInvaders } from "space-invaders";

const canvas = document.querySelector("canvas");

if (!(canvas instanceof HTMLCanvasElement)) {
  throw new Error("Canvas element not found");
}

const game = await createSpaceInvaders({ canvas, autoStart: true });
```

Usage snippet: host-controlled restart and HUD sync.

```ts
import { createSpaceInvaders } from "space-invaders";

const game = await createSpaceInvaders({
  canvas,
  onEvent(event) {
    if (event.type === "score-changed") scoreNode.textContent = String(event.score);
    if (event.type === "game-over") overlay.hidden = false;
  },
});

restartButton.addEventListener("click", () => {
  overlay.hidden = true;
  game.restart();
});
```

Usage snippet: local asset overrides with muted audio.

```ts
import { createSpaceInvaders } from "space-invaders";

await createSpaceInvaders({
  canvas,
  muted: true,
  assetBaseUrl: "./assets",
  assets: {
    sprites: { player: "./assets/player.png" },
    audio: { shoot: "./assets/shoot.wav" },
  },
});
```

Invalid configuration should reject `createSpaceInvaders` with `invalid-config`. Unsupported platform features should reject startup with `unsupported-browser`. Asset failures should reject in development; in production they should emit a `warning` event and fall back to playable defaults where possible, such as silent audio or generated placeholder sprites. Side effects must be explicit: the library owns a `requestAnimationFrame` loop, attaches keyboard listeners, reads bundled asset URLs, draws into the provided canvas, and creates browser audio resources. No network, filesystem, backend, analytics, or global persistence side effects are in MVP.

## File tree
```text
.
├── ASSETS.md
├── index.html
├── package.json
├── pnpm-lock.yaml
├── tsconfig.json
├── tsconfig.node.json
├── vite.config.ts
├── eslint.config.js
├── public/
│   └── favicon.svg
├── src/
│   ├── index.ts
│   ├── main.ts
│   ├── game/
│   │   ├── createSpaceInvaders.ts
│   │   ├── gameLoop.ts
│   │   ├── types.ts
│   │   └── errors.ts
│   ├── core/
│   │   ├── initialState.ts
│   │   ├── updateSimulation.ts
│   │   ├── collision.ts
│   │   ├── waveProgression.ts
│   │   └── scoring.ts
│   ├── input/
│   │   └── keyboard.ts
│   ├── render/
│   │   ├── canvasRenderer.ts
│   │   └── spriteAtlas.ts
│   ├── audio/
│   │   └── webAudio.ts
│   ├── assets/
│   │   ├── manifest.ts
│   │   └── placeholders.ts
│   └── ui/
│       └── hud.ts
└── tests/
    ├── collision.test.ts
    ├── waveProgression.test.ts
    ├── scoring.test.ts
    └── createSpaceInvaders.test.ts
```

## Dependencies
- Runtime dependencies: none beyond the browser platform APIs already allowed by the constitution (`CanvasRenderingContext2D`, `requestAnimationFrame`, `KeyboardEvent`, `AudioContext`).
- Dev dependencies: `typescript`, `vite`, `vitest`, `eslint`, `@typescript-eslint/parser`, `@typescript-eslint/eslint-plugin`, `@types/node`.
- Package manager and commands must remain the constitution values: `pnpm install`, `pnpm vitest run`, `pnpm eslint .`, `pnpm tsc --noEmit`, `pnpm build`.
- No game engine or other runtime library should be introduced unless the constitution is amended first.

## Data model
- `GameState`: `{ status, score, wave, lives, tick, player, invaders, projectiles, cooldowns, rngSeed }`.
- `PlayerState`: `{ x, y, width, height, speed, alive, shotCooldownMs }`.
- `InvaderState`: `{ id, row, column, x, y, width, height, alive, sprite: "invader-a" | "invader-b" | "invader-c" }`.
- `ProjectileState`: `{ id, owner: "player" | "invader", x, y, width, height, velocityY, active }`.
- `WaveState`: `{ index, direction: -1 | 1, horizontalSpeed, descentStep, stepIntervalMs, enemiesRemaining }`.
- `InputState`: `{ leftPressed, rightPressed, firePressed }`.
- `AssetManifest`: maps sprite and audio IDs to bundled local URLs plus metadata needed by loaders.
- `SimulationResult`: `{ state: GameState, events: readonly SpaceInvadersEvent[] }`.
- `SpaceInvadersError`: discriminated union for startup and asset failures; no untyped throw payloads.

## Implementation phases
1. Scaffold the toolchain and browser shell: add `package.json`, `pnpm-lock.yaml`, TypeScript config, Vite config, ESLint config, `index.html`, and a minimal `src/main.ts`/`src/index.ts` split that proves the canvas entrypoint and exported API shape.
2. Define the public and internal contracts: add the exported types, error union, game handle interface, asset manifest types, and pure state model so later work has stable boundaries.
3. Implement deterministic core simulation: fixed-timestep update loop, keyboard input sampling, player movement, projectile spawning, invader marching, collision detection, scoring, wave progression, and game-over transitions in pure modules covered by Vitest.
4. Add platform adapters: canvas renderer for sprites and HUD, asset loader with placeholder fallback behavior, and Web Audio playback for shoot/hit/explosion/background loop behind explicit browser capability checks.
5. Wire the playable browser flow: startup, resize behavior, restart flow, score display, game-over overlay/state, and event emission so the default Vite page and external consumers both exercise the same API.
6. Harden and verify: populate `ASSETS.md`, add tests for factory validation and core game rules, run `pnpm vitest run`, `pnpm eslint .`, `pnpm tsc --noEmit`, and `pnpm build`, then perform a manual `file://` smoke test and dev HUD frame-timing check.

## Acceptance criteria
- `pnpm install`, `pnpm vitest run`, `pnpm eslint .`, `pnpm tsc --noEmit`, and `pnpm build` all succeed on the committed scaffold and MVP implementation.
- Opening the built output in a modern browser from `file://` or a static host shows a working canvas-based Space Invaders game with keyboard movement, shooting, collisions, score updates, wave advancement, game over, and restart.
- The exported TypeScript API is limited to the planned public surface, has no `any` types, and cleanly reports invalid config, unsupported-browser, and asset-load errors.
- Game-logic tests cover collision outcomes, scoring changes, and wave progression, proving the deterministic core independently of rendering.
- Rendering uses raw Canvas rather than a third-party engine, audio uses browser APIs only, and the runtime dependency list remains empty.
- Licensed asset provenance is documented in `ASSETS.md`, and missing assets degrade gracefully enough to keep the game playable in production builds.

## Open questions
- Is Canvas 2D alone acceptable for MVP, with renderer abstraction kept ready for a later WebGL path, or is a dual Canvas/WebGL implementation expected immediately?
- What concrete sprite and audio assets are available today, and can placeholder art/audio ship in the first playable cut as long as `ASSETS.md` records provenance?
- What viewport and aspect ratio should define the default playfield for the browser shell: original arcade proportions, a fixed 4:3 canvas, or a wider modern presentation?
- Should audio remain muted until the first user gesture to satisfy browser autoplay rules, or is an explicit start button preferred for the default host page?
