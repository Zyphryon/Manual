# Layouts (`.ui`)

A layout file holds **templates**: named trees of nodes that the game spawns as panes or under an element. This
chapter covers the file, how templates reuse each other with parameters, the fields every node takes, the
flexbox-like layout model, lengths and edges, bands, spawning from code, keyboard shortcuts and pointer flags, drag
and drop, tooltips, context menus, and what is remembered when a panel is spawned again. Each widget's own fields are
in `widgets.md`, and bindings (`{Key}`, `@Key`) in `binding.md`.

## 1. The file

```json
{
  "Templates": {
    "Heading": { "Widget": "Label", "Style": "Title" },
    "MainMenu": {
      "Layer": "Base", "Style": "Card", "Placement": "Absolute", "Left": 40, "Top": 40, "Width": 280,
      "Children": [
        { "Widget": "Heading", "Text": "@Menu.Title" },
        { "Widget": "Button",  "Text": "@Menu.Play",    "Action": "Menu.Play" },
        { "Widget": "Button",  "Text": "@Menu.Options", "Action": "Menu.Options" },
        { "Widget": "Button",  "Text": "@Menu.Quit",    "Action": "Menu.Quit" }
      ]
    }
  }
}
```

`Markup::Read(Source, Error)` reads a file's templates into the markup. Call it once per file. A template given again
under a name already read replaces the old one, so a later file can override an earlier one. A template may name
templates of any file read before it is **spawned**, not only before it is read.

A node is a JSON object. Its `Widget` names one of the library's widgets, a widget the game registered
(`Markup::Register`), or another template. A node with no `Widget` is a `Panel` (a plain box).

## 2. Templates and parameters

A node naming a template is that template, with the node's own fields laid over the template's root field by field:

```json
"Heading": { "Widget": "Label", "Style": "Title" },
{ "Widget": "Heading", "Text": "Inventory", "Margin": [0, 0, 0, 8] }
```

The node's `Children` are made **after** the template's own children, inside the same element (or the part the
widget names as its content).

A template can declare `Parameters` with defaults. A parameter reaches only the template fields whose whole value is
`"$Name"`. Fields the node gives that are parameters fill those places, and are not laid over the root:

```json
"Setting": {
  "Parameters": { "Caption": "Unnamed", "Start": 0.5 },
  "Flow": "Row", "Align": "Center", "Gap": 12,
  "Children": [
    { "Widget": "Label",  "Text": "$Caption", "Width": 70 },
    { "Widget": "Slider", "Value": "$Start", "Grow": 1 }
  ]
},

{ "Widget": "Setting", "Caption": "@Audio.Music", "Start": "{Audio.Music}" }
```

Rules:

- `"$Name"` must be the whole string. `"Level $Name"` is left as written.
- Only names declared in `Parameters` are replaced. An unknown `"$Other"` stays as text.
- A parameter's value may be a binding (`"{Audio.Music}"`) or a lexicon key (`"@Audio.Music"`), which then binds in
  the template's field.
- A template that holds itself, directly or through others, is left out with a warning.

## 3. Fields every node takes

