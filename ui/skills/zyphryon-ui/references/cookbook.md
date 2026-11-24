# Cookbook: complete screens

Four screens a game needs, each complete: the theme styles, the layout templates, the language keys and the C++ that
feeds and answers them. They assume the host application of `getting-started.md` (an `App` keeping `mInterface`,
`mModel`, `mLexicon`, `mTheme` and `mMarkup`, with `using namespace ZyPlugin::UI`), one theme file
`Resources/Theme/Game.theme`, one layout file `Resources/Layout/Game.ui` and one language file per language. Merge the
JSON fragments below into those files.

1. A HUD: name, health and mana bars with figures, gold, and an action bar with hotkeys.
2. An inventory: a grid of slots with icons and counts, drag to rearrange, double click to equip, toggled with `I`.
3. A dialogue box with choices: speaker, portrait, line and answers, holding the pointer while open.
4. A settings menu: tabs of audio, video and language settings, applied live, with a confirm on discard.

## 1. A HUD

A plate at the top left showing the player's name and two bars, gold at the top right, and an action bar at the
bottom centre. Everything is bound, so the game only writes numbers.

### Theme

```json
{
  "Styles": {
    "Hud.Plate": {
      "Background": "#201813E6", "Border": "#846844", "BorderWidth": 1, "Radius": 6, "Padding": [12, 9],
      "Shadow": { "Color": "#00000070", "Offset": [0, 4], "Blur": 16 }
    },
    "Hud.Name":    { "Base": "Label", "Tint": "#F4D097", "Size": 19 },
    "Hud.Figures": { "Base": "Label", "Tint": "#F8EEDD", "Size": 12, "Alignment": "Center",
                     "Shadow": { "Color": "#000000C0", "Offset": [0, 1], "Blur": 2 } },
    "Hud.Health":  { "Base": "Progress", "Height": 15, "Radius": 3, "Background": "#3A1C18", "Fill": "#B5402F" },
    "Hud.Mana":    { "Base": "Progress", "Height": 15, "Radius": 3, "Background": "#1B2238", "Fill": "#4A6FC7" },
    "Hud.Gold":    { "Base": "Label", "Tint": "Gold", "Size": 18, "Stroke": "#2A1A08", "StrokeWidth": 1 },
    "Hud.Skill":   { "Base": "Button", "Width": 48, "Height": 48, "Padding": 0, "Radius": 8,
                     "When": { "Disabled": { "Opacity": 0.4 } } },
    "Hud.Skill.Label": { "Base": "Button.Label", "Size": 13 }
  }
}
```

### Layout

