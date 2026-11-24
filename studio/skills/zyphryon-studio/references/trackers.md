# Atlases, arrays and trackers

This chapter covers `pack` and `build`: putting many images into one texture, either packed together on an atlas or
one per slice of a texture array, and the JSON **tracker** that records where each image went. The tracker is the
manifest a game reads to find an image by name, and the file you edit by hand to move, crop or recombine regions.

---

## 1. The idea

```
images / folders ──pack──► tracker.json  +  texture.tex
                               │
                 edit by hand  │
                               ▼
                 tracker.json ──build──► texture.tex (drawn again)
```

- **`pack`** measures every source, works out where each one sits (MaxRects on an atlas, or one slice each on an
  array), draws the texture and writes the tracker beside it.
- **`build`** reads a tracker and draws its texture again. By default every region keeps the place the tracker
  gives it, so hand edits survive. With `--repack`, every region is placed again from scratch.

A region is named after its source file's stem. A game looks an image up by that name and reads its `Slice` and
`Crop`.

---

## 2. `pack`

```sh
Studio pack <image|folder>... --output <tracker.json> [switches]
```

### 2.1 Operands

Each operand is an image file or a folder.

- **A folder** is listed (not recursively). Every file in it with a texture source extension (`png`, `jpg`, `jpeg`,
  `tga`, `bmp`, `hdr`, `tex`) is taken, sorted by name case-insensitively so the pack is the same on every platform.
- **A file** is taken as it is, whatever its extension, and fails later if it cannot be read.
- The texture being written is skipped if a listed folder contains it (compared as text against the `--texture`
  path).
- `.tex` files are sources too. A folder that also holds older baked output packs that output again. Keep baked
  files out of source folders.

### 2.2 Region names

- One-slice source: the file's stem. `Sprites/hero.png` becomes `hero`.
- Multi-slice source (a `.tex` array or cube): one region per slice, `<stem>_000`, `<stem>_001`, ... in slice order.
- Two regions may not share a name, compared case-insensitively: `'A/hero.png' and 'B/hero.png' would both be named
  'hero'`, exit `1`. Rename one, or pack them into separate trackers.

### 2.3 Paths

- `--output` names the tracker. It is required.
- The texture goes beside it as `<tracker stem>.tex`, unless `--texture <path>` names another place.
- The tracker stores the texture path and every source path **relative to the tracker's own folder**, so the
  tracker and its sources can move together.
- Paths are related as text, not through the filesystem. A relative source with an absolute tracker path (or the
  other way round) is refused: `'<path>' is relative and '<folder>' is absolute, so one cannot be written against
  the other`. Use both relative or both absolute.
- Two absolute paths on different drives keep the source path absolute.

### 2.4 Switches

The texture switches of `textures.md` apply (`--format`, `--mipmaps`, `--linear`, `--layered`, `--compressed`).
`--cube` and `--array` do not. The pack switches:

| Switch | Default | Meaning |
|---|---|---|
| `--output <path>` | (required) | The tracker to write |
| `--texture <path>` | `<tracker stem>.tex` beside the tracker | Where to write the texture |
| `--mode <m>` | `Atlas` | `Atlas` packs regions together; `Array` gives each region its own slice |
| `--width <n>` | `2048` | Widest a slice may grow, in pixels |
| `--height <n>` | `2048` | Tallest a slice may grow, in pixels |
| `--padding <n>` | `1` | Atlas: empty pixels between neighbouring regions |
| `--extrude <n>` | `0` | Pixels each region's edge is repeated outward |
| `--extent <w>x<h>` | the largest region | Array: the size of every slice |
| `--fit <f>` | `Stretch` | Array: `Stretch`, `Center` or `Contain` each region into its slice |
| `--headroom <n>` | `1` | Array: round the slice count up to a multiple of n |
| `--fill <RRGGBBAA>` | `00000000` | Colour of every pixel no region covers, as the texture stores it |
| `--pot` | on | Atlas: round each side of the slice up to a power of two |
| `--square` | off | Atlas: keep the slice square |
| `--pages` | off | Atlas: spill what does not fit onto further slices |

