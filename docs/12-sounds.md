# 12. Sounds and Audio (Preload, Playback, Streams)

> **MANDATORY PREREQUISITES — DO NOT SKIP.** This file assumes you have read
> [01-engine-overview.md](01-engine-overview.md) AND
> [08-pitfalls.md](08-pitfalls.md). 01 defines the ID rules, reserved names
> and trust labels this file relies on; 08 lists the real failure modes
> behind every DO/AVOID note. Skipping either is the most common reason
> models generate broken packs from this guide. If you received only this
> file, fetch and read both before writing any code.

This document covers the audio subsystem as seen from a content pack: which
sound files a pack ships and how they are declared in `preload.json`, how
short preloaded sounds differ from streamed audio, how playback is controlled
through speaker ids and channels, and how block materials wire
step/place/break/hit sounds. Read the pitfalls section even if you skim the
API parts: the audio API fails SILENTLY (returns `0`, plays nothing, logs
nothing) in more situations than any other subsystem in this guide, so a
typo usually looks like "the code ran fine but there is no sound".

Out of scope here: runtime audio generation (`audio.PCMStream`, mic input) —
moved to 13-sound-creation.md. Authoring audio files in external tools and
OpenAL backend internals are out of scope entirely.

## Mental model: Sound vs Stream

Two different playable objects exist, and picking the wrong one is the
first pitfall:

| Term | Meaning |
| --- | --- |
| Sound | Fully loaded into RAM at pack load; one asset can play from many speakers at once; requires a `preload.json` entry; addressed by a short NAME without extension |
| Stream | Read from disk while playing; NOT preloaded, not stored in RAM; only one speaker per stream at a time; addressed by a FILE PATH with extension (or a shared alias) |
| Speaker | One-shot playback controller with a unique integer id; destroyed when the audio finishes or `audio.stop()` is called; survives pause |
| Channel | Volume/pause grouping: `master`, `ui`, `regular`, `ambient`, `music` |
| PCMStream | Lua-side source of raw PCM data; can be `share()`d as a stream alias or turned into a Sound via `create_sound` (13-sound-creation.md) |

`[DOC]` Decision rule:

- short, repeatable effects (steps, shots, clicks) that may overlap with
  themselves → preloaded Sound via `audio.play_sound` / `audio.play_sound_2d`
- long audio (music, ambience, radio) that must not sit in RAM → Stream via
  `audio.play_stream` / `audio.play_stream_2d`
- audio generated at runtime → build a `audio.PCMStream`, then either
  `share()` it as a stream or materialize a Sound with `create_sound`
  (13-sound-creation.md)

Formats: OGG (preferred) and WAV. WAV supports 8 and 16 bit only — 24 bit
is not supported by OpenAL. `[DOC]`

## Channels

| Channel | Meaning |
| --- | --- |
| master | Controls the loudness of ALL other channels; NEVER pass it as a playback channel |
| ui | Interface sounds; not paused together with the game |
| regular | Default for world sounds; paused together with the game |
| ambient | Same pause behavior as regular; intended for background/weather loops |
| music | Music playback, typically streamed |

`[DOC]` Three hard facts the table hides:

- An UNKNOWN channel name behaves like `master`: the play call returns `0`
  and nothing plays, with no error. `[VERIFIED: engine source, libaudio.cpp]`
- `regular` (the default) pauses with the game; if you start an ambience
  loop on the default channel it will freeze during pause — use `ambient`
  or `music` for anything that should keep its state sensibly managed.
- Screen switches RESET `regular` and `ambient` to empty: every speaker and
  stream on them is stopped and destroyed (menu ↔ world transitions,
  including the moment a world finishes opening — see Streams). `ui` and
  `music` survive screen switches. `[VERIFIED: engine source,
  Engine::setScreen]`

## Shipping sounds: `preload.json` and the variants pattern

Sound files live under the pack's `sounds/` folder. Declare them in the
pack's `preload.json`: sounds are NOT loaded by default; a sound that is
not preloaded cannot be played with `audio.play_sound` at all. `[DOC]` Two entry forms:

```json
{
    "sounds": [
        "events/explosion",
        {
            "name": "events/explosion",
            "keep-pcm": true
        }
    ]
}
```

- string form: `"sounds/<name>.ogg"` (or `.wav`) is loaded under the name
  `<name>` — e.g. entry `"events/explosion"` loads
  `sounds/events/explosion.ogg` and is played as `"events/explosion"`.
