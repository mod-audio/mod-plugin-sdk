# The MOD GUI (modgui)

Every plugin on a MOD device has a **web interface**. This document explains how to build one.

The interface is plain HTML, CSS and images that you ship inside your LV2 bundle. MOD's web UI
renders it in the browser. There is no compiled GUI, no toolkit, and no plugin-side UI code — if you
can write a web page, you can write a MOD GUI.

> **You do not have to write one.** A plugin without a modgui still loads, still works, and is shown
> using a generic default pedal. A modgui is what makes it *yours*.

---

## 1. The two interfaces

A modgui is two separate HTML templates.

**The icon** is the pedal itself — what the user sees on the pedalboard canvas, connected to other
plugins by cables. It is a small, fixed-size representation designed to look like a real device.
It carries the plugin's most important controls and nothing else.

An icon normally has:

- a pedal-like design
- a footswitch that bypasses the plugin
- a light showing the bypass state
- knobs, switches or sliders for the main parameters
- input and output jacks for cables
- a draggable area for moving the pedal around

**The settings panel** is the full interface — every control the plugin has, presets, and anything
that does not fit on the pedal face. It opens when the user asks for it and can occupy the whole
screen.

The icon is mandatory. The settings panel is optional; without one, MOD renders a generic panel from
your port metadata, which is often perfectly adequate.

---

## 2. Where a modgui lives

Two arrangements. Both work.

**Inside the plugin bundle** — the normal case, and what you should do for a plugin you control:

```
myplugin.lv2/
├── manifest.ttl
├── myplugin.ttl
├── myplugin.so
└── modgui/
    ├── icon-myplugin.html
    ├── settings-myplugin.html
    ├── stylesheet.css
    ├── screenshot.png
    ├── thumbnail.png
    └── myknob.png
```

**In a separate `.modgui` bundle** — used when you are adding a GUI to a plugin whose bundle you
cannot modify, for example a third-party or system-installed plugin:

```
myplugin.modgui/
├── manifest.ttl
└── modgui/
    └── …
```

The `modgui/` subdirectory name is a convention, not a requirement — what matters is that
`modgui:resourcesDirectory` points at it.

---

## 3. Declaring it in TTL

The GUI is attached to your plugin with `modgui:gui`. Put it in your plugin's `.ttl`, or in a
separate `modgui.ttl` referenced with `rdfs:seeAlso`:

```turtle
@prefix lv2:    <http://lv2plug.in/ns/lv2core#> .
@prefix modgui: <http://moddevices.com/ns/modgui#> .
@prefix rdfs:   <http://www.w3.org/2000/01/rdf-schema#> .

<http://example.com/plugins/myplugin>
    modgui:gui [
        a modgui:Gui ;
        modgui:resourcesDirectory <modgui> ;
        modgui:iconTemplate <modgui/icon-myplugin.html> ;
        modgui:settingsTemplate <modgui/settings-myplugin.html> ;
        modgui:stylesheet <modgui/stylesheet.css> ;
        modgui:screenshot <modgui/screenshot.png> ;
        modgui:thumbnail <modgui/thumbnail.png> ;
        modgui:brand "Example Audio" ;
        modgui:label "MY PLUGIN" ;
    ] .
```

If you keep the GUI in its own file, link it from the plugin:

```turtle
<http://example.com/plugins/myplugin> rdfs:seeAlso <modgui.ttl> .
```

### 3.1 Required properties

The specification declares these with `owl:cardinality 1` — a modgui **must** have all five:

| Property | What it is |
|---|---|
| `modgui:resourcesDirectory` | Directory holding every file the GUI serves. Everything else is relative to the bundle |
| `modgui:iconTemplate` | HTML template for the pedal on the canvas |
| `modgui:stylesheet` | CSS for the GUI |
| `modgui:screenshot` | Full-size image of the pedal, used in the plugin store |
| `modgui:thumbnail` | Small version of the same, used in listings |

A modgui missing any of these is invalid and may not render.

