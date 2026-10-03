# Plugin Recipes (`.mk` packages)

The unit MOD builds is not a source tree, it is a **Buildroot package**: a `.mk` file describing
where the source is and how to build it. This is the piece with no equivalent in other plugin
platforms, and the piece newcomers get stuck on.

Why a recipe at all: `mod-plugin-builder` does not know how to build *your* project, it knows how
to drive Buildroot, and Buildroot builds packages. A recipe tells it where the source comes from
(`_SITE` and `_VERSION`: a git URL and a commit, or a local directory), what it depends on
(`_DEPENDENCIES`: other packages in the builder, such as `libmodla` or DPF), how to build and
install it (`_BUILD_CMDS` / `_INSTALL_TARGET_CMDS`, or just `$(eval $(cmake-package))` for CMake),
and which `.lv2` bundles come out (`_BUNDLES`, which `./build`, `./validate` and `./publish` all
read — a bundle not listed there is built but never copied out, see
[BUILDING.md](BUILDING.md)). Patches dropped next to the `.mk` are applied automatically. The
recipe lives at `plugins/package/<name>/<name>.mk` inside a checkout of `mod-plugin-builder`;
the source can live anywhere. The same recipe is what MOD's own publishing pipeline and
[builder.mod.audio](http://builder.mod.audio)'s "buildroot" page consume, so writing one is not
a local convenience: it is the unit of publication.

## A minimal recipe

A complete minimal recipe (`plugins/package/commercial-plugin-example/`):

```make
COMMERCIAL_PLUGIN_EXAMPLE_SITE_METHOD = local
COMMERCIAL_PLUGIN_EXAMPLE_SITE = $($(PKG)_PKGDIR)/
COMMERCIAL_PLUGIN_EXAMPLE_VERSION = 1
COMMERCIAL_PLUGIN_EXAMPLE_DEPENDENCIES = libmodla
COMMERCIAL_PLUGIN_EXAMPLE_BUNDLES = commercial-plugin-example.lv2
COMMERCIAL_PLUGIN_EXAMPLE_TARGET_MAKE = $(TARGET_MAKE_ENV) $(TARGET_CONFIGURE_OPTS) $(MAKE) -C $(@D)/source

define COMMERCIAL_PLUGIN_EXAMPLE_BUILD_CMDS
	$(COMMERCIAL_PLUGIN_EXAMPLE_TARGET_MAKE)
endef

define COMMERCIAL_PLUGIN_EXAMPLE_INSTALL_TARGET_CMDS
	$(COMMERCIAL_PLUGIN_EXAMPLE_TARGET_MAKE) install DESTDIR=$(TARGET_DIR)
endef

$(eval $(generic-package))
```

Notes worth making explicit:
- The `PACKAGE_NAME_` prefix must match the directory name, uppercased with `-` → `_`
- `VERSION` must be set even for local builds; bumping it forces a rebuild
- `BUNDLES` lists the `.lv2` directories produced

## Verified: remote git source, with submodules

A CMake plugin pulled from GitHub, pinned to a commit, whose dependencies are git submodules
(verified 2026-08-24 by building `mod-nam-loader` for `moddwarf-new`):

```make
MOD_NAM_LOADER_VERSION = 92e9a9ab49ee58fa9534a127fbe0282e19879ff8
MOD_NAM_LOADER_SITE = https://github.com/mod-audio/mod-nam-loader.git
MOD_NAM_LOADER_SITE_METHOD = git
MOD_NAM_LOADER_GIT_SUBMODULES = y
MOD_NAM_LOADER_BUNDLES = mod-nam-loader.lv2

MOD_NAM_LOADER_CONF_OPTS += -DCMAKE_BUILD_TYPE=Release

# required for submodules to actually be fetched
MOD_NAM_LOADER_PRE_DOWNLOAD_HOOKS += MOD_PLUGIN_BUILDER_DOWNLOAD_WITH_SUBMODULES

define MOD_NAM_LOADER_INSTALL_TARGET_CMDS
	install -d $(TARGET_DIR)/usr/lib/lv2/$(MOD_NAM_LOADER_BUNDLES)
	install -m 644 $(@D)/$(MOD_NAM_LOADER_BUNDLES)/*.ttl $(TARGET_DIR)/usr/lib/lv2/$(MOD_NAM_LOADER_BUNDLES)/
	install -m 755 $(@D)/$(MOD_NAM_LOADER_BUNDLES)/*.so $(TARGET_DIR)/usr/lib/lv2/$(MOD_NAM_LOADER_BUNDLES)/
endef

$(eval $(cmake-package))
```

Things that are not obvious:

- **`_VERSION` is a full commit hash**, and it is what the download cache is keyed on. Bump it to
  rebuild from new source.
- **`_GIT_SUBMODULES = y` on its own is not enough.** Without the
  `MOD_PLUGIN_BUILDER_DOWNLOAD_WITH_SUBMODULES` pre-download hook, submodules are not fetched and
  your build fails on missing headers. The hook is defined in
  `plugins-dep/package/mod-plugin-builder/mod-plugin-builder.mk`.
- **A failed download poisons the cache for that commit.** The hook chains its steps with `;`
  rather than `&&`, so when the clone fails it still runs `tar` and caches an archive of nothing —
  20 bytes in a case seen on 2026-08-24. It then guards on whether the tarball exists, so every
  later build of that hash skips cloning and fails extracting instead:
  `tar: Exiting with failure status due to previous errors`, then
  `Error 2` on `.stamp_extracted`. **Symptom: a build that fails identically no matter what you
  change, with a different error than the one that actually broke.** Recover by deleting
  `<workdir>/download/<pkg>-<hash>.tar.gz` and `<workdir>/<platform>/build/<pkg>-<hash>/`. Check the
  tarball's size before believing a cached download — a real one for a small plugin with submodules
  is megabytes.
- **A private repository cannot be fetched over HTTPS.** `git clone https://github.com/...` on a
  private repo fails with `fatal: could not read Username for 'https://github.com': No such device
  or address`, which reads like a network fault and is not one. An anonymous clone of a *public*
  repo needs no username, so that message means the repository is private. Either make it public,
  or use the `git@github.com:` form and accept that only someone with an authorised key can build
  it — that rules out CI without a deploy key. Note the failed clone will also poison the download
  cache per the point above, so clean it before retrying.
- **CMake builds in-source**, so `$(@D)` is the CMake binary directory and your bundle is at
  `$(@D)/<bundle>.lv2` — that is why the install commands above look the way they do.
- `$(eval $(cmake-package))` for CMake, `$(eval $(generic-package))` for a plain Makefile.

### Do not ship a local-path `_SITE`

Pointing `_SITE` at a working copy is the fastest way to iterate:

```make
MOD_NAM_LOADER_SITE = /home/you/Repos/mod-nam-loader
MOD_NAM_LOADER_SITE_METHOD = git
```

but a local path plus an unchanged `_VERSION` produces **binaries that cannot be told apart from a
released build** — the plugin reports the same version whatever your working copy contained. Keep it
for local work, and restore the public URL before the recipe leaves your machine.

## Patch conventions

Drop patch files straight into your package directory — `plugins/package/<name>/NN_description.patch`,
two-digit, zero-padded, sequential (`01_...`, `02_...`). Buildroot's generic package infrastructure
applies every `*.patch` file it finds there in filename-sort order automatically; nothing in the
`.mk` needs to reference them. Real examples in this tree: `fomp/01_add-mod-brand-and-label.patch`,
`bolliedelay/{01_send-tempo-to-host,02_fix-mac-win-install}.patch`,
`neural-amp-modeler-lv2/{04_fix-major-minor-macro-conflicts_Version3,06_fix-math-approx-cmake-version}.patch`
(gaps in the sequence, like the missing 05, are normal — a later fix superseded and deleted it; see
[CAVEATS.md](CAVEATS.md)). The occasional four-digit, hyphenated exception (`invada-lv2/0003-pass_flags.patch`)
is a vendored upstream patch series kept in its original numbering, not a second MOD convention.

## `Config.in` — you almost certainly don't need one

`Config.in` only exists under `plugins-dep/package*/` and `global-packages/`, and only for
Buildroot *toolchain dependencies* (JUCE, DPF, carla-backend, libmodla, etc.) that must appear
in Buildroot's menuconfig/defconfig system. **No plugin recipe under `plugins/package/` has a
`Config.in`, and none needs one.** Individual plugin packages are discovered by a plain path
glob, and `./build <platform> <plugin>` invokes that package's Buildroot target directly by
name, with no menuconfig selection step in between.

<details>
<summary>Verification</summary>

`plugins-dep/Config.in` sources each toolchain dependency explicitly. Plugin discovery glob:
`plugins-dep/external.mk:5`: `include $(sort $(wildcard $(BR2_EXTERNAL_PLUGINS_DEP)/../plugins/package/*/*.mk))`.
Direct target invocation: `build:79-96`.

</details>

## Pointing the builder at a recipe outside its own tree

No — not with a documented hook. Your recipe has to live at `plugins/package/<name>/` inside a
checkout of this repo. Your plugin's actual *source* can live anywhere — `_SITE` accepts any git
URL or local path (see "Do not ship a local-path `_SITE`" above) — it's specifically the `.mk`
recipe that's pinned to this repo's own directory layout.

<details>
<summary>Verification</summary>

`BR2_EXTERNAL_PLUGINS_DEP` is set by `.common:155` to `${SOURCE_DIR}/plugins-dep`, where
`SOURCE_DIR` (`.common:154`) is simply `$(pwd)` — the mod-plugin-builder checkout you're
standing in. The discovery glob above resolves relative to that same path
(`.../plugins-dep/../plugins/package/`).

</details>
