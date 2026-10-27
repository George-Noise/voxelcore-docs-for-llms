# STUB — Entities & Skeletons (NOT COVERED YET)

Status: **intentionally out of scope of this guide. TODO: write a full document.**

What belongs here when written: `entities/*.json` definitions, entity
properties, `skeletons/*.json`, ECS components in `scripts/components/`,
spawning from items (spawn-egg pattern).

Until this stub is replaced:

- DO NOT guess entity property names or the component API.
- What you MAY rely on from in-scope docs: `config/defaults.toml` can bind
  `player-entity` (stock: `"base:player"`) `[OBSERVED]`; blocks/items can
  reference entities only through Lua (out of scope).
- Authoritative sources instead:
  - `<voxelcore repo>/doc/ru/entity-properties.md`
  - `<voxelcore repo>/doc/ru/scripting.md` + `doc/ru/scripting/core_components.md`
  - Real usage: `content/my_pack/entities/`, `skeletons/`, `scripts/components/`
    in <https://github.com/George-Noise/wooden_meat_game>
