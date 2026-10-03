# Installing on a device

Two ways to get a bundle onto a unit: post it to mod-ui's `/sdk/install` endpoint, which registers
it live, or copy it into `/root/.lv2` by hand and restart the two services. Both are below, with
what each does and does not do, verified on real units. Nothing here is specific to a device
model; Duo, Duo X and Dwarf run the same mod-ui and mod-host.

## Reaching the unit

- **USB**: the unit is a network device on the cable. It is `http://192.168.51.1`, or
  `http://moddwarf.local` (Duo and Duo X follow the same `<hostname>.local` pattern) where mDNS
  resolves. SSH is on the same address as `root`, password `mod` on a stock image.
- **Bluetooth** (Duo X built in; Dwarf and Duo with a USB dongle): `http://192.168.50.1` after
  pairing.
- **Wi-Fi**: none built in on any unit. The Dwarf manual's "Web UI access" page is the
  user-facing reference for the dongle cases.
- The Web UI's own address is what `curl` and the `/sdk/*` endpoints use; mod-host's command port
  (`5555`) is loopback-only.

## Verified

The toolchain can push directly:

```
./build <platform> <plugin-package>-publish
```

Or manually, posting a base64 tarball to the device:

```
cd ~/mod-workdir/<platform>/plugins
tar czf - mybundle.lv2 | base64 | curl -F 'package=@-' http://192.168.51.1/sdk/install; echo
```

The `/sdk/install` endpoint is live in current mod-ui (`SDKEffectInstaller`,
`mod/webserver.py:833-860` at `7764e1ff`). It base64-decodes the upload, untars it, and for each
bundle: removes an existing bundle of the same name from mod-host and deletes its directory
(presets stored *inside* that bundle go with it), moves the new one into `/root/.lv2`, and sends
`bundle_add` to mod-host (`install_bundles_in_tmp_dir`, `webserver.py:83-157`; `host.py:2246-2256`).
Then it broadcasts a `rescan` to the browser, so no service restart is needed. A URI that was
removed and not reinstalled is dropped from the favourites and banks are re-saved; pedalboard
files are never edited (`webserver.py:120-136`). The reply is
`{"ok": true, "removed": [...uris], "installed": [...uris]}`, or `ok: false` with an `error`.

`/sdk/update` (`SDKEffectUpdater`, `webserver.py:862-894`) is a different, narrower thing: given a
`bundle` path and a `uri` already loaded, it re-reads the bundle's TTL into mod-ui's lilv world
(`host.py:2279-2290`) and broadcasts the same `rescan`. It writes no files and tells mod-host
nothing, so it refreshes metadata and modgui after you edited a `.ttl` in place over SSH; it does
not pick up a new `.so`. For a new binary use `/sdk/install`.

**It will not replace a plugin that is loaded.** If the current pedalboard holds an instance, the
reply is `{"ok": false, "error": "Plugin is currently in use, cannot remove", "installed": [], "removed": []}`
and nothing changes on the device. Remove the instance (or load another pedalboard) and post again.
A successful update lists the URI under both `removed` and `installed`. Seen 2026-09-17 on a Dwarf
running 1.14.0.3333.

Or copy the bundle over with `scp` and restart mod-ui to pick it up — this is the method
Darkglass document for Anagram, and it works the same way on MOD units, since both run the
same mod-ui/mod-host stack:

```
cd ~/mod-workdir/<platform>/plugins
scp -O -r mybundle.lv2 root@192.168.51.1:/root/.lv2/
ssh root@192.168.51.1 "systemctl restart jack2 mod-ui"
```

`/root/.lv2` is mod-ui's user-plugin directory (`LV2_PLUGIN_DIR`, defaults to `~/.lv2` and
mod-ui runs as root on-device — confirmed in `mod-ui/mod/settings.py`). Unlike the
curl+base64 method above, a plain `scp` doesn't trigger a live rescan, so both services need
restarting: `mod-ui` to rebuild its plugin list, and `jack2` because mod-host — the actual LV2
host process — loads its LV2 world once at startup and is tied to the jack2 service, so it
won't see the new bundle until it restarts too. Confirmed in `mod-ui/mod/webserver.py`:
`SystemCleanup` restarts exactly this pair (`jack2` + `mod-ui`) whenever the plugins directory
changes. The `-O` flag forces scp's legacy protocol — needed because current scp/OpenSSH
defaults to SFTP, which the device's older sshd doesn't support.

**Two corrections to the upstream README, which has this wrong:**

1. It gives `tar czf bundle1.lv2 bundle2.lv2 | base64` — that creates an archive *named*
   `bundle1.lv2` containing `bundle2.lv2`. It needs `tar czf - …` to stream to stdout.
