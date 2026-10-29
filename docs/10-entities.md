# 10. Entities (Definitions, Skeletons, Spawning, Components)

> **MANDATORY PREREQUISITES — DO NOT SKIP.** This file assumes you have read
> [01-engine-overview.md](01-engine-overview.md) AND
> [08-pitfalls.md](08-pitfalls.md). 01 defines the ID rules, reserved names
> and trust labels this file relies on; 08 lists the real failure modes
> behind every DO/AVOID note. Skipping either is the most common reason
> models generate broken packs from this guide. If you received only this
> file, fetch and read both before writing any code.

This document covers the PACK-SIDE basics of entities: JSON definitions in
`entities/`, skeletons in `skeletons/`, spawning from items or Lua, and the
minimal shape of a custom component. Bone ANIMATION is covered in
11-animation.md; rigging internals and the deep ECS API stay OUT OF SCOPE —
pointers are at the end.

## What entities are (and are not)

Entities are simulated objects with their own physics body and free 3D
position: dropped items, falling blocks, mobs, projectiles, the player. They
are NOT blocks (no grid cell) and NOT items (no inventory stack). `[DOC]`

- Entity definitions are JSON files in `entities/` of the pack. The FILE NAME
  is the entity name: `entities/drop.json` in pack `base` becomes `base:drop`
  — the same file-name rule as blocks and items. `[VERIFIED: engine loader +
  stock pack]`
- The stock pack ships three: `base:drop` (dropped item), `base:falling_block`
  (sand-like gravity block), `base:player`. `[OBSERVED]`

## Entity definition reference (`entities/<name>.json`)

The engine loader reads EXACTLY the keys below and SILENTLY IGNORES everything
else — a typo in a field name never throws, it just does nothing. `[VERIFIED:
engine source, EntityLoader.cpp]`

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `parent` | string | — | Inherit another entity def (see below) |
| `components` | list | `[]` | Components, in init order (see below) |
| `hitbox` | `[w, h, d]` | — | Physics box size in blocks |
| `body-type` | string | `dynamic` | `static` / `kinematic` / `dynamic` |
| `blocking` | bool | `true` | Blocks block placement inside hitbox |
| `selectable` | bool | `true` | Cursor ray hits it (`on_attack`/`on_use`) |
| `solid` | bool | `false` | Hitbox is a tangible obstacle |
| `mass` | number | `1.0` | Mass in kg (`kinematic` = infinite) |
| `elasticity` | number | `0.0` | Bounciness |
| `step-height` | number | `0.5` | Max auto-step height without jumping |
| `sensors` | list | `[]` | Trigger volumes (see below) |
| `skeleton-name` | string | `pack:<entity name>` | Which skeleton renders this entity |
| `material` | string | — | Same material names as blocks |
| `lighting-mode` | string | `solid` | `solid` / `bone-based` |
| `save` | bool | `true` | Save the entity into the world at all |
| `save-skeleton-pose` | bool | `false` | Save bone pose |
| `save-skeleton-textures` | bool | `false` | Save dynamically assigned textures |
| `save-body-velocity` | bool | `true` | Save velocity |
| `save-body-settings` | bool | `true` | Save modified body settings |

`[VERIFIED: engine source; defaults cross-checked with EntityDef.hpp]`

### `parent` — inheritance (UNDOCUMENTED upstream; use only when necessary)

`"parent": "base:some_entity"` clones the parent definition first, then the
own fields override the inherited values. The parent is a REGISTERED entity
name (`pack:name`), not a file path. `[OBSERVED: engine source]`

**Treat this as an opt-in mechanism, not a recommended default.** It is an
undocumented, lightly-exercised feature, and most pack authors do not use it
— copying the JSON and editing the copy is the common, practical approach.
Think of it as inheritance in OOP: it looks like the "clean" solution, but a
bad setup produces exactly the kind of bugs OOP inheritance is notorious
for. The effective definition is a merge spread across two files, so when
something breaks you are debugging a result you cannot see in one place; if
the parent is a `base:` entity, an engine update can also silently change
your entity. Use `parent` only by necessity — e.g. many near-identical
variants that must stay in sync with one base you fully control — and never
"for cleanliness". Avoiding the field entirely is a perfectly valid safety
choice: authors who skip it never have to hunt for which inherited field
caused the bug.

