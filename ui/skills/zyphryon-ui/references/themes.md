# Themes (`.theme`)

A theme decides how everything looks. Nothing in ZyUI paints itself: every widget and every part of a widget wears a
named **style** (`Button`, `Button.Label`, `Checkbox.Mark`), and the theme says what that style looks like, in each
state, with transitions and animations. This chapter covers the file format, every style field, states and variants,
transitions, keyframed animations, images, sounds, how themes stack, and the full list of built-in style names.

## 1. How themes stack

The library carries its own themes, embedded in the binaries:

| Layer | File | Gives |
|---|---|---|
| Gesture | `Module/UI.Gesture/Theme/Default.theme` | `Ghost`, `Tooltip`, `Tooltip.Label`, the colour `Edge`, the animations `FadeIn`, `FadeOut`. |
| Control | `Module/UI.Control/Theme/Default.theme` | Every widget's styles, the colours `Ink`, `Gold`, `Field`, `Accent`, `Raised`, the animations `Drop`, `Pop`, `Pulse`, `Spin`, the font slot `Body`. |
| Your game | your `.theme` | Only what you change. |

`ControlModule::Declare(Markup)` adds the two built-in layers to the markup. Then:

```cpp
Str Error;
Markup.Dress(Theme, Error);               // the built-in layers alone
Markup.Dress(Theme, GameText, Error);     // the built-in layers, then the game's over them
```

Both return `false` and fill `Error` when a file has a mistake, and the theme keeps what it held. Call `Dress` into
the same `Theme` the interface was given with `Interface::SetTheme`, and every element restyles.

What a game theme does to the layers under it:

- Its **colours, fonts and images** replace those of the same name, and reach the built-in styles too. Redefine
  `Gold` and every built-in style naming `Gold` changes.
- Its **animations** replace those of the same name.
- A **style** it names merges over the built-in one **field by field**. A `When` variant merges over the built-in
  variant of the same states.
- A **font given `null`** is a slot: it holds nothing until `Markup::SetFont(Name, Font)` or a later layer fills it, and
  `null` never empties a filled one.

## 2. The file

A theme is one JSON object with up to five tables, each optional:

```json
{
  "Fonts":      { "Body": "Resources://Font/Body.fnt", "Heading": "Resources://Font/Heading.fnt" },
  "Colors":     { "Gold": "#F0D98F", "Blood": "#B5402F", "Clear": "#00000000" },
  "Images":     { "Frame": "Resources://Image/Frame.tex" },
  "Animations": { "Throb": { "Duration": 0.8, "Repeat": 0, "Alternate": true, "Easing": "InOutSine",
                             "Keys": [ { "At": 0, "Opacity": 1 }, { "At": 1, "Opacity": 0.6 } ] } },
  "Styles":     {
    "Label":  { "Font": "Body", "Size": 16, "Tint": "Ink" },
    "Button": { "Background": "#2E3645", "Radius": 6, "Padding": [12, 6], "Transition": 0.12,
                "When": { "Hover": { "Glow": { "Color": ["Gold", 0.18], "Blur": 8 } },
                          "Press": { "Scale": 0.97 },
                          "Focus": { "Border": "Gold", "BorderWidth": 1 },
                          "Disabled": { "Opacity": 0.5 } } }
  }
}
```

| Table | Maps | Value |
|---|---|---|
| `Fonts` | a font name | The address of a `.fnt`, or `null` for a slot. |
| `Colors` | a palette name | A colour (below). Palette names may then be used wherever a colour goes. |
| `Images` | an image name | The address of a `.tex`. |
| `Animations` | an animation name | An animation object (section 6). |
| `Styles` | a style name | A style object (section 3). |

Any other top-level key, or a table that is not an object, is an error.

### Colours

| Written | Means |
|---|---|
| `"#RRGGBB"` | Opaque. |
| `"#RRGGBBAA"` | With alpha. |
| `"Gold"` | A palette colour. |
| `[r, g, b]`, `[r, g, b, a]` | Components from 0 to 1. |
| `["Gold", 0.18]`, `["#F0D98F", 0.18]` | A colour with an alpha of its own. A palette name keeps following the palette when a later theme changes it. |

