# 09. HUD and UI Basics (layouts, pages, overlays)

> **MANDATORY PREREQUISITES — DO NOT SKIP.** This file assumes you have read
> [01-engine-overview.md](01-engine-overview.md) AND
> [08-pitfalls.md](08-pitfalls.md). 01 defines the ID rules, reserved names
> and trust labels this file relies on; 08 lists the real failure modes
> behind every DO/AVOID note. Skipping either is the most common reason
> models generate broken packs from this guide. If you received only this
> file, fetch and read both before writing any code.

This document covers the PACK-SIDE mechanics of shipping UI: where layout files
live, how the engine registers and addresses them, and the ways to put them on
screen. It intentionally does NOT duplicate the full XML element reference or
the Lua GUI API — pointers to the authoritative upstream docs are at the end.

Minimal Lua appears here because UI cannot be triggered without at least one
script. Everything else stays declarative.

## The one rule that prevents most mistakes

Blocks and items live in ONE namespaced registry (`pack_id:name`, see
01-engine-overview.md). UI does NOT use that registry uniformly — it mixes
THREE different addressing schemes (document ids, menu page names, texture
names; the document-id scheme comes in two flavors, see rows 1–2):

| What you write | Scheme | Example |
| --- | --- | --- |
| Layout document id (`hud.*`) | `pack_id:file_stem`, derived by the engine | `kompot:gallery` |
| Document alias (`gui.load_document` arg 2, iframe `src`) | any alias YOU invent | `my_pack:extra_panel` |
| Menu page name (`menu.page`) | bare name, NO pack id | `settings` |
| `<image src>` texture | global name OR `atlas:tile` | `gui/error`, `blocks:dirt` |

DO memorize this table. The most common LLM errors are putting `pack_id:` into
`menu.page` or `<image src>`, or omitting it in `hud.show_overlay`. `[VERIFIED]`

## UI-relevant folder layout

```text
my_pack/
├── layouts/
│   ├── hello.xml              # auto-registered as document "my_pack:hello"
│   ├── hello.xml.lua          # optional companion script (same name + .lua)
│   ├── pages/                 # menu pages for pagebox (menu.page)
│   │   ├── greeting.xml
│   │   └── some_dir/          # subfolders ARE allowed inside pages/
│   │       └── extra.xml      #   -> menu.page = "some_dir/extra"
│   └── templates/             # templates for gui.template()
├── scripts/
│   └── hud.lua                # per-pack HUD entry script (on_hud_open)
└── config/
    └── bindings.toml          # key bindings (format in 02-pack-structure.md)
```

Registration rules (`[VERIFIED: engine source]`):

- Every TOP-LEVEL `layouts/*.xml` of every pack is auto-loaded as a UI document
  named `<pack_id>:<file_stem>`. `layouts/hello.xml` -> `my_pack:hello`.
- The scan is FLAT: `layouts/pages/` and `layouts/templates/` are NOT
  auto-registered this way (they are consumed by `menu.page` and
  `gui.template()` respectively).
- There is NO `layouts` key in `preload.json`. Layouts are always loaded
  automatically; the preload keys are exactly: `animation`, `atlases`, `fonts`,
  `models`, `post-effects`, `shaders`, `skeletons`, `sounds`, `textures`.
  `[VERIFIED]`
- A companion script named `<file>.xml.lua` (e.g. `hello.xml.lua`) becomes the
  layout's script environment; functions defined there are callable from the
  XML's `onclick` attributes. `[OBSERVED: engine layouts + kompot]`

## Way 1 — HUD overlays and windows (`hud.*`, pack id REQUIRED)

The `hud` Lua library opens pack layouts on top of the game. The layout id is
ALWAYS pack-qualified here: `"pack_id:stem"`. `[DOC + VERIFIED]`

Key functions (full list upstream, see pointers):

