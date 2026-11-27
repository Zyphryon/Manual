# Cookbook

Worked recipes, start to finish. Each one lists the source files, the commands, what lands on disk and what to
check. Paths are examples: a project with source art under `Art/` and baked files under `Content/`.

---

## 1. A character with clips

**Goal:** a skinned knight with idle, walk, attack and death clips, its colour and normal maps, ready for
`ZyRender::Model3D`.

### 1.1 Sources

```
Art/Knight/Knight.fbx          one skinned mesh, four takes: Armature|Idle, Armature|Walk, Armature|Attack, Armature|Death
Art/Knight/Knight_Albedo.png   colour, 2048x2048
Art/Knight/Knight_Normal.png   tangent-space normal, 2048x2048
```

Before baking, check in your modelling tool:

- The mesh is bound to a skin. Without a skin deformer, no skeleton or clips are baked.
- Bone names are unique.
- Y is up. Note the unit the file declares (centimetres is common).
- The material's base colour and normal map textures point at `Knight_Albedo.png` and `Knight_Normal.png`. Only the
  file names matter.

### 1.2 Bake the model

```sh
Studio bake Art/Knight/Knight.fbx Content/Knight/Knight.mdl \
    --units 1 --rate 30 --interpolation Cubic --once Armature_Attack,Armature_Death
```

- `--units 1`: one baked unit is one metre, whatever the FBX declares.
- `--interpolation Cubic`: smooth curves at 30 Hz at no extra size.
- `--once`: the takes' **cleaned** names. `Armature|Attack` becomes `Armature_Attack`.

The log shows, among other lines:

```
Model: 9214 vertices from 27642 corners, 9214 triangles, 1 materials, 7 attributes in 1 streams
Model:     level 0 keeps 9214 triangles (100%) above 3% coverage, and casts 9214
Model:     level 1 keeps 2303 triangles (24%) above 1% coverage, and casts 1890
...
Model: 54 bones, 4 clips at 30 Hz, blended Cubic
```

(Numbers illustrative.)

### 1.3 What lands on disk

```
Content/Knight/Knight.mdl
Content/Knight/Knight.msh
Content/Knight/Knight.skl
Content/Knight/Knight.Armature_Idle.anm
Content/Knight/Knight.Armature_Walk.anm
Content/Knight/Knight.Armature_Attack.anm
Content/Knight/Knight.Armature_Death.anm
```

### 1.4 Bake the textures beside the manifest

The manifest names `Knight_Albedo.tex` and `Knight_Normal.tex` in its own folder:

```sh
Studio bake Art/Knight/Knight_Albedo.png Content/Knight/Knight_Albedo.tex --preset sprite --format BC7UIntNorm_sRGB
Studio bake Art/Knight/Knight_Normal.png Content/Knight/Knight_Normal.tex --preset normal
```

- The `sprite` preset is `--layered --no-linear --mipmaps --transparent`: sRGB colour, a full chain whose colour is
  weighed by alpha, a one-slice array. Drop `--layered` (`--no-layered`) if your model shader samples plain 2D
  textures.
- The `normal` preset keeps the normal map linear and renormalises every mip level (`--normal`).

### 1.5 Tidy the manifest

Open `Knight.mdl`. Rename the clips to the names your game asks for. Keep `Path` as baked:

```json
"Clips": [
    { "Name": "Idle",   "Path": "Knight.Armature_Idle.anm",   "Repeat": "Loop" },
    { "Name": "Walk",   "Path": "Knight.Armature_Walk.anm",   "Repeat": "Loop" },
    { "Name": "Attack", "Path": "Knight.Armature_Attack.anm", "Repeat": "Once", "Rate": 1.2 },
    { "Name": "Death",  "Path": "Knight.Armature_Death.anm",  "Repeat": "Once" }
]
```

A re-bake writes the manifest again. Keep this edit in a script, or rename the takes in the FBX instead.

### 1.6 Check

- Exit code `0`.
- The model is a sensible size in the engine (one unit, one metre).
- Every clip plays. A clip that moves nothing usually means bone names differ from the rig's.
- If your shader reads roughness and metalness from one packed texture, see recipe 5.

---

## 2. A texture array of ground materials

**Goal:** one `Texture2DArray` holding six ground colours at 1024x1024, mipmapped and BC7, with room for two more,
plus a tracker naming each slice.

### 2.1 Sources

```
Art/Ground/Colour/dirt.png    1024x1024
Art/Ground/Colour/grass.png   1024x1024
Art/Ground/Colour/gravel.png  1024x1024
Art/Ground/Colour/mud.png     1024x1024
Art/Ground/Colour/rock.png    2048x2048
Art/Ground/Colour/sand.png    1024x1024
```

### 2.2 Pack

```sh
Studio pack Art/Ground/Colour --output Content/Ground/Colour.json \
    --mode Array --extent 1024x1024 --fit Contain --headroom 4 \
    --no-linear --mipmaps --format BC7UIntNorm_sRGB
```