```json
{
  "Templates": {
    "HudBar": {
      "Parameters": { "Kind": "Hud.Health", "Value": 0, "Figures": "" },
      "Height": 15,
      "Children": [
        { "Widget": "Progress", "Style": "$Kind", "Value": "$Value", "Labelled": false, "Width": "100%" },
        { "Widget": "Label", "Style": "Hud.Figures", "Text": "$Figures",
          "Placement": "Absolute", "Left": 0, "Right": 0, "Top": -1 }
      ]
    },
    "HudSkill": {
      "Parameters": { "Caption": "", "Locked": false, "Tip": "", "Do": "" },
      "Widget": "Button", "Style": "Hud.Skill", "Text": "$Caption", "Action": "$Do",
      "Disabled": "$Locked", "Tooltip": "$Tip"
    },
    "Hud": {
      "Layer": "Base", "Placement": "Absolute", "Left": 0, "Top": 0, "Right": 0, "Bottom": 0,
      "Hidden": "{!Hud.Ready}",
      "Shortcuts": {
        "1": { "Action": "Hud.Skill1", "Scope": "Anywhere" },
        "2": { "Action": "Hud.Skill2", "Scope": "Anywhere" },
        "3": { "Action": "Hud.Skill3", "Scope": "Anywhere" }
      },
      "Children": [
        { "Style": "Hud.Plate", "Placement": "Absolute", "Left": 22, "Top": 20, "Width": 250, "Gap": 6,
          "Children": [
            { "Widget": "Label", "Style": "Hud.Name", "Text": "{Hud.Name}" },
            { "Widget": "HudBar", "Kind": "Hud.Health", "Value": "{Hud.Health}",
              "Figures": "{Hud.HealthNow:0} / {Hud.HealthMax:0}" },
            { "Widget": "HudBar", "Kind": "Hud.Mana", "Value": "{Hud.Mana}",
              "Figures": "{Hud.ManaNow:0} / {Hud.ManaMax:0}" }
          ] },
        { "Widget": "Label", "Style": "Hud.Gold", "Text": "@Hud.Gold",
          "Placement": "Absolute", "Right": 24, "Top": 20 },
        { "Flow": "Row", "Gap": 6, "Justify": "Center",
          "Placement": "Absolute", "Left": 0, "Right": 0, "Bottom": 24,
          "Children": [
            { "Widget": "HudSkill", "Caption": "1", "Do": "Hud.Skill1", "Locked": "{!Hud.Skill1Ready}",
              "Tip": "@Hud.Strike" },
            { "Widget": "HudSkill", "Caption": "2", "Do": "Hud.Skill2", "Locked": "{!Hud.Skill2Ready}",
              "Tip": "@Hud.Heal" },
            { "Widget": "HudSkill", "Caption": "3", "Do": "Hud.Skill3", "Locked": "{!Hud.Skill3Ready}",
              "Tip": "@Hud.Dash" }
          ] }
      ]
    }
  }
}
```

`Locked` fills the button's `Disabled`, so each skill binds the opposite of its ready flag. The full-screen
root takes no clicks of its own (a plain panel is no control), so clicks between the HUD's parts reach the game.

### Language

```json
{
  "Hud": {
    "Gold":   "{Player.Gold:0} gold",
    "Strike": "Strike ({Hud.StrikeCost:0} mana)",
    "Heal":   "Heal",
    "Dash":   "Dash"
  }
}
```

### Code

```cpp
// In OnInitialize, after the actions of other screens:
mInterface.OnClick("Hud.Skill1", [this](Element) { UseSkill(0); });
mInterface.OnClick("Hud.Skill2", [this](Element) { UseSkill(1); });
mInterface.OnClick("Hud.Skill3", [this](Element) { UseSkill(2); });

// Once the layout is read:
mHud = mMarkup->Spawn(mInterface, "Hud");

// Every frame, before mInterface.Update. Equal values are skipped, so this costs nothing when nothing moved.
void App::ShowHud(ConstRef<Hero> Player)
{
    mModel.SetBool("Hud.Ready", true);
    mModel.SetText("Hud.Name", Player.Name);

    mModel.SetNumber("Hud.HealthNow", Player.Health);
    mModel.SetNumber("Hud.HealthMax", Player.HealthMax);
    mModel.SetNumber("Hud.Health", Player.HealthMax > 0.0 ? Clamp(Player.Health / Player.HealthMax, 0.0, 1.0) : 0.0);

    mModel.SetNumber("Hud.ManaNow", Player.Mana);
    mModel.SetNumber("Hud.ManaMax", Player.ManaMax);
    mModel.SetNumber("Hud.Mana", Player.ManaMax > 0.0 ? Clamp(Player.Mana / Player.ManaMax, 0.0, 1.0) : 0.0);

    mModel.SetNumber("Player.Gold", Player.Gold);
    mModel.SetNumber("Hud.StrikeCost", 12.0);
    mModel.SetBool("Hud.Skill1Ready", Player.Mana >= 12.0);
    mModel.SetBool("Hud.Skill2Ready", Player.HealCooldown <= 0.0);
    mModel.SetBool("Hud.Skill3Ready", Player.DashCooldown <= 0.0);
}
```

Why it is shaped this way:

- The bars read a **share** from 0 to 1, because `Progress`'s range does not bind.
- The figures are formatted by the layout (`{Hud.HealthNow:0}`), so the code writes numbers only and the label
  re-formats only when a whole number changes.
- The pane is in `Base`, so windows draw over it.
- The skill hotkeys are shortcuts with `Scope` `Anywhere`, so they run with the pointer anywhere, and never while a
  text field is being typed in (shortcuts skip typing unless they set `Typing`).

## 2. An inventory

A window with a five-column grid of slots. Each slot shows an icon and a count, lights when equipped, shows a tooltip,
can be dragged onto another slot to swap them, and equips on a double click or enter. `I` toggles the window when no
text field holds the keyboard.

### Theme

```json
{
  "Styles": {
    "Bag":       { "Base": "Repeater", "Gap": 6, "Thumb": "Gold" },
    "Bag.Item":  {
      "Background": "Field", "Border": "#3B404F", "BorderWidth": 1, "Radius": 4, "Transition": 0.1,
      "When": {
        "Hover":     { "Border": "Gold" },
        "Selected":  { "Border": "Gold", "Background": "#2E6BDE55" },
        "Marked":    { "InnerGlow": { "Color": ["Gold", 0.5], "Blur": 6 } },
        "DropHover": { "Border": "Gold", "BorderWidth": 2, "Glow": { "Color": "#F0D98F80", "Blur": 10 } },
        "Dragging":  { "Opacity": 0.35 }
      }
    },
    "Bag.Count": { "Base": "Label", "Size": 13, "Tint": "#FFFFFF", "Alignment": "End",
                   "Shadow": { "Color": "#000000", "Offset": [1, 1], "Blur": 1 } }
  }
}
```

`Bag.Item` is what each row of a repeater styled `Bag` wears. `Marked` (built in) lights equipped slots.

### Layout

```json
{
  "Templates": {
    "BagSlot": {
      "Padding": 4, "Drag": "{Name}", "DragKind": "Item", "Drop": "Item", "Tooltip": "@Bag.SlotTip",
      "Children": [
        { "Widget": "Image", "Source": "{Icon}", "Width": 36, "Height": 36, "Fit": "Contain" },
        { "Widget": "Label", "Style": "Bag.Count", "Text": "{Count:0}", "Hidden": "{!Stacked}",
          "Placement": "Absolute", "Right": 4, "Bottom": 2 }
      ]
    },
    "Bag": {
      "Widget": "Window", "Title": "@Bag.Title", "Left": 40, "Top": 160, "Width": 290,
      "Children": [
        { "Widget": "Repeater", "Name": "Slots", "Style": "Bag", "Action": "Bag.Slots",
          "Height": 240, "Columns": 5, "ItemHeight": 48,
          "Items": "{Bag}", "ItemTemplate": "BagSlot", "RowStates": { "Marked": "Equipped" } },
        { "Flow": "Row", "Justify": "Between", "Children": [
          { "Widget": "Label", "Text": "@Bag.Count" },
          { "Widget": "Label", "Text": "@Hud.Gold" }
        ] }
      ]
    }
  }
}
```

### Language

```json
{
  "Bag": {
    "Title":   "Bag",
    "Count":   "{Bag} items",
    "SlotTip": "{Name} (x{Count:0}), double click to equip"
  }
}
```

### Code

The game keeps its own items. The model's `Bag` list is written by position, so a slot's record index is the item's
index (two stacks of one item may share an id, so `Mirror` by id would not fit).

