# Commercial plugins and licensing

> **Coming later.** This chapter is intentionally a placeholder — the approach to documenting
> how a developer obtains `libmodla` is still being settled. The rest of this SDK (building,
> validating, modgui, installing) doesn't depend on it; come back for this one.
<!-- GAP: chapter intentionally deferred — do not fill this in opportunistically, even if
     hands-on plugin work touches licensing, until the libmodla-supply approach is settled. -->

## What this covers

Making a plugin that is copy-protected: it runs as a time-limited trial until a licence issued by
MOD's cloud unlocks it on a specific device. Same binary either way — there is no separate demo
build.

## Verified

**Units only.** Licensed plugins build for MOD Duo, Duo X and Dwarf. **Not MOD Desktop** —
`libmodla` has no x86_64 build, and `BR2_PACKAGE_LIBMODLA` is not set by any generic defconfig.
This must be stated plainly and early, not buried.

**The plugin-side API is four calls** (`plugins/package/commercial-plugin-example/`):

```c
#include <libmodla.h>

mod_license_check(f, PLUGIN_URI);                                  // instantiate()
plugin->run_count = mod_license_run_begin(plugin->run_count, nsamples);   // run()
mod_license_run_silence(plugin->run_count, plugin->output, nsamples, 0);  // run()
return mod_license_interface(uri);                                 // extension_data()
```

**Plus one line in the manifest:**

```turtle
lv2:extensionData <http://moddevices.com/ns/ext/license#interface> .
```

**Plus one line in the recipe:**

```make
MYPLUGIN_DEPENDENCIES = libmodla
```

**Trial length** is defined per target inside the libmodla binary for that architecture —
1,800,000 samples, which at MOD's 48 kHz is **37.5 seconds** of audio, after which the plugin
outputs silence.

**Copyleft licences are incompatible.** libmodla is proprietary, so a GPL plugin cannot link it.
You can still release open-source plugins for MOD; they simply cannot use this mechanism.

## Open — decide before writing

- [ ] **How to describe obtaining libmodla.** Recommendation: describe the mechanism only
      (`DEPENDENCIES = libmodla`, which is what the public `commercial-plugin-example` already
      shows) and do not document the download URL
- [ ] **Beta testing.** Trial mode gives a tester 37.5 s, which is a demo rather than a usable test.
      Real beta testing needs licences issued to tester device UIDs, which only MOD's cloud can do.
      Decide what this chapter tells a developer to do
- [ ] Selling: out of scope here, but the chapter should say where that conversation goes
