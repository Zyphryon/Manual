# Textures

This chapter covers baking images into `.tex`, picking a format, colour space and mip chain, cutting cube maps and
arrays, drawing normal and relief maps, and writing a texture back out as PNG. Atlases and packed arrays have their
own chapter, `trackers.md`.

---

## 1. What a texture bake does

```
source image ──decode──► one or more bitmaps ──cut (cube/array)──► mip chain ──convert──► block-encode ──► LZ4 ──► .tex
```

1. **Decode.** stb_image reads the source at its own depth (8-bit, 16-bit or float) and channel count.
2. **Cut.** With `--cube` or `--array`, the one image becomes six faces or a run of slices.
3. **Mip.** With `--mipmaps`, every slice gets a full chain down to 1x1, filtered at the source's depth.
4. **Convert.** Each level is transcoded into the target format (or the plain format a block format is packed from).
5. **Encode.** A block format (BC1 to BC5, BC7) is packed four by four texels.
6. **Write.** The header, then every slice with its whole chain, LZ4-compressed when that is smaller.

The output is the `ZTEX` file `ZyGraphic::TEXLoader` reads (layout in `formats.md`).

---

## 2. Sources

| Extension | Decoder | Notes |
|---|---|---|
| `png` | stb_image | 8 or 16 bits per channel |
| `jpg`, `jpeg` | stb_image | 8 bits |
| `tga` | stb_image | 8 bits |
| `bmp` | stb_image | 8 bits |
| `hdr` | stb_image | Radiance HDR, read as 32-bit float |
| `tex` | Studio's own reader | A baked texture, see 2.3 |

### 2.1 What format a source decodes into

When `--format` is left out, the texture is written in the format the source decoded into:

| Source channels | 8-bit, `--linear` | 8-bit, `--no-linear` | 16-bit | HDR (float) |
|---|---|---|---|---|
| 1 (grey) | `R8UIntNorm` | `RGBA8UIntNorm_sRGB` (expanded) | `R16UIntNorm` | `R32Float` |
| 2 (grey + alpha) | `RG8UIntNorm` | `RGBA8UIntNorm_sRGB` (expanded) | `RG16UIntNorm` | `RG32Float` |
| 3 (RGB) | `RGBA8UIntNorm` (alpha added, opaque) | `RGBA8UIntNorm_sRGB` | `RGBA16UIntNorm` | `RGBA32Float` |
| 4 (RGBA) | `RGBA8UIntNorm` | `RGBA8UIntNorm_sRGB` | `RGBA16UIntNorm` | `RGBA32Float` |

Two rules come from the engine's formats:

- **There is no three-channel format.** An RGB source always gains an opaque alpha.
- **sRGB exists only as 8-bit RGBA.** `--no-linear` on a 16-bit or float source logs a warning
  (`sRGB was requested but the source is 16-bit ...`) and keeps it linear. `--no-linear` on a one- or two-channel
  8-bit source expands it to RGBA (with a warning).

### 2.2 Size limits

| Limit | Value | Error |
|---|---|---|
| Side of a source | 65535 pixels | `exceeds the 65535 pixel limit` |
| Working canvas | width x height x 16 bytes must fit in 32 bits (about 16384 x 16384) | `too large to work on` |
| Slices in one texture | 65535 | `more than the 65535 one texture can hold` |
| Payload | 4 GiB | `past what one texture can hold` |

### 2.3 A `.tex` as the source

A baked texture can be baked again, to change its format, add or drop mips, or wrap it as an array:

```sh
Studio bake Content/Rock.tex Content/Rock_BC7.tex --format BC7UIntNorm --mipmaps
```

- The header's version must be `1` and the magic `ZTEX`.
- The format must hold plain interleaved texels. A block-compressed or bit-packed `.tex` cannot be read back:
  `'BC7UIntNorm' stores no plain interleaved texels, so it cannot be read back`. Keep an uncompressed master.
- Only the base level of each slice is kept. The chain is rebuilt if `--mipmaps` is on, and dropped if it is off.
- The layout (2D, array, cube) carries over. A source with several slices cannot be cut again with `--cube` or
  `--array`.
- Name a destination. The default (`<stem>.tex` beside the source) is the source itself, which is refused.

---

## 3. Switches

These apply to `bake`, `pack` and `build`, and to `normal` and `relief` when they write a `.tex`.