2. `192.168.51.1` is the device's USB-network address and is not explained. Document where the
   address comes from and what to use over Wi-Fi.

<!-- GAP: MOD Desktop install path unverified — where do bundles go there? -->

Verified again 2026-09-16 on a Dwarf running 1.14.0.3333, 25 bundles at once: `scp -O -r` into
`/root/.lv2/` + `systemctl restart jack2 mod-ui`, then `lv2ls` lists every URI, mod-ui's
`/effect/get?uri=` reports the new `version`, and `/effect/add//graph/<name>?uri=` /
`/effect/remove//graph/<name>` work for each. Things learned doing it:

- **Reinstalling over a newer version works without a bump when the Store copy is absent**, which
  it is on a fresh image (`/usr/lib/lv2` had none of these). With a Store copy present, lilv picks
  the bundle with the higher `lv2:minorVersion`/`microVersion`, so bump before testing a fix.
- **There is no `curl` on the device.** Script the mod-ui checks from the host against
  `http://192.168.51.1`, or use `python3 -c 'import urllib.request…'` on the unit (Python 3.4).
- **A Mac copying bundles leaves `._<bundle>.lv2` AppleDouble files in `/root/.lv2`**; lilv logs an
  error for each on every scan. Delete them (`rm -f /root/.lv2/._*`).
- **Talking to mod-host directly** (`127.0.0.1:5555`, `\0`-terminated commands, `resp N` replies):
  it serves one client, and after accepting it blocks until a second client connects to the
  feedback port 5556 — mod-ui holds both, so stop mod-ui first and hold both sockets yourself.
  `add <uri> <n>` returns `resp <n>` on success, `-102` when instantiate fails, and `connect` to a
  port name the plugin does not have returns `-205`. Adding ~24 instances and then issuing
  `connect` with wrong port names made jackd abort on `AssertPort(port_index < fPortMax)` twice;
  not isolated further, so keep batches small and port names right.
- **Testing a plugin in mod-host without touching the user's session**: leave the running mod-host (and mod-ui)
  alone and start a second one as an ordinary JACK client, fed from a file of commands:

  ```sh
  printf 'add <uri> 0\nparam_get 0 <symbol>\npreset_load 0 <preset-uri>\ncpu_load\nremove 0\nquit\n' > /tmp/cmds
  timeout 30 mod-host -n -i -p 5575 -f 5576 < /tmp/cmds
  ```

  `-n` no fork, `-i` interactive, other ports than 5555/5556. Each command answers `resp <code> [value]`.
  `connect system:capture_1 effect_0:<input symbol>` puts real input on it; leave the outputs unconnected and
  nothing reaches the speakers. It is invisible to mod-ui and gone when it quits (check with `jack_lsp`).
  Verified 2026-09-17 on a Dwarf, a Duo and a Duo X. A command sent to the *running* mod-host on 5555 while
  mod-ui is up is never read: it sits in the socket backlog until mod-ui disconnects.
- `Plugins/mod-plugin-management/tools/` has the on-device probes used for this
  (`rt_check`, `xrun_counter`) and the scripts under
  `build-records/plugins/logs/rt-check-2026-09-16/` show the whole sequence.

## To write / verify

- [x] Reaching the unit: see "Reaching the unit"
- [ ] MOD Desktop install path — **not yet verified**. Where do bundles go? `~/.lv2`?
- [x] `/sdk/update`: re-reads TTL of a loaded bundle only (2026-10-03, from source)
- [x] Confirming the plugin loaded: `lv2ls | grep <uri>` on the unit, then
      `curl "http://192.168.51.1/effect/get?uri=<uri>"` from the host (JSON with `version`,
      `bundles`); `/effect/add//graph/x?uri=<uri>` returns `false` when mod-host cannot
      instantiate it (HardwareBypass does that on a Dwarf, by design: it is published for Duo and
      Duo X only, whose bypass relay it drives — see [CAVEATS.md](CAVEATS.md))
- [x] Removing a plugin: the Web UI's plugin-bar "remove" posts a JSON list of bundle paths to
      `/package/uninstall` (`webserver.py:1331-1366`), which must be under `/root/.lv2`; mod-ui
      unloads the bundle from mod-host and `rmtree`s it. By hand: `rm -rf /root/.lv2/<bundle>.lv2`
      over SSH and `systemctl restart jack2 mod-ui`. Either way pedalboards that used the URI keep
      referring to it and show the plugin as missing
- [x] Reinstalling over an existing version: needed only when another copy of the same URI is
      installed (then the higher version wins); see above
