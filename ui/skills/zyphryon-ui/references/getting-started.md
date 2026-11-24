# Getting started: hosting the UI in an application

This chapter takes a Zyphryon application from nothing to an interface on screen: the CMake setup, the five
libraries, the objects an application keeps, the start-up order, the frame loop, display scale, input shared between
the interface and the game, and how to build elements in code. Everything here follows the library's own example
application (`Example/Source/Example.hpp`), which is the reference when in doubt.

## 1. What you are building

ZyUI is a **retained** interface. You build a tree of elements once (from layouts or from code), change values in a
model as the game runs, and call `Update` and `Draw` every frame. Nothing is rebuilt per frame. A frame where nothing
changed costs next to nothing.

The application owns five objects:

| Object | Type | Lifetime |
|---|---|---|
| The interface | `ZyPlugin::UI::Interface` | The whole run. Holds every pane and element. |
| The canvas | `ZyPlugin::UI::Canvas` | The whole run. `Initialize` at start, `Dispose` at the end. |
| The theme | `ZyPlugin::UI::Theme` | Must outlive the interface's use of it (`SetTheme` keeps a pointer). |
| The markup | `ZyPlugin::UI::Markup` | Must outlive every element it built (bound images load through it later). |
| The model and lexicon | `ZyPlugin::UI::Model`, `ZyPlugin::UI::Lexicon` | Must outlive the interface's use of them. |

Keep all of them as members of your application class. The example keeps the markup in a `Unique<Markup>` because
its constructor takes the content service, which exists only once the kernel is initialized.

## 2. CMake

Add the repository and link `ZyUI`. The UI fetches the engine itself.

```cmake
INCLUDE(FetchContent)

FetchContent_Declare(ZyUI GIT_REPOSITORY https://github.com/Zyphryon/UI GIT_TAG main)
FetchContent_MakeAvailable(ZyUI)

ZyAddTarget(MyGame EXECUTABLE SOURCE "Source")

TARGET_LINK_LIBRARIES(MyGame PRIVATE ZyUI)

ZyApplyOptions(MyGame)
ZyApplyResources(MyGame)
ZyApplyModules(MyGame)
```

`ZyAddTarget`, `ZyApplyOptions`, `ZyApplyResources` and `ZyApplyModules` come from the engine's CMake folder and are
the same calls the UI's own example makes (`CMakeLists.txt`, section "Example").

**`ZyApplyResources` is required.** `UI.Core` registers its `Resources` folder (the canvas shaders under
`Technique/UI/`) with `ZyRegisterResources`. `ZyApplyResources(MyGame)` generates
`<build>/_Generated/MyGame.Embedded.hpp` and `.cpp`, which declare:

```cpp
void ZyRegisterEmbedded(Ref<ZyContent::Service> Service);
```

Call it before the canvas initializes, or the canvas cannot load `Embedded://Technique/UI/Canvas.vfx` and draws
nothing.

### The five libraries

| Library | Gives | Leave out when |
|---|---|---|
| `ZyUI_Core` | The interface, elements, styles, layout, drawing, input. | Never. |
| `ZyUI_Data` | `Model`, `Binder`, `Lexicon`. | You bind nothing and speak one language. |
| `ZyUI_Markup` | Reading `.theme` and `.ui` files. | You build every element in code. |
| `ZyUI_Gesture` | Drag and drop, tooltips, context menus, layout shortcuts. | You need none of them. |
| `ZyUI_Control` | Every widget and the default theme. | You bring all your widgets yourself. |

`ZyUI` links all five. Link them one by one only when you know you need fewer.

### Include roots

| Module | Include as |
|---|---|
| Core | `<UI.Core/Interface.hpp>`, `<UI.Core/Element.hpp>`, `<UI.Core/Facet/Visual/Look.hpp>`, ... |
| Data | `<UI.Data/Model.hpp>`, `<UI.Data/Binder.hpp>`, `<UI.Data/Lexicon.hpp>` |
| Markup | `<UI.Markup/Markup.hpp>` |
| Gesture | `<UI.Gesture/Drag.hpp>`, `<UI.Gesture/Tooltip.hpp>` |
| Control | `<UI.Control/Module.hpp>`, `<UI.Control/Widget/Window.hpp>`, one header per widget |

