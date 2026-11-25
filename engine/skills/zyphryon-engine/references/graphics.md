# Graphics and rendering

This chapter covers the two layers a program draws through: `Zyphryon.Graphic` (`ZyGraphic`: the graphic service,
its Direct3D 11 and OpenGL drivers, GPU resources, shaders, techniques, materials and their loaders) and
`Zyphryon.Render` (`ZyRender`: the frame graph, the encoder, cameras, models and animation, particles, fonts,
sprites and the engine's post effects). Read it to draw anything, to write a custom pass, or to add an effect. It
leans on `content.md` for `Resources://` / `Embedded://` mounts and asynchronous loading, on `runtime.md` for the
startup parameters and the kernel that initializes the service, on `math.md` for matrices and frustums, and on
`scene.md` for keeping what you draw in entities.

---

## 1. The layers

| Layer | Namespace | What it owns | Who uses it |
|---|---|---|---|
| Graphic service | `ZyGraphic` | GPU resources as small integer handles, a command journal, a GPU worker thread, the driver. | Everything that draws. |
| Drivers | modules | `D3D11` (Windows) and `OpenGL` (desktop GL, GLES, WebGL). Plug into the service by name. | The service only. |
| Assets | `ZyGraphic` | `Image`, `Mesh`, `Material`, `Shader`, `Technique` resources and their loaders. | The content service. |
| Render | `ZyRender` | `Blueprint` / `Graph` / `Pass` (frame graph), `Encoder` (binding by name), `Collector` / `Recorder` (sorted batches), `Camera`, `PostEffect`, `Occlusion`, `Model3D` / `Animator3D`, `Emitter` / `Swarm`, `Font`, `Sheet2D` / `Animator2D`, `Album` / `Bank`. | The application. |

`ZyRender::Service` is a small subsystem that only registers the render loaders (`.anm`, `.fnt`, `.mdl`, `.sh2`,
`.skl`) with the content service. Everything else in `ZyRender` is a plain class the application owns.

The runtime kernel creates the graphic, audio and render services only when the program opens a window. A headless
start (`Startup::SetHeadless(true)`, as a server makes) has none of them, so code that draws must check
`GetService<ZyGraphic::Service>()` before using it.

---

## 2. The graphic service

`ZyGraphic::Service` is an engine subsystem. The runtime kernel creates it and calls
`Initialize(Text Name, Ptr<void> Output, ConstRef<Configuration> Config)` with the window, after the application's
modules attached and before `OnInitialize`.

### 2.1 Threading

The service records every call into a **producer journal** on the calling thread and hands it to a **GPU worker
thread** on `Tick` (one flush per frame, `kMaxFrames` = 2 in flight). Consequences for an author:

- `Create*` returns an `Object` (a `UInt16` handle) at once; the GPU object appears when the worker runs the call.
- Per-frame data is written into **in-flight arenas**: `AllocateInFlightVertices<T>`, `AllocateInFlightIndices<T>`,
  `AllocateInFlightUniforms<T>`. Each returns a `Transient<T>` (given a count) or a `Stream` (given a span), valid
  for this frame only. Nothing has to be freed.
- `GetDescription()` returns the backend name, the `Tier`, the `ShaderLanguage`, capabilities and adapters.
- `Reset(Width, Height, Tearless)` resizes the swap chain.

### 2.2 Resources

| Call | Notes |
|---|---|
| `CreateBuffer(Storage, Usage, Capacity, Blob)` / `UpdateBuffer` / `DeleteBuffer` | `Storage`: `Immutable`, `Dynamic`, `Stream`. `Usage` bits: `Vertex`, `Index`, `Uniform`, `Sample`, `Target`. |
| `CreateTexture(Layout, Format, Storage, Usage, W, H, Layers, Levels, Blob)` / `UpdateTexture` / `DeleteTexture` | `Layout`: `Texture2D`, `Texture2DArray`, `TextureCube`, `Texture3D`. Shorter overloads exist for sampled and render-target textures. |
| `ObtainSampler(Sampler)` | Samplers are interned by descriptor; never deleted by hand. |
| `CreatePass(Colors, Depth)` / `DeletePass` | Up to `kMaxAttachments` (8) colour attachments. |
| `CreatePipeline(Program, Signature, States)` / `DeletePipeline` | Normally made for you by `Technique`. |
| `CreateMaterial()` / `DeleteMaterial` | A material slot; made for you by `Material::Upload`. |
| `Prepare(Pass, Name, Viewport, Colors, Depth, Stencil)` / `Commit()` | Open and submit a pass; `ZyRender::Graph` does this for you. |

Handle pools per kind (`Types.hpp`): buffers `kMaxBuffers` 8192, textures `kMaxTextures` 4096, materials
`kMaxMaterials` 8192, pipelines `kMaxPipelines` 512, passes `kMaxPasses` 64, samplers `kMaxSamplers` 256.

### 2.3 Draw commands

A `ZyGraphic::Command` is one draw: pipeline, up to 8 vertex streams, an index stream, 4 uniform streams, 16
samplers, 16 textures, scissor, stencil reference and an `Invocation` (`Count`, `Base`, `Offset`, `Instances`).
You rarely fill one by hand; `ZyRender::Encoder` builds it from names (section 9.3).

---

## 3. Drivers and tiers

| Driver | Name | Priority | Shader language |
|---|---|---|---|
| Direct3D 11 | `"D3D11"` | `High` | HLSL, compiled with `D3DCompiler_47.dll` |
| OpenGL | `"OpenGL"` | `Low` | GLSL, `#version 330 core` up to `460 core`, or `300 es` / `310 es` / `320 es` |

An empty driver name takes the highest priority registered, so Windows uses D3D11. The startup file or
`OnConfigure` picks one explicitly (see `runtime.md`):

```json
{
  "Graphic": { "driver": "OpenGL", "tearless": true }
}
```

If the driver fails, the service falls back to a null driver and logs
`Graphics: Failed to initialize driver, using default driver`. It then logs the backend, the tier and each GPU.

`ZyGraphic::Tier` is the feature level the device reaches:

| Tier | `ZY_TIER` | Hardware | D3D11 profile |
|---|---|---|---|
| `Level1` | 1 | D3D9, GL 2.1, GLES 2.0 | unsupported |
| `Level2` | 2 | D3D10, GL 3.3, GLES 3.0 / WebGL2 | `vs_4_0` / `ps_4_0` |
| `Level3` | 3 | D3D11, GL 4.3, GLES 3.1 | `vs_5_0` / `ps_5_0` |
| `Level4` | 4 | FL12_0, VK 1.1, Metal 2 | `vs_5_0` / `ps_5_0` |
| `Level5` | 5 | FL12_2, VK 1.3, Metal 3 | `vs_5_0` / `ps_5_0` |

Tier 2 (SM4 and WebGL2) has no compute, no storage buffers, no UAVs and no indirect draws. A program that must run
on the web or on older hardware writes tier-3 paths under `#if ZY_TIER >= 3` with a tier-2 fallback.

---

## 4. Content loaders

| Extension | Loader | Resource |
|---|---|---|
| `.tex` | `ZyGraphic::TEXLoader` | `Image` (header `ZTEX` v1: layout, format, width, height, layers, mip count, compressed payload) |
| `.msh` | `ZyGraphic::MSHLoader` | `Mesh` |
| `.mtl` | `ZyGraphic::MTLLoader` | `Material` |
| `.hlsl`, `.glsl` | `ZyGraphic::SHDLoader` | `Shader` |
| `.vfx` | `ZyGraphic::VFXLoader` | `Technique` (registered with the device's language and tier) |
| `.mdl` | `ZyRender::MDLLoader` | `Model3D` |
| `.skl` | `ZyRender::SKLLoader` | `Skeleton3D` |
| `.anm` | `ZyRender::ANMLoader` | `Animation3D` |
| `.fnt` | `ZyRender::FNTLoader` | `Font` |
| `.sh2` | `ZyRender::SH2Loader` | `Sheet2D` |

Every resource loads asynchronously; check `HasCompleted()` (or `HasFailed()`) before reading one. Paths inside a
file are relative to that file unless they carry a mount (`Resources://`, `Embedded://`). The `.tex`, `.msh`, `.mdl`,
`.skl`, `.anm`, `.fnt` and `.sh2` files are baked by offline tools; `.vfx`, `.mtl` and shaders are written by hand.

```cpp
Ref<ZyContent::Service> Content = * GetService<ZyContent::Service>();

const Retainer<ZyGraphic::Technique> Technique = Content.Load<ZyGraphic::Technique>("Resources://Technique/Tint.vfx");

if (Technique->HasCompleted())
{
    // safe to draw with
}
```

---

## 5. Shaders

A shader file holds **both stages**. The pipeline compiles it once with `VERTEX_SHADER` defined and once with
`FRAGMENT_SHADER`, entry point `main`. Includes are resolved by the loader, by mount path:

```hlsl
#include "Embedded://Shader/Vertex.hlsl"
#include "Resources://Technique/Common/Lighting.hlsl"
```

Binding conventions, shared by every engine shader:

| Slot | HLSL | GLSL | Filled by |
|---|---|---|---|
| Frame block | `cbuffer cb_Global : register(b0)` | `layout(std140, binding = 0) uniform ...` | `Graph::Run(Block)` |
| Pass block | `cbuffer cb_Pass : register(b1)` | `binding = 1` | `Encoder::SetPass` |
| Material block | `cbuffer cb_Material : register(b2)` | `binding = 2` | `Encoder` from the material and schema |
| Instance block | `cbuffer cb_Instance : register(b3)` | `binding = 3` | `Binder::Draw(Instances, Uniform, ...)` |
| Textures | `t0`, `t1`, ... in `Signature.Textures` order | `layout(binding = N) uniform sampler2D ...` | material image, pass input, or fallback |
| Samplers | `s0`, `s1`, ... in `Signature.Samplers` order | combined with the texture | material sampler or technique default |
| Vertex inputs | semantics `SLOT0`, `SLOT1`, ... | `layout(location = N) in` | `Layout.Attributes` locations |

The engine library under `Embedded://Shader/` (HLSL and GLSL twins): `Affine`, `Antialias`, `Color`, `Cube`,
`Depth`, `Font`, `Grid`, `Math`, `Noise`, `Occlusion`, `Packing`, `Particle`, `PBR`, `Raymarch`, `Shadow`, `Skin`,
`Tonemap`, `Vertex`. `Math` refuses to compile without `ZY_TIER`. `Vertex` has `ZyEmitScreen(VertexID)`, the
fullscreen triangle every post pass uses: it returns clip-space `xy` and texture coordinates in `zw`.

Defines the loader adds: `ZY_TIER` (1..5) always; `ZY_DISPLAY_RAW` on web builds. A technique's `Program.Defines`
and each enabled feature's `Defines` follow.

---

## 6. Techniques (`.vfx`)

A technique is a pipeline description plus a **signature** naming everything a draw binds. `VFXLoader` reads:

```json
{
  "Properties": {
    "Blend":      { "Channel": "RGBA", "SrcColor": "One", "DstColor": "Zero", "EquationColor": "Add",
                    "SrcAlpha": "One", "DstAlpha": "Zero", "EquationAlpha": "Add" },
    "Depth":      { "Clip": true, "Mask": true, "Condition": "LessEqual", "Bias": 0.0, "BiasClamp": 0.0, "BiasSlope": 0.0 },
    "Rasterizer": { "Fill": "Solid", "Cull": "Back", "Scissor": false },
    "Stencil":    { "ReadMask": 255, "WriteMask": 255,
                    "FrontTest": "Always", "FrontFail": "Keep", "FrontDepthFail": "Keep", "FrontDepthPass": "Keep",
                    "BackTest": "Always", "BackFail": "Keep", "BackDepthFail": "Keep", "BackDepthPass": "Keep" },
    "Layout": {
      "Attributes": [ [0, "Float32x3", 0, 0, 0] ],
      "Primitive": "TriangleList"
    }
  },
  "Signature": {
    "Textures": [ { "Name": "Albedo", "Register": 0, "Fallback": [255, 255, 255, 255], "Layout": "Texture2DArray", "Format": "RGBA8UIntNorm" } ],
    "Samplers": [ { "Name": "Albedo", "Register": 0, "Filter": "LinearMipLinear", "AddressModeU": "Repeat" } ],
    "Uniforms": [ { "Frequency": "Material", "Name": "Tint", "Type": "Color", "Value": [1, 1, 1, 1], "Count": 1 } ]
  },
  "Program": {
    "Defines": [ "NAME", "NAME=VALUE" ],
    "Shaders": {
      "HLSL": [ { "Path": "Thing.hlsl", "Stage": "Vertex" }, { "Path": "Thing.hlsl", "Stage": "Fragment" } ],
      "GLSL": [ { "Path": "Thing.glsl", "Stage": "Vertex" }, { "Path": "Thing.glsl", "Stage": "Fragment" } ]
    }
  },
  "Features": [
    {
      "Name": "Cutout",
      "Enable": { "Parameter": "Cutout" },
      "Properties": { "Rasterizer": { "Cull": "None" } },
      "Program": { "Defines": [ "ENABLE_CUTOUT" ] }
    }
  ],
  "Preload": [ [ "Cutout" ] ]
}
```

The `Shaders` object is keyed by the device's language name (`HLSL` or `GLSL`), so one `.vfx` serves both drivers.

### 6.1 Properties

Every key is optional; defaults are those of `ZyGraphic::States`: cull `Back`, fill `Solid`, blend `One`/`Zero`
`Add`, write mask `RGBA`, depth clip on, depth write on, depth test `LessEqual`, stencil off, `TriangleList`.

- Blend factors: `Zero`, `One`, `SrcColor`, `OneMinusSrcColor`, `SrcAlpha`, `OneMinusSrcAlpha`, `DstColor`,
  `OneMinusDstColor`, `DstAlpha`, `OneMinusDstAlpha`. Equations: `Add`, `Subtract`, `ReverseSubtract`, `Min`, `Max`.
  `Channel`: any of `RGBA`, `RGB`, `RG`, `R`, `A`, ... `None`.
- Conditions: `None`, `Always`, `Never`, `Greater`, `GreaterEqual`, `Equal`, `NotEqual`, `Less`, `LessEqual`.
  Stencil actions: `Keep`, `Replace`, `Zero`, `Decrement`, `DecrementSaturate`, `Invert`, `Increment`,
  `IncrementSaturate`.
- `Cull`: `None`, `Back`, `Front`. `Fill`: `Solid`, `Wireframe`. `Primitive`: `PointList`, `LineList`,
  `LineStrip`, `TriangleList`, `TriangleStrip`.
- An attribute is `[Location, Format, Stream, Offset, Divisor]`. Divisor 0 is per vertex, 1 per instance. Formats
  are the `VertexFormat` names (`Float32x1..x4`, `Float16x2`, `Float16x4`, `UInt8x4`, `UIntNorm8x4`, `SInt16x2`,
  `UInt16x4`, `UIntNorm16x4`, `SInt32x2`, `UInt32x1`, `UIntNorm10_10_10_2`, ...). An empty list draws with no vertex
  input (a fullscreen pass using `SV_VertexID` / `gl_VertexID`).

### 6.2 Signature

- **Textures** take registers in order; `Register`, if written, must equal the index. `Fallback` is one texel of
  four bytes bound when nothing supplies the texture; its `Layout` must match what the shader declares and its
  `Format` is `RGBA8UIntNorm` (default) or `RGBA8UIntNorm_sRGB`. A texture with no fallback and nothing bound reads
  the pass input of that name.
- **Samplers** take registers in order, with the technique's default state (`Filter`, `AddressModeU/V/W`,
  `Border`, `Comparison`). A material sampler of the same name replaces it.
- **Uniforms**: `Frequency` is `Frame`, `Pass`, `Material` or `Instance` (default `Frame`); `Type` is a `Uniform`
  name (`Bool`, `Color`, `IntColor8`, `Float`..`Float4`, `Int`..`Int4`, `UInt`..`UInt4`); `Value` is the default;
  `Count > 1` declares `Name[0]`, `Name[1]`, ... with zero defaults.
- The C++ only packs the **material** block from the schema. Frame, pass and instance blocks are structs the C++
  fills; their schema entries are documentation at most.

**Packing rule.** The schema packs material fields **tightly, in declaration order, with no padding**. The shader's
`cb_Material` (HLSL packing) or `std140` block (GLSL) must land each field at the same offset. Put `Color` /
`Float4` fields first, follow a `Float3` with exactly one scalar, keep `Float2` at 8-byte offsets, and declare the
fields in the block in the same order as the `.vfx`.

### 6.3 Features and variants

A feature is a named patch: extra defines plus whole replaced state blocks (`Blend`, `Depth`, `Rasterizer`). A
variant key is a bitmask of features, so a technique may have up to 32.

A feature turns on:

- by **material**: `Enable.Texture` (the material binds that image) or `Enable.Parameter` (the material has that
  parameter at all, whatever its value);
- by **caller**: `Binder::SetVariant(Text Name)` or a key from `Technique::ResolveByName(Name)`;
- by **binding**: `Binder::SetImage` on a texture a feature is gated on (`Technique::ResolveByTexture`).

`Preload` lists the variants compiled at upload. Any other variant compiles on the first draw that asks for it, a
hitch the first time it shows; preload the ones you use. A preload naming an unknown feature logs
`'<key>' preloads unknown feature '<name>'`. A feature may not patch `Signature` or replace `Shaders` (the loader
logs a warning): every variant shares them.

---

## 7. Materials (`.mtl`)

A material is the per-surface data a technique reads: images by name, sampler overrides by name, and parameters.
It does not name a technique; the draw picks the technique and applies the material to it.

```json
{
  "Images": {
    "Albedo": { "Path": "Textures/Rock_Albedo.tex" },
    "Normal": { "Path": "Textures/Rock_Normal.tex" }
  },
  "Samplers": {
    "Albedo": { "Filter": "LinearMipLinear", "AddressModeU": "Repeat", "AddressModeV": "Repeat" }
  },
  "Parameters": {
    "Tint":   { "Type": "Color", "Value": [0.9, 0.85, 0.8, 1.0] },
    "Cutout": { "Type": "Float", "Value": 0.5 }
  }
}
```

- Each key in `Images`, `Samplers` and `Parameters` is hashed and matched against the technique's signature names.
- A `Color` value is an array of four numbers or a hexadecimal string.
- A sampler written with no `Filter` uses `Point`, not the technique's filter.
- `Parameter` `Type` defaults to `Float`.

In C++, `ZyGraphic::Material` holds the same data:

```cpp
void                 SetImage(UInt64 Name, ConstRetainer<Image> Image);
void                 SetSampler(UInt64 Name, Sampler Descriptor);
template<typename T> void SetParameter(UInt64 Key, AnyRef<T> Value);
void                 UnsetParameter(UInt64 Key);
ConstPtr<Parameter>  GetParameter(UInt64 Key) const;
template<typename T> T Read(UInt64 Key, ConstRef<Schema> Defaults) const;   // material value, else the schema's
```

When a draw applies a material (`Binder::Apply(Material)`), the encoder:

1. binds each image under the texture of the same name, the pass's input of that name, or the fallback;
2. binds each sampler override, else the technique's sampler;
3. packs the material block: for each `Material` uniform in the schema, the material's parameter if its type
   matches exactly, the schema default if the material has none, **zero if the material has one of another type**;
4. turns on every feature the material enables (`Technique::Resolve`).

A run of draws with the same technique and material packs once.

---

## 8. Meshes, models, animation, particles, text, sprites

- `ZyGraphic::Mesh`: one shared vertex and index buffer (`Extended` property for 32-bit indices), split into
  `Primitive`s, each with a range and a material slot. Several detail levels may each serve views below a
  coverage (`GetDetail(Coverage)`, `GetDetailCount()`), with a separate list of **caster** primitives per level for
  shadow maps (`GetCasters(Level)`). Bounds via `GetExtent()`.
- `ZyRender::Model3D`: a mesh, an optional `Skeleton3D`, a material table, and `Clip`s (`Name` hash, `Label`,
  `Motion` = `Animation3D`, `Mode` = `Repeat::Once` / `Loop` / `Mirror`, `Rate`). `FindClip(Hash)`, `GetExtent()`.
- `ZyRender::Animator3D`: plays a clip of a model with cross-fades. `SetModel`, then
  `Play(Clip, Timestamp, Phase = 0, Fade = 0, Back = false, Length = 0)`. `Update(Time, Scratch)` poses and fills a
  palette of one `Matrix4x3` per bone (`GetPalette()`); `Sample` and `Compose` split that in two so a program can
  adjust the local pose in between; `Advance(Time)` moves the clock without posing; `HasEnded()` for play-once clips.
  Clips may be shared by models with the same bones.
- `ZyRender::Emitter` describes a source of particles and what they are born with; `ZyRender::Swarm` carries one
  emitter's particles forward on the CPU.
- `ZyRender::Font` holds glyph metrics, kerning and one material per atlas page (MSDF); `FontEffect` holds text
  effect parameters.
- `ZyRender::Sheet2D` is the layout of a sprite sheet (`.sh2`), `Animator2D` plays one of its clips, with
  `Skeleton2D`, `Animation2D` and `Motion2D` for 2D rigs.

---

## 9. The frame graph

### 9.1 Blueprint and graph

A `ZyRender::Blueprint` declares **targets** and **passes in order**. A `ZyRender::Graph` realizes one blueprint
at one output size: a texture per target, a GPU pass per pass. Several views (a main view and a minimap) can share
one blueprint with a graph each.

```cpp
// Blueprint
UInt32     AddTarget(ConstRef<Target> Description);                    // returns the slot
void       ResizeTarget(UInt32 Slot, Target::Scale Sizing);
void       ResizeTarget(UInt32 Slot, UInt16 Width, UInt16 Height);    // for Scale::Fixed
template<typename Type, typename... Arguments>
Ref<Type>  AddPass(Text Name, AnyRef<Arguments>... Parameters);        // constructs Type(Parameters...)
template<typename Type = Pass>
Ref<Type>  GetPass(UInt32 Index);
```

`Target` is `{ Format, Sizing = Scale::Full, Width, Height }`. Scales: `Full`, `Half`, `Quarter`, `Eighth`,
`Sixteenth`, `ThirtySecond`, `Fixed`. Depth targets use a depth format (`D32Float`, `D24S8UIntNorm`).

```cpp
// Graph
Graph(Ref<ZyEngine::Subsystem::Host> Host, Ref<Blueprint> Blueprint);
void              Resize(UInt16 Width, UInt16 Height);
void              Run(ZyGraphic::Stream Frame);
template<typename Type> void Run(ConstRef<Type> Block);               // packs Block as the frame block (b0)
ZyGraphic::Object GetTexture(UInt32 Slot) const;
UInt16            GetWidth() const, GetHeight() const;                 // and per slot
```

`Run` realizes whatever changed since last time, then runs every **active** pass in order. The blueprint must
outlive its graphs.

### 9.2 Passes

Derive from `ZyRender::Pass` and implement `Run`:

```cpp
virtual void Run(Ref<ZyRender::Encoder> Encoder, ConstRef<ZyRender::Graph> Graph) = 0;
```

Before the graph is run, declare what the pass writes and reads:

| Call | Struct fields (defaults) |
|---|---|
| `AddColor(ColorAttachment)` | `Target` (`kNone` = display), `Load` (`Clear`), `Store` (`Store`), `Tint` (clear colour, opaque black) |
| `SetDepth(DepthAttachment)` | `Target` (`kNone` = no depth), `DepthLoad`, `DepthStore`, `StencilLoad`, `StencilStore`, `Depth` (1.0), `Stencil` (0) |
| `AddInput(InputAttachment)` | `Name` (hash of a texture name in the techniques), `Target` (slot) |
| `SetInput(Index, Target)` | Repoints an input at another slot. |
| `SetActive(Bool)` | Inactive passes are skipped. |

Actions (`ZyGraphic::Action`): `Load`, `Clear`, `Store`, `Discard`. Use `Load` to draw over what an earlier pass
wrote, `Discard` when the pass overwrites every pixel.

### 9.3 The encoder

Inside `Run`, draws are built by name:

```cpp
Encoder.SetPass(GpuMyLayout { ... });              // pass block (b1), packed into a transient buffer
Encoder.SetInput("Noise"_Hash, NoiseTexture);      // an extra input every draw reads when it binds no image

Encoder.Begin(* mTechnique)                         // a Binder: every declared texture starts at input or fallback
       .Apply(* Material)                           // images, samplers, material block, material features
       .SetImage("Mask"_Hash, MaskTexture)          // one image by name
       .SetVariant("Soft")                          // a caller feature
       .SetMesh(* Mesh)                             // vertex blocks and index buffer of a mesh
       .Draw(InstanceStream, Invocation);           // or Draw(Instances, UniformStream, Invocation), DrawFullscreen()
```

Other binder calls: `SetIndices(Stream)`, `SetStencil(Reference)`, `Apply(Material, Variant)`. On the encoder:
`SetScissor(Scissor)`, `SetFrame(Stream)`, `Reset()`. `"Name"_Hash` is the compile-time hash literal from
`Zyphryon.Base/Lexical/Text.hpp`.

`ZyRender::Collector` and `Recorder` sort many small draws by a 64-bit key (technique, material, depth) and write
them as instanced batches; use them for particles, text and anything drawn in the thousands. `Album` lends pages of
same-sized texture arrays one at a time, and `Bank` interns the uniform block a run of instances shares.

### 9.4 Camera

`ZyRender::Camera` holds a transform (`SetTransform`) and a projection matrix (`SetProjection`), and `Compute()`
derives view, view-projection, its inverse and a frustum (`GetFrustum()`), returning `true` when something changed.
`GetScreenCoordinates` / `GetWorldCoordinates` convert points with a viewport.

```cpp
ZyRender::Camera Camera;
Camera.SetProjection(Matrix4x4::CreatePerspective(Angle::FromDegrees(60.0f), Width / Height, 0.1f, 500.0f));
Camera.SetTransform(Transform);
Camera.Compute();

const Matrix4x4 ViewProjection = Camera.GetViewProjection();   // what a frame block usually carries
```

---

## 10. Post effects in the engine

`ZyRender::PostEffect(Host, Blueprint, Scene, Output)` appends a whole chain to a blueprint, reading the HDR `Scene`
slot and writing `Output` (`Pass::kNone` for the display). In order:

| Stage | Technique (`Embedded://Technique/PostEffect/`) | Runs when |
|---|---|---|
| `PostEffectTemporal` | `Temporal.vfx` | antialias is `Temporal`; it jitters the view (`GetJitter`) and blends history |
| `PostEffectBloom` | `Bloom.vfx` | `Bloom.Strength > 0`; down a chain of halves and back up |
| `PostEffectExposure` | `Exposure.vfx` | `Exposure.Auto`; meters the image and eases (`Measure`, `Adapt`, `Keep` stages) |
| `PostEffectFinal` | `Final.vfx` | always: exposure level, tone curve (`Neutral`, `ACES`, `AgX`), lift/gamma/gain, posterize, 3D LUT, vignette, grain, aberration, then `FXAA` or the temporal sharpen |

| Setter | Value |
|---|---|
| `SetBloom(Bloom { Strength = 0, Threshold = 1 })` | Glow the brightest of the image spills. |
| `SetBloomScale(Target::Scale)` | Size of the first bloom level. |
| `SetExposure(Exposure { Auto = false, Level = 1, Minimum = 0.25, Maximum = 4, Curve = ACES })` | Light the image is given and the tone curve. |
| `SetGrade(Grade { Saturation = 1, Posterize = 0, Table, Blend = 1 })` | Colour grade; `Table` is a look-up volume and `Blend` the share of it laid over the image. |
| `SetLens(Lens { Vignette = 0, Grain = 0, Aberration = 0, Cadence = 24 })` | What the glass leaves on the image. |
| `SetAntialias(Antialias::None / FXAA / Temporal)` | Edge smoothing. |
| `SetShift(Shift, Scale)` | Where the last image stood, for temporal reprojection. |
| `Advance(Time)` | Once a frame. |

`ZyRender::Occlusion(Host, Blueprint, Reach, Normal)` appends the horizon search and settle passes
(`Embedded://Technique/Occlusion/Horizon.vfx`, `Settle.vfx`), with `SetDetail(Off | Basic | Reduced | Standard)`,
`SetShading({ Strength = 1, Radius = 2 })`, `SetView(InverseViewProjection)`, `SetGlow(LastImage)`, and outputs
`GetOutput()` (open-sky direction and share) and `GetBounce()` (light sent back).

---

## 11. Worked example: a program with a custom pass

A complete program that clears an HDR target to a sky colour, then runs a custom fullscreen pass that tints it onto
the display. It is the smallest real frame graph: two targets' worth of passes, one technique, one pass block.

### 11.1 The technique and its shaders

`Content/Technique/Tint.vfx`:

```json
{
  "Properties": {
    "Depth":      { "Mask": false, "Condition": "Always" },
    "Rasterizer": { "Cull": "None" },
    "Layout":     { "Attributes": [], "Primitive": "TriangleList" }
  },
  "Signature": {
    "Textures": [ { "Name": "Scene" } ],
    "Samplers": [ { "Name": "Scene", "Filter": "Linear" } ]
  },
  "Program": {
    "Shaders": {
      "HLSL": [ { "Path": "Tint.hlsl", "Stage": "Vertex" }, { "Path": "Tint.hlsl", "Stage": "Fragment" } ],
      "GLSL": [ { "Path": "Tint.glsl", "Stage": "Vertex" }, { "Path": "Tint.glsl", "Stage": "Fragment" } ]
    }
  }
}
```

`Content/Technique/Tint.hlsl`:

```hlsl
#include "Embedded://Shader/Vertex.hlsl"

cbuffer cb_Pass : register(b1)
{
    float4 u_Tint;
};

struct fs_Input
{
    float4 Position : SV_POSITION;
    float2 Texture  : TEXCOORD0;
};

#ifdef VERTEX_SHADER

fs_Input main(uint VertexID : SV_VertexID)
{
    const float4 Screen = ZyEmitScreen(VertexID);

    fs_Input Result;
    Result.Position = float4(Screen.xy, 0.0, 1.0);
    Result.Texture  = Screen.zw;
    return Result;
}

#endif // VERTEX_SHADER

#ifdef FRAGMENT_SHADER

Texture2D    t_Scene : register(t0);
SamplerState s_Scene : register(s0);

float4 main(fs_Input Input) : SV_Target0
{
    const float3 Color = t_Scene.Sample(s_Scene, Input.Texture).rgb;
    return float4(lerp(Color, Color * u_Tint.rgb, u_Tint.a), 1.0);
}

#endif // FRAGMENT_SHADER
```

`Content/Technique/Tint.glsl`:

```glsl
#include "Embedded://Shader/Vertex.glsl"

layout(std140, binding = 1) uniform cb_Pass
{
    vec4 u_Tint;
};

#ifdef VERTEX_SHADER

out vec2 v_Texture;

void main()
{
    vec4 Screen = ZyEmitScreen(gl_VertexID);

    gl_Position = vec4(Screen.xy, 0.0, 1.0);
    v_Texture   = Screen.zw;
}

#endif // VERTEX_SHADER

#ifdef FRAGMENT_SHADER

layout(binding = 0) uniform sampler2D t_Scene;

in vec2 v_Texture;

layout(location = 0) out vec4 out_Color;

void main()
{
    vec3 Color = texture(t_Scene, v_Texture).rgb;
    out_Color  = vec4(mix(Color, Color * u_Tint.rgb, u_Tint.a), 1.0);
}

#endif // FRAGMENT_SHADER
```

### 11.2 The passes

```cpp
// Game/Passes.hpp
#pragma once

#include <Zyphryon.Content/Service.hpp>
#include <Zyphryon.Graphic/Technique.hpp>
#include <Zyphryon.Render/Encoder.hpp>
#include <Zyphryon.Render/Graph.hpp>

namespace Game
{
    /// \brief Represents a pass that only clears its target, standing in for the scene a game would draw.
    class Sky final : public ZyRender::Pass
    {
    public:

        /// \copydoc ZyRender::Pass::Run
        void Run(Ref<ZyRender::Encoder> Encoder, ConstRef<ZyRender::Graph> Graph) override
        {
            // The clear declared on the attachment is the whole pass. A real scene pass draws here.
        }
    };

    /// \brief Represents a fullscreen pass that tints the scene image onto its output.
    class Tint final : public ZyRender::Pass
    {
    public:

        /// \brief Represents the pass block the shader reads at b1.
        struct GpuTintLayout final
        {
            /// The tint in RGB and how much of it is applied in A.
            Vector4 Tint;
        };

        /// \brief Constructs the pass, loading its technique.
        ///
        /// \param Host The host the content service is found in.
        explicit Tint(Ref<ZyEngine::Subsystem::Host> Host)
            : mTechnique { Host.GetService<ZyContent::Service>()->Load<ZyGraphic::Technique>(kTechnique) },
              mTint      { Vector4(1.0f, 0.85f, 0.7f, 0.5f) }
        {
        }

        /// \copydoc ZyRender::Pass::Run
        void Run(Ref<ZyRender::Encoder> Encoder, ConstRef<ZyRender::Graph> Graph) override
        {
            if (!mTechnique->HasCompleted())
            {
                return;
            }
            Encoder.SetPass(GpuTintLayout { mTint });
            Encoder.Begin(* mTechnique).DrawFullscreen();
        }

    private:

        /// The technique the pass draws with.
        static constexpr Text kTechnique = "Resources://Technique/Tint.vfx";

    private:

        Retainer<ZyGraphic::Technique> mTechnique;
        Vector4                        mTint;
    };
}
```

### 11.3 The program

```cpp
// Game/Viewer.cpp
#include "Game/Passes.hpp"
#include <Zyphryon.Platform/Service.hpp>
#include <Zyphryon.Runtime/Entry.hpp>

namespace Game
{
    /// \brief Represents the program, one window drawn by a two-pass graph.
    class Viewer final : public ZyRuntime::Kernel
    {
    protected:

        /// \copydoc ZyRuntime::Kernel::OnInitialize
        Bool OnInitialize() override
        {
            // Resources:// must be mounted before the technique loads (see content.md).

            const UInt32 Scene = mBlueprint.AddTarget({ .Format = ZyGraphic::TextureFormat::R11G11B10Float });

            Ref<Sky> Clear = mBlueprint.AddPass<Sky>("Sky");
            Clear.AddColor({ .Target = Scene, .Tint = Color(0.35f, 0.55f, 0.85f, 1.0f) });

            Ref<Tint> Tinted = mBlueprint.AddPass<Tint>("Tint", static_cast<Ref<ZyEngine::Subsystem::Host>>(* this));
            Tinted.AddColor({ .Target = ZyRender::Pass::kNone, .Load = ZyGraphic::Action::Discard });
            Tinted.AddInput({ .Name = "Scene"_Hash, .Target = Scene });

            Ref<ZyPlatform::Window> Window = GetService<ZyPlatform::Service>()->GetWindow();

            mGraph = Unique<ZyRender::Graph>::Create(* this, mBlueprint);
            mGraph->Resize(Window.GetWidth(), Window.GetHeight());
            return true;
        }

        /// \copydoc ZyRuntime::Kernel::OnTick
        void OnTick(Real64 Delta) override
        {
            // No technique here reads a frame block, so the graph runs with an empty one.
            mGraph->Run(ZyGraphic::Stream());
        }

    private:

        ZyRender::Blueprint      mBlueprint;
        Unique<ZyRender::Graph>  mGraph;
    };
}

ZY_APPLICATION(Game::Viewer)
```

What happens each frame: `OnTick` runs before the services tick. `Graph::Run` opens the `Sky` pass (its colour
attachment clears `Scene`), then the `Tint` pass, whose technique reads `Scene` as the pass input named `Scene` and
draws one triangle onto the display. The graphic service's own tick then hands the frame's journal to the GPU
thread. Resize the graph when the window changes size (`Graph::Resize`); targets scaled `Full`, `Half`, ... follow.

To draw a real scene, give the first pass a depth target (`SetDepth({ .Target = Depth })` with a `D32Float`
target), draw meshes in its `Run` with `Encoder.Begin(Technique).Apply(Material).SetMesh(Mesh).Draw(...)`, and pass
a frame block with the camera's view-projection to `Graph::Run(Block)`.

To run the engine's own chain instead of a hand-written final pass, create
`ZyRender::PostEffect(* this, mBlueprint, Scene, ZyRender::Pass::kNone)` last and drive its setters, calling
`Advance(Time)` each frame.

### 11.4 Adding a knob or a variant to an existing technique

- **A material knob.** Append to `Signature.Uniforms`
  `{ "Frequency": "Material", "Name": "Sway", "Type": "Float", "Value": 0.0 }`, and in both shaders append
  `float u_Sway;` to `cb_Material` in the same position (the packing rule in section 6.2). A material that sets
  `"Sway": { "Type": "Float", "Value": 0.3 }` gets it; every other keeps the default.
- **A variant only some materials pay for.** Add a `Features` entry with `"Enable": { "Parameter": "Frost" }` and a
  define, wrap the extra shader code in `#ifdef ENABLE_FROST`, and add `[ "Frost" ]` to `Preload`.
- **A variant the caller picks.** A feature with no `Enable`, turned on per draw with `SetVariant("Frost")`.

Keep every change in both the `.hlsl` and the `.glsl`.

---

## 12. Common mistakes

| Mistake | What happens | Fix |
|---|---|---|
| Material block fields in a different order or alignment than the `.vfx` | Knobs read each other's values. | Same order; `Float4`/`Color` first; one scalar after a `Float3`. |
| A `Register` that is not the declaration index | Assert in debug builds. | Leave `Register` out or keep it dense. |
| Fallback `Layout` different from the shader's texture type | Wrong or black reads where no image is bound. | Match `Texture2D` / `Texture2DArray` / `TextureCube` / `Texture3D`. |
| Editing only the `.hlsl` | The OpenGL and web builds still run the old shader. | Change both files. |
| SM5-only intrinsics (`GatherRed`, depth-output semantics) outside `#if ZY_TIER >= 3` | Fails on tier 2 (D3D10, WebGL2). | Add a tier-2 path. |
| A variant not in `Preload` | A shader compile the first time it draws. | Preload the variants you use. |
| Reading a resource before `HasCompleted()` | Empty data. | Check, or wait for the content callback. |
| Keeping a `Transient` past the frame | Reads another frame's data. | Allocate in-flight data every frame. |
| A material parameter of another type than the schema's | The field packs as zero. | Match the `Type` exactly. |
| Using the graphic service in a headless build | It does not exist. | Check `GetService<ZyGraphic::Service>()` first. |
| A blueprint destroyed before its graph | The graph reads freed passes. | Keep the blueprint alive as long as every graph over it. |

## 13. Where it lives

- `Engine/Source/Zyphryon.Graphic/Service.hpp`, `Driver.hpp`, `Types.hpp` (enums, `States`, `Sampler`, `Command`, limits)
- `Engine/Source/Zyphryon.Graphic/Technique.hpp`, `Schema.hpp`, `Material.hpp`, `Mesh.hpp`, `Image.hpp`, `Shader.hpp`, `Transient.hpp`
- `Engine/Source/Zyphryon.Graphic/Loader/VFXLoader.cpp`, `MTLLoader.cpp`, `Parser.cpp`, `SHDLoader.cpp`, `TEXLoader.cpp`, `MSHLoader.cpp`
- `Engine/Source/Zyphryon.Render/Service.hpp`, `Blueprint.hpp`, `Graph.hpp`, `Pass.hpp`, `Target.hpp`, `Encoder.hpp`, `Collector.hpp`, `Recorder.hpp`, `Camera.hpp`, `Album.hpp`, `Bank.hpp`
- `Engine/Source/Zyphryon.Render/Effect/PostEffect.hpp`, `PostEffectFinal.hpp`, `PostEffectBloom.hpp`, `PostEffectExposure.hpp`, `PostEffectTemporal.hpp`, `Occlusion.hpp`, `OcclusionHorizon.hpp`, `OcclusionSettle.hpp`
- `Engine/Source/Zyphryon.Render/Model/Model3D.hpp`, `Animator3D.hpp`, `Skeleton3D.hpp`, `Animation3D.hpp`, `Loader/MDLLoader.cpp`
- `Engine/Source/Zyphryon.Render/Particle/Emitter.hpp`, `Swarm.hpp`; `Font/Font.hpp`, `FontEffect.hpp`; `Sprite/Sheet2D.hpp`, `Animator2D.hpp`
- `Engine/Resources/Shader/` (shared library) and `Engine/Resources/Technique/` (post effects, occlusion)
- `Module/Graphic/Driver/D3D11/Source/D3D11Compiler.hpp` (profiles), `D3D11Module.cpp`; `Module/Graphic/Driver/OGL/Source/OGLCompiler.cpp` (GLSL versions), `OGLModule.cpp`
- `Engine/Source/Zyphryon.Runtime/Kernel.cpp`, `Startup.cpp` (driver choice, headless start)
