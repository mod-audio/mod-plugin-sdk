# Caveats

Things that will cost you an afternoon if you do not know them. Each entry was hit for real,
on a build or a device, and says how it was found and how to get out of it. The first list is
general; the sections after it cover the toolchains, the compiler flags, dependency versions
that are known to break, and Pure Data plugins.

## Collected so far

- **Check the builder's CMake version rather than assuming one.** A dependency whose
  `cmake_minimum_required` is newer than the builder's CMake fails at configure time, and the fix is
  a patch lowering that line. The version has moved: two independently bootstrapped platforms on one
  machine (`moddwarf-new`, `darkglass-anagram`) both provide **CMake 3.27.7** at
  `<workdir>/<platform>/host/usr/bin/cmake`, and there is no CMake pin under `plugins-dep/package/`.
  Older bootstraps shipped 3.16.9, which is where the patches lowering minimums to 3.16 come from.
  Run `<workdir>/<platform>/host/usr/bin/cmake --version` for your own tree
- **Port symbols are permanent.** The LV2 URI and port symbols are baked into every pedalboard that
  uses your plugin. Renaming either breaks users' saved work — decide before your first release
- **The plugin URI is forever, for the same reason.** Pick a namespace you will still control
- **`{{{cns}}}` in modgui.** Omitting it silently substitutes the default pedal — see
  [MODGUI.md §5](MODGUI.md)
- **Image URLs in modgui CSS need `{{ns}}`** or they silently do not load
- **`mod-port-symbol` must match `lv2:symbol`,** not `lv2:name`. A mismatch renders a control that
  does nothing
- **armv7 vs aarch64.** MOD Duo is 32-bit ARM; everything else is 64-bit. Assumptions about pointer
  size, NEON availability and alignment differ
- **glibc 2.27 defines `major()` and `minor()` as macros.** Every device target is on glibc 2.27,
  which still pulls `<sys/sysmacros.h>` in through `<sys/types.h>`. Any struct or class of yours (or
  of a dependency) with a member called `major` or `minor` fails to compile with the memorable
  `does not have any field named 'gnu_dev_major'`. It builds fine on a modern host, where glibc
  2.28+ no longer does this, so it appears for the first time when you cross-compile. Fix it in your
  own build rather than patching the dependency: force-include a header that includes
  `<sys/types.h>` and then `#undef major` / `#undef minor` / `#undef makedev`, via
  `target_compile_options(<target> PRIVATE -include path/to/fixup.h)`. Worked example:
  `mod-nam-loader/src/sysmacros_fixup.h`
