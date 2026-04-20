## Overview
Build a small browser-focused keyboard input module at `src/input/keyboard.ts` that tracks currently held keys via `keydown` and `keyup` listeners and exposes a `snapshot()` API for the game loop to call once per frame. The MVP scope is deliberately narrow: provide a stable, testable source of raw keyboard state with no debouncing, throttling, or gameplay policy so later movement and firing systems can consume immediate per-frame input.

## Architecture
The module is a thin adapter between DOM keyboard events and game-loop reads. `bindKeyboard()` attaches `keydown`, `keyup`, and `blur` listeners to a `Window` target and updates an internal `Set<string>` keyed by `KeyboardEvent.code`. The returned handle exposes a live `state` object for inspection, while the exported `snapshot(input)` function clones the current pressed-key set into an immutable frame-local value so the caller can sample once per tick and use that same value throughout simulation. `dispose()` removes listeners and clears state so tests and future scene teardown do not leak input handlers. The companion test file drives synthetic `KeyboardEvent`s through the configured window target and verifies transitions, blur reset, and snapshot stability.

## User experience
Public API from `src/input/keyboard.ts`:

```ts
export interface InputState {
  readonly pressed: ReadonlySet<string>;
}

export interface KeyboardInputOptions {
  readonly target?: Window;
  readonly resetOnBlur?: boolean;
}

export interface KeyboardInput {
  readonly state: InputState;
  dispose(): void;
}

export function bindKeyboard(options?: KeyboardInputOptions): KeyboardInput;
export function snapshot(input: KeyboardInput): InputState;
```

Public names are `InputState`, `KeyboardInputOptions`, `KeyboardInput`, `bindKeyboard`, and `snapshot`. Internal helpers such as the mutable `Set`, event handler functions, and snapshot-cloning utilities are not part of the public contract and may change without a major version bump.

Realistic usage:

```ts
import { bindKeyboard } from "./input/keyboard";

const keyboard = bindKeyboard();
```

```ts
import { snapshot } from "./input/keyboard";

const input = snapshot(keyboard);

const moveLeft = input.pressed.has("ArrowLeft") || input.pressed.has("KeyA");
const moveRight = input.pressed.has("ArrowRight") || input.pressed.has("KeyD");
const fire = input.pressed.has("Space");
```

```ts
const keyboard = bindKeyboard({ target: window, resetOnBlur: true });

// Later, when the current scene or test ends:
keyboard.dispose();
```

Error and result behavior:

- `bindKeyboard()` returns a `KeyboardInput` handle immediately when given a usable browser `Window`.
- `bindKeyboard()` throws a `TypeError` if no usable `Window` target is available, because the module depends on DOM keyboard events.
- `snapshot(input)` never debounces, throttles, or filters repeats; it returns the current held-key set exactly as captured by listeners at the time of the call.
- `dispose()` is idempotent and returns `void`; after disposal, future `snapshot(input)` calls read as empty because the internal set is cleared.

Side effects:

- `bindKeyboard()` registers `keydown`, `keyup`, and optionally `blur` listeners on the target window.
- `dispose()` unregisters those listeners.
- The module has no network, filesystem, timer, or import-time side effects.

## File tree
Planned implementation files:

```text
src/
  input/
    keyboard.ts
    keyboard.test.ts
```

`keyboard.ts` is the production module. `keyboard.test.ts` is the unit test coverage for listener-driven state transitions and per-frame snapshots.

## Dependencies
No new runtime dependencies are required. The implementation should use:

- Platform browser APIs: `Window`, `KeyboardEvent`, `Set`, and `ReadonlySet`
- The existing TypeScript toolchain defined in `CONSTITUTION.md`
- The existing `pnpm vitest run` test runner for unit tests

If the repository already standardizes a DOM-backed Vitest environment, reuse it for `keyboard.test.ts`; the input module itself should not introduce a gameplay or engine dependency.

## Data model
The module centers on two small shapes:

```ts
type KeyCode = string;

interface InputState {
  readonly pressed: ReadonlySet<KeyCode>;
}

interface KeyboardInput {
  readonly state: InputState;
  dispose(): void;
}
```

Internal data flow:

- `pressedKeys: Set<KeyCode>` is the mutable source of truth updated by DOM events.
- `state.pressed` reflects the live held-key set for immediate inspection.
- `snapshot(input)` returns a fresh `InputState` whose `pressed` set will not be mutated by later events in the same frame.
- `resetOnBlur` controls whether a window blur clears the live set to prevent stuck keys when the tab loses focus.

## Implementation phases
1. Create `src/input/keyboard.ts` with the exported interfaces and `bindKeyboard()` entrypoint, defaulting the target to `window` in browser contexts.
2. Implement listener registration and the internal `Set<string>` store so `keydown` adds `event.code`, `keyup` removes it, repeated keydown events stay idempotent, and optional blur handling clears the set.
3. Implement `snapshot(input)` as a cheap clone of the current held-key set and `dispose()` as listener cleanup plus state reset.
4. Add `src/input/keyboard.test.ts` covering `keydown` -> pressed, `keyup` -> released, blur reset when enabled, snapshot immutability across later events, and disposal cleanup.
5. Validate with the project-standard commands from the constitution: `pnpm vitest run`, `pnpm eslint .`, and `pnpm tsc --noEmit`.

## Acceptance criteria
- `src/input/keyboard.ts` exists and exports `InputState`, `KeyboardInputOptions`, `KeyboardInput`, `bindKeyboard`, and `snapshot`.
- `bindKeyboard()` attaches keyboard listeners to a browser `Window` target without using debouncing, throttling, polling timers, or frame scheduling.
- A game-loop caller can invoke `snapshot(input)` once per frame and receive a stable `InputState` value representing the currently held keys for that frame.
- `keydown` adds `KeyboardEvent.code` values to the held-key set, `keyup` removes them, and repeated `keydown` events do not create duplicate state.
- When `resetOnBlur` is enabled, losing window focus clears held keys so movement or firing cannot get stuck.
- `dispose()` removes listeners, clears state, and can be called safely more than once.
- `src/input/keyboard.test.ts` verifies the state transitions above using `pnpm vitest run`.
- The implementation stays within the constitution boundaries: TypeScript, no `any`, no engine dependency, and no new runtime dependency.

## Open questions
- Should the package expose `src/input/keyboard.ts` directly as a public subpath, or should a root barrel/export map re-export it once the package surface is defined?
- Which DOM environment is the project standard for Vitest keyboard-event tests if one is not already configured?
- Should the first gameplay consumer treat only `ArrowLeft` / `ArrowRight` / `Space` as canonical bindings, or should `KeyA` / `KeyD` be documented as equivalent defaults in a follow-up controls layer?
