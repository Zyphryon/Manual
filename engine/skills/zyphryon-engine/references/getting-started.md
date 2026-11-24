# Getting started

This chapter is the first stop. It says what Zyphryon is, what each module under `Engine/Source` does, how to add
the engine to a CMake project, how a program boots, and it ends with a complete program that opens a window,
clears it to a colour that changes over time, and ticks a scene with one system. A headless variant of the same
program closes the chapter. Later chapters go deep on each piece: `base.md` (types, containers, JSON, binary),
`math.md` (vectors, angles, matrices), `content.md` (files and assets), `runtime.md` (the kernel, services, jobs,
input, window), `scene.md` (the ECS), `graphics.md` (drivers, shaders, rendering), `network.md` (transport and
replication), `audio.md` (sound) and `ai.md` (behaviour trees).

## 1. What Zyphryon is

Zyphryon is a cross-platform game engine written in C++23. Every subsystem is first-party: containers, strings,
JSON, the job system, the ECS, the GPU layer, the renderer, the audio mixer and the network transport. The C++
standard library appears only inside `Zyphryon.Base`, where the language leaves no choice.

What that means for you as a user:

- You write against the engine's own vocabulary: `UInt32`, `Real32`, `Bool`, `Str`, `Text`, `Sequence<T>`,
  `Table<K, V>`, `Ref<T>`, `ConstRef<T>`, `Retainer<T>`. `base.md` teaches it.
- The build turns off exceptions and RTTI (`-fno-exceptions -fno-rtti`, `/GR-` on MSVC). Your code compiles under
  the same flags, so `try`, `throw`, `dynamic_cast` and `typeid` are out.
- Assets are baked offline into binary formats (`.tex`, `.msh`, `.snd`, ...) that load without conversion at run
  time. Text formats (`.mtl`, `.vfx`, `.mdl`, `.bhv`) are JSON you can write by hand. See `content.md`.
- A program is one class derived from `ZyRuntime::Kernel`. The kernel brings every service up, runs the frame loop,
  and tears everything down. You override four hooks.

Platforms: Windows 10 (Direct3D 11 or OpenGL), Linux on X11 (OpenGL), the web (WebAssembly, WebGL 2), and a headless
build for dedicated servers. There is no macOS or mobile target yet; the Android entry point is a compile error
(`Zyphryon.Runtime/Entry.hpp`).

## 2. The module map

Every engine module is a folder under `Engine/Source/`, and they all build into one static library target,
`ZyEngine`. Graphic drivers are separate targets under `Module/`.

| Folder | Namespace | What it does | Chapter |
|---|---|---|---|
| `Zyphryon.Base` | `ZyBase` (inline) | Aliases, containers, strings, formatting, delegates, JSON, binary reader and writer, LZ4, logging, assertions, profiling macros, filesystem, threads, reflection. | `base.md` |
| `Zyphryon.Math` | `ZyMath` (inline) | SIMD vectors, quaternions, matrices, transforms, boxes, rays, colours, collision tests, curves, easing, tweens, random numbers, noise. | `math.md` |
| `Zyphryon.Engine` | `ZyEngine` | `Subsystem` (the base of every service), `System` (the service registry), `Locator`, `Module`. | `runtime.md` |
| `Zyphryon.Runtime` | `ZyRuntime` | `Kernel`, `Startup`, `Meter`, the `ZY_APPLICATION` entry point. | `runtime.md` |
| `Zyphryon.Platform` | `ZyPlatform` | The window, displays and monitors, the timer. | `runtime.md` |
| `Zyphryon.Input` | `ZyInput` | Keyboard, mouse, touch, named actions and bindings. | `runtime.md` |
| `Zyphryon.Job` | `ZyJob` | The worker pool: `Main`, `Compute` and `IO` lanes. | `runtime.md` |
| `Zyphryon.Content` | `ZyContent` | URIs, mounts (disk, memory, archive, remote), asynchronous typed resources, loaders. | `content.md` |
| `Zyphryon.Scene` | `ZyScene` | The archetype ECS: entities, components, systems, phases, change lists, saves. | `scene.md` |
| `Zyphryon.Graphic` | `ZyGraphic` | The low-level GPU layer: buffers, textures, passes, pipelines, materials, techniques, shaders. | `graphics.md` |
| `Zyphryon.Render` | `ZyRender` | The renderer: render graph, cameras, sprites and sheets, MSDF fonts, models, skeletons, animation, particles, post effects. | `graphics.md` |
| `Zyphryon.Audio` | `ZyAudio` | The mixer, sounds (`.snd`), categories, spatial voices, filters. | `audio.md` |
| `Zyphryon.Network` | `ZyNetwork` | TCP and UDP transport on its own thread, reporting to the main thread. | `network.md` |
| `Zyphryon.Replication` | `ZyReplication` | Replicates the scene to the peers of a session. | `network.md` |
| `Zyphryon.AI` | `ZyAI` | A generic behaviour-tree runtime with a blackboard, and the `.bhv` loader. | `ai.md` |
| `Zyphryon.Navigation` | `ZyNavigation` | Navigation meshes baked from a grid of cells, and path finding. | (not covered) |

