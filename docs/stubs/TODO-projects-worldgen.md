# STUB — Projects & World Generators (NOT COVERED YET)

Status: **intentionally out of scope of this guide. TODO: write full documents.**

Two related-but-separate topics were excluded:

## Projects (`project.toml`)
What belongs here: project manifests (`name`, `title`, `base_packs`,
`permissions`), running with `--project <path>`, project-level scripts
(`start.lua`, `project_client.lua`), `project:content` source.

Known facts already documented in-scope: `project.toml` with `base_packs =
["base", ...]` enables default packs `[OBSERVED]`; `project:content` has
priority above `res:content` `[DOC]`.

Authoritative sources: `<voxelcore repo>/doc/ru/projects.md` and real example
<https://github.com/George-Noise/wooden_meat_game> (root `project.toml`).

## World generators (`generators/`)
What belongs here: generator TOML + scripting (`demo.toml`, `demo.files` in the
stock pack), biome/ore/height configuration.

Authoritative sources: `<voxelcore repo>/doc/ru/world-generator.md` (RU only —
the largest engine doc, absent from EN set).

Until these stubs are replaced:

- DO NOT guess project flags or generator TOML fields.
- A typical ADD-ON content pack needs neither — a pack is usable from the
  content menu of any world.