| Switch | Profile field | Default | Meaning |
|---|---|---|---|
| `--format <name>` | `Format` | `Unspecified` (inferred, see 2.1) | The format to write. `Studio formats` lists every accepted name |
| `--mipmaps` | `Mipmaps` | **off** | Generate a full mip chain down to 1x1 |
| `--compressed` | `Compress` | on | LZ4-compress the payload when that comes out smaller |
| `--linear` | `Linear` | **on** | Read the source as linear values. `--no-linear` reads it as sRGB colour art |
| `--layered` | `Layered` | off | Write a lone 2D image as a one-slice `Texture2DArray` |
| `--cube <c>x<r>` | `Cube` | none | `bake` only: cut the source into six cube faces, a grid counted in faces |
| `--array <w>x<h>` | `Slice` | none | `bake` only: cut the source into array slices of that size in texels |

Toggles take `--no-` to turn off: `--no-mipmaps`, `--no-compressed`, `--no-linear`, `--no-layered`.

An extent is `<width>x<height>` (an upper-case `X` works too). Each side must be 1 to 65535. A malformed extent is
logged and ignored, as if the switch were absent.

### 3.1 The two defaults to remember

**`--linear` is on.** A PNG of colour art baked with no switches is stored as `RGBA8UIntNorm`: its sRGB-encoded bytes
go to the GPU as if they were linear. Lighting, blending and filtering then work on the wrong values. Bake colour
art (albedo, UI images, sprites) with `--no-linear` or `--preset sprite`. Keep `--linear` for data: normal maps,
roughness, masks, heights, lookup tables.

**`--mipmaps` is off.** A texture without mips shimmers and aliases when drawn smaller than its size. Turn mips on
for anything drawn in a 3D world. Leave them off for UI images drawn at their own size.

### 3.2 How `--linear` and `--format` interact

`--linear` decides how the **source** is read. `--format` decides how the **output** is stored. Studio converts
between the two:

| `--linear` | `--format` | Result |
|---|---|---|
| on | none | Values stored as they are, in a linear format |
| off | none | Values stored as they are, in `RGBA8UIntNorm_sRGB` |
| off | any `_sRGB` format | Values stored as they are (decoded to linear, then re-encoded) |
| on | any `_sRGB` format | **The values are encoded through the sRGB curve.** A colour image comes out brighter |
| off | a linear format | **The values are decoded from sRGB.** A colour image comes out darker and bands in the shadows |

For colour art, use `--no-linear` with either no format or a `_sRGB` one. For data, use `--linear` with a linear
format.

### 3.3 Mip filtering

Each level is a box average of the level above it, every target texel averaging the source texels it covers. For
an sRGB format the average is taken in linear light (each texel is decoded through the sRGB curve, averaged and
re-encoded), so mips of colour art do not darken. Filtering runs at the source's own depth, and the conversion to
the target format happens after the whole chain is built.

The level count is `max(1, bit width of the longer side)`, which is a chain down to 1x1 for any size, power of two
or not.

### 3.4 LZ4 compression

`--compressed` (on) packs the payload with LZ4 at level 9. When that is not smaller, the payload is stored raw. It
is a file-size and load-time saving only. The engine expands it on load, and the GPU sees the same bytes either way.
Turn it off only to inspect a payload by hand.

---

## 4. Formats

### 4.1 What can be written

`Texture::Exporter::IsSupported` accepts:

- **Block formats with an encoder:**

  | Format | Packed from | Encoder |
  |---|---|---|
  | `BC1UIntNorm`, `BC1UIntNorm_sRGB` | `RGBA8UIntNorm` / `_sRGB` | rgbcx BC1, highest quality level |
  | `BC2UIntNorm`, `BC2UIntNorm_sRGB` | `RGBA8UIntNorm` / `_sRGB` | 4-bit explicit alpha plus rgbcx BC1 colour |
  | `BC3UIntNorm`, `BC3UIntNorm_sRGB` | `RGBA8UIntNorm` / `_sRGB` | rgbcx BC3 high quality |
  | `BC4UIntNorm` | `R8UIntNorm` | rgbcx BC4 high quality |
  | `BC5UIntNorm` | `RG8UIntNorm` | rgbcx BC5 high quality (red and green) |
  | `BC7UIntNorm`, `BC7UIntNorm_sRGB` | `RGBA8UIntNorm` / `_sRGB` | bc7enc, uber level 2, linear weights for the non-sRGB one |