| Folder | Target | What it is |
|---|---|---|
| `Module/Graphic/Driver/D3D11` | `ZyModule_GraphicDriverD3D11` | The Direct3D 11 driver. Windows only. Registers itself under the name `D3D11`. |
| `Module/Graphic/Driver/OGL` | `ZyModule_GraphicDriverOGL` | The OpenGL driver: desktop GL and WebGL 2. Registers itself under the name `OpenGL`. |
| `Engine/Resources` | (embedded) | The engine's own shaders and techniques, compiled into your program and served as `Embedded://`. |

A module includes another by its folder name: `#include <Zyphryon.Scene/Service.hpp>`. The include root is
`Engine/Source`, and every header is reached that way.

## 3. Adding the engine to a CMake project

### 3.1 Requirements

| Need | Version |
|---|---|
| CMake | 3.26 or newer |
| Generator | Ninja (the presets use it; other generators work) |
| Compiler | C++23: MSVC 2019+, GCC 11+, Clang 19+ |
| Windows | Windows SDK 10.0+ |
| Linux | `libasound2-dev`, `libx11-dev`, `libxrandr-dev`, and OpenSSL (`FIND_PACKAGE(OpenSSL REQUIRED)`) |
| Web | An activated Emscripten SDK (`EMSDK` set) |

The engine fetches libopus (always, unless headless) and, only with `ZY_PROFILE_BACKEND_TRACY`, the Tracy client.
Nothing else is downloaded. The OpenGL driver loads its own entry points at run time, so no GL loader or
development package is needed.

### 3.2 The CMake helpers

The engine's root `CMakeLists.txt` includes seven helper files from `CMake/`. Once the engine is added to your
build, these functions are available to your own `CMakeLists.txt`:

| Function | Does |
|---|---|
| `ZyAddTarget(<name> [STATIC\|SHARED\|MODULE\|EXECUTABLE] [SOURCE <dir>...] [EXCLUDE <regex>...] [PCH <header>])` | Globs every `.cpp` under the source folders (default `Source`), makes the target, adds the folders as public include roots, and applies `ZyApplyOptions`. The glob uses `CONFIGURE_DEPENDS`, so adding a file needs no manual reconfigure. |
| `ZyApplyOptions(<target>)` | The engine's compiler and linker settings: C++23, no exceptions, no RTTI, SSE4.2 (`/arch:SSE4.2`, `-msse4.2`), `-fvisibility=hidden`, AddressSanitizer in Debug under GCC and Clang, link-time optimization in Release, WebAssembly settings under Emscripten. |
| `ZyApplyModules(<target>)` | Generates `<target>.Modules.cpp`, which defines `ZyRuntime::ZyRegisterModules()` from every module registered with `ZyRegisterModule` (the graphic drivers), and links their libraries. **An application must call it.** |
| `ZyApplyResources(<target>)` | Generates `<target>.Embedded.hpp/.cpp` with `void ZyRegisterEmbedded(Ref<ZyContent::Service> Service)`, which mounts every folder registered with `ZyRegisterResources` as `Embedded://`. |
| `ZyRegisterModule(TARGET <t> HEADER <h> CLASS <c>)` | Registers an engine module so `ZyApplyModules` instantiates it. |
| `ZyRegisterResources(TARGET <t> DIRECTORY <dir> [PREFIX <p>])` | Registers a folder whose files `ZyApplyResources` compiles into each application. |
| `ZyApplyExports(<target> [ALL] [MODULES <targets>...])`, `ZyApplyPlugin(<target> HOST <target>)` | Make a host image that carries the engine's symbols out to plugins loaded beside it, and make a plugin library for that host. See `runtime.md`. |

### 3.3 Build options

