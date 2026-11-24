# Widgets

Every widget the library registers, with the fields a layout node gives it, the parts its theme dresses, the states
it wears, the events it reports and the calls code makes on it. The fields every node takes (`Name`, `Style`,
`Hidden`, `Disabled`, `Action`, layout fields, tooltips, drag and drop, context menus) are in `layouts.md` and apply
to every widget below.

## 1. How to read this chapter

- **Field** tables list the keys the widget's builder reads. "Binds" says whether the value may be a binding
  (`{Key}`, `@Key`, text with values in place): **shown** means the model writes the widget, **both** means the
  widget also writes the user's edit back to the model.
- **Parts** are the styles the widget's pieces wear when its style is the default one. With `"Style": "X"`, each part
  wears `X.` plus the part's suffix (see `themes.md`, section 7).
- **Events** are the signals the widget sends (`Signal::kChange` and so on, in `<UI.Core/Event.hpp>`). `OnClick` hears
  `kClick`, `OnChange` hears `kChange`, `Bind` hears everything (see `binding.md`).
- **Code** lists the static calls in the widget's header, `<UI.Control/Widget/<Name>.hpp>`. Every widget is a class of
  static functions over `Element`.

The library registers 28 widgets: `Panel`, `Label`, `Image`, `Button`, `Checkbox`, `Radio`, `Slider`, `Progress`,
`Spinner`, `Scalar`, `Swatch`, `Textbox`, `Listbox`, `Combobox`, `Repeater`, `Tabs`, `Expander`, `Scroller`,
`Scrollbar`, `Window`, `Dialog`, `Menu`, `Menubar`, `Dropdown`, `Grid`, `Outline`, `Splitter`, `Dock`. `Popup` is a
building block for code only and has no layout name.

Every value widget reports `Signal::kBegin` and `Signal::kEnd` around the `kChange`s of one user edit, so an undo
system can group them: a text field or scalar from its first change to its commit, a slider from press to release, a
spinner per press, notch or key, a checkbox and a toggle button per click.

Code setters (`SetValue`, `SetText`, `Select`...) change the widget **silently**: they report nothing, so your own
`OnChange` does not run. Only the user's edits report.

## 2. Panel

A plain box. A node with no `Widget` is a panel.

| Field | Binds | Meaning |
|---|---|---|
| layout fields only | | See `layouts.md`. |

No default style. Code: `Panel::Create(Parent, Style = Text(), Flow = Direction::Column)`.

## 3. Label

A line or paragraph of text. It breaks onto new lines at its box's width.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Text` | shown | `""` | The text. `"Gold: {Player.Gold:0}"`, `"@Menu.Play"`. |
| `Overflow` | | `Clip` | `Clip` runs on past the box; `Ellipsis` cuts a too long line short with `…`. |

Parts: `Label` (and `Title` for headings). Code: `Label::Create(Parent, Content, Style = "Label")`,
`Label::SetText(Element, Text)`, `Label::SetOverflow(Element, Overflow)`.

## 4. Image

Shows a texture of the content, or a GPU texture you keep.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Source` | shown | none | The address of a `.tex`, as `"Resources://Icon/Sword.tex"`. Bound, each row may show its own. |
| `Region` | shown | `[0, 0, 1, 1]` | `[u0, v0, u1, v1]`. Bound, a text of four numbers (`"0 0 0.5 1"`). |
| `Layer` | shown | `0` | The slice of an array texture. |
| `Tint` | shown | white | `"#RRGGBBAA"` or `[r, g, b, a]` (no palette names here). Bound, a colour of the model. |
| `Filter` | | `Linear` | `Point` keeps pixel art sharp. |
| `Fit` | | `Stretch` | `Contain` keeps its shape whole inside the box; `Cover` fills the box and cuts the rest. |
| `Opaque` | shown | `false` | `true` draws the texture's alpha as data, not coverage (as a height map). |
| `Flip` | | `false` | `true` draws it upside down (render targets). |

No default style. Give it `Width` and `Height`, or it takes the picture's size. Code:
`Image::Create(Parent, Retainer<ZyGraphic::Image>, Style)`, `Image::CreateView(Parent, Texture, Layout, Style)`
(a GPU texture, see `text.md`), `SetSource`, `SetAddress(Element, Markup, Address)`, `SetTexture`, `SetLayer`,
`SetRegion(Element, Rect)`, `SetTint`, `SetFilter`, `SetFit`, `SetOpaque`, `SetFlip`.

## 5. Button

A panel answering the pointer, holding a label.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Text` | shown | `""` | The label's text. |
| `Toggle` | | `false` | `true` turns its `Checked` state over on each click. |
| `Checked` | both | `false` | The toggle's state. |

