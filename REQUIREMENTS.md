# Requirements — how MOD expects a plugin to behave

This chapter is about behaviour, not API: what the host does to your plugin, what it will not
do for you, and what the devices can afford. Everything in it was either measured on a device
or read from `mod-host` at `f14a230` (master, what ships), with the line cited. The short
version:

- `run()` must not allocate, lock, or do I/O. Measure that with a counter, not a code review.
- The budget is one block period shared with the whole pedalboard, and the Dwarf and Duo are
  far slower than a laptop; FFT-heavy code scales worst.
- Denormals are flushed to zero by the host on every audio thread; you do not have to.
- The host does hard bypass for you unless you declare an `lv2:enabled` port; then it is yours.
- Latency reporting does not exist. A plugin's latency is invisible to the host and the user.
- Block size comes from the options feature and can change at runtime; never bake it in.
- Port symbols and the plugin URI are permanent once a pedalboard has used them.
- There is exactly one extra thread you get for free, the worker; use it for everything slow.

## Verified: real-time safety, and why it is not sufficient

**In `run()`: no allocation, no free, no lock, no file or network I/O, no logging that allocates.**
That includes the things that hide behind ordinary C++: `std::vector` growth, `std::string`
assignment that exceeds capacity, `std::function` construction, and any container you touch for the
first time. Anything that can block or take an unbounded amount of time belongs on the worker
thread.

**Your whole budget is one block period, shared with every other plugin in the chain.** At 48 kHz
that is `block_size / 48000` seconds — 2.67 ms at 128 frames, 5.33 ms at 256. It is not per plugin:
if your plugin alone peaks at half the period, a user with four more plugins after yours is going to
xrun, and it will look like their fault rather than yours.

**Measure it rather than reasoning about it.** The cheap and conclusive technique is to interpose
`malloc`/`free`/`pthread_mutex_lock` in a test harness, set a thread-local flag around the `run()`
call, and count anything that happens on that thread — a violation then shows up as a number instead
of a code review opinion. A worked harness is `mod-nam-loader/tools/rt_probe.cpp`: it paces the
audio callback at the block deadline on one thread, runs the LV2 worker on another, and reports block
time percentiles alongside the allocation and lock counts. It cross-compiles with the platform
toolchain, which is the only way to get honest numbers for an A35.

**The rule is broken inside MOD's own catalog, so do not take the shipped plugins as a
reference.** A 2026-09-16 read of the 101 open-source repos behind the stable Store found
`realloc()` in `run()` (`mod-distortion`), FFTW re-planning plus a wisdom-file read from `run()`
on a knob change (`mod-pitchshifter`, all eight plugins), and `fprintf` from `run()`
(`mod-midi-utilities` — a `DEBUG_PLUGIN_LOG` the file defines itself, in a bundle that is not
in the shipped set). Details and file:line references in [CAVEATS.md](CAVEATS.md) and in
`Plugins/mod-plugin-management/analysis/REPORT.md` §R1. All of them were fixed on local branches
on 2026-09-16 (status in `notes/mod-plugin-management.md`); until those ship, the Store copies
still carry the defects. Measured on a Dwarf the same day with
`Plugins/mod-plugin-management/tools/rt_check` (allocation counting on the audio thread, real
signal): the old pitchshifters made 2,200–5,700 allocations during a fidelity sweep and spent up
to **82 ms on one 128-frame block** (deadline 2.67 ms), which is 46–52 xruns per 36 knob moves in
mod-host; the fixed builds make 0 allocations, peak at 1.1 ms and produce 0 xruns. That is the
kind of number to aim for: not "no xruns while I played", but zero allocations under a sweep.

The same tool also settles "does the fix change the sound?" without ears: `rt_check --in <file.f32>
--dump <prefix>` feeds a raw float32 file to the plugin and writes every audio output as raw
float32, so the old and the new build can be diffed sample by sample on the unit. Done for the
pitchshifter fix on 2026-09-17: 46 output streams, bit-identical old vs new, which turned a
listening-test complaint (levels, dry bleed) into a fact about the original algorithm rather than
the fix. A regression check for a real-time fix should always include that step.

