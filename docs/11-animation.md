# 11. Animations (VCA Format, Preload, Playback, core:animator)

> **MANDATORY PREREQUISITES — DO NOT SKIP.** This file assumes you have read
> [01-engine-overview.md](01-engine-overview.md) AND
> [08-pitfalls.md](08-pitfalls.md). 01 defines the ID rules, reserved names
> and trust labels this file relies on; 08 lists the real failure modes
> behind every DO/AVOID note. Skipping either is the most common reason
> models generate broken packs from this guide. If you received only this
> file, fetch and read both before writing any code.

This document covers the animation system of VoxelCore: the text-based VCA
format (`.vca` files), how tracks are loaded through `preload.json`, how to
play them from Lua (`animation.play`), and the built-in state-machine
component `core:animator`. It continues from 10-entities.md: entity
skeletons (bones) are described there, and every bone referenced by an
animation must exist in the entity's skeleton file.

What is OUT OF SCOPE here: creating/exporting animations in external tools,
the VCM 3D-model format itself (see 14-models.md), and camera
internals beyond playing a track on a camera.

## Mental model

- An **animation track** is a `.vca` text file in the pack folder `animation/`
  (singular spelling — an engine constant, do NOT "fix" it to `animations`).
  `[VERIFIED: engine source, ANIMATION_FOLDER]`
- A track is a set of **directives** (`@move`, `@rotate`, `@scale`, `@zoom`,
  `@texture`) that define scalar curves over time. Directive ORDER in the
  file does not matter — the engine applies transforms by channel and axis,
  not by file order. `[DOC]`
- Tracks are applied to a **target**: an entity skeleton (`entity.skeleton`)
  or a camera. Bone-targeted directives name the bone via the `bone`
  attribute. `[DOC]`
- Playback is a CLIENT-SIDE visual effect on the render skeleton. Playing a
  track mutates bone matrices of the local renderer; it does not change
  entity physics or position. `[VERIFIED: engine source, pose matrices]`
- The `.vca` parser is implemented in Lua inside the engine
  (`res/modules/internal/formats/vca.lua`) and compiles each track into a
  generated Lua function. Parse errors are LOUD (they throw); runtime lookups
  can be silent (see pitfalls). `[VERIFIED: engine source]`

## Loading tracks — preload.json and the FLAT namespace

Animations are NOT loaded automatically. A pack must list them in its
`preload.json` under the `"animation"` key. `[DOC: assets-preload]`

```json
{
    "animation": ["hit", "rip"]
}
```

How the engine resolves an entry — read this carefully, it defines the
naming rules: `[VERIFIED: engine source, AssetsLoader.cpp / assetload_funcs.cpp]`

1. The preload entry (e.g. `"hit"`) is used VERBATIM as the track name and as
   a path under the pack folder: the engine searches all content sources for
   `animation/hit.vca`.
2. The name is NOT prefixed with the pack id. Unlike blocks and items
   (`my_pack:my_block`), animation names live in ONE GLOBAL FLAT namespace:
   `animation.play("hit", ...)` — bare name, no colon.
3. If the file does not exist, preload SILENTLY SKIPS it (no error at load
   time). The failure surfaces later: `animation.play` on a missing track
   logs an engine error `animation track not found` and the playback entry is
   dropped.

DO namespace your track names: use subfolders in the preload entry and the
folder tree — `"animation": ["guns/recoil", "mobs/imp_idle"]` loads
`animation/guns/recoil.vca` under the name `guns/recoil`. Longer unique names
are the ONLY protection against cross-pack collisions.

- AVOID short generic track names like `"hit"`, `"idle"`, `"walk"`. Another
  pack preloading the same name overwrites yours (the engine stores tracks
  by bare name, last load wins), and your `animation.play("hit")` may even
  resolve to the OTHER pack's file — name resolution searches all content
  sources, not just your pack. `[VERIFIED: engine source]`
- Real-world example: wooden_meat_game preloads `"hit"`, `"rip"` — short and
  collision-prone; it simply got lucky. Do not copy that naming.

