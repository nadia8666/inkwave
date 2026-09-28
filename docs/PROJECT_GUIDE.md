# INKWAVE project guide

This is a plain-language map of the repository. It is intended to help you answer two questions: "Where does this behavior live?" and "What else will change if I edit it?" The API-level detail is in [CONTRACTS.md](CONTRACTS.md), [EVENTS.md](EVENTS.md), [NET.md](NET.md), [RIG.md](RIG.md), and [BOSS.md](BOSS.md).

## The short version

INKWAVE is a browser 3D game written as native JavaScript ES modules. Three.js is vendored in `vendor/three` and exposed by an import map in `index.html`. There is no transpiler, bundler, or generated source tree. The browser downloads the modules directly; most geometry, materials, audio, particles, and map dressing are created by code at runtime. `assets/` contains supporting content such as fonts, baked lightmaps, news imagery, and stage captures.

A useful first read is `src/config.js`: it defines player feel, weapon data, mode rules, stage metadata, settings defaults, and rendering quality presets. `src/main.js` is the composition root: it creates systems, connects them, owns the frame loop, and switches between menu, attract, match, and results states. For a behavior change, follow the path from its configuration through the system that consumes it rather than assuming the visible object is the gameplay owner.

## Start the game

Install dependencies once with `npm install`. On a machine with Node.js and Python available:

- `npm start` launches the Electron shell.
- `npm run serve` runs the included static server on port 8490; open `http://localhost:8490`.
- `npm run check` syntax-checks the JavaScript modules.
- `npm run smoke` starts the browser smoke test; it needs Google Chrome and Puppeteer tooling.
- `npm run check-maps` validates stage layouts and Zone Control variants.
- `npm run build` assembles a distributable web folder in `dist/`.
- `npm run botlab` runs headless matches for tuning; see `tools/botlab/README.md`.
- `npm run relay` runs the local multiplayer relay; `npm run net-test` tests real headless clients against it.

The npm scripts use a mixture of POSIX shell syntax, `sh`, `python3`, and Node. On Windows, run shell-dependent scripts from Git Bash or WSL, and make sure the requested Python/Chrome programs are installed. The project has no compile step for ordinary browser development: edit a module, refresh the page, and check the browser console.

Useful local URLs include `?map=halyard&time=dusk`, `?autostart=180`, and `?autopilot`. The `?devstage` flag is a development-only way to open an online-only stage as a solo backdrop.

## How one frame works

1. `src/main.js` reads input and advances the active game state. It owns startup and match/menu orchestration.
2. `src/core/input.js` merges keyboard, mouse, and gamepad controls. `src/game/player.js` turns that snapshot into movement, aim, and action intent for the local actor.
3. `src/game/actor.js` is the gameplay owner for each player or bot: movement, collisions, health, ink tank, squid/kid form, respawn, and special state. Bots use the same Actor and weapon rules as people.
4. `src/game/weapons.js`, `src/game/kits/`, `src/game/subs.js`, and `src/game/specials.js` simulate attacks. `src/game/physics.js` performs world collision and hit queries.
5. `src/world/paint.js` writes each splat into a GPU atlas and updates a matching coarse CPU grid. The atlas is for appearance; the grid answers gameplay questions and computes turf totals. Both must describe the same footprint.
6. `src/game/match.js` owns match phases and results; `src/game/zones.js` owns Zone Control rules. `src/game/bots.js` supplies bot decisions, and `src/game/nav.js` provides navigation.
7. Event subscribers in `src/fx/`, `src/audio/`, and `src/ui/` react to gameplay without needing to own it. The renderer and camera update the presentation.
8. Online play adds `src/net/`: each player simulates their own actor, the host runs bots and match authority, and the relay forwards messages. Ink splats are replayed to keep every player's map consistent.

Most systems are reachable through the shared `G` object in `src/core/ctx.js`. This is deliberate lazy wiring: modules can use `G.paint`, `G.physics`, `G.audio`, and other systems without creating import cycles. The same module contains a small event bus (`on`/`emit`). Use events for presentation reactions; keep gameplay decisions in the gameplay owner.

## Where things live

### Application and shared rules

- `index.html`: browser document, import map, canvas/UI roots, and stylesheet/module entry points.
- `src/main.js`: application boot, world assembly, frame loop, menus/matches/results, persistence, and system wiring.
- `src/config.js`: shared gameplay/content data. `PLAYER` is the baseline movement/health/ink configuration; `WEAPONS`, `SUBS`, and `SPECIALS` are the loadout definitions; `MATCH` and `ZONES` hold mode rules; `MAPS` is stage metadata; `DEFAULT_SETTINGS` and `QUALITY` feed settings and rendering.
- `src/core/ctx.js`: shared `G`, event bus, and small math helpers.
- `src/core/input.js`: keyboard/mouse/gamepad snapshot and rumble.
- `src/core/renderer.js`: three.js renderer, render targets, and post-processing.
- `src/dev/stubs.js`: fallback modules used when selected optional modules fail to load.