Palette names work inside themes only. An `Image` node's `Tint` in a layout reads hex and arrays, not palette names.

## 3. Style fields

A style is an object of fields. Every field is optional. Fields a style does not give come from its `Base`, then
from the element's own facets.

### Box and size

| Field | Value | Meaning |
|---|---|---|
| `Base` | style name | The style to start from. Chains are allowed. |
| `Background` | colour | Fill of the box. |
| `Border` | colour | Outline colour. |
| `BorderWidth` | number | Outline width, in units. |
| `Radius` | number | Corner radius, in units. |
| `Opacity` | 0 to 1 | Fades the element and all it holds. |
| `Padding` | edges | Room inside the box: one number, `[across, down]` or `[left, top, right, bottom]`. |
| `Gap` | number | Room between children. |
| `Width`, `Height` | length | A size: units, `"50%"`, `"Auto"` or `"Fit 300"` (see `layouts.md`). |

A layout node's own `Width`, `Padding` and so on are written into the element; a style's sizes apply on top while it
is worn. Margins, flow, alignment and placement are layout-only (they are not style fields).

### Text

| Field | Value | Meaning |
|---|---|---|
| `Font` | font name | A name from `Fonts`. |
| `Size` | number | Text size, in units. |
| `Tint` | colour | Text colour. |
| `Alignment` | `Start`, `Center`, `End`, `Stretch` | Where lines sit in the box. |
| `Stroke` | colour | An outline around each glyph. |
| `StrokeWidth` | number | Its width. |
| `Shadow` | halo | On a text style, shadows the text. |

### Fills of value widgets

| Field | Value | Meaning |
|---|---|---|
| `Fill` | colour | The filled share of a `Progress` or `Slider`. |
| `Thumb` | colour | A slider's thumb, and the thumb of a scrolling element's bars. |

### Effects

| Field | Value | Meaning |
|---|---|---|
| `Gradient` | `{ "From": c, "To": c, "Angle": deg }` or `["#a", "#b"]` | A fill over `Background`. `0` runs upward, `90` to the right, `180` (default) downward. |
| `Shadow` | `{ "Color": c, "Offset": [x, y], "Blur": n, "Spread": n }` | A soft box behind the element. `"Shadow": { }` removes an inherited one. |
| `Glow` | same as `Shadow` | A halo that never moves, ringing the element. |
| `InnerGlow` | `{ "Color": c, "Blur": n }` | A glow just inside the edge. |
| `Scale` | `n` or `[x, y]` | Drawn larger or smaller about its middle. |
| `Rotate` | degrees | Drawn turned clockwise about its middle. |
| `Translate` | `[x, y]` | Drawn moved, in pixels. |

`Scale`, `Rotate` and `Translate` change only how the element is drawn. Layout and clicks keep the untransformed box.

### Motion

| Field | Value | Meaning |
|---|---|---|
| `Transition` | seconds, or `{ "Duration": s, "Delay": s, "Easing": name }` | How the element moves to a new state's values. A bare number eases with `OutQuad`. |
| `Animation` | animation name | Played while the style is worn. |
| `Enter` | animation name | Played once when the element appears. |
| `Exit` | animation name | Played once when the element is hidden. While it plays, the element is drawn but cannot be hit, focused or used. |

### Pictures and sounds

| Field | Value | Meaning |
|---|---|---|
| `Image` | an address, an `Images` name, `""`, or an object | Lays a picture over the box without changing its size. `""` removes an inherited one. |
| `Sounds` | `{ "Hover", "Press", "Click", "Focus", "Change", "Open", "Close" }` | A sound address per moment (section 8). Only on the style itself, never in a `When` variant. |
| `When` | `{ "State": { fields }, "State+State": { fields } }` | Variants worn in states (section 4). |

An `Image` object takes:

| Key | Value |
|---|---|
| `Source` | An address or an `Images` name. |
| `Region` | `[u0, v0, u1, v1]`, the part of the texture shown, in 0 to 1. |
| `Slice` | Nine-slice borders in pixels of the image: one number, `[across, down]` or four. |
| `Tint` | A colour multiplied over the picture. |

A variant may change one member and keep the rest:

```json
"Framed": {
  "Base": "Button", "Background": "#00000000", "Shadow": { }, "Padding": [22, 12],
  "Image": { "Source": "Door", "Region": [0, 0, 0.5, 1], "Slice": 18, "Tint": "#A89F8F" },
  "Transition": 0.15,
  "When": {
    "Hover": { "Image": { "Tint": "#FFE9A8" } },
    "Press": { "Image": { "Tint": "#FFFFFF", "Region": [0.5, 0, 1, 1] } }
  }
}
```

A field the reader does not know (`"no skin has such a field"`) or a value it cannot take fails the whole `Dress`.

## 4. States and variants

An element is in a set of states, one bit each. A style's `When` gives variants worn while the element is in those
states. Join states with `+` for a variant worn only when all hold: `"Hover+Checked"`.

| State | Declared by | Worn when |
|---|---|---|
| `Hidden` | the core | Not laid out, drawn or hit, with all it holds. |
| `Disabled` | the core | Answers nothing. |
| `Hover` | the core | The pointer rests on it or on something it holds. |
| `Press` | the core | Held down by the pointer. |
| `Focus` | the core | Holding the keyboard. |
| `Selected` | `Widget::Selected()` | A picked list item, row, tab or entry. |
| `Checked` | `Widget::Checked()` | A checked box or radio, an open expander, a toggled button, a checked menu item. |
| `Marked` | `Repeater::Marked()` | A row flag the game sets through `RowStates`. |
| `Dragging` | `Drag::Dragging()` | A drag source while it is carried. |
| `DropHover` | `Drag::DropHover()` | A drop target while an accepted drag rests over it. |

**Parts take their control's states.** A button's label is `Hover` while the button is, so `Button.Label` can light up
with `"When": { "Hover": { "Tint": "Gold" } }`.

**A state must be declared before the theme that names it is read.** `ControlModule::Declare` declares all of the
above. A state of your own (`Worn`, `Running`) is declared by name:

```cpp
States::Declare("Worn");         // <UI.Core/States.hpp>, before Markup::Dress
```

A `RowStates` entry in a layout also declares its state, but only when the list is built, which is after the theme
was read. Declare such states in code first, or use `Marked`.

## 5. Transitions

When an element changes state, style or theme, and the style being **entered** gives a `Transition`, these ease to
the new values: colours, opacities, radii, borders, gradients, halos (shadow, glow, inner glow) and transforms. Sizes,
padding, fonts and alignment change at once.

A transition in the normal style applies into and out of every state, since every state's variant is worn over it.

```json
"Card":    { "Background": "#FFFFFF0F", "Transition": { "Duration": 0.2, "Easing": "OutCubic" },
             "When": { "Hover": { "Background": "#FFFFFF1A" } } },
"Warning": { "Base": "Label", "Tint": "#E0B040", "Transition": { "Duration": 0.3, "Delay": 0.1 } }
```

Easings are the names of `ZyMath::Easing`, including `Linear`, `InQuad`, `OutQuad`, `InOutQuad`, `InCubic`,
`OutCubic`, `InOutCubic`, `Smoothstep`, `InSine`, `OutSine`, `InOutSine`, `InExpo`, `OutExpo`, `InCirc`, `OutCirc`,
`InElastic`, `OutElastic`, `InOutElastic`, `InBounce`, `OutBounce`, `InOutBounce`, `InBack`, `OutBack`.

## 6. Animations

An animation is CSS's `@keyframes`:

| Field | Value | Meaning |
|---|---|---|
| `Duration` | seconds | One run. |
| `Repeat` | count | Runs played. `0` plays forever. |
| `Alternate` | `true`/`false` | Plays every second run backward. |
| `Easing` | easing name | The curve of each run. |
| `Keys` | list of keys | Each key has `At` (`0` to `1`, or `"50%"`) and any style field to animate. |