- object form: adds `"keep-pcm": true`, which retains PCM data in memory
  after decoding. Only needed if you access raw samples; otherwise omit it
  (decoded PCM is the main RAM cost of a Sound). `[DOC]`

### Variant pattern (multiple files under one name)

Preload ONE logical name, ship several files; the engine collects them
automatically. `[VERIFIED: engine source, assetload::sound]`

| File present | Role |
| --- | --- |
| `name.ogg` | base sound (tried first) |
| `name_0.ogg` | base sound if `name.ogg` is absent — numbering MAY start at 0 |
| `name_1.ogg` .. `name_N.ogg` | variants, loaded until the FIRST missing number |
| `name_2.ogg` with no `name_1.ogg` | NEVER loaded — a numbering gap silently ends the list |

At play time the engine picks uniformly at random between the base sound
and all variants — the base is one of the choices, not a guaranteed
default. `[VERIFIED: engine source, audio.cpp]`

Hard rules:

1. Extensions are tried in order `.ogg`, then `.wav`; only ONE extension is
   used for the base AND all its variants (decided by which file exists for
   the base). `[VERIFIED: engine source]`
2. If neither `name.ext` nor `name_0.ext` exists, pack loading FAILS with
   `could not to find sound: sounds/<name>` — a typo in `preload.json`
   breaks the whole pack, it does not degrade gracefully. `[VERIFIED]`
3. Variant files must be MONO if the sound is played positionally (see the
   stereo warning below).

## Playing preloaded sounds: `audio.play_sound` / `_2d`

```lua
-- positional: audible from a point in the world, attenuates with distance
audio.play_sound(
    name,     -- preloaded sound NAME: no pack prefix, no "sounds/", no extension, no _N suffix
    x, y, z,  -- world position of the source
    volume,   -- 0.0 .. 1.0
    pitch,    -- playback speed multiplier, positive (1.0 = normal)
    -- optional:
    channel,  -- "regular" (default) / "ambient" / "music" / "ui"
    loop      -- false (default)
) -> speakerid

-- non-positional: same minus position; use for UI and menu sounds
audio.play_sound_2d(name, volume, pitch, --[[ optional: ]] channel, loop)
    -> speakerid
```

`[DOC: audio.md]` Name format is the #1 source of silent failures:

| You pass | Result |
| --- | --- |
| `"shotgun"` | plays `pack sounds/shotgun.ogg` (assuming this pack preloaded it) |
| `"steps/stone"` | plays preloaded `sounds/steps/stone.ogg` — subfolders are part of the name |
| `"sounds/shotgun"` | SILENT no-op — the name never includes the folder root |
| `"shotgun.ogg"` | SILENT no-op — no extension in the name |
| `"other_pack:shotgun"` | SILENT no-op — `play_sound` does NOT understand pack prefixes |

The returned speaker id is `0` in every failure case: unknown name, unknown
channel, or the NoAudio backend (no audio device / `[audio] enabled=false`).
`0` is a valid-looking integer — DO NOT treat it as an error by crashing;
DO check it once when debugging "why is there no sound".
`[VERIFIED: engine source, libaudio.cpp]`

Real examples from `wooden_meat_game` (`content/my_pack/`,
simplified to the audio calls): `[OBSERVED]`

```lua
-- scripts/shotgun.lua — local "2D" shot with two pitch layers
function on_use(pid)
    audio.play_sound_2d("shotgun", 0.5, 0.6)
    audio.play_sound_2d("shotgun", 0.5, 0.4)
end

-- scripts/components/turret_targeting.lua — positional shot at a world point
audio.play_sound("shotgun", start_pos[1], start_pos[2], start_pos[3], 0.6, 0.6)
audio.play_sound("shotgun", start_pos[1], start_pos[2], start_pos[3], 0.6, 0.4)
```

Playing the same sound twice with slightly different `pitch` is a cheap
layering trick that reads as one thicker shot. `[OBSERVED]`

## Streams: `audio.play_stream` / `_2d`

```lua
audio.play_stream(
    filename, -- stream address, SEE THE TABLE BELOW — NOT a preload name
    x, y, z,  -- world position
    volume,   -- 0.0 .. 1.0
    pitch,    -- positive multiplier
    -- optional:
    channel,  -- "regular" (default) / "ambient" / "music" / "ui"
    loop      -- false (default)
) -> speakerid

audio.play_stream_2d(filename, volume, pitch, --[[ optional: ]] channel, loop)
    -> speakerid
```