### `components`

Either plain strings (`"base:drop"`) or objects with `name` + `args`
(`{"name": "base:drop", "args": {"item": "base:stone.item", "count": 1}}`).
The list order IS the initialization order. `args` values arrive in the
component's `ARGS` table both on fresh spawn AND on load from a save.
`[DOC]`

### `body-type`

- `dynamic` (default) — physics moves and collides the body.
- `static` — physics never moves it.
- `kinematic` — moves, but ignores collisions; treated as infinite mass.

**An invalid value is SILENTLY IGNORED** — the entity keeps the default
`dynamic`. Real bug: wooden_meat_game's `entities/nepis.json` says
`"body-type": "rigidbody"` — not a valid name, silently treated as `dynamic`.
`[VERIFIED: engine source, enum registry]`

### `sensors`

Trigger volumes attached to the body. Two shapes: `["radius", r]` and
`["aabb", minx, miny, minz, maxx, maxy, maxz]` (the box FOLLOWS the entity
rotation but does not rotate itself). Sensors are indexed from 0 in list
order; entering/exiting bodies fire `on_sensor_enter`/`on_sensor_exit` events
in components. An unknown shape name is logged as an engine error and the
sensor is skipped — the entity still loads. `[DOC + VERIFIED: engine source]`

### `solid` and `mass`

`"solid": true` makes the hitbox a tangible obstacle for other entities.
For `solid` DYNAMIC bodies DO specify `mass` explicitly; `kinematic` bodies
behave as infinite mass regardless of the field. `[DOC]`

## Real annotated examples (verbatim)

`base:drop` — a dropped-item entity: pickup sensors, drop component with
args, non-blocking:

```json
{
    "components": [
        {
            "name": "base:drop",
            "args": {
                "item": "base:stone.item",
                "count": 1
            }
        }
    ],
    "hitbox": [0.4, 0.25, 0.4],
    "sensors": [
        ["radius", 0.3],
        ["radius", 1.6]
    ],
    "blocking": false
}
```

`base:falling_block` — reuses the shared `block` skeleton, solid hitbox:

```json
{
    "components": [
        "base:falling_block"
    ],
    "skeleton-name": "base:block",
    "hitbox": [0.98, 0.98, 0.98],
    "solid": true
}
```

`my_pack:nepis` (wooden_meat_game, verbatim) — custom components from the
pack, WITH the real-world `"rigidbody"` bug in `body-type`:

```json
{
    "components": [
        "my_pack:health",
        "my_pack:enemy",
        {
            "name": "my_pack:patrol_ai",
            "args": {
                "patrol_size": 10
            }
        }
    ],
    "hitbox": [1.0, 1.0, 1.0],
    "body-type": "rigidbody"
}
```

DO NOT copy that `body-type` line: the engine silently ignores it. The valid
spelling for "a physics mob" is no field at all (`dynamic` is the default).
`[OBSERVED + VERIFIED: engine source]`

## Skeletons (`skeletons/<name>.json`)

A skeleton is the render model tree of an entity. Skeleton files are
pack-namespaced (`skeletons/drop.json` in `base` -> `base:drop`) but are NOT
indexed content: they are looked up by `skeleton-name` at render time.
`[DOC: doc/ru/rigging.md]`

```json
{
    "root": {
        "name": "body",
        "model": "turret",
        "offset": [-0.5, -0.5, -0.5],
        "nodes": [
            {
                "name": "gun_hand",
                "offset": [0.5, 1.0, 0.5],
                "nodes": [
                    { "name": "gun", "model": "shotgun", "offset": [0, -0.2, 0] }
                ]
            }
        ]
    }
}
```

(verbatim `my_pack:turret` skeleton, wooden_meat_game)