Two lessons from doing this that are worth passing on:

- **A harness that instruments allocation will catch itself first.** The first runs of the probe
  above reported 17 allocations and 4,017 locks that were entirely the harness's own queue and
  buffers, called from the audio thread. Exclude your harness's code before believing its output.
- **Passing the RT-safety test does not mean you will not xrun.** A plugin can be perfectly clean in
  `run()` — measured worst block 0.11 ms against a 2.67 ms deadline — and still cause xruns when it
  loads a file, because the *worker* thread competes with audio for CPU and memory bandwidth on a
  4-core embedded part. If your plugin loads models, IRs or samples, budget for that too. What
  helped in practice: keep parsed data resident in RAM instead of re-reading it (a small LRU cache
  turns "switch back to the previous model" into a pointer swap), and stop glibc returning freed
  memory to the kernel with `mallopt(M_TRIM_THRESHOLD, -1)` and `mallopt(M_MMAP_MAX, 0)`, since
  unmapping pages forces TLB shootdowns that stall the core running audio. MOD devices have RAM to
  spare; spend it.

<!-- GAP: whether there's an enforced or advisory CPU limit per pedalboard is unconfirmed; what a
     developer actually sees when a plugin xruns (mod-xrun) is undocumented -->

## Verified: denormals are flushed by the host

You do not need to set flush-to-zero yourself on a MOD device, and setting it does no harm.
mod-host installs a JACK thread-init callback on its global client and on every plugin's client
(`mod-host/src/effects.c:4094,5410`), and that callback (`JackThreadInit`, `effects.c:2902-2934`)
sets the FZ bit of `FPCR` on AArch64 (`orr %0, %0, #0x1000000`, `effects.c:2916-2918`), the same
bit of `FPSCR` on the 32-bit Duo (`effects.c:2924-2926`), and both FTZ and DAZ in `MXCSR` on x86
(`ldmxcsr(mxcsr | 0x8040)`, `effects.c:2912`). JACK runs that callback on each real-time thread
it creates, so every thread that will ever call your `run()` has denormals off before the
first cycle. The worker thread is not a JACK thread and does not get it; if you do heavy float
work there (an IR pre-computation, a model warm-up), set it yourself or accept the slowdown.

What your own build does is irrelevant to this, which is just as well: a plugin compiled by
`mod-plugin-builder` carries GCC's `crtfastmath` constructor (the platform `-ffast-math` plus GCC
9.4's link spec, see the denormals entry in [CAVEATS.md](CAVEATS.md)), but that sets the bit only on the
thread that loads the bundle, and `FPCR` is per thread. On another host, or on a desktop, do not
rely on either: use the usual DC-offset trick in feedback paths or set FZ in `run()`.

## Verified: latency is not reported

mod-host has no latency support of any kind. It never looks for a port designated `lv2:latency`
(the only designations it resolves are `lv2:control`, `lv2:enabled`, `lv2:freeWheeling`,
`time:beatsPerBar`, `time:beatsPerMinute` and `time:speed`, `effects.c:4649,5103-5147`), never
reads `lv2:reportsLatency`, and never calls `jack_port_set_latency_range` or registers a latency
callback (`grep -n -i latency src/*` finds only the Ableton Link output latency,
`effects.c:3826-3831`). A latency output port is an ordinary control output: nobody compensates
for it, and the user has no way to see it. Consequences:

- Keep latency low because nothing hides it. A look-ahead limiter or a spectral effect adds its
  frames to everything behind it in the chain, and to the player's feel.
- Parallel paths are not aligned. A plugin with latency on one branch of a pedalboard and a dry
  branch beside it combs; the user will hear it and cannot fix it.
- If your plugin has a fixed, known latency, say so in its description and the modgui. That is
  the only channel.

## Verified: block size, sample rate and what can change

mod-host reads the block size and sample rate from JACK once (`jack_get_buffer_size`,
`jack_get_sample_rate`, `effects.c:3961-3964`) and hands them to every plugin through the
`opts:options` feature at instantiate (`g_options`, `effects.c:4233-4287`):