`[DOC: audio.md]` Streams need NO `preload.json` entry — that is their
point: the file is read from disk while playing, so a long OGG costs no
RAM. `[DOC]` The stream address is resolved in THREE different ways, in
this order: `[VERIFIED: engine source, libaudio.cpp — only the 2nd/3rd are
in the upstream doc]`

| Form | Example | Resolved as |
| --- | --- | --- |
| Shared alias | `"test-stream"` | A name published by `stream:share(alias)` at runtime (see 13-sound-creation.md) |
| Entry-point path | `"my_pack:sounds/music.ogg"` | Literal file inside that pack — WITH extension |
| Bare path | `"sounds/music.ogg"` | `paths.find()` across ALL packs by content-source priority — WITH extension, collision-prone |

Hard rules for streams:

1. ONE stream can play through only ONE speaker at a time. To restart a
   music/ambience file, `audio.stop(old_id)` first — playing it again while
   the old speaker lives is a silent mistake. `[DOC]`
2. DO NOT preload long audio as Sounds: a 5-minute OGG as a Sound is fully
   decoded into RAM and stays there. Sounds are for short effects; streams
   are for anything measured in tens of seconds or minutes.
3. A stream speaker is destroyed when the file finishes (unless looping) or
   `audio.stop()` is called. Keep the returned id if you plan to touch
   volume/position later. `[DOC]`
4. On the NoAudio backend / headless mode the play calls return `0`; code
   that stores and then uses the id must tolerate that.
   `[VERIFIED: engine source]`

Real pattern from `content/sfx/` in `wooden_meat_game` — an ambience pack
that ships NO `preload.json` at all: `[OBSERVED]`

```lua
-- content/sfx/modules/ambient.lua (simplified)
local BROWN = "sounds/pixabay.com_brown_noise___themediaguy-....ogg"

function ambient.on_hud_open()
    -- channel "ambient", loop=true, start almost muted
    field_stream = audio.play_stream_2d(BROWN, 0.05, 1.0, "ambient", true)
    wind_stream = audio.play_stream_2d(PINK, 0.0, 1.0, "ambient", true)
end

function ambient.on_world_tick()
    -- ...gust state machine computes wind_volume...
    audio.set_volume(wind_stream, wind_volume)
end
```

Note the file naming: long unique filenames (source-prefixed) instead of a
collision-free subfolder — same idea, protecting the BARE-path resolution
from other packs. `[OBSERVED]`

Also note that the example starts ambience from `scripts/hud.lua`
(`on_hud_open`) — that is the only correct place for ambient loops, not a
style choice. World-open sequence, from the engine source:

1. `on_world_open` fires (world.lua context) while the level screen is
   still being constructed.
2. The level screen then finishes opening — the gap is the initial chunk
   loading, observable as roughly half a second `[OBSERVED]` — and the
   engine resets the `regular` AND `ambient` channels
   (`Engine::setScreen`): every speaker and stream started in step 1 is
   destroyed here, silently.
3. `on_hud_open` fires AFTER the reset (hud.lua context), so loops started
   there survive into gameplay.

`[VERIFIED: engine source]` DO start all ambient/music world loops from
`on_hud_open` and nowhere else. The same reset also wipes
`regular`/`ambient` when the engine returns to the menu; `ui`/`music`
channels survive screen switches.

## Mono vs stereo — the positional audio trap

> Sounds played through OpenAL in STEREO ignore the source position
> entirely — they always render as if non-positional. Any sound you pass a
> world position to (`audio.play_sound`, `audio.play_stream`) must be MONO.
> `[DOC: audio.md WARNING]`

This bites silently: a stereo file plays "fine" in `_2d` calls, then sounds
mysterious when played positionally. DO export positional sounds in mono;
keep stereo only for music/UI. DO test positional sounds from a distance,
not standing on the source.

## Controlling a speaker

All functions take the integer speaker id returned by a play call. Calling
anything on a dead/finished/never-existed id is a silent no-op — nothing
throws. `[DOC]`