Runtime loading from a string is also available (advanced):
`assets.parse_animation("vca", content_string, "my_name")` — parses a VCA
text and registers it under the given name. `[DOC: libassets]`

## VCA syntax primer

VCA uses the VCM text syntax (the engine's model-format syntax): directives
look like XML tags, written `@tag attr value attr2 "quoted value"` with an
optional `{ ... }` block of child directives. `[DOC: vcm.md]`

- Attributes take bare words (`curve linear`), quoted strings
  (`bone "body"` — quotes optional for single words), parenthesized vectors
  (`origin (0.5,0.5,0.5)` — not used by VCA), or `{ ... }` blocks (children).
- Indentation and line breaks are insignificant; one directive per line is
  the convention. The example repo uses TAB indentation — this guide shows
  4 spaces; both parse identically.
- A `.vca` file has no explicit root element; the whole file is one implicit
  `track`. `[VERIFIED: engine source, xml.parse_vcd(source, "track")]`

## Directive reference

| Directive | Applies to | Meaning |
| --- | --- | --- |
| `@move` | skeleton bone, camera, object | Translate by offset, per axis |
| `@rotate` | skeleton bone, camera, object | Rotate by angle, per axis — DEGREES |
| `@scale` | skeleton bone, object | Scale multiplier, per axis |
| `@zoom` | camera only | Camera zoom multiplier |
| `@texture` | skeleton only | Swap a dynamic texture slot over time |

`[DOC: vca.md]`

Common attributes: `[DOC + VERIFIED: parser]`

- `bone "name"` — target a bone of the skeleton by name. Directives WITHOUT
  `bone` target the whole object (camera/object transforms); `@texture`
  instead uses `name "$slot"` to pick the texture slot.
- `by x` / `by y` / `by z` — which scalar the curve drives. MUST be exactly
  one letter `x`, `y` or `z` (anything else is silently computed and never
  applied). `[VERIFIED: engine source]`
- `curve const|linear|bezier` or `curve .my_custom` — keyframe curve type
  (see below).
- `func (expression)` — expression curve instead of keyframes (see below).
- `period N` — loop THIS line every N frames (see below).

One line = one directive = one scalar curve. A bone can have up to three
`@move` lines (`by x/y/z`), three `@rotate` lines, three `@scale` lines.

- AVOID two lines of the same channel+axis on the same bone (e.g. two
  `@move bone "body" by y`). The engine keeps only the LAST one — the
  earlier line is silently discarded. `[VERIFIED: engine source, codegen]`

## Time model — configure, frames, duration (read twice)

Meta-information is set with a `@configure` directive; it applies GLOBALLY
no matter where it sits in the file, but DO put it at the top:

```vca
@configure fps 30 frames 15 rotation-order XYZ
```

`[DOC: vca.md]`

- `fps` — frame rate DIVISOR for duration computation. Default: `60`.
- `frames` — frame count (DIVIDEND). Track duration = `frames / fps`
  seconds. A keyframe `frame N` therefore means "at N ticks of 1/fps
  seconds": with default fps 60, frame 12 = 0.2 s; with fps 30, frame 12 =
  0.4 s. `[DOC + VERIFIED: engine source, DEFAULT_FPS = 60]`
- `duration` — explicit duration in seconds; overrides `frames`. The
  upstream docs note that keyframe curves REQUIRE `fps` (or `frames`) via
  configure or explicit `duration`.
- `rotation-order` — Euler application order (`XYZ`, `ZYX`, ...) for
  `@rotate` lines, because file order is NOT honored. Default `XYZ`.

**The dangerous default: a track with NO `configure` never ends.**
`[VERIFIED: engine source]` The parser initializes duration to `math.huge`
and only a `configure` directive (or explicit `duration`) shrinks it. The
playback loop stops a track only when its timer exceeds duration — so a
track without configure runs FOREVER, holding the pose of its last keyframe
values (keyframe curves clamp to the last value; expression curves keep
evaluating forever).

This is not always a bug — it is a pattern: wooden_meat_game's `rip.vca`
(death animation) intentionally has no configure, so the entity stays in its
fallen pose permanently. For action clips (recoil, hits) the same default
just means the clip freezes at its final values — which is harmless ONLY if
the final values are the rest pose (0).

DO end action keyframes at the rest pose (value 0) unless you intentionally
freeze a pose.
DO give looping idle/walk tracks either no configure plus `period` on each
line, or a `configure` and a driver that loops (see core:animator).

## Keyframe curves (curve const / linear / bezier)

A keyframe curve drives `f(t)` by interpolating between `@key` directives in
the directive's `{ ... }` block:

```vca
@rotate bone "body" by z curve linear {
    @key frame 0 value 0
    @key frame 6 value 45
    @key frame 12 value 0
}
```

Curve types: `[DOC: vca.md]`

- `const` — no interpolation: the value jumps at each keyframe.
- `linear` — piecewise-linear interpolation (the everyday choice).
- `bezier` — cubic Bezier between keys with explicit tangents; a bezier key
  carries `lx ly rx ry` — tangent handle OFFSETS relative to the key's own
  frame (`lx`, `rx`) and value (`ly`, `ry`). `[VERIFIED: engine source,
  codegen adds them to the key's frame/value]` Hand-writing bezier keys is
  rare — exporters produce them.

Each key needs `frame N` (in fps ticks) and `value V` (a number — EXCEPT the
`@texture` directive, whose values are texture name strings, stepped
const-style). `[DOC + VERIFIED]`

Texture example from the upstream docs (slot `$0`, stepping between two
faces of a tireman entity):

```vca
@texture name $0 {
    @key frame 0 value entities/tireman:face_0
    @key frame 24 value entities/tireman:face_1
}
```

Looping one line: add `period N` (in frames) to wrap that line's time
modulo N — useful for endless idle motions in a track without `configure`:

```vca
@move by y curve linear period 24 {
    @key frame 0 value 0
    @key frame 12 value 1
    @key frame 24 value 0
}
```

`[DOC: vca.md]`

- Keyframes do NOT need to start at frame 0; before the first key the curve
  clamps to the first value.
- DO keep keyframe `frame` numbers ordered and within `frames` from
  `configure` if you set one.

## Expression curves (func)

Instead of keyframes, a line can define `f(t) = value` directly, where `t`
is time in SECONDS:

```vca
@move bone "antenna" by y func (sin(t * 2.5) * 0.1)
```

`[DOC: vca.md]`

Expressions are transpiled into a Lua function body. Documented operators:
`+ * / % ^`; constants `pi`, `e`. `[DOC]`

The ACTUAL function set available inside VCA expressions (verified against
the engine's codegen environment — this differs from the upstream doc!):
`[VERIFIED: engine source, animation_codegen.lua]`

| Available | Notes |
| --- | --- |
| `sqrt min max floor ceil abs sign round exp` | standard math |
| `sin cos tan` | trigonometry (radians) |
| `deg rad log log10` | unit conversion, logarithms |
| `noise(x, octaves)` | 1D smooth pseudo-noise, range about -1..1 |
| `pi e` | constants |
| `t` | time in seconds (only in `func` lines) |
| `kl kr` | left/right keyframe objects (`.value`, `.frame`) — only in custom curves |

Documented but NOT actually available — using them raises a runtime nil-call
error when the track plays: `[VERIFIED: engine source — not in the codegen
environment]`

- `rand(low, high)` — listed in `doc/*/vca.md`, absent from the codegen env.
- `noise2d(x, y, octaves)` — the math library exports `noise2d`, but the
  codegen binds a name that does not exist, so neither `noise2d` nor
  `noise2` works in VCA expressions.

DO verify any function beyond the table above with a quick in-engine test
before relying on it — the upstream doc and the code disagree here.

## Custom curves (@curve)

Declare a reusable interpolation curve, then reference it with a leading
dot: `[DOC: vca.md]`

```vca
@curve ease-in-out func (kl.value + (kr.value - kl.value) * (t * t * (3 - 2 * t)))

@move bone "body" by y curve .ease-in-out {
    @key frame 0 value 0
    @key frame 12 value 1
}
```

Inside `@curve func`, `t` is the 0..1 position BETWEEN the two neighboring
keys, `kl`/`kr` are the left/right keyframes (`kl.value`, `kr.value`).
`[DOC + VERIFIED]`

## Units and transform semantics

- `@rotate` values are **DEGREES**. The engine's `mat4.rotate` converts to
  radians internally: `value 90` is a quarter turn. `[VERIFIED: engine
  source, libmat4.cpp applies glm::radians]` (The upstream docs never state
  the unit.)
- `@move` values are in world blocks — the same units as skeleton node
  offsets (see 10-entities.md). Values like `0.25` are typical for a recoil
  kick; `hit.vca` below uses exactly that.
- `@scale` and `@zoom` are multipliers (`1` = unchanged).
- Multiple `@rotate` lines on one bone are applied in `rotation-order`
  (default `XYZ`), NOT in file order. `[DOC + VERIFIED]`
- Move/rotate/scale compose as a matrix applied to the bone's bind-pose
  matrix each frame.

## Playing tracks — animation.play

The `animation` library (global) is small: `[DOC: libanimation.md,
RU-only upstream]`

```lua
animation.play(name, target)  -- returns a playing-track handle
animation.action(func)        -- run a coroutine outside pause, dies on world exit
```

`animation.play` in detail: `[VERIFIED: engine source, animation.lua]`

```lua
local handle = animation.play("guns/recoil", entity.skeleton)
```

- The target's pose is RESET (to bind pose) at play time, then re-reset and
  re-applied EVERY frame while the track runs.
- Returns a handle object with UNDOCUMENTED control methods:
  `handle:stop()` (stop and reset timer), `handle:pause()`,
  `handle:resume()`. `[OBSERVED: engine source]`
- Playing a second track on the same target does not stop the first, but the
  newer track applies LATER each frame and its pose reset wipes the older
  track's output — in practice the last-played track wins visually.
  `[VERIFIED: engine source, playback loop order]`
- Playing an unknown name logs `animation track not found` and does nothing.

Targets: `[DOC: animation.md, RU-only upstream]`

```lua
-- 1. An entity skeleton (the normal case):
animation.play("mobs/imp_walk", entity.skeleton)

-- 2. A camera:
local camera = cameras.get(cameras.index("my_pack:cutscene_cam"))
animation.play("cutscenes/flyover", camera)
```

Camera targets use directives WITHOUT `bone`: `@move`/`@rotate` drive the
camera transform, `@zoom` drives zoom. `[DOC + VERIFIED: codegen camera path]`

The first-person hand skeleton is also addressable via
`gfx.skeletons.get("hand")` — same API surface as entity skeletons (see
`doc/*/scripting/builtins/libgfx-skeletons.md`). `[DOC]`

Where to call `play` from: any client-side Lua — component scripts
(`health.lua` in wooden_meat_game plays `hit` when damaged), HUD scripts, or
layout companion scripts (see 09-hud-and-ui.md). `[OBSERVED]`

## Runtime behavior every author must know

`[VERIFIED: engine source, animation.lua]` unless noted:

- Tracks advance per RENDERED frame (`time.delta()` per frame), so playback
  is a client-side visual only; pausing and world changes are handled by the
  engine (tracks are unloaded with the world).
- While a track runs, the target pose is reset to bind pose EVERY FRAME and
  only the animated channels are applied on top. Manual bone edits (e.g.
  `skeleton:set_matrix`) made elsewhere get overwritten while a track plays.
- When a track ENDS (timer exceeds duration), the pose stays at the last
  applied state — there is NO automatic return to rest. Either end your
  keyframes at rest values, or call `entity.skeleton:reset_pose()` yourself.
- There is no built-in "queue" or "on finish" callback. For state-driven
  characters use `core:animator` (below) or drive tracks from your own
  component.

## core:animator — the built-in state machine

For skeletons that switch between looping states (idle/walk/attack), the
engine ships a component: `core:animator`. The upstream doc marks it as
"under development" — treat its interface as provisional. `[DOC + OBSERVED:
engine source, res/scripts/components/animator.lua]`

What it does (from the engine's own script): `[VERIFIED: engine source]`

- Maintains named STATES, each bound to a track name.
- On `set_state(name, fade_time)` it cross-fades: the new state fades in and
  the old one fades out over `fade_time` seconds using the track function's
  `intensity` parameter (multiplier channels fade correctly, additive
  channels scale toward 0).
- On every render frame it resets the skeleton pose and evaluates ALL active
  states, looping each track's time modulo its duration — so state tracks
  loop automatically regardless of `configure`.

DO use `core:animator` for mob-style looping states instead of hand-rolling
play/stop bookkeeping; keep one-shot action clips (hit, death) on plain
`animation.play` from your gameplay scripts, as wooden_meat_game does.

Manual rig manipulation (no VCA at all) is also legitimate — the base
pack's `player_animator` component positions the held item every frame with
`rig:index("item")`, `rig:set_model(index, ...)`, `rig:set_matrix(index,
mat4.rotate({0, 1, 0}, -80))`. Mix deliberately, not accidentally: while a
VCA track plays, its per-frame pose reset erases manual `set_matrix` edits.

## Real annotated examples (verbatim, wooden_meat_game)

Registered in `content/my_pack/preload.json`:

```json
{
    "animation": ["hit", "rip"]
}
```

`hit.vca` — one-shot recoil on the `body` bone; NOTE: no `configure`, so the
track technically runs forever while clamping to the last (rest) values:

```vca
@rotate bone "body" by y curve linear {
    @key frame 0 value 0
    @key frame 2 value 7
    @key frame 6 value -7
    @key frame 10 value 7
    @key frame 12 value 0
}

@move bone "body" by y curve linear {
    @key frame 0 value 0
    @key frame 6 value 0.25
    @key frame 12 value 0
}
```

Line by line: `[OBSERVED]`

- `@rotate ... by y` — yaw shake: +7 degrees at frame 2 (= 2/60 s at default
  fps 60), over-corrects to -7, settles at 0 by frame 12 (= 0.2 s).
- `@move ... by y` — a 0.25-block lift peaking mid-clip, back to 0.
- Both lines end at value 0 — the rest pose — so the infinite-duration
  default is harmless here.

`rip.vca` — death clip that FREEZES the fallen pose on purpose:

```vca
@rotate bone "body" by x curve linear {
    @key frame 0 value 0
    @key frame 18 value 90
}

@move bone "body" by y curve linear {
    @key frame 0 value 0
    @key frame 18 value 0
    @key frame 19 value -100
}
```

- Rotates the body 90 degrees around X (falls over — degrees, as verified
  above), then yanks it 100 blocks down at the last frame (sinks it out of
  sight). No configure: the pose persists forever. That is the intended
  "rest in peace" behavior.

Playback, from the pack's health component:

```lua
animation.play("hit", entity.skeleton)
```

- Bare track name (flat namespace), entity skeleton as target. The full
  component context is in 10-entities.md.

## Pitfalls

1. **Plural spellings.** Folder `animations/` or preload key `"animations"`
   load nothing. The engine constant is `animation` (singular).
   `[VERIFIED: engine source]` (Also listed in 08-pitfalls.md.)
2. **Expecting pack prefixes.** `animation.play("my_pack:hit", ...)` finds
   nothing — track names are bare preload entries in a flat global
   namespace. `[VERIFIED: engine source]`
3. **Inventing fields.** A typo in a directive or attribute name is a LOUD
   parse error (`unknown directive ...`) the moment the pack loads — the
   opposite of entity JSON, where unknown fields are silently ignored. But
   attribute VALUE typos can stay silent: a wrong `by` letter computes and
   applies nothing. `[VERIFIED: engine source]`
4. **Bone name typos.** `skeleton:index(name)` returns nil for an unknown
   bone; in the Lua fallback path nil becomes bone index 0 and the animation
   silently animates the FIRST bone. No error is thrown. Check bone names
   against your `skeletons/<name>.json`. `[VERIFIED: engine source]`
5. **Trust the duration default.** Without `configure` the track never ends
   (duration = infinity). For one-shot clips always end keyframes at rest
   values — or the pose freezes mid-motion forever.
6. **Playing a track that was never preloaded.** Preload silently skips
   missing files; `play` then errors `animation track not found` at runtime.
   Register the track in `preload.json` and check the spelling.
7. **Assuming auto-return or auto-loop.** Neither exists in `animation.play`:
   finished tracks keep their last pose, and looping only happens via
   `period`, an infinite default, or core:animator's modulo.
8. **Calling `rand`/`noise2d` in expressions.** They are documented upstream
   but absent from the codegen environment — the track fails at play time.
   Use the verified function table above. `[VERIFIED: engine source]`
9. **Two same-channel lines on one bone.** The last one silently wins; split
   complex motion across bones or merge keys into one line.

## DO / AVOID summary

- DO list every `.vca` in `preload.json` under `"animation"` (singular).
- DO namespace track names (`"guns/recoil"`, `"mobs/imp_walk"`) — the
  namespace is global and collision-prone.
- DO put a `@configure` at the top when the track must END; end action clips
  at rest values otherwise.
- DO end keyframes at value 0 for one-shot clips unless freezing is intended.
- DO use `core:animator` for looping character states; keep one-shots on
  `animation.play`.
- DO recheck bone names against the skeleton file — typos animate bone #0
  silently.
- AVOID `animations` (plural) anywhere: not the folder, not the key.
- AVOID pack prefixes in `animation.play` names.
- AVOID relying on `rand`, `noise2d` or any function outside the verified
  expression table.
- AVOID two lines of the same channel+axis on the same bone — the last one
  silently replaces the first.

## Upstream pointers (read before writing real animations)

The paths below are files of the ENGINE's own repository
(github.com/MihailRis/voxelcore, folder `doc/`) — NOT files of this guide.
The Russian documentation is usually MORE complete and more current; when the
two diverge, prefer `doc/ru/`:

| Topic | English (`doc/en/`) | Russian (`doc/ru/`) — prefer this |
| --- | --- | --- |
| Animation system overview | — (no EN file) | `doc/ru/animation.md` |
| VCA format reference | `doc/en/vca.md` | `doc/ru/vca.md` |
| VCM syntax (shared grammar) | `doc/en/vcm.md` | `doc/ru/vcm.md` |
| Skeletons, rigging, bone format | `doc/en/rigging.md` | `doc/ru/rigging.md` |
| `animation.*` Lua API | — (no EN file) | `doc/ru/scripting/builtins/libanimation.md` |
| Skeleton object API (`set_texture`, ...) | `doc/en/scripting/builtins/libgfx-skeletons.md` | `doc/ru/scripting/builtins/libgfx-skeletons.md` |
| Camera targets (`cameras.*`) | `doc/en/scripting/builtins/libcameras.md` | `doc/ru/scripting/builtins/libcameras.md` |
| Preload categories | `doc/en/assets-preload.md` | `doc/ru/assets-preload.md` |

Where this guide's claims go beyond the upstream text (degrees in `@rotate`,
the infinite-duration default, the flat namespace, the exact expression
function table, the returned handle methods), they were verified against the
engine's source code and Lua internals at the time of writing — re-verify
against your engine version if behavior seems to differ.

Real usage to learn from: `wooden_meat_game` `content/my_pack/` —
`animation/hit.vca`, `animation/rip.vca`, `preload.json`, and
`scripts/components/health.lua` (one `animation.play` call). The engine's
own `res/scripts/components/animator.lua` is the reference implementation of
the state-machine pattern.
