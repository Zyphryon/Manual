# Materials

This chapter covers the material JSON the engine reads: as a standalone `.mtl` file, or inline in a model manifest.
Studio writes materials in two places. The model bake writes one per FBX material inside the `.mdl`. The
`Studio_Material` library (`Studio::Material::Document`) reads and writes `.mtl` files for a tool or editor. The
command line has no material command; write `.mtl` files by hand or through the library.

---

## 1. What a material is

A material is a set of named bindings a technique (the shader) reads:

- **Images**: textures, each under the name the technique declares its texture by.
- **Samplers**: how each texture is read (filter, address modes), keyed by the same name.
- **Parameters**: constants (colours, factors, vectors), each under the name the technique declares.

A material does not name its technique. Whatever draws the model picks the technique. Names that the technique
does not declare are carried but never read.

The engine's `ZyGraphic::MTLLoader::Parse` reads the three sections. The same function reads a `.mtl` file and an
inline entry of a model manifest's `Materials` array.

---

## 2. The file

```json
{
    "Images": {
        "Albedo": { "Path": "Crate_Albedo.tex" },
        "Normal": { "Path": "Crate_Normal.tex" }
    },
    "Samplers": {
        "Albedo": { "Filter": "Anisotropic8x", "AddressModeU": "Repeat", "AddressModeV": "Repeat", "AddressModeW": "Clamp" },
        "Normal": { "Filter": "LinearMipLinear", "AddressModeU": "Repeat", "AddressModeV": "Repeat", "AddressModeW": "Clamp" }
    },
    "Parameters": {
        "Albedo":    { "Type": "Color", "Value": [1.0, 0.95, 0.9, 1.0] },
        "Roughness": { "Type": "Float", "Value": 0.7 },
        "Tiling":    { "Type": "Float2", "Value": [2.0, 2.0] },
        "Wet":       { "Type": "Bool", "Value": false }
    }
}
```

Each section is optional. A missing section is empty.

### 2.1 `Images`

| Key | Type | Meaning |
|---|---|---|
| `<Name>.Path` | string | The texture to load. Paths are resolved by the engine's content service against the material's own location (or a mounted url) |

### 2.2 `Samplers`

| Key | Values | `.mtl` default |
|---|---|---|
| `Filter` | `Point`, `PointMipPoint`, `PointMipLinear`, `Linear`, `LinearMipPoint`, `LinearMipLinear`, `Anisotropic2x`, `Anisotropic4x`, `Anisotropic8x`, `Anisotropic16x` | `Point` |
| `AddressModeU`, `AddressModeV`, `AddressModeW` | `Clamp`, `Border`, `Repeat`, `Mirror` | (see note) |
| `Comparison` | `None`, `Always`, `Never`, `Greater`, `GreaterEqual`, `Equal`, `NotEqual`, `Less`, `LessEqual` | `None` |
| `Border` | `OpaqueBlack`, `OpaqueWhite`, `TransparentBlack` | `OpaqueBlack` |

The engine's loader starts each sampler from its defaults with the filter set to `Point`. `Studio::Material::Document`
reads a missing address mode as `Clamp`. Write every key you care about. Note the default filter: a sampler entry
without `Filter` reads its texture **point-sampled**, mips ignored.

A material with no sampler entry for a texture keeps the technique's own sampler.

### 2.3 `Parameters`

Each parameter is `{ "Type": <Uniform>, "Value": ... }`.

| `Type` | `Value` |
|---|---|
| `Bool` | `true` / `false` |
| `Color` | `[r, g, b, a]` floats, or a hexadecimal string (`"FF8800FF"`) |
| `IntColor8` | `[r, g, b, a]` bytes, or a hexadecimal string |
| `Float` | number |
| `Float2`, `Float3`, `Float4` | array of 2, 3 or 4 numbers |
| `Int`, `UInt` | whole number |
| `Int2`..`Int4`, `UInt2`..`UInt4` | array of 2 to 4 whole numbers |

A missing `Type` reads as `Float`.

---

## 3. Materials inside a model manifest

