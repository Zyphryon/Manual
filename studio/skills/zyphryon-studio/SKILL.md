---
name: zyphryon-studio
description: Use when baking assets for the Zyphryon engine with Zyphryon Studio - running `Studio bake`, `pack`, `build`, `normal`, `relief`, `extract`, `formats` or `version`, choosing texture switches (`--format`, `--mipmaps`, `--linear`, `--layered`, `--cube`, `--array`), packing atlases or texture arrays into a JSON tracker, baking FBX models into `.mdl`/`.msh`/`.skl`/`.anm`, TTF/OTF fonts into MTSDF `.fnt`, WAV/FLAC/MP3 into `.snd`, writing `.mtl` materials, presets files, or linking the `Studio_Texture`, `Studio_Font`, `Studio_Model`, `Studio_Sound`, `Studio_Material` libraries into an editor or tool; also when debugging a baked `.tex`, `.fnt`, `.snd`, mesh or clip the engine refuses or draws wrong.
---

# Zyphryon Studio

Studio is the offline asset baker of the Zyphryon engine. It is one command line program, `Studio`, and a set of
static libraries with the same bakers inside. It reads source art (images, FBX scenes, typefaces, sound files) and
writes the native files the engine's loaders read:

| Source | Command | Writes | Engine loader |
|---|---|---|---|
| `.png .jpg .jpeg .tga .bmp .hdr` (or a baked `.tex`) | `bake`, `pack`, `build`, `normal`, `relief` | `.tex` (and a `.json` tracker for `pack`) | `ZyGraphic::TEXLoader` |
| `.fbx` | `bake` | `.mdl` manifest, `.msh`, `.skl`, one `.anm` per clip | `ZyRender::MDLLoader` and friends |
| `.ttf .otf .ttc` | `bake` | `.fnt` (MTSDF atlas inside) | `ZyRender::FNTLoader` |
| `.flac .mp3 .wav` | `bake` | `.snd` | `ZyAudio::SNDLoader` |
| (library only) | `Studio::Material::Document` | `.mtl` | `ZyGraphic::MTLLoader` |

The same source and the same switches always bake to the same bytes.

## How the pieces fit

```
source art ──► Importer ──► in-memory data ──► Process (mips, encode, weld, LOD, ...) ──► Exporter ──► native file
```

- Each domain lives in `Library/<Domain>` and depends on the engine only, never on another domain.
- A domain's `Baker` is the entry point. A `Profile` struct holds its settings, and `Profile::From(Environment)`
  reads them from a parsed command line.
- `Application/` is the command line: it parses switches, expands presets, picks the baker by the source's
  extension and maps results to exit codes.

## Chapters

| Chapter | Read it for |
|---|---|
| `references/getting-started.md` | Building Studio, the command line grammar, every command, presets, exit codes, linking the libraries |
| `references/textures.md` | `bake` for images, formats, colour space, mipmaps, cube maps, arrays, `normal`, `relief`, `extract` |
| `references/trackers.md` | `pack` and `build`: atlases, texture arrays, the JSON tracker, channel splicing |
| `references/models.md` | FBX into `.mdl`/`.msh`/`.skl`/`.anm`: units, attributes, levels of detail, rigs, clips, materials |
| `references/fonts.md` | TTF/OTF into `.fnt`: size, range, charsets, fallback faces, the atlas |
| `references/sounds.md` | WAV/FLAC/MP3 into `.snd`: resampling, Linear, Adaptive (IMA ADPCM), Opus |
| `references/materials.md` | The `.mtl` JSON file, the materials a model manifest carries inline, `Studio::Material::Document` |
| `references/formats.md` | Byte layout of every native file Studio writes, and the JSON files |
| `references/cookbook.md` | Worked recipes: a character with clips, a texture array, a font, a batch of sounds |

## Top traps

1. **Operands first, switches after.** A switch takes the next word as its value unless that word starts with
   `--`. `Studio bake --mipmaps hero.png` reads `hero.png` as the value of `--mipmaps` and finds no source.
2. **Textures default to linear, unmipmapped.** `--linear` is on and `--mipmaps` is off. Colour art needs
   `--no-linear` (or `--preset sprite`), anything drawn smaller than its size needs `--mipmaps`.
3. **A `_sRGB` format does not make the source sRGB.** With `--linear` on, `--format BC7UIntNorm_sRGB` re-encodes
   the source's values through the sRGB curve, which brightens them. Pair a `_sRGB` format with `--no-linear`.
4. **Block formats need sides that are multiples of four** (BC1, BC2, BC3, BC4, BC5, BC7). Other BC, ETC2 and EAC
   formats are not written at all. `Studio formats` lists what can be written.
5. **`build` remembers colour space and mips, but not `--layered`.** The tracker stores the format, padding,
   extrude, fill, headroom, mipmaps, linear and compress settings, and `build` uses them unless a switch is typed. An
   older tracker without them reads as mips off and linear on, so pass the switches to its first `build` once.
6. **`build --repack` forgets the array mode.** Repacking reads `--mode`, `--width`, `--height`, `--fit` and the
   rest from the command line with their defaults, so an array tracker repacked without `--mode Array` becomes an
   atlas.
7. **A model manifest names textures by file name only.** Every image the FBX references is rewritten to
   `<image stem>.tex` beside the `.mdl`. Bake each texture to that name in the same folder.
8. **Studio writes each material image under its own slot name**: `Albedo`, `Normal`, `Roughness`, `Metallic`,
   `Specular`, `Emissive`, `Displacement`, `Occlusion`, `Opacity`. A technique that reads packed maps instead (one
   texture holding roughness, metalness and occlusion) will not see the separate `Roughness`, `Metallic` and
   `Occlusion` images. Pack them with `pack` and channel splicing, then edit the manifest.
9. **Clips come only with a skin.** An FBX whose meshes have no skin deformer bakes no skeleton and no clips, even
   when its nodes are animated. `--no-rig` bakes a rigged source as a static prop.
10. **Sounds are mono or stereo, at 48 kHz.** More than two channels is refused. Everything is resampled to the
    mixer's 48000 Hz, and the engine refuses any `.snd` at another rate.

## Where it lives

| Path | Holds |
|---|---|
| `Application/Source/Console.cpp` | Every command's handler |
| `Application/Source/Usage.cpp` | The `help` text, the canonical switch list with defaults |
| `Application/Source/Preset.hpp` | The built-in presets |
| `Library/Texture/Source/Texture/` | `Baker`, `Profile`, `Import/`, `Export/`, `Process/`, `Atlas/`, `Generate/`, `Image/` |
| `Library/Model/Source/Model/` | `Baker`, `Profile`, `Scene`, `Import/FBX`, `Process/`, `Export/Exporter` |
| `Library/Font/Source/Font/` | `Baker`, `Profile`, `Typeface`, `Process/Generator`, `Process/Atlas`, `Export/Exporter` |
| `Library/Sound/Source/Sound/` | `Baker`, `Profile`, `Sample`, `Import/Importer`, `Export/Exporter` |
| `Library/Material/Source/Material/Document.hpp` | The `.mtl` reader and writer |
