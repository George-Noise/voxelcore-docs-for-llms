# 07. Annotated Breakdown of Real Repositories

Two real-world repositories are referenced by this guide. Read this file to map
their structure onto the concepts, and to know which of their folders are OUT OF
SCOPE of this guide.

- Game project: <https://github.com/George-Noise/wooden_meat_game>
- UI pack: <https://github.com/George-Noise/fork_of_kompot>

Both are unfinished but load and run; they are referenced as *structure
evidence*, not as style guides.

## wooden_meat_game — a game project wrapping several packs

Top level is a **project**, not a pack (projects are an excluded topic — see
docs/stubs/TODO-projects-worldgen.md):

```text
wooden_meat_game/
├── project.toml            # project manifest
├── project_client.lua      # project-level client script   (out of scope)
├── start.lua               # project entry script           (out of scope)
├── voxelcore_types.lua     # Lua type annotations           (out of scope)
└── content/                # project:content source — packs live here
```

`project.toml`:

```toml
name = "wooden_meat_game"
title = "Wooden Meat Game"
base_packs = ["base", "kompot", "wooden_meat_menu"]
permissions = []
```

`base_packs` lists packs enabled by default; they resolve against
`project:content` (this repo's `content/`) and lower-priority sources.
`[OBSERVED]`

### Packs inside content/

| Folder                      | id in package.json | Lesson                                              |
| --------------------------- | ------------------ | --------------------------------------------------- |
| `kompot_0f9103ff-1.2.2/`    | `kompot`           | **Folder name ≠ pack ID** — engine reads the ID from package.json `[OBSERVED]` |
| `my_pack/`                  | `my_pack`          | Title differs from id: "Simple Box and Gun" — fine  |
| `sfx/`                      | `vd_ambient_sfx`   | Folder `sfx`, id `vd_ambient_sfx` — again ≠         |
| `vd_belt_system/`           | `vd_belt_system`   | Conventional 1:1 naming (recommended default)       |
| `wooden_meat_menu/`         | `wooden_meat_menu` | Declares `"dependencies": ["kompot"]` `[OBSERVED]`  |

Vendored-pack naming (`kompot_<hash>-<version>`) is a human-organization trick;
the engine only sees id `kompot`.

### my_pack — what is in scope

```text
my_pack/
├── package.json            # id "my_pack", version "0.0.1"
├── icon.png                # 210x208 — icon size is not enforced
├── content.json            # auto-generated (items/entities lists)
├── items/
│   ├── shotgun.json
│   ├── spawn_nepis.json
│   └── spawn_turret.json
├── textures/
│   └── items/              # shotgun.png, nepis.png, turret.png → items atlas
├── preload.json
├── scripts/ ...            # Lua event handlers (out of scope)
├── entities/ ...           # out of scope
├── models/ ...             # out of scope
├── animation/, skeletons/, sounds/   # out of scope
```

`package.json` has no `dependencies` yet references only its own content —
correct, because it does not use `base:*` properties.

`items/shotgun.json`:

```json
{
  "icon-type": "sprite",
  "icon": "items:shotgun",
  "stack-size": 1,
  "model-name": "shotgun"
}
```

Takeaways (all `[OBSERVED]`):
- Sprite icon `"items:shotgun"` ↔ file `textures/items/shotgun.png`.
- `stack-size: 1` for a weapon; `model-name: "shotgun"` pulls
  `models/shotgun.obj`.
- `items/spawn_nepis.json` has only `icon-type`+`icon` — the rest of behavior
  lives in `scripts/spawn_nepis.lua` (handler `on_use(pid)`, spawns an entity).
- `preload.json` preloads only non-auto assets, with keys `animation` (NOTE the
  singular spelling — engine constant), `sounds`, `textures`: two animations,
  one sound, two effect textures (see 05).
- Note the absence of `blocks/` — a pack is NOT required to define blocks.

## fork_of_kompot — a pack distributing a Lua framework

Kompot is a declarative-UI framework pack ("Jetpack-Compose-like") distributed
AS a content pack. Kompot's own framework internals stay out of scope, but the
engine-side HUD/UI mechanics its files use (`scripts/hud.lua`, `layouts/`,
`config/bindings.toml`) are now covered in 09-hud-and-ui.md. `[OBSERVED]`
throughout.

```text
fork_of_kompot/
├── package.json            # id "kompot", version "1.2.2", "license": "MPL-2.0"
├── icon.png
├── preload.json            # 28 font entries + 2 custom atlases
├── modules/                # Lua libraries: modules/kompot.lua + modules/kompot/...
├── scripts/hud.lua         # HUD entry script — on_hud_open() (see 09)
├── layouts/                # gallery.xml + gallery.xml.lua (layout + companion script, see 09)
├── config/bindings.toml    # [kompot] gallery = "key:f12"
├── texts/                  # en_US.txt, ru_RU.txt — incl. dotted keys
├── fonts/                  # IBM Plex TTFs (out of scope)
├── textures/               # UI skin/icons textures (out of scope)
├── annotations/            # editor/LLM Lua annotations (out of scope)
├── docs/, art/, LICENSES/, CHANGELOG.md, README.md   # repo hygiene, engine ignores
```

Takeaways relevant to pack authors:

- `package.json` extra field `"license"` is tolerated (engine ignores unknown
  fields). Metadata for humans goes fine there.
- A pack CAN ship pure code/assets with zero blocks and zero items — Kompot has
  no `blocks/` or `items/` folder at all.
- `preload.json` fonts use object entries:
  `{"name": "kompot_12", "path": "fonts/IBMPlexSans-Regular.ttf", "size": 12}`
  — font NAMES are how other content references them.
- `config/bindings.toml` binds a pack action to a key:
  ```toml
  [kompot]
  gallery = "key:f12"
  ```
- Localization files mix content-caption style keys and dotted keys:
  `kompot.gallery=Kompot / Gallery`.
- Consuming packs declare `"dependencies": ["kompot"]` and call its modules via
  `require "kompot:kompot"` — modules are namespaced by pack id (Lua topic).

## Which files in these repos are engine-relevant vs ignored

Engine-relevant (affect loading/content): `package.json`, `icon.png`,
`preload.json`, `content.json` (generated), `blocks/`, `items/`,
`block_materials/`, `textures/`, `texts/`, `config/`, plus the out-of-scope
content folders listed in 02.

Ignored by the engine (repo hygiene, safe to keep for humans/LLMs): `README.md`,
`CHANGELOG.md`, `LICENSE`, `LICENSES/`, `docs/`, `art/`, `annotations/`,
`*.toml` at project level. DO keep them — they cost nothing and help everyone.

## When to copy what

- Starting a new pack → copy the *shape* of `my_pack` (minimal + only needed folders).
- Need items with behavior → look at `my_pack/items/*` + its scripts, then read
  engine scripting docs before writing your own.
- Need the game-shell around packs → that is a project; out of scope, see stub.
- Need UI → Kompot's own docs (`docs/` inside fork_of_kompot) are the entry point.