| Option | Value |
|---|---|
| `param:sampleRate` (float) | the JACK rate: **48 000 on every MOD device** |
| `bufsz:minBlockLength`, `bufsz:maxBlockLength`, `bufsz:nominalBlockLength` (int) | all three the JACK period: 128 on current images, 256 on some Duo X setups |
| `bufsz:sequenceSize` (int) | the JACK MIDI buffer size, i.e. how much atom data an event port holds |
| `threads:schedPolicy`, `threads:schedPriority` (int) | the policy and priority mod-host wants for any thread you create (`MOD_PLUGIN_THREAD_PRIORITY` in the environment, `effects.c:3968-3972`) |

`bufsz:fixedBlockLength`, `bufsz:powerOf2BlockLength` and `bufsz:boundedBlockLength` are
passed as features (`effects.c:763-767`) but never checked or enforced: mod-host relies on JACK
running a fixed, power-of-two period. **The period can change while you are instantiated.** Each
plugin client has a buffer-size callback (`effects.c:5412`); on a change mod-host updates its
globals and event buffers and, if you export `opts:interface`, calls its `set()` with the new
nominal, min, max and sequence size (`BufferSize`, `effects.c:990-1054`). A plugin without
`opts:interface` simply starts receiving `run(nframes)` with the new count. So: size for
`maxBlockLength` at instantiate, and either implement `opts:interface` or process in chunks of
what you allocated (`mod-utilities/Shared_files/BufferSize.h` is a worked example of the chunking).
Never test `n_samples == 128`.

## Port conventions

**Symbols and the plugin URI are permanent.** Pedalboards store your URI and each port's
`lv2:symbol`; rename either and every saved pedalboard that used the plugin breaks, silently for
the user. Decide both before the first release. Adding ports, port groups or scale points is safe,
and pedalboards and presets both address ports by symbol, so indices may change. (This is the one
fact [CAVEATS.md](CAVEATS.md) repeats, because it is the expensive one.)

**Ranges are enforced by the host, scale points are not.** A control value arriving by `param_set`
(what mod-ui sends) is clamped to `mod:minimum`/`mod:maximum`, falling back to
`lv2:minimum`/`lv2:maximum`, default 0–1 (`effects.c:4833-4851,6036-6041`; a range with min ≥ max
is widened to min + 0.1, `effects.c:4854-4855`; `lv2:sampleRate` ports are scaled by the rate,
`effects.c:4858-4862`). Nothing is rounded on that path: an `lv2:integer` or `lv2:enumeration`
port can receive 2.4, and an enumeration is never snapped to its scale points. Integer rounding
happens only for CV-addressed (`effects.c:1946-1947`) and MIDI-mapped (`effects.c:2433-2434`)
ports, and values coming from Control Chain or the change-request feature are neither clamped
nor rounded (`SetPortValue`, `effects.c:2290-2355`). Treat every control input as a float that may
be slightly off-grid and pick the nearest valid value yourself.

## State and persistence

What survives: when the user saves a pedalboard, mod-ui records every control port's value, and
mod-host writes each plugin's **LV2 state** (what your `state:interface` `save()` emits) into the
pedalboard bundle, one `effect-<N>/effect.ttl` per instance (`state_save`, `effects.c:7799-7910`).
On load the two come back by different routes: control values through `param_set`, state through
`lilv_state_restore` with **no port-value callback** (`effects.c:7782-7786`), so port values inside
your state file are ignored. Put in state only what is not a control port: a file path, a loaded
model, a captured sample. Paths you make with `state:makePath` land under the pedalboard
directory during save/load and under a scratch directory (`/tmp/mod-host-scratch-dir` by default)
at runtime (`effects.c:3690-3700,4334`). A preset (`pset:Preset`) is restored with port values
*and* state (`preset_load`, `effects.c:5457-5510`), and `state:loadDefaultState` is honoured at
instantiate (`effects.c:4706-4714`). Without `state:threadSafeRestore` mod-host takes a mutex
around restore and outputs silence from your plugin while it holds it (`effects.c:1716-1736,
4700-4704`), which is the audible reason to implement it. The feature-level detail (which
features you get, in which callbacks) is in [LV2-FEATURES.md](LV2-FEATURES.md).

