# Building

MOD plugins are cross-compiled with [`mod-plugin-builder`](https://github.com/mod-audio/mod-plugin-builder),
a public, open-source wrapper around [Buildroot](https://buildroot.org/) and crosstool-NG. It
builds one toolchain per device (`./bootstrap.sh`), then builds any plugin described by a
Buildroot package recipe against it (`./build`). The recipe format is the subject of
[RECIPES.md](RECIPES.md); this chapter is about getting the toolchain and running builds.

It runs on Linux. On macOS or Windows the supported path is the Docker container the repo ships
(see "Docker path" below) — there is no native bootstrap for either. A bootstrap takes about
half an hour and 12–15 GB per platform on a modern machine, once; a plugin build after that
takes a minute or two.

If you don't want a local toolchain at all, [builder.mod.audio](http://builder.mod.audio) builds
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
modern multi-core machine (see "Bootstrap time" below):

```
./bootstrap.sh <platform>
```

Output goes to `~/mod-workdir`; override with `WORKDIR`.

Build a plugin:

```
./build <platform> <plugin-package>
./build <platform> <plugin-package>-dirclean     # clean
```

Bundles land in `~/mod-workdir/<platform>/plugins`. Building the bundled example plugins needs
submodules: `git submodule init && git submodule update`.

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

`docker-mount.sh <platform> [extra-mount-path]` is the whole mechanism:

1. If an image tagged `mpbi_<platform>` doesn't exist yet, it builds one from `docker/Dockerfile`.
2. If a container named `mpb_<platform>` already exists, it restarts and attaches to it.
3. Otherwise it creates one, bind-mounting the repo checkout to `/home/builder/mod-plugin-builder`
   and either your `PLUGINS_DIR` or an extra path you pass in, to `/mnt`.

Once attached you're in a normal Linux shell with the toolchain image's dependencies
pre-installed — run `./bootstrap.sh <platform>` and `./build <platform> <plugin>` exactly as on
a native Linux host. This is the real answer for macOS and Windows: no native bootstrap exists
for either, but the container gives you the same Linux environment.

## Disk usage

```
du -sh <workdir>/<platform>
```

| Platform | Size | Notes |
|---|---|---|
| `moddwarf-new` | 15 GB | full bootstrap + built plugins; `build/` (intermediate Buildroot state) alone is 13 GB, `host/` 735 MB, `target/` (device rootfs) 139 MB, toolchain sysroot 289 MB |
| `darkglass-anagram` | 3.9 GB | full bootstrap |
| `modduo-new` | 12 GB | full bootstrap + built plugin |
| `modduox-new` | 12 GB | full bootstrap + built plugin |
| `generic-aarch64` | 7.6 GB | **toolchain only** — see the failure mode below |
| `download/` | 1.4 GB | shared source-tarball cache across all platforms |

`build/` is the largest and most disposable component — the sysroot/host/target directories are
what you need to keep working.

## Bootstrap time

Single, uninterrupted runs, each timed start-to-finish:

| Platform | Bootstrap | Tree size |
|---|---|---|
| `modduo-new` (32-bit ARM Cortex-A7) | **30 min** | 12 GB |
| `modduox-new` (aarch64 Cortex-A53) | **32 min** | 12 GB |

Run sequentially on a 12-core/7 GB-RAM Linux host with the workdir on an external SSD: 1h05m for
both. **RAM is the binding constraint** — running two bootstraps in parallel on a small-memory
machine is slower than running them one after another, not faster.

A single plugin build after that (`./build <platform> <plugin>`) takes **89–93 seconds**, and
`./validate` is effectively instant. The multi-hour cost is the toolchain, once per platform;
everything after it is short.

**Benign noise to expect:** each bootstrap log contains roughly 26 `error:`/`fatal error` lines
from Qt5's configure feature probes for optional features that are absent by design. Judge a
bootstrap by its exit status and by whether `build/<buildroot-version>/` exists, not by grepping
the log for "error".

## Alternate build path: some repos vendor their own cross-compile script

Not every MOD plugin repo builds through `./build <platform> <plugin>`'s recipe mechanism. A
few vendor their own script that cross-compiles directly with CMake against a
`mod-plugin-builder`-bootstrapped toolchain's output directories (`<workdir>/<platform>/host`,
`.../staging`, `.../target`, and the toolchain file under
`.../host/usr/share/buildroot/toolchainfile.cmake`) rather than going through `./build`. Both
paths consume the same bootstrapped toolchain tree — they're just two different front ends to
it. **Check which one a given repo uses before assuming `./build <platform> <plugin>` is the
entry point** — grep the repo root for a custom build script first.

## Common bootstrap failures

**Toolchain finished, Buildroot stage never ran.** `bootstrap.sh` builds the ct-ng toolchain
first, then downloads/extracts Buildroot (`bootstrap.sh:89-199`). If interrupted between those
stages, `<workdir>/<platform>/toolchain/` is fully populated but
`<workdir>/<platform>/build/<buildroot-version>/` doesn't exist yet. Symptom, from `./build`:

```
./build: line 121: cd: <workdir>/<platform>/build/<buildroot-version>: No such file or directory
```

**Fix: just re-run `./bootstrap.sh <platform>`.** The ct-ng stage is checkpointed with marker
files (`bootstrap.sh:93-152`) and the Buildroot stage is guarded by a directory-existence check
(`bootstrap.sh:156`) — re-running skips the completed toolchain entirely and resumes where it
stopped. No need to delete anything first.

## Building a local working tree without pushing it

The builder fetches every package from its recipe's `_SITE`/`_VERSION`, so testing a fix
normally means pushing first. Buildroot's source override avoids that: put

```
MOD_UTILITIES_OVERRIDE_SRCDIR = /home/you/Repos/mod-utilities
```

(the `_VERSION` variable's prefix, i.e. the package name upper-cased with `-` → `_`) into
`<WORKDIR>/<platform>/local.mk` — the file already exists, empty, in every bootstrapped tree —
and run `./build <platform> <package>-rebuild`. The `-rebuild` matters: for an override package
it re-rsyncs the tree into `build/<package>-custom/` before building; a plain `./build` after
the first one reuses the stale copy. Three things to know:

- **Recipe patches are not applied** to an override tree (Buildroot skips the patch step), so
  fold them into the tree first — which is what you want when the goal is to retire them.
- **An override left in `local.mk` silently replaces the pin for every later build of that
  package on the host.** Empty the file when done.
- The pinned-fetch path is still untested until the branch is pushed and `./build` runs once
  without the override. Treat it as unverified until then.

The override rsync excludes `.git`, so a submodule checkout comes along as plain files — fine
for building.

**A new plugin added to a multi-bundle package builds with the recipe unchanged, but only as
far as `target/`.** Adding a new sub-directory to a multi-plugin repo's top-level Makefile gets
it compiled and installed into `<WORKDIR>/<platform>/target/usr/lib/lv2/` by the untouched
recipe, because the recipe just runs the repo's `make`/`make install`. It is **not** copied to
`<WORKDIR>/<platform>/plugins/`: `./build` copies exactly the bundles named in the recipe's
`<PKG>_BUNDLES` line (`build:93`), and `./validate`/`./publish` read the same line. Its
"possibly missing from `_BUNDLES`" warning only prints when `plugins/` was empty beforehand, so
on a tree that's already been used nothing warns you. Take a trial bundle from
`target/usr/lib/lv2/` and add it to `_BUNDLES` before pinning the branch.

## Common plugin-build failures

**An interrupted `./build` is NOT safe to just re-run.** This is the opposite of the bootstrap
advice above: `bootstrap.sh` is checkpointed, so re-running resumes correctly. A plugin build is
a parallel `make` — re-running it can silently produce a broken plugin that exits 0.

If a `make -j<N>` is killed partway, every object file open for writing at that instant is left
**zero bytes**. On a re-run, `make` compares mtimes, sees each zero-byte `.o` as *newer* than
its source, skips recompiling it, and links the empty objects. The link succeeds. The build
reports success. One real case: a reset mid-compile left several objects at zero bytes, and the
resumed build produced a plugin binary roughly 50× smaller than a sound one, with an entire
subsystem missing — `readelf --dyn-syms -W <so> | awk '$7=="UND"'` showed an undefined symbol
nothing else in the link provides, so the plugin installed and then failed to instantiate on
the device — a failure that only shows up on real hardware.

**Fix: wipe the package build directory and rebuild.**

```bash
rm -rf <workdir>/<platform>/build/<package>-<version>
WORKDIR=<workdir> ./build <platform> <package>
```

Wipe rather than delete just the empty objects: a kill mid-write can also leave *partially*
written objects that are non-zero and pass a size check. Wiping costs only a compile — the
download tarball and the toolchain are untouched.

To check before rebuilding, rather than wiping unconditionally:

```bash
find <workdir>/<platform>/build/<package>-<version> -name '*.o' -size 0
```

**Never treat exit status 0 as acceptance for a cross-build.** You cannot run the binary on the
host, so the exit code is nearly all you get for free — and as above, it lies. Check the
artifact against the last known-good build:

- `.so` size in the expected band (an order-of-magnitude drop means code is missing)
- `readelf --dyn-syms` for unexpected `UND` symbols
- `readelf -d` for an unexpected `NEEDED` entry — a library on your build host but not the device
- max GLIBC symbol version at or below the target device's glibc:
  `readelf --dyn-syms -W <so> | grep -o 'GLIBC_[0-9.]*' | sort -uV | tail -1`

**Dangling `.lv2` symlinks → `cp: cannot stat` at install time.** Most third-party recipes keep
their modgui in a separate repository and reach it through symlinks in the package directory,
copied over the built bundle at install. Those symlinks only resolve if the builder's
`lv2-data`/`lv2-data-creative-commons` submodules are checked out — on a fresh clone they
aren't, which breaks the install step of any recipe using the overlay. Fix once per clone:

```bash
git -C mod-plugin-builder submodule update --init      # ~650 MB
find mod-plugin-builder/plugins/package -maxdepth 2 -type l ! -exec test -e {} \; -print | wc -l   # expect 0
```

Recipes whose modgui lives in-repo rather than through the overlay (`mod-nam-loader`, for
example) are unaffected either way.

**GitHub rate-limits anonymous clones under parallel load.** Fetching several packages at once
over HTTPS can produce `fatal: could not read Username for 'https://github.com'` or `fatal:
expected flush after ref listing` — rate-limiting of unauthenticated `git` traffic, not a
missing repository (a genuinely missing repo says `Repository not found`). Fetching serially,
or over SSH if you have a GitHub key, avoids it:

```bash
export GIT_TERMINAL_PROMPT=0 GIT_CONFIG_COUNT=1 \
       GIT_CONFIG_KEY_0=url.git@github.com:.insteadOf GIT_CONFIG_VALUE_0=https://github.com/
```

**Rust recipes need `rustup` on the host, and flip its default.** Rust-based recipes run
`rustup default nightly` before building and `rustup default stable` after, so they need
`rustup` installed with an undated `nightly` toolchain available, and must not run concurrently
with any other cargo build on the same machine. A host without `~/.cargo` cannot build them.

## Building without a toolchain: the Cloud Builder

[builder.mod.audio](http://builder.mod.audio) (public source: `mod-audio/mod-cloud-builder`)
cross-builds for the Duo, Duo X and Dwarf without any local setup, from four inputs: a FAUST
`.dsp`, a Max gen~ export, a Pure Data patch (through hvcc), or a Buildroot `.mk` recipe
pointing at your repository (the same format as [RECIPES.md](RECIPES.md)). Behind the page it
runs `mod-plugin-builder` with the `-new` toolchains, so what it produces is what this
documentation describes.

- **It installs straight onto a unit connected over USB.** The page opens a WebSocket to the
  unit at `ws://192.168.51.1/rplsocket`, served by mod-ui since **1.13.3** — an older unit
  can't receive builds from it.
- **Which URL works depends on the browser, and the site sorts that out itself.** Chromium
  browsers refuse local-network connections from a plain `http://` page (silently — the socket
  just closes), so they're sent to `https://builder.mod.audio`, where Chrome asks once for
  local-network device access; allow it. Firefox and Safari can't open `ws://` from an
  `https://` page at all, so the https site redirects them back to `http://`, and they stay
  there. On macOS, a stuck connection in Safari is worth checking against the per-app "Local
  Network" switch in System Settings → Privacy & Security.
- **A persistent build gets a share page**, `/install/<id>`, which installs the stored bundle on
  a unit from the same browser rules. What it shows is read from your bundle's TTL (name,
  author, brand, category) — see [METADATA.md](METADATA.md). A build that failed for one target
  still gets a share link, with a 0-byte tarball for that target.
- **An `.mk` the builder rejects says "Invalid package version."** This usually means the
  pasted file's `<PKG>_VERSION` doesn't resolve — the builder reads it from the pasted file, not
  from the repository.
- **The Pure Data route is pinned to hvcc v0.14.0**, which has no `[expr]`/`[expr~]` (added in
  v0.15.0). What that pin and a newer hvcc do on the devices is in [CAVEATS.md](CAVEATS.md)
  § "Pure Data (hvcc) plugins on MOD devices".