A misspelt `--mode` or `--fit` is refused with `Studio: '<x>' is not a choice '--<name>' takes`, exit `2`.
`--square` with a `--width` other than `--height` is refused with `Texture: '--square' needs '--width' and
'--height' to match, not <w> and <h>`, exit `1`.

The tracker records the `--mipmaps`, `--linear` and `--compressed` settings the texture was baked with, so `build`
writes it the same way.

---

## 3. Atlas mode

```sh
Studio pack Sprites/ --output Atlas/Sprites.json --padding 2 --extrude 1 --no-linear
```

### 3.1 How regions are placed

1. Every source is measured. A region's size is its source's size, or its `From` crop when one is set.
2. Regions are ordered largest area first, ties in authored order, so the same input always packs the same way.
3. Each region takes `width + 2 * extrude + padding` by `height + 2 * extrude + padding` of space.
4. The search starts from the smallest power-of-two bin that holds the largest region and whose area could hold all
   of them, capped by `--width` and `--height`. The bin grows (the shorter side doubles) until every region fits.
5. Placement is MaxRects with the best-short-side-fit rule: each region takes the free rectangle that leaves the
   thinnest sliver along its tighter side.
6. When one slice at `--width` by `--height` still cannot hold every region:
   - with `--pages`, regions spill onto further slices of the full size, and the texture becomes a
     `Texture2DArray`;
   - without it, the pack fails: `'<name>' (WxH) does not fit a 2048x2048 slice, and '--pages' is off`.
7. A single slice shrinks to what its regions use (border included), then rounds up to powers of two with `--pot`
   and to a square with `--square`. Every slice of a multi-page atlas is the full `--width` by `--height`.

### 3.2 Padding and extrusion

- **Padding** is empty space between neighbours. It stops bilinear filtering from reading one region's texels at
  another's edge.
- **Extrusion** repeats each region's outermost texels outward by `n` pixels, so a filter that reads past the
  edge (bilinear, or a mip chain) samples the region's own edge colour. Its `Rect` stays the image's own size, and
  the extruded border sits around it.

For mipmapped atlases, give each region more extrusion and padding the deeper the chain goes. Two pixels hold up to
about the second mip level.

### 3.3 Format of the atlas

The tracker's `Format` is settled at the first draw: `--format` if given, else the format the **first** region's
source decoded into. With `--no-linear` that is `RGBA8UIntNorm_sRGB`; with the default `--linear`, `RGBA8UIntNorm`
(for RGB or RGBA sources). Every other source is converted into it.

---

## 4. Array mode

```sh
Studio pack Ground/ --output Ground.json --mode Array --extent 512x512 --fit Contain --mipmaps --no-linear
```

Every region gets a slice of its own, in the order the sources were given. The texture is always a
`Texture2DArray`.

| `--fit` | Region larger than the slice | Region smaller than the slice |
|---|---|---|
| `Stretch` (default) | Resized to fill the slice | Resized to fill the slice |
| `Center` | **Refused**: `'<name>' is WxH, larger than its slice; '--fit Contain' shrinks it` | Kept at its size, centred |
| `Contain` | Shrunk to fit, shape kept, centred | Kept at its size, centred |

- `--extent` sets the slice size. Without it, the slice is the widest region's width by the tallest region's height.
- `--padding` is not used in an array (written as `0`).
- `--extrude` repeats the edge into the empty space around a centred region. With `Stretch` it is written as `0`,
  since a stretched region fills its slice.
- `--headroom <n>` rounds the slice count up to a multiple of `n`, leaving blank slices (filled with `--fill`) for
  regions you add later without changing the array's depth.

Resizing (for `Stretch` and `Contain`) is a box filter over `RGBA32Float`.

---

## 5. The tracker file

The tracker is JSON, four-space indented. Revision `1` (`Texture::Tracker::kVersion`). An atlas of two images:

```json
{
    "Version": 1,
    "Texture": "Sprites.tex",
    "Layout": "Texture2D",
    "Format": "RGBA8UIntNorm_sRGB",
    "Width": 128,
    "Height": 64,
    "Slices": 1,
    "Padding": 2,
    "Extrude": 1,
    "Mipmaps": false,
    "Linear": false,
    "Compress": true,
    "Regions": [
        {
            "Name": "hero",
            "Source": "../Sprites/hero.png",
            "From": [0, 0, 48, 48],
            "Slice": 0,
            "Rect": [1, 1, 48, 48],
            "Crop": [0.0078125, 0.015625, 0.3828125, 0.765625]
        },
        {
            "Name": "coin",
            "Source": "../Sprites/coin.png",
            "From": [0, 0, 16, 16],
            "Slice": 0,
            "Rect": [53, 1, 16, 16],
            "Crop": [0.4140625, 0.015625, 0.5390625, 0.265625]
        }
    ]
}
```

### 5.1 Top-level keys

| Key | Type | Read default | Meaning |
|---|---|---|---|
| `Version` | number | (required) | Must be `1`, or the tracker is refused |
| `Texture` | string | empty | The texture to write, relative to the tracker. `build` falls back to `<tracker stem>.tex` when empty |
| `Layout` | string | `Texture2D` | `Texture2D` for a flat atlas, `Texture2DArray` for pages or an array |
| `Format` | string | `Unspecified` | The texture format. `Unspecified` takes the `--format` switch, else the first source's |
| `Width`, `Height` | number | `0` | The size of every slice, in pixels. `build` refuses `0` without `--repack` |
| `Slices` | number | `1` | The slice count, never below 1 |
| `Padding` | number | `0` | The padding the regions were packed with, reused by `build --repack` |
| `Extrude` | number | `0` | Pixels each region's edge is repeated outward when drawn |
| `Mipmaps` | bool | `false` | `true` to write a full mip chain. Always written |
| `Linear` | bool | `true` | `true` to read the sources as linear values, `false` for sRGB colour art. Always written |
| `Compress` | bool | `true` | `true` to LZ4-compress the payload when it comes out smaller. Always written |
| `Headroom` | number | `1` | Array slice count multiple. Written only when above 1 |
| `Fill` | `[r, g, b, a]` bytes | transparent | Colour of uncovered pixels, as stored. Written only when not transparent |
| `Regions` | array | empty | The regions, in authored order |

### 5.2 Region keys

| Key | Type | Read default | Meaning |
|---|---|---|---|
| `Name` | string | empty | The name a game looks the image up by |
| `Source` | string | (required) | The source image, relative to the tracker's folder |
| `Layer` | number | `0` | The slice of the source taken. Written only when above 0 |
| `From` | `[x, y, w, h]` | whole source | The part of the source taken, in source pixels |
| `Slice` | number | `0` | The slice of the texture the region sits on |
| `Rect` | `[x, y, w, h]` | `From`'s size at 0,0 | The place the region takes on its slice, in pixels. The image is resized to it |
| `Channels` | object | none | Channels read from other images, see 6 |
| `Retired` | bool | `false` | Kept and drawn, flagged for the game as no longer offered. Written only when `true` |
| `Crop` | `[u0, v0, u1, v1]` | (written only) | `Rect` as fractions of the slice, left, top, right, bottom |

Notes:

- `pack` measures every region, so `From` is always written after a pack, even for whole images.
- `Crop` is output only. `build` ignores it and writes it again from `Rect`, so edit `Rect`, never `Crop`.
- An area is `[x, y, width, height]` in whole pixels from the top-left. An area with a zero side is invalid and
  reads as "not set".
- `Retired` changes nothing in the texture. It is a flag for whatever reads the tracker. Use it to stop offering a
  region without moving the others.

---

## 6. Channel splicing

A region can take one or more of its channels from other images. This is how several greyscale maps are packed
into one texture (roughness in R, metalness in G, occlusion in B, height in A), or how a height is put into a
colour image's alpha.