## Threading

The LV2 worker is the one additional thread you get, and the only sanctioned place for anything
slow: file loads, model parsing, FFT planning, allocation. mod-host provides `worker:schedule`
only if you export `worker:interface` (`effects.c:2966-2973`) and delivers `work_response()` and
`end_run()` on the audio thread after `run()` (`effects.c:2083-2090`). If you must create a thread
of your own, use the `threads:schedPolicy` / `threads:schedPriority` options above so it sits
where mod-host expects relative to JACK, and remember the lesson in "real-time safety" above: a
busy worker on a four-core embedded part competes with audio for cache and memory bandwidth, so
a plugin can be clean in `run()` and still xrun during a load.

## Verified: bypass is done by mod-host, hard, with no crossfade

Read from `mod-host/src/effects.c` at `f14a230` (2026-09-17). Bypass is not UI wiring; the modgui
footswitch only sets a flag that the audio thread acts on.

**A plugin without an `lv2:enabled` port is bypassed by the host** (`effects.c:2003`):

- Audio output *i* gets a `memcpy` of audio input *i*. Outputs beyond the last input get a copy of
  the **last** input, so a mono-in/stereo-out plugin passes its input to both sides
  (`effects.c:2009-2022`). **The switch is instantaneous: no crossfade, no ramp.** A plugin whose
  wet and dry signals differ a lot in level or phase will click.
- `run()` **keeps being called**, on zeroed audio and CV inputs and with no MIDI
  (`effects.c:2024-2033`, comment: "to avoid 'pause behavior' in delay plugins"). Tails decay,
  smoothers keep converging, CPU is still spent. What the plugin writes is thrown away.
- A plugin with **no audio inputs** (a synth, a CV source) has its audio **and CV** outputs
  zeroed (`effects.c:2037-2049`). So bypassing a Control to CV block is a clean way to switch a
  CV between 0 and its value, and pedalboards do exactly that with a footswitch.
- On the transition into bypass, MIDI inputs receive all-notes-off and all-sound-off on all 16
  channels (`effects.c:1833-1862`).

**A plugin with an input port designated `lv2:enabled` handles bypass itself**
(`effects.c:5103-5107`). The host never takes the branch above (`enabled_index >= 0`); it writes
`0.0` to that port for bypassed and `1.0` for active (`effects.c:2297-2298`) and processes the
plugin normally. This is the only way to get a click-free or tail-preserving bypass: declare the
port and do the crossfade yourself.

## Verified: what a plugin costs on each device, and how to measure it

**Measure on the device, against the installed binary, and read the distribution of block times, not the
average.** The budget is per block (2667 µs at 128 frames, 48 kHz); a plugin that averages 15 % of a core
but does an FFT once every few blocks is judged by the block with the FFT in it.

Two plugins of one family (`mod-gsynth`, `mod-gsynth-flexible`: a YIN pitch tracker, oscillators, filters, a
plate reverb; the tracker does an FFT once every eleven blocks), same source on both devices, 2026-09-17, 128
frames at 48 kHz, firmware 1.14.0.3333:

| at `chrt -f 40`, against the installed binary | Dwarf (Cortex-A35) | Duo (Cortex-A7, 32-bit) |
| --- | --- | --- |
| lighter plugin: share of one core | 8–10 % | 16 % |
| lighter plugin: ordinary block (median) / block carrying the FFT (p99) | 0.18 ms / 0.64 ms | 0.29 ms / 1.5 ms |
| heavier plugin, everything switched on: share of one core | 13–15 % | 25.5 % |
| heavier plugin: ordinary block (median) / block carrying the FFT (p99) | 0.30 ms / 0.77 ms | 0.50 ms / 1.7 ms |
| heavier plugin added to a live pedalboard (JACK DSP load) | not measured | +11.7 points |