- **Plain formats** that are samplable, not depth or stencil, not bit-packed, and either normalized at 8 or 16
  bits per component or float at 16 or 32 bits per component. For example `R8UIntNorm`, `RG8UIntNorm`,
  `RGBA8UIntNorm`, `RGBA8UIntNorm_sRGB`, `R16UIntNorm`, `RGBA16UIntNorm`, the signed-normalized variants,
  `R16Float`, `RGBA16Float`, `R32Float`, `RGBA32Float`.

Not written: `BC4SIntNorm`, `BC5SIntNorm`, `BC6UFloat`, `BC6SFloat`, every `ETC2` and `EAC` format, raw integer
formats (`R8UInt`, `RGBA32UInt`, ...), packed formats (`RGB10A2UIntNorm`, `R11G11B10Float`, `RGB9E5Float`) and depth
formats. Run `Studio formats` for the exact list your build prints.

### 4.2 Block format rules

- **Both sides must be multiples of four**: `'BC7UIntNorm' needs sides that are multiples of four, which 250x250 are
  not`. Resize the source, or let `pack` pick a size (`--pot` on by default rounds slices to powers of two).
- Mip levels smaller than a block are padded by repeating their last texel, so a full chain down to 1x1 works.
- The channel count is the encoder's: BC4 keeps red only, BC5 red and green only. A three-channel normal map
  written as `BC5UIntNorm` keeps X and Y, and the shader must rebuild Z.
- An unknown name in `--format` is refused (exit `2`), never silently replaced by the source's format.

### 4.3 Choosing a format

| Content | Suggested switches |
|---|---|
| Colour art for a 3D surface | `--no-linear --mipmaps --format BC7UIntNorm_sRGB` (or `BC1UIntNorm_sRGB` with no alpha) |
| UI image drawn at its own size | `--no-linear` |
| Tangent-space normal map | `--linear --mipmaps` (`RGBA8UIntNorm`), or `--format BC5UIntNorm` if the shader rebuilds Z |
| One greyscale map (height, mask, roughness) | `--linear --mipmaps --format R8UIntNorm` or `BC4UIntNorm` |
| HDR environment | `.hdr` source, `--format RGBA16Float` |

Whether a shader samples a 2D texture or an array is up to the shader. When it samples an array, write even a
single image with `--layered`.

---

## 5. Cube maps

```sh
Studio bake Sky.png Sky.tex --cube 4x3 --mipmaps --no-linear
```

`--cube <columns>x<rows>` cuts the source into six faces and writes a `TextureCube`. The grid is counted in faces,
not texels, and must be one of these:

| Grid | Arrangement | Face cells (column,row) for +X, -X, +Y, -Y, +Z, -Z |
|---|---|---|
| `6x1` | Strip across | (0,0) (1,0) (2,0) (3,0) (4,0) (5,0) |
| `1x6` | Strip down | (0,0) (0,1) (0,2) (0,3) (0,4) (0,5) |
| `4x3` | Horizontal cross | (2,1) (0,1) (1,0) (1,2) (1,1) (3,1) |
| `3x4` | Vertical cross | (2,1) (0,1) (1,0) (1,2) (1,1) (1,3), the last turned 180 degrees |
| `3x2` | Two rows of three | (0,0) (1,0) (2,0) (0,1) (1,1) (2,1) |
| `2x3` | Three rows of two | (0,0) (1,0) (0,1) (1,1) (0,2) (1,2) |

The horizontal cross, laid out:

```
        +Y
  -X    +Z    +X    -Z
        -Y
```

The image's width must divide by the columns and its height by the rows. Studio does not check that the faces come
out square. Make them square. Any other grid fails with `<c>x<r> is not a grid any cube arrangement is packed in`.

---

## 6. Arrays cut from a grid

```sh
Studio bake Tiles.png Tiles.tex --array 64x64 --mipmaps --no-linear
```

`--array <width>x<height>` cuts the source into cells of that size in texels, walking it row by row from the top
left, and writes a `Texture2DArray` with one slice per cell. A 256x128 source cut at `64x64` gives eight slices:
0 to 3 along the top row, 4 to 7 along the bottom.

- The cell size must divide the image exactly: `64x48 cells do not tile a 250x128 atlas exactly`.
- `--cube` and `--array` cannot both be given.
- A source that already holds several slices (a baked array or cube) cannot be cut again.

