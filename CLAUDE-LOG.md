# CLAUDE-LOG

Contributor log for this repo (see `mod-publishing-ops/CLAUDE.md`, "Contributor log"). Append, don't rewrite.

## 2026-10-16 — João (joao@mod.audio), MacBook — INSTALLING: USB audio skews DSP; `/sdk/install` file field

- `INSTALLING.md`: USB audio interface mode (`jack-usbgadget-*`) costs ~78% DSP on a Duo and ~50% on a
  Duo X, so stop it before measuring a plugin; the websocket `stats` message. Also that `package` must be a
  file field (`package=<file` gives a bare 500). Found while testing MIRRORZz on João's devices.