```json
{
    "Name": "Crate_Surface",
    "Source": "../Art/Crate_Roughness.png",
    "From": [0, 0, 512, 512],
    "Slice": 0,
    "Rect": [0, 0, 512, 512],
    "Channels": {
        "G": { "Source": "../Art/Crate_Metallic.png",  "Channel": "R" },
        "B": { "Source": "../Art/Crate_Occlusion.png", "Channel": "R" },
        "A": { "Source": "../Art/Crate_Height.png",    "Channel": "R" }
    }
}
```

- `Channels` is keyed by the channel of the region written: `R`, `G`, `B` or `A` (upper case). A key may appear
  once, so one region never writes the same channel twice.
- `Source` is the image the channel is read from, relative to the tracker. It is always read as **linear data**,
  whatever `--linear` says, since it holds values, not colour.
- `Channel` is the channel of that image read: `R`, `G`, `B` or `A` (either case). Default `R`.
- The whole of the channel's source is fitted to the region's `Rect` (resized if needed), then the channel is copied
  in, after the region's own image is drawn and fitted.
- A channel entry without `Source`, or with a `Channel` that is not one letter of `RGBA`, fails the read.

`pack` never writes `Channels`. Pack the base image, add `Channels` by hand, then `build`.

Bake spliced data with `--linear` (the default). The base image is read in the profile's colour space, the spliced
ones always linearly, and an sRGB format would re-encode them all.

---

## 7. `build`

```sh
Studio build <tracker.json> [--repack] [switches]
```

### 7.1 What it does

1. Reads the tracker. Version must be `1`, every region must name a `Source`.
2. `--format`, `--fill`, `--headroom`, `--mipmaps`, `--linear` and `--compressed` (or their `--no-` forms) typed on
   the command line replace the tracker's own. The tracker is written back with the values used.
3. With `--repack`, every region's `Slice` and `Rect` are cleared and the regions are placed again (see 7.3).
4. Warns, without failing, when two regions on one slice overlap: `'a' and 'b' overlap on slice 0; the later one is
   drawn on top`.
5. Draws every slice and writes the texture and the tracker.

What `build` refuses:

| Error | Cause |
|---|---|
| `the tracker has no extent to draw into; pack it first` | `Width`, `Height` or `Slices` is 0 |
| `'<name>' sits on slice N of a flat texture, which has only one` | A `Texture2D` tracker with a region past slice 0 |
| `'<name>' sits outside its WxH slice` | A `Rect` past the slice's edge |
| `'<name>' takes WxH at X,Y, which leaves its WxH source` | A `From` past the source's edge |
| `'<name>' takes slice N of '<source>', which holds M` | A `Layer` past the source's slices |

An array tracker (`Texture2DArray`) grows its slice count to cover every region's `Slice`, rounded up to `Headroom`,
so adding a region on a new slice just works. A flat atlas does not grow.

### 7.2 What `build` remembers

The tracker holds `Format`, `Padding`, `Extrude`, `Fill`, `Headroom`, `Mipmaps`, `Linear` and `Compress`. `pack`
stores the `--mipmaps`, `--linear` and `--compressed` settings it baked with, and `build` uses them unless the
command line names one. A plain `Studio build <tracker.json>` therefore writes the texture as `pack` did.

A tracker written before these keys existed reads as `Mipmaps` off, `Linear` on and `Compress` on. Pass the switches
to its first `build` once, and the tracker keeps them from then on.

`--layered` is the one texture switch not stored. Left out of `build`, a flat atlas is written as `Texture2D`. Once
a layered build has run, the tracker's `Layout` is `Texture2DArray` and stays so.

### 7.3 `--repack`

`--repack` places every region again, reading the pack switches from the command line:

- `--padding` and `--extrude` carry over from the tracker unless typed.
- `--mode`, `--width`, `--height`, `--extent`, `--fit`, `--pot`, `--square` and `--pages` are read **with their
  defaults**. An array tracker repacked without `--mode Array` is repacked as an atlas. Retype every pack switch you
  used.
- `From` crops are kept, so a region keeps its crop and only its place changes.

### 7.4 Typical edits