The model bake (`models.md`) writes one material per FBX material into the `.mdl`'s `Materials` array. Each entry:

- **`Images`** holds every slot the FBX binds a texture to, under the slot name: `Albedo`, `Normal`, `Roughness`,
  `Metallic`, `Specular`, `Emissive`, `Displacement`, `Occlusion`, `Opacity`. Each path is the source image's file
  name with `.tex`, beside the manifest. `Images` is always written, possibly empty.
- **`Samplers`** holds one sampler per image, always `Repeat` on U, V and W and `LinearMipLinear` filtering.
- **`Parameters`** holds the constants the FBX states for `Albedo`, `Specular`, `Emissive` (as `Color`) and
  `Roughness`, `Metallic`, `Opacity` (as `Float`, the red component). `Normal`, `Displacement` and `Occlusion` never
  get a constant. The emissive colour is multiplied by the FBX's emission factor.

```json
"Materials": [
    {
        "Images": {
            "Albedo":    { "Path": "Barrel_Albedo.tex" },
            "Normal":    { "Path": "Barrel_Normal.tex" },
            "Roughness": { "Path": "Barrel_Roughness.tex" }
        },
        "Samplers": {
            "Albedo":    { "AddressModeU": "Repeat", "AddressModeV": "Repeat", "AddressModeW": "Repeat", "Filter": "LinearMipLinear" },
            "Normal":    { "AddressModeU": "Repeat", "AddressModeV": "Repeat", "AddressModeW": "Repeat", "Filter": "LinearMipLinear" },
            "Roughness": { "AddressModeU": "Repeat", "AddressModeV": "Repeat", "AddressModeW": "Repeat", "Filter": "LinearMipLinear" }
        },
        "Parameters": {
            "Albedo":    { "Type": "Color", "Value": [0.8, 0.8, 0.8, 1] },
            "Roughness": { "Type": "Float", "Value": 1 },
            "Metallic":  { "Type": "Float", "Value": 0 }
        }
    }
]
```

### 3.1 Matching your technique

The slot names are fixed. If your technique declares other names, or reads packed maps, edit the manifest after each
bake (or keep a shared `.mtl`, see 3.2):

- Rename keys to the technique's names.
- Replace separate `Roughness`, `Metallic` and `Occlusion` images with one packed texture (made with `pack` and
  channel splicing, `trackers.md` section 6) under the name the technique reads.
- Drop entries the technique never reads.

A re-bake writes the manifest again and loses hand edits. Script the edit, or point the manifest at a `.mtl`.

### 3.2 A manifest entry may be a path

`MDLLoader` accepts a string in place of an inline object. The string is loaded as a `.mtl`:

```json
"Materials": [ "Materials/Barrel.mtl", "Materials/Iron.mtl" ]
```

Then hand edits live in the `.mtl` files. The order still follows the material slots the primitives index, which is
the FBX's material order.

---

## 4. `Studio::Material::Document`

`Library/Material/Source/Material/Document.hpp`, linked as `Studio_Material`.

```cpp
struct Document final
{
    static constexpr Text kOutput = "mtl";

    struct Binding final
    {
        Str                       Name;
        Str                       Path;
        Bool                      Sampled    = false;
        ZyGraphic::TextureFilter  Filter     = ZyGraphic::TextureFilter::Point;
        ZyGraphic::TextureAddress AddressU   = ZyGraphic::TextureAddress::Clamp;
        ZyGraphic::TextureAddress AddressV   = ZyGraphic::TextureAddress::Clamp;
        ZyGraphic::TextureAddress AddressW   = ZyGraphic::TextureAddress::Clamp;
        ZyGraphic::TestCondition  Comparison = ZyGraphic::TestCondition::None;
        ZyGraphic::TextureBorder  Border     = ZyGraphic::TextureBorder::OpaqueBlack;
    };

    struct Constant final
    {
        Str                Name;
        ZyGraphic::Uniform Type  = ZyGraphic::Uniform::Float;
        Array<Real64, 4>   Value = Array<Real64, 4>(0.0, 0.0, 0.0, 0.0);
    };

    Sequence<Binding>  Bindings;
    Sequence<Constant> Constants;

    Ref<Binding>  Bind(Text Name);
    Ref<Constant> Set(Text Name);
    Bool          Remove(Text Name);

    static Bool Read(Text Path, Ref<Document> Output);
    Bool        Write(Text Path) const;
};
```