| Function | Meaning |
| --- | --- |
| `audio.stop(id)` | Stop and destroy the speaker |
| `audio.pause(id)` / `audio.resume(id)` | Pause / unpause (speaker survives pause) |
| `audio.set_loop(id, state)` / `audio.is_loop(id)` | Looping flag / read it back (`false` if id is dead) |
| `audio.set_volume(id, v)` / `audio.get_volume(id)` | 0.0..1.0 (`0.0` if id is dead) |
| `audio.set_pitch(id, p)` / `audio.get_pitch(id)` | positive multiplier (`1.0` if id is dead) |
| `audio.set_time(id, t)` / `audio.get_time(id)` | Position in SECONDS (`0.0` if id is dead) |
| `audio.set_position(id, x, y, z)` / `audio.get_position(id)` | World position (`nil` if id is dead) |
| `audio.set_velocity(id, x, y, z)` / `audio.get_velocity(id)` | Source velocity for the Doppler effect (`nil` if dead) |
| `audio.get_duration(id)` | Length in seconds; `0` also means "unknown" (e.g. radio-style streams) |
| `audio.count_speakers()` / `audio.count_streams()` | Diagnostics: live speakers / playing streams |

`[DOC]` Usage notes:

- Speaker ids start at 1, are never reused during the engine's lifetime,
  and `0` means "nothing was created". Store the id (an integer), not any
  reference. `[DOC]`
- Start ambience streams MUTED (`volume 0.0`) and ramp `set_volume` from a
  tick handler — this is exactly how the `sfx` pack animates wind gusts.
  `[OBSERVED]`
- For a moving source, update `set_position` periodically (e.g. from an
  entity tick), otherwise the sound stays where it started.
- `get_duration` returning `0` is NOT an error indicator — for generated
  and radio-style streams the length is legitimately unknown. `[DOC]`

## Sounds attached to blocks (materials)

Block materials carry four sound names plus an absorption factor — this is
how blocks get step/place/break/hit audio without any Lua
(see 05-textures-and-assets.md for the materials system itself):

```json
{
    "steps-sound": "steps/ground",
    "place-sound": "blocks/ground_place",
    "break-sound": "blocks/ground_break",
    "hit-sound": "blocks/ground_hit",
    "sound-absorption": 0.7
}
```

`[VERIFIED: engine source + res/content/base/block_materials/ground.json]`

Facts that save debugging time:

1. These sounds are loaded AUTOMATICALLY by the engine for every material —
   they do NOT need `preload.json` entries (the base pack's own preload
   lists none of them). `[VERIFIED: engine source, AssetsLoader::addDefaults]`
2. A material's sound name resolves inside the pack's `sounds/` folder:
   `"steps/ground"` → `sounds/steps/ground.ogg` of the pack defining the
   material — the `sounds/` root is added by the engine, you never write it
   in the material. `[VERIFIED: engine source]`
3. If `hit-sound` is omitted it FALLS BACK to `steps-sound`.
   `[VERIFIED: engine source, ContentLoader::loadBlockMaterial]`
4. `sound-absorption` (0..1) scales how loud block sounds are for this
   material; stock values like `0.7` (ground) are a reference point.
   `[OBSERVED: stock pack values]`
5. The same `*-sound` keys are also accepted per-block in block JSON
   (`Block.cpp` parses them for every block), but the material is the
   primary mechanism the stock pack uses — prefer materials.
   `[VERIFIED: engine source]`
