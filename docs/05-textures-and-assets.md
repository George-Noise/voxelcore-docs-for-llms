# 05. Textures, Atlases, preload.json, Materials, Localization

> **MANDATORY PREREQUISITES — DO NOT SKIP.** This file assumes you have read
> [01-engine-overview.md](01-engine-overview.md) AND
> [08-pitfalls.md](08-pitfalls.md). 01 defines the ID rules, reserved names
> and trust labels this file relies on; 08 lists the real failure modes
> behind every DO/AVOID note. Skipping either is the most common reason
> models generate broken packs from this guide. If you received only this
> file, fetch and read both before writing any code.

This document covers the asset side of the IN-SCOPE topics: block/item textures,
the pack icon, asset preloading, block materials, and caption localization.

## Texture folders and atlases

Two global texture atlases exist and every pack contributes to them
automatically: `[VERIFIED]`

| Atlas | Fed by | Used for |
| --- | --- | --- |
| `blocks` | `<pack>/textures/blocks/*.png` | block `texture`/`texture-faces`, block-model textures |
| `items` | `<pack>/textures/items/*.png` | item sprite icons (`icon: "items:x"`) |

Rules:

- Texture NAME = file name without `.png`. `"texture": "dirt"` →
  `textures/blocks/dirt.png` → atlas entry `blocks:dirt`. `[VERIFIED]`
- Stock pack textures are 16x16 RGBA PNG. DO use 16x16 (or a consistent power
  of two) — mixed sizes across packs are untested territory. `[OBSERVED]`
- AVOID referencing textures by path or with `.png` — always `atlas:name`.
- Name collisions between packs in the same atlas are not documented; DO prefix
  file names with something pack-specific when genericity is likely
  (`mymod_gear.png` instead of `gear.png`). `[OBSERVED-CONCERN]`

Additional pack textures (GUI, particles, sky) are loaded only through
`preload.json` `"textures"` — plain paths relative to `textures/` without
extension, e.g. `"gui/crosshair"` from `<pack>/textures/gui/crosshair.png`.
`[VERIFIED: stock preload.json]`

Custom atlases: `preload.json` accepts an `"atlases"` array, e.g. stock pack:
`"atlases": [{"name": "cracks", "type": "separate"}]` (folder
`textures/cracks/`); the Kompot pack registers atlases `kompot_ui_icons` /
`kompot_ui_skin`. Custom atlas machinery is UI/advanced territory — out of
scope. `[OBSERVED]`

## Pack icon

`icon.png` at the pack root, square PNG, recommended 128x128 `[DOC]`; existing
packs range from 64x64 to 210x208 and load fine `[OBSERVED]`.

## preload.json

Optional file at pack root preloading assets the engine would not load on
demand. Format: object whose keys are asset categories, values are arrays of
asset names/paths (no extensions). `[VERIFIED]`

Categories: `animation`, `fonts`, `models`, `shaders`, `skeletons`, `sounds`,
`textures`. Extras seen in real packs: `atlases` (see above). `[OBSERVED]`
The `animation` category (VCA tracks) has its own document:
11-animation.md.

```json
{
    "atlases": [{"name": "cracks", "type": "separate"}],
    "sounds":  ["blocks/door_open", "events/pickup"],
    "models":  ["drop-item"],
    "textures": ["misc/moon", "misc/sun", "gui/crosshair"]
}
```

Real game-pack example (wooden_meat_game / my_pack):

```json
{
    "animation": ["hit", "rip"],
    "sounds": ["shotgun"],
    "textures": ["gfx/muzzle_flash", "gfx/tracer"]
}
```

Sound names load all variant files by pattern: `sound_name.ogg`,
`sound_name_1.ogg`, `sound_name_2.ogg`, ... or `sound_name_0.ogg`, ... `[DOC]`

Fonts entries are objects `{"name": "kompot_12", "path": "fonts/IBMPlexSans-Regular.ttf", "size": 12}` `[OBSERVED]`.

AVOID preloading things you do not use — upstream docs explicitly warn against
unnecessary preload entries. `[DOC]`

## Block materials (`block_materials/`)

A *material* bundles interaction sounds for blocks. Blocks reference it by
`"material": "pack:material_name"`; files are JSON in `block_materials/`
(`block_materials/wood.json` → `base:wood`). `[VERIFIED]`

```json
{
    "steps-sound": "steps/wood",
    "place-sound": "blocks/wood_place",
    "break-sound": "blocks/wood_break",
    "hit-sound": "blocks/wood_hit",
    "sound-absorption": 0.5
}
```

- Sound values are names resolved against loaded sounds — see 12-sounds.md.
  Material sounds (`steps-sound` etc.) are requested by the engine
  automatically and need NO `preload.json` entry; if `hit-sound` is omitted
  it falls back to `steps-sound`. The stock sounds live in the `base` pack.
- The stock `config/defaults.toml` sets `block-material = "base:stone"` as the
  game-wide fallback. `[OBSERVED]`
- DO reuse existing `base:*` materials (`base:stone`, `base:wood`, `base:ground`,
  `base:glass`, `base:grass`, `base:metal`, ...) instead of creating duplicates.
- AVOID creating a material with sound names that no loaded pack provides.

## Localization (`texts/`)

Files named by locale: `en_US.txt`, `ru_RU.txt`, `fi_FI.txt`, `uk_UA.txt`,
`uz_UZ.txt` are all observed in real packs. `[VERIFIED]`

Format — one `key=value` pair per line. Content captions use the caption string
itself as the key: `[VERIFIED]`

```text
# ru_RU.txt of the base pack (excerpt)
dirt=земля
grass block=дёрн
bazalt breaker=крушитель базальта
```

- Caption keys match the lowercased caption (the engine auto-capitalizes for
  display; docs recommend NOT capitalizing captions yourself). `[DOC]`
- UI-level translations use dotted keys, e.g. `kompot.gallery=Kompot / Gallery`
  in the Kompot pack. `[OBSERVED]`
- DO provide `en_US.txt` only for entries that differ from the caption itself;
  captions are already the English defaults.

## DO / AVOID summary

- DO drop every block texture into `textures/blocks/<name>.png` and every item
  icon into `textures/items/<name>.png` — nothing else needed for them to work.
- DO keep textures 16x16 RGBA PNG for blocks/items.
- DO reference textures as `blocks:name` / `items:name` everywhere.
- DO preload only what is not auto-loaded (extra textures, sounds, models).
- AVOID `.png` extensions and folder paths inside `texture`/`icon` values.
- AVOID editing preload entries for auto-loaded assets (blocks/items atlas
  textures do not need preload).
- AVOID translating by editing captions per language — captions stay in English;
  translations go to `texts/<locale>.txt`.