Everything is in the namespace `ZyPlugin::UI`.

## 3. Files on disk

A small game keeps its UI files under one content mount:

```
Resources/
  Font/Body.fnt              a font baked by Studio from a TTF or OTF
  Image/Frame.tex            textures baked by Studio
  Theme/Game.theme           the game's look, laid over the default theme
  Layout/Game.ui             the game's screens (templates)
  Language/English.lang      one string table per language
  Language/Spanish.lang
  Sound/Click.snd            sounds a style names (optional)
```

Only the addresses inside your own files matter, so any layout of folders works. The example mounts its folder as
`Resources` and every address reads `Resources://...`.

## 4. A minimal host application

The application is a `ZyRuntime::Kernel`. It sets the window up in `OnConfigure`, builds the UI in `OnInitialize`,
draws in `OnTick`, and lets go in `OnTerminate`.

`Source/Game.hpp`:

```cpp
#pragma once

#include "MyGame.Embedded.hpp"
#include <UI.Control/Module.hpp>
#include <UI.Control/Widget/Window.hpp>
#include <UI.Core/Interface.hpp>
#include <UI.Data/Binder.hpp>
#include <UI.Data/Lexicon.hpp>
#include <UI.Data/Model.hpp>
#include <UI.Markup/Markup.hpp>
#include <Zyphryon.Content/Mount/Disk.hpp>
#include <Zyphryon.Platform/Service.hpp>
#include <Zyphryon.Runtime/Kernel.hpp>

namespace Game
{
    using namespace ZyPlugin::UI;

    class App final : public ZyRuntime::Kernel
    {
    protected:

        void OnConfigure(Ref<ZyRuntime::Startup> Startup) override
        {
            ZyRuntime::Startup::Window Window = Startup.GetWindow();
            Window.Title  = "My Game";
            Window.Width  = 1280;
            Window.Height = 720;
            Startup.SetWindow(Window);

            // A plain 8-bit target, so the canvas writes colours as the theme gives them.
            ZyRuntime::Startup::Graphic Graphic = Startup.GetGraphic();
            Graphic.ColorFormat = ZyGraphic::TextureFormat::RGBA8UIntNorm;
            Startup.SetGraphic(Graphic);
        }

        Bool OnInitialize() override
        {
            ConstRetainer<ZyContent::Service> Content = GetService<ZyContent::Service>();
            ZyRegisterEmbedded(* Content);                                       // the canvas shaders
            Content->AddMount("Resources", Retainer<ZyContent::Disk>::Create("Resources"));

            mCanvas.Initialize(* this, Colorspace::Linear);

            // Widgets, their states and the default theme, then the default look until the game's arrives.
            mMarkup = Unique<Markup>::Create(AddressOf(* Content));
            ControlModule::Declare(* mMarkup);

            if (Str Error; !mMarkup->Dress(mTheme, Error))
            {
                LOG_W("UI: the default theme cannot be read: {0}", Error);
            }
            mInterface.SetTheme(AddressOf(mTheme));

            // Input, the clipboard, the display scale and the style sounds.
            Ref<ZyInput::Service>    Input    = * GetService<ZyInput::Service>();
            Ref<ZyPlatform::Service> Platform = * GetService<ZyPlatform::Service>();
            mInterface.Attach(* this, Input, AddressOf(Platform.GetWindow()));

            // The gestures and the data binder on the interface itself.
            ControlModule::Declare(mInterface);

            Binder::SetModel(mInterface, AddressOf(mModel));
            Binder::SetLexicon(mInterface, AddressOf(mLexicon));
            mLexicon.SetLanguage("English");

            mInterface.OnClick("Menu.Quit", [this](Element)
            {
                mModel.SetText("Menu.Status", "Bye");
            });

            Load(* Content);
            return true;
        }

        void OnTick(Real64 Delta) override
        {
            ConstRef<ZyPlatform::Window> Window = GetService<ZyPlatform::Service>()->GetWindow();

            const Real32 Width  = static_cast<Real32>(Window.GetWidth());
            const Real32 Height = static_cast<Real32>(Window.GetHeight());

            ConstRetainer<ZyGraphic::Service> Graphics = GetService<ZyGraphic::Service>();

            Graphics->Prepare(ZyGraphic::kDisplay, "Interface", ZyGraphic::Viewport(0.0f, 0.0f, Width, Height),
                Color(0.07f, 0.08f, 0.10f, 1.0f), 1.0f, 0);
            {
                mInterface.Update(Vector2(Width, Height), static_cast<Real32>(Delta));

                mCanvas.Begin(static_cast<UInt16>(Width), static_cast<UInt16>(Height));
                mInterface.Draw(mCanvas);
                mCanvas.End();
            }
            Graphics->Commit();
        }

        void OnTerminate() override
        {
            mInterface.Detach(* GetService<ZyInput::Service>());
            mCanvas.Dispose();
        }

    private:

        void Load(Ref<ZyContent::Service> Content)
        {
            Content.Read("Resources://Language/English.lang", ZyJob::Lane::Main,
                ZyContent::Mount::OnRead::Create([this](Filesystem::Result Result, Blob Data)
                {
                    Str Error;
                    const Text Source(Data.GetData<Char>(), Data.GetSize());

                    if (Result != Filesystem::Result::Success || !mLexicon.Read("English", Source, Error))
                    {
                        LOG_W("UI: English cannot be read: {0}", Error);
                    }
                }));

            Content.Read("Resources://Theme/Game.theme", ZyJob::Lane::Main,
                ZyContent::Mount::OnRead::Create([this](Filesystem::Result Result, Blob Data)
                {
                    Str Error;
                    const Text Source(Data.GetData<Char>(), Data.GetSize());

                    if (Result != Filesystem::Result::Success || !mMarkup->Dress(mTheme, Source, Error))
                    {
                        LOG_W("UI: the theme cannot be read: {0}", Error);
                    }
                }));

            Content.Read("Resources://Layout/Game.ui", ZyJob::Lane::Main,
                ZyContent::Mount::OnRead::Create([this](Filesystem::Result Result, Blob Data)
                {
                    Str Error;
                    const Text Source(Data.GetData<Char>(), Data.GetSize());

                    if (Result != Filesystem::Result::Success || !mMarkup->Read(Source, Error))
                    {
                        LOG_W("UI: the layout cannot be read: {0}", Error);
                        return;
                    }
                    mMarkup->Spawn(mInterface, "MainMenu");
                }));
        }

    private:

        Canvas         mCanvas;
        Interface      mInterface;
        Model          mModel;
        Lexicon        mLexicon;
        Theme          mTheme;
        Unique<Markup> mMarkup;
    };
}
```

