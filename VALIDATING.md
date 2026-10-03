# Validating

`./validate` is the builder's pre-flight check for a built bundle. It is underused, and it is the
cheapest way to catch the class of problem that otherwise shows up as a plugin that installs and
then never appears on the device. Everything below was verified by reading
`mod-plugin-builder/validate` and running it against real bundles.

## Running it

```
./validate <platform> <plugin-package>
VALGRIND=1 ./validate <platform> <plugin-package>
```

Host tools required:

```
sudo apt install lilv-utils sordi
```

If they are missing, `./validate` fails with exit code **127** and a bare
`lv2_validate: exec: sord_validate: not found` — it does not tell you what to install, and it is
easy to mistake for a problem with your bundle.

The plugin must already have been built successfully.

## What it actually does

Verified 2026-08-24 by reading `mod-plugin-builder/validate` and running it against a real bundle.
Four stages:

1. **Recipe sanity.** The package directory and `.mk` must exist, and the `.mk` must declare
   `<PKGNAME>_BUNDLES`. A recipe that produces bundles without declaring them fails here.
2. **Bundle presence.** Each declared bundle must exist under `<workdir>/<platform>/plugins`.
3. **TTL validation.** Bundles are copied to a sandbox (`/tmp/mpb-plugin-check`), `LV2_PATH` is
   pointed at it, and `lv2_validate` — a `sord_validate` wrapper — checks **every `*.ttl` in the
   bundle root** against the LV2 ontologies plus MOD's own: `mod.lv2`, `modgui.lv2`, `mod-hmi.lv2`,
   `mod-license.lv2` and the `kx-*` extensions. Then `lv2ls` lists what lilv can actually load.
4. **Runtime test, x86_64 only.** If `<workdir>/<platform>/target/usr/lib/carla/carla-bridge-native`
   exists, each plugin is instantiated and run through it in dummy mode
   (`CARLA_BRIDGE_DUMMY=1`), optionally under Valgrind with `--leak-check=full
   --track-origins=yes`. The bridge is launched through `ld-linux-x86-64.so.2`, which is why this
   stage exists only on x86_64. **Carla is not part of the MOD platform**: the devices and MOD
   Desktop run mod-host and nothing else. The bridge is used here only because the builder
   already compiles `carla-backend` as a desktop toolchain dependency, so a disposable LV2 host
   is at hand for a load-and-run smoke test. You never need to install or know Carla.

**So a pass on a device target is a TTL pass and nothing more.** Stage 4 is skipped silently — no
message says so. If you want your plugin actually instantiated and run, validate a
`generic-x86_64` build as well; that is what the target is for.

## What a pass looks like

```
Checking mod-nam-loader.lv2...
Skipping file /mnt/mod-workdir/moddwarf-new/target/usr/lib/lv2/core.lv2/lv2core.doap.ttl
Found 0 errors among 112 files (checked 2152 restrictions)
Found 1 plugin(s)
```

- The `Skipping file … lv2core.doap.ttl` line is normal and not about your plugin.
- `Found 0 errors` is the pass. Anything else names the file and the restriction it broke.
- `Found N plugin(s)` is the check people forget to read: it is `lv2ls` reporting what **lilv** could
  load from your bundle. **`0 plugin(s)` with 0 errors means your Turtle is valid but your manifest
  does not describe a loadable plugin** — a wrong `lv2:binary`, a `rdfs:seeAlso` pointing at a file
  that is not there, a URI mismatch between `manifest.ttl` and the plugin TTL. That is exactly the
  failure mode where a plugin "validates fine" and then never appears on the device.
- On x86_64 you additionally get a `Verifying <uri>... ok` line per plugin from stage 4.

## Two things not to rely on

- **The "bundle has binaries" check does nothing.** It runs `find <bundle> -name "*.so"`, and `find`
  exits 0 whether or not it matches anything, so a bundle with no shared object passes. Check
  yourself.
- **Only `*.ttl` in the bundle root is validated.** Turtle in a subdirectory — a `modgui/` folder,
  for instance — is not globbed and is never seen. A modgui TTL at the bundle root *is* covered, and
  the `modgui.lv2` ontology is loaded, so `modgui:` properties are genuinely checked.

## Use it as a gate

It costs seconds and catches the class of mistake that is most expensive to find on hardware: a
plugin that installs and then silently does not appear. Run it on the device target you ship and on
`generic-x86_64` for the runtime test, before every publish.

<!-- GAP: what the Carla bridge runtime test (stage 4) actually asserts — instantiate/run/cleanup
     only, or more — is unconfirmed; whether stage-4 Valgrind findings are usually actionable or
     mostly noise on LV2 hosts is unconfirmed -->

## To write / verify

- [ ] What the Carla bridge runtime test actually asserts — instantiate/run/cleanup only, or more
- [ ] Whether Valgrind findings from stage 4 are usually actionable or noisy on LV2 hosts