### Player, combat, and match simulation (`src/game/`)

- `actor.js`: one character's authoritative local simulation: physics integration, form changes, damage, ink, special gauge, respawn, super jumps, and events.
- `player.js`: translates the local input snapshot into actor intent, including aim, map selection, and super-jump target selection.
- `physics.js`: collision queries, ground probes, raycasts, and geometric hit tests.
- `weapons.js`: per-actor weapon runner plus projectiles and hit application. Weapon numbers are generally read from `config.js`.
- `kits/`: weapon implementations that need their own simulation class rather than a branch in the general runner. `registry.js` connects implementations to weapon IDs; `index.js` imports the registrations. The kit modules cover Bow, Brolly, Blade, Mitts, Boomerang, Shaker, Waddle, Torpedo, and Tracer behavior, with companion model/effect/sound files where present.
- `subs.js`: non-kit sub-weapon entities and shared sub-system behavior.
- `specials.js`: special-weapon state machines and movement/damage modifiers; `special-props.js` builds special visual objects.
- `bots.js`: decisions and tactics for offline bots and host-owned online bots.
- `nav.js`: stage navigation graph and route queries used by bots and moving entities.
- `match.js`: offline match actors, timers, lifecycle, scoring, and result handoff.
- `zones.js`: Zone Control state machine; its zone geometry data is in `src/world/zones-data.js`.
- `minimap.js`: map paint/player/death/special markers and minimap effects.
- `cameraRig.js`: third-person follow, camera collision, map diorama, spectating, and cinematic views.
- `character.js`: procedural character model and animation, coordinating the files below. Gameplay actor dimensions are not the same thing as the rendered squid mesh dimensions.
- `character-geo.js`, `character-mats.js`, `character-face.js`, `character-hair.js`, `character-outfit.js`, `character-style.js`, `character-weapons.js`: procedural character geometry, materials, face, tentacles, outfit, appearance choices, and held weapon/sub models.
- `character-lod.js`: distance-based model detail selection.
- `lobbySet.js`, `lobbySet-geo.js`, `lobbySet-mats.js`, `lobbySet-tex.js`: the separate, procedural online lobby environment.
- `showcase.js`: menu/lobby/result staging and character presentation.

Character proportions and animation have a detailed bone and pose contract in [RIG.md](RIG.md). Keep gameplay capsule dimensions and visual meshes coordinated when changing the player model; the rig is not the collision system.

### World and paint (`src/world/`)

- `level.js`: constructs a runtime level from a stage layout, exposes bounds/spawn pads, and provides spatial queries.
- `maps.js`: registers imported stage layouts and shared defaults.
- `stages/`: stage-specific `layout.js`, `surfaces.js`, `props.js`, and `murals.js`; `stages/surfaces.js` contains shared surface definitions. Each layout is data/code for one team half and is mirrored to make the other half.
- `variants.js`: selects mode-specific layout changes.
- `zones-data.js`: Zone Control region boundaries and zone definitions.
- `paint.js`: texture-atlas rendering, CPU turf grid, paint queries, region floods, drying, and ripples. Its `splat()` radius is a gameplay input, not just a visual brush size.
- `levelMaterial.js` and `inkShading.js`: stage surface material and the shader that makes ink wet, raised, glossy, and colored.
- `texlib.js`: procedural surface texture library.
- `environment.js`: sky, lighting, water, and ambient environment.
- `decor.js`, `dressing.js`, `props.js`, `props-marina-dock.js`, `props-marina-vessels.js`: stage scenery and procedural props.
- `murals.js`: mural texture creation; `mapThumb.js`: stage-card thumbnail creation; `mapkit.js`: shared stage-building helpers.

`assets/lightmaps/` contains baked lighting data used by stages. `build/bake-ao.cjs` and `tools/bake-ao.mjs` are related bake entry points; do not hand-edit generated bake data without checking the bake workflow. `build/check-maps.mjs` validates stage layouts.

### Network (`src/net/` and `server/`)

- `session.js`: room/lobby state and host actions.
- `netmatch.js`: actor snapshots, event replication, remote interpolation, match authority, and network match flow.
- `transport.js`: WebSocket connection to the relay.
- `mock.js`: local/mock network adapter used by tests or fallback paths.
- `server/src/index.js`: Cloudflare Worker/Durable Object room relay. It forwards room traffic; clients own gameplay simulation.

The detailed authority, timing, event, and testing contract is in [NET.md](NET.md). When changing hit, paint, respawn, or movement behavior, consider what is replicated and which player owns the state. A visual-only remote projectile should never make its own hit or turf decision.

### Presentation (`src/ui/`, `src/fx/`, `src/audio/`, and `styles/`)

