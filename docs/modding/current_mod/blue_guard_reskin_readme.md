# Jak 2 — Haven City Rebellion (`haven-city-rebellion`)

> **Mod Readme**
>
> - **Repository:** [`whozghiar/jak2-mod-haven-city-rebellion`](https://github.com/whozghiar/jak2-mod-haven-city-rebellion)
> - **Type:** `features`
> - **Depends on:** the existing `build-actor` custom-actor pipeline
>   (`goal_src/jak2/lib/project-lib.gp`, `goalc/build_actor/`)

---

## 1. What this is

A blue-recolored Crimson Guard, added as its **own standalone GOAL entity** (`crimson-blue-guard`)
rather than a global texture replacement — the stock red `crimson-guard` keeps spawning
unmodified. The blue variant reuses `crimson-guard`'s animations and collision; its main
difference is that it is passive toward Jak by default, and only becomes
personally hostile toward him if he attacks it directly (no city-wide alarm either way) — see §5.
It also has 8 HP, rolls its own weapon, dissolves on death instead of running the stock death
effect, and keeps a ranged standoff distance (§6, §8).
A separate, manually-triggered function makes it fight another guard on purpose (also §5). It is
mixed into Haven City's ambient guard traffic while City Insurrection is enabled (§9.2).

The source asset is `custom_assets/jak2/models/custom_levels/crimson-blue-guard.glb` (also copied
to `custom_assets/jak2/models/common/crimson-blue-guard-lod0.glb`, see §4.3): the decompiled native
`crimson-guard` skeleton + all 40 of its animations, re-skinned with a recolored texture set in
Blender, then re-exported.

## 2. The core problem: animation slot indices

`crimson-guard`'s ~3650 lines of AI/state-machine code (`guard.gc`,
`goal_src/jak2/levels/city/traffic/citizen/guard.gc`) reference its animations almost entirely by **numeric
slot index** into its art-group's element array — either through overridable fields
(`anim-walk`, `anim-run`, `anim-get-up-front`, ...) set once in `init-enemy!`, or, in a handful of
methods (`enemy-method-77`, `enemy-method-78`, `set-behavior!`), as **raw literals** baked
directly into the method body (`(-> this draw art-group data 42)` and friends).

The native `crimson-guard-ag` art-group has a fixed layout (see
`decompiler/config/jak2/ntsc_v1/art-group-info.min.json`, key `crimson-guard-ag`):

| Slot | Content |
|---|---|
| 0 | `crimson-guard-lod0-jg` (skinned mesh) |
| 1 | `crimson-guard-lod0-mg` |
| 2 | `crimson-guard-lod2-mg` |
| 3 | `crimson-guard-shadow-mg` |
| 4..43 | 40 animations, in a fixed order (`idle`@4, `walk`@5, `run`@6, ..., `get-up-front`@33, `get-up-back`@34, ...) |

The existing `build-actor` tool (`goalc/build_actor/jak2/build_actor.cpp`) does **not** reproduce
this layout for a standalone custom actor: it always emits a 2-slot header (`jgeo`, one dummy
null slot) before the animations, and it orders animations by their order in the source `.glb`'s
`animations` array — which a normal Blender/glTF export sorts alphabetically. Building the blue
guard "as-is" would have put `crimson-blue-guard-ag`'s `idle` at slot 2 instead of 4, `get-up-back`
at some alphabetically-derived slot instead of 34, etc. — silently playing the *wrong* animation
in every hardcoded-index code path, breaking the "identical behavior" requirement in subtle,
hard-to-notice ways (e.g. only the vehicle-knockout or yellow-eco-hit reactions, which use raw
literals, would be wrong).

## 3. The fix — two additive, opt-in pieces

### 3.1 `build-actor :native-header #t`

`goal_src/jak2/lib/project-lib.gp`'s `build-actor` macro gained a new `&key (native-header #f)`
parameter, threaded through to the `build-actor2` data-compiler tool
(`goalc/make/Tools.cpp::BuildActor2Tool`) and finally to
`jak2::BuildActorParams2::native_anim_header` (`goalc/build_actor/jak2/build_actor.h`). When set,
`run_build_actor` (`goalc/build_actor/jak2/build_actor.cpp`) emits **two extra null placeholder
slots** after the mesh, padding the header from 2 to 4 slots — matching the native layout exactly.
Default is `#f`, so every existing custom actor (`test-actor`, the jetboard, etc.) is completely
unaffected.

```lisp
(build-actor "crimson-blue-guard" :force-run #t :native-header #t)
```

### 3.2 Reordering the source `.glb`'s animation array

A one-off Python script reordered `crimson-blue-guard.glb`'s `animations` JSON array (pure
reordering of array elements — no accessor/bufferView/mesh data touched) to match the 40-name
canonical order from `art-group-info.min.json` above. Combined with the 4-slot native header,
this makes `crimson-blue-guard-ag`'s slot N hold the *same* animation as `crimson-guard-ag`'s slot
N, for every N. If you ever need to rebuild the `.glb` from a fresh Blender export, re-run
`python scripts/modding/reorder_crimson_guard_glb_anims.py <in.glb> <out.glb>` before running
`build-actor`, or your animation indices will drift again.

With both pieces in place, `crimson-blue-guard` needs only to override
`init-enemy!` (`goal_src/jak2/levels/city/traffic/citizen/crimson-blue-guard.gc`) to point at its
own skeleton-group by name — every other inherited method/state from `crimson-guard` keeps
working with the exact same numeric indices, unmodified.

