# Data binding, lists and actions

The game never pokes widgets to show its state. It writes values into a **model**, layouts bind fields to those
values, and the next `Interface::Update` shows what changed. The user's edits flow back into the model, and controls
report what happened through **events** and **actions**. This chapter covers the model, the binding syntax, scopes,
item lists, `Mirror`, binding in code, and every way to run code on what the user does.

## 1. The model

`ZyPlugin::UI::Model` (`<UI.Data/Model.hpp>`) holds values under dotted keys. A value is a flag, a number, a text or a
colour. A value may also be a **list** of records, each a model of its own.

```cpp
mModel.SetBool("Hud.Ready", true);
mModel.SetNumber("Player.Gold", 120.0);             // Real64
mModel.SetText("Player.Name", "Rowan");
mModel.SetColor("Player.Banner", Color(0.8f, 0.2f, 0.2f, 1.0f));
mModel.Clear("Player.Banner");                      // holds nothing again

const Bool   Ready = mModel.GetBool("Hud.Ready");
const Real64 Gold  = mModel.GetNumber("Player.Gold", 0.0);
const Str    Name  = mModel.GetText("Player.Name");
```

Point the interface at it once:

```cpp
Binder::SetModel(mInterface, AddressOf(mModel));    // <UI.Data/Binder.hpp>
```

**Writes are cheap.** Setting a value equal to the one held does nothing, and only bindings whose values changed are
shown again. Writing every frame is fine.

**Game writes are silent.** No observer, action or sound hears a change the game makes. A change the user makes (a
slider dragged, a box checked, a field typed in) is written to the model **before** any observer or action runs, so
your `OnChange` code can read the model.

### Keys and paths

| Path | Reaches |
|---|---|
| `Player.Gold` | A value. Dots are part of the name, grouping for you. |
| `Party` | A list. |
| `Party[2]` | The third record of the list. |
| `Party[2].Name` | A value of that record. |
| `Party[2].Bag[0].Count` | Lists nest. |

### Lists

```cpp
const Ptr<Model> Row = mModel.Append("Party");      // a new record at the end
Row->SetText("Name", "Brom");
Row->SetNumber("Level", 7.0);

mModel.Insert("Party", 0)->SetText("Name", "Ayla"); // at a position
mModel.Erase("Party", 1);                           // the records after it move up
mModel.Resize("Party", 3);                          // grow with empty records, or cut
const UInt32     Count  = mModel.GetCount("Party");
const Ptr<Model> Second = mModel.GetRecord("Party", 1);   // nullptr when out of range
```

A list named inside a text binding shows its record count: `"Party ({Party})"` reads `Party (3)`.

### `Mirror`: keeping a list matched to your container

`Mirror` keeps a list record for record with any container read by `GetSize()` and an index:

```cpp
struct Hero { UInt32 Id; Str Name; UInt32 Level; };
Sequence<Hero> mHeroes;

mModel.Mirror("Party", mHeroes,
    [](ConstRef<Hero> Item) { return Item.Id; },                            // a stable key per item
    [](Ref<Model> Row, ConstRef<Hero> Item)
    {
        Row.SetText("Name", Item.Name);
        Row.SetNumber("Level", Item.Level);
    });
```

A key that stays keeps its record, with whatever the user left in it (a branch open, a field half typed). Records
follow the container's order. Call it again whenever the container may have changed; it touches only what differs.

`Repeater::GetKey(List, Control)` gives back the key of the row a control stands in, and `Model::FindKey(List, Key)`
finds the record holding a key. Use `Mirror` when items have a stable identity. When two items share an identity
(two stacks of one item), key them by position with `Resize` and `GetRecord` instead.

## 2. Bindings in a layout

Any field marked "binds" in `widgets.md` takes values in braces:

| Written | Means |
|---|---|
| `"Checked": "{Settings.Loot}"` | One value alone binds the whole value, both ways where the field writes back. |
| `"Text": "Gold: {Player.Gold:0}"` | Text with values in place. Shown only. |
| `"{Player.Speed:2}"` | The number after the colon fixes the decimals. |
| `"Hidden": "{!Hud.Ready}"` | The opposite of a flag. |
| `"Text": "Party ({Party})"` | A list writes its record count. |
| `"{{"`, `"}}"` | A literal brace. |
| `"{Name} owes {/Player.Gold:0}"` | Inside a row, a path opening with `/` reads from the root model. |
| `"Text": "@Menu.Play"` | A text from the lexicon (see `text.md`). It may itself name values. |
| `"@@home"` | A literal at sign. |

A field written with a format (`"Level {Player.Level:0}"`) never writes back, even on a field that binds both ways.
Use one value alone for that.

Fields that bind:

