# File formats

This chapter describes every file Studio writes and the engine then loads: the binary `.tex`, `.fnt`, `.snd`,
`.msh`, `.skl` and `.anm`, and the JSON `.mdl`, `.mtl` and tracker. Use it to write a loader of your own, to check a
file by hand, or to understand a loader's refusal.

All binary files are **little-endian**, unaligned, written field after field with the engine's `Writer`. Every
binary file starts with a four-character magic stored as a little-endian `UInt32` (the characters in reading
order) and a `UInt16` revision. Every revision is currently `1`; `Studio version` prints them.

| File | Magic | Revision | Written by | Read by |
|---|---|---|---|---|
| `.tex` | `ZTEX` | 1 | `Texture::Exporter` | `ZyGraphic::TEXLoader` |
| `.fnt` | `ZFNT` | 1 | `Font::Exporter` | `ZyRender::FNTLoader` |
| `.snd` | `ZSND` | 1 | `Sound::Exporter` | `ZyAudio::SNDLoader` |
| `.msh` | `ZMSH` | 1 | `Model::Exporter::ExportMesh` | `ZyGraphic::MSHLoader` |
| `.skl` | `ZSKL` | 1 | `Model::Exporter::ExportSkeleton` | `ZyRender::SKLLoader` |
| `.anm` | `ZANM` | 1 | `Model::Exporter::ExportAnimation` | `ZyRender::ANMLoader` |
| `.mdl` | JSON | none | `Model::Exporter::ExportModel` | `ZyRender::MDLLoader` |
| `.mtl` | JSON | none | `Material::Document::Write` | `ZyGraphic::MTLLoader` |
| tracker `.json` | JSON | `"Version": 1` | `Texture::Tracker::Write` | `Texture::Tracker::Read` (your game reads it too) |

A loader that meets the wrong magic logs `'<file>' is not a ZTEX file (bad magic)`; the wrong revision,
`'<file>' has an unsupported ZTEX version`.

---

## 1. Shared building blocks

### 1.1 Compressed block

Payloads are written with `Writer::WriteBlockCompressed<UInt32>`:

| Field | Type | Meaning |
|---|---|---|
| `Size` | `UInt32` | The payload's length once expanded |
| `Length` | `UInt32` | The length of the bytes that follow |
| `Bytes` | `Byte[Length]` | LZ4 data, or the raw payload when `Length == Size` |

When LZ4 would not shrink the payload, the raw bytes are stored and `Length` equals `Size`. A "not compressed" bake
(`--no-compressed`) writes the same layout with the raw payload, so a reader handles both the same way: if
`Length == Size`, copy; otherwise LZ4-decode into `Size` bytes. LZ4 levels used: textures and font pages 9, sounds
6, mesh buffers 3.

### 1.2 Chunk

Meshes and fonts use tag-length-value chunks after their header:

| Field | Type | Meaning |
|---|---|---|
| `Tag` | `UInt32` | Four characters, little-endian |
| `Length` | `UInt32` | The byte length of the body |
| `Body` | `Byte[Length]` | Chunk contents |

A reader skips a tag it does not know by its length.

### 1.3 Enumerations

Engine enumerations are written as their underlying type:

| Enum | Type | Values |
|---|---|---|
| `ZyGraphic::TextureLayout` | `UInt8` | `0` Texture2D, `1` Texture2DArray, `2` TextureCube, `3` Texture3D |
| `ZyGraphic::TextureFormat` | `UInt8` | The declaration order in `Zyphryon.Graphic/Types.hpp`, `0` being `Unspecified` |
| `ZyAudio::Encoding` | `UInt8` | `0` Linear, `1` Adaptive, `2` Opus |
| `Interpolation` | `UInt8` | `0` Step, `1` Linear, `2` Cubic, `3` Hermite |
| `ZyGraphic::VertexSlot`, `ZyGraphic::VertexFormat` | `UInt8` | Declaration order in `Zyphryon.Graphic/Types.hpp` |
| `ZyGraphic::Mesh::Property` | `UInt8` | Bit flags, `1` = Extended (32-bit indices) |

Format numbers depend on the engine's declaration order. A loader of your own should share the engine's header
rather than hard-code numbers.

---

## 2. Texture (`.tex`)

| Field | Type | Meaning |
|---|---|---|
| Magic | `UInt32` | `ZTEX` |
| Revision | `UInt16` | `1` |
| Layout | `UInt8` | `TextureLayout` |
| Format | `UInt8` | `TextureFormat` |
| Width | `UInt16` | Base level width, pixels |
| Height | `UInt16` | Base level height, pixels |
| Layers | `UInt16` | Slices (array) or faces (cube, 6); 1 for a plain 2D texture |
| Levels | `UInt8` | Mip levels per slice, the base included |
| Payload | compressed block | Every slice, see below |

