# Gameplay tuning guide

This guide points at the current control points for common balance work, with emphasis on the changes being considered. It describes the existing architecture; it does not make those gameplay changes by itself. Start with small A/B edits and measure the result.

## Before changing a number

The game uses metres, seconds, radians, and metres-per-second. Most shared tuning is in `src/config.js`; the systems that consume those values are in `src/game/`, with paint represented by both `src/world/paint.js` and its callers. A config value is only genuinely global if all relevant code paths use it.

For a change, write down: (1) which players/situations it affects, (2) whether it changes gameplay, visuals, or both, (3) what should stay unchanged, and (4) one repeatable check. Run one match/weapon/form at a time, then broaden to kits, bots, and online replication.

## 1. Increase paint size globally

### What chooses size today

`PaintSystem.splat(center, radius, team, options)` in `src/world/paint.js` receives the footprint radius in world metres. It uses that radius for both the GPU atlas splat and the CPU grid claim. That is why a paint-size change must keep both paths in sync. `REACH` and `DRIP_REACH` in the same module expand the quad/query bounds to fit irregular edges, satellites, and wall drips; these are not the main blob radius and should not be treated as weapon balance values.

The radius itself is generally chosen by the caller. In `src/config.js`, normal shooter-like weapons have values such as `impactRadius` and `trailRadius`; charger uses `lineRadius`; bombs and some specials use `paintRadius`/`radius`; roller and brush use `rollWidth`/`brushWidth` plus their own paint-stamping code. `src/game/weapons.js` applies those values, while each implementation in `src/game/kits/` may have its own paint calls and data. This explains why different weapons do not currently leave identical lines.

The current balance pass raises config-defined paint radii by 18%, including main weapons, kits, subs, and specials. `rollWidth` and `brushWidth` are intentionally unchanged: both also define contact hit width, so changing them would increase melee reach as well as paint coverage. Hardcoded splat sizes that have no config field are not changed by this config-only pass.

Combat damage is scaled by `GLOBAL_DAMAGE_SCALE` in `src/config.js` (currently `0.9`). Actor damage and damage dealt to the boss/crablets apply this once, so weapon, sub, special, and boss-attack hits all use the same reduction. Enemy-ink damage is applied through its own HP drain and is not affected by this multiplier.

### Two ways to tune it

- For a uniform *global* increase, apply a shared scale at the common gameplay splat boundary so every gameplay splat expands once, while preserving the same effective radius for rendering and CPU turf. Decide explicitly whether to exclude tiny cosmetic specks (`speck`/`cosmetic`) and non-splat region floods (`flood`, used for Zone Control). Do not also multiply every weapon config value, or the increase will be applied twice.
- For a targeted consistency pass, change per-weapon radius/width settings and special-case algorithms, then compare the weapons. This offers better class-by-class control, but does not guarantee a future weapon automatically inherits the increase.

A 25–35% radius increase is not a 25–35% area increase: circular area scales as radius squared, so it is about 56–82% more area before overlap and stage geometry. This can materially affect turf scores, special gauge (which charges from newly inked area), Zone Control capture, wall climb routes, and the value of bombs. Use a conservative first pass (for example, +10–15% radius), then measure; the desired 25–35% can be reached if the results support it.

### Twinfire Pistols check

The Twinfire Pistol definition is `WEAPONS.twins` in `src/config.js`. Its projectiles are spawned by the twins branch in `src/game/weapons.js`; the regular shot uses `impactRadius`, its flight trail uses `trailRadius`/`trailEvery`, and post-dodge turret fire has its own projectile cadence/aim values. Verify ordinary fire, planted/turret fire, and dodge-roll trail separately. Increasing only impact radius will not thicken a periodic trail, and increasing the trail radius does not necessarily affect the hit blob.

Check one weapon from each algorithm family: shooter/twins, charger line, blaster/bomb splash, bucket/bow or other kit, roller, brush, and sub/special. Include walls, overlapping friendly/enemy ink, and a Zone Control flood. The painted display and minimap/turf count should agree. For online play, confirm the same seeded splat reaches all clients without double scaling; [NET.md](NET.md) describes who owns and replicates ink.

## 2. Change ink consumption by weapon

Weapon ink spending is deliberately not one universal calculation. Start in `WEAPONS` in `src/config.js`, then verify the matching branch in `src/game/weapons.js` or the weapon's kit module.

