# 13. Creating Audio at Runtime (PCMStream, Mic Input)

> **MANDATORY PREREQUISITES — DO NOT SKIP.** This file assumes you have read
> [01-engine-overview.md](01-engine-overview.md),
> [08-pitfalls.md](08-pitfalls.md) AND
> [12-sounds.md](12-sounds.md). 01 defines the ID rules and trust labels
> this guide relies on; 08 lists the real failure modes behind every
> DO/AVOID note; 12 defines the Sound vs Stream mental model, channels,
> speaker semantics and the stream address resolution rules this file
> builds on. If you received only this file, fetch and read all three
> before writing any code.

This document covers GENERATING audio inside the engine instead of shipping
recorded files: building an `audio.PCMStream` in Lua, feeding raw PCM
samples into it, publishing it as a playable stream alias or materializing
it as a Sound, plus microphone input. Use it when the audio does not exist
as a file: synthesized tones and noise, DSP-processed signals, audio
received from the network or the microphone, anything computed at runtime.
If the audio already exists as an OGG/WAV file, ship the file and use
`preload.json` / `play_stream` (12-sounds.md) — that path is simpler,
cheaper and covers the overwhelming majority of packs.

## When to generate audio instead of shipping files

`[DOC]` Generate at runtime only when one of these holds:

- the audio is computed (sirens, tones, noise beds, procedural effects);
- the audio arrives from outside the pack (network stream, microphone);
- you must process an existing sound in ways files cannot express.