```cpp
struct Item { Str Name; Str Icon; UInt32 Count; Bool Equipped; };
Sequence<Item> mItems;
Element        mBag;

// OnInitialize, after mInterface.Attach:
Input.OnKeyDown.AddMethod<&App::OnKey>(this);

mInterface.Bind("Bag.Slots", Handler::Create([this](Ref<Interface> Owner, ConstRef<Event> Message, Handle Self)
{
    // A double press or enter on a slot activates its record.
    if (Message.Kind == Signal::kActivate && Message.Target == Self)
    {
        Equip(static_cast<UInt32>(Repeater::GetIndex(Element(Owner, Self))));
        return true;
    }
    return false;
}));

// Once the layout is read:
mBag = mMarkup->Spawn(mInterface, "Bag");
Window::Close(mInterface, mBag.GetHandle());                   // windows open shown

mBag.Find("Slots").Observe(Handler::Create([this](Ref<Interface> Owner, ConstRef<Event> Message, Handle)
{
    if (Message.Kind != Drag::kDrop)
    {
        return false;
    }
    const Element Slots = mBag.Find("Slots");
    const SInt32  From  = Repeater::GetRecord(Slots, Element(Owner, Message.Other));
    const SInt32  To    = Repeater::GetRecord(Slots, Element(Owner, Message.Target));

    if (From < 0 || To < 0 || From == To)
    {
        return false;
    }
    Item Held     = Move(mItems[From]);
    mItems[From]  = Move(mItems[To]);
    mItems[To]    = Move(Held);
    ShowBag();
    return true;
}));

ShowBag();

// Write the items whenever they change (on loot, on use, after a swap).
void App::ShowBag()
{
    mModel.Resize("Bag", static_cast<UInt32>(mItems.GetSize()));

    for (UInt32 Index = 0; Index < mItems.GetSize(); ++Index)
    {
        ConstRef<Item>   Piece  = mItems[Index];
        const Ptr<Model> Record = mModel.GetRecord("Bag", Index);
        Record->SetText("Name", Piece.Name);
        Record->SetText("Icon", Piece.Icon);                    // "Resources://Icon/Sword.tex"
        Record->SetNumber("Count", Piece.Count);
        Record->SetBool("Stacked", Piece.Count > 1);
        Record->SetBool("Equipped", Piece.Equipped);
    }
}

void App::Equip(UInt32 Index)
{
    if (Index < mItems.GetSize())
    {
        mItems[Index].Equipped = !mItems[Index].Equipped;
        ShowBag();
    }
}

// Hooked after the interface, so a key a text field took never reaches it.
Bool App::OnKey(ZyInput::Key Key)
{
    if (Key != ZyInput::Key::I || !mBag.IsAlive())
    {
        return false;
    }

    if (mBag.HasState(State::Hidden))
    {
        Window::Show(mInterface, mBag.GetHandle());             // unhide and raise
    }
    else
    {
        Window::Close(mInterface, mBag.GetHandle());
    }
    return true;
}
```

Points to notice:

- The slot template shows only bound values, because rows are reused as the grid scrolls.
- The drop is observed on the repeater, since `Drag::kDrop` bubbles from the slot it landed on. `Message.Other` is the
  dragged slot.
- `Repeater::GetIndex` gives the activated record because the press that activates also picks it.
- The window hides on its close button and on `I`; it is never destroyed, so its scroll position stays.

## 3. A dialogue box with choices

A box at the bottom of the screen in the `Modal` band, so it holds pointer and keyboard and the world gets no clicks
while it is open. A portrait, the speaker's name, a line, and a list of answers. Each answer leads to another line or
ends the talk.

### Theme

```json
{
  "Styles": {
    "Talk.Box":     { "Background": "#1A1410F0", "Border": "#846844", "BorderWidth": 1, "Radius": 8,
                      "Padding": [18, 14], "Gap": 14, "Enter": "Drop", "Exit": "FadeOut" },
    "Talk.Speaker": { "Base": "Label", "Tint": "Gold", "Size": 18 },
    "Talk.Line":    { "Base": "Label", "Size": 16 },
    "Talk.Choice":  { "Base": "Button", "Background": "#00000000", "Shadow": { }, "Padding": [8, 4],
                      "When": { "Hover": { "Background": "#FFFFFF14" }, "Focus": { "Background": "#FFFFFF14" } } },
    "Talk.Choice.Label": { "Base": "Button.Label", "Alignment": "Start", "Tint": "#E8DCC4" }
  }
}
```