Parts: `Button`, `Button.Label`. States: `Hover`, `Press`, `Focus`, `Disabled`, and `Checked` for a toggle. Events:
`kClick` (the action), and `kBegin`, `kChange`, `kEnd` from a toggle. Focusable; enter and space click it.

Code: `Button::Create(Parent, Content, Style = "Button")`, `Button::SetText(Element, Text)`,
`Button::GetLabel(Element)`.

```json
{ "Widget": "Button", "Text": "@Menu.Play", "Action": "Menu.Play" }
{ "Widget": "Button", "Text": "Mute", "Toggle": true, "Checked": "{Audio.Muted}" }
```

## 6. Checkbox

A box with a mark, beside a label.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Text` | shown | `""` | The label. |
| `Checked` | both | `false` | Whether it is checked. |

Parts: `Checkbox`, `Checkbox.Box`, `Checkbox.Mark`, `Checkbox.Label`. State: `Checked`. Events: `kBegin`, `kChange`,
`kEnd` per click. Code: `Checkbox::Create(Parent, Content, Checked = false, Style)`,
`Checkbox::SetChecked(Interface, Handle, Bool)` (reports, as a click), `Checkbox::SetText`. Read it with
`Element.HasState(Widget::Checked())` (`<UI.Control/Widget/Common.hpp>`).

## 7. Radio

A round box with a mark. One radio of a group is checked at a time.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Text` | shown | `""` | The label. |
| `Checked` | both | `false` | Whether it is the one checked. |
| `Group` | | `""` | A group name spanning the whole pane. Empty groups the radios sharing a parent. |

Each radio binds its own flag. Checking one clears the rest of its group, and each writes its flag back, so three
radios bound to `Settings.Easy`, `Settings.Normal`, `Settings.Hard` keep exactly one set.