| Goal | Edit, then `build` |
|---|---|
| Add an image | Append a region with `Name`, `Source` and a free `Rect` (or `build --repack` to place it) |
| Crop a sheet | Set `From` to the part wanted, and `Rect` to the size it should take |
| Use one cell of a sheet as several regions | Several regions with the same `Source`, different `From` |
| Take one slice of an array source | Set `Layer` |
| Recolour the empty space | `--fill` on `build`, or set `Fill` |
| Leave room in an array | `--headroom 8` on `build` |
| Change format | `--format BC7UIntNorm_sRGB` on `build` (sides must be multiples of four) |

---

## 8. `--fill`

`--fill <RRGGBB>` or `--fill <RRGGBBAA>`, hexadecimal, an optional leading `#`. Missing alpha is opaque. Any other
spelling is refused (exit `2`).

The fill is given **as the texture stores it**. For an sRGB format, `--fill 808080FF` stores the byte `0x80`, which
the GPU decodes to about 0.22 linear. A fill other than transparent black is written to the tracker as `Fill`.

---

## 9. Reading a tracker in a game

The tracker is plain JSON. A game reads `Texture` for the texture to load, and for each region `Name`, `Slice` and
`Crop` (or `Rect` with `Width` and `Height`):

```cpp
// Using the engine's JSON types.
const JsonValue  Document = JsonDocument::Parse(Content);
const JsonObject Root(Document);

const JsonArray Regions = Root.GetArray("Regions");
for (UInt Index = 0; Index < Regions.GetSize(); ++Index)
{
    const JsonObject Region = Regions.GetObject(Index);
    if (Region.GetBool("Retired", false))
    {
        continue;
    }
    const Text      Name  = Region.GetString("Name");
    const UInt16    Slice = Region.GetNumber<UInt16>("Slice", 0);
    const JsonArray Crop  = Region.GetArray("Crop");
    // Crop: left, top, right, bottom as fractions of the slice.
}
```

A host linking `Studio_Texture` can use `Studio::Texture::Tracker::Read` instead.

---

## 10. Library API

Headers under `Library/Texture/Source/Texture/Atlas/`.

### 10.1 `Tracker`, `Region`, `Area`, `Splice`

```cpp
struct Area   { UInt16 X, Y, Width, Height; Bool IsValid() const; UInt32 GetRight() const; UInt32 GetBottom() const;
                Bool Contains(ConstRef<Area>) const; Bool Overlaps(ConstRef<Area>) const; };
struct Splice { UInt8 Target = 3; Str Source; UInt8 Origin = 0; };
struct Region { Str Name; Str Source; UInt16 Layer = 0; Area From; UInt16 Slice = 0; Area Rect;
                Sequence<Splice> Channels; Bool Retired = false; };

struct Tracker
{
    static constexpr UInt32 kVersion = 1;
    Str Texture; ZyGraphic::TextureLayout Layout; ZyGraphic::TextureFormat Format;
    Bool Mipmaps = false; Bool Linear = true; Bool Compress = true;
    UInt16 Width, Height, Slices = 1, Padding, Extrude, Headroom = 1; IntColor8 Fill; Sequence<Region> Regions;

    UInt16      GetDepth() const;
    static Bool Read(Text Path, Ref<Tracker> Output);
    Bool        Write(Text Path) const;
    static Str  Resolve(Text Folder, Text Path);
    static Bool Relate(Text Folder, Text Path, Ref<Str> Output);
};
```

(Abbreviated. Field defaults are as in section 5.)

### 10.2 `Packer`

```cpp
enum class Packer::Mode    : UInt8 { Atlas, Array };
enum class Packer::Fitting : UInt8 { Stretch, Center, Contain };

struct Packer::Settings
{
    Mode Layout = Mode::Atlas; UInt16 Width = 2048; UInt16 Height = 2048; Profile::Extent Extent;
    Fitting Fit = Fitting::Stretch; UInt16 Padding = 1; UInt16 Extrude = 0;
    Bool PowerOfTwo = true; Bool Square = false; Bool Pages = false;
    static Settings From(ConstRef<Environment> Environment);
};

static Bool Packer::Pack(Ref<Tracker> Atlas, ConstRef<Settings> Settings);
```

