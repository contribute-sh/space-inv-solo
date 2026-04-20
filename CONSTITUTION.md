# Constitution

Version: 1.0.0

## Purpose

A browser-playable Space Invaders game with modern hi-res graphics, animation, and sound effects. The target experience is an arcade-accurate modern remake focused on classic waves and high-score chasing, not an expanded campaign with story or progression systems. It runs fully client-side in a modern browser with zero install. Non-goals: multiplayer, mobile touch controls, and monetization.

## Principles

- 60fps budget: frame work must fit the 16ms budget; no GC stalls or layout thrash in the hot loop
- Deterministic game logic: simulation is frame-rate independent and reproducible given the same inputs
- Assets are first-class: sprites, audio, and animations are versioned and loaded through a single pipeline
- Zero-install play: runs in a modern browser with no plugins, no build step required at runtime
- Input responsiveness: keyboard input is sampled per frame, never debounced or throttled
- Fail loud in dev, fail graceful in prod: asset load errors surface clearly locally but degrade to playable defaults for users
- Small surface area: prefer the platform (Canvas/WebAudio) over heavyweight engines or frameworks

## Stack

- language: typescript
- package_manager: pnpm
- install: pnpm install
- test: pnpm vitest run
- lint: pnpm eslint .
- typecheck: pnpm tsc --noEmit
- build: pnpm build

## Boundaries

- Will NOT add a game engine dependency (Phaser, PixiJS, Three.js, Babylon) — rendering stays on raw Canvas/WebGL
- Will NOT require a backend server; the game runs fully client-side from static files
- Will NOT add multiplayer, networking, or account systems
- Will NOT add mobile touch controls or on-screen gamepad UI in scope
- Will NOT add monetization, ads, analytics, or third-party tracking
- Will NOT ship assets without explicit license attribution in the repo
- Will NOT add new runtime dependencies without amending this constitution

## Quality Standards

- `pnpm tsc --noEmit` passes with `strict: true`
- `pnpm eslint .` reports zero errors
- `pnpm vitest run` passes with game-logic unit tests (collision, wave progression, scoring)
- `pnpm build` produces a static bundle that runs from `file://` or any static host
- Sustained 60fps on a mid-range laptop for a full wave (measured via `performance.now()` frame timing in a dev HUD)
- All assets have a license entry in `ASSETS.md`
- No `any` types in game-logic modules (renderer/adapter layers may be exempt)

## Roadmap

### MVP
- Player ship moves left/right and fires projectiles
- Grid of invaders descends and shifts, speeding up as fewer remain
- Collision detection between projectiles, invaders, and player
- Wave clear + respawn next wave with increased difficulty
- Score display and game-over screen with restart
- Hi-res sprite rendering via Canvas/WebGL at 60fps
- Sound effects for shoot, hit, explosion, and background loop

### Hardening
- Persistent local high-score table
- Pause/resume and keyboard remapping
- Responsive canvas scaling for different viewport sizes

### Polish
- Power-ups (rapid fire, shield, multi-shot)
- Boss wave every N levels
- Particle effects and screen shake on impacts
