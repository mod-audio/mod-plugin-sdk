# Plugin metadata (the `mod:` extension)

The `mod:` vocabulary is what lets a plain LV2 plugin tell a MOD device how to present it: brand
and label on the hardware, control ranges and per-device defaults, file types for its file
pickers, CV ports. The canonical spec is at https://mod-audio.github.io/mod-ns/mod/ and is not
restated here; this chapter is about *using* it well.

## Where the spec actually lives

`mod:`, `modgui:` and `modpedal:` are the three namespaces MOD actually publishes and documents.
If a property you need for one of those isn't on the spec page, that's a gap in the spec, not a
missing page. Other `moddevices.com/ns/` URIs do appear in shipping code without being published
anywhere — treat a namespace URI you find in source as evidence a term exists, not that it's
specified.

<details>
<summary>Verification</summary>

Namespace URIs resolve: `http://moddevices.com/ns/mod#` 301-redirects to
`https://moddevices.com/ns/mod/`, which serves the rendered extension spec (class and property
tables); the same holds for `modgui#` and `modpedal#`. Verified 2026-09-07.
`mod-ns` currently publishes exactly three namespaces. `http://moddevices.com/ns/ext/license#interface`
(emitted by `mod-ui/utils/utils_lilv.cpp:85`) returns 404, as does `ns/modperformance#` — both
exist in shipping code but aren't published. Read at `7764e1ff` (1.14 RC4).

</details>

## Verified vocabulary (`mod.lv2/mod.ttl`)

**Identity:** `mod:brand`, `mod:label`

**Port behaviour:** `mod:default`, `mod:minimum`, `mod:maximum`, `mod:rangeSteps`,
`mod:tapTempo`, `mod:tempoRelatedDynamicScalePoints`,
`mod:preferMomentaryOffByDefault`, `mod:preferMomentaryOnByDefault`,
`mod:preferColouredListByDefault`

**Per-device defaults:** `mod:default_duo`, `mod:default_duox`, `mod:default_dwarf`,
`mod:default_x64` — different default values per target, which is how one bundle can suit
very different CPUs.

**CV:** `mod:CVPort` is the port class that makes a CV port on MOD (plain `lv2:CVPort` is hidden
unless a developer-only environment flag is set). `mod:volts` is **a unit, not a port
property** — set it as `units:unit mod:volts` and mod-ui labels the value "v". Units only show
on **control** ports, input or output; a CV port never shows one. A CV port's range is
`mod:minimum`/`mod:maximum`, falling back to `lv2:`, default −5…+5, and the UI classifies a CV
output as unipolar or bipolar from the sign of that range.

**Files:** `mod:fileTypes` (see below). `mod:supportedExtensions` is parsed but **read by
nothing** in mod-ui — leave it out.

**MIDI:** `mod:rawMIDIClockAccess`

**Build metadata (internal — do not set these yourself):** `mod:builderVersion`,
`mod:releaseNumber`, `mod:buildEnvironment`, `mod:buildId`. MOD's publishing pipeline writes the
first three into the bundle. `releaseNumber`/`builderVersion` are used only inside version
comparisons and cache keys, never shown as text. `buildEnvironment` drives a badge on anything
that isn't `prod` (a locally installed bundle without it shows "LOCAL") and disables modgui
caching for it. `buildId` and `buildDate` are read by nothing.

<details>
<summary>Verification</summary>

- `mod:CVPort` hidden unless `MOD_UI_ALLOW_REGULAR_CV` is set: `utils_lilv.cpp:189,2648`
- `mod:volts` unit rendering (`%f v`): `utils_lilv.cpp:1938-1943`; units read only on control
  ports: `utils_lilv.cpp:3074`
- CV range fallback and default: `utils_lilv.cpp:2953-3006`; polarity from range sign:
  `mod/host.py:976-986`
- `mod:supportedExtensions` parsed into `/effect/get` JSON but unconsumed:
  `utils_lilv.cpp:1074-1104`, `modtools/utils.py:320`; no file in `mod/`, `html/js` or
  `html/include` reads it
