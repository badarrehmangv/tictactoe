# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Fruit Mountain — a 3D fruit-tossing merge game for the browser, inspired by BeXide's
*Fruit Mountain* (itself a 3D take on the Suika / Watermelon Game formula). You stand
in front of a plate and toss fruit onto it; two touching fruits of the same kind merge
into the next fruit up an 11-tier chain. There are no walls on the plate — anything
that rolls off is gone, and three drops end the run. Target platform: **Poki**.

Stack: Vite + TypeScript + three.js + Rapier3D (`@dimforge/rapier3d-compat`, WASM
physics). Every fruit, the dish, the table, and all sound effects are generated
procedurally at runtime — there are no downloaded assets, textures, or audio files,
which is why the whole game is well under a megabyte gzipped.

## Commands

```bash
npm install
npm run dev        # vite dev server, http://localhost:5173
npm run build      # tsc --noEmit && vite build -> dist/
npm run preview    # serve the production build
npm run typecheck  # tsc --noEmit only
```

There is no test suite and no lint script configured. `npm run build` is the
authoritative check — it runs the typecheck before bundling, so a failing build is
always a type error (or a Vite/Rollup error) rather than a logic regression tests
would catch elsewhere.

Dev-only extras (present at runtime, not gated to `npm run dev`):

- `?gallery=1` query param — loads `src/render/fruit/gallery.ts` instead of the game,
  showing all eleven procedural fruits at equal size for an art review.
