# 06. Walkthrough: Build a Complete Minimal Pack

Goal: a working pack `my_first_pack` that adds:
- block `my_first_pack:ruby_ore` — a stone-like ore cube,
- block `my_first_pack:ruby_lamp` — a glowing cube,
- block `my_first_pack:ruby_flower` — a decorative plant,
- item `my_first_pack:ruby` — a plain sprite item,
- a pack icon.

No Lua, no models, no sounds — pure declarative content. Every file below is
complete and final; assemble them in order and the pack works.

## Step 0 — Choose the location

DO develop in the user content folder so any world can use the pack:

```text
$HOME/.voxeng/content/my_first_pack/
```

(Linux path; `user:content` source, priority above `res:content` — see
02-pack-structure.md.) The folder name is free-form — it does NOT need to
match the pack ID (real packs rarely match). This walkthrough uses the ID as
the folder name purely for tidiness.

## Step 1 — package.json (pack becomes loadable)

File: `my_first_pack/package.json`

```json
{
    "id": "my_first_pack",
    "title": "My First Pack",
    "version": "0.1.0",
    "creator": "Your Name",
    "description": "Ruby ore, lamp, flower and a ruby item",
    "dependencies": ["base"]
}
```

`dependencies: ["base"]` is REQUIRED here because the blocks below use
`base:durability` and `base:loot` (properties registered by `base`).

STOP and verify: the pack must already appear in the engine's content menu with
no other files at all.

## Step 2 — Textures

Create two folders and four PNG files, 16x16, RGBA:

```text
textures/blocks/ruby_ore.png
textures/blocks/ruby_lamp.png
textures/blocks/ruby_flower.png
textures/items/ruby.png
```

Any 16x16 pixel-art works while prototyping. The file names become the texture
names (`ruby_ore`, ...). No `.json` needed for textures — the atlases pick PNGs
up automatically (see 05).

## Step 3 — The ore block

File: `blocks/ruby_ore.json`

```json
{
    "texture": "ruby_ore",
    "material": "base:stone",
    "base:durability": 3.0,
    "base:loot": [
        {"item": "my_first_pack:ruby", "min": 1, "max": 2, "chance": 1.0}
    ]
}
```

Notes:
- `"texture"` — single texture on all 6 faces (order not relevant here).
- `"material": "base:stone"` — reuses base's sounds; no sound files needed.
- loot drops the ITEM `my_first_pack:ruby` defined in Step 6.

## Step 4 — The lamp block

File: `blocks/ruby_lamp.json`

```json
{
    "texture": "ruby_lamp",
    "material": "base:glass",
    "emission": [14, 4, 4],
    "light-passing": true,
    "base:durability": 0.5
}
```

`emission` channels are 0..15; `[14, 4, 4]` is strong red. This block drops
itself when broken: blocks without `base:loot` default to dropping their own
generated item (stock stone/dirt behave this way `[OBSERVED]`).

## Step 5 — The flower block (crossed sprite)

File: `blocks/ruby_flower.json`

```json
{
    "texture": "ruby_flower",
    "material": "base:grass",
    "model": "X",
    "draw-group": 1,
    "light-passing": true,
    "obstacle": false,
    "replaceable": true,
    "grounded": true,
    "hitbox": [0.15, 0.0, 0.15, 0.7, 0.5, 0.7],
    "base:durability": 0.0,
    "base:loot": []
}
```

This is the stock flower recipe: `model X` + no obstacle + replaceable + grounded
+ draw-group 1. `"base:loot": []` means it drops nothing (vs. default self-drop).

## Step 6 — The ruby item

File: `items/ruby.json`

```json
{
    "icon-type": "sprite",
    "icon": "items:ruby",
    "stack-size": 64
}
```

`"icon": "items:ruby"` points at `textures/items/ruby.png` from Step 2. This
item is inert (no `placing-block`, no script) — it is a material/crafting-style
resource.

## Step 7 — Pack icon

Save a square PNG as `icon.png` at the pack root (128x128 recommended).

## Step 8 — Enable and test

1. Launch the engine.
2. Create or open a world; open the content selection and enable
   **My First Pack** (its dependency `base` is required for any world anyway).
3. In game: give items through the inventory/content screen, place the ore,
   break it — the ruby item drops; place the lamp — the area lights red; place
   the flower on top of a full block.

## Final file tree

```text
my_first_pack/
├── package.json
├── icon.png
├── blocks/
│   ├── ruby_ore.json
│   ├── ruby_lamp.json
│   └── ruby_flower.json
├── items/
│   └── ruby.json
└── textures/
    ├── blocks/
    │   ├── ruby_ore.png
    │   ├── ruby_lamp.png
    │   └── ruby_flower.png
    └── items/
        └── ruby.png
```

(`content.json` appears next to `package.json` after the pack loads — generated
by the engine. AVOID touching it.)

## Extensions beyond this walkthrough

- Per-face textures → `texture-faces` (03-blocks.md).
- Localization of captions → `texts/ru_RU.txt` with `ruby ore=рубиновая руда`
  (05-textures-and-assets.md).
- Behavior (drop-on-use, ticking) → Lua scripts, out of scope; see stubs and
  then the engine's `doc/ru/scripting.md`.

## Verification checklist

- [ ] `package.json` parses as JSON (no trailing commas, no comments).
- [ ] `id` passes the character/length rules and is not reserved (the folder
      name may be anything — equality with `id` is not required).
- [ ] Every `texture`/`icon` name has a matching PNG in the right folder.
- [ ] Every `pack:` reference points to an existing pack (dependency declared).
- [ ] `base:*` properties are used only with `"base"` in dependencies.
- [ ] No `content.json` in your source files (engine generates it).