A key may not give `Base`, `When`, `Sounds`, `Transition`, `Animation`, `Enter` or `Exit`.

The built-in animations:

| Name | Duration | Does |
|---|---|---|
| `FadeIn` | 0.15 | Opacity 0 to 1. |
| `FadeOut` | 0.12 | Opacity 1 to 0. |
| `Drop` | 0.16 | Fades in while sliding down 6 units. |
| `Pop` | 0.18 | Fades in while growing from 0.92 (`OutBack`). |
| `Pulse` | 1.2, forever | Scale 1 to 1.06 and back. |
| `Spin` | 1, forever | One full turn, linear. |

```json
"Animations": {
  "Shake": { "Duration": 0.3, "Easing": "Linear", "Keys": [
    { "At": 0, "Translate": [0, 0] }, { "At": "25%", "Translate": [-4, 0] },
    { "At": "75%", "Translate": [4, 0] }, { "At": 1, "Translate": [0, 0] } ] }
},
"Styles": {
  "Window":  { "Enter": "Pop", "Exit": "FadeOut" },
  "Badge":   { "Background": "#E84D3D", "Radius": 6, "Width": 12, "Height": 12, "Animation": "Pulse" },
  "Error":   { "Base": "Label", "Tint": "#FF6B5A", "Enter": "Shake" },
  "Loading": { "Base": "Label", "Animation": "Spin" }
}
```

## 7. Naming your own styles

A widget given `"Style": "Shop"` wears `Shop`, and its parts wear `Shop.` plus the part name (`Shop.Label`,
`Shop.Item`). So a style of your own needs its parts too, each starting from the built-in part:

```json
"Shop":       { "Base": "Repeater" },
"Shop.Item":  { "Base": "Repeater.Item", "When": { "Selected": { "Background": ["Gold", 0.2] } } },
"Buy":        { "Base": "Button", "Background": "#2F6B3A" },
"Buy.Label":  { "Base": "Button.Label", "Tint": "#E8FFE8" }
```

Without `Buy.Label`, a `Button` styled `Buy` has a label wearing a style nobody defined.

To change the look of every button, restyle `Button` itself instead. Your fields merge over the built-in ones.

## 8. Sounds

```json
"Button":   { "Sounds": { "Hover": "Resources://Sound/Hover.snd", "Click": "Resources://Sound/Click.snd" } },
"Checkbox": { "Sounds": { "Change": "Resources://Sound/Tick.snd" } },
"Window":   { "Sounds": { "Open": "Resources://Sound/Open.snd", "Close": "Resources://Sound/Close.snd" } },
"Quiet":    { "Base": "Button", "Sounds": { "Click": "" } }
```

| Moment | Played when |
|---|---|
| `Hover` | The pointer comes to rest on the control. |
| `Press` | The control is pressed. |
| `Click` | The control is clicked. |
| `Focus` | The control takes the keyboard. |
| `Change` | The control's value changes. |
| `Open` | The pane the element is the root of is shown. |
| `Close` | The pane the element is the root of is hidden. |

A style without a sound plays its base's. An empty address silences it. The interface loads each sound the first time
it is named, through the content service of the host given to `Interface::Attach`, and plays it on the audio
service's `Interface` submix. Sounds are `.snd` files baked by Studio. A game's own writes to the model never play
sounds.

## 9. Built-in style names

Every name in `Module/UI.Control/Theme/Default.theme` and `Module/UI.Gesture/Theme/Default.theme`, by widget:

| Widget | Styles |
|---|---|
| Text | `Label`, `Title` (a larger gold label), `Card` (a framed panel for your own use) |
| `Button` | `Button`, `Button.Label` |
| `Checkbox` | `Checkbox`, `Checkbox.Box`, `Checkbox.Mark`, `Checkbox.Label` |
| `Radio` | `Radio`, `Radio.Box`, `Radio.Mark`, `Radio.Label` |
| `Slider` | `Slider`, `Slider.Thumb` |
| `Progress` | `Progress`, `Progress.Label` |
| `Spinner` | `Spinner`, `Spinner.Decrease`, `Spinner.Decrease.Label`, `Spinner.Increase`, `Spinner.Increase.Label`, `Spinner.Label` |
| `Scalar` | `Scalar`, `Scalar.Tag`, `Scalar.Field`, `Scalar.Field.Text`, `Scalar.Field.Hint`, `Scalar.Field.Caret`, `Scalar.Field.Selection`, tag colours `Axis.X`, `Axis.Y`, `Axis.Z`, `Axis.W` |
| `Swatch` | `Swatch` |
| `Textbox` | `Textbox`, `Textbox.Text`, `Textbox.Hint`, `Textbox.Caret`, `Textbox.Selection` |
| `Listbox` | `Listbox`, items `ListItem`, `ListItem.Label` |
| `Combobox` | `Combobox`, `Combobox.Label`, `Combobox.Arrow`, `Combobox.Popup`, `Combobox.List` |
| `Repeater` | `Repeater`, rows `Repeater.Item` (which starts from `ListItem`) |
| `Tabs` | `Tabs`, `Tabs.Header`, `Tabs.Tab`, `Tabs.Tab.Label`, `Tabs.Close`, `Tabs.Close.Label`, `Tabs.Pages` |
| `Expander` | `Expander`, `Expander.Header`, `Expander.Header.Label`, `Expander.Arrow`, `Expander.Content` |
| `Window` | `Window`, `Window.Bar`, `Window.Title`, `Window.Close`, `Window.Close.Label`, `Window.Content`, `Window.Edge`, `Window.Grip` |
| `Dialog` | `Dialog`, `Dialog.Bar`, `Dialog.Title`, `Dialog.Close`, `Dialog.Close.Label`, `Dialog.Content`, `Dialog.Body`, `Dialog.Message`, `Dialog.Buttons`, `Dialog.Button`, `Dialog.Button.Label`, `Dialog.Default`, `Dialog.Default.Label` |
| Modal panes | `Modal.Backdrop` |
| `Menu` | `Menu`, `Menu.Item`, `Menu.Item.Label`, `Menu.Shortcut`, `Menu.Arrow`, `Menu.Separator`, `Menu.Icon`, `Menu.Check`, `Menu.Check.Mark`, `Menu.Heading`, `Menu.Heading.Label` |
| `Menubar` | `Menubar`, `Menubar.Title`, `Menubar.Title.Label`, `Menubar.Trail` |
| `Dropdown` | `Dropdown`, `Dropdown.Label`, `Dropdown.Arrow` |
| `Scrollbar` | `Scrollbar`, `Scrollbar.Thumb` |
| `Scroller` | `Scroller`, `Scroller.Content`, `Scroller.Bar`, `Scroller.Bar.Thumb` |
| `Splitter` | `Splitter`, `Splitter.Pane`, `Splitter.Bar` |
| `Grid` | `Grid`, `Grid.Viewer`, `Grid.Viewer.Content`, `Grid.Viewer.Bar`, `Grid.Viewer.Bar.Thumb`, `Grid.Header`, `Grid.Heading`, `Grid.Heading.Label`, `Grid.Sort`, `Grid.Grip`, `Grid.List`, `Grid.List.Item`, `Grid.Cell`, `Grid.Cell.Label` |
| `Outline` | `Outline`, `Outline.List`, `Outline.List.Item`, `Outline.Marker`, `Outline.Indent`, `Outline.Arrow`, `Outline.Content`, `Outline.Label` |
| `Dock` | `Dock`, `Dock.Split`, `Dock.Split.Pane`, `Dock.Split.Bar`, `Dock.Region`, `Dock.Region.Header`, `Dock.Region.Tab`, `Dock.Region.Tab.Label`, `Dock.Region.Pages`, `Dock.Region.Page`, `Dock.Panel`, `Dock.Window`, `Dock.Window.Bar`, `Dock.Window.Title`, `Dock.Window.Close`, `Dock.Window.Close.Label`, `Dock.Window.Content`, `Dock.Overlay`, `Dock.Target`, `Dock.Preview` |
| Gestures | `Ghost` (the drag image), `Tooltip`, `Tooltip.Label` |