6. Sound names used in materials share the SAME flat global namespace as
   `play_sound` names — two packs defining a material sound named
   `"steps/stone"` collide (see Pitfalls #2).

## Pitfalls (audio-specific)

1. **Silent no-op instead of an error.** Unknown sound name, unknown
   channel, dead speaker id, NoAudio backend — every one of these just
   returns `0` / does nothing. When "sound doesn't play", FIRST verify the
   returned speaker id and the exact name string before suspecting files.
   `[VERIFIED]`
2. **Flat global namespace.** Preload names are registered exactly as
   written, with NO pack prefix, and later registrations overwrite earlier
   ones; bare stream paths resolve across packs by priority. Two packs
   shipping `"explosion"` (or `"sounds/explosion.ogg"` as a bare stream
   path) will silently share/win by load order. DO namespace: put sounds in
   a subfolder (`"events/my explosion"` is still one name — prefer
   `"myevents/explosion"` style or unique long filenames like the `sfx`
   pack does). `[VERIFIED: engine source]`
3. **Wrong name format in `play_sound`.** No `sounds/` root, no extension,
   no `_N` suffix, no pack prefix — any of those present means silence,
   not an error. `[VERIFIED]`
4. **Variant numbering gaps.** `name_1.ogg` + `name_3.ogg` without
   `name_2.ogg` loads only `name_1.ogg` and stops at the gap — silently.
   DO number variants densely from 0 or 1. `[VERIFIED: engine source]`
5. **Stereo positional sound.** Stereo files ignore world position. A
   positional effect must be mono; test from a distance. `[DOC]`
6. **`master` as a channel.** `audio.play_sound_2d("click", 1.0, 1.0,
   "master")` plays nothing. Channels are `ui`/`regular`/`ambient`/`music`
   only. `[VERIFIED: engine source]`
7. **24-bit WAV.** OpenAL path supports 8/16-bit WAV only; a 24-bit file
   fails to load (breaking pack load if preloaded). DO convert to 16-bit
   WAV or OGG. `[DOC]`
8. **Restarting a stream while it plays.** One stream file = one speaker;
   call `audio.stop(old_id)` before playing the same file again, or keep
   ONE long-lived looping speaker and control it via volume.
   `[DOC]`
9. **Missing preloaded file breaks pack load.** `"could not to find sound"`
   is a hard failure at load time. Keep preload entries and files in sync.
   `[VERIFIED: engine source]`
10. **Ambience on the wrong channel.** Loops started on the default
    `regular` channel pause with the game; background weather/music belongs
    on `ambient`/`music`. `[DOC]`
11. **Starting ambience in `on_world_open`.** The `regular` and `ambient`
    channels are reset when the level screen opens — moments after
    `on_world_open` fires (the gap is the initial chunk loading, ~0.5 s
    observed). Any loop started there is silently destroyed. DO start
    ambience in `on_hud_open` (hud.lua). `[VERIFIED: engine source,
    Engine::setScreen]`
12. **Typos in material sound names degrade silently.** A material's
    `steps-sound` that matches no file does not break pack load (unlike
    preload entries) — the step sound is simply missing in game. DO copy
    names from your `preload.json`/file tree, not from memory.
    `[VERIFIED: material sounds are engine-requested assets]`

## DO / AVOID summary

- DO use preloaded Sounds for short effects and Streams for long audio.
- DO name preloaded sounds exactly `name` / `subfolder/name` — no pack
  prefix, no `sounds/` root, no extension, no `_N`.
- DO namespace sound and stream names across packs (subfolder or unique
  long filenames); the namespace is flat and collision-prone.
- DO ship positional effects in MONO and test them from a distance.
- DO keep variant numbering dense (`name.ogg` or `name_0.ogg`, then
  `name_1..N` without gaps).
- DO start long loops muted and ramp volume from a tick handler.
- DO start ambient/music world loops from `on_hud_open` (hud.lua), never
  from `on_world_open` — the level-screen reset wipes `regular` and
  `ambient` moments after the world opens.
- AVOID `preload.json` entries for block-material sounds — the engine
  requests them automatically.
- AVOID passing `master` (or an invented channel) to any play call.
- AVOID preloading minutes-long audio as Sounds (RAM) — stream it.
- AVOID re-playing a stream file while its previous speaker is alive.
- AVOID trusting that a play call succeeded — check the speaker id while
  debugging, the API never raises on failure.

## Upstream pointers (read before writing real audio code)

All paths are files in the VoxelCore engine repository
(github.com/MihailRis/voxelcore), NOT in this guide. The Russian docs are
usually richer and fresher — prefer `doc/ru/` when they disagree.

| Topic | EN | RU |
| --- | --- | --- |
| Audio system, channels, speaker semantics, full `audio.*` API | `doc/en/audio.md` | `doc/ru/audio.md` |
| Preload entries, `keep-pcm`, sound variants note | `doc/en/assets-preload.md` | `doc/ru/assets-preload.md` |
| Block material fields (`steps-sound`, ...) | `doc/en/block-properties.md` | `doc/ru/block-properties.md` |
| Backend/engine config (`[audio] enabled=false`) | `doc/en/console.md` | `doc/ru/console.md` |

Where this guide goes beyond the upstream text (the three stream address
forms, the silent `0`-return contract, the flat namespace and overwrite
behavior, the variant-gap rule, the automatic loading of material sounds,
the `hit-sound` fallback, the screen-switch channel reset and the
world-open event order), the claims were verified against the engine
source at the time of writing — re-verify against your engine version if
behavior seems to differ.

Real usage to learn from: `wooden_meat_game` — `content/my_pack/`
(`preload.json` `"sounds": ["shotgun"]`, `scripts/shotgun.lua`,
`scripts/components/turret_targeting.lua`) for preloaded sounds, and
`content/sfx/` (`modules/ambient.lua` started from `scripts/hud.lua`, no
preload at all) for streams, volume automation and the `on_hud_open`
ambience pattern.
