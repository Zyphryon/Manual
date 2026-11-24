# Getting started

This chapter covers building Studio, the shape of its command line, every command, presets, exit codes, and
linking the baker libraries into your own editor or tool.

---

## 1. What Studio is for

A game built on the Zyphryon engine never loads a PNG, an FBX or a WAV. It loads native files that are cheap to
read: textures already in their GPU format with their mip chain, meshes already welded and split into levels of
detail, fonts as signed distance fields, sounds already at the mixer's rate. Studio turns source art into those
files, ahead of time.

You use it in two ways:

- **The command line**, `Studio <command> <operands> [switches]`, from a shell, a script or a build step.
- **The libraries**, `Studio_Texture`, `Studio_Font`, `Studio_Model`, `Studio_Sound` and `Studio_Material`, linked
  into an editor that bakes in place. The command line is a thin layer over them.

Studio has no GUI. Run with no command, it prints its usage and exits with code `2`.

---

## 2. Building

### 2.1 Prerequisites

- CMake 3.26 or newer.
- A C++23 compiler (the engine requires it).
- Ninja is recommended.

Every dependency is fetched by CMake (`FetchContent`). Nothing needs installing by hand.

| Dependency | Fetched by | Used for |
|---|---|---|
| Zyphryon Engine (`main`) | root `CMakeLists.txt` | Containers, strings, JSON, LZ4, file layouts, the job pool |
| stb | `Library/Texture`, `Library/Font` | Image decoding (PNG, JPEG, TGA, BMP, HDR), PNG writing, TrueType parsing |
| bc7enc_rdo (pinned commit) | `Library/Texture` | BC1 to BC5 (`rgbcx`) and BC7 (`bc7enc`) block encoding |
| msdfgen `v1.12` | `Library/Font` | Multi-channel signed distance fields |
| ufbx `v0.23.1` | `Library/Model` | FBX reading |
| meshoptimizer `v1.3` | `Library/Model` | Vertex welding, cache ordering, simplification |
| dr_libs | `Library/Sound` | FLAC, MP3 and WAV decoding |
| libopus `v1.5.2` | `Library/Sound` | Opus encoding |

### 2.2 Configure and build

```sh
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build --target Studio_Application
```

The target is `Studio_Application`, its output name is `Studio`, and it lands in `build/bin/` (the root
`CMakeLists.txt` sends archives, libraries and executables to `${CMAKE_BINARY_DIR}/bin`).

To build against a local engine checkout instead of fetching it:

```sh
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release \
      -DFETCHCONTENT_SOURCE_DIR_EXTERNAL_ZYENGINE=/path/to/Engine
```

### 2.3 The `STUDIO_BUILD_APPLICATION` option

```cmake
OPTION(STUDIO_BUILD_APPLICATION "Build the Studio executable, not only its libraries" ${PROJECT_IS_TOP_LEVEL})
```

It is on when Studio is the top-level project and off when another project pulls Studio in, so a host gets the
libraries without the executable. Set it explicitly to override.

---

## 3. The command line

```
Studio <command> <operands> [switches]
```

### 3.1 Switch grammar

Switches are parsed by the engine's `Environment` class (`Zyphryon.Base/System/Environment.hpp`). A switch can be
written four ways:

| Spelling | Value |
|---|---|
| `--name` | empty, which a toggle reads as `true` |
| `--name=value` | `value` |
| `--name value` | `value`, when the next word does not start with `--` |
| `--no-name` | `false` |

Rules that follow from the parser:

- **Operands go first.** Any word not starting with `--` is an operand, unless the switch before it swallowed it as
  its value. `Studio bake --wrap rock.png` gives `--wrap` the value `rock.png` and leaves no source.
- **A toggle followed by an operand eats it.** Write toggles last, or spell them `--mipmaps=true`.
- **The last occurrence of a switch wins.** `--format R8UIntNorm --format RGBA8UIntNorm` reads `RGBA8UIntNorm`.
- **Names are case-insensitive.** `--Mipmaps` is `--mipmaps`.
- **Enumerated values are case-insensitive** and must spell an enumerator: `--mode array`, `--fit Contain`.

### 3.2 Defaults

Every default below comes from `Application/Source/Usage.cpp` and the `Profile` structs. `Studio help` prints the
same list. A bracketed value is the default.

---

## 4. Commands