### 3.2 Optional properties

| Property | What it is |
|---|---|
| `modgui:settingsTemplate` | HTML template for the full settings panel. Omit for a generated one |
| `modgui:javascript` | A JS callback for GUI behaviour beyond plain controls — see §8 |
| `modgui:templateData` | Extra data made available to your templates |
| `modgui:monitoredOutputs` | Output ports whose values the GUI receives at runtime — for meters and indicators |
| `modgui:discussionURL` | Link to a forum thread or support page for the plugin |

### 3.3 Descriptive properties

`modgui:brand`, `modgui:label`, `modgui:model`, `modgui:panel`, `modgui:color` and `modgui:knob` are
strings that become available to your templates as Mustache variables. The specification describes
them as *"selected when building a modgui through the MOD SDK"* — they record the choices made in the
GUI wizard, and the stock templates read them.

If you are hand-writing a template you can either set them and use `{{brand}}` / `{{label}}`, or
ignore them and hardcode your text. Setting `modgui:brand` and `modgui:label` is recommended
regardless, because they are what the pedal displays as its name.

**`modgui:model`, `modgui:panel`, `modgui:knob` and `modgui:color` are parsed but mod-ui does
nothing with them itself.** Read at `7764e1ff` (1.14 RC4): `utils/utils_lilv.cpp:2476-2500` reads
all four into the GUI struct, and their only consumer is `html/js/modgui.js:1701-1712`, which
injects them as Mustache variables (`{{model}}`, `{{panel}}`, `{{knob}}`, `{{color}}`) when
rendering **your own** `iconTemplate` / `settingsTemplate`. No stylesheet or script in mod-ui
reads them, and the default pedal template does not use them (`git grep` for the four variables in
templates: no hits). So they have an effect exactly when your template or CSS uses the variable,
which the old mod-sdk templates do; otherwise they are inert. Declare them for those templates,
skip them for a hand-written one.

---

## 4. Templates are Mustache

