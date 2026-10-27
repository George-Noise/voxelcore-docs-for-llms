# 02. Pack Structure and Manifest

This document defines where packs live, what `package.json` must contain, and
which auxiliary files a pack may have.

## Where packs live (content sources)

The engine discovers packs in **content sources**, scanned in priority order
(highest first). If the same pack ID exists in several sources, the
highest-priority one wins and the rest are ignored. `[VERIFIED]`

| Priority | Source           | Path                                        |
| -------- | ---------------- | ------------------------------------------- |
| 1        | `world:content`  | `<world folder>/content/`                   |
| 2        | `user:content`   | `$HOME/.voxeng/content/`                    |
| 3        | `project:content`| `<project folder>/content/` (project mode)  |
| 4        | `res:content`    | `<engine>/res/content/` (built-in packs)    |

Practical guidance:

- For local development DO put your pack into `user:content`:
  `~/.voxeng/content/<pack_id>/` on Linux. This works for any world.
- Packs shipped inside a game project go to `<project>/content/` (see
  07-example-repos.md for a real project doing this).
- `res:content` is for content bundled with the engine build itself (the stock
  `base` pack lives there).

**Folder name vs pack ID.** The engine identifies a pack by the `"id"` inside
`package.json`, NOT by the folder name. The folder name is free-form: real
packs almost always use a folder name different from the ID, and exact matches
are rare in the wild — for example a published project ships a pack in folder
`kompot_0f9103ff-1.2.2` with `"id": "kompot"`, and it loads fine.
`[OBSERVED]` NEVER assume that the folder name equals the pack ID. When you
need a pack's ID, read its `package.json`; when you create a pack folder, any
name works.

## package.json (the only required file)

`[VERIFIED]` Format and field semantics:

```json
{
    "id":          "my_pack",
    "title":       "My Pack",
    "version":     "1.0.0",
    "creator":     "Author Name",
    "description": "What this pack adds",
    "dependencies": ["base", "~randutil@>=1.0"]
}
```

| Field          | Required | Format / notes                                                |
| -------------- | -------- | ------------------------------------------------------------- |
| `id`           | yes      | Pack ID, rules in 01-engine-overview.md                        |
| `title`        | yes*     | Display name in the content menu                               |
| `version`      | yes*     | `major.minor[.patch]`, e.g. `"1.0"`, `"0.32"`                  |
| `creator`      | no       | Author name; OR use an array field `creators` instead `[DOC]`  |
| `description`  | no       | Short free-form text                                           |
| `dependencies` | no       | Array of dependency strings (see below)                        |
| `license`      | no       | Observed in real packs (`"license": "MPL-2.0"`); engine ignores unknown fields `[OBSERVED]` |

\* Official docs show `title`/`version` in every example; packs without them are
unknown territory. DO always include `id`, `title`, `version`.

### Dependencies

Each dependency string is `[prefix]pack_id[@version_spec]`:

| Prefix | Meaning      | Behavior                                                        |
| ------ | ------------ | --------------------------------------------------------------- |
| `!` or none | required | Pack fails to load without it (this is the default) `[DOC]`     |
| `?`    | optional     | Loaded if present, ignored otherwise                            |
| `~`    | weak         | Order-only / best-effort (documented as "weak dependency")      |

Version spec operators exist (e.g. `>=1.0`, `*` = any, which is the default).
Only `>=` is shown by official docs; AVOID inventing other operators (`<`, `=`)
— they are not documented. `[VERIFIED: 'randutil@>=1.0' example]`

Real examples:

- Stock `base` pack has **no dependencies**:
  `{ "id": "base", "title": "Base", "version": "0.32", "description": "basic content package" }`
- Real pack: `"dependencies": ["kompot"]` (hard dependency on the Kompot UI pack).

**DO declare a dependency on `"base"` if your pack uses `base:*` properties**
such as `base:durability` or `base:loot` — those are user-defined properties
registered by the `base` pack, not engine built-ins (see 03-blocks.md).

## icon.png

