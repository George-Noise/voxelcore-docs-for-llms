# 14. Models (VCM, OBJ, Custom Block and Item Models)

> **MANDATORY PREREQUISITES — DO NOT SKIP.** This file assumes you have read
> [01-engine-overview.md](01-engine-overview.md) AND
> [08-pitfalls.md](08-pitfalls.md). 01 defines the ID rules, reserved names
> and trust labels this file relies on; 08 lists the real failure modes
> behind every DO/AVOID note. For the block `texture-faces` order read
> [03-blocks.md](03-blocks.md); for entity/skeleton usage read
> [10-entities.md](10-entities.md). If you received only this file, fetch
> and read the prerequisites before writing any code.

This document covers 3D models shipped by a content pack: the `models/`
folder, the VCM text format, OBJ usage, custom block models, item models
and VCM-embedded skeletons. Models are what make a block look non-cubic,
an item appear in the player's hand, and an entity take shape.

## Where model files live and how names resolve

| Fact | Detail |
| --- | --- |
| Folder | `models/` in the pack folder — engine constant `MODELS_FOLDER` `[VERIFIED: engine source]` |
| Asset name | the file stem: `models/box.vcm` → model `"box"` |
| Extension order | `.vec3` → `.obj` → `.xml` → `.vcm` — first EXISTING file wins `[VERIFIED: engine source, assetload::model]` |
| Missing file | hard failure `could not to find model "<name>"` — breaks pack/world load `[VERIFIED: engine source]` |
| Bare-name resolution | `paths.find()` across ALL packs by content-source priority — flat, collision-prone namespace, same idea as sounds |

Rules:

1. `.vec3` and `.xml` are legacy engine formats. DO NOT create them in
   packs; ship VCM or OBJ only. `[DOC]`
2. Model names containing `:` are NEVER auto-loaded for blocks, items or
   entities — the loader skips them (they exist for engine-internal
   assets). A pack CANNOT write `other_pack:model` into `model-name` and
   expect it to load. `[VERIFIED: engine source, AssetsLoader::addDefaults]`
3. Consequence of rule 2: model names are bare and global. Two packs
   shipping `models/box.vcm` means the higher-priority pack's file wins
   for BOTH, silently. DO make model names unique — subfolder-style names
   work: `"turret/base"` loads `models/turret/base.vcm`.
   `[VERIFIED: engine source]`
4. `preload.json` has a `"models"` key for EXPLICIT preloading:
   `"models": ["name", {"name": "name", "squash": true}]`. Only needed
   for multi-part (boned) VCMs you want usable as ONE model — see VCM
   bones. Models referenced by blocks/items/skeletons load automatically.
   `[VERIFIED: engine source]`

## Who triggers loading (auto-requests)

| Referenced by | Auto-load rule |
| --- | --- |
| Block variant `"model": "custom"` + `"model-name"` | requested when the name has no `:` |
| Item `"model-name"` | requested when the name has no `:` |
| Entity `"skeleton-name"` | requested as a MODEL (expected: VCM with bones) when the name has no `:` |
| `skeletons/*.json` bone `"model"` fields | model per bone, name stripped of the `.<bone>` suffix |
| `preload.json` `"models"` | loaded explicitly at startup |

`[VERIFIED: engine source, AssetsLoader::addDefaults + assetload::skeleton]`

## Block custom models

Block JSON (full block reference in 03-blocks.md):

```json
{
    "model": "custom",
    "model-name": "destroyer_belt"
}
```

`model-name` is read ONLY when `"model"` is `"custom"` — on any other model
type it is ignored. `[VERIFIED: user first-hand]`

| `model` value | Meaning |
| --- | --- |
| none | invisible block |
| block | default cube (default) |
| xsprite | X-shape (grass) |
| aabb | box shaped as the block hitbox |
| custom | custom model via `model-name` |

`[VERIFIED: engine source, Block.hpp]` A typo in the `model` value is logged
(`unknown model: <value>`) and the block silently becomes `none` —
invisible, NO load error. `[VERIFIED: engine source, BlockLoader]`

The `"model-primitives"` JSON property still works but is officially
deprecated in favor of VCM — DO NOT use it in new packs.
`[DOC: block-models.md WARNING]`

