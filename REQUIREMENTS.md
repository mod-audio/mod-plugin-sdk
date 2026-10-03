# Requirements — how MOD expects a plugin to behave

This chapter is about behaviour, not API: what the host does to your plugin, what it will not
do for you, and what the devices can afford. Everything in it was either measured on a device
or read from mod-host source, cited in each section's verification block. The short version:

- `run()` must not allocate, lock, or do I/O. Measure that with a counter, not a code review.
- The budget is one block period shared with the whole pedalboard, and the Dwarf and Duo are
  far slower than a laptop; FFT-heavy code scales worst.
- Denormals are flushed to zero by the host on every audio thread; you do not have to.
- The host does hard bypass for you unless you declare an `lv2:enabled` port; then it is yours.
- Latency reporting does not exist. A plugin's latency is invisible to the host and the user.
- Block size comes from the options feature and can change at runtime; never bake it in.
- Port symbols and the plugin URI are permanent once a pedalboard has used them.
- There is exactly one extra thread you get for free, the worker; use it for everything slow.

## Real-time safety

**In `run()`: no allocation, no free, no lock, no file or network I/O, no logging that
allocates.** That includes things that hide behind ordinary C++ — `std::vector` growth,
`std::string` assignment past capacity, `std::function` construction, any container touched
for the first time. Anything unbounded belongs on the worker thread.

**The budget is one block period, shared with every plugin in the chain** — `block_size /
48000` seconds, 2.67 ms at 128 frames. It's not per plugin: a user with four plugins after
yours will xrun on your overage, and it looks like their fault.

**Measure it; don't reason about it.** Interpose `malloc`/`free`/`pthread_mutex_lock` in a test
harness, flag the audio thread, and count. Cross-compile the harness with the platform toolchain
— that's the only way to get honest numbers for an A35.

Two lessons worth passing on:

- **The harness catches itself first.** Its own queue and buffers will show up as allocations
  and locks on first run. Exclude the harness's own code before trusting its output.
- **A clean `run()` can still xrun.** The worker competes with audio for CPU and memory
  bandwidth on a 4-core embedded part, so loading a model or sample can stall audio even
  though `run()` itself measures clean. Keep parsed data resident in RAM rather than
  re-reading it, and stop glibc returning freed memory to the kernel
  (`mallopt(M_TRIM_THRESHOLD, -1)`, `mallopt(M_MMAP_MAX, 0)`) — unmapping pages forces TLB
  shootdowns that stall the core running audio. MOD devices have RAM to spare; spend it.

<!-- GAP: whether there's an enforced or advisory CPU limit per pedalboard is unconfirmed; what a
     developer actually sees when a plugin xruns (mod-xrun) is undocumented -->

<details>
<summary>Verification</summary>

A worked harness is `mod-nam-loader/tools/rt_probe.cpp`: paces the callback at the block
deadline, runs the worker separately, reports block-time percentiles plus allocation/lock
counts.

</details>

## Denormals are flushed by the host

You don't need to set flush-to-zero yourself, and setting it does no harm. mod-host installs a
JACK thread-init callback that sets FZ in `FPCR` on AArch64, the matching bit of `FPSCR` on the
32-bit Duo, and both FTZ and DAZ in `MXCSR` on x86, on every real-time thread JACK creates — so
every thread that calls your `run()` has denormals off before the first cycle. **The worker
thread is not a JACK thread and does not get this** — set it yourself if you do heavy float work
there (IR pre-computation, model warm-up).

Your own build's `-ffast-math`/`crtfastmath` only sets the bit on the thread that loads the
bundle, and `FPCR` is per-thread — don't rely on it outside `mod-host`. On another host or a
desktop, use the usual DC-offset trick in feedback paths or set FZ in `run()` yourself.

<details>
<summary>Verification</summary>

AArch64/Duo FZ init: `mod-host/src/effects.c:4094,5410`. x86 FTZ/DAZ: `effects.c:2912-2926`.
Read at `f14a230` (master, what ships).

</details>

## Latency is not reported

`mod-host` has no latency support: it never resolves an `lv2:latency` designation, never reads
`lv2:reportsLatency`, never calls `jack_port_set_latency_range`. A latency output port is just
an ordinary control output — nobody compensates, the user can't see it. Consequences:

- Keep latency low; nothing hides it. A look-ahead limiter or spectral effect adds its frames
  to everything downstream, and to the player's feel.
- Parallel paths don't align — a plugin with latency on one branch combs against a dry branch
  beside it, audibly, with no fix available to the user.
- If your plugin has fixed, known latency, say so in its description and modgui. That's the
  only channel.

