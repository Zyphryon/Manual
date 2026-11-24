# Text, languages, fonts and images

This chapter covers everything the interface shows that is not a box: translated text through the `Lexicon` and
`.lang` files, fonts, how labels set text (breaking, ellipses, outlines, shadows), editing in text fields, images and
nine-slices, GPU textures shown inside the interface, and drawing your own overlays with a `Sketch`.

## 1. Languages

A `Lexicon` (`<UI.Data/Lexicon.hpp>`) holds one string table per language. Any text a layout binds may be written
`"@Key"`, and reads the table of the language spoken.

### The `.lang` file

A JSON object of texts. An object groups keys under its name, at any depth. A text may name model values in braces,
exactly as a layout binding does.

```json
{
  "Menu": {
    "Title":   "The Long Road",
    "Play":    "Play",
    "Options": "Options",
    "Quit":    "Quit"
  },
  "Hud": {
    "Health":  "Health {Player.Health:0} / {Player.HealthMax:0}",
    "Gold":    "{Player.Gold:0} gold"
  },
  "Bag": {
    "Title":   "Bag ({Bag})",
    "Empty":   "Nothing carried"
  }
}
```

Keys are the dotted path: `Menu.Play`, `Hud.Health`. A Spanish file gives the same keys:

```json
{
  "Menu": { "Title": "El largo camino", "Play": "Jugar", "Options": "Opciones", "Quit": "Salir" },
  "Hud":  { "Health": "Vida {Player.Health:0} / {Player.HealthMax:0}", "Gold": "{Player.Gold:0} de oro" },
  "Bag":  { "Title": "Bolsa ({Bag})", "Empty": "No llevas nada" }
}
```

### The API

```cpp
Str Error;
mLexicon.Read("English", EnglishText, Error);   // replaces the language's table, once all of it read cleanly
mLexicon.Merge("English", ModText, Error);      // adds keys, replacing the ones given again
mLexicon.Set("English", "Menu.Play", "Start");  // one key
mLexicon.SetLanguage("Spanish");                // every @Key text switches at the next Update

Binder::SetLexicon(mInterface, AddressOf(mLexicon));
```

| Call | Does |
|---|---|
| `Read(Language, Source, Error)` | Reads a table. Returns `false` with `Error` filled on a mistake, keeping the old table. |
| `Merge(Language, Source, Error)` | Reads a table into the language's, adding and replacing keys. |
| `Set(Language, Key, Value)` | Gives one key a text. |
| `SetLanguage(Language)`, `GetLanguage()` | The language spoken. |
| `HasLanguage(Language)`, `GetLanguages()` | Languages read or given a text, in the order they first were. |
| `Has(Key)` | Whether the language spoken gives the key a text. |
| `Translate(Key)` | The text of a key in the language spoken, or the key itself. |

### Where `@Key` works

Every text field that binds: label and button text, titles of windows, tabs and expanders, tooltips, dialog
messages and button texts, text field hints, menu items and headings, list and combo box items, grid headings.
Parameters of templates may be `@Key` too.

```json
{ "Widget": "Button", "Text": "@Menu.Play", "Tooltip": "@Menu.PlayTip" }
{ "Widget": "Combobox", "Items": [ "@Quality.Low", "@Quality.High" ] }
```

`@@` writes a literal at sign. Switching the language changes every such text at the next update, rebuilding
nothing. A key the spoken language lacks shows the key itself, and is logged once per key and language
(`The language 'Spanish' gives no text for 'Menu.Play', so the key is shown`). A language still loading logs nothing.

Read every language at start, so switching only changes which one is spoken:

```cpp
for (Text Language : { "English"_Text, "Spanish"_Text })
{
    Content.Read(Str::Print("Resources://Language/{0}.lang", Language), ZyJob::Lane::Main,
        ZyContent::Mount::OnRead::Create([this, Name = Str(Language)](Filesystem::Result Result, Blob Data)
        {
            Str Error;
            const Text Source(Data.GetData<Char>(), Data.GetSize());

            if (Result != Filesystem::Result::Success || !mLexicon.Read(Name, Source, Error))
            {
                LOG_W("UI: the language '{0}' cannot be read: {1}", Name, Error);
            }
        }));
}
```

### Text from code is not translated

A text the game writes into the model (`SetText("Hud.Name", "Rowan")`) or into a widget (`Label::SetText`) is shown
as written. Keep player-facing words in the `.lang` files and write only names and numbers from code. When code must
show translated words, either bind the element to a key (`Binder::Attach(Element, "@Bag.Empty", ...)`), or look the
text up with `mLexicon.Translate("Bag.Empty")` and write the result (it will not follow later language switches).

## 2. Fonts

Fonts are `.fnt` files baked by Zyphryon Studio from TTF, OTF or TTC into multi-channel signed distance fields, so
text stays sharp at any size and scale:

```
Studio bake Body.ttf --charset latin1,punctuation
```

