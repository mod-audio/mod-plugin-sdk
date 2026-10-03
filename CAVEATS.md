# Caveats

Things that will cost you an afternoon if you do not know them. The first list is general; the
sections after it cover toolchains, compiler flags, dependency versions known to break, and
Pure Data plugins.

## Collected so far

- **Check the builder's CMake version rather than assuming one.** A dependency whose
  `cmake_minimum_required` is newer than the builder's CMake fails at configure time, and the fix is
  a patch lowering that line. The version has moved: current bootstraps provide **CMake 3.27.7** at
  `<workdir>/<platform>/host/usr/bin/cmake`, and there is no CMake pin under `plugins-dep/package/`.
  Older bootstraps shipped 3.16.9, which is where patches lowering minimums to 3.16 come from.
  Run `<workdir>/<platform>/host/usr/bin/cmake --version` for your own tree.
- **Port symbols are permanent.** The LV2 URI and port symbols are baked into every pedalboard that
  uses your plugin. Renaming either breaks users' saved work — decide before your first release.
- **The plugin URI is forever, for the same reason.** Pick a namespace you will still control.
- **`{{{cns}}}` in modgui.** Omitting it silently substitutes the default pedal — see
  [MODGUI.md §5](MODGUI.md).
- **Image URLs in modgui CSS need `{{ns}}`** or they silently do not load.
- **`mod-port-symbol` must match `lv2:symbol`,** not `lv2:name`. A mismatch renders a control that
  does nothing.
- **armv7 vs aarch64.** MOD Duo is 32-bit ARM; everything else is 64-bit. Assumptions about pointer
  size, NEON availability and alignment differ.
- **glibc 2.27 defines `major()` and `minor()` as macros.** Every device target is on glibc 2.27,
  which still pulls `<sys/sysmacros.h>` in through `<sys/types.h>`. A struct or class with a member
  called `major` or `minor` fails to compile with the memorable `does not have any field named
  'gnu_dev_major'`. It builds fine on a modern host (glibc 2.28+ no longer does this), so it appears
  for the first time when you cross-compile. Fix it in your own build rather than patching the
  dependency: force-include a header that includes `<sys/types.h>` and then `#undef major` /
  `#undef minor` / `#undef makedev`, via `target_compile_options(<target> PRIVATE -include
  path/to/fixup.h)`. Worked example: `mod-nam-loader/src/sysmacros_fixup.h`.