- Build metadata: `releaseNumber`/`builderVersion` read as integers, `utils_lilv.cpp:2082-2091`;
  `buildEnvironment` read at `utils_lilv.cpp:2115-2137`, badge logic `html/js/effects.js:460-461`,
  cache bypass `modgui.js:155`

</details>

## `mod:fileTypes`

`mod:fileTypes` goes on an `lv2:Parameter` of `rdfs:range atom:Path`, and its value is a
comma-separated list of **file-type keys — not extensions**:

```turtle
<https://example.com/plugins/myplugin#model>
    a lv2:Parameter ;
    mod:fileTypes "nammodel" ;
    rdfs:label "Model" ;
    rdfs:range atom:Path .
```

The keys, the user-files folder each maps to, and the extensions mod-ui will list:

| Key | Folder under `/data/user-files` | Extensions |
|---|---|---|
| `audioloop` | Audio Loops | all libsndfile + ffmpeg audio |
| `audiorecording` | Audio Recordings | all libsndfile + ffmpeg audio |
| `audiosample` | Audio Samples | all libsndfile + ffmpeg audio |
| `audiotrack` | Audio Tracks | all libsndfile + ffmpeg audio |
| `cabsim` | Speaker Cabinets IRs | `.aif .aifc .aiff .flac .w64 .wav` |
| `ir` | Reverb IRs | `.aif .aifc .aiff .flac .w64 .wav` |
| `h2drumkit` | Hydrogen Drumkits | `.h2drumkit` |
| `midiclip` | MIDI Clips | `.mid .midi` |
| `midisong` | MIDI Songs | `.mid .midi` |
| `sf2` | SF2 Instruments | `.sf2 .sf3` |
| `sfz` | SFZ Instruments | `.sfz` |
| `aidadspmodel` | Aida DSP Models | `.aidax .json` |
| `nammodel` | NAM Models | `.nam` |
| `easyspinprog` | Easy Spin Programs | `.json` — 1.14 and on |

Write the list without spaces — a stray space after the comma produces a second key that
matches nothing.

**An unrecognised key is silently ignored** — the parameter still exists, the file browser just
has nowhere to look, which reads to a user as "the plugin is broken." Declare only keys from
this list, and only the ones you can actually load: listing extras does not extend what the
device can offer, it just widens what a user can hand you.

**Adding a new key is a three-repo change, not a plugin change.** It needs mod-ui to map the key
to a folder and extensions, the on-device File Manager to know about the folder, and the OS
image to pre-create the directory. A plugin author asks MOD for the key; they cannot ship one.

Reading such a parameter needs `patch:writable`, the worker and `state:mapPath` —
see [LV2-FEATURES.md](LV2-FEATURES.md).

<details>
<summary>Verification</summary>

Keys and mappings from `mod-ui/mod/webserver.py` (`FilesList._get_dir_and_extensions_for_filetype`).
`easyspinprog` added in mod-ui PR #159 (Audiofab's Easy Spin), in the 1.14 release line since
RC1 (`v1.14.0.3333`). Comma-splitting is plain `,` with no trim: `utils_lilv.cpp:1056-1062`.
Three-repo change confirmed adding `easyspinprog` (2026-09-15): (1) mod-ui's
`_get_dir_and_extensions_for_filetype`; (2) MOD's browsepy fork hard-codes the File Manager's
folder groups in `browsepy/templates/browse.html` and types files per folder in
`browsepy/file.py`; (3) the OS image pre-creates the directory in `browsepy.service`
(`ExecStartPre=mkdir -p`).

</details>

### Tone3000 browse entry (in MOD OS from 1.14 RC1)

