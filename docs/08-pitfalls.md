# 08. Pitfalls and Pre-flight Checklist

Every entry below is a real failure mode — either documented upstream, thrown
by the engine, or observed in real packs. Treat each AVOID as a hard rule.

## Fatal mistakes

1. **Editing `content.json`.** It is auto-generated and used to convert world
   saves. Manual edits corrupt worlds irreversibly (upstream warning).
   AVOID creating, editing, or deleting it in source repos.
2. **Missing `package.json`.** A folder without it is not a pack — the scanner
   skips it. DO always start from the manifest.
3. **Invalid pack ID.** Outside `a-z0-9_`, starting with a digit, or shorter
   than 2 / longer than 24 chars breaks registration. Reserved IDs (`res`,
   `user`, `world`, `project`, `config`, `core`) are unavailable. `[VERIFIED]`
4. **`texture-faces` with fewer than 6 entries.** The engine throws
   "texture-faces requires 6 elements" and the pack fails to load.
   `[OBSERVED: engine source]`
5. **Using `texture` and `texture-faces` together.** Documented as incompatible;
   pick one. `[DOC]`
6. **Paths or extensions in texture names.** `"texture": "blocks/dirt.png"` is
   wrong. The value is a NAME resolved inside an atlas: `"dirt"` or
   `"blocks:dirt"` depending on context. `[VERIFIED]`
7. **Using `base:*` properties without depending on `base`.** `base:durability`
   and `base:loot` are registered by the `base` pack; without
   `"dependencies": ["base"]` they are not guaranteed to exist. `[VERIFIED]`
8. **Loot referencing a block instead of its item.** `base:loot[].item` needs an
   ITEM id; a block's item is `<pack>:<block>.item`. `[VERIFIED]`

## Likely-wrong assumptions (seen in practice)

9. **Assuming folder name = pack ID.** The engine keys packs by the `"id"` in
   `package.json`; the folder name is free-form and has NO meaning to the
   engine. In real packs the folder name almost always DIFFERS from the ID
   (e.g. folder `kompot_0f9103ff-1.2.2` ships `"id": "kompot"`), and exact
   matches are rare. NEVER infer a pack's ID from its folder name — read the
   manifest. `[OBSERVED]`
10. **Assuming `icon.png` must be 128x128.** It is a recommendation; 64x64 and
    210x208 exist in working packs. Keep it square. `[OBSERVED]`
11. **Assuming texture sizes are enforced.** Stock uses 16x16; other sizes are
    uncharted. DO stay 16x16 for blocks/items.
12. **Assuming `version` matters for pack resolution.** When two sources carry
    the same pack ID, priority decides; version is ignored. `[DOC]`
13. **Assuming a pack needs content.** A folder containing ONLY `package.json`
    is a complete, valid, loadable pack — nothing else is required. Blocks,
    items, textures, config and code are all optional extras (Kompot ships no
    blocks/items; confirmed first-hand). `[VERIFIED]`
14. **Inventing dependency version operators.** Only `@>=x.y` and `*` are
    documented. AVOID `<`, `<=`, `=`, ranges. `[DOC]`
15. **Capitalizing captions yourself.** Docs recommend lowercase captions; the
    engine upper-cases for display. Localization keys must match the lowercase
    caption string. `[DOC]`
16. **Preloading auto-loaded assets.** Block/item atlas textures are picked up
    automatically; preloading them is noise (upstream warns to keep preload
    minimal). `[DOC]`
17. **Putting `pack_id:` into `<image src>` or `menu.page`.** These do NOT use
    the pack namespace: `src` without a colon is a global texture name; `src`
    with a colon is `atlas_name:tile_name`; `menu.page` takes a bare page name
    scanned across all packs. Pack-qualified ids belong only to layout ids
    (`hud.show_overlay("pack:layout")`) and document paths
    (`gui.load_document`). See 09-hud-and-ui.md. `[VERIFIED]`
18. **Spelling the animation folder or preload key `animations`.** The engine
    constant is `animation` (singular): pack folder `animation/`, `preload.json`
    key `"animation"`. An upstream quirk you must reproduce. `[VERIFIED]`

## JSON hygiene

- AVOID trailing commas and comments — block/item files are strict JSON.
- DO keep UTF-8 (localization values are non-ASCII routinely).
- DO keep one definition per file; the file name is the content name.

## Pre-flight checklist (run before delivering a pack)

Manifest:
- [ ] `package.json` exists, parses, has `id`/`title`/`version`.
- [ ] `id` passes rules and is not reserved. Folder name is free-form — it
      does NOT need to equal `id`.
- [ ] `dependencies` lists every pack whose `pack:` references you use.

Content:
- [ ] Every `blocks/*.json` and `items/*.json` uses only properties documented
      in 03/04 (or upstream docs) — nothing invented.
- [ ] `texture-faces` arrays have exactly 6 entries in west/east/bottom/top/
      north/south order.
- [ ] No `texture` + `texture-faces` in the same block.
- [ ] Every texture/icon name resolves to a real PNG in `textures/blocks/` or
      `textures/items/`.
- [ ] `base:*` properties used only with `base` dependency; loot items use
      `.item` suffix where needed.

Assets & config:
- [ ] `icon.png` present at pack root (square).
- [ ] `preload.json` (if present) contains only genuinely needed entries.
- [ ] `texts/*.txt` keys match captions exactly (lowercase, spaces not
      underscores).
- [ ] No `content.json` committed from your side.

UI (09):
- [ ] Layout XMLs sit at `layouts/*.xml` (flat folder); menu pages at
      `layouts/pages/`.
- [ ] No pack id in `menu.page` names or in `<image src>` values.
- [ ] Animation folder/key spelled `animation` (singular), if used.

Runtime:
- [ ] Pack loads without errors in the engine log/console.
- [ ] Blocks appear in the content/inventory screen and place correctly.
- [ ] Loot drops, lights emit, plants sit on ground — as designed.

## Debugging hints

- Load errors are printed by the engine with the failing pack and file name
  (e.g. unknown model, JSON parse errors, missing textures render as
  `notfound`). Read the console output first. `[OBSERVED]`
- A pack that loads but shows missing textures/`notfound` icons indicates a
  name/folder mismatch (03/05 rules), not a manifest problem.
- The in-game console and engine logging are documented in the engine repo
  (`doc/ru/console.md`, mirrored as `doc/en/console.md`) — out of scope here.