| Command | Operands | Does |
|---|---|---|
| `help` | none | Prints the usage, exit `0` |
| `version` | none | Prints the revision of every file Studio writes |
| `bake` | `<source> [destination]` | Bakes a texture, font, model or sound, picked by the source's extension |
| `pack` | `<image\|folder>...` and `--output <tracker.json>` | Packs images into an atlas or an array and writes a tracker |
| `build` | `<tracker.json>` | Draws a tracker's texture again after its regions were edited |
| `normal` | `<source> [destination]` | Draws a normal map from an image read as a height |
| `relief` | `<source> [destination]` | Draws a greyscale relief map from an image read as a height |
| `extract` | `<texture> [destination]` | Writes each slice of a texture as a `.png` |
| `formats` | none | Lists every name `--format` accepts |

`--help` and `--version` work as switches too. With `--version` anywhere on the line, Studio prints the version and
does nothing else. With `--help` anywhere, it prints the usage, does nothing else and exits `0`, `Studio --help` on
its own included.

### 4.1 `bake`

```sh
Studio bake <source> [destination] [switches]
```

The source's extension picks the baker, checked in this order:

| Extensions | Baker | Default destination |
|---|---|---|
| `png jpg jpeg tga bmp hdr tex` | Texture | `<source stem>.tex` beside the source |
| `ttf otf ttc` | Font | `<source stem>.fnt` |
| `fbx` | Model | `<source stem>.mdl` (plus `.msh`, `.skl`, `.anm` beside it) |
| `flac mp3 wav` | Sound | `<source stem>.snd` |

Extensions are compared case-insensitively. Missing destination folders are created. Anything else is refused:
`Studio: nothing bakes '<file>'`, exit `2`.

A `.tex` source re-bakes a baked texture (to change its format, add mips, or turn it into an array). Name a
destination for it: the default destination is the source itself, which is refused.

### 4.2 `pack` and `build`

```sh
Studio pack <image|folder>... --output <tracker.json> [switches]
Studio build <tracker.json> [--repack] [switches]
```

See `trackers.md`.

### 4.3 `normal` and `relief`

```sh
Studio normal <source> [destination] [switches]
Studio relief <source> [destination] [switches]
```

The default destination is `<source stem>_normal.png` or `<source stem>_relief.png`. A destination ending in
`.tex` writes a native texture instead. See `textures.md`.

### 4.4 `extract`

```sh
Studio extract <texture> [destination]
```

Writes each slice as a PNG. The default destination is `<stem>.png`. With more than one slice, each one gets
`_0`, `_1`, ... appended to the destination's stem. See `textures.md`.

### 4.5 `formats` and `version`

```sh
$ Studio formats
BC1UIntNorm
BC1UIntNorm_sRGB
...
$ Studio version
Studio 1.0.0
  texture '.tex' revision 1
  tracker '.json' revision 1
  font    '.fnt' revision 1
  mesh    '.msh' revision 1
  rig     '.skl' revision 1
  clip    '.anm' revision 1
  sound   '.snd' revision 1
```

`formats` prints every `ZyGraphic::TextureFormat` that `Texture::Exporter::IsSupported` accepts.

---

## 5. Presets

`--preset <name>` stands for a set of switches. The built-in presets live in `Application/Source/Preset.hpp`:

```json
{
    "sprite": { "layered": true, "linear": false, "mipmaps": true },
    "normal": { "layered": true, "linear": true,  "mipmaps": true },
    "relief": { "layered": true, "linear": true,  "mipmaps": true, "format": "R8UIntNorm" }
}
```

| Preset | Same as |
|---|---|
| `sprite` | `--layered --no-linear --mipmaps` |
| `normal` | `--layered --linear --mipmaps` |
| `relief` | `--layered --linear --mipmaps --format R8UIntNorm` |

### 5.1 Your own presets

Pass a JSON file with `--presets <file>`. It is searched before the built-in presets, so a preset of yours can
replace a built-in one of the same name. Preset names are matched case-insensitively.

```json
{
    "ui":     { "layered": false, "linear": false, "mipmaps": false, "format": "RGBA8UIntNorm_sRGB" },
    "albedo": { "layered": true,  "linear": false, "mipmaps": true,  "format": "BC7UIntNorm_sRGB" },
    "data":   { "layered": true,  "linear": true,  "mipmaps": true },
    "music":  { "encoding": "Opus", "bitrate": 128000 },
    "hero":   { "rate": 60, "once": "Attack,Death", "lod": 2 }
}
```

```sh
Studio bake Art/Button.png Content/UI/Button.tex --presets Studio.json --preset ui
```