| Option | Default | Effect |
|---|---|---|
| `ZY_MODE_HEADLESS` | `OFF` | Compiles out the window, input, graphic, audio and render modules, and skips the graphic drivers. For a dedicated server. |
| `ZY_GRAPHIC_DRIVER_D3D11` | `ON` on Windows | Builds the Direct3D 11 driver. |
| `ZY_GRAPHIC_DRIVER_OPENGL` | `ON` | Builds the OpenGL driver. |
| `ZY_PROFILE_BACKEND_TRACY` | `OFF` | Fetches Tracy v0.13.1 and turns the `ZY_PROFILE*` macros on. |
| `ZY_PROFILE_MEMORY` | `OFF` | Also reports every allocation and free to the profiler. |
| `ZY_WEB_THREAD` | `OFF` | Builds the web target with threads (pthreads, wasm workers, an audio worklet). |

### 3.4 Your `CMakeLists.txt`

Fetch the engine, make an executable, link `ZyEngine`, and apply modules and resources:

```cmake
CMAKE_MINIMUM_REQUIRED(VERSION 3.26)

PROJECT(Hello VERSION 1.0.0 LANGUAGES CXX)

INCLUDE(FetchContent)

FetchContent_Declare(External_ZyEngine
        GIT_REPOSITORY https://github.com/Zyphryon/Engine
        GIT_TAG        main
        GIT_SHALLOW    TRUE)
FetchContent_MakeAvailable(External_ZyEngine)

# Every .cpp under Source/ becomes part of the program.
ZyAddTarget(${PROJECT_NAME} EXECUTABLE SOURCE "Source")

TARGET_LINK_LIBRARIES(${PROJECT_NAME} PRIVATE "ZyEngine")

# Embedded:// (the engine's own shaders and techniques) and the graphic drivers.
ZyApplyResources(${PROJECT_NAME})
ZyApplyModules(${PROJECT_NAME})
```

A local checkout works the same way with `ADD_SUBDIRECTORY(<path to Engine> Engine)` in place of the two
`FetchContent` calls.

Notes:

- `ZyApplyModules` is not optional. `ZY_APPLICATION` calls `ZyRuntime::ZyRegisterModules()`, which only that
  function defines. Without it the link fails with an unresolved `ZyRegisterModules`.
- Without the driver modules there is no graphic driver to pick, and the graphic service fails to start (the kernel
  logs `Kernel: Failed to initialize graphic service` and exits).
- `ZyApplyResources` is needed when you call `ZyRegisterEmbedded`, which you should whenever you use `ZyRender`: its
  post effects load their techniques from `Embedded://Technique/...`.
- `ZyEngine` publishes a precompiled header (`Zyphryon.Base/Base.hpp`), so the whole of Base is visible in every
  file of a target that links it.
- When the engine is the top-level project, it writes binaries to `<build>/bin`. As a dependency it leaves your
  output folders alone.

### 3.5 Presets for the engine itself

Building the engine repository on its own uses `CMakePresets.json`:

```
cmake --preset release
cmake --build --preset release
```

| Preset | Builds |
|---|---|
| `debug` | Unoptimized, with AddressSanitizer under GCC and Clang. |
| `relwithdebinfo` | Optimized without whole-program optimization, with symbols. |
| `release` | Fully optimized and stripped. |
| `headless` | `release` with `ZY_MODE_HEADLESS=ON`. |
| `profile` | `relwithdebinfo` with `ZY_PROFILE_BACKEND_TRACY=ON`. |
| `web` | `release` with the Emscripten toolchain from `$EMSDK`. |

Each preset builds into `build/<preset>`. Copy the same cache variables into your own presets when you consume the
engine (for example `"ZY_MODE_HEADLESS": "ON"` for a server build).

## 4. How a program boots

### 4.1 The entry point

A program is one class derived from `ZyRuntime::Kernel`, named once in a `.cpp`:

```cpp
#include <Zyphryon.Runtime/Entry.hpp>

ZY_APPLICATION(Demo::Application)
```

`ZY_APPLICATION(Type)` writes `int main(int, char *[])`. It calls `ZyRuntime::Launch<Type>`, which static-asserts
that `Type` derives from `Kernel`, constructs it on the stack, and calls
`Kernel::Run(Count, Arguments, ZyRegisterModules())`. Never call `Run` yourself.

### 4.2 What `Run` does