- Every node field is optional: `name` (for script access), `model` (model
  name from the pack's `models/` folder), `offset` `[x, y, z]` relative to the
  parent, `nodes` (children). `[DOC]`
- `"model"` is a NAME without prefix or extension; models are loaded
  automatically — DO NOT add them to `preload.json`. Formats: `obj`, `vec3`,
  `vcm`; `.mtl` of OBJ files is ignored, the texture is chosen by material
  name. `[DOC: doc/ru/rigging.md]`
- `skeleton-name` defaults to `pack:<entity name>`: an entity `my_pack:nepis`
  renders from `my_pack/skeletons/nepis.json` unless another skeleton is set.
  DO ship a skeleton for every visible entity or set `skeleton-name`.
  `[DOC + OBSERVED: every entity in base and wooden_meat_game has one]`
- Moving/bones/animating them is script-only; VCA animations are covered in
  11-animation.md. `[DOC]`

## Spawning entities

### Way 1 — spawn-egg item (declarative + one function) `[OBSERVED]`

The item file is icon-only; behavior lives in the item's script (script name
defaults to the item name — see 04-items.md):

```json
{
    "icon-type": "sprite",
    "icon": "items:nepis"
}
```

`scripts/spawn_nepis.lua` (verbatim, wooden_meat_game):

```lua
local spawn_entity_name = "my_pack:nepis"

function on_use_on_block(x, y, z, playerid, normal)
    entities.spawn(spawn_entity_name,
        { x + 0.5 + normal[1], y + 0.5 + normal[2], z + 0.5 + normal[3] })
end
```

An icon-only item WITHOUT such a script does nothing when used. `[OBSERVED]`

### Way 2 — `entities.spawn` from any Lua `[DOC]`

```lua
local e = entities.spawn("my_pack:nepis", {x, y, z})
-- optional 3rd argument overrides component ARGS:
-- {["my_pack__patrol_ai"] = {patrol_size = 5}}
-- note the DOUBLE underscore: pack id + "__" + component name
```

Useful registry functions (`[DOC: doc/ru/scripting/builtins/libentities.md]`):
`entities.get(uid)`, `entities.exists(uid)`, `entities.def_index("pack:name")`,
`entities.def_name(index)`, `entities.def_hitbox(id)`, `entities.def_solid(id)`,
`entities.get_all()`, `entities.get_all_in_box(pos, size)`,
`entities.get_all_in_radius(center, radius)`, `entities.raycast(...)`.
An entity handle also exposes `entity:despawn()`, `entity:get_uid()`,
`entity:def_name()`. `[OBSERVED: wooden_meat_game components]`

**Iteration trap:** the `entities.get_all*` functions return tables with
INTEGER keys (uid -> entity). Iterate with `pairs`, measure with
`table.count_pairs`; `ipairs` and `#t` are WRONG there. `[DOC]`

### Built-in components you will meet

- `core:mob` (arg `jump_force`), `core:player`, `core:pathfinding` —
  `[DOC: doc/ru/scripting/core_components.md, RU-only file]`
- `base:drop` (args `item`, `count`), `base:falling_block`,
  `base:player_animator` — `[OBSERVED: stock pack entity files]`
- The player entity is bound in `config/defaults.toml`:
  `player-entity = "base:player"`. `[OBSERVED: stock pack config]`

## Your own components (`scripts/components/`)

A component is a Lua file `scripts/components/<name>.lua`; it is addressed as
`pack:name` in the entity's `components` list. `[DOC + OBSERVED:
wooden_meat_game]`

Environment available inside a component:

- `ARGS` — the `args` table from the entity definition (or the
  `entities.spawn` override).
- `SAVED_DATA` — the component's persisted table; write what must survive a
  save/load cycle.
- `this` — the component's own function table (expose API for other
  components: `this.take_damage = ...`).
- `entity` — the entity handle (`entity.skeleton`, `entity:get_uid()`,
  `entity:despawn()`, ...).
- Events: `on_spawn()`, `on_update(tps)`, `on_save()`, and
  `on_sensor_enter(index)` / `on_sensor_exit(index)` for sensors. `[OBSERVED:
  real components; DOC: upstream]`

Persistence pattern from a real component (`my_pack:health`, excerpt):

```lua
local props = {}

local function def_prop(name, def_value)
    props[name] = SAVED_DATA[name] or ARGS[name] or def_value
    this["get_" .. name] = function() return props[name] end
    this["set_" .. name] = function(value)
        props[name] = value
        if math.abs(value - def_value) < 1e-7 then
            SAVED_DATA[name] = nil
        else
            SAVED_DATA[name] = value
        end
    end
end

def_prop("health", 100)

function on_save()
        SAVED_DATA.health = props.health