<details>
<summary>Verification</summary>

`effects.c:4649,5103-5147` and a source-wide grep for `latency`, at `f14a230`.

</details>

## Block size, sample rate and what can change

`mod-host` reads block size and sample rate from JACK once and hands them to every plugin via
`opts:options` at instantiate:

| Option | Value |
|---|---|
| `param:sampleRate` (float) | the JACK rate: **48 000 on every MOD device** |
| `bufsz:minBlockLength`, `maxBlockLength`, `nominalBlockLength` (int) | all three the JACK period: 128 on current images, 256 on some Duo X setups |
| `bufsz:sequenceSize` (int) | the JACK MIDI buffer size — how much atom data an event port holds |
| `threads:schedPolicy`, `schedPriority` (int) | the policy/priority mod-host wants for any thread you create |

`bufsz:fixedBlockLength`/`powerOf2BlockLength`/`boundedBlockLength` are passed but never
enforced — mod-host just relies on JACK running a fixed power-of-two period. **The period can
change while you're instantiated**: on a change, mod-host updates its buffers and, if you
export `opts:interface`, calls your `set()` with the new sizes; without it, you simply start
receiving `run(nframes)` with the new count. Size for `maxBlockLength` at instantiate and either
implement `opts:interface` or chunk what you allocated. Never hardcode `n_samples == 128`.

<details>
<summary>Verification</summary>

Options populated at `effects.c:3961-3964,4233-4287`. Resize path at `effects.c:990-1054`.
`mod-utilities/Shared_files/BufferSize.h` is a worked chunking example.

</details>

## Port conventions