To build an array from separate files instead of one grid, use `pack --mode Array` (`trackers.md`).

---

## 7. A one-slice array: `--layered`

`--layered` turns a lone 2D image into a `Texture2DArray` with one slice. It changes nothing for a source that is
already an array or a cube. Use it when a shader declares its texture as an array, so one material can take either a
single image or a stack. The built-in presets `sprite`, `normal` and `relief` all turn it on.

---

## 8. Normal maps: `normal`

```sh
Studio normal <source> [destination] [switches]
```

Reads one image as a height field and draws a tangent-space normal map from its slopes.

### 8.1 Switches

| Switch | Default | Meaning |
|---|---|---|
| `--channel <c>` | `Luminance` | What is read as height: `Luminance`, `Red`, `Green`, `Blue` or `Alpha` |
| `--filter <k>` | `Sobel` | Slope filter: `Central`, `Sobel`, `Scharr` or `Prewitt` |
| `--strength <x>` | `2` | How steep one unit of height reads |
| `--blur <px>` | `0` | Gaussian blur of the height before slopes are measured, its sigma in pixels |
| `--octaves <n>` | `1` | Scales summed, each twice as coarse as the one before, clamped to 1-8 |
| `--falloff <x>` | `0.5` | Weight of each coarser scale against the one before |
| `--invert` | off | Read dark as raised |
| `--flip-x` | off | Mirror the X axis |
| `--flip-y` | off | Mirror the Y axis, for programs that expect green pointing down |
| `--wrap` | off | Wrap around the edges, for tiling textures |
| `--masked` | **on** | Transparent pixels (alpha below one half) are ground level, and their normal points straight out |

A misspelt `--channel` or `--filter` (and `--shape` on `relief`) is refused with `Studio: '<x>' is not a choice
'--<name>' takes`, exit `2`.

The filters, from sharpest to softest:

| Filter | Outer rows | Centre row | Character |
|---|---|---|---|
| `Central` | 0 | 1 | Two neighbours along the axis, sharpest and noisiest |
| `Sobel` | 1 | 2 | The usual choice |
| `Scharr` | 3 | 10 | Responds the same in every direction |
| `Prewitt` | 1 | 1 | Weighs its three rows alike, softest |

The output packs the normal into RGB as `n * 0.5 + 0.5` and **keeps the source's alpha**. With `--masked` on, the
outline of cut-out art reads as the edge of a raised shape. For a fully opaque tiling texture, `--masked` changes
nothing; add `--wrap` so the edges match.

### 8.2 Output

| Destination | Written as |
|---|---|
| none | `<source stem>_normal.png` beside the source |
| `*.png` | An 8-bit RGBA PNG, values stored as they are (no sRGB curve) |
| `*.tex` | A native texture. The format defaults to `RGBA8UIntNorm` when `--format` is left out; every texture switch applies |

A source with several slices (a `.tex` array) can only go to a `.tex` destination, which keeps every slice:
`'<file>' holds 4 slices, which only a '.tex' destination keeps`.

```sh
# A tiling rock normal map, as a mipmapped one-slice array
Studio normal Rock_Height.png Rock_Normal.tex --preset normal --wrap --strength 4

# Broad shapes and fine grain together
Studio normal Bark.png Bark_N.png --octaves 4 --falloff 0.6 --blur 0.5
```

---

## 9. Relief maps: `relief`

```sh
Studio relief <source> [destination] [switches]
```

Draws a greyscale height map: white where raised, darker as it sinks. Useful as a displacement or parallax map, or to
bevel flat art.

### 9.1 Switches

`--channel`, `--invert`, `--blur`, `--wrap` and `--masked` work as for normal maps. With `--masked`, transparent
pixels sink to the bottom (0).

| Switch | Default | Meaning |
|---|---|---|
| `--low <x>` | `0` | The height read as the bottom, anything below clamped |
| `--high <x>` | `1` | The height read as the top, anything above clamped |
| `--contrast <x>` | `1` | Spread around the middle. Above 1 hardens |
| `--brightness <x>` | `0` | Added to every height, before the contrast |
| `--gamma <x>` | `1` | Exponent on the heights. Above 1 sinks the middle tones |
| `--bevel <px>` | `0` | Distance from the outline to full height, in pixels. 0 for none |
| `--shape <s>` | `Round` | Bevel profile: `Linear`, `Round`, `Sharp` or `Plateau` |
| `--detail <x>` | `1` | How much of the image's own height rides on the bevel |