The header is 15 bytes before the payload block.

**Payload order is slice-major**: slice 0 with its whole chain (level 0, level 1, ... level N-1), then slice 1 with
its whole chain, and so on. Each level is tightly packed rows, top row first:

- Plain formats: `width * height * bytes per texel`.
- Block formats: `ceil(width / 4) * ceil(height / 4) * block size` (8 bytes for BC1 and BC4, 16 for BC2, BC3, BC5,
  BC7).

Level `n`'s extent is `max(1, side >> n)`. The engine's `ZyGraphic::GetLevelOffset(Format, Width, Height, Level)`
gives each level's offset within one slice, and `GetLevelOffset(Format, Width, Height, Levels)` the length of one
slice's whole chain. Studio's own reader checks `slice length * Layers == payload size`.

Cube faces are stored in order +X, -X, +Y, -Y, +Z, -Z.

A `.tex` is also the body of every font atlas chunk (section 3).

---

## 3. Font (`.fnt`)

| Field | Type | Meaning |
|---|---|---|
| Magic | `UInt32` | `ZFNT` |
| Revision | `UInt16` | `1` |
| Metrics | `ZyRender::Font::Metrics`, through the engine's `Archive` | Seven `Real32`: `Size`, `Distance`, `Ascender`, `Descender`, `UnderlineOffset`, `UnderlineSize`, `UnderlineThickness` |
| Glyphs | `ZyRender::Font::Glyphs` (`Table<UInt32, Glyph>`), through `Archive` | Per codepoint: `Local` and `Atlas` rectangles, `Advance`, `Page` |
| Chunks | chunk stream | `KERN` (optional), then one `ATLS` per page in page order |

| Chunk | Body |
|---|---|
| `KERN` | `ZyRender::Font::Kerning` (`Table<UInt64, Real64>`) through `Archive`: key `(first << 32) \| second`, value in em units. Left out when there are no pairs |
| `ATLS` | A complete `.tex` file: `ZTEX`, revision 1, `Texture2D`, `RGBA8UIntNorm`, side, side, 1 layer, 1 level, LZ4 level 9 payload |

Pages are square. The `Page` of a glyph indexes the `ATLS` chunks in order. A font with no `ATLS` chunk is refused:
`'<file>' is missing its atlas chunk`.

The `Archive` encoding of metrics, tables and their elements is the engine's own serializer. Read it with the
engine's `Archive` class rather than by offset.

Units: everything in metrics, glyph `Local`, `Advance` and kerning is in **em units** (one em = 1.0). `Atlas` is in
page fractions. `Distance` is in atlas texels.

---

## 4. Sound (`.snd`)

| Field | Type | Meaning |
|---|---|---|
| Magic | `UInt32` | `ZSND` |
| Revision | `UInt16` | `1` |
| Encoding | `UInt8` | `ZyAudio::Encoding` |
| Stride | `UInt16` | Channels, 1 or 2 |
| Frequency | `UInt32` | Always `48000` |
| Frames | `UInt64` | Frames in the sound (one frame = one sample per channel) |
| Payload | compressed block | Encoded samples |

The engine refuses a stride of 0 or above 2, and a frequency other than 48000.

### 4.1 Linear payload

`SInt16[Frames * Stride]`, interleaved.

### 4.2 Adaptive payload

`ceil(Frames / 512)` frame blocks, each holding one 260-byte block per channel, channel 0 first:

| Offset | Size | Meaning |
|---|---|---|
| 0 | 2 | First sample of the block, `SInt16` |
| 2 | 1 | Step index the block starts at |
| 3 | 1 | Zero |
| 4 | 256 | 511 four-bit IMA ADPCM codes, two per byte, earlier sample in the low nibble (the last nibble unused) |

The step index carries over between blocks. The last block is zero-padded past `Frames`.

### 4.3 Opus payload