| Field | Value | Meaning |
|---|---|---|
| `Widget` | name | The widget or template. `Panel` when left out. |
| `Name` | text | The name `Element::Find` looks up, and the key under which scroll and open state are remembered. |
| `Style` | style name | The style it wears. Each widget has a default (`Button`, `Label`, ...). |
| `Hidden` | `true`/`false` or a flag binding | Not laid out, drawn or hit. `"{!Hud.Ready}"` follows the model. |
| `Disabled` | `true`/`false` or a flag binding | Answers nothing, wears its `Disabled` variant. |
| `Action` | text | The action name `OnClick`, `OnChange` and `Bind` hear it by. |
| `Children` | list of nodes | Made inside it, or inside the part the widget names as content (a window's body). |
| `Layer` | band name | On a template's **root** only: the band its pane opens in. |
| layout fields | see section 4 | `Placement`, `Flow`, `Wrap`, `Align`, `Justify`, `Width`, `Height`, `Basis`, `Left`, `Top`, `Right`, `Bottom`, `Margin`, `Padding`, `Gap`, `Grow`, `Shrink`. |
| `Tooltip` | text or template name | Section 7. |
| `Drag`, `DragKind`, `DragTemplate`, `Drop` | | Section 8. |
| `ContextMenu` | items, a template name or a node | Section 9. |
| `Shortcuts` | object | Section 6. |
| `Hover`, `AnyButton`, `Typing` | `true`/`false` | Section 6. |

A field a node gives that neither the widget nor any module reads is ignored silently. Check spelling against
`widgets.md` when a field seems to do nothing.

## 4. The layout model

Layout is flexbox-like. Every element has a box. A box lays its children along its `Flow`.

| Field | Values | Default | Meaning |
|---|---|---|---|
| `Placement` | `Flow`, `Absolute` | `Flow` | `Absolute` stands over its siblings, placed by `Left`, `Top`, `Right`, `Bottom` from the parent's edges. |
| `Flow` | `Column`, `Row` | `Column` | The axis children are laid along. |
| `Wrap` | `true`/`false` | `false` | Children that overrun the axis start a new line. |
| `Align` | `Start`, `Center`, `End`, `Stretch` | `Stretch` | Where children sit across the axis. |
| `Justify` | `Start`, `Center`, `End`, `Between` | `Start` | How the room left along the axis is shared. |
| `Width`, `Height` | length | content | The box's size. |
| `Basis` | length | own size | The size along the parent's axis it grows or shrinks from. |
| `Left`, `Top`, `Right`, `Bottom` | length | free | Distances from the parent's edges, for `Absolute`. Give `Left` and `Right` to stretch across. |
| `Margin` | edges | 0 | Room outside the box. |
| `Padding` | edges | 0 | Room inside the box. |
| `Gap` | number | 0 | Room between children, and between wrapped lines. |
| `Grow` | number | 0 | Share of the room left along the parent's axis it takes. |
| `Shrink` | number | 1 | Share of an overrun it gives up, weighed by its size. |

### Lengths

| Written | Means |
|---|---|
| `120` | 120 units. |
| `"50%"` | Half of the parent's room. |
| `"Auto"` | The size the content asks for. |
| `"Fit 300"` | The size the content asks for, up to 300 units, the content scrolling past it. `"Fit"` alone is `"Auto"`. |

### Edges

`Margin`, `Padding` and a style's `Padding` take one number for all sides, `[across, down]`, or
`[left, top, right, bottom]`.

### Units

Every length is in units of the interface. A unit covers the display scale times the player's scale in pixels (see
`getting-started.md`), so the same layout is the same size on a 4K monitor and a laptop.

### Patterns

```json
{ "Flow": "Row", "Align": "Center", "Gap": 8, "Children": [
    { "Widget": "Label", "Text": "Name", "Grow": 1 },
    { "Widget": "Button", "Text": "Edit" }
] }

{ "Flow": "Row", "Justify": "Between", "Children": [ { "Widget": "Label", "Text": "Left" },
                                                     { "Widget": "Label", "Text": "Right" } ] }

{ "Placement": "Absolute", "Right": 16, "Bottom": 16, "Widget": "Label", "Text": "v1.0" }

{ "Placement": "Absolute", "Left": "20%", "Right": "20%", "Bottom": 40 }

{ "Flow": "Row", "Wrap": true, "Gap": 6, "Children": [ "... many chips ..." ] }
```

A label overlaid on a bar, filling it:

```json
{ "Height": 18, "Children": [
    { "Widget": "Progress", "Value": "{Hud.Health}", "Labelled": false, "Width": "100%" },
    { "Widget": "Label", "Text": "{Hud.HealthNow:0}", "Style": "Hud.Figures",
      "Placement": "Absolute", "Left": 0, "Right": 0, "Top": 0, "Bottom": 0 }
] }
```

## 5. Bands and spawning

`Layer` on a template's root picks the band its pane opens in:

| Name | Use |
|---|---|
| `Base` | HUDs, menus behind windows. |
| `Window` | Default when no `Layer` is given. |
| `Popup` | Panes that close on a press outside or on escape. |
| `Modal` | Panes that hold pointer and keyboard over a backdrop until hidden. |
| `Tooltip`, `Drag` | Overlays, never hit. |
| your own | `Layers::Declare("Hud", Layer { 40, Layer::Trait::Cycled })` in code first. |

An unknown band name is ignored, and the pane opens in the default band.

A root naming a `Window` or `Dialog` opens its own pane (windows in `Window`, dialogs in `Modal`), so `Layer` on such
a root does nothing. A `Window` or `Dialog` written deeper in a tree also opens its own pane.

Spawning from code:

| Call | Does |
|---|---|
| `Markup::Spawn(Interface, "Name")` | Opens a pane and makes the template in it. Returns the root (a window for a window template), or an element naming nothing. |
| `Markup::Spawn(Parent, "Name")` | Makes the template as a child of an element. |
| `Markup::Spawn(Parent, "Name", Scope, Within)` | Makes it reading the record `Scope` (see `binding.md`). |
| `Markup::HasTemplate("Name")` | `true` once a template of that name was read. |
| `Markup::Build(Parent, Node)` | Makes one node written in code as JSON. |

Spawn a template as many times as you want. Each spawn is a separate copy.

