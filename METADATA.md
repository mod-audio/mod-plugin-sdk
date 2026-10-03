# Plugin metadata (the `mod:` extension)

The `mod:` vocabulary is what lets a plain LV2 plugin tell a MOD device how to present it: brand
and label on the hardware, control ranges and per-device defaults, file types for its file
pickers, CV ports. The canonical spec is at https://mod-audio.github.io/mod-ns/mod/ and is not
restated here; this chapter is about *using* it well, and about what mod-ui actually does with
each term, read from `mod-ui/utils/utils_lilv.cpp` and friends at `7764e1ff` (1.14 RC4).

## Verified: where the spec actually lives

The `mod:` vocabulary is published from the `mod-audio/mod-ns` repo ("Combined repo for
publication of MOD namespaces specifications"). Namespace URIs resolve: `http://moddevices.com/ns/mod#`
301-redirects to `https://moddevices.com/ns/mod/`, which serves the rendered extension spec
(class and property tables); the same holds for `modgui#` and `modpedal#`. Verified 2026-09-07.
If a property you need isn't documented there, that's a gap in `mod-ns`, not a missing page.
`mod-ns` currently publishes exactly three namespaces — `mod`, `modgui`, `modpedal`. Other
`moddevices.com/ns/` URIs appear in shipping code without being published there: verified
2026-09-07, `http://moddevices.com/ns/ext/license#interface` (emitted by
`mod-ui/utils/utils_lilv.cpp:85`) returns 404, as does `ns/modperformance#`. Treat a namespace
URI in code as evidence that a term exists, not that it is specified.

## Verified vocabulary (`mod.lv2/mod.ttl`)

**Identity:** `mod:brand`, `mod:label`

**Port behaviour:** `mod:default`, `mod:minimum`, `mod:maximum`, `mod:rangeSteps`,
`mod:tapTempo`, `mod:tempoRelatedDynamicScalePoints`,
`mod:preferMomentaryOffByDefault`, `mod:preferMomentaryOnByDefault`,
`mod:preferColouredListByDefault`

**Per-device defaults:** `mod:default_duo`, `mod:default_duox`, `mod:default_dwarf`,
`mod:default_x64` — different default values per target, which is how one bundle can suit
very different CPUs

**CV:** `mod:CVPort`, `mod:volts`. `mod:CVPort` is the port class that makes a CV port on MOD (plain
`lv2:CVPort` is hidden by mod-ui unless `MOD_UI_ALLOW_REGULAR_CV` is set, `utils_lilv.cpp:189,2648`).
`mod:volts` is **a unit, not a port property**: set it as `units:unit mod:volts` and mod-ui labels
the value "v" with a `%f v` render (`utils_lilv.cpp:1938-1943`). Units are only read on **control**
ports, input or output (`utils_lilv.cpp:3074`); a CV port never shows a unit. A CV port's range is
`mod:minimum`/`mod:maximum`, falling back to `lv2:`, default −5…+5 (`utils_lilv.cpp:2953-3006`),
and the UI uses the sign of that range to classify a CV output as unipolar or bipolar
(`mod/host.py:976-986`).

**Files:** `mod:fileTypes` (see below). `mod:supportedExtensions` is parsed into the `/effect/get`
JSON (`utils_lilv.cpp:1074-1104`, `modtools/utils.py:320`) and **read by nothing**: no file in
`mod/`, `html/js` or `html/include` consumes it. Leave it out.

**MIDI:** `mod:rawMIDIClockAccess`

**Build metadata (internal — do not set these yourself):** `mod:builderVersion`,
`mod:releaseNumber`, `mod:buildEnvironment`, `mod:buildId`. MOD's publishing pipeline writes the
first three into the bundle; mod-ui reads `releaseNumber` and `builderVersion` as integers
(`utils_lilv.cpp:2082-2091`) and uses them only inside version tuples (update comparison, cache
keys, the pedalboard TTL), never as text, and reads `buildEnvironment` (`prod`, `dev`, `labs`,
`:2115-2137`) to show a badge on anything that is not `prod` (`html/js/effects.js:460-461`:
a locally installed bundle without it shows "LOCAL") and to bypass modgui caching for it
(`modgui.js:155`). `mod:buildId` and `mod:buildDate` are read by nothing.

## Verified: `mod:fileTypes`

`mod:fileTypes` goes on an `lv2:Parameter` of `rdfs:range atom:Path`, and its value is a
comma-separated list of **file-type keys — not extensions**:

```turtle
<https://example.com/plugins/myplugin#model>
    a lv2:Parameter ;
    mod:fileTypes "nammodel" ;
    rdfs:label "Model" ;
    rdfs:range atom:Path .
```

The keys, the user-files folder each maps to, and the extensions mod-ui will list, from
`mod-ui/mod/webserver.py` (`FilesList._get_dir_and_extensions_for_filetype`, verified 2026-08-24):

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
| `easyspinprog` | Easy Spin Programs | `.json` — **1.14 on** (mod-ui PR #159, Audiofab's Easy Spin; in the 1.14 release line since RC1, `v1.14.0.3333`) |

Write the list without spaces: mod-ui splits on `,` only (`utils_lilv.cpp:1056-1062`), so
`"nammodel, cabsim"` yields a second key `" cabsim"` that matches nothing.

**An unrecognised key is silently ignored** — the parameter still exists, the file browser just has
nowhere to look, which reads to a user as "the plugin is broken". Declare only keys from this list,
and only the ones you can actually load: listing extras (as some plugins do) does not extend what
the device can offer, it just widens what a user can hand you.

**Adding a new key is a three-repo change, not a plugin change** (learned adding
`easyspinprog`, 2026-09-15): (1) mod-ui `FilesList._get_dir_and_extensions_for_filetype`
maps key → folder + extensions; (2) MOD's browsepy fork hard-codes the File Manager's folder
groups in `browsepy/templates/browse.html` and types files per folder in `browsepy/file.py` —
without this the folder exists but the File Manager never shows it; (3) the OS image
pre-creates the directory in `browsepy.service` (`ExecStartPre=mkdir -p`). A plugin author asks
MOD for the key; they cannot ship one.

Reading such a parameter needs `patch:writable`, the worker and `state:mapPath` —
see [LV2-FEATURES.md](LV2-FEATURES.md).

### Tone3000 browse entry (in MOD OS from 1.14 RC1)

The file-type key is also what decides whether a plugin gets a Tone3000 "Browse tones" entry
in its file dropdown. There is no plugin allow-list: the community (Starless) integration
that MOD decided to adopt checks each `atom:Path` parameter's `mod:fileTypes` and adds the
entry when the list contains `nammodel`, `aidadspmodel`, `cabsim` or `ir`
(`sejerpz/mod-ui` `feature/t3k`, `html/js/modgui.js` `supportsT3K()`); the same key picks
the Tone3000 gear category the catalog is filtered to (`nammodel`/`aidadspmodel` → amp,
amp-cab, pedal, outboard; `cabsim` → cab; `ir` → space; `html/js/pedalboard.js`
`T3KIntegration.startSelectFlow`). After a download, every plugin on the board whose
parameter shares a file type with the requesting one gets its dropdown refreshed. So: declare
the right key and the integration comes for free; declare `audiosample` or a custom key and
it stays off. Shipped first in Starless 17 (2026-09-02); MOD's copy, rebased onto master
2026-09-15, is in every 1.14 image from RC1 (`v1.14.0.3333`): the gate is
`html/js/modgui.js:27-28` at `7764e1ff`, the gear mapping `html/js/pedalboard.js:3141-3150`.

Where the downloaded files land (`mod/webserver.py` `FilesUpload.process_file` on that
branch): the browser uploads each model to `/files/upload` with an `X-Upload-Config` header;
the target folder comes from the requesting parameter's file type
(`FilesList._get_dir_and_extensions_for_filetype`), or, when the client sends no type, from
the extension — `.nam` → `nammodel`, `.wav` → `cabsim`, `.aidax` → `aidadspmodel`, anything
else → `ir`. Files go under `<user-files>/<type dir>/<tone name>/`, names sanitised to ASCII,
collisions suffixed ` 01`, ` 02`. A plugin sees them exactly as if the user had copied them
over: nothing in the plugin has to know about Tone3000.

The image's public Tone3000 key resolves prefs `t3k-api-key` → `t3k_api_key.pub` next to the
MOD API key → `MOD_TONE3000_CLIENT_ID` from the environment (`mod/settings.py`
`TONE3000_CLIENT_ID`, injected by the MBS `mod-ui.mk` via `/etc/mod-tone3000.env`). Plugins
never see the key.

## Verified: `mod:label` and `mod:brand` lengths

mod-ui truncates the label to **24 bytes** (`utils/utils_lilv.cpp:2295-2296`, mini parser
`:1595-1612`), falling back to `doap:name`, itself truncated to 24, when no `mod:label` is set.
`mod:brand` is cut to **16 bytes** (`:2262-2263`, fallback the author's name, `:2274-2282`), and a
port's or parameter's `lv2:shortName` to 16 (`:2832-2833,1029-1030`). The cuts are `strlen`-based,
so a multibyte character on the boundary is split. Hardware displays are narrower than all of
these; treat them as hard ceilings, not design targets.

## Verified: what the Cloud Builder share page shows

A persistent build on builder.mod.audio gets a share page (`/install/<id>`) that names the
plugin, its author and its category. For a build from your own `.mk` (the `/buildroot` page)
those come from the built bundle's `.ttl`, nothing in the `.mk` sets them
(`mod-cloud-builder` `webserver/bundleinfo.py`, live since 2026-09-28):

| Shown | Read from |
| --- | --- |
| name | `doap:name` |
| author (after "by") | `foaf:name` of the plugin's `doap:maintainer`, else of its `lv2:project`'s |
| brand (page title) | `mod:brand`, else the author |
| category | a `mod:` plugin class if there is one, else the `lv2:` one, reduced to its top-level category (`lv2:CompressorPlugin` → Dynamics) |

A plugin without `doap:name` gets a page that says "custom plugin build". Only `manifest.ttl`
and the files it points to with `rdfs:seeAlso` are read, and only at the top level of the
bundle. Proven on the 233 builds stored on the server: 190 of the 204 buildroot builds
read with no parse error, the other 14 are failed builds with nothing in them.

## To write / verify

- [x] `mod:supportedExtensions`: parsed, consumed by nothing (2026-10-03, from source)
- [x] `mod:volts`: a unit on control ports, in or out (2026-10-03, from source)
- [x] Build-metadata properties: pipeline-written, read for version tuples and the environment badge
      only (2026-10-03, from source)