- Repeating shooter-like weapons commonly use `inkPerShot`.
- Charger/spinner/bow-style charge weapons use `inkFull` with charge-dependent spending; the Bow also has `inkMin`.
- Roller/brush styles spend per travelled metre (`rollInkPerMeter` / `brushInkPerMeter`) plus separate costs for flick/swipe actions.
- Other weapons use explicit shot/action fields such as `flickInk`, `swipeInk`, `inkPerPunch`, `tapInk`, `heavyInk`, or `inkPerShot`.
- Sub weapons spend `SUBS[id].inkCost` in the shared `WeaponRunner` path. Specials refill the tank on activation; their internal repeated attacks may have distinct costs or no extra cost.
- `PLAYER.inkMax`, `inkRefillSwim`, `inkRefillKid`, and `inkRefillDelay` control the tank and recovery shared by all kits. Poison and special effects can modify spending/refill behavior.

When changing consumption, hold damage, cadence, and footprint fixed at first. Compare how many shots/actions one tank supports and how long it takes to refill under swim and kid-form rules. Avoid changing both cost and refill in the same pass; otherwise it is hard to identify why weapon uptime changed. Test empty-tank behavior and the local low-ink warning too.

## 3. Replace respawn protection with a launcher-style entry

There are three distinct mechanisms that can look like a "spawn shield"; they should not be conflated:

- `PLAYER.spawnInvuln` in `src/config.js` is the temporary post-spawn invulnerability duration. `Actor.spawnAt()` assigns it and `Actor.damage()` rejects damage while `invuln > 0`. The HUD and character VFX also read it.
- `Actor._spawnBarrier()` in `src/game/actor.js` pushes an enemy out of a radius around the other team's spawn pad. It is spatial spawn protection, not invulnerability; removing this changes spawn-camping behavior.
- `Actor.superJump()` is the teammate/map jump system: charge pose, launch arc, landing marker, and landing paint. During flight it sets invulnerability. It is not currently the automatic respawn route.

A respawn currently runs `Actor.respawn()`: it picks a small offset around the team's pad, starts about 4.5 m above it, gives the actor downward velocity, plays the spawn presentation, and uses `PLAYER.spawnInvuln`. The separate opposing-pad barrier runs in the movement update.

Initial match placement is different: `src/game/match.js` calls `spawnAt()` on the spawn deck and immediately clears `invuln`, so a respawn-protection redesign should not silently add protection at match start.

To replace the protection window with a launcher-like respawn, first decide the intended design: whether a respawned player chooses a destination or launches from a fixed base pad; whether the arc is interruptible; when input returns; what warns nearby enemies; and whether any protection remains during/after travel. Then reuse or extend the existing super-jump phases where they fit instead of creating a second unrelated trajectory. Keep the automatic respawn entry separate from player-requested super jumps unless the design intentionally unifies them.

The change spans `actor.js` (respawn and state transitions), likely `player.js`/`match.js` (selection and timing), `src/fx/fxHooks.js` and `src/fx/screenfx.js` (telegraph/presentation), `src/ui/hud.js` (status/markers), and `src/net/netmatch.js` (replicated launch/landing/teleport state). Keep `_spawnBarrier()` until a separate decision says to remove spawn-area protection. Check initial match spawns, ordinary respawns, spawn in Zone Control, online host migration, and a player leaving during travel. For the current code's exact super-jump contract, see [CONTRACTS.md](CONTRACTS.md), [EVENTS.md](EVENTS.md), and [NET.md](NET.md).

## 4. Make squid form larger without enlarging the kid

Player collision and visual character geometry are separate layers. `PLAYER.height` (kid height), `PLAYER.squidHeight`, and `PLAYER.radius` in `src/config.js` feed collision/hit queries across `actor.js`, `physics.js`, and weapon/kit hit tests. A larger physical squid height changes ceiling clearance, collision, hits, and potentially wall movement; it is not merely cosmetic.

The rendered character and squid-form pose are built by `src/game/character.js` using procedural geometry/materials from `character-geo.js`, `character-mats.js`, and related character modules. The form animation has squash/stretch channels and transition curves. To make only the squid visibly larger, adjust the squid-form visual scale at the form-pose/model layer while leaving kid-form scale, `PLAYER.height`, and shared kid geometry unchanged. If the collision silhouette should also grow, update `PLAYER.squidHeight` and audit its consumers separately. Do not scale the whole character root: that would enlarge the kid, weapon, held sub, and often animation offsets together.

