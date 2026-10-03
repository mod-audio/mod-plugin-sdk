# Categories

How a plugin is classified decides which tab it appears under in the device's plugin bar and in
the Store, and what the plugin info dialog prints as its category. The mapping is read from
`mod-ui/utils/utils_lilv.cpp` (`_get_plugin_categories`, `:1267-1424`, tables `:568-608`) and the
tab lists in `html/index.html` at `7764e1ff` (1.14 RC4), 2026-10-03.

## The rule in one paragraph

Declare **one** class. A `mod:` class if one fits, otherwise the standard LV2 class. mod-ui maps
whichever it recognises to a short list, `["Top", "Sub"]`, and only the first element is used for
tabs (`html/js/effects.js:274,331`); the info dialog prints it (`effects.js:441`). A plugin with no
recognised class gets an empty list, appears only under "All", and its dialog says "None". A plugin
with several classes gets one of them, not a union: the **first `mod:` class found wins and stops the
scan** (`utils_lilv.cpp:1413-1415`), and among plain LV2 classes the last one lilv happens to iterate
overwrites the earlier ones (`:1291-1374`, no `break`), so "several" means "unpredictable".
`modpedal:Pedalboard` marks the bundle as not a plugin (`:1283-1289`).

## The tabs

Device plugin bar (`index.html:795-811`): Favorites, All, Control Voltage, Delay, Distortion,
Dynamics, Filter, Generator, MIDI Utility, Modulator, Reverb, Simulator, Spatial, Spectral,
Utility. The Store (`index.html:882-897`) shows the same without Favorites. Two more exist in the
code but are hidden: Max gen~ and Camomile (`display:none`; a loop-variable bug in
`effects.js:288-293` ties the Max gen~ tab's visibility to the Camomile count, so it never shows).
Sub-categories are never a tab; they only appear in the info dialog.

## `mod:` classes

| Class | Category list |
|---|---|
| `mod:DelayPlugin`, `mod:DistortionPlugin`, `mod:DynamicsPlugin`, `mod:FilterPlugin`, `mod:GeneratorPlugin`, `mod:ModulatorPlugin`, `mod:ReverbPlugin`, `mod:SimulatorPlugin`, `mod:SpatialPlugin`, `mod:SpectralPlugin`, `mod:UtilityPlugin` | the same single top-level string as the LV2 equivalent |
| `mod:MIDIPlugin` | `["MIDI"]` |
| `mod:ControlVoltagePlugin` | `["ControlVoltage"]` |
| `mod:MaxGenPlugin` | `["MaxGen"]` (hidden tab) |
| `mod:CamomilePlugin` | `["Camomile"]` (hidden tab) |

Any other `mod:` class is skipped as invalid. **`mod:ControlVoltagePlugin` and `mod:MaxGenPlugin`
are real categories, not markers**: they replace whatever LV2 class the plugin also declares. So a
CV utility that wants the Control Voltage tab declares `mod:ControlVoltagePlugin` and nothing
else; declaring `lv2:UtilityPlugin` as well changes nothing, declaring it *instead* puts the
plugin under Utility.

## Standard LV2 classes

| LV2 class | Category list |
|---|---|
| `lv2:DelayPlugin` | Delay |
| `lv2:DistortionPlugin` | Distortion |
| `lv2:WaveshaperPlugin` | Distortion, Waveshaper |
| `lv2:DynamicsPlugin` | Dynamics |
| `lv2:AmplifierPlugin` | Dynamics, Amplifier |
| `lv2:CompressorPlugin` | Dynamics, Compressor |
| `lv2:ExpanderPlugin` | Dynamics, Expander |
| `lv2:GatePlugin` | Dynamics, Gate |
| `lv2:LimiterPlugin` | Dynamics, Limiter |
| `lv2:FilterPlugin` | Filter |
| `lv2:AllpassPlugin` | Filter, Allpass |
| `lv2:BandpassPlugin` | Filter, Bandpass |
| `lv2:CombPlugin` | Filter, Comb |
| `lv2:EQPlugin` | Filter, Equaliser |
| `lv2:MultiEQPlugin` | Filter, Equaliser, Multiband |
| `lv2:ParaEQPlugin` | Filter, Equaliser, Parametric |
| `lv2:HighpassPlugin` | Filter, Highpass |
| `lv2:LowpassPlugin` | Filter, Lowpass |
| `lv2:GeneratorPlugin` | Generator |
| `lv2:ConstantPlugin` | Generator, Constant |
| `lv2:InstrumentPlugin` | Generator, Instrument |
| `lv2:OscillatorPlugin` | Generator, Oscillator |
| `lv2:ModulatorPlugin` | Modulator |
| `lv2:ChorusPlugin` | Modulator, Chorus |
| `lv2:FlangerPlugin` | Modulator, Flanger |
| `lv2:PhaserPlugin` | Modulator, Phaser |
| `lv2:ReverbPlugin` | Reverb |
| `lv2:SimulatorPlugin` | Simulator |
| `lv2:SpatialPlugin` | Spatial |
| `lv2:SpectralPlugin` | Spectral |
| `lv2:PitchPlugin` | Spectral, Pitch Shifter |
| `lv2:UtilityPlugin` | Utility |
| `lv2:AnalyserPlugin` | Utility, Analyser |
| `lv2:ConverterPlugin` | Utility, Converter |
| `lv2:FunctionPlugin` | Utility, Function |
| `lv2:MixerPlugin` | Utility, Mixer |
| `lv2:MIDIPlugin` | MIDI, Utility (the MIDI tab) |

`lv2:Plugin` on its own is skipped. There is no Pitch Shifter, Equaliser or Compressor tab: those
are sub-categories, shown in the dialog only.

## The Cloud Builder share page

builder.mod.audio reduces the class the same way for its share page (a `mod:` class first, else
the `lv2:` one, to the top-level name); see [METADATA.md](METADATA.md).