This is the part that surprises people. **Template files are not static HTML.** MOD's web UI renders
them with [Mustache](https://mustache.github.io/) in the browser, against your plugin's own LV2
metadata, every time the plugin is loaded.

That is deliberate and you should not try to avoid it. A template that iterates its own ports adapts
automatically when you add a control; one with hardcoded markup silently breaks the day your port
list changes.

### 4.1 Available variables

| Variable | Value |
|---|---|
| `{{brand}}` | `modgui:brand` |
| `{{label}}` | `modgui:label` |
| `{{color}}`, `{{knob}}`, `{{model}}`, `{{panel}}` | The corresponding `modgui:` properties |
| `{{{cns}}}` | **The class namespace.** Critical — see §5 |
| `{{name}}` | Within a port or control section, that port's human-readable name |
| `{{symbol}}` | Within a port or control section, that port's LV2 symbol |
| `{{comment}}` | Within a control section, that port's `rdfs:comment` |

### 4.2 Available sections

Sections iterate. `{{#controls}}…{{/controls}}` repeats its contents once per control input port:

| Section | Iterates over |
|---|---|
| `{{#controls}}` | Control input ports |
| `{{#effect.ports.audio.input}}` / `.output` | Audio ports |
| `{{#effect.ports.midi.input}}` / `.output` | MIDI ports |
| `{{#effect.ports.cv.input}}` / `.output` | CV ports |
| `{{#presets}}` | Available presets (settings template) |

A minimal but complete icon looks like this:

```html
<div class="mod-pedal mod-pedal-boxy{{{cns}}} mod-two-knobs mod-{{color}}">
    <div mod-role="drag-handle" class="mod-drag-handle"></div>
    <div class="mod-plugin-brand"><h1>{{brand}}</h1></div>
    <div class="mod-plugin-name"><h1>{{label}}</h1></div>
    <div class="mod-light on" mod-role="bypass-light"></div>

    <div class="mod-control-group mod-{{knob}} clearfix">
        {{#controls}}
        <div class="mod-knob" title="{{comment}}">
            <div class="mod-knob-image"
                 mod-role="input-control-port"
                 mod-port-symbol="{{symbol}}"></div>
            <span class="mod-knob-title">{{name}}</span>
        </div>
        {{/controls}}
    </div>

    <div class="mod-footswitch" mod-role="bypass"></div>

    <div class="mod-pedal-input">
        {{#effect.ports.audio.input}}
        <div class="mod-input mod-input-disconnected" title="{{name}}"
             mod-role="input-audio-port" mod-port-symbol="{{symbol}}">
            <div class="mod-pedal-input-image"></div>
        </div>
        {{/effect.ports.audio.input}}
    </div>

    <div class="mod-pedal-output">
        {{#effect.ports.audio.output}}
        <div class="mod-output mod-output-disconnected" title="{{name}}"
             mod-role="output-audio-port" mod-port-symbol="{{symbol}}">
            <div class="mod-pedal-output-image"></div>
        </div>
        {{/effect.ports.audio.output}}
    </div>
</div>
```

---

## 5. `{{{cns}}}` — the class namespace, and the mistake everyone makes

Every plugin's CSS is injected into the same page. Without namespacing, two plugins both defining
`.mod-knob` would fight, and the last one loaded would win.

MOD solves this by giving each plugin a **class namespace** — a suffix derived from the plugin's URI
and version, expanded into `{{{cns}}}`:

```html
<div class="mod-pedal mod-pedal-boxy{{{cns}}}">
```

renders as something like:

```html
<div class="mod-pedal mod-pedal-boxy_http___example_com_plugins_myplugin1">
```

Three rules follow, and getting them wrong is the most common modgui failure:

**Use triple braces.** `{{{cns}}}` not `{{cns}}` in HTML — double braces HTML-escape the value.

**Never hardcode a rendered class name.** If you copy HTML out of your browser's inspector and paste
the *expanded* class into your template, you have hardcoded another plugin's namespace.

**Never use a bare MOD-reserved class.** These are reserved and must always carry `{{{cns}}}`:

```
mod-pedal-boxy      mod-pedal-british     mod-pedal-japanese
mod-pedal-lata      mod-combo-model-001   mod-head-model-001
mod-rack-model-001
```

If MOD's UI finds one of these **without** a namespace suffix, it rejects your icon entirely and
substitutes the default pedal. It logs `This icon uses old MOD reserved css classes, this is not
allowed anymore` to the browser console and otherwise says nothing.

> **If your plugin shows a generic pedal instead of your design, this is almost certainly why.**
> Open the browser console and look for that message.

### 5.1 Your stylesheet is a Mustache template too

The CSS is rendered through Mustache as well, with two variables:

| Variable | Use |
|---|---|
| `{{cns}}` | The same class namespace. Every selector for a reserved class needs it |
| `{{ns}}` | URL suffix for serving files out of your resources directory |

```css
.mod-pedal-boxy{{cns}} {
    background-image: url(pedal.png{{ns}});
    width: 260px;
    height: 140px;
}

.mod-pedal-boxy{{cns}} .mod-knob-image {
    background-image: url(myknob.png{{ns}});
}
```

**Image URLs need `{{ns}}`.** Your files are served through MOD's web UI rather than from a plain
directory, and `{{ns}}` carries the plugin URI and version that make the request resolve. A
stylesheet referencing `url(pedal.png)` without it will silently show no image.

Note `{{cns}}` in CSS uses double braces — there is no HTML escaping to avoid.

---

## 6. `mod-role` — how HTML becomes functional

A `<div>` is decoration until it carries a `mod-role` attribute. That attribute is the contract
between your markup and MOD's UI: it tells the host what an element *does*. Controls additionally
need `mod-port-symbol` naming the LV2 port they drive.

| `mod-role` | Purpose |
|---|---|
| `input-control-port` | A knob, slider or switch driving a control input. Needs `mod-port-symbol` |
| `input-control-value` | Text display of the current value. Editable unless the port is enumerated, toggled or a trigger |
| `input-control-minimum` / `input-control-maximum` | Displays the port's range |
| `enumeration-option` | One option of an enumerated port |
| `input-parameter` / `input-parameter-value` | As above, for LV2 patch parameters rather than ports |
| `bypass` | The footswitch. Toggles the plugin |
| `bypass-light` | The indicator showing bypass state |
| `drag-handle` | The grabbable area for moving the pedal on the canvas |
| `icon-button` | A generic clickable button |
| `presets` | Preset selector |
| `input-audio-port` / `output-audio-port` | Audio jacks. Need `mod-port-symbol` |
| `input-midi-port` / `output-midi-port` | MIDI jacks |
| `input-cv-port` / `output-cv-port` | CV jacks |

Ports are matched by `mod-port-symbol`, which must equal the `lv2:symbol` in your plugin's TTL —
not the human-readable `lv2:name`. A mismatch produces a control that renders but does nothing.

---

## 7. Screenshot and thumbnail

Both are required.

- **`modgui:screenshot`** — full-size image of the rendered pedal, used on the plugin's store page
- **`modgui:thumbnail`** — reduced version, used in browse listings. Maximum **256 × 64 px**;
  larger images are scaled down proportionally

Generate them by rendering the icon and capturing it, rather than drawing them by hand — a
screenshot that doesn't match the actual pedal is worse than none. `mod-sdk` includes a capture tool,
though it depends on PhantomJS, which is dead software; expect to do this manually or with a
headless browser of your own.

---

## 8. JavaScript, and when you need it

`modgui:javascript` points at a file containing a **single JavaScript function expression** — no
wrapper, no module, just a function. MOD evaluates it and calls it on GUI events.

Use it for behaviour plain controls cannot express: a display that reformats a value, a control whose
appearance depends on another control, a meter driven by `modgui:monitoredOutputs`.

Do not use it for anything you can achieve with `mod-role` and CSS. Every line here is a line that
can break when the UI changes, and it runs inside MOD's page rather than in a sandbox.

If your file fails to evaluate, MOD logs `Failed to evaluate javascript for '<uri>' plugin` to the
console and the plugin loads without its custom behaviour.

---

## 9. Starting from an existing design

MOD's `mod-sdk` repository ships a library of finished pedal templates and artwork — background
families (`boxy`, `boxy-small`, `british`, `japanese`, `lata`, plus combo, head and rack models),
knobs, sliders, lights and footswitches, in layouts from one knob up to twelve sliders.

Those templates are ordinary modguis. They use the same Mustache markup and the same `mod-role`
vocabulary described here, and you can copy one into your bundle and adapt it. **Keep `{{{cns}}}`
wherever it appears** — that is what makes the reserved class names legal (§5).

> `mod-sdk` also provides an interactive editor for building a modgui without writing markup. It is
> unmaintained and depends on PhantomJS, so this document does not cover running it. The templates
> and artwork remain useful regardless.

---

## 10. Checklist

Before shipping:

- [ ] `modgui:gui` declares all five required properties
- [ ] `mod-port-symbol` on every control matches an `lv2:symbol` exactly
- [ ] Every reserved class carries `{{{cns}}}` — triple braces in HTML, double in CSS
- [ ] Every image URL in the stylesheet carries `{{ns}}`
- [ ] Thumbnail is within 256 × 64 px
- [ ] The pedal renders as your design, not the default — check the browser console
- [ ] Every control moves the parameter it names, and the footswitch bypasses
- [ ] Adding a port to the plugin does not break the layout

---

## See also

- [`modgui:` specification](https://mod-audio.github.io/mod-ns/modgui/) — the canonical vocabulary
- [`mod:` specification](https://mod-audio.github.io/mod-ns/mod/) — plugin metadata: brand, label,
  ranges, per-device defaults
- [`METADATA.md`](METADATA.md) — how those affect what the GUI can show
- [`INSTALLING.md`](INSTALLING.md) — getting the bundle onto a device to look at it
