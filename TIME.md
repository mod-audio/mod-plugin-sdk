# Tempo and transport

Read from `mod-host/src/effects.c` at `f14a230` (master, what ships), 2026-10-03. Not yet
exercised by a plugin built for this documentation; the citations are the evidence.

## Where tempo comes from

mod-host owns a global tempo: `g_transport_bpm` (default 120, accepted range 20–280) and
`g_transport_bpb` (default 4, range 1–16) (`effects.c:3993-3995,6864,6883`). It registers as the
**JACK timebase master**, conditionally, so it does not displace another master but reclaims the
role if that master stops providing bar/beat/tick (`effects.c:4095,8082-8085`), and fills JACK's
BBT from those globals with `beat_type = 4` and 1920 ticks per beat (`JackTimebase`,
`effects.c:2807-2900`). The values can be set by:

- mod-ui, through the `set_bpm`, `set_bpb`, `transport <rolling> <bpb> <bpm>` and
  `transport_sync <none|link|midi>` commands (`mod-host.h:68-69,92-93`);
- the user, through the global pseudo-ports `:bpm`, `:bpb` and `:rolling`, which can be addressed
  to a knob, a MIDI CC or a Control Chain actuator like any plugin control (`effects.c:183-192,
  2388-2443,3735`);
- **Ableton Link**, when sync is `link` (`effects.c:2511-2533`);
- **MIDI clock** on the device's MIDI input, when sync is `midi`: 0xF8 ticks are filtered into a
  tempo, Start/Continue roll the transport, Stop stops it and locates to 0; Song Position
  Pointer is not handled (`effects.c:2546-2602`).

**A plugin cannot be the tempo source.** Nothing a plugin outputs, control, atom or a
`time:Position` it forges, reaches `g_transport_bpm` (its only writers are the paths above). A tap
tempo plugin works by being addressed to the `:bpm` pseudo-port from the UI side, not by
publishing a tempo. `mod:tapTempo` on a control port tells mod-ui to offer the "tap" gesture for
that port; it is a UI hint, not a transport link.

## How a plugin receives it

Two mechanisms, both driven every cycle, and you may use either or both:

**1. Designated control ports.** Declare a control input with `lv2:designation
time:beatsPerMinute`, `time:beatsPerBar` or `time:speed`. mod-host resolves exactly these three
(`effects.c:4166-4169,5125-5155`) and overwrites them **unconditionally every cycle** from the
globals (`effects.c:1814-1825`; speed is 1.0 while rolling, 0.0 while stopped), and again after a
preset load (`effects.c:5489-5500`). The port is never user-controllable: its value is the
transport's. This is the simplest way to build a tempo-synced delay: read `beatsPerMinute` at
the top of `run()`, convert the note division to samples, smooth the change.

**2. A `time:Position` atom.** An atom input port that declares `atom:supports time:Position`
gets a `time:Position` object at frame 0, before any MIDI events (`effects.c:5039-5042,
1771-1811,1869-1871`). The designation of the port is irrelevant; the `atom:supports` triple is
what mod-host checks. Keys written: `time:speed` (float, 1 or 0), `time:frame` (long), and,
when JACK's BBT is valid (it is whenever mod-host is timebase master), `time:bar` (long,
zero-based), `time:barBeat` (float), `time:beat` (double), `time:beatUnit` (int, 4),
`time:beatsPerBar` (float), `time:beatsPerMinute` (float), `time:ticksPerBeat` (double, 1920).
It is forged only when something changed: rolling state, frame, tempo or signature, or the
plugin coming out of bypass (`effects.c:1760-1764`). While the transport rolls the frame changes
every cycle, so you get one every cycle; while it is stopped you get one on each change only,
so keep the last one. **It is not sent while the plugin is hard-bypassed** (the bypass branch
skips it, `effects.c:1863`); a plugin that handles its own bypass through `lv2:enabled` keeps
receiving it.

Use the atom when you need position (bar, beat, frame) for something that must stay in phase
with the transport, a step sequencer or a synced LFO; use the control ports when tempo alone is
enough. Both agree, as they come from the same globals.

## What the transport does, and does not, do

- "Rolling" is JACK's transport. The device's play/stop in the Web UI and the `:rolling`
  pseudo-port start and stop it; stopping also relocates to frame 0 (`effects.c:8064-8127`).
- Audio keeps flowing regardless. Transport state is information for your plugin, nothing is
  muted or paused by the host.
- There is no song structure beyond bar and beat, no loop points, no SPP.
- Tempo changes are not ramped. The BPM port jumps; smooth it yourself if a delay time follows it.
- `mod:tempoRelatedDynamicScalePoints` on a control port marks its scale points as note
  divisions whose displayed value depends on the tempo (the UI shows "1/8 = 250 ms" style
  labels). It changes display only; the value your port receives is the scale-point value you
  declared (mod-ui: `html/js/utils/tempo.js:270-275`, `html/js/hardware.js:876` at `7764e1ff`).
  `mod:tapTempo` is honoured by the addressing code (`hardware.js:181,388`,
  `mod/addressings.py:841-986`). `mod:rawMIDIClockAccess` on a MIDI input port asks for the clock
  messages to be left in the MIDI stream delivered to that port, for plugins that do their own
  MIDI-clock handling; declared in `mod.lv2/mod.ttl`, not traced through the host for this page.

## To write / verify

- [ ] Worked example of a tempo-synced delay, built and heard on a device
- [ ] `mod:rawMIDIClockAccess`: trace what mod-host does with it