Check the fully submerged form, the dry-hop squid, emergence/dive transitions, wall climbing, narrow ceilings, and hit detection. If visual size and collision size differ on purpose, document the intended gap; otherwise other players will see a larger target than the hit capsule.

## 5. Slow movement and make fights easier to read

The base ground and squid movement values live under `PLAYER` in `src/config.js`: `runSpeed`, `swimSpeed`, `squidDrySpeed`, `enemyInkSpeed`, jump velocities, gravity, and the handling group (`runAccel`, `runDecel`, `turnRate`, `swimAccel`, `swimDecel`, and related knees/rates). `Actor._horizontal()` and `_integrate()` in `src/game/actor.js` consume them. `WeaponRunner.moveSpeed()` in `src/game/weapons.js` and individual kit `moveSpeed` definitions can override the base run speed while firing, charging, rolling, or using a weapon. Specials also own their movement speeds. A global run-speed reduction will not make all weapon states equally slower unless those overrides are considered.

For a more deliberate pace, tune in this order and one category at a time:

1. Reduce `PLAYER.runSpeed` and the faster weapon-specific move-speed overrides that defeat it.
2. Adjust acceleration/braking and `turnRate` to reduce sudden direction changes; keep the S-curve parameters coherent so the actor still stops and reverses predictably.
3. Review `swimSpeed` and swim acceleration separately. Swimming is intentionally faster than running, and changing it affects escape, ink recovery, and map rotation.
4. Review `jumpVel`, `swimJumpVel`, `gravity`, and air steering as a group; changing only gravity can make arcs unintuitive or alter landings.
5. Revisit special movement speeds last, since specials are intentionally distinct bursts of power.

Change values gradually, then use `tools/measure-handling.mjs` and headless `tools/botlab/` runs to record time-to-speed, braking distance, turning, travel time, and match outcomes. Run the same stage, weapon, mode, and duration for each A/B. Bots are useful for repeatable exposure, but finish by playing manually; they do not judge readability or aiming feel.

## 6. Bring the camera closer and narrow its view

`src/game/cameraRig.js` owns gameplay framing. Its follow boom starts from `dist`/`wantDist` (currently 4.5 m) and adjusts around geometry. The player's horizontal field of view comes from `DEFAULT_SETTINGS.fov` in `src/config.js` (default 82 at a 16:9 reference), read as `baseFov`; the rig converts it to the viewport's vertical FOV. Settings are saved in local storage, and `main.js` migrates old vertical-FOV settings when needed.

For a closer view, test a smaller follow distance in the camera rig; for less peripheral coverage, test a narrower horizontal FOV setting/default. These affect aiming, perceived speed, awareness, framing, and camera collision. Avoid using only FOV to disguise an overly wide boom, and check ultrawide/mobile aspect ratios as well as 16:9. The game camera (`gameCam`) is also used for aiming and hit presentation while the map diorama is open, so preserve the separate gameplay camera behavior.

Character perceived size can also be affected by viewport and the camera target height, but changing `PLAYER.height` is not a camera fix and would affect collision. Keep the requested kid-model proportions separate from the camera and squid-only scale work.

## Verification checklist

- `npm run check` catches syntax errors across source, tools, and server modules.
- `npm run smoke` catches startup/runtime console errors and plays a short autopilot match.
- `npm run check-maps` protects stage and Zone Control layout assumptions.
- `npm run botlab` measures aggregate handling and balance with repeatable matches; use `MAP`, `MODE`, and `SECS` as described in its README.
- `npm run net-test` checks client/relay consistency, including actor snapshots, paint coverage, roster, and results.
- `tools/measure-handling.mjs` is the focused movement measurement helper; `tools/character-lab.html` and `tools/model-lab.html` are useful for visual form/scale inspection; `tools/shot.mjs` captures a real browser frame with console errors.

Start with `check` plus the narrowest relevant test, then run smoke and multiplayer checks for changes that cross those boundaries. A successful syntax check alone does not establish that the CPU paint grid matches the visible paint or that online clients see the same result.