| Field | Type | Meaning |
|---|---|---|
| Pre-skip | `UInt16` | Frames to discard at the start (the encoder's lookahead, plus the tail a `--loop` sound is primed with, which makes `959` in all) |
| Packet frames | `UInt16` | `960` |
| Packets | `UInt32` | Packet count |
| Lengths | `UInt16[Packets]` | Each packet's byte length |
| Data | bytes | The packets, back to back |

Packets are Opus at 48 kHz with `Stride` channels.

---

## 5. Mesh (`.msh`)

| Field | Type | Meaning |
|---|---|---|
| Magic | `UInt32` | `ZMSH` |
| Revision | `UInt16` | `1` |
| Chunks | chunk stream | `MESH`, `VBUF`, `IBUF`, in that order |

### 5.1 `MESH` chunk

| Field | Type | Meaning |
|---|---|---|
| Properties | `UInt8` | `Mesh::Property`, `Extended` when indices are 32-bit |
| Extent | `Box` | Local-space bounds of every vertex, the engine's `Box` written raw |
| Bindings | `UInt8` | Binding count, at most 255 |
| per binding: Slot | `UInt8` | `VertexSlot` |
| per binding: Format | `UInt8` | `VertexFormat` |
| per binding: Stride | `UInt16` | Bytes between one vertex's element and the next |
| per binding: Offset | `UInt32` | Byte offset of vertex 0's element in the vertex buffer |
| Levels | `UInt8` | Level count, 1 to 8 (`Mesh::kMaxDetail`) |
| per level: Coverage | `Real32` | Smallest view-height share the level is drawn at; the last is 0 |
| Primitives | `UInt16` | Primitive count |
| per primitive | `Mesh::Primitive`, written raw | Draw range (index count, base vertex 0, first index, 1 instance), material slot, level, caster flag |

Primitives are grouped by level, level 0 first. Within a level, its own runs (one per material) come first, then
its shadow caster runs (`Caster` set). A caster run indexes the same vertices as the rest, and may name fewer of
them than the run it stands for, where corners a shadow cannot tell apart were welded. The loader refuses a
primitive naming a level past `Levels`.

### 5.2 `VBUF` chunk

A compressed block holding the vertex buffer. Streams are laid one after another. Within a stream, each vertex's
attributes are interleaved in the order the `--attributes` run named them. Element encodings:

| Attribute | Format | Bytes |
|---|---|---|
| Position | `Float32x3` | 12 |
| Normal | `Float32x3` | 12 |
| Tangent | `Float32x4` (W = bitangent sign) | 16 |
| TexCoord0, TexCoord1 | `Float32x2` | 8 |
| Color0 | `UIntNorm8x4` | 4 |
| BlendIndices | `UInt16x4` | 8 |
| BlendWeights | `UIntNorm16x4`, summing to exactly 65535 | 8 |

### 5.3 `IBUF` chunk

A compressed block holding the index buffer: `UInt16` indices, or `UInt32` when `Extended` is set (more than 65535
vertices). Level 0's runs come first, each reduced level's runs are appended after.

---

## 6. Skeleton (`.skl`)

| Field | Type | Meaning |
|---|---|---|
| Magic | `UInt32` | `ZSKL` |
| Revision | `UInt16` | `1` |
| Count | `SInt16` | Bone count, 1 to 32767 |
| Bones | `ZyRender::Skeleton3D::Bone[Count]`, the struct's memory image | See below |

`Skeleton3D::Bone`:

| Field | Type | Meaning |
|---|---|---|
| `Name` | `UInt64` | Hash of the bone's name (`Text::Hash`), the key clip lanes match |
| `Parent` | `SInt16` | Index of the parent bone, or -1 for a root |
| `Position` | `Vector3` | Rest position, relative to the parent |
| `Scale` | `Vector3` | Rest scale, relative to the parent |
| `Rotation` | `Quaternion` | Rest rotation, relative to the parent |
| `Inverse` | `Matrix4x3` | Inverse bind matrix, bind pose to bone space |

The table is copied in one block, so it includes the struct's padding. A reader must use the engine's struct
definition. Parents always come before their children.

---

## 7. Animation (`.anm`)

| Field | Type | Meaning |
|---|---|---|
| Magic | `UInt32` | `ZANM` |
| Revision | `UInt16` | `1` |
| Duration | `Real64` | Clip length, seconds |
| Extent | `Sphere` (`Vector3` centre, `Real32` radius) | Holds the mesh in every pose of the clip |
| Lanes | `UInt32` | Lane count, one per bone |
| Lanes | lane records | See below |

Each lane:

| Field | Type | Meaning |
|---|---|---|
| Name | `UInt64` | Bone name hash, matching `Bone::Name` |
| Rate | `Real32` | Samples per second |
| Count | `UInt32` | Samples |
| Mask | `UInt8` | Components driven: `1` position, `2` scale, `4` rotation. Studio always writes `7` |
| Fixed | `UInt8` | Components held still, same bits. A fixed component is stored once |
| Interpolation | `UInt8` | `Interpolation` |
| Position track | `Vector3[keys]` | `keys` = 1 if fixed, else `Count` |
| Scale track | `Vector3[keys]` | |
| Rotation track | `Quaternion[keys]` | |

With `Hermite` and a component that is not fixed, its values are followed by two more arrays of the same length:
incoming tangents, then outgoing tangents (Studio writes the same slope for both). Tangents are per second.

The loader refuses a lane with rate 0 or no samples, a bone repeated by two lanes, or a truncated track.

---

## 8. Model manifest (`.mdl`)

JSON, four-space indented. Full description in `models.md` section 9.

```json
{
    "Mesh": "Wolf.msh",
    "Skeleton": "Wolf.skl",
    "Materials": [ { "Images": { "Albedo": { "Path": "Wolf_Albedo.tex" } } } ],
    "Clips": [ { "Name": "Run", "Path": "Wolf.Run.anm", "Repeat": "Loop" } ]
}
```

| Key | Type | Required | Meaning |
|---|---|---|---|
| `Mesh` | string | yes | The `.msh`. A manifest without it is refused |
| `Skeleton` | string | no | The `.skl`. Absent means static geometry |
| `Materials` | array of objects or strings | no | Inline materials, or `.mtl` paths, in slot order |
| `Clips[].Name` | string | yes per clip | The name a game asks for, hashed for lookup |
| `Clips[].Path` | string | yes per clip | The `.anm` |
| `Clips[].Repeat` | `Once`, `Loop`, `Mirror` | no, `Loop` | How the clip repeats |
| `Clips[].Rate` | number | no, `1` | Playback speed factor, never below 0 (never written by Studio) |

---

## 9. Material (`.mtl`)

JSON with optional `Images`, `Samplers` and `Parameters` sections. Full description in `materials.md`.

```json
{
    "Images":     { "Albedo": { "Path": "Brick.tex" } },
    "Samplers":   { "Albedo": { "Filter": "LinearMipLinear", "AddressModeU": "Repeat", "AddressModeV": "Repeat", "AddressModeW": "Clamp" } },
    "Parameters": { "Tint": { "Type": "Color", "Value": [1, 1, 1, 1] } }
}
```

---

## 10. Tracker (`.json`)

JSON, four-space indented, `"Version": 1`. Full description in `trackers.md` section 5.

```json
{
    "Version": 1,
    "Texture": "Icons.tex",
    "Layout": "Texture2DArray",
    "Format": "RGBA8UIntNorm_sRGB",
    "Width": 64,
    "Height": 64,
    "Slices": 4,
    "Padding": 0,
    "Extrude": 0,
    "Mipmaps": false,
    "Linear": false,
    "Compress": true,
    "Mode": "Array",
    "Headroom": 4,
    "Regions": [
        { "Name": "sword",  "Source": "../Art/Icons/sword.png",  "Slice": 0, "Rect": [0, 0, 64, 64], "Crop": [0, 0, 1, 1] },
        { "Name": "shield", "Source": "../Art/Icons/shield.png", "Slice": 1, "Rect": [0, 0, 64, 64], "Crop": [0, 0, 1, 1] }
    ]
}
```

---

## 11. Checking a file by hand

The first six bytes identify any binary file:

```sh
$ xxd -l 15 -g 1 Content/Crate.tex
00000000: 5a 54 45 58 01 00 01 38 00 02 00 02 01 00 0a
```

Reading that: `ZTEX`, revision `1`, layout `1` (Texture2DArray), format `0x38` (56, which is `RGBA8UIntNorm_sRGB`
in the current enum order), width `0x0200` (512), height 512, 1 layer, 10 levels. The next 8 bytes are the payload
block's expanded size and stored length.

To look at a texture's pixels, `Studio extract` writes it back as PNG (plain formats only).

---

## 12. Where it lives

| Path | Holds |
|---|---|
| `Library/Texture/Source/Texture/Export/Exporter.cpp` | `.tex` writing |
| `Library/Texture/Source/Texture/Import/Importer.cpp` | `.tex` reading (`ImportTEX`), with its checks |
| `Library/Font/Source/Font/Export/Exporter.cpp` | `.fnt` writing |
| `Library/Sound/Source/Sound/Export/Exporter.cpp` | `.snd` writing, ADPCM and Opus payloads |
| `Library/Model/Source/Model/Export/Exporter.cpp` | `.msh`, `.skl`, `.anm`, `.mdl` writing |
| `Library/Texture/Source/Texture/Atlas/Tracker.cpp` | Tracker JSON |
| `Library/Material/Source/Material/Document.cpp` | `.mtl` JSON |
| Engine `Engine/Source/Zyphryon.Base/Marshal/Binary/Writer.hpp`, `Reader.hpp` | Blocks, compressed blocks |
| Engine `Engine/Source/Zyphryon.Content/Loader.cpp` | `Expect`, the magic and revision check |
| Engine `Engine/Source/Zyphryon.Graphic/Loader/`, `Zyphryon.Render/Model/Loader/`, `Zyphryon.Render/Font/Loader/`, `Zyphryon.Audio/Loader/` | The loaders |