`Pack` places regions that are already measured (their `Rect` holds their size).

### 10.3 `Composer`

```cpp
Composer(ConstRef<Baker> Baker, Text Folder, ConstRef<Profile> Profile);

Bool   Measure(Ref<Tracker> Atlas);                                // fill From and Rect sizes from the sources
Bool   Compose(Ref<Tracker> Atlas, Ref<Sequence<Bitmap>> Output);  // draw every slice
Blob   Bake(Ref<Tracker> Atlas);                                   // draw and encode as .tex bytes
Bool   Build(Ref<Tracker> Atlas);                                  // draw, encode and write the tracker's Texture
UInt16 Count(Text Source);                                         // slices a source holds
```

The composer reads each source once and caches it. `Folder` is the tracker's folder, which every relative path is
resolved against.

### 10.4 Packing from code

This is what `pack` does, minus folder listing and name checks:

```cpp
Studio::Texture::Baker   Textures(* Scheduler);
Studio::Texture::Profile Settings;
Settings.Linear  = false;
Settings.Mipmaps = true;

Studio::Texture::Tracker Atlas;
Atlas.Texture  = "Icons.tex";
Atlas.Mipmaps  = Settings.Mipmaps;   // kept so a later `build` writes it the same way
Atlas.Linear   = Settings.Linear;
Atlas.Compress = Settings.Compress;

for (const Text Name : { "sword"_Text, "shield"_Text, "potion"_Text })
{
    Ref<Studio::Texture::Region> Entry = Atlas.Regions.Append();
    Entry.Name   = Str(Name);
    Entry.Source = Str::Print("../Art/Icons/{0}.png", Name);
}

Studio::Texture::Packer::Settings Layout;
Layout.Padding = 2;
Layout.Extrude = 1;

Studio::Texture::Composer Drawer(Textures, "Content/UI", Settings);
if (Drawer.Measure(Atlas) && Studio::Texture::Packer::Pack(Atlas, Layout) && Drawer.Build(Atlas))
{
    Atlas.Write("Content/UI/Icons.json");
}
```

---

## 11. Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| `'pack' takes one or more images or folders, and an '--output <tracker>.json'` | `--output` missing | Add it |
| `would both be named 'x'` | Two sources share a stem | Rename one |
| `is relative and ... is absolute` | Mixed relative and absolute paths | Use one kind for both |
| `does not fit a 2048x2048 slice, and '--pages' is off` | Too much art for one slice | `--pages`, or raise `--width` / `--height` |
| Neighbouring sprites bleed at the edges | No padding or extrusion with filtering or mips | `--padding 2 --extrude 1` or more |
| Colours change or mips vanish after `build` | A tracker from before `Mipmaps` and `Linear` were stored | Pass `--no-linear` or `--mipmaps` to one `build`, which writes them into the tracker |
| `is not a choice '--mode' takes` | A misspelt `--mode` or `--fit` | Check the spelling |
| `'--square' needs '--width' and '--height' to match` | `--square` with unequal limits | Give both the same value |
| An array turned into an atlas | `build --repack` without `--mode Array` | Retype every pack switch with `--repack` |
| Old output packed into a new atlas | A `.tex` inside a source folder | Keep baked files elsewhere |
| A hand-edited `Crop` was lost | `Crop` is written from `Rect` | Edit `Rect` |
| A centred array region is refused | `--fit Center` with a region larger than the slice | `--fit Contain` or a larger `--extent` |

---

## 12. Where it lives

| Path | Holds |
|---|---|
| `Library/Texture/Source/Texture/Atlas/Tracker.hpp`, `Tracker.cpp` | The tracker structs, JSON reading and writing, path relating |
| `Library/Texture/Source/Texture/Atlas/Packer.hpp`, `Packer.cpp` | Atlas (MaxRects) and array placement |
| `Library/Texture/Source/Texture/Atlas/Composer.hpp`, `Composer.cpp` | Measuring, drawing, extrusion, channel splicing |
| `Application/Source/Console.cpp` | `Pack`, `Build`, `Collect`, `ReadTracker` |