### Textures inside models

| Texture string | Resolved as |
| --- | --- |
| `$0` .. `$5` | the BLOCK's own texture-faces, SAME order as the texture-faces JSON |
| `blocks:<name>` | texture from the blocks atlas (UVs remapped into the atlas region) |
| bare name (`models/box`) | standalone texture auto-loaded from `textures/<name>` of the packs |

`[VERIFIED: engine source, ModelsGenerator + ContentGfxCache +
request_textures]` `$N` binding is a BLOCK-model mechanism — in item or
skeleton models a `$N` mesh texture is never bound (see Item models).

Verified details:

- Box sides default to `$0`..`$5`; tri/rect default to `$0`.
- `$N` with N outside 0..5 stays unbound — silently no texture.
- A bare texture name without `textures/<name>.png` is logged at load and
  the mesh stays untextured — NOT a hard failure.
  `[VERIFIED: engine source, assetload::texture]`

Scale: one block = 1.0 unit; a block custom model should fit the
1×1×1 cube (the example `box.vcm` spans 0..1). Coordinates are NOT
normalized. `[OBSERVED: stock and example packs]`

## VCM format

A text format describing a model from primitives; replaces the deprecated
`model-primitives`. Syntax: `@primitive attr (v1,v2,...) attr2 "string" {
nested... }` — think XML without a single root; multiple root elements are
allowed. `[DOC: vcm.md]`

> By default texture coordinates derive from primitive sizes. Above 1.0
> the result is UNDEFINED and may change between engine versions — use
> `region` or `region-scale`. `[DOC: vcm.md NOTE]`

### `box`

```vcm
@box from (0, 0, 0) to (1, 1, 1) texture "entities/nepis" {
    @part tags (north) texture "entities/nepis_face"
}
```
(from `wooden_meat_game`, `models/nepis.vcm` — an entity texture example)

| Property | Meaning |
| --- | --- |
| from / to | opposite corners of the box |
| origin | rotation pivot; default: box center |
| rotate | degrees per axis `(45,0,0)` or quaternion `(x,y,z,w)` |
| texture | texture for ALL sides; default: `$0`..`$5` per side |
| shading | `shading off` disables shading |
| delete | remove sides by name: top, bottom, east, west, north, south |

`@part` (nested in `box`) overrides per-side properties:

| Property | Meaning |
| --- | --- |
| tags | sides to affect: top, bottom, east, west, north, south |
| texture | side texture; default: `$<side index>` |
| region / region-scale | UV region inside the chosen texture |

Side indices match the texture-faces order `[VERIFIED: engine source,
vcm.cpp side_indices]`:

| Index | Side | Face |
| --- | --- | --- |
| 0 | west | −X |
| 1 | east | +X |
| 2 | bottom | −Y |
| 3 | top | +Y |
| 4 | north | −Z |
| 5 | south | +Z |

### `rect` (parallelogram)

| Property | Meaning |
| --- | --- |
| from | start point |
| right / up | X and Y vectors — also define width/height |
| texture / region / region-scale | as above; default `$0` |
| shading / flip / normal / cull-face | `flip v` mirrors the UV region vertically |

### `tri` (triangle)

| Property | Meaning |
| --- | --- |
| a / b / c | corner points, e.g. `a (0,0,0)` |
| uv | six numbers, 2 per point: `uv (0,0, 1,0, 1,1)` |
| texture / region / region-scale | as above; default `$0` |
| shading / normal / cull-face | as above |

Tri UVs do NOT depend on shape or size (unlike rect/box). `[DOC]`

Shared primitive properties `[DOC: vcm.md]`:

| Property | Meaning |
| --- | --- |
| `shading off` | disable shading on the primitive |
| `normal (x,y,z)` | override the normal vector |
| `cull-face front` / `cull-face off` | which side is culled; `off` = two-sided |

## VCM bones (a skeleton inside the model)

`@bone` wraps primitives and other bones `[DOC: vcm.md]`:

- WITH `name` → enters the model's skeleton (rigging-compatible).
- WITHOUT a name → "dissolved": its transforms apply to children, nothing
  enters the skeleton.