```lisp
(deftype crimson-blue-guard (crimson-guard)
  (...)) ;; knocked-fatal? + the inert squad-* fields

(def-art-elt crimson-blue-guard-ag crimson-blue-guard-lod0-jg 0)
(def-art-elt crimson-blue-guard-ag crimson-blue-guard-lod0-mg 1)

(defskelgroup skel-crimson-blue-guard crimson-blue-guard crimson-blue-guard-lod0-jg -1
              ((crimson-blue-guard-lod0-mg (meters 999999)))
              :bounds (static-spherem 0 0 0 5)
              :origin-joint-index 3)

(defmethod init-enemy! ((this crimson-blue-guard))
  ;; identical to crimson-guard's init-enemy!, except the skeleton-group name
  ...)
```

## 4. Getting it into the world

- **Code residency:** `crimson-blue-guard.gc` compiles to `crimson-blue-guard.o`, added next to
  `guard.o` in `goal_src/jak2/dgos/cwi.gd` (the always-resident common DGO that already carries
  `crimson-guard`'s own code).
- **Art residency:** `crimson-blue-guard-ag.go` was added next to every existing
  `crimson-guard-ag.go` entry (append-only, nothing removed) in the 10 level DGOs that carry it:
  `cas.gd`, `dg1.gd`, `fdb.gd`, `fea.gd`, `fob.gd`, `fra.gd`, `lwidea.gd`, `lwideb.gd`, `lwidec.gd`,
  `pae.gd`. This guarantees the blue variant's assets are loaded everywhere the stock guard's are,
  so it can never be picked for a spawn without its art being resident.
- **Ambient traffic spawning:** `traffic-manager.gc::traffic-object-spawn` is the single place
  where the traffic simulation turns a `(traffic-type crimson-guard-1)` / `crimson-guard-2` /
  `(traffic-type crimson-guard-0)` pick into a concrete process, via
  `(citizen-spawn arg0 crimson-guard arg1)` in stock. Both arms now ask
  `*mod-city-guard-spawn-blue-hook*` (declared in `traffic-h.gc`, wired in `mod-city-hooks.gc`)
  and spawn `crimson-blue-guard` instead of `crimson-guard` when it returns `#t`. The hook
  returns `#f` while `*mod-city-insurrection?*` is off, and otherwise picks the faction from
  Jak's district (§9.2). `*crimson-blue-guard-ratio*` (default `2`) is still defined, but nothing
  reads it any more. This is the **only** spawn-type touch point: the `traffic-type` enum and the
  `guard-type-info-array` weighting table are untouched — `crimson-blue-guard` is just an
  alternate concrete type for an existing spawn decision, so all traffic-engine bookkeeping (nav
  mesh, alert state, population counts) behaves identically whichever variant lands in that
  process slot. The other traffic-engine changes are City Insurrection hooks (§6, §9.2).
- `(declare-type crimson-blue-guard crimson-guard)` was added near the top of `traffic-manager.gc`
  so the reference above compiles independent of file ordering (same idiom as `crimson-guard`'s
  own forward declaration in `traffic-engine.gc`).

### 4.3 A second, easy-to-miss piece: the actual drawable geometry ("Circuit 2")

`build-actor` (Circuit 1, §3) only produces the skeleton/animations art-group. The actual triangles
+ textures the PC renderer draws (Circuit 2) come from a completely separate system: the
decompiler bakes them into `.fr3` files, looked up **by name** at runtime. See
`.agents/skills/custom-actors-levels/SKILL.md` (knowledge base) for the full mechanism.
`build-actor`'s own merc-ctrl output is a placeholder (`generate_dummy_merc_ctrl` in
`build_actor.cpp` literally reuses a hardcoded dummy mesh) — without Circuit 2, the guard spawns,
moves and makes sound normally, but is **invisible**.

The fix: a second copy of the same `.glb`, renamed to match the placeholder merc-ctrl's own name
(`<art-group-name>-lod0`, here `crimson-blue-guard-lod0.glb`), dropped in
`custom_assets/jak2/models/common/`. The decompiler's `add_custom_model_to_level`
(`decompiler/level_extractor/extract_merc.cpp`) auto-scans that folder at `task extract` time — no
config needed — and bakes the model + all its textures into `GAME.fr3` (`common` → always
resident, regardless of level). This is a one-time step (or after any `.glb` model change); it
does **not** need to be repeated after ordinary `(mi)` GOAL-code iteration.

## 5. Faction behavior

`crimson-blue-guard` keeps `crimson-guard`'s collision, animations and movement, and differs from
it in one faction respect: it does not fight *for* the Crimson Guard side against Jak by default.
Its other differences — 8 HP, its own weapon roll, the death dissolve and the ranged combat
states — are listed in §6 and §8. The faction overrides, in
`goal_src/jak2/levels/city/traffic/citizen/crimson-blue-guard.gc`, are:

- **`citizen-init!` override** — forces the "not targeting Jak" `focus collide-with` collide-spec
  unconditionally (crimson-guard's own version picks it based on the *shared*, city-wide
  `traffic-alert-flag target-jak` flag, which can't be used to keep just one variant passive). The
  guard keeps its `enemy` collide-as bit (so it's still a valid target for others), it just never
  opportunistically treats Jak as a target on its own.
- **`general-event-handler` override**:
  - `'hit`/`'hit-flinch`/`'hit-knocked`: differs from crimson-guard's own case in two ways — a
    hit from another blue guard or its projectile (`crimson-blue-guard-friendly-attacker?`) is
    ignored outright, and any other attacker, Jak included, is remembered as its target
    (`traffic-target-status handle` + focus) instead of calling `trigger-alert`, so
    the city-wide alarm never raises. It then falls through to
    `(method-of-type nav-enemy general-event-handler)`, the exact same call stock crimson-guard
    makes — so the actual flinch/knockback/get-up/hostile transition is stock.
  - `'panic`/`'clear-path`: identical to stock, except danger attributed to Jak (gunfire near the
    guard, not necessarily a direct hit — see `traffic-engine::update-danger-from-target`, which
    always stores Jak's handle as the source) never raises the alert either. Without this, firing a
    weapon near the guard would still sound the alarm even with the `'hit` fix above.
  - `'alert-begin` is turned into a deliberate no-op: stock crimson-guard's version targets whoever
    triggered the alert (almost always Jak) and goes hostile toward them — exactly the "attacks Jak
    during a general alert" behavior this variant must not have.
- **`crimson-blue-guard-attack-guards`** (plain `defun`, not a method, not called from anywhere
  automatically) — the one way to make this guard fight another guard on purpose. Finds the nearest
  other (non-blue) `crimson-guard` within ~60m via `find-nearest-enemy-guard` (`guard.gc`), which
  with `#f` only returns red guards (`type-type?` excludes `crimson-blue-guard`), so blue
  guards can't be made to target each other, then sets the target and calls `go-hostile` — same
  mechanism `'alert-begin`/`'hit` use. Call it from the REPL once you have a handle on the guard
  (e.g. `(define g (spawn-crimson-blue-guard-debug 0))`, then
  `(crimson-blue-guard-attack-guards (the-as crimson-blue-guard g))`).

None of these overrides modify `crimson-guard`/`guard.gc` itself (the City Insurrection changes to
`guard.gc` are listed in §6). **Caveat on the manual trigger:** it reuses
crimson-guard's own combat state machine (through this variant's `hostile`/`gun-shoot`/`close-attack`
overrides), which is generic about *what* the current
target is (it reads `(-> this focus handle)`/`traffic-target-status handle`, not a hardcoded
`*target*` check) — but stock `crimson-guard` never actually has occasion to point that machinery at
another guard, only at Jak, so this exact combination (guard vs. guard) has no native precedent to
verify against. Whether a red guard that gets shot back fights back is governed by `guard.gc`'s
`'hit` case, which this mod extends: a red guard hit by a blue guard targets it and goes hostile
without raising the alert ("Reciprocal retaliation", §9.2).

## 6. Engine Changes Made on This Branch

| File | Change | Why |
|---|---|---|
| `goalc/build_actor/jak2/build_actor.h` | `BuildActorParams2` gained `bool native_anim_header = false;` | carries the new opt-in flag |
| `goalc/build_actor/jak2/build_actor.cpp` | `run_build_actor` emits 2 extra null header slots when the flag is set | matches the native 4-slot art-group header so reskins can reuse original anim indices |
| `goalc/make/Tools.cpp` | `BuildActor2Tool::needs_run`/`::run` accept a 9th `:in` element, parsed into `native_anim_header`; max input count raised from 8 to 9 | plumbs the flag from the GOAL macro through to the tool |
| `goal_src/jak2/lib/project-lib.gp` | `build-actor` macro gained `&key (native-header #f)`, appended to the `:in` list | GOAL-side opt-in switch, defaults preserve all existing custom actors |
| `goal_src/jak2/levels/city/traffic/citizen/crimson-blue-guard.gc` (new) | `deftype`, `def-art-elt` x2, `defskelgroup`, `init-enemy!` override | the new entity itself |
| `goal_src/jak2/game.gp` | `(build-actor "crimson-blue-guard" ...)` + `(goal-src ...)` registration | builds the art-group, registers the new source file |
| `goal_src/jak2/dgos/cwi.gd` | `"crimson-blue-guard.o"` added next to `"guard.o"` | code residency |
| `goal_src/jak2/dgos/{cas,dg1,fdb,fea,fob,fra,lwidea,lwideb,lwidec,pae}.gd` | `"crimson-blue-guard-ag.go"` added next to each `"crimson-guard-ag.go"` | art residency, matching the stock guard's footprint exactly |
| `goal_src/jak2/levels/city/traffic/traffic-manager.gc` | `(declare-type crimson-blue-guard crimson-guard)`, `*crimson-blue-guard-ratio*` (default `2`, no longer read), `*mod-city-guard-spawn-blue-hook*` faction pick in the `crimson-guard-1`/`crimson-guard-2` and `crimson-guard-0` arms of `traffic-object-spawn`, `spawn-crimson-blue-guard-debug` REPL helper | mixes the blue variant into ambient city traffic, without touching the traffic-type enum or any weighting table; gives a one-liner to force-spawn one for testing |
| `custom_assets/jak2/models/common/crimson-blue-guard-lod0.glb` (new) | copy of the build-actor `.glb`, renamed | Circuit 2 — see §4.3 |
| `goal_src/jak2/levels/city/traffic/citizen/crimson-blue-guard.gc` | `citizen-init!`, `general-event-handler` overrides + standalone `crimson-blue-guard-attack-guards` function | passivity toward Jak + manual guard-vs-guard trigger — see §5 |
| `goal_src/jak2/levels/city/traffic/citizen/crimson-blue-guard.gc` | no `crimson-guard-method-214`/`216`/`222` override (gun shot, line-of-sight probe, taser lightning) | both `.glb` copies keep the native 38-bone skeleton, so the inherited methods read the muzzle/beam origin from native joints 14/15 ("blast"/"dirblast") unchanged; the former method-214 override is gone (§9.3) |
| `goal_src/jak2/levels/city/traffic/citizen/crimson-blue-guard.gc` | `die` state + `crimson-blue-guard-dissolve-sequence` + `enemy-method-78` override | Robust custom actor death dissolution: skips standing die animation if `knocked-fatal?` so guard stays flat on the ground, plays `"enemy-fizz"`, launches purple dissolution particles (`merc-death-spawn 73`) across joints for 60 frames with jitter, and hides mesh on frame 5. Replaces `do-effect 'death-default` to prevent the C++ `generic_merc_death` crash (`exit status 5`) on dummy `build-actor` geometry |
| `goal_src/jak2/engine/ai/traffic-h.gc` | `(define-extern *mod-city-peaceful?* symbol)` / `(define-extern *mod-city-insurrection?* symbol)` | forward declarations so engine code and the menu file can reference the flags regardless of compile order |
| `goal_src/jak2/levels/city/traffic/citizen/mod-city-hooks.gc` | `*mod-city-peaceful?*` / `*mod-city-insurrection?*` globals, both default `#f` | CWI-resident non-debug home for the flags. `*mod-city-insurrection?*` is the **master enable** for this branch. |
| `goal_src/jak2/pc/features/crimson-blueguard-insurrection-menu.gc` *(new)* | `mod-crimson-blueguard-insurrection-build-menu` + 3 toggle/select functions (`…-toggle-enable`, `…-toggle-district`, `…-select-music`) + two flushes — `mod-crimson-blueguard-insurrection-rebuild-guards` (`'kill-all` + `'spawn-all`, used by `Enable`) and `mod-crimson-blueguard-insurrection-flush-guards` (light park, used by the war-zone picker) — registered via `(mods-menu-register "crimson-blueguard-insurrection" …)` | the mandatory Mods-menu toggle (L3 + SELECT, retail-safe popup menu): `Enable` / `War zones` submenu / `War music` submenu. **Replaces** the old hand-rolled "Mods" root-menu block in `default-menu-pc.gc` (now reverted to stock). Wired in `game.gd` after `mods-menu.o`. |
| `goal_src/jak2/dgos/game.gd` | `"crimson-blueguard-insurrection-menu.o"` after `"mods-menu.o"` | menu-file residency in GAME.CGO |
| `goal_src/jak2/levels/city/traffic/traffic-manager.gc` | `*mod-city-insurrection?*` gate added to: `traffic-want-counts` slots 18/19 (`(if *mod-city-insurrection?* 8 4)` / `… 8 3)`), and the `spawn-all` dark-guard roll's pool-0/2 extension | OFF ⇒ retail want-counts and retail dark-guard eligibility |
| `goal_src/jak2/levels/city/traffic/citizen/guard.gc` | `*mod-city-insurrection?*` gate added to every previously-ungated addition: the `'touched`/`'touch` melee case, the `'traffic-activate` intercept, `crimson-guard-method-214`'s grenade branch, the `dead` traffic-target drop, guard-type-2 ranged-fire eligibility, the 3→2/0 burst-count change, and both `event-self 'touched` cshape-init sets | OFF ⇒ every stock `crimson-guard` behaves byte-for-byte like retail |

**Native non-regression:** with `*mod-city-insurrection?*` `#f` (the shipped default) Haven City is
byte-for-byte stock Jak 2 — `mod-city-hooks.gc` installs `#f`-returning / stock-delegating hooks,
`mod-city-insurrection-update-traffic` no-ops until the flag is first set, no `crimson-blue-guard`
is ever constructed, and all the `guard.gc` war-zone tweaks above are gated. The only always-on
deltas are cosmetic/harmless: the `crimson-blue-guard` art-group logs in at city load (unused), a
`citizen` skips its look-at when the focus is `dead`/`inactive` (`citizen.gc`), a dormant
`blue-guard-frustum` entry is appended to `*minimap-class-list*` (`minimap.gc`, only
`crimson-blue-guard` references it), every `crimson-guard` carries a new `grenade-last-time` field
at the end of its type (reset in `citizen-init!`, read only by the gated grenade branch), and the
custom art-group link path in `joint.gc`/`level.gc` is inert. The C++
`build-actor`/`Tools.cpp` changes are opt-in (`native-header #f` default).

## 7. How to Test

1. `task build-release-game` (or `build-debug-game`) — only needed after a C++ change
   (`build_actor.cpp`/`Tools.cpp`); not needed for GOAL-only iteration.
2. `task extract` — required once (or after the `.glb` model changes) to bake Circuit 2, see §4.3.
   Check the log for `Adding custom model crimson-blue-guard-lod0 to common` and no
   `merc failed to find texture` for it.
3. `task repl`, then `(mi)` — must reach "Successfully built all N targets" with no
   `could not find a master slot to link` / `link-art` errors.
4. `task boot-game` (or `(r)` from the REPL), reach Haven City.
4b. **Enable the mod:** `L3 + SELECT ▸ Mods ▸ crimson-blueguard-insurrection ▸ Enable`. OFF by default —
    verify first that with it OFF the city shows only red guards, retail traffic density, and
    normal guard combat (4-round rifle bursts, no grenade launchers). Then set the war zone in the
    `War zones` submenu (default: Industrial). Toggling re-rolls / purges the city traffic live
    — `Enable` destroys and rebuilds every pool, so blue guards must appear within a second or
    two **without reloading a save**. If they only show up after a reload, the rebuild is not
    running.
5. With the mod ON, walk into the Slums (every ambient guard spawns blue there), or at the REPL
   `(spawn-crimson-blue-guard-debug 0)` / `(...  1)` to force-spawn a baton/gun guard in front of you (it reuses an
   inactive blue guard from the traffic pools, so stand in the Slums or a war zone);
   confirm it's textured and its idle/walk/run/notice/hostile/knocked/get-up/die animations all
   play correctly and match a regular guard's timing and sound cues 1:1.
6. **Passivity:** with no alert active, walk up to / bump a blue guard — it should not attack.
7. **No alarm on a general alert:** trigger a real city alert some other way (shoot a red guard,
   commit a crime). A nearby blue guard should stay passive toward Jak — it must not join the
   alert against him.
8. **Personal retaliation, no alarm:** hit/shoot a blue guard directly. It should react exactly
   like a red guard would (flinch/knockback/get-up animation, then fight back at normal range),
   but the city-wide alert (top-right alarm indicator) should **not** trigger from this.
9. **Death, collision, everything else:** kill a blue guard, get it hit by a vehicle, shocked
   (yellow hit), etc. It must look and behave identically to a red guard in every respect — same
   death animation, no different collision/attack range. Any difference here is a bug (most likely
   an animation-index drift — see the native-header/reorder pitfall in tip 23).
10. **Manual guard-vs-guard trigger:** `(define g (spawn-crimson-blue-guard-debug 0))` then
    `(crimson-blue-guard-attack-guards (the-as crimson-blue-guard g))` near a red guard — it should
    go hostile and fight. This combination has no native precedent (stock guards never fight each
    other), so pay attention to whether the approach/attack range looks normal.
11. Walk from the Slums into a loyalist district and confirm the blue guards are retired within a
    second or two, leaving only red and yellow guards.
12. Regression: boot a couple of other, untouched levels/cities and confirm no new spawn/link-art
    errors in `log/jak2.<ts>.log`.

## 8. Status

| Item | State |
|---|---|
| `build-actor :native-header #t` (C++ + GOAL macro) | ✅ done, compiled and boot-tested |
| `.glb` animation reordering | ✅ done, verified programmatically and in-game (correct animations play) |
| `crimson-blue-guard` entity (`deftype`/`defskelgroup`/`init-enemy!`) | ✅ done, compiled and boot-tested |
| DGO residency (code + art, 11 files) | ✅ done |
| Ambient traffic mixing | ✅ done, boot-tested |
| Circuit 2 (`models/common` + `task extract`) | ✅ done — guard renders fully textured |
| Passivity toward Jak + personal retaliation, no alarm | ✅ done, verified in-game |
| `crimson-blue-guard-attack-guards` manual trigger | ✅ done, verified in-game |
| Death dissolve sequence (`die` state override + `merc-death-spawn 73` + `knocked-fatal?`) | ✅ done, verified in-game: solves the C++ `generic_merc_death` exit status 5 crash via direct GOAL particle dissolution loop, plays `"enemy-fizz"`, hides mesh, and keeps knocked-down guards flat on the ground |
| City Peaceful patrol squads (2-3 members, formation navigation, adaptive speed, leader promotion) | not in this mod: City Peaceful ships in [`whozghiar/jak2-mod-peaceful-haven-city`](https://github.com/whozghiar/jak2-mod-peaceful-haven-city) |
| Squad mutual defense & Faction friendly-fire immunity | ✅ friendly-fire immunity done: blue members & projectiles are fully immune to friendly fire; squad mutual defense is inert here, since no squad forms without City Peaceful |
| Squad weapon loadout diversity | not in this mod: City Peaceful only |
| Faithful Crimson Guard combat AI | ✅ done, verified in-game: standoff distance (~6.5m–9m), reactive laser bursts/parabolic grenades as soon as LOS is acquired (up to 50m), evasive sideways rolls, emergency-only close attack (< 2.5m) followed by evasive recovery roll |
| "Mods" menu entry (L3 + SELECT ▸ `Mods ▸ crimson-blueguard-insurrection`: `Enable` + `War zones` + `War music`) | ✅ done; `Enable` rebuilds the city traffic and each war-zone pick flushes the city guards, so the new rules apply immediately |
| City Insurrection — nickname-based district zoning (`city-level-name-at-pos` → `city-district-of-level`) | ✅ done: Slums (`ctysluma/b/c`) = blue unless toggled into the war zone, the selected war-zone districts = conflict, everything else = red — verified level names, no hardcoded coordinates; probes only the traffic-engine's linked `level-data-array` grids (never a raw `*level*` bsp pointer — that crashed on the `ctyport→ctyinda` transition) |
| City Insurrection — **configurable war zone** (`*mod-city-conflict-mask*`) | ✅ done: `L3 + SELECT ▸ Mods ▸ crimson-blueguard-insurrection ▸ War zones` toggles any mix of Slums, Water Slums, Industrial (default), Port, Bazaars, Gardens, Main Town, Stadium and Palace, or All City; changing it re-zones and flushes the guards live |
| City Insurrection — strict per-zone spawning (single-faction pools, faction by district) | ✅ done: `traffic-object-spawn` picks blue in the Slums, red in Loyalist districts, 70/30 (loyalists/insurgents) in the war zone; a district change is reconciled incrementally (`mod-city-guard-pool-reconcile`, ≤2 wrong-faction retirements per pool per frame) so the pool is always the right faction — no filtering, no wasted slots |
| City Insurrection — war zone: no civilians/vehicles + dense guard battle | ✅ done: `want-count` for citizens (0–3), metalheads (8–10) and vehicles (11–19) forced to 0 in the war zone (drained by `kill-excess-once` + natural despawn); **two** guard pools — stock `crimson-guard-1` (`*mod-city-war-pool6-count*` 12) + the secondary `crimson-guard-0` (`*mod-city-war-pool4-count*` 10) — `inv-density-factor` 1.25 → 22 guards, 70/30, under the stock 64 nav ceiling; all restored on zone/mode change |
| City Insurrection — Loyalist district police density | ✅ the stock `crimson-guard-1` pool is left **byte-for-byte vanilla** in Loyalist districts (base `want-count`, alert-scaled `target-count`) |
| City Insurrection — autonomous inter-faction combat (`crimson-guard-insurrection-scan`) | ✅ done: red hunts blue / blue hunts red within **~150 m** (was 40 m) from `active` **and** `search`, full weapon AI, zero effect on Jak's wanted level; `find-nearest-enemy-guard` scans both trackers (the decomp's `citizen`/`vehicle` tracker aliases are swapped — guards are in `vehicle-tracker-array`) |
| City Insurrection — alert-free zones (`increase-alert-level` choke + `set-alert-level 0`) | ✅ done: no alert can start or persist in the Slums **or** the war zone, from any source — hitting a red guard in the war zone raises nothing; only loyalist districts run the wanted system |

## 9. "Mods" Menu Entry & Features

The retail-safe "Mods" popup menu (**L3 + SELECT**, in a normal launcher boot too — see
[`mods_menu.md`](../guides/mods_menu.md)) has one entry for this mod,
`crimson-blueguard-insurrection` (`goal_src/jak2/pc/features/crimson-blueguard-insurrection-menu.gc`):
**Enable** (City Insurrection, `*mod-city-insurrection?*`, off by default), a `War zones` submenu
and a `War music` submenu. It is freely reversible.

### 9.1 City Peaceful (not part of this mod)
City Peaceful ships in [`whozghiar/jak2-mod-peaceful-haven-city`](https://github.com/whozghiar/jak2-mod-peaceful-haven-city);
this mod keeps only its inert squad plumbing in `crimson-blue-guard.gc`. There, when toggled on:
- **Ambient Patrol Squads:** blue guards spawn in tight 2-to-3 member squads walking Haven City in
  formation (wingmen offset relative to the leader's rotation quaternion). Followers dynamically
  accelerate (up to 1.5×) or slow down (0.85×) to keep rank, and automatically promote follower 1
  to squad leader if the leader dies.
- **Weapon Diversity:** every 3-man squad features exactly one Taser guard (`guard-type 0`), one
  Rifle guard (`guard-type 1`), and one Grenade Launcher guard (`guard-type 2`). Every 2-man squad
  has two distinct weapons.
- **Mutual Defense:** if any squad member is attacked by Jak or another enemy, the entire squad
  retaliates together in self-defense, without triggering the city-wide alarm or calling red guards.
- **Friendly-Fire Immunity:** projectiles and attacks originating from blue guards are filtered out
  within the faction, preventing infighting or fratricidal aggro.
- **Faithful Combat AI:** ranged guards maintain standoff engagement distance, fire bursts or
  grenades upon acquiring LOS (up to 50m), and execute evasive sideways rolls (`roll-left` /
  `roll-right`). Melee rifle-butts are strictly an emergency counter (< 2.5m) immediately followed
  by an evasive roll to get back into a firing stance.

### 9.2 City Insurrection (✅ Fully Implemented)
Haven City becomes a three-front territorial civil war. Districts are classified by the **loaded
city-level name** that owns a position — `city-level-name-at-pos` → `city-district-of-level` →
`city-zone-from-level-name` in
[`mod-city-insurrection.gc`](../../../goal_src/jak2/levels/city/traffic/citizen/mod-city-insurrection.gc) — using
only verified level names (`level-info.gc`), never hardcoded map coordinates:

> [!WARNING]
> ### ⚠️ Work in Progress — Stability Notice
> City Insurrection is currently under **active development**. While fully playable, players and testers may encounter **occasional unexpected game crashes** (e.g. `exit status 5` / process allocation limits) due to the high density of concurrent combatants, process slot exhaustion or heap memory fatigue under sustained heavy battle, or level streaming crossfades.
> Detailed health telemetry is periodically printed to the console terminal to help monitor heap memory and active process slots.

| Zone | City levels | Rule |
|---|---|---|
| **Blue — Slums (Rebel Stronghold)** | `ctysluma`, `ctyslumb`, `ctyslumc` | 100% lone blue guards, random weapons; alert-free safe haven |
| **Red — Loyalist (Baron's districts)** | every district that is *not* the Slums or the selected war zone | 100% stock red/yellow Crimson Guards, **fully vanilla** density & policing toward Jak |
| **Conflict — War Zone** | the districts toggled in `L3 + SELECT ▸ Mods ▸ crimson-blueguard-insurrection ▸ War zones` — **Industrial (`ctyinda/b`) by default**, or any mix of Slums / Water Slums / Port / Bazaars / Gardens / Main Town / Stadium / Palace, or All City | **22 guards** (`*mod-city-war-pool6-count*` 12 + `*mod-city-war-pool4-count*` 10), dynamic 70% loyalists / 30% insurgents, **no civilians, no metalheads, no vehicles**; the two factions fight each other on sight; alert-free |

When toggled on in the Mods menu:
- **Configurable war zone** (`*mod-city-conflict-mask*`): the `War zones` sub-menu
  toggles districts in a bitmask — All City, Slums, Water Slums, Industrial (default), Port, Bazaars,
  Gardens, Main Town, Stadium and Palace — so several can be at war at once. Changing it
  re-zones the city and re-rolls the guards. The Slums are the blue haven unless they are toggled
  into the war zone.
- **Strict territorial spawning — single-faction pools, faction chosen by district**
  (`mod-ins-guard-spawn-blue?` + `mod-city-insurrection-shape-guard-pools`):
  `traffic-object-spawn` picks the concrete process type per spawn from the district Jak is in —
  `crimson-blue-guard` in the Slums, the stock red `crimson-guard` in Loyalist districts, a dynamic 70/30
  ratio in the war zone. A district change is reconciled **incrementally** — `mod-city-guard-pool-reconcile`
  retires up to 2 wrong-faction guards per pool per frame while `spawn-all` refills with the new
  faction, so the street crossfades over ~1-2 s and a red guard never ends up patrolling the Slums
  (nor a blue guard a Loyalist district).
- **War-Zone Guard Density (22 Active Combatants)** (`mod-city-insurrection-shape-guard-pools`):
  Two guard pools are mobilized: Pool 6 (`crimson-guard-1`, `*mod-city-war-pool6-count*` 12) and
  Pool 4 (`crimson-guard-0`, `*mod-city-war-pool4-count*` 10). Pool 7 (`crimson-guard-2`) can never
  spawn and is left off (§9.3). With `inv-density-factor` set to `1.25` (`*mod-city-war-density*`,
  4x denser spacing than retail) and continuous `fast-spawn #t`, the battlefield keeps up to 22
  simultaneous combatants.
- **Dynamic Faction Balancing (70% Loyalists / 30% Insurgents):**
  Street-level real-time balancing ensures loyalist forces (red & yellow guards) maintain tactical superiority
  over the rebel forces.
- **Arsenal Overhaul & Melee Minimization:**
  - **0% Tasers:** Taser/baton guards are completely disabled.
  - **Grenade Launchers for Red & Yellow Guards:** Red and yellow guards are equipped with high-explosive
    grenade launchers (`vehicle-grenade`) featuring parabolic ballistic trajectories alongside standard pulse rifles.
  - **Melee Minimization:** Rifle-butt melee swings are suppressed, the vanilla 10-meter shooting lockout
    is eliminated (point-blank fire allowed), and guards maintain tactical standoff distances in flanking arcs (~6.5m for rifle, ~9m for grenade launcher).
- **Periodic Health Telemetry Logs:**
  Console terminal prints heap memory, alive/free process slots, combatant counts, and pool states every 5 seconds:
  `[INS-METRICS] Heap: ... | Slots: ... | Combatants (act/tot): ... | P4(...) P6(...) P7(...) | Zone: conflict`
- **Crash fixed — district transitions are incremental** ([commit 1](../../../goal_src/jak2/levels/city/traffic/citizen/mod-city-insurrection.gc)):
  an earlier version force-deactivated every civilian + vehicle + hard-killed all three guard
  pools + fast-spawned on the frame Jak crossed a border — that coincides with the outgoing city
  level's teardown and hard-crashed the game (`exit status 5`, log ending at
  `kill #<level active ctysluma>`). Now a Slums ↔ loyalist crossover is spread over ~1-2 s at a
  few process ops per frame (`mod-city-guard-pool-reconcile`). Entering or leaving a war zone
  still runs a one-frame purge: `mod-city-flush-conflict-traffic` on the way in, pools 4 and 7
  killed on the way out.
- **Autonomous inter-faction warfare** (`crimson-guard-insurrection-scan` in `mod-city-insurrection.gc`): in the
  war zone every guard scans for the nearest **opposing-faction** guard within **~150 m** (the faction is derived
  from `this`, so one helper covers both red and blue). On acquisition it targets the foe directly
  and goes hostile — laser bursts, parabolic grenades — and **never touches Jak's
  wanted level**. The hook runs from both `active` and `search`, so a guard that loses a foe
  re-acquires the next nearest one or drops back to patrol instead of idling.
  `find-nearest-enemy-guard` scans **both** of the traffic engine's trackers — the decomp aliases
  `citizen-tracker-array` / `vehicle-tracker-array` onto the two `tracker-array` slots *backwards*
  (guards live in the one called `vehicle-tracker-array`).
- **Reciprocal retaliation:** a red guard hit (melee *or* projectile — `incoming attacker-handle`
  resolves a bolt/grenade back to the firing guard via the process parent chain) by a blue guard
  targets and returns fire on that blue guard directly, no city alarm, no siren. The blue guard
  side already had this.
- **Alert-free zones (Slums *and* war zone):**
  - `increase-alert-level` (`traffic-engine.gc`) is short-circuited whenever Jak is in the blue
    zone **or** the war zone — the **single choke point** for the alert rising, so it blocks the
    menu event, the direct `citizen::trigger-alert` path *and* kill-count escalation. Hitting a
    red guard in the war zone raises nothing. Only loyalist districts run the wanted system.
  - `mod-city-insurrection-update-traffic` additionally snaps `set-alert-level` to `0` on every
    frame Jak is in either zone, so any alert he *carried in* drops instantly.
  - Loyalist gunships (`guard-bike` 18, `hellcat` 19) are kept out of the Slums and the war zone
    (`want-count` 0). Hitting a blue guard still triggers only that guard's personal self-defense.
- **Live mode / config switching** (`crimson-blueguard-insurrection-menu.gc`), in two strengths:
  - `Enable` runs `mod-crimson-blueguard-insurrection-rebuild-guards`: a full `'kill-all` + `'spawn-all` on the traffic
    manager. This is the only thing that works, because the traffic engine allocates each
    pool's processes once at city load and then parks and reuses them — a `crimson-guard` that
    already exists keeps its GOAL type forever, so merely parking the pools handed the same
    red guards back and the mod appeared to do nothing until you reloaded a save. Destroying
    and rebuilding is what re-runs `*mod-city-guard-spawn-blue-hook*`. Guard vehicles and
    their riders come along for free. Standing in a war zone, the rebuild is followed by
    `mod-city-flush-conflict-traffic` to purge the non-combatants `'spawn-all` just recreated.
  - The `War zones` picker keeps the lighter `mod-crimson-blueguard-insurrection-flush-guards`: it
    only parks the three crimson-guard pools (4, 6, 7) and the guard vehicles (18, 19), or runs
    `mod-city-flush-conflict-traffic` when Jak stands in a war zone. The pooled processes
    already have the right types there, and district changes are handled incrementally by
    `mod-city-insurrection-update-traffic` — a full rebuild has no business fighting that.
- **Crash fixed (`ctyport → ctyinda` transition):** `city-level-name-at-pos` used to probe
  `sphere-in-grid?` on every loaded level's raw `(-> lev bsp city-level-info)` pointer. During a
  level transition an outgoing city level's `-vis` heap is freed while the traffic manager keeps
  running, so that probe walked freed memory → hard crash with no GOAL error. It now only probes
  the ≤2 grids the traffic engine has linked in `level-data-array` (the same set `update-traffic`
  uses) and recovers the level name by pointer identity.

### 9.3 War-zone load budget (and the crash it used to cause)

Entering a war zone killed the runtime within a couple of seconds. The logs in `log/` pin it
precisely: across every session there, `Zone: conflict` was only ever reached in the two runs that
died, and both stop mid-frame right after `Load music danger9` with no GOAL-level error — one of
them having first printed `Turns = 1150917018!!!` from `sprite-distort.gc`, i.e. the distort-sprite
renderer reading a float where an integer turn count belongs. That is corrupted data reaching the
sprite DMA path, not a `#f` dereference. Memory was not the constraint either (`Heap: 733/6160 KB`,
`Slots: 319/3072`). What the zone asked for per frame was:

| Knob | Was | Now | Where |
|---|---|---|---|
| Guards mobilized | 20+20+20 (really 40 — see pool 7 below) | `*mod-city-war-pool6-count*` 12 + `*mod-city-war-pool4-count*` 10 | `mod-city-insurrection.gc` |
| `inv-density-factor` | 0.1 (50x retail) | `*mod-city-war-density*` 1.25 (the engine's own dense preset) | `mod-city-insurrection.gc` |
| Red-guard re-arm | 0.2-0.5 s | 0.4-1.0 s | `guard.gc` `hostile:trans` |
| Blue-guard re-arm | flat 0.2 s, same value for every guard | 0.4-1.0 s, randomized | `crimson-blue-guard.gc` |
| Grenade launchers | 1 guard in 2 | 1 in 3, via `mod-city-roll-war-weapon` | `mod-city-insurrection.gc` |
| Grenade cooldown | **none** | 2 s per guard | `guard.gc` `crimson-guard-method-214` |

Together that is roughly an order of magnitude fewer projectiles, explosions and distortion sprites
in flight. The three count/density knobs are plain `define`s, so they can be raised from the REPL
— `(set! *mod-city-war-pool6-count* 16)`, then leave the district and come back — to find this
machine's ceiling.

**The grenade launcher is now one implementation.** `crimson-blue-guard` used to carry its own
`crimson-guard-method-214` override that was a verbatim copy of the parent's grenade branch minus
the `*mod-city-insurrection?*` gate and minus any cooldown, so every blue guard threw a grenade on
every shot. It reads native joint 14 (`"blast"`) exactly like the parent, so it bought nothing: it
is gone, and blue guards inherit `guard.gc`'s method-214. That one is rate-limited exactly like the
sibling branches' launcher (`jak2/features/crimson-blueguard/{peaceful,crimson-redguard-behavior}`,
where it lives on `crimson-blue-guard` behind a `grenade-launcher?` flag): same 8192 tilt, same
184320 gravity, same 4 s timeout, same lobbed-throw fallback when
`traj3d-calc-initial-velocity-using-tilt` finds no solution, and the same 2-second cooldown during
which the shoot animation still plays but nothing leaves the muzzle. The cooldown timestamp lives
in a new `grenade-last-time` field appended to the **end** of `crimson-guard`, so every retail
field keeps its offset.

**Pool 7 never worked.** `crimson-guard-2` (traffic-type 7) looks available — it has a tracker and
a `traffic-object-spawn` arm — but `lwide-activate` leaves its `level` `#f` for both `lwidea` and
`lwideb`, and `spawn-all` only arms `trtflags-3`, which spawning requires, for a type whose level is
currently active. The metrics confirmed it: `P7(0/0)` in a war zone asking for 20. It is no longer
mobilized, and the docstring no longer claims 60 combatants. Giving it a level in `lwide-activate`
is the way to actually use it, if the budget above ever allows a third pool.

> Not verified in-game yet: these are the fixes for every over-budget knob and every real defect
> found by inspection, but the crash itself has not been reproduced since. If a war zone still
> dies, halve `*mod-city-war-pool6-count*` / `*mod-city-war-pool4-count*` first — that isolates
> "too much of everything" from a specific bad actor in two runs.

---
*(AI-assisted)*