```lua
-- Show layout as an overlay. Shows player inventory too if playerinv=true.
-- args (optional table) is passed to the layout's on_open.
hud.show_overlay("my_pack:hello", playerinv, args)

-- Open a layout bound to an inventory (like a chest screen). Returns invid.
-- Throws an exception if the layout does not exist.
hud.open("my_pack:my_window", disable_player_inv, invid)

-- Add a layout to the screen permanently (removed on world close only).
hud.open_permanent("my_pack:status_bar")

hud.close("my_pack:hello")     -- remove layout
hud.is_open("my_pack:hello")   -- -> bool

-- Block UI: opens the ui-layout of the block at x,y,z plus its inventory.
hud.open_block(x, y, z)        -- -> invid, layoutid
```

`[DOC]` for signatures; the `pack_id:stem` id format is `[VERIFIED]` (engine
registers layouts under exactly that name).

### scripts/hud.lua — the HUD entry script

Each pack MAY ship one special script: `scripts/hud.lua`. The engine registers
it as `<pack_id>:scripts/hud.lua` (type `hud`) and calls its `on_hud_open()`
when the in-game HUD opens (world joined). DO register your input callbacks and
open permanent HUD elements there — not at file scope. `[VERIFIED]`

```lua
-- my_pack/scripts/hud.lua
function on_hud_open()
    -- runs every time the HUD (re)opens; safe place for input.add_callback
end
```

Upstream events doc lists `on_hud_open(playerid)`; real packs define it with no
parameters — both forms appear in the wild. `[OBSERVED]`

### Key binding -> callback wiring (complete chain)

A pack action bound to a key is THREE files that must agree:

1. `config/bindings.toml` — section = pack id, key = action name:
   ```toml
   [my_ui_pack]
   hello = "key:h"
   ```
2. `scripts/hud.lua` — callback name is the DOTTED form `pack_id.action`:
   ```lua
   function on_hud_open()
       input.add_callback("my_ui_pack.hello", function()
           -- your toggle logic
       end)
   end
   ```
3. `texts/*.txt` (optional) — display name for the binding, same dotted key:
   `my_ui_pack.hello=Toggle hello overlay`

`[OBSERVED: kompot ships exactly this chain: [kompot] gallery = "key:f12" ->
input.add_callback("kompot.gallery", ...) -> kompot.gallery=Kompot / Gallery]`

### Real-world example (kompot, simplified)

`fork_of_kompot` toggles its gallery overlay with F12 — the exact pattern to
copy: `scripts/hud.lua` registers the callback, then:

```lua
if hud.is_open("kompot:gallery") then
    hud.close("kompot:gallery")
else
    hud.show_overlay("kompot:gallery", false)
end
```