- The folder is listed and sorted by name: `dirt`, `grass`, `gravel`, `mud`, `rock`, `sand` take slices 0 to 5.
- `--fit Contain` shrinks the 2048 `rock` into its 1024 slice. `Stretch` (the default) would do the same for a
  square image. `Center` would refuse it.
- `--headroom 4` rounds six slices up to eight. Slices 6 and 7 are blank.
- 1024 is a multiple of four, so BC7 is allowed.

Output:

```
Content/Ground/Colour.json
Content/Ground/Colour.tex
```

A game looks up `"grass"` in `Colour.json` and reads `"Slice": 1`.

### 2.3 Add a seventh material later

Append a region to `Colour.json`, on the first blank slice:

```json
{ "Name": "snow", "Source": "../../Art/Ground/Colour/snow.png", "Slice": 6 }
```

`Rect` is filled in from the image's size when the region is measured, but for an array the region must fit its
slice. Give `Rect` explicitly when the image is not 1024x1024:

```json
{ "Name": "snow", "Source": "../../Art/Ground/Colour/snow.png", "Slice": 6, "Rect": [0, 0, 1024, 1024] }
```

Then draw again:

```sh
Studio build Content/Ground/Colour.json
```

The format (`BC7UIntNorm_sRGB`), headroom, `--no-linear` and `--mipmaps` all come from the tracker, which `pack`
wrote them into. Typing them again on `build` does no harm.

### 2.4 A matching normal array

```sh
Studio pack Art/Ground/Normal --output Content/Ground/Normal.json \
    --mode Array --extent 1024x1024 --fit Contain --headroom 4 --linear --mipmaps
```

Name the normal maps after the colour maps (`dirt.png`, `grass.png`, ...) in their own folder, so both trackers give
each material the same slice.

---

## 3. A UI font with icons

**Goal:** a font covering Western European text and punctuation, with arrows from a symbol font and icons from a
private-use icon font.

```sh
Studio bake Art/Fonts/Inter-Regular.ttf Content/UI/Inter.fnt \
    --size 48 --range 8 --charset latin1,punctuation,0x20AC \
    --fallback "Art/Fonts/NotoSansSymbols.ttf=0x2190-0x21FF;Art/Fonts/Icons.ttf=0xE000-0xE0FF"
```

- `--size 48 --range 8`: sharp glyphs, a band wide enough for a thin outline. Raise `--range` for thick outlines
  or glows.
- `0x20AC` adds the euro sign.
- Arrows come from the symbol font, icons from the icon font. Each fallback is asked only for its own range.
- Metrics (line height, underline) come from Inter.

The log reports the glyph, pair and page counts:

```
Font: 253 glyph(s), 1720 kerning pair(s), 1 page(s) of 1012x1012
```

(Numbers illustrative.) If the bake opens several pages, raise `--limit` or lower `--size`.

Check by drawing a test string with every range: Latin-1 letters, a dash and quotes, the euro, an arrow, an icon. A
`?` marks a codepoint no face had.

---

## 4. A batch of sounds

**Goal:** every effect as ADPCM, every music track as Opus, every voice line as Opus at a lower rate, in one script.

### 4.1 A presets file

`Studio.json`, kept with the project:

```json
{
    "effect": { "encoding": "Adaptive" },
    "music":  { "encoding": "Opus", "bitrate": 128000 },
    "voice":  { "encoding": "Opus", "bitrate": 48000 }
}
```

### 4.2 The script (POSIX shell)

```sh
#!/bin/sh
set -u
failed=0

bake() {   # bake <folder> <extension> <preset>
    for f in Art/Audio/"$1"/*."$2"; do
        [ -e "$f" ] || continue
        out="Content/Audio/$1/$(basename "$f" ."$2").snd"
        Studio bake "$f" "$out" --presets Studio.json --preset "$3" || failed=1
    done
}

bake Effects wav  effect
bake Music   flac music
bake Voice   wav  voice

exit $failed
```

- Each file is baked on its own, so one bad file does not stop the rest.
- When one preset fits a whole folder, one run does the same:
  `Studio bake Art/Audio/Effects --output Content/Audio/Effects --presets Studio.json --preset effect`. A folder
  gives every file in it that a baker reads, its images and fonts too, and the run bakes them in parallel. One bad
  file does not stop the rest there either, and the run exits `1` if any failed.
- Studio exits `1` for a broken asset and `2` for a broken command line. A `2` in a script like this usually means a
  preset name or switch is wrong for every file.
- Sources at 44.1 kHz are resampled to 48 kHz. Sources with more than two channels fail. Downmix them first.

### 4.3 The same in PowerShell

```powershell
Get-ChildItem Art/Audio/Effects -Filter *.wav | ForEach-Object {
    Studio bake $_.FullName "Content/Audio/Effects/$($_.BaseName).snd" --presets Studio.json --preset effect
    if ($LASTEXITCODE -ne 0) { Write-Warning "failed: $($_.Name)" }
}
```

---

## 5. A packed surface map for a model

**Goal:** one texture holding roughness in R, metalness in G and occlusion in B, for a technique that reads them
packed, and a manifest that binds it.

### 5.1 Pack the first map as a one-slice array