1. Parses the command line into an `Environment` (`GetEnvironment()`).
2. Calls your `OnConfigure(Ref<Startup>)`. No service exists yet. Set the window, graphic and audio parameters here.
3. Registers the services in a fixed order: platform, input, job, network, content, scene, replication, graphic,
   audio, render (the window, input, graphic, audio and render ones only when not headless).
4. Attaches each module (`Module::OnAttach`). The graphic drivers register themselves with the graphic service here.
5. Opens the graphic device on the window, then the audio device. A failed graphic device ends the program; a
   failed audio device is a warning and the program runs silent.
6. Calls your `OnInitialize()`. Return `true` to run. The window is shown only now, and only when you returned
   `true`.
7. Loops: each frame calls your `OnTick(Delta)`, then every service's `OnTick` (the scene's phases run there, and the
   graphic service submits the frame), until `Quit()` is called or the window is closed.
8. Calls your `OnTerminate()`, detaches the modules, tears every service down, and flushes the log.

`runtime.md` covers each step in detail, the `Startup` fields, the services, and the headless switch.

### 4.3 The four hooks

| Hook | When | Typical work |
|---|---|---|
| `void OnConfigure(Ref<ZyRuntime::Startup> Startup)` | Before any service. | Window title and size, driver choice, tick rate, headless. |
| `Bool OnInitialize()` | Every service is up. | Mount folders, load assets, declare components and systems, spawn entities. |
| `void OnTick(Real64 Delta)` | Every frame, before the services tick. | Per-frame application work: drawing, polling input. `Delta` is in seconds. |
| `void OnTerminate()` | Once, before teardown. | Remove handlers you added, release GPU objects you own. |

From inside a kernel, find a service with `GetService<T>()`, which returns a `Retainer<T>` (empty when the service
was not registered, for example the graphic service in a headless run).

## 5. A complete first program

The program below opens a 1280 by 720 window, declares one component and one system, spawns one entity, and each
frame clears the window to a colour read from that entity. The system turns the entity's angle on the scene's
clock; `OnTick` reads the angle and picks the colour.

Folder layout:

```
Hello/
    CMakeLists.txt          the file from section 3.4
    Source/
        Hello.cpp
```

`Source/Hello.cpp`:

```cpp
#include <Zyphryon.Content/Service.hpp>
#include <Zyphryon.Graphic/Service.hpp>
#include <Zyphryon.Platform/Service.hpp>
#include <Zyphryon.Runtime/Entry.hpp>
#include <Zyphryon.Scene/Service.hpp>
#include "Hello.Embedded.hpp"

namespace Demo
{
    // A component: any default-constructible, movable type.
    struct Spin final
    {
        /// The turns per second.
        Real32 Speed  = 0.25f;

        /// The angle turned so far.
        Angle  Turned = Angle();
    };

    class Application final : public ZyRuntime::Kernel
    {
    protected:

        void OnConfigure(Ref<ZyRuntime::Startup> Startup) override
        {
            ZyRuntime::Startup::Window Window = Startup.GetWindow();
            Window.Title  = "Hello Zyphryon";
            Window.Width  = 1280;
            Window.Height = 720;
            Startup.SetWindow(Window);
        }

        Bool OnInitialize() override
        {
            // The engine's own techniques, served as Embedded://.
            ZyRegisterEmbedded(* GetService<ZyContent::Service>());

            Ref<ZyScene::Service> Scene = * GetService<ZyScene::Service>();

            // Components are declared before anything uses them.
            Scene.Declare<Spin>("Demo::Spin");

            // The engine declares no phases. Make one, then put a system in it.
            const UInt32 Update = Scene.Phase("Update");

            // The callback's parameters are the query: the clock singleton, and every entity that owns a Spin.
            Scene.System("Demo::Turn", Update).Each([](ConstRef<ZyScene::Clock> Clock, Ref<Spin> Spin)
            {
                const Angle Step = Angle::FromTurns(Spin.Speed) * static_cast<Real32>(Clock.GetDelta());
                Spin.Turned = Angle::Normalize(Spin.Turned + Step);
            });

            mWheel = Scene.Spawn(Spin());

            LOG_I("Demo: ready, the wheel is entity {0}", mWheel.GetID());
            return true;
        }

        void OnTick(Real64 Delta) override
        {
            ConstRef<ZyPlatform::Window> Window = GetService<ZyPlatform::Service>()->GetWindow();

            const Real32 Width  = static_cast<Real32>(Window.GetWidth());
            const Real32 Height = static_cast<Real32>(Window.GetHeight());

            // Read the value the system wrote on the last frame.
            Real32 Shade = 0.5f;

            // The variable is named after its type, so the type is qualified where the name already means the variable.
            if (const ConstPtr<Spin> Spin = mWheel.Get<const Demo::Spin>())
            {
                Shade = 0.5f + 0.5f * Angle::Sine(Spin->Turned);
            }

            // Clear the window's own pass, then hand it to the graphic service, which submits it this frame.
            ConstRetainer<ZyGraphic::Service> Graphic = GetService<ZyGraphic::Service>();

            Graphic->Prepare(ZyGraphic::kDisplay, "Clear", ZyGraphic::Viewport(0.0f, 0.0f, Width, Height),
                Color(0.1f, 0.2f * Shade, 0.6f * Shade, 1.0f), 1.0f, 0);
            Graphic->Commit();
        }

        void OnTerminate() override
        {
            LOG_I("Demo: closing");
        }

    private:

        ZyScene::Entity mWheel;
    };
}

ZY_APPLICATION(Demo::Application)
```