Everything else is a file + `preload.json` problem. Runtime generation
costs CPU, RAM and code, and its failure modes are silent like all audio
failures (12-sounds.md Pitfalls #1).

## `audio.PCMStream`: the object

| Member | Meaning |
| --- | --- |
| `audio.PCMStream(sample_rate, channels, bits)` | Constructor: sample rate in Hz; `channels`: 1 = mono, 2 = stereo; `bits`: 8 or 16 |
| `stream:feed(data)` | Append PCM bytes (a `Bytearray`) to the stream buffer |
| `stream:share(alias)` | Publish the stream under an alias address for `audio.play_stream` |
| `stream:create_sound(name)` | Materialize the PCM fed SO FAR into a Sound registered under `name` |

`[DOC: audio.md]` Verified semantics the upstream doc does not spell out:

1. `share(alias)` stores the stream in the asset registry that
   `audio.play_stream` consults FIRST, before pack paths — the shared-alias
   row of the stream address table in 12-sounds.md.
   `[VERIFIED: engine source, libaudio.cpp]`
2. On the headless backend `share()` is a silent no-op: the alias is NOT
   registered and playback would return `0` anyway. Do not expect
   generated audio to work in headless test runs.
   `[VERIFIED: engine source, lua_type_pcmstream.cpp]`
3. `create_sound(name)` registers a normal multi-speaker Sound under `name`
   in the SAME flat global namespace as preloaded sounds — playable with
   `audio.play_sound(name)` with NO `preload.json` entry, and subject to
   the same overwrite-on-collision rule (12-sounds.md Pitfalls #2).
   `[VERIFIED: engine source, lua_type_pcmstream.cpp]`
4. `create_sound(name)` captures only the samples fed BEFORE the call;
   feeding afterwards does not change the created Sound.
   `[VERIFIED: engine source]`
5. A second `share()` or `create_sound()` with the same string silently
   replaces the earlier registration — there is no error and no merge.
   `[VERIFIED: engine source]`

## Sample format rules

| Parameter | Rule |
| --- | --- |
| `sample_rate` | integer Hz (e.g. 44100); must match your sample-count math |
| `channels` | 1 = mono, 2 = stereo — positional playback REQUIRES mono (12-sounds.md) |
| `bits` | 8 or 16 only |
| 16-bit samples | write through `I16view` (or `U16view`) over the `Bytearray`, never as raw bytes |
| Buffer size | `bits / 8` bytes per sample per channel (`2` for 16-bit mono) |

`[DOC]` 24-bit (or wider) PCM has no place anywhere in the audio system:
24-bit WAV is unsupported by OpenAL (12-sounds.md), and `PCMStream` accepts
8/16 bits only.

## Worked example: a generated tone

`[DOC: audio.md]` Minimal working example from the upstream doc (formatting
normalized):

```lua
local max_amplitude = 32767
local sample_rate = 44100
local total_samples = sample_rate * 5
local bytes = Bytearray(total_samples * 2)
local samples = I16view(bytes)

for i = 1, total_samples do
    samples[i] = math.sin(i * math.pi * 2 / sample_rate * 400) * max_amplitude
end

local stream = audio.PCMStream(sample_rate, 1, 16)
stream:feed(bytes)
stream:share("test-stream")

audio.play_stream_2d("test-stream", 1.0, 1.0, "ui")
```

What it does, step by step:

- allocates a `Bytearray` of `total_samples * 2` bytes — 5 s of 16-bit mono;
- writes a 400 Hz sine through `I16view` (never raw bytes for 16-bit);
- feeds the buffer into a `PCMStream(44100, 1, 16)` and publishes it as the
  alias `"test-stream"`;
- plays it as a non-positional stream on the `ui` channel.

To materialize the same buffer as a Sound instead (multi-speaker, played by
name): `[OBSERVED: API shape per engine source]`

```lua
stream:create_sound("my_pack:tone")
audio.play_sound_2d("my_pack:tone", 1.0, 1.0)
```

## `share()` vs `create_sound()`

| | `share(alias)` | `create_sound(name)` |
| --- | --- | --- |
| Played via | `audio.play_stream` (alias resolved first) | `audio.play_sound` |
| Speaker model | ONE speaker per stream at a time | Many speakers at once |
| RAM | backend reads the buffer as it plays | fully buffered as a Sound |
| Live updates | you can keep `feed()`ing while it plays | frozen at call time |

`[VERIFIED: engine source]` Decision rule follows the 12-sounds.md mental
model: repeated one-shots → `create_sound`; long, generated or
still-growing audio → `share`. The upstream doc explicitly names network
audio streaming as the intended use of the PCMStream interface — feeding
while playing (radio-style) is the designed pattern, not a hack. `[DOC:
audio.md]`

## Naming: the namespaces you are writing into

- Shared aliases and created-sound names are runtime REGISTRY keys, not
  files; they never appear in `preload.json`.
- DO prefix BOTH with your pack id (`"my_pack:tone"`): aliases resolve
  before file paths and created sounds share the flat preload namespace —
  an unprefixed name can collide with another pack's file or sound and be
  silently overwritten (12-sounds.md Pitfalls #2). `[VERIFIED: engine
  source]`
- An alias shadows a bare file path with the same string at resolution
  time; resolution order is fixed (12-sounds.md Streams table).

## Microphone input (pointer only)

`audio.input.request_open(callback)` asks the user for recording permission
and passes an access token to the callback; `audio.input.fetch(token,
max_read_size)` then returns new PCM chunks you can `feed()` into a
PCMStream. This is niche for content packs and gated by user consent — read
`doc/ru/audio.md` ("Запись звука") before using. `[DOC]`

## Pitfalls (creation-specific)

1. **Unprefixed alias or sound name.** Both registries are flat global
   namespaces with silent overwrite on collision — DO prefix with the pack
   id. `[VERIFIED: engine source]`
2. **Headless `share()` no-op.** Generated aliases do not register on the
   headless backend; automated test runs will silently have no generated
   audio. `[VERIFIED: engine source, lua_type_pcmstream.cpp]`
3. **Feeding after `create_sound`.** The Sound is a snapshot of the buffer
   at call time; late feeds change nothing. Keep feeding only streams you
   published with `share()`. `[VERIFIED: engine source]`
4. **Raw byte writes to 16-bit samples.** Use `I16view`/`U16view` over the
   `Bytearray`; byte-wise writes produce garbage. `[DOC]`
5. **Stereo PCM for positional playback.** Stereo ignores source position
   (12-sounds.md) — generated positional audio must be built with
   `channels = 1`. `[DOC]`
6. **`get_duration` returns `0` for shared streams.** The length of
   generated audio is legitimately unknown to the engine; not an error.
   `[DOC]`

## DO / AVOID summary

- DO ship OGG/WAV files whenever the audio exists as a file; generate only
  what cannot be shipped.
- DO prefix shared aliases and created-sound names with the pack id.
- DO write 16-bit samples via `I16view`/`U16view` over the `Bytearray`.
- DO keep one long-lived PCMStream and keep feeding it for radio-style
  audio — that is the intended pattern, not a hack.
- AVOID `share()`ing and expecting it to work in headless test runs.
- AVOID feeding after `create_sound()` and expecting the Sound to change.
- AVOID unprefixed names (flat namespaces, silent overwrites).
- AVOID stereo PCM for positional playback.

## Upstream pointers (read before writing real audio code)

All paths are files in the VoxelCore engine repository
(github.com/MihailRis/voxelcore), NOT in this guide. The Russian docs are
usually richer and fresher — prefer `doc/ru/` when they disagree.

| Topic | EN | RU |
| --- | --- | --- |
| `audio.PCMStream`, mic input, generation example | `doc/en/audio.md` | `doc/ru/audio.md` |

Where this guide goes beyond the upstream text (the registry semantics of
`share`/`create_sound`, the headless `share()` no-op, the snapshot-at-call
rule, the overwrite-on-collision behavior), the claims were verified
against the engine source (`lua_type_pcmstream.cpp`, `libaudio.cpp`,
`Assets.hpp`) at the time of writing — re-verify against your engine
version if behavior seems to differ.