```sh
Studio pack Art/Knight/Knight_Roughness.png --output Content/Knight/Knight_Surface.json --mode Array
```

This writes `Knight_Surface.json` and `Knight_Surface.tex`, with one region named `Knight_Roughness`.

### 5.2 Splice the other channels in

Edit the region in `Knight_Surface.json`:

```json
{
    "Name": "Knight_Roughness",
    "Source": "../../Art/Knight/Knight_Roughness.png",
    "Slice": 0,
    "Rect": [0, 0, 2048, 2048],
    "Channels": {
        "G": { "Source": "../../Art/Knight/Knight_Metallic.png",  "Channel": "R" },
        "B": { "Source": "../../Art/Knight/Knight_Occlusion.png", "Channel": "R" }
    }
}
```

The base image fills every channel, and the spliced ones replace G and B. Its own R stays as roughness. To set
alpha too, splice `"A"`.

### 5.3 Build

```sh
Studio build Content/Knight/Knight_Surface.json --linear --mipmaps --format RGBA8UIntNorm
```

Keep it linear: these are data, not colour. Name the format: a greyscale first source decodes as one channel
(`R8UIntNorm`), which the `pack` recorded in the tracker, and a one-channel texture would drop the spliced G and B.
A `--format` typed on `build` replaces the tracker's. `BC7UIntNorm` or `BC1UIntNorm` work too at 2048x2048.

### 5.4 Bind it in the manifest

In `Knight.mdl`, replace the separate `Roughness`, `Metallic` and `Occlusion` entries (in `Images`, `Samplers` and
`Parameters`) with one under the name your technique declares, for example `Surface`:

```json
"Images": {
    "Albedo":  { "Path": "Knight_Albedo.tex" },
    "Normal":  { "Path": "Knight_Normal.tex" },
    "Surface": { "Path": "Knight_Surface.tex" }
},
"Samplers": {
    "Surface": { "AddressModeU": "Repeat", "AddressModeV": "Repeat", "AddressModeW": "Repeat", "Filter": "LinearMipLinear" }
}
```

Or move the whole material into `Content/Knight/Knight.mtl` and make the manifest's entry the string
`"Knight.mtl"`, so a re-bake of the model does not undo the edit (`materials.md` section 3.2). The path is resolved
by the engine's content service.

---

## 6. A UI atlas that survives rebuilds

**Goal:** every button and icon of a game's UI in one sRGB atlas, rebuilt after each art change without colour or
mip drift.

### 6.1 A preset for the atlas

In `Studio.json`:

```json
{
    "ui": { "linear": false, "mipmaps": false, "layered": false }
}
```

### 6.2 Pack once

```sh
Studio pack Art/UI/Buttons Art/UI/Icons --output Content/UI/Atlas.json \
    --padding 2 --extrude 1 --presets Studio.json --preset ui
```

Region names are file stems, so a button and an icon with the same stem collide. Keep stems unique across both
folders.

### 6.3 Rebuild after edits

```sh
# Art changed, places kept
Studio build Content/UI/Atlas.json --presets Studio.json --preset ui

# A new image was added: place everything again
Studio build Content/UI/Atlas.json --repack --presets Studio.json --preset ui
```

The tracker already keeps the preset's `linear` and `mipmaps`, so the preset on `build` only guards against a
tracker written before Studio stored them. `--repack` keeps the tracker's mode, padding and extrusion. Images added
since the first pack must be appended to the tracker as regions first. `build` only draws what the tracker lists.

If the art has soft or cut-out edges, add `"transparent": true` to the preset, so the clear space around each image
takes its edge colour and filtering shows no dark fringe.

---

## 7. A sky cube

```sh
Studio bake Art/Sky/Sky_Cross.hdr Content/Sky/Sky.tex --cube 4x3 --format RGBA16Float --mipmaps
```

- The source is a horizontal cross: four faces across, three down, each face square.
- HDR sources decode as 32-bit float and are stored as `RGBA16Float` when no `--format` is given, half the size of
  `RGBA32Float`. The `--format` here only makes that explicit.
- Mips give the shader a cheap blurred lookup for rough reflections.

---

## 8. A tiling stone material from one height map

```sh
# Normal map, tiling, a bit of broad shape
Studio normal Art/Stone/Stone_Height.png Content/Stone/Stone_Normal.tex \
    --preset normal --wrap --strength 3 --octaves 3 --falloff 0.5

# Height for parallax or displacement, one channel
Studio relief Art/Stone/Stone_Height.png Content/Stone/Stone_Height.tex \
    --preset relief --wrap --contrast 1.3

# Colour
Studio bake Art/Stone/Stone_Albedo.png Content/Stone/Stone_Albedo.tex --preset sprite
```

Then write `Content/Stone/Stone.mtl` binding the three under your technique's names (`materials.md`).

---

## 9. Where it lives

| Recipe | Chapters |
|---|---|
| 1, 5 | `models.md`, `textures.md`, `trackers.md` section 6, `materials.md` |
| 2, 6 | `trackers.md` |
| 3 | `fonts.md` |
| 4 | `sounds.md`, `getting-started.md` section 5 |
| 7, 8 | `textures.md` |