### 9.2 Order of operations

For each pixel, in this order (the order an image editor applies them):

1. Levels: `v = clamp((h - low) / (high - low), 0, 1)`.
2. Tone: `v = clamp((v + brightness - 0.5) * contrast + 0.5, 0, 1)`.
3. Gamma: `v = v ^ gamma`.
4. Bevel (when `--bevel` > 0): `v = rise(d / bevel) * (1 - detail + detail * v)`, where `d` is the distance to the
   outline in pixels.
5. Mask: a transparent pixel becomes 0.

The bevel shapes:

| Shape | Curve | Looks like |
|---|---|---|
| `Linear` | straight ramp | a chamfer |
| `Round` | quarter circle (out-circ) | a pillow |
| `Sharp` | stays low, rises late (in-quad) | a blade |
| `Plateau` | smooth step with a flat top | a coin |

The output stores the height in RGB and keeps the source's alpha. Destinations work as for `normal`, with the
default `<source stem>_relief.png`. The built-in `relief` preset writes `R8UIntNorm`, one channel.

```sh
# A coin-like bevel on a cut-out icon, as a one-channel mipmapped array
Studio relief Icon.png Icon_Height.tex --preset relief --bevel 6 --shape Plateau --detail 0.3
```

---

## 10. Back to PNG: `extract`

```sh
Studio extract <texture> [destination]
```

Writes each slice of a texture as an 8-bit RGBA PNG. The source can be any texture source, a `.tex` included.

- One slice: written to the destination, default `<stem>.png`.
- Several slices: each written as `<destination stem>_<index>.png`, starting at `_0`.
- A slice in an sRGB format is written sRGB-encoded. Anything else, data included, is written as the values it holds.
- Only plain formats can be extracted (see 2.3). A BC-compressed `.tex` cannot.

```sh
$ Studio extract Content/Tiles.tex Review/Tiles.png
Texture: 'Content/Tiles.tex' -> 8 picture(s) at 'Review/Tiles.png'
# writes Review/Tiles_0.png ... Review/Tiles_7.png
```

---

## 11. Library API

Headers under `Library/Texture/Source/Texture/`.

### 11.1 `Texture::Profile`

```cpp
struct Profile final
{
    struct Extent final { UInt16 Width = 0; UInt16 Height = 0; Bool IsValid() const;
                          static Extent From(ConstRef<Environment> Environment, Text Name); };

    ZyGraphic::TextureFormat Format   = ZyGraphic::TextureFormat::Unspecified;
    Bool                     Mipmaps  = false;
    Bool                     Linear   = true;
    Bool                     Compress = true;
    Bool                     Layered  = false;
    Extent                   Cube;
    Extent                   Slice;

    ZyGraphic::TextureLayout GetLayout(ZyGraphic::TextureLayout Layout) const;
    static Profile From(ConstRef<Environment> Environment);
};
```

`Profile::From` reads an unknown `--format` as `Unspecified`. The command line checks that case itself and refuses
it. A host reading formats from its own UI should do the same.

### 11.2 `Texture::Baker`

```cpp
explicit Baker(Ref<ZyJob::Service> Scheduler);

Surface Load(Text Path, ConstRef<Profile> Profile) const;
Blob    Encode(AnyRef<Sequence<Bitmap>> Slices, ZyGraphic::TextureLayout Layout, ConstRef<Profile> Profile) const;
Blob    Bake(ConstSpan<Byte> Source, Text Type, ConstRef<Profile> Profile) const;
Bool    Bake(Text Source, Text Destination, ConstRef<Profile> Profile) const;
```

- `Load` decodes a file into a `Surface` (a layout plus one `Bitmap` per slice) without writing anything.
- `Encode` writes your own bitmaps as a texture. Every slice must share one extent.
- The `Bake` pair runs the whole pipeline in memory or from file to file.

### 11.3 Lower-level pieces