The file-type key also decides whether a plugin gets a Tone3000 "Browse tones" entry in its
file dropdown. There's no plugin allow-list — the integration checks each `atom:Path`
parameter's `mod:fileTypes` and adds the entry when the list contains `nammodel`,
`aidadspmodel`, `cabsim` or `ir`; the same key picks which Tone3000 gear category the catalog is
filtered to. After a download, every plugin on the board whose parameter shares a file type with
the requesting one gets its dropdown refreshed. Declare the right key and the integration comes
for free; declare `audiosample` or a custom key and it stays off.

Downloaded files land under `<user-files>/<type dir>/<tone name>/`, named for the requesting
parameter's file type (or, failing that, guessed from the file extension), sanitised to ASCII
with collisions suffixed. A plugin sees them exactly as if the user had copied the files over —
nothing in the plugin needs to know Tone3000 exists.

<details>
<summary>Verification</summary>

Community (Starless) integration MOD adopted; gate is `html/js/modgui.js` `supportsT3K()`
checking the file-type list (`nammodel`/`aidadspmodel`/`cabsim`/`ir`), gear-category mapping in
`html/js/pedalboard.js` `T3KIntegration.startSelectFlow`. Shipped first in Starless 17
(2026-09-02); MOD's rebase onto master 2026-09-15, in every 1.14 image from RC1 (`v1.14.0.3333`):
gate at `html/js/modgui.js:27-28`, mapping at `html/js/pedalboard.js:3141-3150`, both at
`7764e1ff`. Upload path: `mod/webserver.py` `FilesUpload.process_file`, browser posts to
`/files/upload` with an `X-Upload-Config` header; extension-to-type fallback is `.nam` →
`nammodel`, `.wav` → `cabsim`, `.aidax` → `aidadspmodel`, anything else → `ir`. The image's
public Tone3000 key resolves `t3k-api-key` → `t3k_api_key.pub` next to the MOD API key →
`MOD_TONE3000_CLIENT_ID` (`mod/settings.py` `TONE3000_CLIENT_ID`, injected via
`/etc/mod-tone3000.env`). Plugins never see the key.

</details>

## `mod:label` and `mod:brand` lengths

mod-ui truncates the label to **24 bytes**, falling back to `doap:name` (itself truncated to
24) when no `mod:label` is set. `mod:brand` is cut to **16 bytes**, falling back to the author's
name. A port or parameter's `lv2:shortName` is also cut to 16. The cuts are byte-length based,
so a multibyte character sitting on the boundary gets split. Hardware displays are narrower than
all of these — treat the limits as hard ceilings, not design targets.

<details>
<summary>Verification</summary>

Label truncation: `utils/utils_lilv.cpp:2295-2296`, mini parser `:1595-1612`. Brand truncation:
`:2262-2263`, author-name fallback `:2274-2282`. `lv2:shortName`: `:2832-2833,1029-1030`.

</details>

## What the Cloud Builder share page shows

A persistent build on builder.mod.audio gets a share page (`/install/<id>`) that names the
plugin, its author and its category, read from the built bundle's TTL — nothing in a `.mk`
recipe sets these:

| Shown | Read from |
| --- | --- |
| name | `doap:name` |
| author (after "by") | `foaf:name` of the plugin's `doap:maintainer`, else of its `lv2:project`'s |
| brand (page title) | `mod:brand`, else the author |
| category | a `mod:` plugin class if there is one, else the `lv2:` one, reduced to its top-level category (`lv2:CompressorPlugin` → Dynamics) |

A plugin without `doap:name` gets a page that says "custom plugin build." Only `manifest.ttl`
and the files it points to with `rdfs:seeAlso` are read, and only at the top level of the
bundle.

<details>
<summary>Verification</summary>

`mod-cloud-builder` `webserver/bundleinfo.py`, live since 2026-09-28. Checked against the builds
stored on the server at the time: the large majority of buildroot builds read with no parse
error; the rest were failed builds with nothing in them to read.

</details>

## To write / verify

- [x] `mod:supportedExtensions`: parsed, consumed by nothing
- [x] `mod:volts`: a unit on control ports, in or out
- [x] Build-metadata properties: pipeline-written, read for version tuples and the environment badge only