Another plugin of the family, measured on a Duo X and on the Dwarf: about 2 % of a core, median 0.04 ms, p99
0.09 ms on the Duo X; 7–11 %, 0.15 ms, 1.1 ms on the Dwarf. Shares of a core move by a few points between
runs with whatever else the device is doing.

So as a first guess a Duo costs 1.7 to 2 times a Dwarf on average and rather more in its worst block, and
both are far from a Duo X or a laptop (the same plugins take 0.4–0.7 % of a core on a desktop x86). A plugin
accepted on a Dwarf is not thereby fine on a Duo.

**FFT-heavy code scales far worse than that.** A spectral plugin (`mod-polyoctaver`, a 4096-point real FFT
and its inverse per frame, plus per-bin work) measured 2026-09-17: 40 µs per 128-frame block on a desktop
x86, **1535 µs on the Dwarf** (worst block 2626 of 2667) — about 40 times, not the 7 that had been assumed
from the table above. What FFTW itself costs on the Dwarf's A35 (single precision, NEON build, wisdom
loaded, `fftwf_plan_dft_r2c_1d`, timed alone): **4096 points 230–260 µs, 2048 points 70–110 µs**, the inverse
the same; inside a plugin, with the rest of its data competing for the cache, the 4096 transform took 410 µs.
So one 4096 FFT pair per 128-frame block is a third of the budget before anything else happens, and a
4096 frame costs three times a 2048 one, not twice. What brought that plugin to 666 µs average / 1530 worst:
a frame every 256 samples instead of every 128, with the analysis done in one 128-frame block and the
synthesis in the next so that no block carries both transforms (one block of extra latency), and not
touching bins nobody will hear. For scale, on the same unit and input `mod-capo` costs 362 µs per block at
Fidelity 1 and 621 µs at Fidelity 2.

How those were taken, and what goes wrong:

- Cross-compile a small test host with the platform's own toolchain (`<WORKDIR>/<platform>/host/usr/bin/*-g++`),
  copy it to `/tmp`, run it against `/root/.lv2/<bundle>/<plugin>.so`: it `dlopen`s the plugin, connects
  buffers, calls `run()` with 128 frames and times every call. Run it as `chrt -f 40 ./test_host …`: real-time,
  below jackd's priority 80. At normal priority jackd preempts it and the worst blocks are jackd's, not yours.
  (`Plugins/mod-gsynth-creator/tools/{test_host.cpp,device-test.sh}` is one such.)
- **A flat-out test at FIFO priority shows one to three blocks of about 43 ms per run.** That is the kernel
  throttling real-time tasks (`sched_rt_runtime_us` 950000 of 1000000), acting on the test because it never
  sleeps. It is not the plugin. Read median, p95 and p99.
- **For the load inside the live graph, do not use `jack_cpu_load`**: on these images it never exits and
  ignores SIGTERM (`killall -9`). Ask a second mod-host instead, see [INSTALLING.md](INSTALLING.md). Its
  `cpu_load` is JACK's figure for the whole graph: read it before and after adding the plugin, with the
  input connected, and give it several seconds to settle.
- **The device toolchains compile with `-ffast-math`.** If a measurement differs between your desktop build and
  the device build, rebuild the desktop one with the device's flags before suspecting the hardware
  (`BR2_TARGET_OPTIMIZATION` in `mod-plugin-builder/plugins-dep/configs/<platform>_defconfig`), and compare the
  two builds' audio sample by sample. In the case that prompted this note the builds differed by −72 dB and the
  "11 cents sharp on the device" was the test's own pitch estimator picking a different autocorrelation lag.

## To write / verify

- [x] Per-device cost figures: "Verified: what a plugin costs on each device"
- [ ] Is there an enforced or advisory CPU limit per pedalboard? (none found in mod-host; mod-ui's
      load display is the only feedback — to confirm)
- [ ] What happens to a plugin that xruns — `mod-xrun` exists; document what a developer sees
- [x] Denormals: "Verified: denormals are flushed by the host" (2026-10-03)
- [x] Latency: not supported (2026-10-03)
- [x] Block size / sample rate / runtime change (2026-10-03)
