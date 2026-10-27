# STUB — Sounds & Audio (NOT COVERED YET)

Status: **intentionally out of scope of this guide. TODO: write a full document.**

What belongs here when written: `sounds/` folder formats (OGG etc.), sound name
variants pattern, `audio.*` Lua API, positional/2D playback, audio in
`preload.json`.

Until this stub is replaced:

- DO NOT guess sound file requirements or playback API.
- What you MAY rely on from in-scope docs: block materials reference sound
  names (`steps-sound`, `place-sound`, `break-sound`, `hit-sound` — see
  `docs/05-textures-and-assets.md`), and `preload.json` `"sounds"` entries load
  `name.ogg`, `name_1.ogg`, ... variants `[DOC]`.
- Authoritative sources instead:
  - `<voxelcore repo>/doc/ru/audio.md` (RU only — richer than EN)
  - `<voxelcore repo>/doc/en/audio.md`
  - Real usage: `content/sfx/` and `content/my_pack/sounds/` in
    <https://github.com/George-Noise/wooden_meat_game>