```cpp
mHud  = mMarkup->Spawn(mInterface, "Hud");
mBag  = mMarkup->Spawn(mInterface, "Bag");
Window::Close(mInterface, mBag.GetHandle());        // windows open shown, so hide it until asked for
const Element Line = mMarkup->Spawn(mLog, "ChatLine");
```

## 6. Shortcuts and pointer flags

`Shortcuts` maps a key chord to an action:

```json
{ "Name": "Map", "Hover": true, "Shortcuts": {
    "R":      "Map.Reset",
    "Ctrl+Z": { "Action": "Edit.Undo", "Scope": "Anywhere", "Typing": true, "Blur": true },
    "Equal":  { "Action": "Map.Zoom", "Repeat": true }
} }
```

| Key in the object | Value | Default | Meaning |
|---|---|---|---|
| `Action` | text | | The action run, as `Bind` binds one. Required. |
| `Scope` | `Anywhere`, `Hover`, `Focus` | `Hover` | Where the pointer or keyboard must be: anywhere while usable, over the element, or the element holding the keyboard. |
| `Typing` | `true`/`false` | `false` | Run even while a text field holds the keyboard. |
| `Blur` | `true`/`false` | `false` | Take the keyboard from a text field before running. |
| `Repeat` | `true`/`false` | `true` | Run again on each key repeat. |

A plain string value is the action, with the defaults. Chords join modifiers (`Ctrl` or `Control`, `Shift`, `Alt`) and a key with `+` (`Ctrl+Shift+Z`). The key is a
name of `ZyInput::Key` (`A`, `F2`, `Space`, `Escape`, `Delete`, `PageUp`, `Equal`...) or a single digit. The
modifiers must match exactly. Shortcuts run before the control holding the keyboard hears the key.

In code, `Interface::AddShortcut(Shortcut("Ctrl+S", "File.Save"))` adds one that runs anywhere, and
`AddShortcut(Entry, Element.GetHandle())` one an element keeps. `RemoveShortcut(Key, Modifiers, Owner)` removes them.

Pointer flags make a node a control that asks for more of the pointer:

| Field | Meaning |
|---|---|
| `Hover` | The control hears `Signal::kEnter`, `kMove` and `kLeave`. A node's `Shortcuts` with `Scope` `Hover` need the pointer to rest on it. |
| `AnyButton` | Every pointer button presses, drags and clicks it, named by `Event::Button`. Otherwise only the left button does. |
| `Typing` | Holding the keyboard counts as typing, so `Interface::IsTyping()` is `true` (for a game-made text control). |

## 7. Tooltips

```json
{ "Widget": "Button", "Text": "Attack", "Tooltip": "Strike the nearest foe" }
{ "Widget": "Button", "Text": "Gold",   "Tooltip": "You carry {Player.Gold:0} gold" }
{ "Widget": "Button", "Text": "Help",   "Tooltip": "@Help.Tip" }
{ "Widget": "Button", "Text": "Gate",   "Tooltip": "GateTip" }
```

A `Tooltip` string that names a template the markup knows makes the tooltip from that template. Anything else is the
text shown, which may bind values and lexicon keys. A template tooltip lets you lay out a title and a body:

```json
"GateTip": { "Gap": 2, "Width": 220, "Children": [
  { "Widget": "Label", "Style": "Tooltip.Title", "Text": "@Gate.Title" },
  { "Widget": "Label", "Style": "Tooltip.Label", "Text": "@Gate.Text" }
] }
```

Tooltips wear `Tooltip` and `Tooltip.Label`, show once the pointer rests (half a second unless
`Tooltip::SetDelay(Interface, Seconds)` says otherwise), stay on screen, and hide when the pointer leaves or presses.
In code: `Tooltip::SetText(Element, Text)`, `Tooltip::SetTemplate(Element, Markup, Template)`,
`Tooltip::Clear(Element)`.

## 8. Drag and drop

| Field | Value | Meaning |
|---|---|---|
| `Drag` | text, number, flag, or a binding | The element can be picked up, carrying this value (its payload). |
| `DragKind` | text | The kind of thing it is. Targets accept drags by kind. |
| `DragTemplate` | template name | What the ghost under the pointer is made of. A copy of the element otherwise. |
| `Drop` | a kind, a list of kinds, or `true` | Drags of those kinds, or of any kind, can be dropped here. |

Any of `Drag`, `DragKind` or `DragTemplate` makes the element a source.

```json
"Slot": { "Style": "Slot", "Drag": "{Name}", "DragKind": "Item", "Drop": ["Item", "Rune"],
          "Children": [ { "Widget": "Label", "Text": "{Count:0}" } ] }
```

