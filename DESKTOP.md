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

## To write / verify

- [ ] A worked example from a bare checkout to an installed bundle on each of the three targets,
      with timings.
- [ ] Whether the Cloud Builder (builder.mod.audio) can target the Desktop. Today it builds for
      the device targets only and pushes to a USB-connected unit; a Desktop target would need
      the PawPaw toolchains server-side.
- [ ] Code signing on macOS and Windows for third-party bundles: the Desktop's own packages are
      unsigned today; what Gatekeeper does with an unsigned `.dylib` loaded by an unsigned app
      has not been tested.
- [ ] Validation numbers for macOS: the Carla discovery step has not been verified on that
      target yet.
