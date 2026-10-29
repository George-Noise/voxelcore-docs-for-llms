# 03. Blocks Reference

> **MANDATORY PREREQUISITES — DO NOT SKIP.** This file assumes you have read
> [01-engine-overview.md](01-engine-overview.md) AND
> [08-pitfalls.md](08-pitfalls.md). 01 defines the ID rules, reserved names
> and trust labels this file relies on; 08 lists the real failure modes
> behind every DO/AVOID note. Skipping either is the most common reason
> models generate broken packs from this guide. If you received only this
> file, fetch and read both before writing any code.

Block definitions are JSON files in `blocks/` of the pack. The file name defines
the content name: `blocks/dirt.json` in pack `base` becomes `base:dirt`.
`[VERIFIED]` ID rules: see 01-engine-overview.md.

A block file can be almost empty — every property has a default. The minimal
sensible block is:

```json
{ "texture": "stone" }
```

Full real example from the stock pack (`base:grass_block`, comments added here,
JSON itself is comment-free):

```json
{
    "material": "base:grass_block",
    "texture-faces": [
        "grass_side",      // index 0: -X (west)
        "grass_side",      // index 1: +X (east)
        "dirt",            // index 2: -Y (bottom)
        "grass_top",       // index 3: +Y (top)
        "grass_side",      // index 4: -Z (north)
        "grass_side"       // index 5: +Z (south)
    ],
    "base:durability": 1.7,
    "base:loot": [
        {"item": "base:dirt.item"}
    ]
}
```

## Texture properties

### `texture` — single texture for all faces

Texture NAME only: no path, no `.png` extension. The PNG must be in the pack's
`textures/blocks/` folder. `[VERIFIED]` Example: `"texture": "dirt"` resolves to
`<pack>/textures/blocks/dirt.png` and lands in the global `blocks` atlas.

### `texture-faces` — per-face textures

Array of exactly 6 texture names, order:

| Index | Face | Direction |
| --- | --- | --- |
| 0 | west | −X |
| 1 | east | +X |
| 2 | bottom | −Y |
| 3 | top | +Y |
| 4 | north | −Z |
| 5 | south | +Z |

`[VERIFIED]` against engine face constants (`FACE_MX, FACE_PX, FACE_MY, FACE_PY,
FACE_MZ, FACE_PZ` in `src/voxels/Block.hpp`) and the stock `grass_block.json`.
This order is NOT stated in official docs — it was extracted from source + real
files. Fewer than 6 elements is a hard load error. `[OBSERVED: engine throws]`

- `texture` and `texture-faces` are mutually exclusive. `[DOC — IMPORTANT]`

### `overlay-texture`

Seen in stock `water.json` (`"overlay-texture": "blocks:water"`); value is an
atlas reference (`atlas:name`). Not documented upstream. `[OBSERVED]`

## Model properties

### `model` — built-in model type

| Value | Behavior |
| --- | --- |
| `block` | Default full cube (default for all blocks) |
| `none` | Invisible block (air-like) |
| `X` | Two crossed sprites — grass/flowers style (see `flower.json` example) |
| `aabb` | Mesh matches the block hitbox (pipes, torch, doors, lamps) |
| `custom` | The ONLY value that makes the engine read `model-name` — OUT OF SCOPE, see stubs |

`[VERIFIED]`

### `model-name` — custom model file

`model-name` is read ONLY and ONLY when `"model": "custom"` is set. With any
other `model` value (`block`, `none`, `X`, `aabb`) or with no `model` field at
all, `model-name` is silently ignored — do not add it "just in case".
`[VERIFIED: user first-hand]`