| Widget | Shown only | Both ways |
|---|---|---|
| every node | `Hidden`, `Disabled`, `Tooltip` | |
| `Label`, `Button`, `Window`, `Dialog`, `Expander`, tab, `Dropdown` | `Text` / `Title`, a dialog's `Message` and button `Text` | |
| `Button` | | `Checked` (toggle) |
| `Checkbox`, `Radio` | `Text` | `Checked` |
| `Expander` | | `Expanded` |
| `Slider`, `Spinner` | `Minimum`, `Maximum` | `Value` |
| `Scalar` | `Minimum`, `Maximum`, `Tag` | `Value` |
| `Progress` | `Value` | |
| `Swatch` | `Value` (a colour) | |
| `Image` | `Source`, `Layer`, `Tint`, `Region`, `Opaque` | |
| `Textbox` | `Hint` | `Text` |
| `Listbox`, `Combobox`, `Repeater`, `Grid` | | `Selected` (a position, `-1` for none) |
| `Outline` | | `Selected` (a record path) |
| `Tabs` | | `Selected` |
| menu item | `Text`, `Icon`, `Hidden`, `Disabled`, `Tooltip`, `Heading` | `Checked` |
| `Menubar` menu | `Text`, `Hidden`, `Disabled` | |
| splitter pane | `Hidden` | |
| drag source | `Drag` (its payload) | |

Lists themselves (`Items` of a `Repeater`, `Grid`, `Outline`, `Listbox`, `Combobox`) are named in braces but are not
values: they name which list the widget shows.

A template parameter may be a binding. `{ "Widget": "Setting", "Start": "{Audio.Music}" }` binds the slider inside.

## 3. Scopes

A binding reads from a **scope**: the root model, or one record. Rows of a list read their own record, so `{Name}`
inside a row's template means `Inventory[7].Name` for the eighth row. A path opening with `/` climbs to the root.

In code, `Markup::PushScope("Party[2]")` and `PopScope()` make every binding built between them read that record, as a
row does. `Markup::Spawn(Parent, Template, Scope, Within)` spawns a template reading one record.

## 4. Item lists with `Repeater`

A `Repeater` shows one row per record of a model list, each row made from `ItemTemplate`.

```json
"BagRow": { "Flow": "Row", "Gap": 8, "Children": [
  { "Widget": "Image", "Source": "{Icon}", "Width": 24, "Height": 24 },
  { "Widget": "Label", "Text": "{Name}", "Grow": 1 },
  { "Widget": "Label", "Text": "x{Count:0}", "Hidden": "{!Stacked}" }
] },
"Bag": { "Widget": "Window", "Title": "Bag ({Bag})", "Width": 320, "Children": [
  { "Widget": "Repeater", "Name": "Rows", "Height": 300, "ItemHeight": 32, "Items": "{Bag}",
    "ItemTemplate": "BagRow", "Selected": "{Ui.BagPick}", "Action": "Bag.Pick",
    "RowStates": { "Marked": "Equipped" } }
] }
```

What you must know:

- **Only rows in view exist, and rows are reused** for other records as the list scrolls. Everything a row shows
  must come from bindings of its record. A value set in code on a row's label lands on whatever record the row shows
  next.
- **Give `Height`** (200 otherwise) and `ItemHeight` when rows are all one height (the first row is measured
  otherwise).
- **Rows wear the list's style plus `.Item`** (`Repeater.Item`, or `Bag.Item` with `"Style": "Bag"`) and are `Selected`
  while picked.
- **`RowStates`** gives each row a state by a flag of its record. `"!Key"` uses the opposite. Prefer the built-in
  `Marked`; a state of your own must be declared before the theme is read (see `themes.md`).
- **`Columns`** and **`ItemWidth`** turn the list into a grid of cells.
- A list given a `Name` scrolls back where it was when made again.

Reading which record a control stands in, from an action:

```cpp
mInterface.OnClick("Bag.Use", [this](Element Control)
{
    const SInt32 Record = Repeater::GetRecord(mBag.Find("Rows"), Control);   // below 0 when in no row
    if (Record >= 0)
    {
        Use(static_cast<UInt32>(Record));
    }
});
```

In code:

```cpp
const Element Grid = Repeater::Create(Card, "Slots");
Grid.EditBox().Height = Length::Pixels(156.0f);
Repeater::SetColumns(Grid, 5);
Repeater::SetItemHeight(Grid, 48.0f);
Repeater::SetTemplate(Grid, Maker::Create([this](Element Slot, Text Scope) { FillSlot(Slot, Scope); }));
Repeater::SetItems(Grid, "Inventory");

void FillSlot(Element Slot, Text Scope)
{
    const Element Count = Label::Create(Slot, "", "Slots.Count");
    Binder::Attach(Count, "{Count:0}", Binder::PutText<&Label::SetText>(), Reader(), Scope);
}
```