**Symbols and the plugin URI are permanent.** Pedalboards store your URI and each port's
`lv2:symbol` by value; rename either and every saved pedalboard using the plugin breaks,
silently, for the user. Decide both before your first release. Adding ports, groups or scale
points is safe — pedalboards and presets address by symbol, not index. (The one fact
[CAVEATS.md](CAVEATS.md) repeats, because it's the expensive one.)

**Ranges are enforced by the host; scale points are not.** A value arriving via `param_set`
(what mod-ui sends) is clamped to `mod:minimum`/`maximum`, falling back to
`lv2:minimum`/`maximum`. Nothing is rounded on that path — an `lv2:integer` or
`lv2:enumeration` port can receive 2.4, unsnapped. Rounding only happens for CV-addressed and
MIDI-mapped ports; values from Control Chain or a change-request are neither clamped nor
rounded. Treat every control input as a float that may be off-grid and snap it yourself.

<details>
<summary>Verification</summary>

Clamp path at `effects.c:4833-4862`, `f14a230`.

</details>

## State and persistence

On save, mod-ui records every control port's value and mod-host writes each plugin's LV2 state
(your `state:interface` `save()`) into the pedalboard bundle. On load the two come back by
different routes — control values via `param_set`, state via `lilv_state_restore` with **no
port-value callback** — so port values inside your own state file are ignored. Put in state
only what isn't a control port: a file path, a loaded model, a captured sample. Paths made with
`state:makePath` land under the pedalboard directory at save/load and a scratch directory at
runtime. A preset restores port values *and* state; `state:loadDefaultState` is honoured at
instantiate. Without `state:threadSafeRestore`, mod-host holds a mutex around restore and
outputs silence from your plugin while it does — the audible reason to implement it. Feature-
level detail is in [LV2-FEATURES.md](LV2-FEATURES.md).

## Threading

The LV2 worker is the one extra thread you get, and the only sanctioned place for anything
slow — file loads, model parsing, FFT planning, allocation. mod-host provides `worker:schedule`
only if you export `worker:interface`, and delivers `work_response()`/`end_run()` on the audio
thread after `run()`. If you create your own thread, use the `threads:schedPolicy`/
`schedPriority` options above so it sits where mod-host expects relative to JACK — and remember
a busy worker still competes with audio for cache and memory bandwidth on a four-core part.

## Bypass is done by mod-host, hard, with no crossfade

Bypass is not UI wiring — the modgui footswitch just sets a flag the audio thread acts on.

**No `lv2:enabled` port → the host bypasses you:** audio output *i* gets a `memcpy` of audio
input *i* (outputs beyond the last input copy the last input, so mono-in/stereo-out passes to
both sides). **The switch is instantaneous — no crossfade, no ramp**; wet/dry signals that
differ much in level or phase will click. `run()` keeps being called on zeroed audio/CV inputs
with no MIDI — tails decay, smoothers keep converging, CPU is still spent, the output is just
discarded. A plugin with no audio inputs (a synth, a CV source) gets its audio *and* CV outputs
zeroed — a clean way to switch a CV between 0 and its value with a footswitch. On the
transition into bypass, MIDI inputs get all-notes-off/all-sound-off on all 16 channels.

**Declare an `lv2:enabled` input port and you own bypass yourself** — the host writes 0.0/1.0
to that port instead and processes normally. This is the only way to get a click-free or
tail-preserving bypass.

<details>
<summary>Verification</summary>

Confirmed from `mod-host/src/effects.c` at `f14a230`: hard `memcpy` bypass, zeroed outputs for
sourceless plugins, all-notes-off/all-sound-off transition, `lv2:enabled` opt-out.

</details>

## What a plugin costs on each device, and how to measure it

**Measure on the device, against the installed binary, and read the distribution of block
times, not the average.** The budget is 2667 µs per block at 128 frames/48 kHz; a plugin that
averages 15% of a core but does an FFT every few blocks is judged by the block with the FFT
in it.

A CPU-heavy plugin family (pitch tracking, oscillators, filters, a plate reverb — the tracker
doing an FFT roughly once every eleven blocks), same source cross-compiled for both devices:

| at `chrt -f 40`, against the installed binary | Dwarf (Cortex-A35) | Duo (Cortex-A7, 32-bit) |
| --- | --- | --- |
| lighter variant: share of one core | 8–10% | 16% |
| lighter variant: median block / FFT block (p99) | 0.18 ms / 0.64 ms | 0.29 ms / 1.5 ms |
| heavier variant, everything on: share of one core | 13–15% | 25.5% |
| heavier variant: median block / FFT block (p99) | 0.30 ms / 0.77 ms | 0.50 ms / 1.7 ms |

A lighter plugin from the same family measured on a Duo X and a Dwarf: about 2% of a core,
0.04 ms median, 0.09 ms p99 on the Duo X; 7–11%, 0.15 ms, 1.1 ms on the Dwarf. As a first guess,
a Duo costs 1.7–2× a Dwarf on average and more in its worst block; both are far from a Duo X or
a desktop (the same plugins run 0.4–0.7% of a core on x86).

**FFT-heavy code scales far worse than that.** A spectral plugin (`mod-polyoctaver`, a
4096-point real FFT and its inverse per frame, plus per-bin work): 40 µs per 128-frame block on
desktop x86, **1535 µs on the Dwarf** — about 40×, not the 7× the table above would suggest.
FFTW itself on the Dwarf's A35 (single precision, NEON, wisdom loaded): 4096 points cost
230–260 µs, 2048 points 70–110 µs — a 4096 frame costs three times a 2048 one, not twice. So
one 4096 FFT pair per 128-frame block is a third of the budget before anything else runs. What
fixed that plugin's worst case: a frame every 256 samples instead of 128, analysis in one block
and synthesis in the next (one block of extra latency) so no block carries both transforms, and
not touching bins nobody will hear.

How to take these measurements:

- Cross-compile a small test host with the platform's own toolchain, copy it to the device,
  and have it `dlopen` the installed `.so`, connect buffers, call `run()` with 128 frames, and
  time every call. Run it as `chrt -f 40` — real-time, below jackd's priority 80; at normal
  priority jackd preempts it and you measure jackd's worst blocks, not yours.
- **Expect one to three ~43 ms blocks per run at FIFO priority regardless of the plugin** —
  that's the kernel throttling real-time tasks (`sched_rt_runtime_us`), not your code. Read
  median, p95, p99, not the max.
- For CPU load inside a live pedalboard, don't use `jack_cpu_load` — on these images it never
  exits and ignores `SIGTERM`. Ask a second `mod-host` instance instead (see
  [INSTALLING.md](INSTALLING.md)); read its `cpu_load` before and after adding the plugin, with
  input connected, and give it a few seconds to settle.
- **The device toolchains compile with `-ffast-math`.** If a measurement differs between your
  desktop build and the device build, rebuild the desktop one with the device's optimisation
  flags before suspecting the hardware, and diff the two builds' audio sample by sample —
  a perceived pitch or level difference is sometimes just the measurement tool disagreeing with
  itself across builds, not a real defect.

## To write / verify

- [x] Per-device cost figures: "What a plugin costs on each device"
- [ ] Is there an enforced or advisory CPU limit per pedalboard? (none found in mod-host;
      mod-ui's load display is the only feedback — to confirm)
- [ ] What happens to a plugin that xruns — `mod-xrun` exists; document what a developer sees
- [x] Denormals, latency, block size/sample rate: all answered above