Parts: `Radio`, `Radio.Box`, `Radio.Mark`, `Radio.Label`. State: `Checked`. Events: `kChange`. The arrow keys move the
check through the group. Code: `Radio::Create(Parent, Content, Checked, Group, Style)`, `Radio::SetChecked` (reports),
`Radio::Toggle` (silent), `Radio::GetChecked(Any)` (the one checked in that radio's group), `Radio::SetText`.

## 8. Slider

A track filled up to its value, with a thumb.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Value` | both | `0` | The value. |
| `Minimum` | shown | `0` | The low end. |
| `Maximum` | shown | `1` | The high end. |

Parts: `Slider` (its `Fill` colours the filled share, `Background` the track), `Slider.Thumb`. Events: `kBegin` on
press, `kChange` while dragging, `kEnd` on release. Keys: arrows move a twentieth of the range, home and end go to the
ends. Code: `Slider::Create(Parent, Value, Minimum = 0, Maximum = 1, Style)`, `SetValue`, `GetValue`, `SetMinimum`,
`SetMaximum`.

```json
{ "Widget": "Slider", "Value": "{Settings.Volume}", "Maximum": 100, "Action": "Settings.Volume" }
```

## 9. Progress

A bar filled up to its value, showing its share as a percentage.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Value` | shown | `0` | The value. Never writes back. |
| `Minimum` | | `0` | The low end. Does not bind. |
| `Maximum` | | `1` | The high end. Does not bind. |
| `Labelled` | | `true` | `true` shows `40%` over it; `false` shows the bar alone. |

Since the range does not bind, feed a share from 0 to 1 and keep the default range when the maximum changes in play.
Parts: `Progress` (`Fill`, `Background`, `Height` 18), `Progress.Label`. Reports nothing. Code:
`Progress::Create(Parent, Value, Minimum, Maximum, Labelled, Style)`, `SetValue`, `GetValue`.

## 10. Spinner

A value between a button taking a step away and one adding a step.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Value` | both | `0` | The value. |
| `Minimum` | shown | `0` | The low end. |
| `Maximum` | shown | `100` | The high end. |
| `Step` | | `1` | How far one press, notch or key moves it. |
| `Decimals` | | `0` | Digits shown after the point. |

Parts: `Spinner`, `Spinner.Decrease`, `Spinner.Decrease.Label`, `Spinner.Increase`, `Spinner.Increase.Label`,
`Spinner.Label`. Events: `kBegin`, `kChange`, `kEnd` per step. Up and down keys and the wheel step it. Code:
`Spinner::Create(Parent, Value, Minimum, Maximum, Step = 1, ...)`, `SetValue`, `GetValue`, `SetMinimum`,
`SetMaximum`.

## 11. Scalar

A number typed into a field, behind an optional tag. Built for editors (vectors, properties).

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Value` | both | `0` | The value. |
| `Minimum`, `Maximum` | shown | `0`, `0` | The range. Both at zero means no range. |
| `Decimals` | | `0` | Digits kept and shown after the point (at most 9). |
| `Tag` | shown | `""` | A short tag before the value, as `X`. |
| `TagStyle` | | `Scalar.Tag` | The tag's style, as `Axis.X`. |

Parts: `Scalar`, `Scalar.Tag`, `Scalar.Field` (a text field: `.Text`, `.Hint`, `.Caret`, `.Selection`). Events:
`kBegin` at the first typed change, `kChange`, `kEnd` at the commit. Code:
`Scalar::Create(Parent, Value = 0.0, Decimals = 0, Style)`, `SetValue`, `GetValue`, `SetRange`, `SetMinimum`,
`SetMaximum`, `SetDecimals`, `SetTag`, `SetTagStyle`, `GetTag`, `GetField`.

```json
{ "Flow": "Row", "Gap": 2, "Children": [
    { "Widget": "Scalar", "Tag": "X", "TagStyle": "Axis.X", "Value": "{Pos.X}", "Decimals": 3, "Grow": 1 },
    { "Widget": "Scalar", "Tag": "Y", "TagStyle": "Axis.Y", "Value": "{Pos.Y}", "Decimals": 3, "Grow": 1 }
] }
```

## 12. Swatch

A box filled with a colour over a checkerboard, so transparency shows.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Value` | shown | white | `"#RRGGBBAA"`, or a bound colour (`Model::SetColor`). |

Part: `Swatch`. Reports nothing. Code: `Swatch::Create(Parent, Color, Style)`, `SetValue`, `GetValue`.

## 13. Textbox

A text field, one line or several.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Text` | both | `""` | The text. |
| `Hint` | shown | `""` | Shown while it is empty. |
| `Multiline` | | `false` | Enter breaks lines, and the text wraps to the width. |
| `Immutable` | | `false` | The text cannot be changed (still selectable and copyable). |
| `Numeric` | | `false` | Accepts only digits, signs and points. |
| `Password` | | `false` | Shows a mask instead of the text. |
| `Limit` | | `0` | The most characters, `0` for no limit. |

Parts: `Textbox`, `Textbox.Text`, `Textbox.Hint`, `Textbox.Caret`, `Textbox.Selection`. State: `Focus` while typing.
Events: `kBegin` before the first edit, `kChange` per edit, `kCommit` and `kEnd` when enter commits or it loses the
keyboard. Escape puts back the text it started with. While it holds the keyboard `Interface::IsTyping()` is `true`.

Code: `Textbox::Create(Parent, Content, Hint, Style)`, `SetText`, `GetText`, `SetHint`, `SetMultiline`,
`SetImmutable`, `SetNumeric`, `SetPassword`, `Select(Element, Anchor, Caret)`, `GetCaret`, `GetSelected`. Editing keys
are in `text.md`.

## 14. Listbox

A scrolling column of text items, one picked at a time. It makes every item up front, so use a `Repeater` for long
lists.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Items` | | | A list of texts (each may be `@Key`), or one list of the model in braces, `"{Spells}"`. |
| `ItemText` | | `Name` | For a model list: what each item shows, a key (`Label`) or a text naming values (`"{Label} ({Cost:0})"`). |
| `ItemHidden` | | | For a model list: a key or text whose flag hides an item (`"{!Known}"`). |
| `ItemDisabled` | | | For a model list: a key or text whose flag disables an item. |
| `Selected` | both | `-1` | The picked position, `-1` for none. Counts hidden items. |

Parts: `Listbox`, items `ListItem`, `ListItem.Label`. States: `Selected` on the picked item. Events: `kChange` when the
pick changes. Up and down move the pick.

Code: `Listbox::Create(Parent, Style)`, `AddItem(Owner, Text)`, `SetItems(Owner, Source, Content = "{Name}", Scope)`,
`SetFilter(Owner, Hidden, Disabled)`, `Select(Owner, Item)`, `GetSelected(Owner)`, `SetIndex`, `GetIndex`,
`GetText(Item)`, `GetList`.

## 15. Combobox

A header showing the pick, opening a `Listbox` of the items in a popup.

| Field | Binds | Meaning |
|---|---|---|
| `Items`, `ItemText`, `ItemHidden`, `ItemDisabled` | | As `Listbox`. |
| `Selected` | both | The picked position, `-1` for none. |

Parts: `Combobox`, `Combobox.Label`, `Combobox.Arrow`, `Combobox.Popup`, `Combobox.List`, items `ListItem`. Events:
`kChange` on a pick. A click toggles the popup, the arrows step the pick. Read the pick with
`Listbox::GetIndex(Control)` or `Listbox::GetSelected(Control)`, which work on any widget keeping a list.

```json
{ "Widget": "Combobox", "Action": "Settings.Quality", "Selected": "{Settings.Quality}",
  "Items": [ "@Quality.Low", "@Quality.Medium", "@Quality.High" ] }
```

## 16. Repeater

A virtual list of model records, one row per record made from a template. Only rows in view exist, so a list of
thousands costs what a screenful does. See `binding.md` for the full story.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Items` | | | One list of the model in braces, `"{Inventory}"`. |
| `ItemTemplate` | | | A template name, or a node written in place. Its bindings read the row's record. |
| `ItemHeight` | | measured | Every row's height. Left out, the first row is measured. |
| `Columns` | | `0` | Rows side by side, making a grid. |
| `ItemWidth` | | `0` | A cell's width. Without `Columns`, as many columns as fit. |
| `Selected` | both | `-1` | The picked position. |
| `Multiple` | | `false` | Control and shift pick several, control and `A` all. |
| `RightPick` | | `true` | A right press picks its row, so a context menu opens over the pick. |
| `RowStates` | | | `{ "Marked": "Worn", "Disabled": "!Usable" }`, states each row wears by a flag of its record. |

Default `Height` is 200. Parts: `Repeater`, rows `Repeater.Item` (wearing `Selected` when picked). Events: `kChange`
on a pick; `Signal::kActivate` on a double press or enter, and `Repeater::kRename` on `F2`, both with `Event::Content`
naming the record's path (`Inventory[3]`) and `Event::Other` the row.

Code: `Create(Parent, Style)`, `SetItems(List, Source, Scope)`, `SetTemplate(List, Maker)` (or a markup and template
name or node), `SetItemHeight`, `SetItemWidth`, `SetColumns`, `SetIndex`, `GetIndex`, `GetCount`, `GetRealized`,
`GetRow(List, Record)`, `GetRecord(List, Row)`, `GetKey(List, Row)`, `ScrollTo(List, Record)`, `GetScope`,
`SetMultiple`, `IsMultiple`, `SetPicked`, `IsPicked`, `GetPicked`, `SetRightPick`, `AddRowState(List, State, Key)`,
`Marked()`.

## 17. Tabs

A row of tab buttons over pages, showing the picked tab's page.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Tabs` | | | A list of tabs, each `{ "Title", "Name", "Closable", "Children" }`. `Title` binds. |
| `Closable` | | `false` | Gives every tab a close button (a tab's own `Closable` wins). |
| `Selected` | both | `0` | The shown tab's index. |

Parts: `Tabs`, `Tabs.Header`, `Tabs.Tab`, `Tabs.Tab.Label`, `Tabs.Close`, `Tabs.Close.Label`, `Tabs.Pages`. State:
`Selected` on the shown tab. Events: `kChange` when another tab is shown; `Tabs::kClose` when a close button is
clicked, and the tab goes unless a handler returns `true` to stop it. Left and right keys move between tabs.

Code: `Create`, `AddTab(Owner, Title, Style, Closable)` (returns the page), `SetClosable`, `GetClose`, `RemoveTab`,
`Select`, `GetSelected`, `GetCount`, `GetTab`, `GetPage`.

```json
{ "Widget": "Tabs", "Height": 260, "Tabs": [
    { "Title": "@Options.Audio", "Children": [ { "Widget": "Label", "Text": "..." } ] },
    { "Title": "@Options.Video", "Name": "Video", "Children": [ ] }
] }
```

## 18. Expander

A header that opens and closes the content under it.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Title` | shown | `""` | The header's text. |
| `Expanded` | both | `false` | Whether it is open. |

Children go into its content. Given a `Name`, it reopens as it was left when made again. Parts: `Expander`,
`Expander.Header`, `Expander.Header.Label`, `Expander.Arrow`, `Expander.Content`. State: `Checked` while open. Events:
`kChange` on a toggle. Code: `Create(Parent, Title, Expanded, Style)`, `SetExpanded(Interface, Handle, Bool)`
(reports), `Show(Element, Bool)` (silent), `IsExpanded`, `GetContent`, `SetTitle`.

## 19. Scroller

Content with a bar down its side and along its bottom, each shown only while there is something to scroll. The wheel
scrolls it.

| Field | Default | Meaning |
|---|---|---|
| `Horizontal` | `true` | `true` lets content keep its width and scroll sideways; `false` fits content to the viewer's width, so labels wrap. |
| `Follow` | `false` | Keeps it at its end as content grows, as a chat log, until the user scrolls away. |

Default `Height` is 200. Children go into its content. Given a `Name`, it scrolls back where it was. Parts:
`Scroller`, `Scroller.Content`, `Scroller.Bar`, `Scroller.Bar.Thumb`. Code: `Create`, `GetContent`,
`GetBar(Viewer, Vertical)`, `SetHorizontal`, `SetFollow`, `ScrollTo(Viewer, Vector2)`, `GetOffset`.

## 20. Scrollbar

A bar scrolling another element found by name.

| Field | Default | Meaning |
|---|---|---|
| `Target` | | The `Name` of the element it scrolls, looked up the first time it is needed. |
| `Orientation` | `Vertical` | `Vertical` or `Horizontal`. |
| `Automatic` | `false` | `true` hides it while what it scrolls fits. |

Parts: `Scrollbar`, `Scrollbar.Thumb`. Code: `Create(Parent, Vertical, Style)`, `SetTarget(Bar, Target)`,
`GetTarget`, `SetAutomatic`, `GetThumb`, and for any scrolling element `ScrollTo(Target, Vertical, Offset)`,
`GetOffset`, `GetLimit`, `GetView`.

## 21. Window

A pane of its own in the `Window` band: a title bar, a close button and a body. Dragged by its bar and kept on screen.
Clicking it raises it over other windows.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Title` | shown | `""` | The bar's text. |
| `Resizable` | | `false` | Edges and a corner grip size it. |
| `Minimum` | | `[160, 100]` | The least size, `[width, height]`. |
| `Left`, `Top`, `Width`, `Height` | | | Where it opens and how big (layout fields). |

Children go into its body. It opens **shown**. Parts: `Window` (default `Enter` `Pop`, `Exit` `FadeOut`),
`Window.Bar`, `Window.Title`, `Window.Close`, `Window.Close.Label`, `Window.Content`, `Window.Edge`, `Window.Grip`.

Events: the close button reports `Window::kClose` first. A `Bind` handler returning `true` on it keeps the window
open (an editor asking "Discard changes?"). Then the window hides and reports `kChange`.

Code: `Window::Open(Tree, Title, Style = "Window", Band = Window::kLayer)`, `Show(Interface, Handle)` (unhide and
raise), `Close(Interface, Handle)` (hide, reporting nothing), `Request(Interface, Handle)` (as the close button),
`GetContent`, `MoveTo(Element, Vector2)`, `SetTitle`, `SetResizable`, `IsResizable`, `SetMinimum`, `GetGrip`,
`GetClose`.

## 22. Dialog

A window in the `Modal` band, centred, asking a message through buttons. Holds pointer and keyboard over a backdrop
while shown.

| Field | Binds | Meaning |
|---|---|---|
| `Title` | shown | The bar's text. |
| `Message` | shown | The question. |
| `Buttons` | | A list of `{ "Text", "Action", "Default", "Cancel", "Name" }`. `Text` binds. |
| `Left`, `Top` | | Given, it stays where put instead of centring. |

Children go into its body under the message. Enter clicks the `Default` button, escape the `Cancel` one. Any button
runs its action and closes the dialog, unless the action's `Bind` handler returns `true` on the click.

Parts: `Dialog` (and the window parts as `Dialog.Bar`, `Dialog.Title`, `Dialog.Close`, `Dialog.Content`),
`Dialog.Body`, `Dialog.Message`, `Dialog.Buttons`, `Dialog.Button`, `Dialog.Button.Label`, `Dialog.Default`,
`Dialog.Default.Label`, and the backdrop `Modal.Backdrop`.

Code: `Dialog::Open(Tree, Title, Message, Style)`, `AddButton(Dialog, Text, Action, Default, Cancel)`,
`SetMessage`, `Show(Interface, Handle)`, `Close(Interface, Handle)`. It opens shown.

```json
"Quit": { "Widget": "Dialog", "Title": "@Quit.Title", "Message": "@Quit.Message", "Buttons": [
    { "Text": "@Quit.Leave", "Action": "Game.Quit", "Default": true },
    { "Text": "@Quit.Stay", "Cancel": true } ] }
```

## 23. Menu

A popup list of items: actions, checks, headings, separators and submenus. Used by `ContextMenu`, `Menubar` and
`Dropdown`, and as a template root for a context menu.

| Field | Meaning |
|---|---|
| `Items` | The items. |

Each item is `"-"` (a separator), a text (an item showing it, `@Key` allowed), or an object:

| Item key | Binds | Meaning |
|---|---|---|
| `Text` | shown | What it shows. |
| `Action` | | The action run when it is picked. |
| `Shortcut` | | Text shown at its right, as `"Ctrl+S"`. Display only; give the real chord in `Shortcuts`. |
| `Items` | | Makes it a submenu of these items. |
| `Checked` | both | Makes it a check item, which keeps its menu open when clicked. |
| `Heading` | shown | Makes it a heading over the items that follow (the value is its text). |
| `Separator` | | `true` makes it a separator. |
| `Icon` | shown | A glyph before the text. |
| `Hidden`, `Disabled` | shown | Hide or grey it. |
| `Tooltip` | shown | A tooltip on the item. |
| `Name` | | A name to `Find` it by. |

Parts: `Menu`, `Menu.Item`, `Menu.Item.Label`, `Menu.Shortcut`, `Menu.Arrow`, `Menu.Separator`, `Menu.Icon`,
`Menu.Check`, `Menu.Check.Mark`, `Menu.Heading`, `Menu.Heading.Label`. Picking an item sends `kClick` to it (its action
runs) and closes the menus; a check item sends `kChange`. Arrows, enter and escape drive it.

Code: `Menu::Create(Opener, Style)`, `AddItem(Menu, Text, Action, Shortcut)`, `AddCheck`, `AddHeading`,
`AddSeparator`, `AddSubmenu(Menu, Text)`, `SetText`, `SetIcon`, `SetChecked`, `IsChecked`, `Attach(Element, Menu)`
(as a context menu), `Open(Interface, Handle, At, Subject)`, `Close`, `Dismiss`, `GetSubject(Item)`.

## 24. Menubar

A row of titles, each opening its menu under it. Left and right move between open menus.

| Field | Meaning |
|---|---|
| `Menus` | A list of `{ "Text", "Name", "Hidden", "Disabled", "Items" }`. `Text`, `Hidden`, `Disabled` bind. |
| `MenuStyle` | The style its menus wear, `Menu` by default. |
| `Children` | Stand at the bar's far end (a frame counter, a status button). |

Parts: `Menubar`, `Menubar.Title`, `Menubar.Title.Label`, `Menubar.Trail`. Code: `Create`, `AddMenu(Bar, Text, Style)`,
`GetMenu(Title)`, `GetTitle(Menu)`, `SetTitle`, `GetTrail`, `Open`, `Close`, `GetOpen`.

```json
{ "Widget": "Menubar", "Menus": [
    { "Text": "File", "Items": [ { "Text": "Save", "Shortcut": "Ctrl+S", "Action": "File.Save" }, "-",
                                 { "Text": "Exit", "Action": "File.Exit" } ] },
    { "Text": "View", "Items": [ { "Text": "Grid", "Checked": "{View.Grid}", "Action": "View.Grid" } ] }
  ], "Children": [ { "Widget": "Label", "Text": "{Stats.Fps:0} fps" } ] }
```

## 25. Dropdown

A button with a fixed text that opens a menu of its items under it.

| Field | Binds | Meaning |
|---|---|---|
| `Text` | shown | The button's text, which never changes with a pick. |
| `Items` | | Menu items, as `Menu`. |
| `MenuStyle` | | The menu's style, `Menu` by default. |

Its own `Action` hears the click before the menu opens, so the game can ready the items. Parts: `Dropdown`,
`Dropdown.Label`, `Dropdown.Arrow`. Code: `Dropdown::Create(Parent, Text, Style, MenuStyle)`, `GetMenu(Button)`.

## 26. Grid

A table: a row per record, a column per value, under sortable, resizable headings. Rows behave as a `Repeater`'s.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Items` | | | One list in braces. |
| `Columns` | | | A list of columns (below). |
| `Sort` | | | The key of the column sorted by at start. |
| `Descending` | | `false` | Sort the other way. |
| `Group` | | | A flag key keeping records together ahead of the sort (folders before files). |
| `GroupDescending` | | `true` | Which side of the flag comes first. |
| `Selected` | both | `-1` | The picked record. |
| `ItemHeight`, `Multiple`, `RightPick`, `RowStates` | | | As `Repeater`. |

A column:

| Column key | Default | Meaning |
|---|---|---|
| `Title` | | The heading (binds, `@Key` allowed). |
| `Key` | | The record value it shows and sorts by. |
| `Width` | `0` | A fixed width, or `0` to share the room left. |
| `Share` | `1` | Its share of the room left. |
| `Minimum` | `40` | The least width, by share or drag. |
| `Align` | `Start` | Where cell text sits. |
| `Sortable` | `true` | A click on the heading sorts by it. |
| `Cell` | | A template name, or a node in place, each cell is made from. A label otherwise. |

The model keeps its own order; the grid only shows it sorted. Default `Height` 200. Events: `kChange` on a pick,
`Grid::kSort` when a heading sorts. Code: `Create`, `AddColumn(Owner, Column, Markup)`, `SetItems`, `GetCount`,
`SetWidth`, `GetWidth`, `SetSort(Owner, Index, Descending)`, `GetSort`, `IsDescending`, `SetComparer(Owner, Comparer)`,
`SetGroup`, `Refresh`, `SetMultiple`, `Select`, `GetSelected`, `GetSelection`, `GetRecord(Owner, Position)`, `GetRow`,
`GetHeading`, `GetList`.

## 27. Outline

A tree view of records and the lists of children they hold.

| Field | Binds | Default | Meaning |
|---|---|---|---|
| `Items` | | | One list in braces, the top level. |
| `Nested` | | `Children` | The name of each record's list of children. |
| `ItemTemplate` | | label of `{Name}` | What each row shows. |
| `Unfolded` | | `false` | Every branch starts open. |
| `Selected` | both | | The picked record's path, as text (`Scene[1].Children[0]`). |
| `DragRows` | | | A drag kind. Rows can be dragged onto other rows. |
| `DropBetween` | | `false` | Drops near a row's top or bottom land beside it. |
| `ItemHeight`, `Multiple`, `RightPick`, `RowStates` | | | As `Repeater`. |

A record with children, or with a flag `Expandable`, shows an arrow. Events: `kChange` on a pick, `Outline::kExpand`
and `Outline::kCollapse` when a branch opens or closes (fill children lazily on expand), `Outline::kDrop` on a row drop;
`Outline::GetLanding(Owner)` gives `Source`, `Target` (paths) and `Where` (`Before`, `Inside`, `After`). The game moves
the record in its model.

Default `Height` 200. Parts: `Outline`, `Outline.List`, `Outline.List.Item`, `Outline.Marker`, `Outline.Indent`,
`Outline.Arrow`, `Outline.Content`, `Outline.Label`. Code: `Create`, `SetItems`, `SetNested`, `SetTemplate`,
`SetExpanded(Owner, Path, Bool)`, `IsExpanded`, `Reveal`, `SetUnfolded`, `SetMultiple`, `Select(Owner, Path)`,
`GetSelected`, `GetSelection`, `GetCount`, `GetPath`, `GetDepth`, `GetRow`, `GetArrow`, `GetList`, `Refresh`,
`SetDraggable(Owner, Kind, Between)`, `GetLanding`.

## 28. Splitter

Panes side by side (or stacked) with draggable bars between them.

| Field | Default | Meaning |
|---|---|---|
| `Orientation` | side by side | `"Vertical"` stacks the panes. |
| `Name` | | The name `Splitter::Save` and `Load` keep it under. |
| `Panes` | | A list of `{ "Size", "Minimum", "Name", "Hidden", "Children" }`. |

A pane's `Size` is pixels (kept) or a share (`"25%"`, kept as a share). Panes with no size share what is left.
`Minimum` defaults to 40. `Hidden` binds, so sidebars toggle from the model. A double click on a bar folds a pane.
Parts: `Splitter`, `Splitter.Pane`, `Splitter.Bar`. Events: `kBegin`, `kChange`, `kEnd` around a bar drag.

Code: `Create(Parent, Flow, Style)`, `AddPane(Owner, Size, Minimum, Style)`, `GetCount`, `GetPane`, `GetBar`,
`SetSize`, `GetSize`, `SetShare`, `GetShare`, `GetMeasure`, `SetFolded`, `IsFolded`, `Save(Scope)` (every splitter
below, as text), `Load(Scope, Text, Error)`.

## 29. Dock

Tabbed regions along the edges and in the middle, and panels floating in windows. For editors and tools.

| Field | Meaning |
|---|---|
| `Panels` | A list of `{ "Name", "Title", "Side", "Children" }`. `Side` is `Left`, `Right`, `Top`, `Bottom`, `Center` (default) or `Float`. `Title` defaults to `Name`. |
| `Sizes` | `[left, right, top, bottom]`, the edge regions' sizes. |

Dragging a tab out floats its panel. Dragging a floating window over the dock's targets docks it. Parts: the `Dock.*`
styles in `themes.md`. Code: `Create`, `AddPanel(Owner, Name, Title, Side)`, `Move(Owner, Name, Side, Position)`,
`GetSide`, `GetPanel`, `GetWindow`, `GetRegion`, `GetCount`, `GetTarget`, `GetOverlay`, `Save(Owner)` (JSON text),
`Load(Owner, Text, Error)`.

## 30. Popup (code only)

A pane over everything else, opened by a control and closed by a press outside both. Widgets like `Combobox` use it.
Code: `Popup::Create(Opener, Style)`, `Show(Interface, Handle, Anchor, Below = true)`, `Drop(Interface, Handle,
Anchor)`, `Hide`, `IsOpen`.

## 31. Your own widgets

A game teaches the markup a widget with a builder:

```cpp
// Element Build(Ref<Markup> Reader, Element Parent, ConstRef<JsonValue> Node)
static Element BuildMeter(Ref<Markup> Reader, Element Parent, ConstRef<JsonValue> Node)
{
    const Element Result = Progress::Create(Parent, 0.0f, 0.0f, 1.0f, false, Markup::GetText(Node, "Style", "Meter"));
    Reader.Bind(Result, Node, "Value", Binder::Put<Real32, &Progress::SetValue>());
    return Result;
}

mMarkup->Register("Meter", Markup::Builder::Create<&BuildMeter>());
```

The markup applies the common fields (`Name`, `Style`, layout, `Action`, tooltips...) after the builder returns, and
makes `Children` inside the element `Markup::SetHost(Owner, Content)` names. `Markup::AddField` adds a field every node
may give. Read node values with `Markup::GetText`, `GetNumber`, `GetBool` and `GetField`.

## 32. Common mistakes

| Mistake | What happens | Do instead |
|---|---|---|
| Binding a `Progress`'s `Maximum` | Ignored, the bar fills against 1. | Feed a share from 0 to 1. |
| A `Listbox` of thousands of items | Every item is made up front. | Use a `Repeater`. |
| `Combobox` `Selected` expected to skip hidden items | It counts them. | Index by the full list. |
| Expecting `SetValue` in code to run `OnChange` | Setters are silent. | Call your handler yourself. |
| A `Scroller` holding paragraphs | Lines never break (`Horizontal` is `true`). | `"Horizontal": false`. |
| Hiding a window with `SetState(State::Hidden, ...)` | No exit animation, no close sound, not raised again. | `Window::Close`, `Window::Show`. |
| A menu item's `Shortcut` expected to work | It is display text. | Add a real chord in `Shortcuts` or `AddShortcut`. |

## 33. Where it lives

| What | Path |
|---|---|
| Registration of every widget | `Module/UI.Control/Source/UI.Control/Module.cpp` |
| Each widget (fields in its `Build`) | `Module/UI.Control/Source/UI.Control/Widget/<Name>.cpp`, `.hpp` |
| Shared helpers, `Selected` and `Checked` states | `Module/UI.Control/Source/UI.Control/Widget/Common.hpp` |
| Signals | `Module/UI.Core/Source/UI.Core/Event.hpp` |
| Every part's look | `Module/UI.Control/Theme/Default.theme` |