- **Some MOD plugins are device-specific, and the Store's `platforms` field is the only record of
  it.** HardwareBypass (`mod-bypass.lv2`, package `mod-utilities`) drives the true-bypass relay
  that Duo and Duo X have and the Dwarf does not; it is published for `duo+duox` only and fails to
  instantiate on a Dwarf (mod-host returns `-102`, mod-ui's `/effect/add` returns `false`). The
  builder recipe builds every bundle for every platform, so a bundle in the build output says
  nothing about where it is published: filter device test lists by the catalog column
  (`mod-plugin-management/catalog/<date>/stable-devices.csv`, `platforms`). Confirmed by
  Gianfranco 2026-09-17 after it was mis-logged as a Dwarf defect the day before
- **x86_64 is for validation, not shipping**
- **Dead documentation.** `mod-plugin-builder`'s README links twice to `wiki.moddevices.com`, which
  no longer exists. This repository replaces those pages
- **A CMake build directory pins the absolute path it was configured from.** Move or rename the
  repo checkout on disk (e.g. relocating it into a different parent directory) and a pre-existing
  build dir refuses to reconfigure: `CMake Error: The current CMakeCache.txt directory ... is
  different than the directory ... where CMakeCache.txt was created.` There's no fix but deleting
  the stale build directory and reconfiguring from scratch — it's disposable output, so this is
  safe, just easy to be surprised by. Verified 2026-08-28 on `mod-neural-amps/build-anagram/`
  after the repo moved from `~/MOD/Plugins/` to `~/MOD/Repos/Plugins/`
- **`cmake --install` runs every `install()` rule regardless of which targets you actually built.**
  In a multi-plugin repo built with one target per plugin (e.g. `mod-neural-amps`'s
  `add_plugin(TARGET BUNDLE ...)` pattern in `CMakeLists.txt`), building a subset with
  `cmake --build <dir> --target <name1> --target <name2>` leaves the other plugins' `.so` files
  absent — a subsequent full `cmake --install <dir>` then fails on the first `install(TARGETS ...)`
  for a target that was never built. There's no per-target install without CMake `COMPONENT`
  tags, which these repos don't define. To install just the plugins you built, assemble each
  bundle by hand instead: copy the built `.so` plus that bundle's `.ttl`/`modgui` assets from
  source, mirroring the `install()` rules' own layout. Verified 2026-08-28 building `overjay` +
  `the-rude` only for `darkglass-anagram` in `mod-neural-amps`
- **Never bake the block size in.** The host's block size is not 128, 256 or any other
  constant; read `bufsz:maxBlockLength`/`nominalBlockLength` from the `opts:options` feature
  at instantiate, or size for the largest value you accept and reject bigger blocks. MOD's own
  catalog carries the counter-examples: `mod-utilities` hard-codes `TAMANHO_DO_BUFFER 1024`
  and the builder patches it to 8192 with a `FIXME use options interface`
  (`mod-plugin-builder/plugins/package/mod-utilities/02_fix-buffer-size.patch`);
  `mod-distortion`'s DS1 `realloc()`s nine buffers inside `run()` the first time `n_samples`
  exceeds 256 (`ds1/src/DS1.cpp:199-209`). Only 11 of the 101 open-source repos behind the
  stable catalog read the options feature at all (2026-09-16 scan,
  `Plugins/mod-plugin-management/analysis/corpus-scan.md`). *Both fixed 2026-09-16 on local
  branches (`mod-utilities` `fix/block-size`, `mod-distortion` `fix/block-size`): buffers sized
  from `bufsz:maxBlockLength` at instantiate, `run()` chunked to that size — see the shared
  `Shared_files/BufferSize.h` in either repo for the pattern; exercised on a Dwarf 2026-09-16, branches
  pushed 2026-10-03, not yet merged or published*
- **Don't re-plan or re-allocate from `run()` on a parameter change.** `mod-pitchshifter`'s
  Harmonizer family calls `Realloc()` from `run()` whenever the fidelity knob or the block
  size changes, which deletes and re-creates its FFTW objects — planning plus a wisdom-file
  read on the audio thread (`Harmonizer/src/Harmonizer.cpp:59-80,174`; all eight plugins in
  the repo share it). Pre-build every configuration at instantiate, or do it on the worker
  thread and swap a pointer. *Fixed 2026-09-16 on `mod-pitchshifter` `fix/rt-safe-fidelity`:
  every fidelity is built at instantiate and the knob swaps pointers; a block size other than
  the one built for goes through `Shared_files/BlockAdapter.h` (one hop of latency) instead of
  re-planning; exercised on a Dwarf 2026-09-16 and heard by Gianfranco 2026-09-17, branch pushed 2026-10-03,
  not yet merged or published*
- **Check the option's type the right way round.** `mod-pitchshifter`'s `GetBufferSize()` did
  `if (options[i].type == atom:Int) continue;` — it skipped precisely the entries it wanted and
  always returned its 128 default, so on a 256-frame Duo X the first `run()` took the
  re-allocation path above. Found 2026-09-16 while fixing it. Compare `!=` and `continue`, or
  better, copy `mod_block_capacity()` from `mod-utilities/Shared_files/BufferSize.h`
- **Hard-coded compiler flags in your Makefile will be wrong on at least one target.** The
  builder compiles every package with the platform's `BR2_TARGET_OPTIMIZATION` (`-O3
  -ffast-math ... -mcpu=cortex-a35` on Dwarf, `-mfpu=neon-vfpv4 -mfloat-abi=hard` on Duo).
  Makefiles that add their own `-msse2`, `-march=armv7-a` or `-march=native` either break the
  cross-compile or silently win over the platform flags. 30 of the 101 open-source repos
  behind the stable catalog do this; the builder survives by passing `OPTIMIZATIONS=` /
  `NOOPT=true` in 33 recipes. Follow x42's convention so a recipe can override you:
  `OPTIMIZATIONS ?= ...` and `CXXFLAGS += $(OPTIMIZATIONS)`, never `:=`
- **The platform flags include `-ffast-math -funsafe-loop-optimizations` for everyone.** Two
  packages carry patches for NaNs that only appear under them (`caps-lv2/003_fix-
  overoptimization-nan-kaiser.patch`, `artyfx/02_filta-workaround-over-optimization.patch`)
  and `aidadsp-lv2.mk` filters `-funsafe-loop-optimizations` back out. If your DSP relies on
  IEEE semantics (NaN checks, signed zeros, `isinf`), test under the real flags. For denormals
  see the next entry
- **Denormals: do not count on flush-to-zero being on for your `run()`.** Measured 2026-10-03
  with the `moddwarf-new` and `modduo-new` toolchains: the `<platform>/host/usr/bin/<triple>-gcc`
  you build with is Buildroot's `toolchain-wrapper`, which injects the platform's
  `-ffast-math`, and GCC 9.4.0's link spec adds `crtfastmath.o` to **every** link, shared objects
  included (`%{Ofast|ffast-math|funsafe-math-optimizations:crtfastmath.o%s}` in `-dumpspecs`,
  with no `!shared` guard; newer GCCs have one). So each plugin `.so` carries a constructor that
  writes the FZ bit into `FPCR` (`msr fpcr, x0`; `vmsr fpscr` on the Duo) when it is loaded;
  the same object built with the raw `toolchain/bin/<triple>-gcc` and no `-ffast-math` has no
  such write. But `FPCR` is a per-thread register, and the constructor runs on the thread that
  `dlopen`s the bundle, which is mod-host's command thread, not the JACK audio thread that will
  call `run()`. What saves you on a MOD device is mod-host, which sets FZ on every JACK thread
  itself (see [REQUIREMENTS.md](REQUIREMENTS.md) § "Verified: denormals are flushed by the host");
  on any other host do not count on your build flags. The portable answer is the usual one: add
  a tiny DC offset or noise to feedback paths, or set and restore the FZ bit yourself at the
  start and end of `run()` (one `mrs`/`msr` pair, negligible cost)

## Compiler and libc per target

Verified 2026-08-24 from `mod-plugin-builder/toolchain/<target>.config` (`CT_ARCH_CPU`,
`CT_GCC_VERSION`, `CT_GLIBC_V_*`):

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

Verified 2026-10-02/03 with generated test patches, cross-built for the Dwarf and the Duo with
the Cloud Builder's recipe and pins (hvcc v0.14.0 and v0.17.2, DPF `61d38eb`) and run on a
Dwarf and a Duo, both on v1.14.0.3366, with `rt_check` (mod-plugin-builder attempts 058 and
060 in `build-records/plugins/log.md`; Duo X build and emulation only, attempt 059).

- **builder.mod.audio builds Pd patches with hvcc v0.17.2 since 2026-10-03** (before that
  v0.14.0, which failed on `[expr]`/`[expr~]` with `Don't know how to parse object "expr"`).
  The pin is `PURE_DATA_SKELETON_VERSION` in mod-cloud-builder `webserver/server.py`. The
  builder sets `nosimd` itself when a patch contains `expr~` and ships the include fix below.
  Unsupported in `expr`/`expr~` as of v0.17.2: `mtof ftom dbtorms rmstodb powtodb dbtopow size
  sum Sum avg Avg random`, `drem` in `expr~`, table and symbol access, float inlets (`$f2`) in
  `expr~`, `fexpr~`.
- **Abstractions must be uploaded flat.** The builder writes every uploaded file into one
  folder: upload every abstraction the patch uses, including the ones your abstractions use,
  and drop folder prefixes (`wstd.cmpnnts/eq_pass` → `eq_pass`). `declare -path` is ignored.
  The project's own hvcc metadata JSON is not used; the builder generates generic metadata
  (no UI, no port groups, no enumerators).
- **Heavy's SIMD-off code path allocates on the audio thread right after start.** On the Dwarf
  (no SIMD, see below): 17 `malloc` calls for a patch with one `[r name @hv_param]`, 34 with
  heavylib abstractions, 0 without parameters. On the Duo the NEON build of the same patch makes
  0, its `nosimd` build 17 (attempt 060). Same with v0.14.0 and v0.17.2. One-off, not per
  block, no locks.
- **SIMD off costs a plain patch about x2 on the Duo** (0.015 → 0.032 ms per 128-frame block for
  a gain patch, +15 % for a reverb from heavylib), so set `nosimd` only when the patch needs it.
  On the Dwarf and Duo X it changes nothing.
- **heavylib's `hv.reverb~` on the Duo:** one block over the 2.667 ms deadline per run
  (3.8–4.7 ms, not seen on the Dwarf) and a different output level between the NEON and the
  SIMD-off build. Same with v0.14.0 and v0.17.2; cause not investigated.
- **Heavy uses NEON on the Duo only.** It selects NEON on `__ARM_NEON__`. The Duo toolchain
  defines it; the aarch64 toolchains (Duo X, Dwarf) define only `__ARM_NEON`
  (`echo | <cross-gcc> -dM -E - | grep -i neon`), so on those two the generated code runs
  without SIMD.
- **`[expr~]` (hvcc v0.17.2) needs SIMD off.** With NEON its math functions are
  `hv_assert(0)` stubs and a numeric constant does not compile. Set `"nosimd": true` at the top
  level of the hvcc metadata JSON for a Duo build; built with NEON it outputs full-scale garbage
  (peak 4.0 for a 0.2 sine on a real Duo) from a build that reports success. `expr~ tanh($v1*4)`
  matched the reference to 8e-8 on both units and cost 0.037 ms per 128-frame block on the Dwarf,
  0.087 ms on the Duo (deadline 2.667 ms).
- **`expr~` with a numeric constant fails to compile in hvcc v0.17.2** (`'__hv_var_k_i' was not
  declared`) unless the patch also contains an object that pulls in `HvSignalVar.h`, such as
  `sig~`. `SignalExpr.get_C_header_set()` omits the header; not fixed upstream as of this
  writing.
- **What hvcc v0.17.2's `[expr]` / `[expr~]` accept** (scan of every function, SIMD off,
  with the include fix above; `notes/hvcc-upgrade/results/expr-scan-v0.17.2-patched.txt` in
  the MOD workspace): the C math set works in both — `abs acos acosh asin asinh atan atan2
  atanh cbrt ceil copysign cos cosh erf erfc exp expm1 fact finite float floor fmod if imodf
  int isinf isnan ldexp ln log log10 log1p max min modf pow remainder rint round nearbyint sin
  sinh sqrt tan tanh trunc`, plus `$i` inlets and several outputs separated by `;` in `[expr]`.
  Not available, in either: Pd's `mtof ftom dbtorms rmstodb powtodb dbtopow size sum Sum avg
  Avg` (documented by hvcc) and **`random`** (not documented: C compile error in `[expr]`, parse
  error in `[expr~]`). Also missing: `drem` in `[expr~]` (use `remainder`), table reads
  (`$s2[...]`), symbol inlets, `[fexpr~]`, and **float inlets (`$f2`) in `[expr~]`**, which
  make hvcc exit with a Python `NotImplementedError` (signal inlets `$v` only). In control
  `[expr]`, `%`, `&`, `|`, `^` and `~` fail at C compile time on `$f` inlets; use `$i`.
- **A Pd build on builder.mod.audio also builds the JACK and VST targets you did not ask for.**
  The generated `plugin.json` spells the key as `"plugin_formats "` (trailing space), so hvcc
  ignores it and builds its defaults. Harmless, only slower.