- **Some plugins are device-specific, and the catalog's `platforms` field is the only record of
  it.** A plugin that drives hardware only some devices have (a true-bypass relay, say) can be
  published for a subset of targets and fail to instantiate on the rest (mod-host returns an error;
  mod-ui's `/effect/add` returns `false`). The builder builds every bundle for every platform
  regardless, so a bundle existing in the build output says nothing about where it's actually
  published — filter by the catalog's `platforms` column, not by what built successfully.
- **x86_64 is for validation, not shipping.**
- **Dead documentation.** `mod-plugin-builder`'s README links twice to `wiki.moddevices.com`, which
  no longer exists. This repository replaces those pages.
- **A CMake build directory pins the absolute path it was configured from.** Move or rename the
  repo checkout on disk and a pre-existing build dir refuses to reconfigure: `CMake Error: The
  current CMakeCache.txt directory ... is different than the directory ... where CMakeCache.txt
  was created.` There's no fix but deleting the stale build directory and reconfiguring from
  scratch — it's disposable output, so this is safe, just easy to be surprised by.
- **`cmake --install` runs every `install()` rule regardless of which targets you actually
  built.** In a multi-plugin repo built with one CMake target per plugin, building a subset with
  `cmake --build <dir> --target <name1> --target <name2>` leaves the other plugins' `.so` files
  absent — a subsequent full `cmake --install <dir>` then fails on the first `install(TARGETS
  ...)` for a target that was never built. There's no per-target install without CMake
  `COMPONENT` tags, which these repos typically don't define. To install just the plugins you
  built, assemble each bundle by hand instead: copy the built `.so` plus that bundle's
  `.ttl`/`modgui` assets from source, mirroring the `install()` rules' own layout.
- **Never bake the block size in.** The host's block size is not 128, 256 or any other constant;
  read `bufsz:maxBlockLength`/`nominalBlockLength` from the `opts:options` feature at
  instantiate, or size for the largest value you accept and reject bigger blocks. Treating the
  block size as fixed is a common source of bugs that only appear on a device configured
  differently from your dev machine.
- **Don't re-plan or re-allocate from `run()` on a parameter change.** Recomputing an FFT plan
  or re-reading a wisdom file inside `run()` when a knob changes puts allocation and heavy setup
  on the audio thread. Pre-build every configuration at instantiate, or do it on the worker
  thread and swap a pointer; for a block-size change you don't have a pre-built variant for, add
  one hop of latency through an adapter rather than re-planning in place.
- **Check the option's type the right way round when reading `opts:options`.** A backwards type
  check (`if (type == atom:Int) continue;` instead of `!=`) silently skips the entries you
  wanted and falls back to a default — the symptom is a block-size assumption that's wrong only
  on the one target where the default doesn't match reality. Worth a dedicated unit test rather
  than visual review.
- **Hard-coded compiler flags in your Makefile will be wrong on at least one target.** The
  builder compiles every package with the platform's `BR2_TARGET_OPTIMIZATION` (`-O3
  -ffast-math ... -mcpu=cortex-a35` on Dwarf, `-mfpu=neon-vfpv4 -mfloat-abi=hard` on Duo).
  A Makefile that adds its own `-msse2`, `-march=armv7-a` or `-march=native` either breaks the
  cross-compile or silently wins over the platform flags. Follow x42's convention so a recipe
  can override you: `OPTIMIZATIONS ?= ...` and `CXXFLAGS += $(OPTIMIZATIONS)`, never `:=`.
- **The platform flags include `-ffast-math -funsafe-loop-optimizations` for everyone.** Some
  dependencies carry patches for NaN issues that only appear under them, and at least one
  recipe filters `-funsafe-loop-optimizations` back out. If your DSP relies on IEEE semantics
  (NaN checks, signed zeros, `isinf`), test under the real flags. For denormals, see below.
- **Denormals: do not count on flush-to-zero being on for your `run()` by virtue of your own
  build.** The toolchain's GCC 9.4.0 link spec adds `crtfastmath.o` to every link, shared objects
  included, with no `!shared` guard (newer GCCs have one) — so each plugin `.so` carries a
  constructor that sets the FZ bit when it's loaded. But that constructor runs on the thread
  that `dlopen`s the bundle (mod-host's command thread), not the JACK audio thread that calls
  `run()` — `FPCR` is per-thread. What actually saves you on a MOD device is mod-host, which
  sets FZ on every JACK thread itself (see [REQUIREMENTS.md](REQUIREMENTS.md) § "Denormals are
  flushed by the host"); on any other host, don't count on your build flags. The portable
  answer: add a tiny DC offset or noise to feedback paths, or set and restore the FZ bit
  yourself at the start and end of `run()`.

## Compiler and libc per target

From `mod-plugin-builder/toolchain/<target>.config`:

| Target | CPU | GCC | glibc |
|---|---|---|---|
| `modduo` | Cortex-A7 (ARMv7) | ct-ng default (older) | — |
| `modduox` | Cortex-A53 | ct-ng default (older) | — |
| `moddwarf` | Cortex-A35 | ct-ng default (older) | 2.27 |
| `modduo-new` | Cortex-A7 (ARMv7) | 9.4.0 | 2.27 |
| `modduox-new` | Cortex-A53 | 9.4.0 | 2.27 |
| `moddwarf-new` | Cortex-A35 | 9.4.0 | 2.27 |

The plain targets pin no GCC version, so they take crosstool-NG's default for their config
vintage; the `-new` targets pin 9.4.0 explicitly. **GCC 9.4.0 means C++17 is comfortable and C++20
is partial** — `cmake_minimum_required` style feature checks matter, and `CMAKE_CXX_STANDARD 20`
should be paired with `CMAKE_CXX_STANDARD_REQUIRED OFF` if you want it to degrade rather than fail.

glibc 2.27 also sets your symbol-version ceiling: check with
`objdump -T yourplugin.so | grep -o 'GLIBC_[0-9.]*' | sort -uV | tail -1`. Anything above
`GLIBC_2.27` will not load on a device.

## Optimisation flags

Device defconfigs compile with `-ffast-math -fno-finite-math-only` (plus `-fomit-frame-pointer`,
`-fprefetch-loop-arrays`, `-ftree-vectorize`, `-funroll-loops` and `-mcpu`/`-mtune` for the target).
So: **IEEE semantics are not guaranteed** — no strict rounding, reassociation is allowed, and
`errno` is not set by math functions — but `-fno-finite-math-only` means NaN and Inf are still
handled rather than assumed away. If your DSP depends on strict IEEE behaviour, say so in your
recipe by filtering the flag out; several MOD recipes already filter individual flags this way.

## Known-bad dependency versions

- **`math_approx`** (a CMake dependency pulled in transitively by `NeuralAudio`, itself a
  dependency of `neural-amp-modeler-lv2`) requires `cmake_minimum_required(VERSION 3.18)`.
  This only bites on a builder tree still pinning the old CMake 3.16.9 (see above) — patched
  down to 3.16 in `plugins/package/neural-amp-modeler-lv2/06_fix-math-approx-cmake-version.patch`.
  Harmless but unnecessary on a tree that already provides 3.27.7.
- **`NeuralAmpModelerCore`** (used by both `neural-amp-modeler-lv2` and MOD's own
  `mod-nam-loader`) hits the glibc 2.27 `major()`/`minor()` macro collision described above on
  every device target. Not fixed upstream as of this writing.

## Pure Data (hvcc) plugins on MOD devices

Cross-built for the Dwarf and the Duo with the Cloud Builder's recipe and pins, and run on real
hardware:

- **builder.mod.audio builds Pd patches with hvcc v0.17.2 as of this writing** (previously
  v0.14.0, which failed on `[expr]`/`[expr~]` with `Don't know how to parse object "expr"`).
  The builder sets `nosimd` itself when a patch contains `expr~` and ships an include fix (see
  below). Unsupported in `expr`/`expr~` as of v0.17.2: `mtof ftom dbtorms rmstodb powtodb dbtopow
  size sum Sum avg Avg random`, `drem` in `expr~`, table and symbol access, float inlets (`$f2`)
  in `expr~`, `fexpr~`.
- **Abstractions must be uploaded flat.** The builder writes every uploaded file into one
  folder: upload every abstraction the patch uses, including the ones your abstractions use,
  and drop folder prefixes. `declare -path` is ignored. The project's own hvcc metadata JSON is
  not used; the builder generates generic metadata (no UI, no port groups, no enumerators).
- **Heavy's SIMD-off code path allocates on the audio thread right after start.** On the Dwarf
  (no SIMD, see below): roughly 15–35 one-off `malloc` calls depending on how many parameters
  and heavylib abstractions the patch uses; none per block, no locks. On the Duo the NEON build
  of the same patch makes none at all; its `nosimd` build matches the Dwarf's count.
- **SIMD off costs a plain patch about 2× on the Duo** (0.015 → 0.032 ms per 128-frame block for
  a gain patch, +15% for a reverb from heavylib), so set `nosimd` only when the patch needs it.
  On the Dwarf and Duo X it changes nothing.
- **heavylib's `hv.reverb~` on the Duo:** one block over the 2.667 ms deadline per run (not seen
  on the Dwarf), and a different output level between the NEON and the SIMD-off build. Cause
  not investigated.