A theme names fonts under `Fonts`, and a style picks one with `Font`:

```json
{
  "Fonts":  { "Body": "Resources://Font/Body.fnt", "Display": "Resources://Font/Display.fnt" },
  "Styles": {
    "Label": { "Font": "Body", "Size": 16 },
    "Title": { "Base": "Label", "Font": "Display", "Size": 28 }
  }
}
```

The built-in styles all use `Body`, which the default theme leaves as a slot. Fill it in your theme or with
`Markup::SetFont("Body", Font)`, where `Font` is a `Retainer<ZyRender::Font>` you loaded:

```cpp
mFont = Content->Load<ZyRender::Font>("Resources://Font/Body.fnt");
mMarkup->SetFont("Body", mFont);                    // before Dress
```

Text in a language your font lacks shows nothing for the missing glyphs. Bake every character set you ship
(`--charset`, and `--fallback` for a second face), see the Studio manual.

## 3. How labels set text

| Behaviour | How |
|---|---|
| Line breaking | A label breaks its text onto new lines at its box's width. Give it a `Width`, `Grow` or a stretching parent. |
| Hard breaks | `\n` in the text. |
| Ellipsis | `"Overflow": "Ellipsis"` cuts a too long line short with `…`. |
| Alignment | The style's `Alignment`: `Start`, `Center`, `End`. |
| Size and colour | The style's `Size` and `Tint`. |
| Outline | The style's `Stroke` and `StrokeWidth`. |
| Drop shadow | The style's `Shadow` on a label shadows its glyphs. |

```json
"Hud.Figures": { "Base": "Label", "Size": 12, "Alignment": "Center", "Tint": "#F8EEDD",
                 "Shadow": { "Color": "#000000C0", "Offset": [0, 1], "Blur": 2 } },
"Banner":      { "Base": "Title", "Stroke": "#2A1A08", "StrokeWidth": 2 }
```

A label in a `Scroller` only breaks when the scroller is given `"Horizontal": false`, since a scroller that scrolls
sideways lets its content keep its own width.

## 4. Text fields

`Textbox` (see `widgets.md` for its fields) edits as a desktop field does:

| Input | Does |
|---|---|
| left, right | A character; with control, a word. Shift selects. |
| home, end | Start or end of the line; with control, of the whole text. |
| up, down, page keys | Move by lines in a field of several lines. |
| backspace, delete | The selection, the character, or with control the word. |
| control and A, C, X, V | Select all, copy, cut, paste. |
| a press, a drag | Places the caret or selects. Two presses select a word, three the whole text. |
| enter | Commits (`Signal::kCommit`), or breaks a line when `Multiline`. |
| escape | Puts back the text it started with. |

Copy and paste go through the window given to `Interface::Attach`, or through the interface's own clipboard when it
was given none (`Interface::Copy`, `Interface::Paste`).

A chat input that sends on enter and clears itself:

```json
{ "Widget": "Textbox", "Name": "ChatInput", "Hint": "@Chat.Hint", "Limit": 200, "Action": "Chat.Send" }
```

```cpp
mInterface.Bind("Chat.Send", Handler::Create([this](Ref<Interface> Owner, ConstRef<Event> Message, Handle Self)
{
    if (Message.Kind == Signal::kCommit && Message.Target == Self)
    {
        const Element Field(Owner, Self);
        const Str     Line(Textbox::GetText(Field));

        if (!Line.IsEmpty())
        {
            Send(Line);
            Textbox::SetText(Field, "");
        }
    }
    return false;
}));
```

## 5. Images

### The `Image` widget

Shows a texture of the content (`.tex` baked by Studio):

```json
{ "Widget": "Image", "Source": "Resources://Image/Portrait.tex", "Width": 64, "Height": 64, "Fit": "Cover" }
{ "Widget": "Image", "Source": "Resources://Image/Tiles.tex", "Region": [0, 0, 0.25, 0.25], "Filter": "Point",
  "Width": 32, "Height": 32 }
```

Most of its fields bind, so each row of a list can show its own picture or frame of a sheet:

```json
{ "Widget": "Repeater", "Items": "{Assets}", "ItemHeight": 40, "ItemTemplate": { "Flow": "Row", "Children": [
    { "Widget": "Image", "Source": "{Thumb}", "Layer": "{Slice}", "Region": "{Frame}", "Fit": "Contain",
      "Width": 32, "Height": 32 },
    { "Widget": "Label", "Text": "{Name}" }
] } }
```

A bound `Region` is a text of four numbers, `"0 0 0.5 1"` (spaces or commas). A bound `Tint` is a colour of the model
(`Model::SetColor`). A bound `Source` loads through the markup the first time each address is shown.

An icon kept in your game as an atlas and a pixel frame becomes a `Region` by dividing the frame by the atlas size:

```cpp
Row.SetText("Frame", Str::Print("{0} {1} {2} {3}",
    X / AtlasWidth, Y / AtlasHeight, (X + W) / AtlasWidth, (Y + H) / AtlasHeight));
```

### Pictures in styles and nine-slices

A style's `Image` lays a picture over any element's box without changing its size, so a frame or button skin is a
theme matter, not a layout one:

```json
"Images": { "Frame": "Resources://Image/Frame.tex" },
"Styles": {
  "Window": { "Image": { "Source": "Frame", "Slice": 24 }, "Background": "#00000000" },
  "Slot":   { "Image": { "Source": "Frame", "Slice": [8, 8, 8, 8], "Tint": "#A89F8F" },
              "When": { "Hover": { "Image": { "Tint": "#FFE9A8" } } } }
}
```

`Slice` keeps that many pixels of each edge of the image unscaled and stretches the middle, so one small frame image
fits any size. `Region` picks a part of a sheet, so one texture holds every state of a button.

## 6. GPU textures inside the interface

`Image::CreateView` shows a texture you keep alive, as a 3D viewport, a minimap or a character preview rendered to a
target of your own:

```cpp
const Element View = Image::CreateView(Panel, mPreviewTarget);   // a ZyGraphic::Object
View.EditBox().Grow = 1.0f;
Image::SetFlip(View, true);                                       // when the target comes out upside down
```

`Image::SetTexture(Element, Texture, Layout)` points it at another texture. `Interface::GetPointer(Element)` gives the
pointer from the view's top left corner in units, to map clicks into what the view shows. Give the view
`"Hover": true` (or `Control::Hover`) to hear the pointer move over it.

## 7. Sketches: drawing your own overlays

A `Sketch` facet runs your draw code every frame inside an element's frame, for gizmos, selection boxes, graphs and
crop marks:

```cpp
// void Draw(Ref<Canvas> Canvas, ConstRef<Painter::Brush> Element)
View.Add<Sketch>().Code = Painter::Draw::Create<&Overlay::Draw>(this);

void Overlay::Draw(Ref<Canvas> Canvas, ConstRef<Painter::Brush> Element)
{
    const Rect Box = Element.Frame;   // in pixels
    Canvas.DrawFrame(Box, Color(1.0f, 0.85f, 0.4f, 1.0f), 2.0f, 4.0f);
    Canvas.DrawLine(Box.GetCenter(), Box.GetCenter() + Vector2(40.0f, 0.0f), Color::White(), 1.5f);
}
```

`Sketch::Clip` (default `true`) cuts the drawing to the element. A `Painter::Brush` gives `Frame` and `Inner` (inside
the padding) in pixels, `Opacity`, `Radius`, and the element's `Shade` (`Look`).

The canvas draws: `DrawRect`, `DrawFrame`, `DrawGradient`, `DrawShadow`, `DrawInnerGlow`, `DrawImage`, `DrawSliced`,
`DrawTexture`, `DrawLine`, `DrawPolyline`, `DrawTriangle`, `DrawQuad`, `DrawCircle`, `DrawRing`, `DrawArc`,
`DrawEllipse`, `DrawText`, `DrawGlyphs`, under a stack of `PushClip`/`PopClip` and `PushTransform`/`PopTransform`.

## 8. Common mistakes

| Mistake | What happens | Do instead |
|---|---|---|
| `Label::SetText(Label, "@Menu.Play")` | Shows `@Menu.Play`. | `Binder::Attach(Label, "@Menu.Play", Binder::PutText<&Label::SetText>())`. |
| Writing translated words into the model | They do not follow `SetLanguage`. | Keys in `.lang`, `@Key` in the layout. |
| A key typed differently in two languages | One language shows the key. | Keep the files' keys identical; watch the log. |
| A label with no width in a row | It never breaks, pushing siblings off. | `"Grow": 1` or a `Width`. |
| `Source` given a file path, not an address | Nothing loads. | `"Resources://Image/X.tex"`, as the content mount names it. |
| A font baked without a language's glyphs | Characters missing. | Bake the charset you ship. |

## 9. Where it lives

| What | Path |
|---|---|
| Lexicon | `Module/UI.Data/Source/UI.Data/Lexicon.hpp`, `Lexicon.cpp` |
| Label, Textbox, Image | `Module/UI.Control/Source/UI.Control/Widget/Label.hpp`, `Textbox.hpp`, `Image.hpp` |
| Text facet | `Module/UI.Core/Source/UI.Core/Facet/Visual/Caption.hpp` |
| Picture facet | `Module/UI.Core/Source/UI.Core/Facet/Visual/Picture.hpp` |
| Sketch, painter brush | `Module/UI.Core/Source/UI.Core/Facet/Visual/Sketch.hpp`, `System/Painter.hpp` |
| Canvas | `Module/UI.Core/Source/UI.Core/Paint/Canvas.hpp` |
| Example languages | `Example/Resources/Language/English.lang`, `Spanish.lang` |