and `layouts/gallery.xml.lua` (the layout's own companion script) calls
`hud.close("kompot:gallery")` from its close button. `[OBSERVED]`

## Way 2 — Menu pages (`menu.page`, NO pack id)

The main menu is a `pagebox` element; the global `menu` object IS the root menu
pagebox. Setting `menu.page` swaps the displayed page. `[OBSERVED: engine
stdlib `menu = _GUI_ROOT.menu`; documented in upstream projects.md]`

```lua
menu.page = "settings"          -- open page "settings"
menu.page = "content?mode=world" -- page with query arguments
menu.page = ""                  -- close the current page
```

How the lookup works (`[VERIFIED: engine source gui_util.lua]`):

1. The engine takes the page name (everything before `?` in the query).
2. It searches `layouts/pages/<name>.xml` across the core and ALL loaded packs
   — a plain virtual-filesystem find. The page name carries NO pack id and
   there is NO syntax to specify one.
3. The FIRST match in the file-system search order wins; the found document is
   loaded as `<pack>:pages/<name>` and its `on_open` receives the parsed query
   arguments.
4. Subfolders work: `menu.page = "some_dir/extra"` loads
   `layouts/pages/some_dir/extra.xml`. `[VERIFIED: format string + user
   confirmation]`

Consequences you MUST respect:

- DO use distinctive page names for your pack's pages. A page named like an
  engine page (`main`, `settings`, `content`, `pause`, `worlds`, ...) will
  SHADOW the built-in one when your pack has search priority (user:content
  ranks above res:content). This override trick is how game projects replace
  the main menu — DO NOT trigger it accidentally. `[OBSERVED]`
- AVOID assuming your page loads "from your pack" — if another pack ships a
  page with the same name and higher priority, its page wins.
- Pages are NOT auto-loaded at startup; they load on demand when addressed.

Pages also get companion scripts: `layouts/pages/greeting.xml` +
`greeting.xml.lua`. Engine pages use `@Label` texts resolved through
localization; plain text strings also work. `[OBSERVED: engine layouts]`

## Way 3 — Direct document loading (advanced)

For custom shells (project main menus, embedded panels):

```lua
-- Load any layout as a document; path IS pack-qualified here. [DOC]
gui.load_document(
    "my_pack:layouts/extra/panel.xml",  -- arg 1: path to the xml (a REAL file on disk)
    "my_pack:extra_panel",              -- arg 2: an ALIAS you invent for the loaded document
    {}                                  -- arg 3: args -> on_open
)

-- Then embed it, e.g. via an iframe element added to the root:
gui.root.root:add(
    "<iframe id='panel_frame' src='my_pack:extra_panel' " ..
    "size-func='unpack(gui.get_viewport())'/>"
)
```

Arg 2 ("my_pack:extra_panel") is an ALIAS, not a path. The engine does not look
it up on disk and does not derive it from the xml file name — you may pick any
string, and it never has to match `panel.xml`. The alias exists purely as the
HANDLE to the loaded document: the iframe `src` above embeds the document by
this exact alias. DO keep aliases unique and pack-prefixed
(`my_pack:whatever`) so two packs cannot pick colliding labels — a convention
this guide recommends, not a verified engine rule. `[VERIFIED: alias semantics
confirmed by the user first-hand]`

`[OBSERVED: wooden_meat_game start.lua does exactly this with its menu pack:
gui.load_document("wooden_meat_menu:layouts/main_menu/main_menu.xml",
"wooden_meat_menu:main_menu") + iframe add]`

Note that wooden_meat's alias (`wooden_meat_menu:main_menu`) happens to equal
`pack_id:file_stem` of its xml — that is their naming choice, NOT an engine
requirement. DO NOT generalize from that example.

Related helpers (all `[DOC]`): `gui.template(name, params)` renders templates
from `layouts/templates/`; `gui.root` is the root document; `gui.show_message`,
`gui.ask`, `gui.alert`, `gui.confirm` show ready-made dialogs.

### Block UI (declarative, no code to open)

Blocks can carry their own screens: give the block `"inventory-size": N` and
optionally `"ui-layout": "pack:layout"` (default: the block's own id ->
`pack/layouts/<block_name>.xml`). Opening is engine-driven (`hud.open_block`,
right-click etc.). See 03-blocks.md for the block properties. `[DOC]`

## XML essentials (the 20% you need most)

Layouts are XML files with a single root element. Element types include:
`container`, `panel`, `button`, `label`, `image`, `textbox`, `checkbox`,
`trackbar`, `select`, `canvas`, `iframe`, `pagebox`, `inventory`, `slot`,
`slots-grid`, `splitbox`, `bindbox`. `[DOC]`

Most-used common attributes (`[DOC]`, full list upstream):

- `id` — element id; ids are global within the document and are how scripts
  reach the element (`document["my_button"]`).
- `pos`, `size`, `min-size` — 2D vectors as strings: `"500,200"`.
- `margin`, `padding` — 4D vectors: `"left,top,right,bottom"`.
- `color` — RGBA hex: `"#FF8000"`, `"#FFFFFF80"`.
- `visible`, `enabled`, `interactive`, `scrollable`, `z-index`.
- `gravity` — auto-positioning inside the container: `top-left` ...
  `center-center` ... `bottom-right`.
- `position-func`, `size-func` — Lua expressions evaluated on layout/resize
  (engine layouts use `unpack(gui.get_viewport())` for fullscreen roots).
- Event handlers: `onclick`, `onrightclick`, `ondoubleclick`, `onfocus`,
  `ondefocus`, `onmouseenter`, `onmouseleave`, ... — Lua expressions, usually
  calls into the companion script (`onclick='close_self()'`).

Script-side access (companion script or any script after load): the global
`document` table indexes elements by id — `document["my_label"].text = "hi"`,
`document["box"]:destruct()`. Properties per element type are listed upstream
(scripting/ui.md). `[DOC]`

Inventory warning (`[DOC]`): `<inventory>` position is engine-controlled
(`pos`/`margin` do not apply), and slots not linked to an inventory id are not
displayed (`inventory_ui.inventory = invid` links them).

## `<image src>` — texture addressing without pack ids

`<image>` resolves `src` through the ASSET system, not the pack namespace.
Resolution rules (`[VERIFIED: engine source, get_texture_region]`):

1. **No colon** in `src` -> looked up as a global TEXTURE asset name:
   `src="gui/error"` -> engine texture `gui/error` (preloaded from the
   engine's own `res/preload.json`). No pack id, no extension.
2. **With colon** -> treated as `atlas_name:tile_name` — an ATLAS lookup, NOT
   a pack id: `src="blocks:dirt"` (blocks atlas), `src="items:shotgun"`,
   `src="kompot_ui_icons:pen"` (custom atlas).
3. Missing texture -> `fallback` attribute is tried; otherwise the image
   renders empty.

How to show YOUR OWN PNG in XML UI — two options:

- Global texture: put the PNG under `textures/` (e.g.
  `textures/gfx/my_pic.png`) and list `"gfx/my_pic"` in `preload.json`'s
  `"textures"` array. Then `src="gfx/my_pic"`. The name is GLOBAL — DO prefix
  it with something pack-specific to avoid collisions (wooden_meat_game uses
  `gfx/muzzle_flash`; kompot prefixes everything `kompot_*`). `[VERIFIED]`
- Custom atlas: declare an atlas in `preload.json` (`"atlases":
  [{"name": "my_pack_ui"}]`), put PNGs into `textures/my_pack_ui/`, and use
  `src="my_pack_ui:<file_stem>"`. `[OBSERVED: kompot does this for its UI
  skin/icons]`

Engine GUI textures live under the reserved-looking `gui/` name — DO NOT name
your textures `gui/...`; the engine ones would collide or confuse.

## Minimal working example (complete pack)

A pack that adds a toggleable overlay on key H. Four files, all complete and
final. Pack name: `my_ui_pack`.

```text
my_ui_pack/
├── package.json
├── config/bindings.toml
├── layouts/hello.xml
├── layouts/hello.xml.lua
└── scripts/hud.lua
```

`package.json`:

```json
{
    "id": "my_ui_pack",
    "title": "My UI Pack",
    "version": "0.1.0",
    "description": "Toggleable hello overlay"
}
```

No `dependencies` needed: `hud`, `input`, `gui` are engine built-ins, and no
`base:*` properties are referenced.

`config/bindings.toml`:

```toml
[my_ui_pack]
hello = "key:h"
```

`layouts/hello.xml`:

```xml
<container
    id="root"
    color="#00000080"
    position-func="0,0"
    size-func="unpack(gui.get_viewport())"
>
    <panel size="320,140" padding="16" interval="8"
           gravity="center-center" color="#507090E0">
        <label text="Hello from my_ui_pack" gravity="top-center"/>
        <button onclick='close_self()'>Close</button>
    </panel>
</container>
```

`layouts/hello.xml.lua`:

```lua
function on_open(args)
    -- args comes from hud.show_overlay(..., args) / gui.load_document(..., args)
    print("hello overlay opened")
end

function close_self()
    hud.close("my_ui_pack:hello")
end
```

`scripts/hud.lua`:

```lua
function on_hud_open()
    input.add_callback("my_ui_pack.hello", function()
        if hud.is_open("my_ui_pack:hello") then
            hud.close("my_ui_pack:hello")
        else
            hud.show_overlay("my_ui_pack:hello", false)
        end
    end)
end
```

Verify: enable the pack for a world, join it, press H — the overlay toggles;
Close and H both dismiss it; the engine log shows no errors.

## DO / AVOID summary

- DO use `hud.show_overlay("pack_id:stem", ...)` with the pack id — it is
  REQUIRED there.
- DO use bare names for `menu.page` — pack ids are IMPOSSIBLE there.
- DO use global texture names or `atlas:tile` in `<image src>` — never
  `pack_id:...`.
- DO treat the `gui.load_document` document id as a free-form ALIAS you invent
  (not a path) — and keep it pack-prefixed and unique (see Way 3).
- DO register input callbacks inside `scripts/hud.lua`'s `on_hud_open()`.
- DO give custom global texture/font/asset names a pack-specific prefix.
- DO copy the engine's own `res/layouts/` as format examples — they are
  authoritative and always up to date.
- AVOID naming your pages `main`, `settings`, `content`, `pause`, `worlds` —
  you will shadow built-in menu pages (or get shadowed).
- AVOID subfolders directly under `layouts/` expecting auto-registration —
  only top-level `layouts/*.xml` auto-registers (`pages/` and `templates/`
  are special).
- AVOID preloading layouts in `preload.json` — there is no such key.
- AVOID the spelling `animations` for the folder or preload key — the engine
  constant is `animation` (singular). `[VERIFIED]`

## UI-specific pitfalls

1. **Putting `pack_id:` into `menu.page` or `<image src>`.** See the addressing
   table at the top; these schemes do not use pack ids. `[VERIFIED]`
2. **Omitting the pack id in `hud.*` calls.** `hud.show_overlay("hello", ...)`
   will not find `my_pack:hello` and throws. `[DOC]`
3. **Nesting layouts deeper than one level.** `layouts/sub/x.xml` is NOT
   auto-registered; only `layouts/*.xml` is. Pages must be under
   `layouts/pages/`. `[VERIFIED]`
4. **Expecting `layouts/pages/` files at startup.** Pages load on demand only.
   `[VERIFIED]`
5. **Binding mismatch across the 3-file chain.** `bindings.toml` action
   `hello` in section `[my_ui_pack]` must match callback name
   `"my_ui_pack.hello"` exactly. `[OBSERVED]`
6. **Editing a page while it is open and expecting hot reload.** UI changes
   need a world/menu reload; DO restart to verify.
7. **Treating the `gui.load_document` document id as a path.** It is an ALIAS
   you invent (Way 3): not read from disk, not derived from the file name, no
   match with the xml required. `[VERIFIED: user first-hand]`

## Upstream pointers (read before writing real UI)

The paths below are files of the ENGINE's own repository
(github.com/MihailRis/voxelcore, folder `doc/`) — NOT files of this guide.
Each topic exists twice: `doc/en/` (English) and `doc/ru/` (Russian). The
Russian documentation is usually MORE complete and more current; when the two
diverge, prefer `doc/ru/`. File names match on both sides:

| Topic | English (`doc/en/`) | Russian (`doc/ru/`) — prefer this |
| --- | --- | --- |
| FULL XML element and attribute reference | `doc/en/xml-ui-layouts.md` | `doc/ru/xml-ui-layouts.md` |
| Element properties/methods from Lua | `doc/en/scripting/ui.md` | `doc/ru/scripting/ui.md` |
| `gui.*` API (dialogs, documents, templates) | `doc/en/scripting/builtins/libgui.md` | `doc/ru/scripting/builtins/libgui.md` |
| `hud.*` API | `doc/en/scripting/builtins/libhud.md` | `doc/ru/scripting/builtins/libhud.md` |
| Input callbacks and key codes | `doc/en/scripting/builtins/libinput.md` | `doc/ru/scripting/builtins/libinput.md` |
| `on_hud_open` and other events | `doc/en/scripting/events.md` | `doc/ru/scripting/events.md` |
| `menu.page`, main-menu replacement | `doc/en/projects.md` | `doc/ru/projects.md` |

Also in the engine repository:

- `res/layouts/` — the engine's own layouts (pages, templates, console,
  inventory); authoritative and always up-to-date format examples.
- Real packs: `fork_of_kompot` (`layouts/gallery.xml*`, `scripts/hud.lua`),
  `wooden_meat_game` (`start.lua` document embedding, `wooden_meat_menu`
  pack).

Lua scripting beyond these UI entry points is out of scope of this guide (see
README scope table and docs/stubs/).