end
```

`[OBSERVED: wooden_meat_game scripts/components/health.lua]`

Deep ECS internals (cross-component lookup, engine component APIs beyond the
entity handle) are OUT OF SCOPE — read
`doc/ru/scripting/core_components.md` and `doc/ru/scripting/ecs.md` before
going further.

## Entity-specific pitfalls

1. **Inventing `body-type` values.** Only `static` / `kinematic` / `dynamic`
   exist; anything else (e.g. `rigidbody`) is silently ignored and the entity
   stays `dynamic`. `[VERIFIED: engine source]`
2. **Typos in any definition field.** The loader reads only known keys and
   ignores the rest — `"hitbox"` misspelled means default-size hitbox, no
   error. Verify against the field table above. `[VERIFIED: engine source]`
3. **Misspelled component names.** A component string that does not resolve
   to `scripts/components/<name>.lua` of some loaded pack breaks the entity
   at spawn/load time. Component ids are `pack:name`, same registry rules as
   blocks. `[DOC]`
4. **Visible entity without a skeleton.** `skeleton-name` defaults to
   `pack:<entity name>`; if no such file exists in `skeletons/` the entity
   has nothing to render. DO ship `skeletons/<name>.json` for every visible
   entity. `[OBSERVED: base and wooden_meat_game both do]`
5. **Preloading models.** Models used by skeletons are auto-loaded;
   adding them to `preload.json` is noise. `[DOC]`
6. **`ipairs` over `entities.get_all*` results.** Those tables are keyed by
   uid, not sequential; use `pairs` + `table.count_pairs`. `[DOC]`
7. **Expecting a bare item to spawn anything.** A spawn item is just an icon
   until a script with `on_use_on_block` (or another handler) calls
   `entities.spawn`. `[OBSERVED]`

## DO / AVOID summary

- DO name entity files after the entity: `entities/<name>.json` ->
  `pack:name`, same rule as blocks/items.
- DO read the field table above before inventing fields — everything not
  listed is silently ignored.
- DO use `parent` only by necessity — several variants sharing one base you
  fully control. Copy-pasting the JSON is the normal approach, not a sin.
- DO give `solid` dynamic bodies an explicit `mass`.
- DO ship a skeleton for every visible entity (or set `skeleton-name`).
- DO iterate entity query results with `pairs`, not `ipairs`.
- AVOID `body-type` values outside `static`/`kinematic`/`dynamic` — silent
  fallback to `dynamic`.
- AVOID `parent` by default: it is a lightly-tested, undocumented feature,
  and an invisible merge across files hides bugs the same way bad OOP
  inheritance does. Skip the field unless you actually need it.
- AVOID relying on sensor indices other than list order (0-based).
- AVOID copying the `"rigidbody"` line from wooden_meat_game — it is a real
  bug in that repo, kept here only as a warning example.

## Upstream pointers (read before writing real entities)

The paths below are files of the ENGINE's own repository
(github.com/MihailRis/voxelcore, folder `doc/`) — NOT files of this guide.
The Russian documentation is usually MORE complete and more current; when the
two diverge, prefer `doc/ru/`:

| Topic | English (`doc/en/`) | Russian (`doc/ru/`) — prefer this |
| --- | --- | --- |
| Entity definition properties | `doc/en/entity-properties.md` | `doc/ru/entity-properties.md` |
| Built-in components (`core:mob`, ...) | — (no EN file) | `doc/ru/scripting/core_components.md` |
| `entities.*` Lua API | `doc/en/scripting/builtins/libentities.md` | `doc/ru/scripting/builtins/libentities.md` |
| Skeletons, rigging, bone animation | — (no EN file) | `doc/ru/rigging.md` |
| ECS scripting | `doc/en/scripting/ecs.md` | `doc/ru/scripting/ecs.md` |

Also in the engine repository:

- Stock entities: `res/content/base/entities/` and `res/content/base/skeletons/`
  — authoritative, always up-to-date format examples.
- Real usage: `wooden_meat_game` `content/my_pack/` — `entities/`,
  `skeletons/`, `scripts/components/`, spawn items. (Contains the `"rigidbody"`
  bug documented above — do not treat as style guide.)

VCM/OBJ model formats are covered in 14-models.md; bone animation is
covered in 11-animation.md.
