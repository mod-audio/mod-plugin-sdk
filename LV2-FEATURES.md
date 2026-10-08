# Supported LV2 features

A plain list of what the host provides and what it ignores, so you can check in ten seconds
whether the feature you want exists. The first table is read from mod-host source; the sections
after it are what real plugins exercised on devices.

## What mod-host passes to `instantiate()`

`GetFeatures()` builds this list for every plugin:

| Feature | Data | Notes |
|---|---|---|
| `urid:map`, `urid:unmap` | yes | also the legacy `uri-map` |
| `options:options` | yes | sample rate, min/max/nominal block length, sequence size, thread policy and priority; see [REQUIREMENTS.md](REQUIREMENTS.md) § "block size" |
| `bufsz:boundedBlockLength`, `bufsz:fixedBlockLength`, `bufsz:powerOf2BlockLength` | NULL | passed as promises, never checked |
| `log:log` | yes | messages go to the device log |
| `state:makePath` | yes | paths under the pedalboard directory during save/load, under a scratch directory at runtime |
| `state:freePath` | yes | |
| `worker:schedule` | yes, **only if you export `worker:interface`** | |
| `http://kx.studio/ns/lv2ext/control-input-port-change-request` | yes | a plugin may ask the host to change one of its own control inputs; no clamping on this path |
| `http://moddevices.com/ns/hmi#WidgetControl` | yes, on MOD devices | the hardware-display extension; see "mod-hmi" below |
| `http://moddevices.com/ns/ext/license#feature` | yes | libmodla's hook; see [LICENSING.md](LICENSING.md) |

Not passed: `state:mapPath` (lilv supplies a map-path feature of its own inside
`lilv_state_new_from_instance`/`lilv_state_restore`, which is why `save()`/`restore()` still see
one, as the hardware table below confirms), `instance-access` and `data-access` (external UIs
only), `ui:*`. `lv2:hardRTCapable` is not required; mod-host does not check it.

**Extension data it uses:** `worker:interface` (`work_response()` and `end_run()` run on the
audio thread after `run()`); `options:interface` (`set()` only, on a buffer-size change);
`state:interface` (with `state:threadSafeRestore` honoured, and `state:loadDefaultState` applied
at instantiate); `license#interface`; `hmi#PluginNotification` on MOD devices. `ui:idleInterface`
exists for external UIs only; there is no plugin-side idle callback.

**Port types.** Audio ports become JACK audio ports. Atom ports that `atom:supports
midi:MidiEvent` become JACK MIDI ports (MIDI in, MIDI out, with the host inserting all-notes-off
on bypass); atom ports that `atom:supports time:Position` receive the transport
([TIME.md](TIME.md)); atom ports carrying `patch:Message` are how parameters work (below). CV
ports (`mod:CVPort`; plain `lv2:CVPort` is accepted by mod-host but hidden by mod-ui unless a
developer-only environment flag is set) get JACK ports of their own (JACK has no CV type, they
carry one float per frame like audio); their range is `mod:`/`lv2:minimum`/`maximum`, defaulting
to −5…+5, and the UI derives a CV output's mode from it: unipolar positive, unipolar negative or
bipolar. MIDI and CV ports have been read from source for this table, not yet exercised by a
plugin built for this documentation.

<!-- GAP: MIDI and CV are read from mod-host source, not yet exercised by a plugin built for this
     documentation -->

<details>
<summary>Verification</summary>

Read from `mod-host/src/effects.c` at `f14a230` (master, what ships), 2026-10-03.
`GetFeatures()`: `:2936-3003`. `urid:map`/`unmap`/`uri-map`: `:776-777,775`. `options:options`
data: `:4233-4287`. `bufsz:*` promises unchecked: `:763-767`. `log:log`: `:772`. `state:makePath`:
`:2947-2950,3690-3700`. `state:freePath`: `:774`. `worker:schedule` gating: `:2966-2973`.
Control-input change-request feature: `:2960,3666-3688`. `hmi#WidgetControl`: `:769`,
`src/lv2/lv2-hmi.h`. `license#feature`: `:771`. `state:mapPath` commented out: `:3150-3156`.
Extension data lookups: `:4669-4725`; `work_response()`/`end_run()` timing: `:2083-2090`;
`options:interface` `set()`: `:1043`; `state:interface` honouring: `:4693-4716`.
MIDI port handling and bypass all-notes-off: `:1833-1861,2131-2151`. CV port class gate:
`utils_lilv.cpp:189,2648`. CV JACK port creation: `effects.c:391-396`. CV range default:
`utils_lilv.cpp:2953-3006`. CV polarity derivation: `mod/host.py:976-986`.

