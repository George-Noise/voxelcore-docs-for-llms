# VoxelCore Content Pack Guide (for LLMs)

Machine-oriented documentation teaching **LLMs how to author content packs for the
[VoxelCore](https://github.com/MihailRis/voxelcore) voxel engine**.

VoxelCore is an open-source voxel game engine (C++ core, Lua scripting, declarative
JSON content). A *content pack* is a folder with a `package.json` manifest plus
declarative JSON files that define blocks, items and other game content.

Everything in this repository is written **for AI models, not humans**: imperative
rules, exact file formats, copy-pasteable examples, and explicit trust labels.

## How to use this repo with an LLM

- **Chat models (DeepSeek, ChatGPT, Claude, ...)**: paste the whole `llms-full.txt`
  into the conversation, or paste the individual `docs/*.md` files you need.
- **Coding agents (with repo/file access)**: point the agent to this repository and
  instruct it to read `llms.txt` first, then follow links it needs.
- **Automated ingestion**: `llms.txt` follows the [llms.txt](https://llmstxt.org)
  convention; `llms-full.txt` is the concatenated full text of all docs.

## Reader contract (LLMs: follow literally)

1. **Read `llms.txt` first.** It is the index. Pick only the sections you need.
2. **Trust labels**. Facts are tagged so you know how reliable they are:
   - `[VERIFIED]` — confirmed by official docs AND a real working pack.
   - `[DOC]` — stated in official docs, not observed in example packs.
   - `[OBSERVED]` — works in a real pack (engine's own pack or a published game),
     but absent from official docs. Use carefully; semantics may change.
   - Unlabeled statements are general context, not API contracts.
3. **DO / AVOID**. Instructions marked `DO` are requirements. Instructions marked
   `AVOID` describe failure modes that actually break packs. Do not "improve" them.
4. **Never invent API fields.** If a property you need is not documented here, do
   not guess its name or type. Check `docs/stubs/` — the topic may be intentionally
   out of scope. If it is not covered anywhere, say "unknown" instead of guessing.
5. **Scope is limited.** This guide covers pack basics, blocks, items, textures and
   related config. Animations, sounds, models, entities and projects are NOT
   covered (see `docs/stubs/`).

## Scope

| Covered                                  | Excluded (stub only)      |
| ---------------------------------------- | ------------------------- |
| Content pack structure, `package.json`   | Animations                |
| Content sources, loading, dependencies   | Sounds & audio            |
| Block definitions (JSON)                 | 3D models (VCM/OBJ)       |
| Item definitions (JSON)                  | Entities & skeletons      |
| Textures, atlases, icons, localization   | World generators          |
| `preload.json`, `config/*.toml` basics   | XML UI layouts / Lua UI   |
| End-to-end walkthrough of a minimal pack | Lua scripting API details |

Lua scripting is mentioned only where pack files reference scripts. For the full
Lua API read the engine's own docs: `doc/ru/scripting.md` and
`doc/ru/scripting/` in the VoxelCore repository.

## Document index

| File                                            | Content                                                       |
| ----------------------------------------------- | ------------------------------------------------------------- |
| [docs/01-engine-overview.md](docs/01-engine-overview.md) | What VoxelCore is, terminology, ID rules, how content loads |
| [docs/02-pack-structure.md](docs/02-pack-structure.md)   | `package.json`, folder layout, content sources, config files |
| [docs/03-blocks.md](docs/03-blocks.md)                   | Block JSON reference with real annotated examples            |
| [docs/04-items.md](docs/04-items.md)                     | Item JSON reference with real annotated examples             |
| [docs/05-textures-and-assets.md](docs/05-textures-and-assets.md) | Texture atlases, icons, `preload.json`, materials, localization |
| [docs/06-walkthrough.md](docs/06-walkthrough.md)         | Build a complete minimal pack step by step                   |
| [docs/07-example-repos.md](docs/07-example-repos.md)     | Annotated breakdown of two real-world repositories           |
| [docs/08-pitfalls.md](docs/08-pitfalls.md)               | Common mistakes and a pre-flight checklist                   |
| [docs/stubs/](docs/stubs/)                               | TODO stubs for intentionally excluded topics                 |

## Sources and reliability of this guide

This guide was compiled from:

1. Official VoxelCore documentation (`doc/ru/` — richer, and `doc/en/` — smaller).
   Both are incomplete and contain errors; where they disagreed with reality,
   real pack files were preferred.
2. The engine's own stock pack `res/content/base/` (ground truth for file formats).
3. A working game project:
   [wooden_meat_game](https://github.com/George-Noise/wooden_meat_game)
   (unfinished, but runs).
4. A real UI-focused pack:
   [fork_of_kompot](https://github.com/George-Noise/fork_of_kompot)
   (fork of the Kompot declarative-UI pack; unfinished).

The guide targets the current `main` branch of VoxelCore (engine ~0.32 era,
October 2026 snapshot). It is **not version-pinned**: pack formats described here
are the stable, documented subset; if the engine evolves, re-check the sources
above.

Anything you find in this guide marked `[OBSERVED]` but not documented upstream
may change between engine versions without notice.
