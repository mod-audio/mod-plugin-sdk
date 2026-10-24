# Building and installing plugins for MOD Desktop

MOD Desktop runs the same LV2 plugins as the devices, built for the computer instead of the
ARM boards. It ships with a fixed set and loads plugins you built yourself. This page is the
developer side of that: how to build a recipe for the three Desktop targets, and how to get the
result into a running MOD Desktop.

## Targets

The targets are `linux-x86_64`, `win64` and `macos-universal-10.15` (x86_64 and arm64 in one
binary). A plugin built for a Dwarf or Duo is an ARM binary and will not load on the Desktop,
and the reverse is also true.

## The recipes are the same

MOD Desktop builds its plugins from the `mod-plugin-builder` recipes through its own driver,
`utils/plugin-builder/plugin-builder.sh <target> <recipe>` in the
[mod-desktop](https://github.com/mod-audio/mod-desktop) repository. It wraps `plugin-builder.mk`
with the PawPaw environment of the target and adds `-D_MOD_DESKTOP` to the flags, so a recipe
that builds on the devices usually builds for the Desktop with no change. Of the 56 recipes in
the Desktop set, 55 build for Linux and Windows without any Desktop-specific change; the two
exceptions needed recipe-side fixes (a serial install for `mod-ams-lv2`, a download hook for
`fluidplug`), not source changes.

## Toolchains

Toolchains come from PawPaw, bootstrapped once per target with
`./src/PawPaw/bootstrap-mod.sh <target>` from a mod-desktop checkout. On a 12-core Linux host
this takes about 13 minutes for `linux-x86_64` and 18 minutes for `win64`, and about 3 GB on
disk. macOS needs a Mac with Xcode.

## Where the output lands

Built bundles land in the PawPaw cache, `~/PawPawBuilds/targets/<target>/lib/lv2/<bundle>.lv2`,
not in the mod-desktop tree. That directory is a normal LV2 bundle: `manifest.ttl`, the plugin's
`.ttl`, the binary (`.so`, `.dll` or `.dylib`) and, if the recipe has one, `modgui.ttl` with its
`modgui/` folder.

Validation is what the Desktop build itself runs: Carla's discovery and bridge
(`utils/plugin-builder/validate-plugins.sh`). It instantiates every plugin it finds; a bundle
that fails there will fail in the Desktop too.

## Installing into a running MOD Desktop

MOD Desktop looks for plugins in this order: the user folder `<Documents>/MOD Desktop/lv2`,
then the application's own bundle set, then, if "include system plugins" is enabled in the
Desktop settings, the OS LV2 folders (`~/.lv2`, `/usr/lib/lv2`,
`~/Library/Audio/Plug-Ins/LV2`, `%APPDATA%\LV2` and the common-files one).

To install, copy the `.lv2` folder into `<Documents>/MOD Desktop/lv2` and restart MOD Desktop.
The app creates that folder and scans it first, so a bundle there wins over a bundled one with
the same URI.

A plugin without a modgui still loads and works; it appears in the Constructor with the generic
box, as on a device. See [MODGUI.md](MODGUI.md).

A Desktop build identifies itself as `<major>.<minor>.<patch>.<build>` in the web UI's address
bar (`?v=`), the same way an OS build shows its release string. Quote it in bug reports.

<details>
<summary>Verification</summary>

Full Desktop plugin set built for `linux-x86_64` and `win64` on 2026-10-06/07 from the
mod-desktop `main` line; PawPaw bootstrap times measured on the same host. Search order:
`src/systray/utils.cpp` in mod-desktop, `getLV2Path`. The `/effect/install` endpoint also works on
the Desktop, but no UI control exposes it as of 0.0.14.

</details>

## Block size and sample rate your plugin sees

MOD Desktop always runs the engine at 48 kHz. The block size depends on how it is started:

- **Standalone app:** the "Buffer size" setting in the launcher window, 64, 128 or 256 (default 128), passed to
  `jackd -p`. Mirror of the units' choice; nothing above 256 is offered or tested.
- **Inside a DAW (VST3 / CLAP / AU / VST2 plugin):** always 128 frames at 48 kHz, whatever the DAW uses. The plugin
  starts its own jackd with the `mod-desktop` driver and `-r 48000 -p 128`, fed through shared memory from the DAW's
  audio thread; it cuts the DAW's buffer into 128-frame blocks, resamples to and from 48 kHz, and reports one block
  (scaled to the DAW rate) as latency. The launcher's buffer setting has no effect here.

So test at 64, 128 and 256 frames, 48 kHz. Verification: `mod-desktop` `src/plugin/DesktopPlugin.cpp:22-23`
(`kAudioBufferSize = 128`, `kAudioSampleRate = 48000`), `:479-490` (jackd args), `:158-169` (latency);
`src/systray/mod-desktop.hpp:713-722` and `mod-desktop.ui` (the three sizes), `:952-953` (`-p`).

## To write / verify

- [ ] A worked example from a bare checkout to an installed bundle on each of the three targets,
      with timings.
- [ ] Whether the Cloud Builder (builder.mod.audio) can target the Desktop. Today it builds for
      the device targets only and pushes to a USB-connected unit; a Desktop target would need
      the PawPaw toolchains server-side.
- [ ] Code signing on macOS and Windows for third-party bundles: the Desktop's own packages are
      unsigned today; what Gatekeeper does with an unsigned `.dylib` loaded by an unsigned app
      has not been tested.
- [x] Validation on macOS, partial (2026-10-08): the Carla discovery step still does not run on the
      Mac build, but every plugin of the 188-bundle set was added to and removed from a running
      mod-host on an M1 Mac mini (jackd dummy driver, the frozen 0.0.14.25 payload): 238 plugin
      URIs, 238 `add` ok, 238 `remove` ok, no instantiation failure. One caveat found the same
      day and not plugin-specific: jack2 on macOS could crash the host on plugin removal and on a
      failed activation (fixed in the Desktop build). Still open: the
      Carla discovery step itself on macOS.