Real example — `content/vd_belt_system/blocks/destroyer_belt.json` from
[wooden_meat_game](https://github.com/George-Noise/wooden_meat_game), verbatim:

```json
{
    "caption": "Destroyer Belt",
    "texture": "box",
    "model": "custom",
    "model-name": "destroyer_belt",
    "rotation": "pane",
    "hitbox": [0, 0, 0, 1, 0.5, 1],
    "light-passing": true,
    "sky-light-passing": true,
    "material": "base:metal"
}
```

The value is a model name without the `<pack>:models/` prefix and without
extension (in the example above the model file is
`vd_belt_system/models/destroyer_belt.*`). Custom models are covered in
14-models.md. Textures inside custom models resolve as `$0`..`$5` (the
block's texture-faces), `blocks:name` (blocks atlas) or bare names
(standalone `textures/<name>` files). `[DOC]`

### `draw-group` — integer

Render group number. Relevant for translucent blocks (water uses `3`, sprites
like flowers use `1`) to fix invisible-face sorting issues. `[VERIFIED]`

### `translucent` — boolean

Enables alpha translucency of the texture (water, ice). Costs performance; NOT
needed for full transparency (leaves, flowers) and NOT needed with `draw-group`
alone. `[DOC]`

### `solid` — boolean

Declares a `custom`-model block as fully solid so hidden-face culling works like
for `block` cubes. Ignored when `false`. `[DOC]`

### `culling` — `"default"` | `"optional"` | `"disabled"`

Face culling mode within a draw group. `[DOC]`

### `shadeless` — boolean

Disables lighting on the block model (torch flame uses it). `[VERIFIED]`

### `ambient-occlusion` — boolean (default true)

Vertex ambient occlusion toggle. `[VERIFIED]`

## Rotation

### `rotation` — placement rotation profile

| Value | For blocks like |
| --- | --- |
| `none` | default; no rotation |
| `pipe` | logs, pipes, torches (axis alignment) |
| `pane` | panels, doors, signs (flat alignment) |
| `stairs` | stairs (`pane` + upside-down variants) |
| `ladder` | ladders (`pipe` on vertical + `pane` on horizontal surfaces) |

`[DOC]` Real usage: `wood.json` and `torch.json` use `"rotation": "pipe"`,
`wooden_door.json` uses `"pane"`. `[OBSERVED]`

## Lighting

### `emission` — `[R, G, B]`, each 0..15

Light emitted by the block. `[15,15,15]` = brightest white; `[0,0,0]` (default) =
no emission. `[VERIFIED]` Example: stock lamp: `"emission": [15, 14, 13]`,
torch: `"emission": [13, 10, 2]`.

### `light-passing` — boolean

Block lets light from emitters pass through. Flowers, torch, water, glass set it. `[VERIFIED]`

### `sky-light-passing` — boolean

Block does not block the vertical sunbeam. Water explicitly sets `false`
(so it darkens underwater). `[VERIFIED]`

## Physics

### `obstacle` — boolean (default true)

`false` removes the hitbox — players walk through (flowers, water, torch). `[VERIFIED]`

### `hitbox` — `[x, y, z, w, h, d]`, all in 0..1

Offset + size of the hitbox inside the block cell. Example from `torch.json`:
`[0.4375, 0.0, 0.4375, 0.125, 0.5, 0.125]` = small stick in the middle.
`[VERIFIED]` Axis semantics: +X=east, +Y=up, +Z=south (from upstream docs of the
hitbox example). For multiple boxes use `hitboxes` with an array of such arrays
(stock door uses it). `[VERIFIED]`

### `grounded` — boolean

Block must be placed on a full block and breaks when support breaks (flowers,
torch). `[VERIFIED]`

### `selectable` — boolean (default true)

`false` — the cursor targets blocks behind it (water). `[VERIFIED]`

### `replaceable` — boolean

`true` — placing any block replaces this one (water, grass, flowers). `[VERIFIED]`

### `breakable` — boolean (default true)

`false` — block cannot be broken. `[DOC]`

### `size` — `[sx, sy, sz]` integers, default `[1,1,1]`

Extended (multi-cell) blocks; the stock door is `"size": [1, 2, 1]`. `[VERIFIED]`
With `grounded` on extended blocks, `grounding-behaviour` picks which segments
need support: `partial` (default) / `complete` / `origin`. `[DOC]`

## Inventory & interaction

### `caption` — display name

Human-readable name. Blocks normally skip it: the default caption is derived
from the ID with underscores turned into spaces (`grass_block` → `grass block`),
and the engine upper-cases it for display. Localization then maps caption
strings (see 05). The stock pack sets `"caption": "tall grass"` only where the
auto name is wrong. `[OBSERVED]`

### `hidden` — boolean

Hides the block from inventory and suppresses auto-generated item; upstream
docs say from engine 0.17+ such blocks MUST define `picking-item`. `[DOC]`

### `picking-item` — item ID

Item given by middle-click pick. Stock example: hidden `door:door_open` sets
`"picking-item": "door:door.item"`. `[DOC]`

### `inventory-size` — integer (default 0)

Slots of the block's own inventory (containers). UI hookup is an excluded topic. `[DOC]`

### `ui-layout` — layout ID for the block's UI. Excluded topic. `[DOC]`

## Materials (interaction sounds)

### `material` — `pack:material_name`

References a definition in `block_materials/` that maps to step/place/break/hit
sounds. See 05-textures-and-assets.md for the material file format. Stock
examples: `"material": "base:wood"`, `"base:ground"`, `"base:glass"`. `[VERIFIED]`

## Data & scripting hooks (short)

- `fields` — typed per-block data slots (int8..int64, float32/64, char; total
  ≤ 240 bytes; array `length` supported; `convert-strategy`: `reset`/`clamp`).
  `[DOC]` Useful only together with Lua — see stubs before using.
- `state-based` — variant table switching `model`/`model-name`/`texture`/
  `texture-faces` by user bits (`offset`, `bits`, `variants`, max 16 variants).
  `[DOC]` Controlled from Lua via `block.set_variant(x, y, z, index)`.
- `script-name` — script file name (without `pack:scripts/` and extension) that
  holds this block's event handlers. Default: the block's own name. `[DOC]`
- `tick-interval` — world ticks (1/20 s) between `on_block_tick` events. `[DOC]`
- `properties@append` — any list property can be suffixed with `@append` by
  *other packs* to extend instead of overwrite. `[DOC]`

## Tags

`"tags": ["core:liquid", ...]` — free-form labels; prefix them with a namespace
to avoid collisions. The engine uses the `core:*` namespace (stock water block:
`"tags": ["core:liquid"]`); the stock pack namespaces its own tags as `base:ore`
(coal ore). `[VERIFIED]` Other packs can tag your content from their own
`tags.toml` file:

```toml
"my_pack:gemlike" = ["some_pack:some_ore", "my_pack:ruby_ore"]
"my_pack:flammable" = ["my_pack:wooden_floor"]
```

## Properties registered by the `base` pack (not engine built-ins!)

- `base:durability` — seconds to break without tools. Stock: dirt 1.5,
  grass_block 1.7, stone 12.0, wood 6.0, flower 0.0. `[VERIFIED]`
- `base:loot` — array of `{ "item": "pack:item", "count": n | "min": n, "max": m, "chance": p }`.
  `item` MUST be an ITEM id — for a block's item form append `.item`
  (`"base:dirt.item"`). `chance` default 1.0. `[VERIFIED]`
- Drop semantics observed in the stock pack: blocks WITHOUT `base:loot` (stone,
  dirt) drop their own generated item when broken; `"base:loot": []` (tall
  grass) means drop nothing. `[OBSERVED]`
- Using either REQUIRES a dependency on `"base"` (02-pack-structure.md).

## Observed-but-undocumented properties

- `surface-replacement` — stock `dirt.json` sets `"surface-replacement": "base:grass_block"`;
  grass spreading to exposed dirt is implemented via this + the block script's
  `on_random_update`. Semantics are not documented upstream. `[OBSERVED]`
- `overlay-texture` — see Texture properties. `[OBSERVED]`
- Block-level `caption` — see above. `[OBSERVED]`

DO NOT rely on undocumented properties for anything critical; prefer documented
equivalents and check the stock pack again when the engine updates.

## Annotated stock examples

`base:dirt` — minimal solid cube with base-pack props:

```json
{
    "texture": "dirt",
    "material": "base:ground",
    "surface-replacement": "base:grass_block",
    "base:durability": 1.5
}
```

`base:flower` — crossed sprite, no collision, replaceable, no loot:

```json
{
    "texture": "flower",
    "material": "base:grass",
    "draw-group": 1,
    "light-passing": true,
    "obstacle": false,
    "replaceable": true,
    "grounded": true,
    "model": "X",
    "hitbox": [0.15, 0.0, 0.15, 0.7, 0.7, 0.7],
    "base:durability": 0.0
}
```

`base:water` — translucent liquid:

```json
{
    "texture": "water",
    "overlay-texture": "blocks:water",
    "draw-group": 3,
    "light-passing": true,
    "sky-light-passing": false,
    "obstacle": false,
    "selectable": false,
    "replaceable": true,
    "translucent": true,
    "tags": ["core:liquid"]
}
```

`base:torch` — `aabb` model, light, particles, rotation:

```json
{
    "texture-faces": ["torch_side","torch_side","torch_bottom","torch_top","torch_side","torch_side"],
    "emission": [13, 10, 2],
    "model": "aabb",
    "hitbox": [0.4375, 0.0, 0.4375, 0.125, 0.5, 0.125],
    "light-passing": true,
    "shadeless": true,
    "obstacle": false,
    "grounded": true,
    "rotation": "pipe",
    "material": "base:wood",
    "base:durability": 0.1
}
```
(`particles` object shortened; particle configs are an excluded topic.)

## DO / AVOID summary

- DO start every block JSON with `texture` or `texture-faces`; add the rest as needed.
- DO give `texture-faces` exactly 6 entries in the west/east/bottom/top/north/south order.
- DO use `"model": "X"` + `obstacle:false` + `replaceable:true` + `grounded:true` + `draw-group:1` for plants.
- DO reference block-derived items with the `.item` suffix in loot/pick properties.
- AVOID combining `texture` with `texture-faces` in one block.
- AVOID adding `model-name` to a block whose `model` is not `"custom"` — the
  field is ignored there and only misleads readers.
- AVOID `translucent` unless you need alpha blending (prefer `draw-group` + `light-passing`).
- AVOID inventing property names; everything allowed is listed above or in upstream docs.
- AVOID depending on `[OBSERVED]` properties without a fallback plan.