- A `root` bone is created automatically for the whole model.
- Bone properties: `move`, `scale` (applied BEFORE move), `rotate`.

Verified consequences `[VERIFIED: engine source, load_vcm_model]`:

1. A VCM with named bones (more than one part, not squashed) stores its
   parts as `<model>.<bone>` assets plus a skeleton under the bare model
   name — and NO single model under the bare name. DO NOT reference such
   a model from a block or item: the model will not be found. Use it as
   an entity skeleton (10-entities.md) instead.
2. `"squash": true` in `preload.json` merges the VCM back into ONE model
   under the bare name — usable as a block/item model, no skeleton.
3. Unnamed bones keep the model single-part — safe for blocks/items.

## OBJ

Verified against the engine's parser, the supported subset is: `v`, `vn`,
`vt`, `f` and `usemtl`. `[VERIFIED: engine source, obj.cpp]`

- `usemtl <name>` — the name IS the texture reference (same three forms:
  `$N` / `blocks:name` / bare name). MTL FILES ARE NOT PARSED — `mtllib`
  lines are ignored and no `.mtl` file is needed. Real example:
  `my_pack/models/shotgun.obj` (a Blender 4.0.2 export) references
  `shotgun.mtl` that is not even shipped; its `usemtl Material` resolves
  to `textures/Material.png`, which is shipped. `[OBSERVED]`
- A bare texture name with no `textures/<name>.png` logs an error at load
  and the mesh stays untextured (not a hard failure).
  `[VERIFIED: engine source, assetload::texture]`
- Models are used at their authored scale; block custom models should fit
  the 1×1×1 cube. `[OBSERVED]`

## Item models

`model-name` on an item (04-items.md) selects the 3D model shown in the
player's hand: the engine's own `hud.lua` renders `item.model_name(id)` on
the hand bone. `[VERIFIED: engine source + res/scripts/hud.lua]`

| Case | Model used |
| --- | --- |
| No `model-name` | a GENERATED model stored as `<packid>:<item>.model`: block-icon items reuse the block's model (scaled to 0.2); sprite-icon items get a flat quad textured with the icon |
| `"model-name": "shotgun"` | `models/shotgun.*` of the packs, auto-loaded (no `:` in the name) |

`[VERIFIED: engine source, ModelsGenerator + ItemDef]` The generated
default name contains `:`, so it is never file-loaded — it exists only as
the generated asset. `[VERIFIED]`

Real example (`wooden_meat_game`, `content/my_pack/items/shotgun.json`):
`[OBSERVED]`

```json
{
    "icon-type": "sprite",
    "icon": "items:shotgun",
    "stack-size": 1,
    "model-name": "shotgun"
}
```

## Models and entities

- ALL files in a pack's `skeletons/` folder are auto-loaded as skeletons
  named `<packid>:<file stem>` — no preload entry needed.
  `[VERIFIED: engine source]`
- Entity `skeleton-name` defaults to the entity's own id — so entity
  `my_pack:turret` automatically picks up the skeleton `my_pack:turret`
  from `skeletons/turret.json`. That name coincidence is how the example
  pack wires its turret and nepis models without any explicit field.
  `[VERIFIED: engine source, Entities.cpp]`
- `skeleton-name` may also point at a bare VCM name — the engine then
  expects a VCM WITH bones and requests it as a model.
  `[VERIFIED: engine source]`
- A missing skeleton: WARNING if the name equals the entity id, hard
  ERROR otherwise. `[VERIFIED: engine source, Entities.cpp]`
- Skeleton bones reference models by name (stripped of the `.<bone>`
  suffix) and those models auto-load — `skeletons/turret.json` pulls
  `models/turret.vcm` and `models/shotgun.obj`. `[VERIFIED: engine source,
  assetload::skeleton]`

Stock example: `res/content/base/entities/falling_block.json` uses
`"skeleton-name": "base:block"`. `[OBSERVED]` Bone ANIMATION is covered in
11-animation.md; the skeleton JSON format in 10-entities.md and
`doc/ru/rigging.md`.

## Pitfalls (models-specific)

1. **Missing model file breaks world load.** `could not to find model`
   is a hard failure for any auto-requested or preloaded model. Keep
   references and files in sync. `[VERIFIED: engine source]`