### Layout

```json
{
  "Templates": {
    "TalkChoice": {
      "Widget": "Button", "Style": "Talk.Choice", "Action": "Talk.Pick", "Text": "{Text}", "Justify": "Start"
    },
    "Talk": {
      "Layer": "Modal", "Style": "Talk.Box", "Flow": "Row",
      "Placement": "Absolute", "Left": "15%", "Right": "15%", "Bottom": 40,
      "Children": [
        { "Widget": "Image", "Source": "{Talk.Portrait}", "Width": 96, "Height": 96, "Fit": "Cover" },
        { "Grow": 1, "Gap": 8, "Children": [
          { "Widget": "Label", "Style": "Talk.Speaker", "Text": "{Talk.Speaker}" },
          { "Widget": "Label", "Style": "Talk.Line", "Text": "{Talk.Line}" },
          { "Widget": "Repeater", "Name": "Choices", "Height": 120, "ItemHeight": 30,
            "Items": "{TalkChoices}", "ItemTemplate": "TalkChoice" }
        ] }
      ]
    }
  }
}
```

Four answers fit in 120 units at 30 each; more scroll.

### Code

The lines here live in a table in code to keep the example short. A real game reads them from a JSON file of its own
and keeps one file per language, since text written into the model is shown as written.

```cpp
struct Answer { Text Words; UInt32 Next; };
struct Line   { Text Words; Array<Answer, 3> Answers; UInt32 Count; };

static constexpr UInt32 kEnd = 0xFFFFFFFF;

static ConstPtr<Line> FindLine(UInt32 Index)
{
    static constexpr Array<Line, 2> kLines =
    {
        Line { "You look pale, traveller. A wolf?",
               { Answer { "It bit me.", 1 }, Answer { "I am fine.", kEnd } }, 2 },
        Line { "Hold still, and the bleeding stops.",
               { Answer { "Thank you.", kEnd } }, 1 },
    };
    return Index < kLines.GetSize() ? &kLines[Index] : nullptr;
}

Element mTalk;
UInt32  mLine = kEnd;

// OnInitialize:
mInterface.OnClick("Talk.Pick", [this](Element Control)
{
    const SInt32         Picked = Repeater::GetRecord(mTalk.Find("Choices"), Control);
    const ConstPtr<Line> Shown  = FindLine(mLine);

    if (Shown && Picked >= 0 && static_cast<UInt32>(Picked) < Shown->Count)
    {
        Say(Shown->Answers[Picked].Next);
    }
});

void App::OpenTalk(Text Speaker, Text Portrait)
{
    mModel.SetText("Talk.Speaker", Speaker);
    mModel.SetText("Talk.Portrait", Portrait);

    if (!mTalk.IsAlive())
    {
        mTalk = mMarkup->Spawn(mInterface, "Talk");
    }
    mTalk.SetState(State::Hidden, false);
    Say(0);
}

void App::Say(UInt32 Index)
{
    const ConstPtr<Line> Shown = FindLine(Index);
    mLine = Index;

    if (!Shown)
    {
        mTalk.SetState(State::Hidden, true);                    // plays Talk.Box's Exit
        return;
    }

    mModel.SetText("Talk.Line", Shown->Words);
    mModel.Resize("TalkChoices", Shown->Count);

    for (UInt32 Choice = 0; Choice < Shown->Count; ++Choice)
    {
        mModel.GetRecord("TalkChoices", Choice)->SetText("Text", Shown->Answers[Choice].Words);
    }

    // A pad or keyboard player starts on the first answer.
    if (const Element First = Repeater::GetRow(mTalk.Find("Choices"), 0); First.IsAlive())
    {
        mInterface.Focus(First.GetHandle());
    }
}
```

Notes:

- The answers are buttons inside rows, so `Repeater::GetRecord(List, Control)` maps the clicked button to its record.
- `Escape` closes the modal pane in front, so a player can always back out. The talk then stays hidden with `mLine`
  set, and the next `OpenTalk` starts it over.
- While the box is shown, `mInterface.GetModal().IsValid()` is `true`. Skip world input then.
- `Repeater::GetRow` returns a row only once it was made. Right after `Resize` the row may not exist until the next
  `Update`, in which case focus nothing and let `Navigate` pick the first control.

### A yes/no question with `Dialog`

For a fixed question, `Dialog` does everything: centred, modal, enter for the default, escape for the cancel.

```json
"QuitDialog": { "Widget": "Dialog", "Title": "@Quit.Title", "Message": "@Quit.Message", "Buttons": [
    { "Text": "@Quit.Leave", "Action": "Game.Quit", "Default": true },
    { "Text": "@Quit.Stay", "Cancel": true } ] }
```

```cpp
mQuit = mMarkup->Spawn(mInterface, "QuitDialog");
Dialog::Close(mInterface, mQuit.GetHandle());           // dialogs open shown
mInterface.OnClick("Game.Quit", [this](Element) { RequestExit(); });

// On escape with nothing else open:
Dialog::Show(mInterface, mQuit.GetHandle());
```

The second button has no action and only closes the dialog.

## 4. A settings menu

A window with three tabs. Audio sliders and the language pick apply as they move. Video settings apply on **Apply**.
Closing the window with unapplied video changes asks first.

### Theme

```json
{
  "Styles": {
    "Settings.Row":   { "Gap": 12 },
    "Settings.Name":  { "Base": "Label", "Width": 120 },
    "Settings.Value": { "Base": "Label", "Tint": "#99A3B8", "Width": 44, "Alignment": "End" },
    "Settings.Note":  { "Base": "Label", "Size": 13, "Tint": "#99A3B8" }
  }
}
```

### Layout

```json
{
  "Templates": {
    "VolumeRow": {
      "Parameters": { "Caption": "", "Level": 0.5, "Do": "" },
      "Flow": "Row", "Align": "Center", "Gap": 12,
      "Children": [
        { "Widget": "Label", "Style": "Settings.Name", "Text": "$Caption" },
        { "Widget": "Slider", "Value": "$Level", "Action": "$Do", "Grow": 1 }
      ]
    },
    "Settings": {
      "Widget": "Window", "Title": "@Settings.Title", "Action": "Settings.Window",
      "Left": 340, "Top": 120, "Width": 460,
      "Children": [
        { "Widget": "Tabs", "Height": 230, "Selected": "{Settings.Tab}", "Tabs": [
          { "Title": "@Settings.Audio", "Children": [
            { "Widget": "VolumeRow", "Caption": "@Settings.Master", "Level": "{Settings.Master}",
              "Do": "Settings.Master" },
            { "Widget": "VolumeRow", "Caption": "@Settings.Music", "Level": "{Settings.Music}",
              "Do": "Settings.Music" },
            { "Widget": "VolumeRow", "Caption": "@Settings.Effects", "Level": "{Settings.Effects}",
              "Do": "Settings.Effects" },
            { "Widget": "Checkbox", "Text": "@Settings.Mute", "Checked": "{Settings.Muted}" }
          ] },
          { "Title": "@Settings.Video", "Children": [
            { "Flow": "Row", "Align": "Center", "Gap": 12, "Children": [
              { "Widget": "Label", "Style": "Settings.Name", "Text": "@Settings.Quality" },
              { "Widget": "Combobox", "Grow": 1, "Selected": "{Settings.Quality}", "Action": "Settings.Dirty",
                "Items": [ "@Quality.Low", "@Quality.Medium", "@Quality.High" ] }
            ] },
            { "Flow": "Row", "Align": "Center", "Gap": 12, "Children": [
              { "Widget": "Label", "Style": "Settings.Name", "Text": "@Settings.Scale" },
              { "Widget": "Combobox", "Grow": 1, "Selected": "{Settings.Scale}", "Action": "Settings.Scale",
                "Items": [ "100%", "125%", "150%" ] }
            ] },
            { "Widget": "Checkbox", "Text": "@Settings.Fullscreen", "Checked": "{Settings.Fullscreen}",
              "Action": "Settings.Toggle" },
            { "Widget": "Label", "Style": "Settings.Note", "Text": "@Settings.Pending",
              "Hidden": "{!Settings.Pending}" }
          ] },
          { "Title": "@Settings.Language", "Children": [
            { "Widget": "Radio", "Text": "English", "Checked": "{Settings.English}", "Action": "Settings.Speak" },
            { "Widget": "Radio", "Text": "Español", "Checked": "{Settings.Spanish}", "Action": "Settings.Speak" }
          ] }
        ] },
        { "Flow": "Row", "Justify": "End", "Gap": 8, "Children": [
          { "Widget": "Button", "Text": "@Settings.Apply", "Action": "Settings.Apply",
            "Disabled": "{!Settings.Pending}" },
          { "Widget": "Button", "Text": "@Settings.Close", "Action": "Settings.Close" }
        ] }
      ]
    },
    "Discard": {
      "Widget": "Dialog", "Title": "@Settings.Title", "Message": "@Settings.Discard", "Buttons": [
        { "Text": "@Settings.Drop", "Action": "Settings.Drop", "Default": true },
        { "Text": "@Settings.Keep", "Cancel": true } ]
    }
  }
}
```