Build and run it:

```
cmake -S Hello -B Hello/build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build Hello/build
```

What to notice:

- `"Hello.Embedded.hpp"` is the header `ZyApplyResources(Hello)` generates. Its name follows the target name.
- `* GetService<T>()` turns the returned `Retainer<T>` into a reference. The kernel holds the service for the
  program's life, so the reference stays valid.
- `ZyGraphic::kDisplay` is the pass that draws into the window. `Prepare` sets the viewport and clears; `Commit`
  closes the pass. Real drawing goes between the two (see `graphics.md`).
- `OnTick` runs before the services tick, so it sees what the scene's systems wrote on the previous frame.
- The scene clock (`ZyScene::Clock`) is a singleton the scene service declares. Its `GetDelta()` is the frame's
  delta times the timescale (`Scene.SetTimescale(0.0f)` pauses every system that reads it).
- A system holds no handle. It runs every frame, in the phase it was made in, until the world ends. See `scene.md`.
- Logs print to standard output with a coloured `[INFO]` prefix.

Closing the window ends the program. To end it from code, call `Quit()` from any hook.

## 6. The same program, headless

A dedicated server, a tool or a test has no window. There are two ways to run without one:

| Way | How | Effect |
|---|---|---|
| Build time | `-DZY_MODE_HEADLESS=ON` (the `headless` preset) | The window, input, graphic, audio and render modules are not compiled; their headers still exist but nothing links. |
| Run time | `Startup.SetHeadless(true)` in `OnConfigure` | The same services are not registered, in a normal build. |

A headless run also has no window to wait on for the display's refresh, so the loop spins as fast as it can. Cap it
with `Startup.SetRate(N)`.

```cpp
#include <Zyphryon.Runtime/Entry.hpp>
#include <Zyphryon.Scene/Service.hpp>

namespace Demo
{
    struct Counter final
    {
        /// The ticks counted so far.
        UInt32 Ticks = 0;
    };

    class Server final : public ZyRuntime::Kernel
    {
    protected:

        void OnConfigure(Ref<ZyRuntime::Startup> Startup) override
        {
            Startup.SetHeadless(true);
            Startup.SetRate(30);                                      // 30 ticks a second
        }

        Bool OnInitialize() override
        {
            Ref<ZyScene::Service> Scene = * GetService<ZyScene::Service>();
            Scene.Declare<Counter>("Demo::Counter");

            Scene.System("Demo::Count", Scene.Phase("Update")).Each([](Ref<Counter> Counter)
            {
                ++Counter.Ticks;
            });

            mCounter = Scene.Spawn(Counter());
            return true;
        }

        void OnTick(Real64 Delta) override
        {
            const ConstPtr<Counter> Counter = mCounter.Get<const Demo::Counter>();

            if (Counter && Counter->Ticks == 90)
            {
                LOG_I("Demo: three seconds at {0} ticks a second", GetMeter().GetRate());
                Quit();
            }
        }

    private:

        ZyScene::Entity mCounter;
    };
}

ZY_APPLICATION(Demo::Server)
```

In a headless run `GetService<ZyGraphic::Service>()` and the other skipped services return an empty retainer.
Code shared between a windowed and a headless program checks for that before using them:

```cpp
if (ConstRetainer<ZyGraphic::Service> Graphic = GetService<ZyGraphic::Service>())
{
    // Draw only where there is something to draw on.
}
```