- `ui/menus.js`: title, mode/setup, loadout, settings, online lobby, pause, and result screens.
- `ui/hud.js`: in-match HUD; `ui/hud-boss.js` is the boss-mode HUD.
- `ui/diorama.js`: expanded map overlay and its interaction; `ui/boss-art.js` supplies boss UI artwork.
- `ui/ui-util.js`, `ui/ui-icons.js`, `ui/menu-art.js`, `ui/news.js`: reusable UI helpers, icons, decorative artwork, and news content.
- `fx/fx.js`: pooled gameplay particles and impact effects; `fx/fxHooks.js` connects game events to effects; `fx/screenfx.js` handles screen-space effects; `fx/swimWake.js` and `fx/zoneMarks.js` render specialized ink wakes/zone marks.
- `audio/audio.js`: procedural sound effects and audio engine; `audio/music.js`: procedural music; `audio/bossAudio.js`: boss-specific sound behavior.
- `styles/ui.css` and `styles/hud.css`: menu and gameplay UI styles.

The event bus contract and existing event names are in [EVENTS.md](EVENTS.md). Sound definitions belong in `src/audio/audio.js`; procedural music belongs in `src/audio/music.js`. UI and effects should respond to events rather than duplicate the underlying game rules.

### Boss mode (`src/boss/`)

`boss.js` is the simulation owner; `bossBrain.js` selects moves; `bossHazards.js` builds timed attack hazards; `bossNav.js` constrains navigation; `bossMode.js` connects boss state to the match. `bossModel.js`, `bossModelGeo.js`, `bossMats.js`, `bossAnim.js`, and `bossModelFx.js` own the procedural model, geometry, materials, animation, and visual effects. See [BOSS.md](BOSS.md) before changing this mode.

### Assets, build, desktop, and tools

- `assets/fonts/`: bundled fonts. `assets/news/`: news images/content. `assets/stages/`: stage preview captures and `manifest.json`. `assets/lightmaps/`: baked lighting files.
- `songs/`: optional user-supplied music; see `songs/README.md`.
- `vendor/three/`: vendored Three.js module and add-ons.
- `electron/main.cjs`, `electron/preload.cjs`: desktop window lifecycle and the limited bridge exposed to the web game.
- `server/wrangler.jsonc`: Cloudflare Worker development/deployment configuration.
- `build/`: packaging, AO baking, map checks, music manifest, and macOS metadata/icon helpers.
- `tools/serve.py`: local static server; `tools/smoke.sh`, `play.mjs`, and `shot.mjs`: headless startup/playback and screenshots; `audio-test.mjs` / `audio-lab.*`, `core-lab.*`, `fx-lab.*`, `ui-lab.*`, `texlib-lab.*`, `props-lab.*`, `lobbyset-lab.*`, `model-lab.*`, `character-lab.*`, and `boss-lab.*`: focused development labs. `measure-handling.mjs` and `tools/botlab/` measure player/bot behavior. `stage-shots.mjs`, `light-shots.mjs`, `reel.mjs`, `film.py`, `reel-compose.py`, and `tools/reels/` support captures and reels. `build-dist.py`, `release*.sh`, and `release-pages.sh` assemble or publish builds. `tools/net-test.mjs` tests online sessions.
- `CONTRIBUTING.md`: contribution workflow and code conventions. `vercel.json` and `server/wrangler.jsonc` hold hosting/relay deployment configuration.

## Data flow and safe ownership rules

- **Configuration describes defaults; runtime objects hold live state.** Edit `config.js` for shared constants/data. Do not treat its objects as per-player mutable state.
- **The actor decides movement and damage.** Input, bot code, menus, and rendering should not directly implement a second movement/damage rule.
- **Paint has two synchronized representations.** The GPU atlas is what you see; the CPU grid decides where you can swim, turf counts, and map ownership. Change the common paint operation rather than only its shader if the actual gameplay footprint must change.
- **Events carry consequences to presentation.** Emit the gameplay event at the source; let HUD/audio/FX subscribe. Existing payloads are documented in [EVENTS.md](EVENTS.md).
- **Network ownership matters.** Local actors simulate locally, the host owns bots and match-level decisions, and remote actors are interpolated views. Keep event order and deterministic paint records intact.
- **Units are metres, seconds, and radians.** The world uses Y up. Character roots are at the feet and face +Z at yaw zero. See [CONTRACTS.md](CONTRACTS.md).

## First-pass change workflow

1. Search for the config property or function named in the relevant guide; follow its call sites to find exceptions.
2. Change one behavior layer at a time. For tuning, record the old/new values and run the focused handling, bot, paint, or network check before stacking more changes.
3. Run `npm run check`, then the matching focused test (`npm run smoke`, `npm run check-maps`, `npm run botlab`, or `npm run net-test`).
4. Test both solo and multiplayer when the behavior can be replicated; check a normal and a special/kit path when they use different implementations.
5. For movement or paint tuning, capture repeatable results with the tools in `tools/` rather than relying only on one subjective match.

See [GAMEPLAY_TUNING.md](GAMEPLAY_TUNING.md) for detailed walkthroughs of paint size, ink costs, respawn launch flow, squid-only scale, movement pace, and camera framing.