Each control that needs code names an action. Controls may share one (both radios name `Settings.Speak`), since
every control naming an action runs its one handler. A click and a change need two names, because `OnClick` and
`OnChange` on one name replace each other.

### Language

```json
{
  "Settings": {
    "Title": "Settings", "Audio": "Audio", "Video": "Video", "Language": "Language",
    "Master": "Master", "Music": "Music", "Effects": "Effects", "Mute": "Mute everything",
    "Quality": "Quality", "Scale": "Interface", "Fullscreen": "Fullscreen",
    "Pending": "Video changes wait for Apply.",
    "Apply": "Apply", "Close": "Close",
    "Discard": "Drop the video changes you did not apply?", "Drop": "Drop them", "Keep": "Keep editing"
  },
  "Quality": { "Low": "Low", "Medium": "Medium", "High": "High" }
}
```

### Code

```cpp
struct Video { SInt32 Quality = 1; Bool Fullscreen = false; };
Video   mVideo;                                          // what is applied
Element mSettings;
Element mDiscard;

void App::BindSettings()
{
    // Audio applies live. The slider already wrote the model when OnChange runs.
    mInterface.OnChange("Settings.Master",  [this](Element) { ApplyAudio(); });
    mInterface.OnChange("Settings.Music",   [this](Element) { ApplyAudio(); });
    mInterface.OnChange("Settings.Effects", [this](Element) { ApplyAudio(); });

    // Video waits for Apply.
    mInterface.OnChange("Settings.Dirty",  [this](Element) { NotePending(); });
    mInterface.OnChange("Settings.Toggle", [this](Element) { NotePending(); });

    mInterface.OnChange("Settings.Scale", [this](Element Control)
    {
        static constexpr Array<Real32, 3> kScales(1.0f, 1.25f, 1.5f);
        mInterface.SetScale(kScales[Clamp(Listbox::GetIndex(Control), 0, 2)]);
    });

    // Both radios share the action. Each radio's flag is written before the action runs.
    mInterface.OnChange("Settings.Speak", [this](Element)
    {
        mLexicon.SetLanguage(mModel.GetBool("Settings.Spanish") ? "Spanish" : "English");
    });

    mInterface.OnClick("Settings.Apply", [this](Element)
    {
        mVideo.Quality    = static_cast<SInt32>(mModel.GetNumber("Settings.Quality"));
        mVideo.Fullscreen = mModel.GetBool("Settings.Fullscreen");
        ApplyVideo(mVideo);
        mModel.SetBool("Settings.Pending", false);
    });

    mInterface.OnClick("Settings.Close", [this](Element)
    {
        Window::Request(mInterface, mSettings.GetHandle());   // as the close button: asks first
    });

    // The close button reports Window::kClose before hiding. Returning true keeps the window open.
    mInterface.Bind("Settings.Window", Handler::Create([this](Ref<Interface>, ConstRef<Event> Message, Handle Self)
    {
        if (Message.Kind == Window::kClose && Message.Target == Self && mModel.GetBool("Settings.Pending"))
        {
            Dialog::Show(mInterface, mDiscard.GetHandle());
            return true;
        }
        return false;
    }));

    mInterface.OnClick("Settings.Drop", [this](Element)
    {
        // Put the model back to what is applied, then close for real.
        mModel.SetNumber("Settings.Quality", mVideo.Quality);
        mModel.SetBool("Settings.Fullscreen", mVideo.Fullscreen);
        mModel.SetBool("Settings.Pending", false);
        Window::Close(mInterface, mSettings.GetHandle());
    });
}

void App::SpawnSettings()                                 // once the layout is read
{
    mModel.SetNumber("Settings.Tab", 0.0);
    mModel.SetNumber("Settings.Master", 0.8);
    mModel.SetNumber("Settings.Music", 0.6);
    mModel.SetNumber("Settings.Effects", 0.7);
    mModel.SetBool("Settings.Muted", false);
    mModel.SetNumber("Settings.Quality", mVideo.Quality);
    mModel.SetNumber("Settings.Scale", 0.0);
    mModel.SetBool("Settings.Fullscreen", mVideo.Fullscreen);
    mModel.SetBool("Settings.English", true);
    mModel.SetBool("Settings.Spanish", false);
    mModel.SetBool("Settings.Pending", false);

    mSettings = mMarkup->Spawn(mInterface, "Settings");
    Window::Close(mInterface, mSettings.GetHandle());
    mDiscard = mMarkup->Spawn(mInterface, "Discard");
    Dialog::Close(mInterface, mDiscard.GetHandle());
}

void App::NotePending()
{
    const Bool Changed = static_cast<SInt32>(mModel.GetNumber("Settings.Quality")) != mVideo.Quality
                      || mModel.GetBool("Settings.Fullscreen") != mVideo.Fullscreen;
    mModel.SetBool("Settings.Pending", Changed);
}

void App::ApplyAudio()
{
    const Real64 Master = mModel.GetBool("Settings.Muted") ? 0.0 : mModel.GetNumber("Settings.Master");
    SetMusicVolume(Master * mModel.GetNumber("Settings.Music"));
    SetEffectsVolume(Master * mModel.GetNumber("Settings.Effects"));
}
```

`ApplyVideo`, `SetMusicVolume`, `SetEffectsVolume` and `RequestExit` are the game's own. Save the `Settings.*`
values with your save system and write them back into the model before spawning.

Points to notice:

- Every value a control edits is bound both ways, so the handlers read the model and never the widgets.
- The **Mute** checkbox names no action, since the slider handlers read its flag when they next run. To apply it at
  once, give it an action of its own and call `ApplyAudio` there.
- `Window::Request` closes exactly as the close button does, so the discard question covers both paths.
- The dialog's **Drop them** button runs its action and closes the dialog. **Keep editing** has no action and only
  closes it.
- The interface scale is applied at once and also rescales the settings window under the pointer.

## 5. Where it lives

| What | Path |
|---|---|
| The library's own demo of every widget, with drag and drop between a list and a grid | `Example/Source/Example.hpp`, `Example/Resources/Layout/Controls.ui` |
| The example's theme and languages | `Example/Resources/Theme/Example.theme`, `Example/Resources/Language/` |