A press that moves four pixels lifts the source. It wears `Dragging`, a ghost wearing `Ghost` follows the pointer in
the `Drag` band, and a target accepting the kind wears `DropHover`. Escape, or a release over no target, cancels.

The events (`<UI.Gesture/Drag.hpp>`):

| Signal | Sent to | `Event::Other` |
|---|---|---|
| `Drag::kStart` | the source | |
| `Drag::kEnter`, `Drag::kLeave` | a target | the source |
| `Drag::kDrop` | the target | the source |
| `Drag::kEnd` | the source | |

The events bubble, so one observer on a list hears drops on all its rows. `Drag::GetPayload(Source)` reads what the
source carries (a `Datum`). Code: `Drag::SetSource(Element, Kind, Payload)`, `Drag::SetTarget(Element, Kinds)`,
`Drag::SetPayload`, `Drag::SetGhost`, `Drag::Cancel(Interface)`. See the inventory in `cookbook.md`.

## 9. Context menus

`ContextMenu` opens a menu at the pointer on a right press. It takes a list of items, the name of a template whose
root is a `Menu`, or a `Menu` node written in place:

```json
{ "Widget": "Listbox", "Items": [ "Fireball", "Heal" ], "ContextMenu": [
    { "Text": "Cast", "Action": "Spell.Cast", "Shortcut": "Enter" },
    { "Text": "Assign to bar", "Items": [
        { "Text": "Slot 1", "Action": "Spell.Slot1" },
        { "Text": "Slot 2", "Action": "Spell.Slot2" } ] },
    "-",
    { "Text": "Forget", "Disabled": true }
] }
```

Menu items are described in `widgets.md` (Menu). Picking an item runs its action and closes the menus.
`Menu::GetSubject(Item)` gives the element the menu was opened over, so one action serves every row. A menu written
in a list row reads the row's record, so its items can bind to it.

## 10. What is kept when a panel is made again

Destroying a pane and spawning its template again normally starts it fresh. Named parts remember how they were left:

| Widget given a `Name` | Remembers |
|---|---|
| `Scroller`, `Repeater`, `Grid`, `Outline` | Its scroll offset. |
| `Expander` | Whether it was open. A bound `Expanded` still wins. |

The key is the `Name` within the scope it was made in, so two rows' expanders named alike are kept apart.
`Splitter::Save` and `Dock::Save` keep sizes and docking across runs as text you store yourself.

## 11. Common mistakes

| Mistake | What happens | Do instead |
|---|---|---|
| `"Layer": "Modal"` on a `Window` root | Ignored. | Use a `Dialog`, or a plain root with `"Layer": "Modal"`. |
| `"Text": "Level $Level"` | The text stays as written. | Bind instead: `"Text": "Level {Player.Level:0}"`. |
| A parameter not in `Parameters` | `"$Name"` stays text, and the node's field lays over the root instead. | Declare every parameter with a default. |
| `"Layer": "Hud"` with no `Layers::Declare` | Opens in the `Window` band among the windows. | Declare the band first, or use `Base`. |
| Giving `Left` without `"Placement": "Absolute"` | No effect on a flowing element. | Set `Placement`. |
| `Hidden: "{Ui.Show}"` meaning "show" | Hidden while the flag is set. | `"{!Ui.Show}"`. |
| `Tooltip` text equal to a template's name | The template is made instead of the text. | Name templates distinctly (`...Tip`). |
| Spawning a window and expecting it hidden | Windows and dialogs open shown. | `Window::Close` or `Dialog::Close` right after `Spawn`. |

## 12. Where it lives

| What | Path |
|---|---|
| Node fields, spawning | `Module/UI.Markup/Source/UI.Markup/Markup.cpp`, `Markup.hpp` |
| Templates and parameters | `Module/UI.Markup/Source/UI.Markup/Template.cpp` |
| Lengths, edges, bands, enums | `Module/UI.Markup/Source/UI.Markup/Parser.cpp`, `Module/UI.Core/Source/UI.Core/Facet/Layout/Box.hpp`, `Types.hpp` |
| Shortcuts and pointer flags | `Module/UI.Gesture/Source/UI.Gesture/Input.cpp`, `Module/UI.Core/Source/UI.Core/Shortcut.hpp` |
| Drag and drop, tooltips | `Module/UI.Gesture/Source/UI.Gesture/Drag.cpp`, `Tooltip.cpp` |
| Context menus | `Module/UI.Control/Source/UI.Control/Widget/Menu.cpp` (`Menu::Read`) |
| Remembered state | `Module/UI.Control/Source/UI.Control/Widget/Memory.hpp` |
| Example layout | `Example/Resources/Layout/Controls.ui` |
