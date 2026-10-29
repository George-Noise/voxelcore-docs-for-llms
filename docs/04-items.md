# 04. Items Reference

> **MANDATORY PREREQUISITES — DO NOT SKIP.** This file assumes you have read
> [01-engine-overview.md](01-engine-overview.md) AND
> [08-pitfalls.md](08-pitfalls.md). 01 defines the ID rules, reserved names
> and trust labels this file relies on; 08 lists the real failure modes
> behind every DO/AVOID note. Skipping either is the most common reason
> models generate broken packs from this guide. If you received only this
> file, fetch and read both before writing any code.

Item definitions are JSON files in `items/` of the pack. The file name defines
the content name: `items/shotgun.json` in pack `my_pack` becomes
`my_pack:shotgun`. `[VERIFIED]`

Every block also automatically gets an item — addressed as
`<pack>:<block_name>.item` — which places the block on right-click. DO NOT
create `items/<name>.json` for a block that already exists; that suffix form is
for *referencing* the generated item, not a file path.

## Real annotated examples

Stock pack, `base:bazalt_breaker` (a tool-like item):

```json
{
    "icon-type": "sprite",
    "icon": "items:bazalt_breaker",
    "uses": 100
}
```

Real game pack, `my_pack:shotgun` (wooden_meat_game):

```json
{
    "icon-type": "sprite",
    "icon": "items:shotgun",
    "stack-size": 1,
    "model-name": "shotgun"
}
```
`model-name: "shotgun"` resolves to `my_pack:models/shotgun.obj` — models are
covered in 14-models.md; omitting `model-name` makes the
engine generate a default model.

Real game pack, `my_pack:spawn_nepis` (spawn-egg style item driven by a script):

```json
{
    "icon-type": "sprite",
    "icon": "items:nepis"
}
```

## Property reference

### Visual

| Property | Type | Meaning |
| --- | --- | --- |
| `icon-type` | string | `sprite` (2D icon) \| `block` (3D block preview). `none` exists only for `core:empty` and may be removed — AVOID using it. `[DOC]` |
| `icon` | string | For `sprite`: `atlas:texture` — e.g. `items:shotgun`, `blocks:dirt`. For `block`: block ID — e.g. `base:wood`. `[VERIFIED]` |
| `model-name` | string | Item model name; default `packid:itemname`; auto-generated when absent. `[DOC]` |
| `caption` | string | Display name in inventory. Supports inline `md` markup. `[DOC]` Default caption derives from the ID (underscores → spaces). `[OBSERVED]` |
| `description` | string | Inventory tooltip. Supports inline `md` markup. `[DOC]` |

The two automatic atlases are `blocks` (from `textures/blocks/*.png`) and
`items` (from `textures/items/*.png`) — same for every pack; your PNGs simply
join them by file name. See 05-textures-and-assets.md.

### Behavior

| Property | Type | Meaning |
| --- | --- | --- |
| `placing-block` | string | Block ID placed on right-click. This is what generated block-items use internally. `[DOC]` |
| `emission` | [R,G,B] | Light emitted while held in hand, each channel 0..15. `[DOC]` |
| `stack-size` | int | Max per inventory slot. Default 64. Shotgun uses 1. `[VERIFIED]` |
| `uses` | int | Durability/consumable counter, driven via `inventory.use()` from Lua. Shown in inventory UI. `[VERIFIED]` |
| `uses-display` | string | `none` \| `number` \| `relation` \| `vbar` (default `vbar`). `[DOC]` |
| `script-name` | string | Script file (without `pack:scripts/` and extension) with the item's event handlers. Default: the item's own name. `[DOC]` |

### Tags

`"tags": ["core:fuel", "my_pack:gems"]` — same rules as block tags: namespaced,
`core:*` reserved for the engine. `[VERIFIED]` Other packs may tag your items
from their `tags.toml`.

### Item scripts (boundary knowledge only)

An item whose JSON has no `placing-block` usually gets behavior from a Lua
script `scripts/<script-name>.lua`. Event handlers are global Lua functions, e.g.
`function on_use(pid)` — the stock pack's `bazalt_breaker.lua` and the game's
`shotgun.lua` both define `on_use(pid)`. `[OBSERVED]` The full event/API list is
a Lua-scripting topic: NOT covered here — read `doc/ru/scripting.md` +
`doc/ru/scripting/` in the engine repo before writing item scripts.

## Localization of captions

Caption strings are looked up in the pack's `texts/<locale>.txt` files
(`key=value` per line). The stock pack translates `"bazalt breaker"` (the
auto-derived caption of `bazalt_breaker`) in `ru_RU.txt` as
`bazalt breaker=крушитель базальта`. `[VERIFIED]` Details in 05.

## DO / AVOID summary

- DO give every item an explicit `icon-type: "sprite"` + `icon: "items:<name>"`.
- DO ship a 16x16 PNG at `textures/items/<name>.png` for each sprite icon.
- DO use `placing-block` if you want an item that places an existing block under
  a different identity/name.
- DO use `stack-size: 1` for tools/weapons.
- AVOID creating an item file duplicating a block (`base:dirt.item` already exists).
- AVOID `icon-type: "none"` — documented as removable, used only by `core:empty`.
- AVOID referencing atlas textures with paths or `.png` extensions — only `atlas:name`.
- AVOID guessing item event handler names beyond `on_use(pid)` without checking
  the engine's scripting docs.