The `Maker` is `Repeater::Maker`, `Delegate<void(Element Row, Text Scope)>`. Bind every part with the `Scope` it is
handed, so the row follows the record it is moved to.

`Listbox` and `Combobox` can also show a model list (`"Items": "{Spells}"`, `"ItemText": "{Name} ({Cost:0})"`), but
they make every item up front. Keep those for short lists.

## 5. Binding in code

`Binder::Attach(Element, Source, Writer, Reader = Reader(), Scope = Text())` binds any element to a text of values or
a lexicon key, and shows it at once:

```cpp
Binder::Attach(Title, "{Player.Name}", Binder::PutText<&Label::SetText>());
Binder::Attach(Gauge, "{Player.HealthShare}", Binder::Put<Real32, &Progress::SetValue>());
Binder::Attach(Badge, "{!Rare}", Binder::PutState<State::Hidden>(), Reader(), Scope);
Binder::Attach(Box,   "{Settings.Loot}",
               Binder::PutState<&Widget::Checked>(), Binder::TakeState<&Widget::Checked>());
```

| Maker | Makes |
|---|---|
| `Binder::Put<Type, &Setter>()` | A writer handing a flag, number or colour to `void Setter(Element, Type)`. |
| `Binder::PutText<&Setter>()` | A writer handing text to `void Setter(Element, Text)`. |
| `Binder::PutState<State::X>()` | A writer turning a core state on by a flag. |
| `Binder::PutState<&Widget::Checked>()` | The same for a declared state. |
| `Binder::Take<Type, &Getter>()` | A reader for a value bound both ways. |
| `Binder::TakeState<...>()` | A reader taking a state as a flag. |

A `Writer` is `Delegate<void(Element, ConstRef<Datum>)>`, so a lambda works for anything else:

```cpp
Binder::Attach(Picture, "{Hero.Portrait}", Writer([this](Element Target, ConstRef<Datum> Value)
{
    Image::SetAddress(Target, * mMarkup, Value.GetText());
}));
```

`Binder::Detach(Element)` drops an element's bindings. `Binder::IsBound(Text)` and `Binder::IsLocalized(Text)` tell
whether a text binds. A custom widget binds its layout fields with `Markup::Bind(Element, Node, Key, Writer, Reader)`,
which binds only when the field's value is a binding or a lexicon key.

## 6. Events