| Member | Behaviour |
|---|---|
| `Bind(Name)` | The binding under `Name`, added empty if missing. The reference stays valid until another binding is added or removed |
| `Set(Name)` | The constant under `Name`, added as zero if missing |
| `Remove(Name)` | Removes the binding and the constant under `Name`. `true` if anything was removed |
| `Read(Path, Output)` | Reads a `.mtl`. A colour given as a hexadecimal string is read into components (bytes for `IntColor8`, fractions otherwise) |
| `Write(Path)` | Writes the layout the engine reads, creating folders |

What `Write` emits:

- `Images` lists bindings with a non-empty `Path`.
- `Samplers` lists bindings with `Sampled` set, always with `Filter` and the three address modes, `Comparison` and
  `Border` only when not their defaults.
- `Parameters` writes each constant's `Type` by name and its `Value`: a bool for `Bool`, a number for one component,
  an array otherwise; whole numbers for integer types.
- Sections with nothing in them are left out. The file is four-space indented.

A sampler can exist without a texture (a binding with `Sampled` set and an empty `Path`), to override how a texture
the technique already binds is read.

### 4.1 Example

```cpp
#include <Material/Document.hpp>

Studio::Material::Document Crate;

Ref<Studio::Material::Document::Binding> Albedo = Crate.Bind("Albedo");
Albedo.Path     = "Crate_Albedo.tex";
Albedo.Sampled  = true;
Albedo.Filter   = ZyGraphic::TextureFilter::Anisotropic8x;
Albedo.AddressU = ZyGraphic::TextureAddress::Repeat;
Albedo.AddressV = ZyGraphic::TextureAddress::Repeat;

Ref<Studio::Material::Document::Constant> Tint = Crate.Set("Tint");
Tint.Type  = ZyGraphic::Uniform::Color;
Tint.Value = Array<Real64, 4>(1.0, 0.9, 0.8, 1.0);

Crate.Write("Content/Props/Crate.mtl");
```

Round-tripping a file:

```cpp
Studio::Material::Document Loaded;
if (Studio::Material::Document::Read("Content/Props/Crate.mtl", Loaded))
{
    Loaded.Remove("Tint");
    Loaded.Set("Roughness").Value[0] = 0.6;
    Loaded.Write("Content/Props/Crate.mtl");
}
```

`Read` leaves `Output` untouched when the file cannot be read or is not a JSON object.

---

## 5. Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Texture looks blocky, mips ignored | A sampler entry with no `Filter` reads `Point` | Set `Filter` (`LinearMipLinear` or anisotropic) |
| A texture or value has no effect | Its name is not one the technique declares | Use the technique's names |
| Hand edits to a model's materials vanish | The model was baked again | Point the manifest at `.mtl` files, or script the edit |
| Colour parameter reads black | `Type` left out, so it was read as `Float` | Give `"Type": "Color"` |

---

## 6. Where it lives

| Path | Holds |
|---|---|
| `Library/Material/Source/Material/Document.hpp`, `Document.cpp` | The `.mtl` reader and writer |
| `Library/Model/Source/Model/Export/Exporter.cpp` | `ExportModel`, the inline materials of a manifest |
| `Library/Model/Source/Model/Scene.hpp` | `Scene::Slot`, the slot names |
| Engine `Engine/Source/Zyphryon.Graphic/Loader/MTLLoader.cpp` | What the engine reads |
| Engine `Engine/Source/Zyphryon.Graphic/Loader/Parser.cpp` | Sampler and parameter parsing |
| Engine `Engine/Source/Zyphryon.Render/Model/Loader/MDLLoader.cpp` | Inline or path materials in a manifest |