- **Heavy uses NEON on the Duo only.** It selects NEON on `__ARM_NEON__`. The Duo toolchain
  defines it; the aarch64 toolchains (Duo X, Dwarf) define only `__ARM_NEON`, so on those two the
  generated code runs without SIMD.
- **`[expr~]` (hvcc v0.17.2) needs SIMD off.** With NEON its math functions are stubs and a
  numeric constant doesn't compile — built with NEON it outputs full-scale garbage from a build
  that reports success. Set `"nosimd": true` at the top level of the hvcc metadata JSON for a
  Duo build.
- **`expr~` with a numeric constant fails to compile in hvcc v0.17.2** unless the patch also
  contains an object that pulls in the signal-variable header, such as `sig~`. Not fixed
  upstream as of this writing.
- **What hvcc v0.17.2's `[expr]`/`[expr~]` accept:** the C math set works in both — `abs acos
  acosh asin asinh atan atan2 atanh cbrt ceil copysign cos cosh erf erfc exp expm1 fact finite
  float floor fmod if imodf int isinf isnan ldexp ln log log10 log1p max min modf pow remainder
  rint round nearbyint sin sinh sqrt tan tanh trunc`, plus `$i` inlets and several outputs
  separated by `;` in `[expr]`. Not available, in either: Pd's `mtof ftom dbtorms rmstodb powtodb
  dbtopow size sum Sum avg Avg` and `random`. Also missing: `drem` in `[expr~]` (use
  `remainder`), table reads, symbol inlets, `[fexpr~]`, and float inlets (`$f2`) in `[expr~]`
  (signal inlets `$v` only). In control `[expr]`, `%`, `&`, `|`, `^` and `~` fail at C compile
  time on `$f` inlets; use `$i`.
- **A Pd build on builder.mod.audio also builds the JACK and VST targets you did not ask for.**
  A formatting quirk in the generated metadata means the builder ignores your plugin-format
  choice and builds its defaults. Harmless, only slower.