In a `ZY_MODE_HEADLESS` build those service types are not compiled at all, so guard such code with
`#if !defined(ZY_MODE_HEADLESS)` as the kernel itself does.

## 7. Reading the command line and saving settings

`GetEnvironment()` returns the parsed command line. Switches start with `--`:

```cpp
void OnConfigure(Ref<ZyRuntime::Startup> Startup) override
{
    ConstRef<Environment> Command = GetEnvironment();

    ZyRuntime::Startup::Window Window = Startup.GetWindow();
    Window.Width      = Command.GetNumber<UInt32>("width", 1280);
    Window.Height     = Command.GetNumber<UInt32>("height", 720);
    Window.Fullscreen = Command.GetBool("fullscreen", false);
    Startup.SetWindow(Window);
}
```

```
Hello --width=1920 --height 1080 --fullscreen
```

`Kernel::Save(JsonObject)` writes the window, graphic and audio parameters the user may change, as they stand
(the window's current size, monitor and fullscreen flag). `Startup::Load(JsonObject)` reads them back. Section 1.4
of `runtime.md` shows the round trip through a settings file.

## 8. Common first mistakes

| Mistake | What happens | Do instead |
|---|---|---|
| No `ZyApplyModules(<target>)` | Link error: `ZyRegisterModules` is unresolved. | Call it on the executable. |
| Driver options both off, or a headless build expecting a window | `Kernel: Failed to initialize graphic service`, the program exits. | Keep a driver on; use `SetHeadless(true)` only where no window is wanted. |
| Including `"<target>.Embedded.hpp"` without `ZyApplyResources(<target>)` | The header does not exist. | Call `ZyApplyResources` on the same target. |
| Using a service in `OnConfigure` | `GetService<T>()` is empty; nothing is registered yet. | Do that work in `OnInitialize`. |
| Using a component before `Declare<T>` | The scene does not know the type. | Declare every component first, in `OnInitialize`. |
| `try` / `throw` / `dynamic_cast` in your code | Compile error under `-fno-exceptions -fno-rtti`. | Return `Bool` or a result enum; use `Retainer<T>::Cast` for known downcasts. |
| Expecting `OnTick` to see this frame's system writes | It runs before the scene ticks. | Read last frame's values, or do the work in a system. |
| Calling `Run` or writing your own `main` | Two entry points, or a kernel without modules. | `ZY_APPLICATION(Type)` once. |
| Logging with a `Text` variable as the pattern | Does not compile: the macros paste `_Text` onto a literal. | `LOG_I("{0}", Value)`. |

## 9. Where it lives

| Path | Holds |
|---|---|
| `CMakeLists.txt` | Root project: options, includes of the helpers, the engine and the drivers. |
| `CMakePresets.json` | The six presets. |
| `CMake/CMakeAddTarget.cmake` | `ZyAddTarget`. |
| `CMake/CMakeApplyOptions.cmake` | `ZyApplyOptions`: flags per compiler and platform. |
| `CMake/CMakeApplyModules.cmake`, `CMake/CMakeRegisterModule.cmake` | `ZyApplyModules`, `ZyRegisterModule`. |
| `CMake/CMakeApplyResources.cmake`, `CMake/CMakeRegisterResources.cmake` | `ZyApplyResources`, `ZyRegisterResources`. |
| `CMake/CMakeApplyLibrary.cmake` | `ZyApplyExports`, `ZyApplyPlugin`. |
| `Engine/CMakeLists.txt` | The `ZyEngine` target, headless exclusions, platform libraries, Opus, Tracy, embedded resources. |
| `Engine/Source/Zyphryon.Runtime/Entry.hpp` | `ZY_APPLICATION`, `Launch`, `ZyRegisterModules`. |
| `Engine/Source/Zyphryon.Runtime/Kernel.hpp`, `Kernel.cpp` | The kernel: hooks, service order, frame loop. |
| `Engine/Source/Zyphryon.Runtime/Startup.hpp` | Startup parameters. |
| `Module/Graphic/Driver/D3D11/`, `Module/Graphic/Driver/OGL/` | The graphic drivers and their module registration. |
| `Engine/Resources/` | The engine's embedded shaders and techniques. |
| `README.md` | Overview, prerequisites, status. |

The repository ships no sample program and no automated test suite yet. The two programs in this chapter are the
smallest complete applications.