- `` ` `` (backquote) in-game — toggles a debug overlay (fps, live fruit count, game
  state, draw calls).
- The `⚙ Dev` button (bottom-right corner) — opens a `lil-gui` panel (`src/dev/panel.ts`)
  exposing every tunable value in `CONFIG` and `TIERS` live, with a "Copy Changed
  Parameters" export and a localStorage-backed "Reset to Defaults". Ships in production
  builds (not stripped), hidden until clicked.

## Architecture

### `CONFIG` is the single source of tuning truth

`src/config.ts` holds every gameplay-feel constant (physics, plate geometry, throw
solving, camera, merge/chain rules, fail conditions, scoring, juice) as one object,
plus `src/gameplay/tiers.ts`'s `TIERS` array (per-fruit radius/award/color). Both are
**mutable at runtime**, not `as const` — the dev panel and `src/dev/overrides.ts`
write into them directly and persist changes to `localStorage`, so nearly every system
reads `CONFIG.<section>.<value>` live rather than capturing it once at construction.
`DEFAULT_CONFIG` / `DEFAULT_TIERS` are frozen JSON snapshots used to reset. When
changing a system's behavior, look for the relevant constant in `CONFIG` first —
`docs/TUNING.md` maps symptoms ("fruit skates across the dish", "runs never end") to
the exact knob to turn.

### Event bus is the only seam between simulation and presentation

`src/core/events.ts` defines a tiny typed pub/sub (`bus.on(type, handler)` /
`bus.emit(type, payload)`) with a fixed event union: `merge`, `land`, `fall`, `throw`,
`newTier`, `score`, `gameover`, `start`. `src/gameplay/pile.ts` (the physics/merge
simulation) only emits; `src/core/game.ts` wires every subscriber — HUD, particles,
score popups, camera shake, SFX (`src/audio/sfx.ts`), haptics (`src/audio/haptics.ts`).
Gameplay code never touches three.js materials or DOM directly; juice/audio/UI code
never touches Rapier bodies directly. Adding a new reaction to a game event means
subscribing in `Game`'s `wireEvents()`, not reaching into the simulation.

### Fixed-step physics with render interpolation

`src/physics/world.ts` builds one Rapier `World` (gravity, solver iterations, a
trimesh collider revolved from the same profile that builds the visible dish —
`src/render/plate.ts`'s `plateProfile()` feeds both `createPlateMesh()` and
`plateCollisionBuffers()`, so the fruit rolls on exactly what's rendered).
`src/gameplay/pile.ts` (`Pile`) owns every fruit body + mesh pair, steps physics at a
fixed `CONFIG.physics.fixedStep` with an accumulator (`Game`'s render loop), and
interpolates between the last two physics transforms for smooth rendering at any
refresh rate. Merge detection is **distance-based**, not Rapier contact events —
`resolveMerges()` groups live fruit by tier and checks pairwise distance each step,
which is what lets chains resolve reliably across a fixed step size. Fall detection
(`checkFalls()`) uses a height threshold plus a grace period (`CONFIG.fail.graceSeconds`)
so a fruit teetering on the plate's lip can still be saved by the time it resolves.

### The throw solver targets a landing distance, not a raw velocity

`src/gameplay/thrower.ts`'s `solve()` takes a charge (0..1) and a yaw and solves the
ballistic launch speed analytically so the fruit lands a charge-mapped distance from
the **plate center** (`CONFIG.throw.landNear`/`landFar`), not from wherever the camera
happens to be. This is why the dish stays centered in the power range regardless of
current zoom/aspect ratio — `originFor()` derives the throw origin from
`CameraRig.boom` (current camera distance) rather than a fixed point, and `predict()`
raycasts the analytic arc through the live Rapier world each frame to draw the landing
ring and detect risky (near-rim) landings.

### Rendering is layered: shared prototypes, instance clones

`src/render/fruit/builders.ts` builds one prototype `Group` per tier (`buildFruitPrototypes()`)
using procedural geometry generators in `geometry.ts` (`displace`, `colorize`,
`fibonacciSphere`, noise helpers) — vertex-colored, textureless. Every fruit placed in
the world is `clone(true)` of the shared prototype scaled to that tier's radius, so
geometry/material stay shared and draw calls stay low. `src/render/scene.ts` builds
the rest of the environment (sky gradient shader, table, lights); `src/render/cameraRig.ts`
is a yaw/radius orbit rig around the plate with shake support and a portrait-mode boom
adjustment (`CameraRig.boom`) that the thrower reads to keep the held fruit's on-screen
size consistent when zooming or rotating to portrait.

### Input is one unified scheme across mouse and touch

`src/platform/input.ts`'s `InputController` maps a single drag gesture to both aim/throw
(charge = drag down, yaw = drag sideways) and free orbit (right-drag, two-finger drag,
on-screen buttons, arrow keys), disambiguating by pointer count/button and exposing an
`AimState` that `Game`'s render loop reads every frame — it does not push events itself.

### Poki SDK is stubbed, not wired

`src/platform/poki.ts` wraps `gameLoadingFinished`/`gameplayStart`/`gameplayStop`/
`commercialBreak`/`rewardedBreak` as safe no-ops when `window.PokiSDK` isn't present, so
the game runs identically in dev and once the real SDK script is dropped in. `Game`
already calls these at the right lifecycle points (first input, pause/game over,
restart, tab visibility change) — the Poki-readiness pass just needs the real script tag.

## Design reference

Full design/scoring/fail rules, the complete asset list and how each procedural fruit
is built, and the tuning-symptom table live in `docs/`:

- `docs/GDD.md` — core loop, the 11-tier fruit chain (radius/award per tier), scoring
  formula, fail rules, feel rules (why charge maps to landing distance, why fruit
  "sticks" on landing).
- `docs/ASSETS.md` — every asset and its procedural construction; the two-function path
  (`createFruitMesh`/`createFruitIcon`) to swap in glTF models later if needed.
- `docs/TUNING.md` — symptom → exact `CONFIG` field to change.
- `docs/ROADMAP.md` — planned work beyond the current first-playable state (juice pass,
  timed/sprint modes, real Poki SDK integration, meta/cosmetics).

Note: `docs/ASSETS.md`/`README.md`'s file-layout listing predates the `src/dev/` (live
tuning panel) and `src/audio/haptics.ts` additions merged in from a parallel branch of
this same project — trust the actual `src/` tree over that listing.