The palette the built-in styles use: `Ink` (text), `Gold` (highlights and the focus ring), `Field` (input
backgrounds), `Accent` (fills and the default dialog button), `Raised` (buttons), `Edge` (borders). Redefining these
six recolours the whole library.

## 10. A complete game theme

A dark theme with a parchment-gold accent, one font, a framed window and a pulsing call to action:

```json
{
  "Fonts":  { "Body": "Resources://Font/Body.fnt" },
  "Colors": {
    "Ink": "#EDE3CF", "Gold": "#E7BC72", "Accent": "#8C5A2B", "Raised": "#2B2420",
    "Field": "#17120F", "Edge": "#5A4630", "Clear": "#00000000"
  },
  "Images": { "Frame": "Resources://Image/Frame.tex" },
  "Styles": {
    "Label":        { "Font": "Body", "Size": 16, "Tint": "Ink" },
    "Title":        { "Base": "Label", "Size": 24, "Tint": "Gold", "Stroke": "#000000", "StrokeWidth": 1 },
    "Button":       { "Radius": 4, "Gradient": { "From": "#3A2F27", "To": "#241D18" },
                      "When": { "Hover": { "Glow": { "Color": ["Gold", 0.25], "Blur": 10 } } } },
    "Window":       { "Image": { "Source": "Frame", "Slice": 24 }, "Background": "#120E0BF0", "Radius": 2 },
    "Window.Title": { "Base": "Label", "Tint": "Gold", "Size": 18 },
    "Primary":      { "Base": "Button", "Background": "Accent", "Animation": "Pulse" },
    "Primary.Label":{ "Base": "Button.Label", "Tint": "#FFF6E0" },
    "Danger":       { "Base": "Label", "Tint": "#D9503F" }
  }
}
```

## 11. Common mistakes

| Mistake | What happens | Do instead |
|---|---|---|
| No `Body` font anywhere | Every label draws nothing. | Give `Fonts.Body`, or `Markup::SetFont("Body", Font)`. |
| `Sounds` inside `When` | `Dress` fails: a variant cannot give sounds. | Give sounds on the style itself. |
| A variant naming an undeclared state | `Dress` fails: the variant names a state there is not. | `States::Declare` it before `Dress`. |
| A style of your own with no parts | Its label or rows wear styles nobody defined. | Define `Name.Label`, `Name.Item` with a `Base`. |
| `Margin` or `Flow` in a style | `Dress` fails: no skin has such a field. | Put layout fields in the layout node. |
| Expecting `Transition` to ease `Width` | Sizes change at once. | Animate `Scale` or `Opacity` instead. |
| A palette name in a layout `Image` node's `Tint` | Logged, the picture shows untinted. | Use hex there, or tint through a style. |
| Dressing a second `Theme` object | The interface keeps dressing from the first. | `Dress` into the theme given to `SetTheme`, or call `SetTheme` again. |

## 12. Where it lives

| What | Path |
|---|---|
| Theme reader (every field, every error) | `Module/UI.Markup/Source/UI.Markup/Interpreter.cpp` |
| Value parsers (colours, halos, gradients, images, lengths) | `Module/UI.Markup/Source/UI.Markup/Parser.cpp` |
| Style, skin, sound moments | `Module/UI.Core/Source/UI.Core/Style/Style.hpp`, `Skin.hpp` |
| State names | `Module/UI.Core/Source/UI.Core/States.hpp` |
| Built-in themes | `Module/UI.Control/Theme/Default.theme`, `Module/UI.Gesture/Theme/Default.theme` |
| Example theme | `Example/Resources/Theme/Example.theme` |