</details>

## Verified on hardware

These were exercised end to end by a plugin running on a MOD Dwarf (2026-08-24). Each is
confirmed working, not inferred from source:

| Feature | Notes |
|---|---|
| `urid:map` | Required feature; always present |
| `worker:schedule` + `worker:interface` | Required feature. `work()` on a real worker thread, `work_response()` delivered on the audio thread |
| `state:interface` with `state:mapPath` / `state:freePath` | Paths are saved abstract and restored absolute; pedalboard save/recall works |
| `state:threadSafeRestore` | `restore()` may schedule worker jobs, which is the documented way to load a file without blocking audio |
| `options:interface` + `bufsz:maxBlockLength` | Delivered at instantiation; this is how you learn your maximum block size. mod-host also sends `bufsz:nominalBlockLength` and `bufsz:minBlockLength`; on a device max = nominal = the JACK period. Check `option.type == atom:Int` before reading `value` — and check it the right way round (see CAVEATS.md) |
| `log:log` | Messages appear in the device log |
| Atom sequence ports carrying `patch:Message` | Both directions — `patch:Set` in, `patch:Set` out on a notify port |

**Patch parameters are supported.** This was the open question and the answer is yes: an
`lv2:Parameter` with `rdfs:range atom:Path`, declared `patch:writable`, is set by mod-ui through
a `patch:Set` on the plugin's control port, and mod-ui renders a file browser for it when the
parameter carries `mod:fileTypes`. Report the current value back by forging a `patch:Set` onto
a notify port so the UI can display which file is loaded, and answer `patch:Get` the same way.
Note this is a MOD statement only — Darkglass Anagram does not support patch parameters except
for file-loading via `atom:Path`, so a plugin that depends on them for anything else is not
portable across the whole family. Darkglass's own `juce-anagram-lv2` wrapper exists specifically
to give JUCE plugins old-style Control Ports instead of JUCE's default Patch Parameters, for
exactly this reason — see
[Plugin-Dev-Setup](https://github.com/Darkglass-Electronics/Plugin-Dev-Setup).

One practical constraint that follows: **loading the file must happen on the worker thread.** The
audio thread receives the `patch:Set`, hands the path to `work()`, and swaps in the result when
`work_response()` arrives. See [REQUIREMENTS.md](REQUIREMENTS.md).

<details>
<summary>Verification</summary>

`mod-ui/html/js/modgui.js:190` renders the file browser when `mod:fileTypes` is present on a
`patch:writable` `atom:Path` parameter.

</details>

## Verified: factory presets

Exercised by a family of control-port-only synths on a Dwarf, a Duo and a Duo X. Declare each
preset in `manifest.ttl` and put its values in a file it points to:

```turtle
# manifest.ttl
<https://example.org/synth#preset-dark>
    a pset:Preset ; lv2:appliesTo <https://example.org/synth> ; rdfs:label "Dark" ; rdfs:seeAlso <presets.ttl> .

# presets.ttl
<https://example.org/synth#preset-dark>
    a pset:Preset ; lv2:appliesTo <https://example.org/synth> ; rdfs:label "Dark" ;
    lv2:port [ lv2:symbol "cutoff" ; pset:value 1.2 ] ,
             [ lv2:symbol "resonance" ; pset:value 0.4 ] .
```

mod-ui lists them (`/effect/get?uri=` → `presets`, sorted by label, not in file order) and
mod-host applies one with `preset_load <instance> <preset-uri>`; `param_get` afterwards returns
the preset's values. Give every control port a value in every preset: a port a preset does not
mention keeps whatever it had, so the same preset sounds different depending on what was loaded
before it.

## mod-hmi: writing to the hardware display

`mod-hmi` is developer-facing. mod-host passes the `hmi#WidgetControl` feature to every plugin
on a MOD device and calls the plugin's `hmi#PluginNotification` extension data when one of its
ports is **addressed** to a hardware actuator or unaddressed again. Through the feature a plugin
can then drive what the device shows for that control: `set_label`, `set_value`, `set_unit`,
`set_indicator`, `set_led_with_blink`, `set_led_with_brightness` and `popup_message`. The
extension is defined in
[`mod-lv2-extensions/mod-hmi.lv2`](https://github.com/mod-audio/mod-lv2-extensions) and
specified at [moddevices.com/ns/hmi](http://moddevices.com/ns/hmi/). No public MOD plugin uses
it yet, so there is no worked example here. The feature is absent on MOD Desktop, so declare it
as `lv2:optionalFeature` and always check for it.

The API grows by appending: new functions go at the end of `LV2_HMI_WidgetControl`, each with
its own `LV2_HMI_WIDGETCONTROL_SIZE_*` constant that the plugin compares against the struct's
`size` before calling (this is how `popup_message` was added), and per-addressing capability bits
in `LV2_HMI_AddressingInfo::caps` say which calls do anything for a given hardware control. A
plugin built against a newer header therefore runs on an older host as long as it checks before
each call.

<!-- GAP: mod-hmi has no worked example here -->

<details>
<summary>Verification</summary>

Feature pass under `__MOD_DEVICES__`: `effects.c:769`. Addressed/unaddressed notification call:
`effects.c:4719-4724,7541,7590`. Method list: `mod-host/src/lv2/lv2-hmi.h:223-272`. Size
constants: `lv2-hmi.h:193-198`; capability bits: `lv2-hmi.h:52-58`. The spec URL returned 404
until the page was published on 2026-10-08. The stock Dwarf firmware also has a raw `glcd_draw`
command (display id, x, y, hex bitmap, one byte per 8 vertical pixels;
`mod-dwarf-controller/app/src/protocol.c:749`) that nothing in mod-ui sends and that the firmware
does not cache; a plugin-driven display would need the firmware to own the bitmap.

</details>

## Verified: port groups (mod-ui 1.14 on)

In mod-ui from `v1.14.0.3333`; 1.13.5 ignores the triples, so a grouped plugin is harmless
there. **Confirmed on a Dwarf**: three plugins from one family, with the TTL below;
`curl "http://<device>/effect/get?uri=<uri>"` returns `portGroups` in `lv2:index` order and each
control input's `group` set to the group URI. (How it looks on screen is not described here
yet.)

What mod-ui reads: on each control port, `pg:group <group-uri>`; on the group node,
`lv2:symbol`, `lv2:name` and **`lv2:index`**, which mod-ui uses as the display order (standard
LV2 does not give groups an index). Groups are sorted by index, then name, then symbol.

```turtle
@prefix pg: <http://lv2plug.in/ns/ext/port-groups#> .

    lv2:port [ ... lv2:symbol "cutoff" ; pg:group <https://example.org/synth#group-vcf> ; ] .

<https://example.org/synth#group-vcf>
    a pg:InputGroup ;
    lv2:symbol "vcf" ;
    lv2:name "VCF" ;
    lv2:index 1 .
```

In the UI, control inputs are **re-sorted by group index, then by port index**, and each run of
one group is drawn together. So grouping never requires renumbering ports: add it to a shipped
plugin without touching a symbol or an index. Ungrouped ports sort by port index among
themselves. The CSS colours 32 groups. `sord_validate` accepts the triples above.

<details>
<summary>Verification</summary>

Read from `mod-ui/utils/utils_lilv.cpp:2864-2899`, group sort at `:3142`. UI re-sort and
grouping render: `html/js/host.js:374-425`.

</details>

## To write / verify

- [x] Full supported list read from `mod-host` (2026-10-03)
- [x] `mod-hmi` is developer-facing (2026-10-03); a worked example is still missing
- [x] Presets — verified 2026-09-17, see "Verified: factory presets"
- [ ] MIDI, CV — read from source, not yet exercised on a device by a plugin built for this page
- [x] Port groups — format read from source and confirmed on a device through `/effect/get` (above)
