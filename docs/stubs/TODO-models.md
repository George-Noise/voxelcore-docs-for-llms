# STUB — 3D Models (NOT COVERED YET)

Status: **intentionally out of scope of this guide. TODO: write a full document.**

What belongs here when written: `models/` folder, VCM model format, OBJ usage,
item models (`model-name` default `packid:itemname`), block custom models
(`model: "custom"` + `model-name`), textures inside models (`blocks:name`
references into the blocks atlas).

Until this stub is replaced:

- DO NOT guess model file formats or conversion tooling.
- What you MAY rely on from in-scope docs: items accept `model-name`
  (auto-generated when omitted) `[DOC]`; blocks accept `model: "custom"` +
  `model-name` without `<pack>:models/` prefix or extension `[DOC]`; model
  textures must be `blocks:<name>` `[DOC]`.
- Authoritative sources instead:
  - `<voxelcore repo>/doc/ru/block-models.md`, `doc/ru/vcm.md`
  - Real usage: `content/my_pack/models/*.vcm|*.obj` in
    <https://github.com/George-Noise/wooden_meat_game>