How a preset is applied:

- Each key becomes `--key=value`. A JSON boolean becomes `true` or `false`, a number is printed, a string is copied.
  Any switch name works, not only texture ones.
- The preset's switches are placed before the typed ones, and the last occurrence wins, so **a switch typed on the
  command line always beats the same switch in the preset**.
- An unknown preset name fails with `Studio: '<name>' is not a preset; the built-in ones are sprite, normal and
  relief`, exit `2`. So does a `--presets` file that cannot be read.
- `--presets` alone, without `--preset`, does nothing.

---

## 6. Exit codes

From `Application/Source/Outcome.hpp`:

| Code | Name | Meaning |
|---|---|---|
| `0` | `Success` | Everything asked for was done |
| `1` | `Failure` | The command was understood, and some of its work failed (unreadable source, encode error, write error) |
| `2` | `Misuse` | The command itself was wrong, unknown or missing what it needs (bad switch value, no source, no command) |

A script can tell a broken asset (`1`) from a broken invocation (`2`). Every error is logged first, prefixed with
its domain: `Texture:`, `Font:`, `Model:`, `Sound:`, `Studio:`.

Switch values that are checked and refused with exit `2`:

| Switch | Check |
|---|---|
| `--format` | Must name a `TextureFormat` |
| `--fill` | Must be `RRGGBB` or `RRGGBBAA` hexadecimal, an optional leading `#` allowed |
| `--encoding` | Must be `Linear`, `Adaptive` or `Opus` |
| `--interpolation` | Must be `Step`, `Linear`, `Cubic` or `Hermite` |
| `--attributes` | Must parse and name `Position` |
| `--lod` | Must be `0` to `3`, else `Model: '--lod' goes from 0 to 3, which <n> is not` |
| `--charset`, `--fallback` charsets | Every entry must be a preset, a codepoint or a range |
| `--mode`, `--fit`, `--filter`, `--shape`, `--channel` | Must name one of the switch's choices, else `Studio: '<x>' is not a choice '--<name>' takes` |

---

## 7. Logging

Studio logs through the engine's `LOG_I`, `LOG_W` and `LOG_E`. A successful bake prints one line per file written:

```
Texture: 'Art/hero.png' -> 'Content/hero.tex' (65612 bytes)
Model: 4182 vertices from 12546 corners, 4182 triangles, 2 materials, 7 attributes in 1 streams
Model:     level 0 keeps 4182 triangles (100%) above 3% coverage, and casts 4182
Model: 'Art/Knight.fbx' -> 'Content/Knight.msh' (301224 bytes)
```

(The numbers above are illustrative.)

---

## 8. Using the libraries

A host fetches Studio and links the domains it needs:

```cmake
FetchContent_Declare(External_Studio
        GIT_REPOSITORY https://github.com/Zyphryon/Studio
        GIT_TAG        main)
FetchContent_MakeAvailable(External_Studio)

TARGET_LINK_LIBRARIES(${PROJECT_NAME} PRIVATE "Studio_Font" "Studio_Model" "Studio_Sound" "Studio_Texture" "Studio_Material")
```

| Target | Namespace | Bakes |
|---|---|---|
| `Studio_Texture` | `Studio::Texture` | `.tex`, atlas and array trackers, normal and relief maps, PNG pictures |
| `Studio_Font` | `Studio::Font` | `.fnt` |
| `Studio_Model` | `Studio::Model` | `.mdl`, `.msh`, `.skl`, `.anm` |
| `Studio_Sound` | `Studio::Sound` | `.snd` |
| `Studio_Material` | `Studio::Material` | `.mtl` reading and writing |

Include paths are rooted at each library's `Source/` folder: `#include <Texture/Baker.hpp>`,
`#include <Model/Baker.hpp>`, `#include <Font/Baker.hpp>`, `#include <Sound/Baker.hpp>`,
`#include <Material/Document.hpp>`.

### 8.1 The bakers