`Source/Game.cpp`:

```cpp
#include "Game.hpp"
#include <Zyphryon.Runtime/Entry.hpp>

ZY_APPLICATION(Game::App)
```

### The start-up order, and why

| Step | Call | Why it is here |
|---|---|---|
| 1 | `ZyRegisterEmbedded(* Content)` | The canvas's technique is an embedded resource. |
| 2 | `Content->AddMount(...)` | Every `Resources://` address in themes and layouts resolves through it. |
| 3 | `Canvas::Initialize(Host, Colorspace)` | Loads the technique and makes its textures. |
| 4 | `Markup` constructed with the content service | Themes load fonts and images through it. |
| 5 | `ControlModule::Declare(Markup)` | Registers every widget, declares the states and bands a theme names, adds the default theme. Declaring twice does nothing. |
| 6 | `Markup::Dress(Theme, Error)` | The default theme alone, so widgets have a look before your theme lands. |
| 7 | `Interface::SetTheme(&Theme)` | The interface dresses from this theme from now on, including later `Dress` calls into it. |
| 8 | `Interface::Attach(Host, Input, Window)` | Hooks pointer and keyboard events, the clipboard, the display scale and style sounds. |
| 9 | `ControlModule::Declare(Interface)` | Declares the data binder, drag and drop, tooltips and context menus on this interface. |
| 10 | `Binder::SetModel`, `Binder::SetLexicon` | Bindings read from these. |
| 11 | `Bind`, `OnClick`, `OnChange` | Actions are names, so they may be bound before or after the templates that name them. |
| 12 | Read layouts, then `Spawn` | A template exists only once its file was read. |

### The colour space