Every input becomes an `Event` (`<UI.Core/Event.hpp>`) sent to its target control, then **bubbled** up through every
control holding it. At each control: first its own behaviour (the widget's logic), then its **observers**, then its
**action**. Any of them returning `true` stops the bubbling.

| Field | Holds |
|---|---|
| `Kind` | The signal. |
| `Target` | The control it happened to. |
| `Content` | Typed text, or a record path for list signals. |
| `Other` | The other element of a drag, or the row of a list signal. |
| `Position`, `Delta` | The pointer in units, the wheel or drag movement. |
| `Count` | Presses in a row (2 is a double click). |
| `Button`, `Key`, `Modifiers` | The pointer button, the key, the modifiers held. |

Signals (`Signal::k...`):

| Signal | When |
|---|---|
| `kPress`, `kRelease`, `kClick` | Pointer button down, up, down and up on the same control. |
| `kDrag` | The pointer moved with the button held. |
| `kWheel` | The wheel turned over it. |
| `kFocus`, `kBlur` | It took or lost the keyboard. |
| `kKey`, `kType` | A key went down, or text was typed, while it holds the keyboard. |
| `kChange` | Its value changed by the user. |
| `kOpen` | A popup was opened at the pointer. |
| `kEnter`, `kLeave`, `kMove` | Pointer hover, only for controls with `"Hover": true`. |
| `kBegin`, `kEnd` | Around one edit gesture (undo grouping). |
| `kCommit` | A text field committed (enter, or losing the keyboard). |
| `kActivate` | A row double pressed or entered. |

Widgets add their own: `Window::kClose`, `Tabs::kClose`, `Repeater::kRename`, `Grid::kSort`, `Outline::kExpand`,
`Outline::kCollapse`, `Outline::kDrop`, `Drag::kStart`, `kEnter`, `kLeave`, `kDrop`, `kEnd`. A game makes its own with
`Identify("Name")` and sends it with `Interface::Dispatch(Event(Kind, Target))`.

## 7. Running code on what the user does

Four ways, from simplest to most general:

### `OnClick` and `OnChange`

```cpp
mInterface.OnClick("Menu.Play", [this](Element Control) { StartGame(); });
mInterface.OnChange("Settings.Volume", [this](Element Control)
{
    SetVolume(Slider::GetValue(Control));           // or mModel.GetNumber("Settings.Volume")
});
```

`OnClick` hears only a click **of the control naming the action** (not of its parts), and `OnChange` only a change of
its own value. Both let the event bubble on, so a menu or dialog holding the control still closes. The code is a
`Reaction`, `Delegate<void(Element Control)>`.

**One action name holds one handler.** `OnClick`, `OnChange` and `Bind` all bind the action, so the last call wins.
Give a click and a change different action names.

### `Bind`

`Bind` hears every event reaching any control naming the action, raw, and may stop it:

```cpp
mInterface.Bind("Bag.List", Handler::Create([this](Ref<Interface> Owner, ConstRef<Event> Message, Handle Self)
{
    if (Message.Kind == Signal::kActivate && Message.Target == Self)
    {
        Use(static_cast<UInt32>(Repeater::GetIndex(Element(Owner, Self))));
        return true;                                  // handled, stop here
    }
    return false;                                     // let it bubble on
}));
```

Use it for widget signals (`kActivate`, `Window::kClose`, `Tabs::kClose`, `kCommit`), and to **refuse** something:
returning `true` on `Window::kClose` keeps the window open, on a dialog button's `kClick` keeps the dialog open.
Always compare `Message.Target == Self` unless you mean to hear the control's parts too.

`Unbind(Action)`, `UnbindAll()` and `IsBound(Action)` manage bindings. Actions are only names, so binding before or
after the templates naming them both work, and every control naming one action shares its handler.

### `Element::Observe`

An observer runs on every event reaching one element, before its action. The example observes its inventory grid to
swap records on drops:

```cpp
Grid.Observe(Handler::Create([this](Ref<Interface> Owner, ConstRef<Event> Message, Handle Self)
{
    if (Message.Kind != Drag::kDrop)
    {
        return false;
    }
    const SInt32 From = Repeater::GetRecord(mGrid, Element(Owner, Message.Other));
    const SInt32 To   = Repeater::GetRecord(mGrid, Element(Owner, Message.Target));
    // ... swap the two records in the model ...
    return true;
}));
```

### Shortcuts

`Shortcuts` in a layout or `Interface::AddShortcut` run an action from a key chord (see `layouts.md`).

## 8. Reading widgets from an action

| Widget | Read |
|---|---|
| Checkbox, toggle | `Control.HasState(Widget::Checked())` |
| Radio group | `Radio::GetChecked(Control)` |
| Slider, Spinner | `Slider::GetValue(Control)`, `Spinner::GetValue(Control)` |
| Scalar | `Scalar::GetValue(Control)` |
| Textbox | `Textbox::GetText(Control)` |
| Listbox, Combobox | `Listbox::GetIndex(Control)`, `Listbox::GetText(Listbox::GetSelected(Control))` |
| Repeater | `Repeater::GetIndex(List)`, `Repeater::GetRecord(List, Control)`, `Repeater::GetKey(List, Control)` |
| Tabs | `Tabs::GetSelected(Control)` |
| Menu item | `Menu::GetSubject(Control)`, the element the menu was opened over |

The model already holds every both-ways value, so reading the model works as well.

## 9. Common mistakes

| Mistake | What happens | Do instead |
|---|---|---|
| `OnClick("X")` and `OnChange("X")` | The second replaces the first. | Two action names. |
| A row's label set in code | It shows on other records as the list scrolls. | Bind it to the record. |
| `"Value": "{Volume:0}"` on a slider | Shown, but the slider's edits never reach the model. | `"{Volume}"` alone. |
| Expecting game writes to run `OnChange` | Game writes are silent. | Call your code directly. |
| Comparing nothing in `Bind` | The handler runs for every event of every part. | Check `Kind` and `Target == Self`. |
| `Binder::SetModel` never called | Every binding shows nothing. | Call it once at start. |
| Keying `Mirror` by position | Records lose what the user left in them as items move. | Key by a stable id. |
| A `Repeater` named `Items` with `"Items": "Inventory"` | Logged: must name one list in braces. | `"Items": "{Inventory}"`. |

## 10. Where it lives

| What | Path |
|---|---|
| Model | `Module/UI.Data/Source/UI.Data/Model.hpp` |
| Values | `Module/UI.Data/Source/UI.Data/Datum.hpp` |
| Binder | `Module/UI.Data/Source/UI.Data/Binder.hpp`, `Link.hpp` (`Writer`, `Reader`) |
| Repeater | `Module/UI.Control/Source/UI.Control/Widget/Repeater.hpp` |
| Events, signals, `Handler` | `Module/UI.Core/Source/UI.Core/Event.hpp` |
| `Bind`, `OnClick`, `OnChange` | `Module/UI.Core/Source/UI.Core/Interface.hpp` |
| Every pattern in use | `Example/Source/Example.hpp` |
