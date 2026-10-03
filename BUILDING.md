# Building

MOD plugins are cross-compiled with [`mod-plugin-builder`](https://github.com/mod-audio/mod-plugin-builder),
a public, open-source wrapper around [Buildroot](https://buildroot.org/) and crosstool-NG. It
builds one toolchain per device (`./bootstrap.sh`), then builds any plugin described by a
Buildroot package recipe against it (`./build`). The recipe format is the subject of
[RECIPES.md](RECIPES.md); this chapter is about getting the toolchain and running builds.

It runs on Linux. On macOS or Windows the supported path is the Docker container the repo ships
(see "Docker path" below); there is no native bootstrap for either. A bootstrap takes about half
an hour and 12–15 GB per platform on a modern machine, once; a plugin build after that takes a
minute or two. Every figure in this chapter was measured on a real bootstrap or build, with the
date, rather than copied from the upstream README, which is out of date in places (its
`wiki.moddevices.com` links are dead; this repository replaces them).

If you do not want a local toolchain at all, [builder.mod.audio](http://builder.mod.audio) builds
FAUST, Max gen~, Pure Data and `.mk` recipes in the cloud and installs the result on a unit over
USB. See "Building without a toolchain" at the end.

## The commands

Host dependencies (Debian/Ubuntu), from the repo README:

```
sudo apt install acl bc curl cvs git mercurial rsync subversion wget \
bison bzip2 flex gawk gperf gzip help2man nano perl patch tar texinfo unzip \
automake binutils build-essential cpio libtool libcrypt-dev libncurses-dev pkg-config python-is-python3 libtool-bin
```

Bootstrap — builds ct-ng toolchain + Buildroot, **around half an hour** per platform on a
modern multi-core machine (see "Bootstrap time" below; upstream's README says "more than 1
hour"):

```
./bootstrap.sh <platform>
```

Output goes to `~/mod-workdir`; override with `WORKDIR`.

Build a plugin:

```
./build <platform> <plugin-package>
./build <platform> <plugin-package>-dirclean     # clean
```

Bundles land in `~/mod-workdir/<platform>/plugins`.

Building the bundled example plugins needs submodules:

```
git submodule init && git submodule update
```

## Platform name suffixes

A `<platform>` argument is not just a device name — suffixes change which toolchain and which
Buildroot config get used (`mod-plugin-builder/.common:45-71`):

| Suffix | Effect |
|---|---|
| *(none)* | `modduo`/`modduox`/`moddwarf` — the original toolchain for that device (ct-ng default GCC, see [CAVEATS.md](CAVEATS.md)) |
| `-new` | Same device, newer pinned toolchain (GCC 9.4.0, glibc 2.27). **This is what MOD's own current builds use — prefer it** |
| `-static` | Intermediate ct-ng-1.24 toolchain, statically linked (`modduo-static`, `modduox-static`) |
| `-debug` | Same toolchain as the base name, different (debug) Buildroot defconfig |
| `-kernel` | Builds the Linux kernel instead of a plugin, always against the `-new` toolchain regardless of what you typed |
| `generic-aarch64-*` / `darkglass-anagram*` (no `-gcc15`) | Share one `generic-aarch64` toolchain, each with its own Buildroot target |

`darkglass-anagram-gcc15` and `moddwarf-gcc15` are further one-off custom targets, each with
their own dedicated toolchain (ct-ng 1.28, GCC 15).

## Docker path

`docker-mount.sh <platform> [extra-mount-path]` (verified against its source,
`mod-plugin-builder/docker-mount.sh`) is the whole mechanism — there's nothing beyond this script
to learn:

1. If an image tagged `mpbi_<platform>` doesn't exist yet, it builds one from `docker/Dockerfile`
   (`docker buildx build --build-arg platform=<platform> --build-arg target=toolchain`).
2. If a container named `mpb_<platform>` already exists, it just restarts and attaches to it
   (`docker start -i`).
3. Otherwise it creates one (`docker run -ti`, no `--privileged`, no special `--ulimit`), bind-mounting
   the repo checkout to `/home/builder/mod-plugin-builder` and either your `PLUGINS_DIR` or an
   extra path you pass in to `/mnt`.

Once attached you're in a normal Linux shell with the toolchain image's dependencies pre-installed —
run `./bootstrap.sh <platform>` and `./build <platform> <plugin>` inside it exactly as on a native
Linux host. This is the real answer for macOS and Windows today: there's no native bootstrap for
either, but a Docker container gives you the same Linux environment the README's dead
`wiki.moddevices.com` link used to describe.

## Disk usage — measured on this host's bootstrapped trees

```
du -sh <workdir>/<platform>
```

| Platform | Size | Notes |
|---|---|---|
| `moddwarf-new` | 15 GB | full bootstrap + built plugins; `build/` (intermediate Buildroot state) alone is 13 GB, `host/` (host-side tools, e.g. CMake) 735 MB, `target/` (device rootfs) 139 MB, toolchain sysroot 289 MB |
| `darkglass-anagram` | 3.9 GB | full bootstrap |
| `modduo-new` | 12 GB | full bootstrap + built plugin |
| `modduox-new` | 12 GB | full bootstrap + built plugin |
| `generic-aarch64` | 7.6 GB | **toolchain only** — see the failure mode below |
| `download/` | 1.4 GB | shared source-tarball cache across all platforms |

`build/` is by far the largest and most disposable component — the actual sysroot/host/target
directories are what you need to keep working.

## Bootstrap time — measured

Two platforms were bootstrapped from nothing in one attended session (2026-09-10), each run
start-to-finish under `tmux` with a tee'd log, so these are real single-run durations rather
than numbers back-computed from file timestamps:

| Platform | Bootstrap | Tree size |
|---|---|---|
| `modduo-new` (32-bit ARM Cortex-A7) | **30 min** | 12 GB |
| `modduox-new` (aarch64 Cortex-A53) | **32 min** | 12 GB |

1h05m for both end to end, run **sequentially**, on a 12-core / 7 GB-RAM Linux host with the
workdir on an external SSD. Upstream's README estimate of "more than 1 hour" is conservative
for hardware of that class, but the shape of the machine matters more than the core count:
**RAM is the binding constraint**, so running two bootstraps in parallel on a small-memory
machine is slower than running them one after the other, not faster.

Following those, a single plugin build (`./build <platform> <plugin>`) took **89–93 seconds**
and `./validate` was effectively instant. The multi-hour cost is the toolchain, once per
platform; everything after it is short.

**Benign noise to expect:** each bootstrap log contains roughly 26 `error:` / `fatal error`
lines from Qt5's configure feature probes (openvg, X11, xkbcommon, sybase, sqlite2). They are
probes for optional features that are absent by design. Judge a bootstrap by its exit status
and by whether `build/<buildroot-version>/` exists, not by grepping the log for "error".

## Alternate build path — repo-embedded direct cross-compile scripts

Not every MOD plugin repo builds through `mod-plugin-builder`'s `./build <platform> <plugin>`
recipe mechanism described above. `mod-neural-amps` (verified 2026-08-28) instead vendors its
own `build-for-pablito.sh`, which cross-compiles directly with CMake against a
`mod-plugin-builder`-bootstrapped toolchain's own output directories — same
`WORKDIR`-overridable convention, but pointed at `<workdir>/<platform>/host`,
`.../staging`, `.../target` and `.../host/usr/share/buildroot/toolchainfile.cmake` itself,
rather than going through `./build`. Both paths consume the same bootstrapped toolchain tree;
they're just two different front ends to it. **Check which one a given repo uses before
assuming `./build <platform> <plugin>` is the entry point** — grep the repo root for a
`build-for-*.sh` or similar before reaching for `mod-plugin-builder`.

## Common bootstrap failures

**Toolchain finished, Buildroot stage never ran.** `bootstrap.sh` builds the ct-ng toolchain
first, then downloads/extracts Buildroot (`bootstrap.sh:89-199`). If the script is interrupted
between those stages, `<workdir>/<platform>/toolchain/` and `.../<triple>/` are fully populated
but `<workdir>/<platform>/build/<buildroot-version>/` doesn't exist yet — confirmed on this host's
`generic-aarch64` tree. Symptom, from `./build`:
```
./build: line 121: cd: <workdir>/<platform>/build/<buildroot-version>: No such file or directory
```
**Fix: just re-run `./bootstrap.sh <platform>`.** The ct-ng stage is checkpointed with
`.stamp_configured` / `.stamp_built1` / `.stamp_patched` / `.stamp_built2` marker files
(`bootstrap.sh:93-152`) and the Buildroot stage is guarded by `if [ ! -d
${BUILD_DIR}/${BUILDROOT_VERSION} ]` (`bootstrap.sh:156`) — re-running skips the completed
toolchain entirely and picks up exactly where it stopped. No need to delete anything first.

## Building a local working tree without pushing it

Verified 2026-09-16 (moddwarf-new, `build-records/plugins` attempts 030–034, five packages).
The builder fetches every package from its recipe's `_SITE`/`_VERSION`, so testing a fix
normally means pushing first. Buildroot's source override avoids that: put

```
MOD_UTILITIES_OVERRIDE_SRCDIR = /home/you/MOD/Repos/Plugins/mod-utilities
```

(the `_VERSION` variable's prefix, i.e. the package name upper-cased with `-` → `_`) into
`<WORKDIR>/<platform>/local.mk` — the file already exists, empty, in every bootstrapped tree —
and run `./build <platform> <package>-rebuild`. The `-rebuild` matters: for an override
package it re-rsyncs the tree into `build/<package>-custom/` before building; a plain
`./build` after the first one reuses the stale copy. Three things to know:

- **Recipe patches are not applied** to an override tree (Buildroot skips the patch step), so
  fold them into the tree first — which is what you want when the goal is to retire them.
- **An override left in `local.mk` silently replaces the pin for every later build of that
  package on the host.** Empty the file when done.
  `Plugins/mod-plugin-management/scripts/build-local.sh` wraps all of this and clears it on
  exit.
- The pinned-fetch path is still untested until the branch is pushed and `./build` runs once
  without the override. Say so in the attempt record.

The override rsync excludes `.git`, so a submodule checkout (e.g. `mod-audio-mixer-lv2`'s
`dpf/`) comes along as plain files — fine for building.

**A new plugin added to a multi-bundle package builds with the recipe unchanged, but only as
far as `target/`.** Verified 2026-09-17 (attempt 055): a ninth directory added to
`mod-pitchshifter`'s top-level Makefile was compiled and installed into
`<WORKDIR>/<platform>/target/usr/lib/lv2/mod-polyoctaver.lv2` by the untouched recipe, in 4 s,
because the recipe only runs the repo's `make` and `make install`. It was **not** copied to
`<WORKDIR>/<platform>/plugins/`: `./build` copies exactly the bundles named in the recipe's
`<PKG>_BUNDLES` line (`build:93`, `try_copy_plugin_bundles`), and `./validate` and `./publish`
read the same line (`validate:38`, `publish:71`). Its "possibly missing from _BUNDLES" note
only prints when `plugins/` was empty beforehand, so on a used tree nothing warns you. Take a
trial bundle from `target/usr/lib/lv2/`, and add it to `_BUNDLES` before pinning the branch.

## Common plugin-build failures

**An interrupted `./build` is NOT safe to just re-run.** This is the opposite of the
bootstrap advice above, and the difference matters: `bootstrap.sh` is checkpointed with
stamp files, so re-running resumes correctly. A plugin build is not — it is a parallel
`make`, and re-running it can silently produce a broken plugin that exits 0.

If a `make -j<N>` is killed partway (dropped SSH session, machine reset, Ctrl-C), every
object file open for writing at that instant is left **zero bytes**. On a re-run, make
compares mtimes, sees each zero-byte `.o` as *newer* than its source, skips recompiling it,
and links the empty objects. The link succeeds. The build reports success.

Confirmed on this host with `neural-amp-modeler-lv2`: a reset during the compile left 12 of
16 objects at zero bytes, and the resumed build produced a **30 KB** `.so` where a sound one
is ~1.5 MB, with the entire DSP engine missing:

```
$ readelf --dyn-syms -W neural_amp_modeler.so | awk '$7=="UND"' | grep NeuralAudio
UND  _ZN11NeuralAudio17NeuralModelLoader14CreateFromFileERKNSt10filesystem7__cxx114pathEb
$ readelf -d neural_amp_modeler.so | grep NEEDED
    libstdc++.so.6  libm.so.6  libgcc_s.so.1  libpthread.so.0  libc.so.6
```

Nothing provides that symbol, so the plugin installs and then fails to instantiate on the
device — a failure that only shows up on real hardware.

**Fix: wipe the package build directory and rebuild.**

```bash
rm -rf <workdir>/<platform>/build/<package>-<version>
WORKDIR=<workdir> ./build <platform> <package>
```

Wipe rather than delete just the empty objects: a kill mid-write can also leave *partially*
written objects that are non-zero and pass a size check. Wiping costs only a compile — the
download tarball and the toolchain are untouched, so Buildroot re-extracts from cache rather
than re-downloading.

To check before rebuilding, rather than wiping unconditionally:

```bash
find <workdir>/<platform>/build/<package>-<version> -name '*.o' -size 0
```

**Verifying the download cache is not enough.** A separate, better-known failure leaves a
corrupt tarball in `<workdir>/download/` when a build is interrupted mid-download. Proving
that tarball sound (`gzip -t`, checking its contents) says nothing about the build tree —
the two failures are independent and an interruption can cause either or both.

**Never treat exit status 0 as acceptance for a cross-build.** You cannot run the binary on
the host, so the exit code is nearly all you get for free — and as above, it lies. Check the
artifact against the last known-good build:

- `.so` size in the expected band (a collapse of one or two orders of magnitude means code
  is missing)
- `readelf --dyn-syms` for unexpected `UND` symbols
- `readelf -d` for an unexpected `NEEDED` entry — a library that exists on your build host
  but not on the device
- max GLIBC symbol version at or below the target device's glibc:
  `readelf --dyn-syms -W <so> | grep -o 'GLIBC_[0-9.]*' | sort -uV | tail -1`

**Dangling `.lv2` symlinks → `cp: cannot stat` at install time.** Most third-party recipes
keep their modgui in the separate `mod-lv2-data` repository and reach it through symlinks in
the package directory (`plugins/package/<pkg>/<bundle>.lv2 -> ../../../lv2-data/...`), copied
over the built bundle with `cp -rL` in `<PKG>_INSTALL_TARGET_CMDS`. Those symlinks resolve
only if the builder's `lv2-data` and `lv2-data-creative-commons` submodules are checked out.
On a fresh clone they are not: counted 2026-09-16, **468 of the 476 symlinks under
`plugins/package/` were dangling**, which breaks the install step of every recipe that uses
the overlay (55 of the 103 recipes behind the stable catalog). Recipes whose modgui is
in-repo (`neural-amp-modeler-lv2`, `mod-nam-loader`) are unaffected, which is why a NAM build
can succeed on a host where bolliedelay's cannot. Fix once per clone:

```bash
git -C mod-plugin-builder submodule update --init      # ~650 MB
find mod-plugin-builder/plugins/package -maxdepth 2 -type l ! -exec test -e {} \; -print | wc -l   # expect 0-4
```

The four that remain dangling on this host are absolute paths into a deleted
`mod-plugin-commercial` checkout from the untracked `lead-trilogy`/`plexi-breed`/`rocker83`/
`the-dude` packages — not yours to fix.

**GitHub refuses anonymous clones under parallel load.** Fetching several packages at once over
HTTPS produces `fatal: could not read Username for 'https://github.com'` and `fatal: expected
flush after ref listing` — GitHub rate-limiting unauthenticated `git` traffic, not a missing
repository (a genuinely missing repo says `Repository not found`). Seen 2026-09-16: 22 of 103
clones failed this way, all succeeded serially over SSH. Buildroot's `$(call github,...)`
recipes download archive tarballs and are less exposed, but `SITE_METHOD = git` recipes and
the `DOWNLOAD_WITH_SUBMODULES` hook go through `git`. If you have a GitHub SSH key, route the
traffic through it without touching global config:

```bash
export GIT_TERMINAL_PROMPT=0 GIT_CONFIG_COUNT=1 \
       GIT_CONFIG_KEY_0=url.git@github.com:.insteadOf GIT_CONFIG_VALUE_0=https://github.com/
```

**Rust recipes need `rustup` on the host, and flip its default.** The 16 `dm-*` recipes run
`~/.cargo/bin/rustup default nightly` before `cargo build` and `rustup default stable` after
(`plugins/package/dm-ds1/dm-ds1.mk`), so they need rustup with an undated `nightly` installed
(the Docker image does this, a bare host does not) and must not run concurrently with any other
cargo build on the machine. A host without `~/.cargo` cannot build any of them.

## Building without a toolchain: the Cloud Builder

[builder.mod.audio](http://builder.mod.audio) (public source: `mod-audio/mod-cloud-builder`)
cross-builds for the Duo, Duo X and Dwarf without any local setup, from four inputs:
a FAUST `.dsp`, a Max gen~ export, a Pure Data patch (through hvcc) or a buildroot `.mk`
recipe pointing at your repository (the same format as [RECIPES.md](RECIPES.md)). Behind the
page it runs `mod-plugin-builder` with the `-new` toolchains, so what it produces is what this
documentation describes. Facts verified 2026-09-24 to 2026-10-03 on the live site:

- **It installs straight onto a unit connected over USB.** The page opens a WebSocket to the
  unit at `ws://192.168.51.1/rplsocket`, served by mod-ui since **1.13.3**; an older unit
  cannot receive builds from it. The unit's own version check is the `bin-compat` /
  `platform` JSON it answers with on that socket.
- **Which URL works depends on the browser, and the site sorts that out itself.** Chromium
  browsers (Chrome, Edge, Brave, Opera) refuse local-network connections from a plain
  `http://` page since Chrome 147 (WebSockets; `fetch` since 142), silently: the page's socket
  just closes with code 1006. So they are sent to `https://builder.mod.audio`, where Chrome
  asks once for "access devices on your local network"; click **Allow**. Firefox and Safari
  cannot open `ws://` from an `https://` page at all, so the https site sends them back to
  `http://` (a 302 from nginx keyed on the user agent) and they stay there. If a connection
  still fails in Safari on macOS 15 or later, check the per-app "Local Network" switch in
  System Settings → Privacy & Security. Live since 2026-09-24; before that the http-only site
  simply stopped connecting in current Chrome.
- **A persistent build gets a share page**, `/install/<id>`, which installs the stored bundle
  on a unit from the same browser rules. What it shows is read from your bundle's TTL (name,
  author, brand, category) since 2026-09-28; see [METADATA.md](METADATA.md) § "Verified: what
  the Cloud Builder share page shows". A build that failed for one target still gets a share
  link, with a 0-byte tarball for that target (shown as a generic "custom plugin build").
- **An `.mk` the builder rejects says "Invalid package version".** Seen with a repository
  whose committed `.mk` had been emptied to a stub; the builder reads `<PKG>_VERSION` from the
  pasted file, not from the repository.
- **The Pure Data route is pinned to hvcc v0.14.0**, which has no `[expr]`/`[expr~]`
  (added in v0.15.0). What that pin and a newer hvcc do on the devices, including the
  SIMD rule for `[expr~]`, is in [CAVEATS.md](CAVEATS.md) § "Pure Data (hvcc) plugins on
  MOD devices". The pin is `PURE_DATA_SKELETON_VERSION` in the builder's `webserver/server.py`;
  the DPF and heavylib pins live in `builder/Dockerfile`.