| Class | Use |
|---|---|
| `Importer::Import(ConstSpan<Byte>, Text Type, ConstRef<Profile>)` | Decode bytes into a `Surface` |
| `Exporter::Export(Scheduler, Slices, Layout, Profile)` | Write bitmaps as `.tex` bytes |
| `Exporter::Prepare(Scheduler, Bitmap, UInt8 Levels, TextureFormat)` | Mip, convert and block-encode one slice for a live upload, without the file wrapper |
| `Exporter::IsSupported(TextureFormat)` | Whether a format can be written |
| `Encoder::IsSupported`, `Encoder::GetSource` | Whether a block format has an encoder, and the plain format it packs from |
| `Slicer::Describe(Columns, Rows)`, `Slicer::Divide(Bitmap, W, H)`, `Slicer::Slice(Bitmap, Layout)` | Cube and grid cutting |
| `Resampler::Resize`, `Resampler::Mipmap` | Box resize and mip chains |
| `Transcoder::Transcode(Bitmap, Format)` | Convert between plain formats, colour space included |
| `Canvas` | One level as linear `RGBA32Float`, for drawing; `Canvas::From(Bitmap)`, `Canvas::To(Format)` |
| `Normal::Generate(Canvas, Settings)`, `Relief::Generate(Canvas, Settings)` | The generators behind `normal` and `relief` |
| `Picture::Encode(Canvas, Bool sRGB)`, `Picture::Write(Path, Canvas, Bool sRGB)` | Write a canvas as an 8-bit RGBA PNG |

```cpp
// Draw a normal map in memory and keep it as a PNG for review.
Studio::Texture::Baker Textures(* Scheduler);
const Studio::Texture::Surface Height = Textures.Load("Art/Brick_Height.png", Studio::Texture::Profile());

Studio::Texture::Normal::Settings Knobs;
Knobs.Strength   = 3.0f;
Knobs.Input.Wrap = true;

const Studio::Texture::Canvas Map
    = Studio::Texture::Normal::Generate(Studio::Texture::Canvas::From(Height.Slices.GetFront()), Knobs);
Studio::Texture::Picture::Write("Review/Brick_Normal.png", Map, false);
```

---

## 12. Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Colour art looks washed out or wrong under light | Baked with the default `--linear` | `--no-linear` or `--preset sprite` |
| Colour art comes out brighter | `--format ..._sRGB` with `--linear` on | Add `--no-linear` |
| Colour art comes out darker, banded | `--no-linear` with a linear `--format` | Use a `_sRGB` format |
| Texture shimmers in the distance | No mips (the default) | `--mipmaps` |
| `needs sides that are multiples of four` | A BC format on an odd-sized image | Resize, or pick a plain format |
| `is not a format this exporter can write` | ETC2, EAC, BC6, a raw integer or packed format | Pick from `Studio formats` |
| `stores no plain interleaved texels` | Re-baking or extracting a BC `.tex` | Keep and bake from the source image |
| Normal map's edges seam on a tiling surface | No `--wrap` | Add `--wrap` |
| Normal map lighting is upside down | The other program expects green down | `--flip-y` |
| A shader declared for arrays shows nothing | A plain 2D texture was baked | `--layered` |

---

## 13. Where it lives

| Path | Holds |
|---|---|
| `Library/Texture/Source/Texture/Profile.hpp`, `Profile.cpp` | Settings and switch reading |
| `Library/Texture/Source/Texture/Baker.hpp`, `Baker.cpp` | The bake pipeline, cutting rules |
| `Library/Texture/Source/Texture/Import/Importer.cpp` | stb decoding, format resolution, `.tex` reading |
| `Library/Texture/Source/Texture/Export/Exporter.cpp` | Supported formats, mips, payload, header |
| `Library/Texture/Source/Texture/Export/Picture.cpp` | PNG writing |
| `Library/Texture/Source/Texture/Process/Encoder.cpp` | BC1 to BC5 and BC7 encoding |
| `Library/Texture/Source/Texture/Process/Resampler.cpp` | Box resize and mip filtering |
| `Library/Texture/Source/Texture/Process/Transcoder.cpp` | Format and colour-space conversion |
| `Library/Texture/Source/Texture/Process/Slicer.cpp` | Cube arrangements and grid cutting |
| `Library/Texture/Source/Texture/Generate/Field.cpp` | Height reading, blur |
| `Library/Texture/Source/Texture/Generate/Normal.cpp`, `Relief.cpp` | The two generators |
| `Application/Source/Console.cpp` | `Generate`, `Normal`, `Relief`, `Extract`, `Formats` handlers |