2. **Flat global namespace with no pack prefix.** Bare model names resolve
   across packs by priority; a colliding name silently shadows another
   pack's model — and you CANNOT fix it with `pack:name` (auto-load skips
   `:` names). DO use unique names or subfolders.
   `[VERIFIED: engine source]`
3. **`model-name` on a non-custom block.** Read only when
   `"model": "custom"`; otherwise ignored. `[VERIFIED: user first-hand]`
4. **Typo in the `model` value.** Logged as `unknown model` and the block
   becomes `none` — invisible, no load error. Copy values from this table,
   not from memory. `[VERIFIED: engine source]`
5. **Named-bone VCM as a block/item model.** Such a file stores parts as
   `<model>.<bone>` and NO model under the bare name — the block/item
   model is missing. DO use `"squash": true` or keep bones unnamed.
   `[VERIFIED: engine source]`
6. **`$N` outside 0..5 or outside block models.** Silently unbound — no
   texture, no error. `[VERIFIED: engine source]`
7. **Expecting MTL materials.** The OBJ parser never reads `.mtl`;
   `usemtl` names are texture asset names. `[VERIFIED: engine source]`
8. **VCM UVs beyond 1.0.** Undefined per upstream; use `region` /
   `region-scale`. `[DOC]`
9. **Missing model textures fail silently.** Logged at load, mesh stays
   untextured. Check the log when a model renders untextured.
   `[VERIFIED: engine source]`
10. **Legacy extensions.** `.vec3`/`.xml` win over `.vcm` in the extension
    order — an old file with the same stem silently overrides your VCM.
    DO NOT ship them in packs. `[VERIFIED: engine source]`

## DO / AVOID summary

- DO ship `.vcm` (or `.obj`) under `models/` with unique names (subfolders
  count as part of the name).
- DO use `$0`..`$5` in block custom models to reuse the block's
  texture-faces; `blocks:name` for atlas textures; bare names only with a
  matching file under `textures/`.
- DO keep block/item models single-part; use named bones only for entity
  skeletons.
- DO export OBJ without expecting `.mtl` to be read; name `usemtl` after
  your texture.
- AVOID `model-primitives` in new packs (deprecated by VCM).
- AVOID `:` inside your own model names (auto-load skips them).
- AVOID default UVs beyond 1.0 — use `region` / `region-scale`.
- AVOID shipping `.vec3`/`.xml` models in packs.

## Upstream pointers (read before writing real models)

All paths are files in the VoxelCore engine repository
(github.com/MihailRis/voxelcore), NOT in this guide. The Russian docs are
usually richer and fresher — prefer `doc/ru/` when they disagree.

| Topic | EN | RU |
| --- | --- | --- |
| VCM format, primitives, bones | `doc/en/vcm.md` | `doc/ru/vcm.md` |
| Deprecated `model-primitives` | `doc/en/block-models.md` | `doc/ru/block-models.md` |
| Skeleton/rigging format | `doc/en/rigging.md` | `doc/ru/rigging.md` |
| Entity `skeleton-name` | `doc/en/entity-properties.md` | `doc/ru/entity-properties.md` |

Where this guide goes beyond the upstream text (extension order, auto-load
rules, the flat namespace and the `:` skip, texture binding `$N` /
`blocks:` / bare name, the single-part storage rule, MTL behavior, item
model generation), the claims were verified against the engine source
(`AssetsLoader`, `assetload_funcs`, `vcm.cpp`, `obj.cpp`, `ModelsGenerator`,
`ContentGfxCache`, `Entities`) at the time of writing — re-verify against
your engine version if behavior seems to differ.

Real usage to learn from: `wooden_meat_game` — `content/my_pack/models/`
(`box.vcm`, `nepis.vcm`, `turret.vcm`, `shotgun.obj`), `skeletons/`
(`turret.json`, `nepis.json`), `items/shotgun.json`, `textures/`
(`Material.png`, `models/`, `items/`); engine stock `res/models/`
(`block.vec3`, `drop-item.vec3`, `human.vcm`) and
`res/content/base/entities/falling_block.json` (`skeleton-name`).
