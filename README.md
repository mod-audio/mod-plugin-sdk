# MOD Plugin SDK

Everything you need to build an audio plugin for MOD-powered devices — from source code to a
working pedal on your unit.

This is the reference documentation for the MOD platform. If you are building plugins for MOD Duo,
MOD Duo X, MOD Dwarf or MOD Desktop, start here.

---

## LV2

Plugins for MOD are [LV2](https://lv2plug.in/) plugins. LV2 is an open, extensible standard — MOD
did not invent it and does not control it, and a plugin you write for MOD is a normal LV2 plugin that
runs anywhere LV2 runs.

**There are no licensing requirements for making LV2 plugins for MOD.** Build one, install it, give
it away, sell it elsewhere — none of that involves us.

MOD defines a small number of LV2 extensions that let a plugin describe itself more richly to MOD
devices: how it looks, how it is categorised, how its controls behave. They are published openly:

| Extension | Purpose |
|---|---|
| [`mod:`](https://mod-audio.github.io/mod-ns/mod/) | Plugin metadata — brand, label, ranges, per-device defaults, CV, file types |
| [`modgui:`](https://mod-audio.github.io/mod-ns/modgui/) | The plugin's web interface |
| [`modpedal:`](https://mod-audio.github.io/mod-ns/modpedal/) | Pedalboard format — written by MOD's UI, not by plugin developers |
| [`mod:license`](https://mod-audio.github.io/mod-license/) | Copy protection and trial mode, for commercial plugins |

All are ISC licensed. Using them is optional; a plugin with no MOD extensions at all still works.

### Frameworks

These frameworks export LV2 and are known to work:

- **[DPF](https://github.com/DISTRHO/DPF/)** — lightweight, C++, first-class MOD support. The path of
  least resistance
- **[JUCE](https://juce.com/)** — LV2 export since version 7
- **[Rust](https://docs.rs/lv2/latest/lv2/)**
- **Pure Data**, via [hvcc](https://github.com/Wasted-Audio/hvcc) — no C++ required
- **Max gen~**, via [max-gen-skeleton](https://github.com/mod-audio/max-gen-skeleton)

You can also write plain C against the LV2 headers directly — no framework required.

FAUST, Max gen~ and Pure Data sources, and any `.mk` recipe, can also be built with no local
toolchain at all on [builder.mod.audio](http://builder.mod.audio), which installs the result on
a USB-connected unit from the browser — see [BUILDING.md](BUILDING.md) § "Building without a
toolchain: the Cloud Builder" for which browser needs which URL.

---

## Documentation

**Getting a plugin built and running**

- [REQUIREMENTS.md](REQUIREMENTS.md) — how MOD expects a plugin to behave. Read before writing DSP
- [BUILDING.md](BUILDING.md) — building with `mod-plugin-builder`
- [RECIPES.md](RECIPES.md) — the `.mk` package format, which is how MOD builds your plugin
- [VALIDATING.md](VALIDATING.md) — checking your bundle before you ship it
- [INSTALLING.md](INSTALLING.md) — getting the bundle onto your MOD unit

**Making it a real plugin**

- [MODGUI.md](MODGUI.md) — the web interface. How to give your plugin a pedal face
- [METADATA.md](METADATA.md) — brand, label, control ranges, per-device defaults
- [CATEGORIES.md](CATEGORIES.md) — how plugins are classified in the store
- [LV2-FEATURES.md](LV2-FEATURES.md) — which LV2 features and extensions MOD supports
- [TIME.md](TIME.md) — tempo and transport
- [CAVEATS.md](CAVEATS.md) — target-specific gotchas worth knowing before you hit them

**Commercial plugins**

- [LICENSING.md](LICENSING.md) — copy protection and trial mode. *Coming later — placeholder for now*

---

## Under the hood

A MOD device runs an embedded Linux system with a real-time kernel. Audio is handled by
[JACK2](https://jackaudio.org/), with [mod-host](https://github.com/mod-audio/mod-host) on top loading
and connecting plugins. [mod-ui](https://github.com/mod-audio/mod-ui) serves the web interface that
users actually see, and renders your modgui.

**All MOD systems run at 48 kHz.**

The same stack runs on every MOD product, which is why one plugin bundle works across the range.

### Targets

| Target | Architecture | Product |
|---|---|---|
| `modduo` | ARMv7 (Cortex-A7) | MOD Duo |
| `modduox` | AArch64 (Cortex-A53) | MOD Duo X |
| `moddwarf` | AArch64 (**Cortex-A35**) | MOD Dwarf |
| `generic-x86_64` | x86-64 | Desktop — for building and validating |

A target name is exactly the name of its `plugins-dep/configs/<target>_defconfig` in
`mod-plugin-builder`; `./build` and `./validate` will list them if you get one wrong. Note that
the desktop target is `generic-x86_64`, not `x86_64`.

Most targets also have a `-new` variant — `moddwarf-new`, `modduox-new`, `modduo-new` — which is
the same device on a newer toolchain (GCC 9.4.0, glibc 2.27). **This is what MOD's own current
builds use**, so prefer it unless you have a reason not to. See [CAVEATS.md](CAVEATS.md) for the
compiler and libc of each.

**The Dwarf is the weakest target, not the strongest.** Its Cortex-A35 is in-order and
significantly slower per clock than the Duo X's A53. If you size your CPU budget on one device,
size it on the Dwarf.

`generic-x86_64` is not a shipping device target. It exists so you can build and test on your own
machine before cross-compiling, and it is the only target on which the automated runtime tests can
run. Use it constantly; it will save you hours.

### Darkglass Anagram

`mod-plugin-builder` also cross-compiles for the Darkglass Anagram — Darkglass's own hardware, on
the same LV2 / JACK2 / mod-host stack. That's why `darkglass-anagram` platform names show up
alongside MOD's own inside the toolchain (see [BUILDING.md](BUILDING.md)). Darkglass maintain
their own developer documentation for Anagram:
[Plugin-Dev-Setup](https://github.com/Darkglass-Electronics/Plugin-Dev-Setup). If you're building
specifically for Anagram, start there — it covers Anagram-specific mechanics this repo doesn't.

---

## Related repositories

| Repository | What it is |
|---|---|
| [mod-plugin-builder](https://github.com/mod-audio/mod-plugin-builder) | The cross-compilation toolchain. This documentation describes how to use it |
| [mod-lv2-extensions](https://github.com/mod-audio/mod-lv2-extensions) | The MOD LV2 extension definitions |
| [mod-plugin-cookbook](https://github.com/mod-audio/mod-plugin-cookbook) | Generate a plugin from a plain-language description |
| [mod-host](https://github.com/mod-audio/mod-host) | The LV2 host |
| [mod-ui](https://github.com/mod-audio/mod-ui) | The web interface that renders your modgui |
| [mod-sdk](https://github.com/mod-audio/mod-sdk) | Older interactive modgui editor. Unmaintained; its template and artwork library is still useful — see [MODGUI.md](MODGUI.md) |
| [Darkglass Plugin-Dev-Setup](https://github.com/Darkglass-Electronics/Plugin-Dev-Setup) | Darkglass's own developer docs for the Anagram, built on the same `mod-plugin-builder` toolchain this repository documents |

---

## Getting help

- **Forum:** [forum.mod.audio](https://forum.mod.audio) — the plugin developer community
- **Issues:** open one on this repository for documentation problems

---

## Licence

ISC. Use this documentation, quote it, adapt it, build on it.