`Canvas::Initialize(Host, Space)` takes the colour space of the target it draws into:

| `Colorspace` | Use when |
|---|---|
| `Linear` | The target stores what is written, as a plain 8-bit display (the example's setting). Theme colours reach the screen as written. |
| `sRGB` | The target is an sRGB format that encodes on write. The canvas decodes colours first so they survive the round trip. |
| `sRGBUnmanaged` | The target stores sRGB but nothing encodes it, so the canvas encodes itself. |

`Canvas::SetColorspace` changes it later.

## 5. The frame

Each frame calls, in this order and inside a render pass:

```cpp
mInterface.Update(Vector2(Width, Height), Delta);   // target size in pixels, seconds since the last frame
mCanvas.Begin(static_cast<UInt16>(Width), static_cast<UInt16>(Height));
mInterface.Draw(mCanvas);
mCanvas.End();
```

`Update` shows changed bound values and texts, makes rows that came into view, runs animations, restyles and lays out
every pane that changed. `Draw` records every pane back to front into the canvas, and `End` uploads and submits the
batches. A `Delta` of zero moves no animation.

To draw the UI over a game world, draw the world first in the same frame and call the interface's `Draw` last, so it
lands on top. To show a render target of your own inside the UI (a 3D view, a minimap), see `Image::CreateView` in
`text.md`.

## 6. Panes, layers and the first element

A **pane** is a root element standing in a **band** (`Layer`). Higher bands draw over lower ones and get the pointer
first.

| Name in a layout | Constant | Order | What the band does |
|---|---|---|---|
| `Base` | `Layers::kBase` | 32 | Plain panes, HUDs. Takes part in pane-to-pane keyboard moves. |
| `Window` | `Window::kLayer` | 64 | Windows. The markup's default band for a template naming none. |
| `Popup` | `Layers::kPopup` | 96 | Menus and dropdown lists. Closes on a press outside or on escape. |
| `Modal` | `Layers::kModal` | 128 | Holds pointer and keyboard while shown, over a backdrop wearing `Modal.Backdrop`. |
| `Tooltip` | `Tooltip::kLayer` | 192 | Tooltips. Never hit. |
| `Drag` | `Drag::kLayer` | 224 | The ghost under the pointer while dragging. Never hit. |

A game may name a band of its own:

```cpp
Layers::Declare("Hud", Layer { 40, Layer::Trait::Cycled });   // then "Layer": "Hud" in a layout
```

Declare it before you spawn a template naming it. A `Layer` is an order (`UInt8`) plus traits (`None`, `Cycled`,
`Popup`, `Modal`, `Overlay`).

Opening a pane in code:

```cpp
const Element Card(mInterface, mInterface.Open(Window::kLayer));   // Tree::Open returns a Handle
```

`Markup::Spawn(Interface, Template)` opens a pane in the band the template's root names (`"Layer": "Base"`), or in
`Window` when it names none, and returns the root. A template whose root is a `Window` or `Dialog` returns the window,
which opened a pane of its own.

To close a pane for good, `Element::Destruct()` its root. To hide it, `Root.SetState(State::Hidden, true)`; for a
window or dialog use `Window::Close` and `Dialog::Close`, which also play the exit animation and sound.

## 7. Display scale

Layouts are in **units**, not pixels. One unit covers `GetPixelRatio()` pixels, which is the display's scale
(`SetDensity`) times the player's choice (`SetScale`). A window 300 units wide covers 450 pixels at 150%.

```cpp
// At start, from the monitor holding the middle of the window.
ConstRef<ZyPlatform::Window> Window = Platform.GetWindow();
const SInt32 X = Window.GetX() + static_cast<SInt32>(Window.GetWidth() / 2);
const SInt32 Y = Window.GetY() + static_cast<SInt32>(Window.GetHeight() / 2);

if (const ConstPtr<ZyPlatform::Monitor> Found = Platform.GetDisplay().GetMonitor(X, Y))
{
    mInterface.SetDensity(Found->GetScale());
}

mInterface.SetScale(1.25f);   // the player's choice, as from a settings screen
```

Once attached, the interface hears the input service's `OnWindowScale` and follows the window to monitors of another
DPI. `Interface::Update` still takes the target's size in **pixels**. `Interface::GetSize()` gives it back in units.

## 8. Input between the interface and the game

`Interface::Attach` adds the interface's handlers to the input service's events. Those events are
`MulticastDelegate`s that the input service runs with `Propagate`: handlers run in the order they were added and stop
at the first that returns `true`. The interface returns `true` when it took the event:

| Event | The interface takes it when |
|---|---|
| `OnMouseDown`, `OnMouseUp` | The press lands on the interface. |
| `OnMouseMove`, `OnMouseScroll` | The pointer rests on the interface. |
| `OnKeyDown`, `OnKeyRepeat` | A shortcut or the control holding the keyboard took the key, or the key moved the keyboard. |
| `OnKeyType` | A control taking text or a modal pane holds the keyboard. |

So **attach the interface before you add your own game hooks**. A hook you add afterwards hears only what the
interface did not take:

```cpp
Input.OnKeyDown.AddMethod<&App::OnKey>(this);     // after mInterface.Attach(...)

Bool App::OnKey(ZyInput::Key Key)
{
    if (Key == ZyInput::Key::I)
    {
        ToggleBag();
        return true;
    }
    return false;
}
```

`OnKeyUp` is not hooked by the interface.

**Polling is different.** `IsKeyHeld` and friends read the device state, which every event updates whether the
interface took it or not. A player typing `W` into a text field also holds `W`. Guard polled game input:

| Check | Means |
|---|---|
| `mInterface.IsTyping()` | A text field holds the keyboard. Skip movement keys and hotkeys. |
| `mInterface.GetFocused().IsValid()` | Some control holds the keyboard (arrows and enter belong to it). |
| `mInterface.GetOver().IsValid()` | The pointer rests on the interface, so a click is the UI's. |
| `mInterface.GetPressed().IsValid()` | The left button went down on a control and is still held. |
| `mInterface.GetModal().IsValid()` | A modal pane is shown and holds pointer and keyboard. |

```cpp
if (!mInterface.IsTyping() && !mInterface.GetModal().IsValid())
{
    // Read WASD here.
}
```

(`Handle::IsValid` is the handle's own check; `Element::IsAlive` is the check on an element.)

### Built-in keys

Once no control keeps the key:

| Key | Does |
|---|---|
| `Tab`, shift `Tab` | The next or previous control, in layout order, wrapping around. |
| arrows | The nearest control that way on screen. |
| `Enter`, `Space` | Clicks the control holding the keyboard. |
| `Escape` | Closes the popup or modal pane in front, or lets go of a drag. |
| `PageUp`, `PageDown`, `Home`, `End` | Scroll whatever scrolls around the control holding the keyboard. |

### Gamepads

Pads drive the interface through calls, not events. Map pad buttons to actions with the input service and call:

```cpp
mInterface.Navigate(Heading::Down);   // the d-pad, as the arrow keys (Up, Down, Left, Right)
mInterface.Activate();                // confirm, as enter
mInterface.Cancel();                  // back, as escape
mInterface.NextFocus();               // shoulders, as tab
mInterface.PreviousFocus();           // and shift tab
```

`Navigate` with nothing holding the keyboard gives it to the first control, so a pad needs no pointer to start.
`mInterface.Focus(Element.GetHandle())` gives the keyboard to one control.

## 9. Building elements in code

Layouts are the usual way, but every widget has a `Create` (windows and dialogs an `Open`), and any element can be
built by hand from facets. The example's left card is built this way.

```cpp
// A pane holding a card placed at (24, 24), 360 wide.
const Element Card(mInterface, mInterface.Open(Window::kLayer));

Ref<Box> Shape = Card.EditBox();
Shape.Placement = Position::Absolute;
Shape.Left      = Length::Pixels(24.0f);
Shape.Top       = Length::Pixels(24.0f);
Shape.Width     = Length::Pixels(360.0f);
Shape.Padding   = Edges(16.0f);
Shape.Gap       = 10.0f;

Ref<Look> Shade = Card.Add<Look>();          // <UI.Core/Facet/Visual/Look.hpp>
Shade.Background  = Color(1.0f, 1.0f, 1.0f, 0.06f);
Shade.Border      = Color(0.23f, 0.25f, 0.31f, 1.0f);
Shade.BorderWidth = 1.0f;
Shade.Radius      = 10.0f;

// Widgets made in code wear their default style unless given one.
Label::Create(Card, "Inventory", "Title");

// Text set in code is shown as written. To follow the language or the model, bind it as a layout would.
const Element Gold = Label::Create(Card, "");
Binder::Attach(Gold, "@Card.Gold", Binder::PutText<&Label::SetText>());
const Element Attack = Button::Create(Card, "Attack");
Attack.Obtain<Control>().Action = Identify("Attack");   // what "Action": "Attack" does in a layout
```

`Element` calls used most:

| Call | Does |
|---|---|
| `Create(Name = Text(), Before = Element())` | Makes a child, optionally named and placed before a sibling. |
| `Destruct()` | Destroys the element and all it holds. |
| `Find(Name)` | The first element below it of that `Name`, at any depth. |
| `GetParent()`, `GetFirst()`, `GetNext()`, `GetRoot()` | Walk the tree. |
| `Children(Callback)` | Calls back with each child. |
| `SetState(State, Bool)`, `HasState(State)` | Turn a state on or off, or read it. |
| `SetStyle(Text)`, `SetName(Text)` | Change the worn style or the name. |
| `EditBox()`, `GetFrame()` | Edit the layout box, or read where it was last laid out (units). |
| `Add<T>()`, `Obtain<T>()`, `Edit<T>()`, `Get<T>()`, `Has<T>()`, `Remove<T>()` | Facets: add, get or add, get to change, read. |
| `Observe(Handler)` | Runs code on every event reaching the element (see `binding.md`). |
| `IsAlive()`, `GetHandle()`, `GetTree()` | Checks and access. |

The core facets: `Box` (layout, always present), `Look` (background, border, radius), `Caption` (text), `Picture`
(image), `Bar` (a fill share, as a progress bar), `Scroll` (scrolling with bars), `Sketch` (custom draw code),
`Control` (answers pointer and keyboard). Headers are under `<UI.Core/Facet/...>`.

A theme style overrides facet values the style gives, so an element built in code that also wears a style looks as
the style says.

## 10. Common mistakes

| Mistake | What happens | Do instead |
|---|---|---|
| No `ZyApplyResources` / `ZyRegisterEmbedded` | The canvas has no technique and nothing draws. | Add both. |
| No `Body` font | Labels have no typeface and draw nothing. | `"Fonts": { "Body": "..." }` in your theme, or `Markup::SetFont("Body", Font)`. |
| `Spawn` before the layout was read | `No template is named '...'`, an element naming nothing comes back. | Spawn in the layout read's callback. |
| `ControlModule::Declare(Interface)` left out | No bindings show, no tooltips, drag and drop or context menus. | Call it once after `Attach`. |
| Game key hooks added before `Attach` | The game hears keys the UI should have taken. | Attach the interface first. |
| `Update` given units | The layout is wrong at any scale but 100%. | Pass the target's size in pixels. |
| `Markup` destroyed after spawning | Bound images load through a dead markup. | Keep it for the whole run. |
| Theme as a local variable | `SetTheme` holds a dangling pointer. | Keep the `Theme` as a member. |

## 11. Where it lives

| What | Path |
|---|---|
| Host application, start to end | `Example/Source/Example.hpp` |
| `ZY_APPLICATION` entry | `Example/Source/Example.cpp` |
| Build setup of the example | `CMakeLists.txt` |
| Interface | `Module/UI.Core/Source/UI.Core/Interface.hpp` |
| Tree (panes, `Open`, states, facets) | `Module/UI.Core/Source/UI.Core/Tree.hpp` |
| Element | `Module/UI.Core/Source/UI.Core/Element.hpp` |
| Bands | `Module/UI.Core/Source/UI.Core/Layers.hpp`, `Types.hpp` (`Layer`, `State`, `Colorspace`) |
| Canvas | `Module/UI.Core/Source/UI.Core/Paint/Canvas.hpp` |
| Canvas shaders | `Module/UI.Core/Resources/Technique/UI/` |
| Module declarations | `Module/UI.Control/Source/UI.Control/Module.cpp`, `Module/UI.Gesture/Source/UI.Gesture/Module.cpp` |