| Class | Constructor | Bake to disk | Bake in memory |
|---|---|---|---|
| `Texture::Baker` | `explicit Baker(Ref<ZyJob::Service> Scheduler)` | `Bool Bake(Text Source, Text Destination, ConstRef<Profile> Profile) const` | `Blob Bake(ConstSpan<Byte> Source, Text Type, ConstRef<Profile> Profile) const` |
| `Font::Baker` | `explicit Baker(Ref<ZyJob::Service> Scheduler)` | `Bool Bake(Text Source, Text Destination, ConstRef<Profile> Profile, ConstSpan<Fallback> Fallbacks = { }) const` | `Blob Bake(ConstSpan<Source> Sources, ConstRef<Profile> Profile) const` |
| `Model::Baker` | `explicit Baker(Ref<ZyJob::Service> Scheduler)` | `Bool Bake(Text Source, Text Destination, ConstRef<Profile> Profile) const` | `Bundle Bake(ConstSpan<Byte> Source, Text Type, Text Name, ConstRef<Profile> Profile) const` |
| `Sound::Baker` | default | `Bool Bake(Text Source, Text Destination, ConstRef<Profile> Profile) const` | `Blob Bake(ConstSpan<Byte> Source, Text Type, ConstRef<Profile> Profile) const` |

`Type` is the source extension without its dot. The scheduler must outlive the baker. Bakers are `const` once
constructed, and the work inside a bake (slices, glyphs, LOD reductions, clip ticks, block rows) is spread over
the scheduler's `ZyJob::Lane::Compute` lane.

### 8.2 A minimal host

This mirrors `Application/Source/Main.cpp`:

```cpp
#include <Texture/Baker.hpp>
#include <Sound/Baker.hpp>

int main(int Count, char * Arguments[])
{
    ZyEngine::Subsystem::Host      Host;
    const Retainer<ZyJob::Service> Scheduler = Host.Register<ZyJob::Service>();

    {
        Studio::Texture::Baker Textures(* Scheduler);

        Studio::Texture::Profile Colour;
        Colour.Linear  = false;   // sRGB colour art
        Colour.Mipmaps = true;
        Colour.Layered = true;
        Textures.Bake("Art/Crate.png", "Content/Crate.tex", Colour);

        Studio::Sound::Baker Sounds;
        Studio::Sound::Profile Effect;
        Effect.Encoding = ZyAudio::Encoding::Adaptive;
        Sounds.Bake("Art/Hit.wav", "Content/Hit.snd", Effect);
    }

    Host.Teardown();
    ZyLog::Flush();
    return 0;
}
```

### 8.3 Reading settings the way the command line does

Every `Profile` can be filled from an `Environment`, so a host can reuse the switch names:

```cpp
Environment Parsed;
Parsed.Parse(Count, Arguments);

const Studio::Texture::Profile Texture = Studio::Texture::Profile::From(Parsed);
const Studio::Font::Profile    Font    = Studio::Font::Profile::From(Parsed);
const Studio::Sound::Profile   Sound   = Studio::Sound::Profile::From(Parsed);

Studio::Model::Profile Model;
if (!Studio::Model::Profile::From(Parsed, Model))
{
    // a switch could not be read
}
```

Note the differences: `Model::Profile::From` returns `Bool` and writes through an output parameter, while the others
return the profile. `Font::Profile::From` leaves `Charset` empty when `--charset` cannot be read.

---

## 9. Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| `Studio: 'bake' takes a source and, optionally, a destination` although a source was typed | A switch before the source swallowed it | Put operands first |
| Exit `2` with the usage printed | No command given | Name a command |
| `Texture: '<file>' is both the source and the destination` | Re-baking a `.tex` without a destination | Name a destination |
| A preset seems ignored for one switch | The same switch was typed too, and typed switches win | Drop the typed one |
| `Studio: 'Arry' is not a choice '--mode' takes` | A misspelt enumerated switch | Check the spelling |
| `Model: '--lod' goes from 0 to 3, which 5 is not` | `--lod` is limited to `0`..`3` | Use `0` to `3` |

---

## 10. Where it lives

| Path | Holds |
|---|---|
| `CMakeLists.txt` | Project, output folders, `STUDIO_BUILD_APPLICATION`, the engine fetch |
| `Library/CMakeLists.txt` | Adds the `Font`, `Material`, `Model`, `Sound`, `Texture` libraries |
| `Library/<Domain>/CMakeLists.txt` | Each library's dependencies |
| `Application/Source/Main.cpp` | Preset expansion, the job host, `main` |
| `Application/Source/Console.hpp`, `Console.cpp` | The command table and every handler |
| `Application/Source/Usage.cpp` | The help text |
| `Application/Source/Preset.hpp`, `Preset.cpp` | Built-in presets and `--presets` loading |
| `Application/Source/Outcome.hpp` | Exit codes |
| `Application/Source/Path.hpp` | Stem, extension and derived-destination helpers |