- Optional file at the **pack root** (NOT inside `textures/`). `[VERIFIED]`
- Recommended size 128x128 `[DOC]`; the stock pack ships 64x64 and a real pack
  ships 210x208, so the size is clearly not enforced. `[OBSERVED]`
- DO keep it square PNG.

## Folder layout

Full layout documented upstream, annotated with this guide's scope. A minimal
pack only needs `package.json`; create subfolders only when you actually define
content.

```text
my_pack/
├── package.json            # REQUIRED manifest
├── icon.png                # optional, pack root
├── blocks/                 # IN SCOPE  — 03-blocks.md
├── items/                  # IN SCOPE  — 04-items.md
├── block_materials/        # IN SCOPE  — 05-textures-and-assets.md
├── textures/
│   ├── blocks/             # IN SCOPE — auto-joins the global "blocks" atlas
│   └── items/              # IN SCOPE — auto-joins the global "items" atlas
├── texts/                  # IN SCOPE  — localization (05)
├── preload.json            # IN SCOPE  — 05-textures-and-assets.md
├── config/                 # partially in scope — see below
├── content.json            # AUTO-GENERATED. NEVER hand-edit, NEVER create.
├── entities/               # OUT OF SCOPE (stub)
├── skeletons/              # OUT OF SCOPE (stub)
├── models/                 # OUT OF SCOPE (stub)
├── sounds/                 # OUT OF SCOPE (stub)
├── shaders/                # OUT OF SCOPE
├── fonts/                  # OUT OF SCOPE
├── animations/             # OUT OF SCOPE (stub)
├── layouts/                # OUT OF SCOPE (stub: XML UI)
├── generators/             # OUT OF SCOPE (stub: worldgen)
├── scripts/                # OUT OF SCOPE (Lua) — referenced by blocks/items
├── modules/                # OUT OF SCOPE (Lua libraries)
├── presets/                # OUT OF SCOPE
├── devtools/               # OUT OF SCOPE
├── resources.json          # engine resources (cameras, effects) — rare, skip
└── resource-aliases.json   # resource aliases — rare, skip
```

`config/` subfolder:

- `config/defaults.toml` — overrides default content bindings of the game:
  the stock pack sets
  ```toml
  generator = "base:demo"
  player-entity = "base:player"
  hand-skeleton = "base:hand"
  block-material = "base:stone"
  ```
  `[OBSERVED]` Only relevant for total-conversion-style packs; typical add-on
  packs do not ship this file.
- `config/bindings.toml` — key bindings declared by the pack. Real format:
  ```toml
  [kompot]
  gallery = "key:f12"
  ```
  i.e. `[pack_id]` section, `action_name = "key:<code>"`. `[OBSERVED]`
- `config/user-props.toml` — declares custom properties the pack adds to
  blocks/items. Real example from `base`:
  ```toml
  "base:durability" = {}
  "base:loot" = {}
  ```
  `[VERIFIED]` Custom properties are then read at runtime via Lua
  (`block.properties[id]["prop"]`) — scripting, out of scope here.

## Enabling a pack for a world

- Normal mode: enable the pack in the engine's content menu when
  creating/configuring a world (UI flow, not file-driven). Dependencies are
  enabled automatically.
- Project mode: a `project.toml` at the project root lists default packs:
  ```toml
  base_packs = ["base", "kompot", "wooden_meat_menu"]
  ```
  `[OBSERVED]` (from the wooden_meat_game project). Projects themselves are out
  of scope — see docs/stubs/TODO-projects.md.

## DO / AVOID summary

- DO make the minimal pack first (`package.json` only), verify it loads, then add content.
- DO treat the folder name as free-form; NEVER assume it must match the pack
  ID (real packs rarely match — the ID lives only in `package.json`).
- DO put `icon.png` at pack root, not in `textures/`.
- DO declare `dependencies` when you reference `base:*` or any other pack's content.
- AVOID editing `content.json` — it is generated; manual edits corrupt worlds (upstream warning).
- AVOID version specs other than the documented `>=` form.
- AVOID relying on pack `version` for resolution — content sources ignore versions entirely. `[DOC]`
