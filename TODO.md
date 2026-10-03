# mod-plugin-sdk — documentation checklist

*Generated 2026-08-24 from a full pass over every chapter plus the reference repos. Not for
publication — working checklist for whoever picks this up. Delete or move to Issues once it's
worked through.*

Legend: 🔴 blocks a claim already made elsewhere in the docs · 🟡 real gap, no blocker · ⚪ nice
to have / needs an external answer (marked who from)

**Finding these programmatically:** every chapter also carries `<!-- GAP: ... -->` HTML comments
at the exact spot in the text where a claim is unverified or content is missing. They're invisible
on GitHub's rendered page — `grep -rn "GAP:" *.md` from the repo root lists all of them with file
and line. This file is the human-readable summary of the same set of gaps; the inline comments are
so an agent working hands-on on a specific chapter (building and testing a real plugin, the way
`mod-nam-loader` produced most of RECIPES/VALIDATING/CAVEATS/LV2-FEATURES/METADATA's real content)
notices the open item in context, without cross-referencing this whole list first. Update both when
a gap gets closed: remove the inline comment, check the box here.

---

## 0. Fix first — inconsistencies already in the published pages

- [x] 🔴 ~~`index.html`'s target table still shows the old, wrong data~~ — **stale entry, already
      fixed.** Both `index.html` and `README.md` have the correct Duo X=A53/Dwarf=A35/`generic-x86_64`
      table as of commit `a068042` (2026-08-24), which predates this checklist. `-new` variants
      aren't in the table itself but are covered in prose right below it (`README.md` lines 97-100).
- [x] 🔴 `README.md`'s plain-C claim — no `examples/` dir exists, so the claim was changed rather
      than an examples directory added: now reads "You can also write plain C against the LV2
      headers directly — no framework required."
- [x] 🟡 `MULTI-THREADING.md` — decided 2026-10-03: no separate file. REQUIREMENTS.md § "Threading"
      states the rule, LV2-FEATURES.md has the worker mechanics.
- [ ] 🟡 Add a real `examples/` directory: minimal plain-C-against-LV2-headers plugin(s), the thing
      README's dropped claim used to promise. Not a writing task — needs an actual worked example
      built and validated, the same way `mod-nam-loader` grounded RECIPES/VALIDATING/CAVEATS.
- [ ] 🟡 Verify the JUCE affirmations — currently unverified assertions, not evidence from a real
      build (unlike the mod-nam-loader-grounded content elsewhere). Specifically: `README.md`/
      `index.html` claim "LV2 export since version 7"; `plugins-dep/package/` in `mod-plugin-builder`
      also has `Config.in`/toolchain support for `juce-6.0` and `juce-6.1`, which either contradicts
      that claim or those older entries serve a different (non-LV2) purpose — find out which. Also
      confirm the patch-parameters-matter-for-JUCE-users claim in `LV2-FEATURES.md`. Needs an actual
      JUCE plugin built through `mod-plugin-builder`, not a doc read.

---

## 1. BUILDING.md — least developed, and the first thing a newcomer reads after README

- [x] Document the Docker path — `docker-mount.sh`'s three-case logic written up from source, plus
      what it gets you for macOS/Windows (a Linux container, not a native bootstrap)
- [x] macOS / Windows story — answered above: Docker is the real answer, no native path exists
- [x] Realistic bootstrap time and disk usage — disk usage measured per-platform on this host;
      **time now measured too** (2026-09-10): `modduo-new` 30 min, `modduox-new` 32 min, both
      bootstrapped from nothing in one attended tmux session, plus the Qt5 benign-`error:`
      noise and the RAM-is-the-constraint note
- [x] Common bootstrap failures and fixes — the toolchain-finished-but-Buildroot-never-ran failure
      mode, confirmed live on this host's `generic-aarch64` tree, with the exact fix
- [x] Add the `-new` variants — folded into a fuller platform-suffix table (`-new`/`-static`/
      `-debug`/`-kernel`/`generic-aarch64`/`-gcc15`), sourced from `.common`'s actual case logic
- [x] Recommend `-new` as the default — carried into the same table
- [x] The Cloud Builder as the no-toolchain path — written 2026-10-03 from the 2026-09-24/28
      work on `mod-cloud-builder` (browser split, 1.13.3 unit requirement, share page, hvcc pin)

## 2. RECIPES.md — solid, hands-on content already in place

- [x] Patch conventions and numbering — `NN_description.patch`, sorted-order auto-apply, real
      examples cited; the one four-digit exception explained as vendored upstream numbering
- [x] `Config.in` — confirmed it's toolchain-dependency-only (`plugins-dep`/`global-packages`);
      no plugin recipe has or needs one, with the exact glob that makes that true
- [x] Can the builder be pointed at a recipe outside its own tree? — No, confirmed from
      `BR2_EXTERNAL_PLUGINS_DEP`'s definition; said plainly, with the one-line reason why

## 3. VALIDATING.md — thorough

- [ ] What the Carla bridge runtime test (stage 4) actually asserts — instantiate/run/cleanup only, or more
      (needs a `generic-x86_64` bootstrap on this host; none yet)
- [ ] Whether stage-4 Valgrind findings are usually actionable or mostly noise on LV2 hosts

## 4. INSTALLING.md — still mostly gaps

- [x] Connecting to a device — "Reaching the unit" (2026-10-03)
- [ ] MOD Desktop install path — **currently marked explicitly unverified.** Where do bundles go?
      (mod-desktop source is not in this workspace)
- [x] What `/sdk/update` does — TTL re-read of a loaded bundle only (2026-10-03, from source)
- [x] Confirming a plugin loaded (2026-09-16)
- [x] Removing a plugin — `/package/uninstall` or `rm -rf` + restart (2026-10-03)
- [x] Reinstalling over an existing version (2026-09-16)

## 5. REQUIREMENTS.md — real-time safety section is done and good

- [x] State and persistence — REQUIREMENTS.md § "State and persistence" is the behaviour home,
      LV2-FEATURES.md the feature home (2026-10-03)
- [x] Port conventions — REQUIREMENTS.md § "Port conventions" is the home; CAVEATS keeps the one-liner
- [x] Latency reporting — not supported by `mod-host` (2026-10-03, from source)
- [x] Bypass behaviour — **answered 2026-09-17 from `mod-host/src/effects.c` @ `f14a230`:** the host
      does it, as a hard `memcpy` of input to output with no crossfade, while still calling `run()` on
      silence; sources with no audio input get zeroed audio and CV outputs; a plugin that declares an
      `lv2:enabled` port does its own bypass instead. Written up in REQUIREMENTS.md § "Verified: bypass"
- [x] Denormals — mod-host sets FZ (and DAZ on x86) on every JACK thread; the builder also links
      `crtfastmath` into every `.so` (2026-10-03, source + toolchain test)

## 6. MODGUI.md — complete

- [x] `modgui:model`/`panel`/`knob`/`color` — parsed, injected as Mustache variables only (2026-10-03,
      from source)

## 7. METADATA.md — strong

- [x] `mod:supportedExtensions` — parsed, read by nothing (2026-10-03, from source)
- [x] `mod:volts` — a unit on control ports, in or out (2026-10-03, from source)
- [x] Build-metadata properties — pipeline-written, used in version tuples and the env badge (2026-10-03)

## 8. CATEGORIES.md — vocabulary only, no prose yet

- [x] Mapping, no-category, several-classes, tabs, and MaxGen/ControlVoltage as real categories —
      all written 2026-10-03 from `utils_lilv.cpp` and `index.html`

## 9. LV2-FEATURES.md — patch parameters resolved definitively; rest still one plugin's proof

- [x] Presets — **verified 2026-09-17** on all three devices (LV2-FEATURES.md § "Verified: factory presets")
- [ ] Audit MIDI, CV — port handling now read from mod-host source (LV2-FEATURES.md "Port types");
      still not exercised by a plugin built for this documentation
- [ ] Port groups — **written up 2026-09-17 from mod-ui source @ `18dfe55a`** (LV2-FEATURES.md): `pg:group` on the
      port; `lv2:symbol`, `lv2:name`, `lv2:index` on the group; UI re-sorts by group then port index. **Confirmed on a Dwarf the same day** via `/effect/get`.
      Still to do: describe how the grouped panel looks
- [x] `mod-hmi` is developer-facing: feature + notification extension passed on MOD devices
      (2026-10-03, from source); worked example still missing

## 10. TIME.md — untouched stub

- [x] Tempo/transport mechanism — written from `mod-host` source 2026-10-03 (both the `time:Position`
      atom and the three designated control ports)
- [ ] Worked example: a tempo-synced delay

## 11. CAVEATS.md — excellent, hands-on

- [x] Known-bad dependency versions — `math_approx`'s stale CMake-3.18 requirement and
      `NeuralAmpModelerCore`'s glibc 2.27 macro collision, both with the fix already in this repo
- [x] Pure Data (hvcc) on the devices — measured on a Dwarf and a Duo 2026-10-02/03: the v0.14.0
      pin and `[expr]`, NEON on the Duo only, `[expr~]` needs SIMD off, the two v0.17.2 defects,
      what expr accepts, start-up allocations on the SIMD-off path

## 12. LICENSING.md — blocked on a decision, not a writing task

- [ ] Settle the libmodla-supply approach (how a developer actually obtains `libmodla`) before
      this chapter can be finalised — internal decision, tracked outside this repo
- [ ] Minor: "no x86_64 build" should clarify architecture vs the `generic-x86_64` target name, to
      match the rest of the docs

---

## Images

Convention: `images/<chapter-slug>/filename.png`, referenced with normal markdown image syntax —
renders on GitHub now, drops into the generated Pages site later with no path changes.

- [ ] `images/modgui/icon-vs-settings.png` — the two interfaces side by side (MODGUI.md §1)
- [ ] `images/modgui/cns-failure.png` — the missing-`{{{cns}}}` console warning next to the
      resulting default-pedal fallback (MODGUI.md §5 — the single most common mistake)
- [ ] `images/modgui/template-gallery.png` — thumbnails of the mod-sdk template families (boxy,
      british, japanese, lata, combo, head, rack) (MODGUI.md §9)
- [ ] `images/installing/store-after-install.png` — plugin visible in mod-ui after a successful
      install (INSTALLING.md — currently no way to show "it worked")
- [ ] `images/installing/device-connection.png` — USB-network or Wi-Fi connection screen
      (INSTALLING.md — once that chapter is written)
- [ ] `images/readme/architecture.png` — browser → mod-ui → mod-host → JACK → hardware diagram
      (README.md / index.html "Under the hood")
- [ ] `images/categories/store-browser.png` — where a category actually surfaces in the store
      (CATEGORIES.md)
- [ ] `images/lv2-features/patch-parameter-sequence.png` (or TIME.md) — sequence diagram of the
      atom message flow. Diagram, not a screenshot

No images planned for BUILDING, RECIPES, VALIDATING, CAVEATS, LICENSING — command output and code
read better as text than as pictures of text.
