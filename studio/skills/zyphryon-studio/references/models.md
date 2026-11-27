# Models

This chapter covers baking an FBX scene into the engine's model files: the `.mdl` manifest, the `.msh` mesh with its
levels of detail, the `.skl` skeleton and one `.anm` per clip. It explains what Studio does to the geometry, the
switches that steer it, and what the scene has to look like to come out right.

---

## 1. What a model bake writes

```sh
Studio bake Art/Knight/Knight.fbx Content/Knight/Knight.mdl --once Attack,Death
```

| File | Written | Holds | Engine type |
|---|---|---|---|
| `Knight.mdl` | always, last | JSON manifest naming everything else, with the materials inline | `ZyRender::Model3D` |
| `Knight.msh` | always | Vertex and index buffers, attribute bindings, levels of detail, bounds | `ZyGraphic::Mesh` |
| `Knight.skl` | when the scene is skinned | The bone table | `ZyRender::Skeleton3D` |
| `Knight.<Clip>.anm` | one per clip, when skinned | One clip's sampled motion | `ZyRender::Animation3D` |

Every file lands beside the manifest, named after the manifest's stem. The manifest refers to them by file name
only. The default destination is `<source stem>.mdl` **beside the source**, so name a destination, or an
`--output` folder, to bake into your content folder.

The manifest is written last, so a failed bake never leaves a manifest pointing at missing files. (It can leave an
older manifest pointing at newer files, if a later file fails to write.)

---

## 2. The pipeline

```
FBX ──ufbx──► Scene (triangles, materials, joints, clips) ──weld──► tangents ──► LOD reduce ──► shadow weld ──► order ──► pack ──► write
```

1. **Read.** ufbx loads the scene right-handed, Y-up, and mirrors a left-handed source on the way in. It converts
   units if `--units` is set, and generates missing normals if `--normals` is on. Embedded images are never read,
   and with `--no-rig` neither are the animation curves. Studio then mirrors Z itself (the engine's handedness),
   which also turns the source's counter-clockwise fronts into the engine's clockwise ones.
2. **Flatten.** Every mesh instance of every node is placed in world space and merged into one mesh. Faces are
   triangulated and grouped by material, one run per material the faces use.
3. **Rig and clips.** Bones are gathered from the skin, and every animation stack is resampled near `--rate`.
4. **Weld.** Identical corners of one material are merged into one vertex (meshoptimizer). A triangle that then
   repeats another corner for corner, or has two corners alike, is dropped (6.6).
5. **Tangents.** Generated after the weld when `--tangents` is on and `Tangent` is requested (5.5).
6. **Levels of detail.** Each reduced level is made from the one before over the same vertex buffer, and kept only
   when it passes two checks (6.4).
7. **Shadow weld.** Every level gets caster runs, in which corners a shadow cannot tell apart are merged (6.4).
8. **Order.** Each run's indices are ordered for the vertex cache, and its clusters are then drawn front to back so
   less of the model is shaded twice (6.6). The vertices are numbered by first use from the coarsest level up, so
   every coarser level reads a prefix of the vertex buffer.
9. **Pack.** Requested attributes are written into streams, indices as 16 or 32 bits.
10. **Write** the mesh, skeleton, clips and manifest.

A scene with no triangles fails: `Model: the scene carries no drawable geometry`. A scene whose every triangle the
weld drops fails: `Model: every triangle of the scene repeats another or has two corners alike`. A position that is
not a finite number fails: `the scene carries a position that is not a finite number`.

---

## 3. Switches

From `Application/Source/Usage.hpp` and `Model::Profile`:

| Switch | Profile field | Default | Meaning |
|---|---|---|---|
| `--scale <factor>` | `Scale` | `1` | Uniform factor every position and bone offset is scaled by, after any unit conversion. Must be above 0 |
| `--units <metres>` | `Units` | `0` | Metres one baked unit stands for. `0` keeps the source's units. Cannot be negative |
| `--rate <hertz>` | `Rate` | `30` | Samples per second a clip is keyed at, bent so its last sample is its last pose (8.2). Must be above 0 |
| `--interpolation <i>` | `Blend` | `Linear` | `Step`, `Linear`, `Cubic` or `Hermite` |
| `--attributes <list>` | `Attributes` | all but `TexCoord1`, one stream | Attributes to write and their streams, see 5 |
| `--normals` | `Normals` | on | Generate normals where the scene carries none |
| `--tangents` | `Tangents` | on | Generate the tangents normal maps are read through (5.5). `--no-tangents` writes none |
| `--rig` | `Rig` | on | Carry the skin and its clips across. `--no-rig` bakes a rigged source as a static prop |
| `--lod <n>` | `Details` | `3` | Reduced levels after the full mesh, `0` to `3` |
| `--coverage <list>` | `Details[].Coverage` | `0.03,0.015,0.0075` | View-height share each level is kept down to, by commas (6.3) |
| `--drift <pixels>` | `Drift` | `1` | Pixels a level may stray on a 1080-line view before a finer one is drawn. `0` keeps the coverages (6.3) |
| `--density <n>` | `Density` | `64` | Pixels one unit covers where the model is seen nearest, to warn of a mesh too dense. `0` for no warning (6.5) |
| `--once <names>` | `Once` | none | Clips that play once and hold, by baked (cleaned) name, by commas. All others loop |

Wrong values exit `2`: a switch that takes a number given anything else, an `--interpolation` that is not one of
the four (`'<x>' is not a value '--interpolation' takes`), a bad `--attributes`, and `--lod` above 3
(`'--lod' goes from 0 to 3, which <n> is not`). A non-positive `--scale` or `--rate`, or a negative `--units`, fails
the bake (exit `1`).

---

## 4. Units, axes and placement

### 4.1 Units

`--units` sets how many metres one baked unit stands for. `--units 1` converts the scene so one baked unit is one
metre, whatever unit it was authored in (ufbx reads the unit the FBX declares). The default `0` keeps the source's
units: a scene authored in centimetres bakes a hundred times larger than one in metres.

`--scale` multiplies on top, after the conversion. Use it to fix a scene whose declared units are wrong, or to size
an asset on purpose.

Both scale the vertices and every bone's offset from its parent, never a bone's own axes. A skinned mesh deforms the
same way at any scale, and a prop hung on a bone at runtime keeps its own size.

```sh
# A scene exported in centimetres, baked so one unit is one metre
Studio bake Crate.fbx Content/Crate.mdl --units 1

# The same, at half size
Studio bake Crate.fbx Content/Crate.mdl --units 1 --scale 0.5
```

### 4.2 Axes

The scene is read right-handed with Y up, then mirrored across Z. A source saved left-handed is mirrored by the
reader first, which winds its faces back as it goes, so its fronts still face out. Author with Y up, or let your
exporter convert. Texture V is flipped (FBX runs V up the image, the engine runs it down).

### 4.3 Flattening

Every node's meshes are placed where the scene puts them (their world transform) and merged into **one** mesh. The
node hierarchy is not kept. To place parts separately at runtime, export them as separate FBX files.

A mesh under a mirrored (negatively scaled) node has its triangles wound back, so its fronts face out like the
rest.

---

## 5. Vertex attributes and streams

### 5.1 Attributes Studio can write

| Name in `--attributes` | Engine slot | Stored as | Size | Read from |
|---|---|---|---|---|
| `Position` | `VertexSlot::Position` | `Float32x3` | 12 | Positions, scaled |
| `Normal` | `VertexSlot::Normal` | `Float32x3` | 12 | Normals, or generated |
| `Tangent` | `VertexSlot::Tangent` | `Float32x4` | 16 | Generated (5.5), tangent in XYZ, bitangent sign (-1 or +1) in W |
| `TexCoord0` | `VertexSlot::TexCoord0` | `Float32x2` | 8 | First UV set |
| `TexCoord1` | `VertexSlot::TexCoord1` | `Float32x2` | 8 | Second UV set |
| `Color0` | `VertexSlot::Color0` | `UIntNorm8x4` | 4 | Vertex colour, white where the scene has none |
| `BlendIndices` | `VertexSlot::BlendIndices` | `UInt16x4` | 8 | Up to four bone indices |
| `BlendWeights` | `VertexSlot::BlendWeights` | `UIntNorm16x4` | 8 | Their weights, summing to one |

### 5.2 Which are written

The default list is `Position, Normal, Tangent, TexCoord0, Color0, BlendIndices, BlendWeights`, all interleaved in
one stream. `TexCoord1` is written only when named in `--attributes`.

An attribute is written only when it is requested **and** the scene carries it:

- `Normal`: carried when any mesh has normals, or `--normals` generated them.
- `TexCoord0`: when any mesh has UVs. `TexCoord1`: when any mesh has a second UV set.
- `Tangent`: when `--tangents` is on and the scene carries normals and a first UV set. The source's own tangents
  are never read, so `--no-tangents` writes no tangent stream at all.
- `Color0`: **always**, white (`255,255,255,255`) where the source has no vertex colour. Drop it with
  `--attributes` if your shaders do not read it; it costs 4 bytes per vertex.
- `BlendIndices`, `BlendWeights`: when the scene is skinned and `--rig` is on.

### 5.3 The list syntax

A **bracketed run** shares one interleaved stream. A **bare name** is a stream of its own. Commas and spaces
separate entries.

```sh
# Everything in one stream (what the default does)
--attributes "[Position,Normal,Tangent,TexCoord0,Color0,BlendIndices,BlendWeights]"

# Positions alone (cheap for depth-only passes), the rest interleaved
--attributes "[Position],[Normal,Tangent,TexCoord0]"

# Three separate streams
--attributes "Position,Normal,TexCoord0"
```

Rules, each refused with exit `2`:

- `Position` must be named: `the attribute list has to name 'Position', since nothing draws without it`.
- A name may appear once: `'Normal' is named more than once`.
- Every name must be one of the eight in 5.1: `'Color1' is not an attribute this baker can write`.
- Brackets must open and close in turn.

Quote the list in a shell, since brackets and commas mean something to some shells.

All streams go into one vertex buffer, one stream after another. Each binding in the mesh records its slot, format,
stride (the size of its stream's record) and byte offset.

### 5.4 Indices

Indices are 16-bit when the baked mesh (welded, then split for its tangents) has 65535 vertices or fewer, 32-bit
otherwise (the mesh's `Extended` property).

### 5.5 Tangents

Tangents are always generated when they are written, the way MikkTSpace weighs them, since that is what normal maps
are baked against. They are built after the weld from the normals and the first UV set (`Tangent::Generate`):

- Each corner gets its own frame. A vertex whose corners disagree on the tangent or its sign is split into one
  vertex per frame, as on either side of a mirrored UV seam.
- A corner whose texture coordinates pin no direction takes any direction across its normal.
- The sign in W is turned over from what MikkTSpace gives, because the engine winds its fronts clockwise.

Tangents need normals and a first UV set, both requested in `--attributes`. A scene that lacks either bakes no
tangent stream. Bake normal maps against MikkTSpace tangents, which most tools use by default.

---

## 6. Levels of detail

### 6.1 The default table

`Model::Profile::StandardDetail()`:

| Level | Aims to keep of the triangles | Drawn down to (view-height share) | Allowed error (share of the mesh's extent) |
|---|---|---|---|
| 0 | 100% (the source) | 3% (`0.03`) | 0.02 (not reduced, so not read) |
| 1 | 25% | 1.5% (`0.015`) | 0.005 |
| 2 | 6% | 0.75% (`0.0075`) | 0.015 |
| 3 | 1.5% | 0 (any size) | 0.05 |

Each level is drawn while the model covers at least its share of the viewport's height. Below that the next level
takes over. The last level kept always has coverage 0, so it draws at any size. The table's coverages stand only
where `--drift` does not set them (6.3).

The ratio quarters at each step while the coverage halves, because triangle count follows screen area, which goes
with the square of the coverage. The ratio is a share of the source's triangles and only a target. The error limit
and the checks in 6.4 decide how far a level really goes. The extent is the widest side of the mesh's bounds.

### 6.2 `--lod`

`--lod <n>` asks for the full mesh plus `n` reduced levels, `0` to `3`. `--lod 0` writes the source geometry alone,
drawn at every size. A value above 3 logs `Model: '--lod' goes from 0 to 3, which <n> is not` and exits `2`.

A model can come out with fewer levels than it asked for. A level that fails its checks (6.4) is left out, and the
level before it stands in for it.

### 6.3 `--drift` and `--coverage`

By default each level's coverage comes from how far the next level strays from the source. With `--drift <pixels>`
(default `1`), a level is drawn until the next coarser one would stray by less than that many pixels of a 1080-line
view. When a level is kept, the coverage of the level before it becomes `drift × extent / (1080 × stray)`, where the
stray is the error of every reduction up to the new level, added up. A coarse level that strays little takes over
early. One that strays far is held off until the model is small on screen.

`--coverage a,b,c` sets the coverage of level 0, 1, 2 in order instead. Naming it turns the drift off, unless
`--drift` is typed as well, in which case the drift still sets every level that a kept level follows. `--drift 0`
keeps the table's coverages. Values past the levels kept are ignored, and the last level kept is forced to 0. A level
left out passes its coverage to the level before it.

```sh
# A hero model: hold full detail longer
Studio bake Hero.fbx Content/Hero.mdl --coverage 0.08,0.04,0.02

# Let each level stray by two pixels before a finer one is drawn
Studio bake Rock.fbx Content/Rock.mdl --drift 2

# A small prop: two levels only
Studio bake Pebble.fbx Content/Pebble.mdl --lod 1
```

### 6.4 How a level is made

The whole mesh is reduced at once, every material together (meshoptimizer's simplifier), and each level is reduced
from the level before it. A level therefore uses only vertices the level before it uses, and the errors add up. Each
level may stray by its own error limit, less what the levels before it strayed already.

- The reduction weighs positions and normals. A unit normal weighs half the mesh's extent. Texture coordinates and
  vertex colours are not weighed. A mesh with no normals is reduced by its positions alone.
- Hard edges do not stop it. It never collapses across a seam a triangle cannot straddle, which is where one
  material meets another, where either UV set splits, or where the tangent sign flips at a mirrored seam. It may
  still shorten such a seam along its length, so seams between materials do not crack and UV charts do not tear.
- Small loose parts may be dropped whole. A skinned mesh is kept regular, since a sliver bends badly between its
  bones.

A reduced level is kept only when it passes two checks:

1. It sheds at least a quarter of the triangles of the level before it (it keeps at most 75%, `kGain`).
2. At most 5% of the source's surface (`kLost`) lies further from it than its error limit. Every source vertex
   stands for a third of each face around it, and is measured to the nearest point of the level's triangles.

A level that sheds too little is left out at once, since less error would shed less still. A level that leaves too
much of the source out is reduced again at half the error, three tries in all (`kTries`). A level that still fails is
left out, and the level before it stands in for it down to its coverage. A level is left out untried when the level
before it already holds no more triangles than its target, or when the levels before it strayed as far as its error
limit allows.

So a model may have fewer levels than the profile asks for. A model made of small loose parts (a chain, a bush of
cards) keeps them, because a level that drops them leaves too much of the surface out.

Reduced levels share the full mesh's vertex buffer. Each adds its own index runs, one per material it still shows.

A level whose error held it above its target share also gets **caster** runs for shadow maps. They are a further
reduction of that level toward the same target, which weighs no normals and may stray three times as far
(`kLoose`). They pass the same two checks against that looser limit. A level that gets no such reduction casts the
triangles it draws.

What a level casts is then **welded for shadows**. Corners alike in position, first texture coordinates and skin
share one vertex there, whatever their normals, tangents and colours, so a shadow map runs fewer vertices on a
model with hard edges. A looser caster is welded in place. A level that casts what it draws gets a welded copy of
its runs, so every level ends up with caster runs and such a level holds its indices twice. A flat-shaded prop or
building runs a quarter to two thirds fewer shadow vertices this way. A smooth or skinned model gains little.

A caster run is therefore meant for a depth-only pass that reads position, the first texture coordinates (for a
cutout) and skin. Its other attributes are those of whichever corner the weld kept.

The bake log reports each level:

```
Model:     level 1 keeps 1043 triangles (24%) above 1% coverage, and casts 812
```

(Numbers illustrative; the coverage is printed as a whole percentage. A welded copy casts as many triangles as its
level draws, so it shows as the same number.)

### 6.5 Warnings

A mesh of more than 300 triangles that was asked for levels warns when its last level still keeps more than a
quarter of them, which includes a mesh that gained no level at all:

```
Model: its levels stall at 41200 of 52000 triangles, held by its seams and loose parts
```

`--density <n>` (default `64`) is the pixels one unit covers where the model is seen nearest. A mesh with more
triangles than the pixels the largest face of its bounding box covers at that density warns
`Model: its <t> triangles are more than the <p> pixels it covers at <n> a unit`. `--density 0` turns it off.

### 6.6 Triangles dropped and their order

After the weld, a triangle is dropped when it repeats another one corner for corner, wound the same way, or when two
of its corners are the same vertex. The first draws the same pixels twice and the second draws nothing. A copy wound
the other way is kept, since two-sided art is made of those. The log says how many went:

```
Model:     left out 9612 triangles that repeat another or have two corners alike
```

The triangle count in the log, the level percentages and both warnings count what is left. Two triangles at one
place that differ in any attribute are both kept, because which of them shows is the art's choice. A material whose
every triangle is dropped keeps its entry in the manifest and gets no run in the mesh.

The order of the triangles inside a run is the tool's own. Each run is ordered for the vertex cache, and its
clusters are then sorted so the outside of the model is drawn before what it hides (meshoptimizer's overdraw pass,
allowed to cost the vertex cache up to 5%, `kCache`). A material that blends and depends on the order its triangles
are drawn in needs a depth pass of its own first.

---

## 7. Rig

### 7.1 Which nodes become bones

Only with `--rig` (on by default), and only when the scene has a skin deformer:

- **Weighted bones**: every node a skin cluster binds with a weight above `1e-4`, measured where it was bound
  (its bind pose), since rigs are often saved in a control pose.
- **Sockets**: a bone node without weight whose parent is a weighted bone is kept too, placed where its parent
  puts it. Use these as attachment points (a hand socket for a weapon).
- Every other node (controls, helpers, meshes) is dropped. A bone's parent is its nearest ancestor that is itself a
  kept bone.
- Bones keep the scene's order, which lists parents first. The engine composes in that order.

Limits:

| Limit | Value |
|---|---|
| Bones | 32767 (`Scene::kMaxJoints`). More are dropped with a warning |
| Influences per vertex | 4 (`Scene::kInfluences`), the heaviest kept, renormalized to sum to one |
| Smallest weight kept | `1e-4` (`Scene::kMinWeight`) |
| Skins | The first skin binding of a mesh only |

In a rigged scene, a mesh no skin binds (a helmet or a sword parented to a bone) follows the nearest bone above it,
placed against that bone as it was bound. A mesh with no bone above it, and a skinned vertex no weight reaches,
follow the root bone. Weights are stored as 16-bit fractions, and any rounding left over goes to the heaviest weight,
so the four always sum to exactly one.

A bone whose children compensate its scale is sampled with that compensation, since ufbx folds it into the nodes it
reads. ufbx splits such a bone's scale onto a helper node, which then holds the weight and becomes the bone under
the bone's own name. A socket under such a bone is not kept, because its parent node holds no weight.

### 7.2 Bone names

Bones and clip lanes are matched by a 64-bit hash of the bone name (`Scene::Identify`). A bone with no name, or a
name another bone already holds, is renamed `<name>_<index>`. Keep bone names unique so clips baked from other files
of the same rig find their bones.

### 7.3 `--no-rig`

Bakes a rigged source as the static prop it rests as: no skeleton, no clips, no blend attributes. Its animation
curves are not even read.

---

## 8. Clips

### 8.1 What becomes a clip

Every animation stack (take) with a length above zero, **only when the scene has bones**. A scene with no skin
bakes no clips, even when its nodes are animated. Bake rigid animation with a skin bound to one bone.

### 8.2 Sampling

- Each clip is sampled at the cadence nearest `--rate` that divides its length into whole steps:
  `round(length * rate)` steps (at least one), so `steps + 1` samples, every bone at every sample. The last sample
  is the clip's last pose, so a looping clip shows no hitch at its seam. The `.anm` stores that cadence.
- Each sample evaluates the bones and their ancestors only.
- Per bone, a component that never changes (position within `1e-5` of the clip's bounding radius, scale within
  `1e-5`, rotation by quaternion dot) is stored once instead of once per sample.
- Every clip gets a bounding sphere that holds the mesh in every pose it takes, so a limb swung past the resting
  mesh is not culled.

### 8.3 Interpolation

| `--interpolation` | Runtime reads between samples | File size |
|---|---|---|
| `Step` | Holds the previous sample | 1x |
| `Linear` (default) | Straight line | 1x |
| `Cubic` | Catmull-Rom spline through the neighbouring samples, at no extra storage | 1x |
| `Hermite` | Hermite spline through the source's own slope at each sample | 3x (value plus two tangents per sample) |

`Cubic` is the cheap way to smooth a low `--rate`. `Hermite` follows the source curves most closely.

### 8.4 Clip names and files

A clip is named after its take, with every space and every character in `/ \ : | < > " ? *` turned into `_`. FBX
often qualifies takes as `Armature|Run`, which becomes `Armature_Run`. A take with no name becomes `Clip`. Two takes
that clean up to the same name: the later one gets `_<n>` appended.

The file is `<model stem>.<clip name>.anm`.

### 8.5 `--once`

`--once Attack,Death` marks clips that play once and hold their last pose. Every other clip loops. Names are matched
against the **cleaned** clip names, case-sensitively: write `--once Armature_Attack`, not `Armature|Attack`. A name
that matches no clip logs `'--once' names 'x', which no clip of the scene is called` and is otherwise ignored.

The manifest also accepts `"Repeat": "Mirror"` (ping-pong) and a per-clip `"Rate"` speed factor, which Studio never
writes. Edit the manifest to use them (see 9).

---

## 9. The manifest (`.mdl`)

Four-space indented JSON:

```json
{
    "Mesh": "Knight.msh",
    "Skeleton": "Knight.skl",
    "Materials": [
        {
            "Images": {
                "Albedo": { "Path": "Knight_Albedo.tex" },
                "Normal": { "Path": "Knight_Normal.tex" }
            },
            "Samplers": {
                "Albedo": { "AddressModeU": "Repeat", "AddressModeV": "Repeat", "AddressModeW": "Repeat", "Filter": "LinearMipLinear" },
                "Normal": { "AddressModeU": "Repeat", "AddressModeV": "Repeat", "AddressModeW": "Repeat", "Filter": "LinearMipLinear" }
            },
            "Parameters": {
                "Albedo":    { "Type": "Color", "Value": [1, 1, 1, 1] },
                "Roughness": { "Type": "Float", "Value": 0.5 },
                "Metallic":  { "Type": "Float", "Value": 0 }
            }
        }
    ],
    "Clips": [
        { "Name": "Idle",   "Path": "Knight.Idle.anm",   "Repeat": "Loop" },
        { "Name": "Attack", "Path": "Knight.Attack.anm", "Repeat": "Once" }
    ]
}
```

| Key | Written | Meaning |
|---|---|---|
| `Mesh` | always | The `.msh`, relative to the manifest |
| `Skeleton` | when skinned | The `.skl`. A manifest without one is static geometry |
| `Materials` | always | One entry per material the faces use, in the order primitives index them (10.4) |
| `Clips` | when there are clips | `Name`, `Path`, `Repeat` (`Loop` or `Once`) |

What the engine's `MDLLoader` reads besides: a `Materials` entry may be a **string**, the path of a `.mtl` file,
instead of an inline object; and a clip may carry `"Rate"`, a playback speed factor (default 1, never below 0).
`"Repeat"` takes `Once`, `Loop` or `Mirror`.

The materials are covered in `materials.md`.

---

## 10. Materials and textures

### 10.1 Slots

ufbx translates every shading model the FBX carries into one physically based set. Studio reads nine maps:

| Slot | From (ufbx PBR) | Constant written as |
|---|---|---|
| `Albedo` | base colour | `Color` |
| `Normal` | normal map | none |
| `Roughness` | roughness | `Float` |
| `Metallic` | metalness | `Float` |
| `Specular` | specular colour | `Color` |
| `Emissive` | emission colour, times the emission factor | `Color` |
| `Displacement` | displacement map | none |
| `Occlusion` | ambient occlusion | none |
| `Opacity` | opacity | `Float` |

For each slot:

- **An image** is written under `Images.<Slot>.Path` when the map has an enabled texture, with a sampler under
  `Samplers.<Slot>`: repeat on every axis, `LinearMipLinear` filtering.
- **A constant** is written under `Parameters.<Slot>` when the FBX states a value for it and the slot takes one
  (column three). A scalar is the red component.

### 10.2 Image paths

Each texture path the FBX records is cut to its file name and given the `.tex` extension (`Scene::Retarget`):
`C:/Work/Textures/Knight_Albedo.png` becomes `Knight_Albedo.tex`. The relative path the FBX stores is preferred over
the absolute one, but only the name survives either way.

So the texture bakes must:

1. Land **in the same folder as the manifest**.
2. Be named **after the source image's stem**.

```sh
Studio bake Art/Knight/Knight.fbx        Content/Knight/Knight.mdl
Studio bake Art/Knight/Knight_Albedo.png Content/Knight/Knight_Albedo.tex --preset sprite
Studio bake Art/Knight/Knight_Normal.png Content/Knight/Knight_Normal.tex --preset normal
```

The model bake does not bake the textures, and does not check that they exist.

### 10.3 Separate maps versus packed maps

Studio writes every image under its own slot name. A technique that samples `Roughness`, `Metallic` and `Occlusion`
as separate textures reads them as baked. A technique that expects them packed into one texture (for example R
roughness, G metalness, B occlusion) does not read the separate entries. In that case:

1. Pack the maps into one texture with `pack` and channel splicing (`trackers.md`, section 6).
2. Edit the manifest's material: remove the separate `Images`/`Samplers` entries, add one under the name the
   technique declares, and adjust `Parameters` to the technique's names.

The slot names are also the sampler and parameter names. They must match what your technique declares, or the
material binds nothing.

### 10.4 Limits

- Only materials that faces use are written, in the scene's order. A material no face uses is left out, so slot
  numbers close up.
- Faces that name no material share one empty entry, `{ "Images": {} }`, written last. A scene with no materials
  still draws, under that one entry.
- At most 255 materials (`Scene::kMaxMaterials`), counting only those faces use. Faces past them are dropped with
  `the faces use more than 255 materials and the <n> triangles past them are dropped`.

---

## 11. Library API

Headers under `Library/Model/Source/Model/`.

### 11.1 `Model::Profile`

```cpp
struct Profile final
{
    struct Channel final { ZyGraphic::VertexSlot Slot = ZyGraphic::VertexSlot::None; UInt8 Group = 0; };
    struct Detail  final { Real32 Ratio = 1.0f; Real32 Coverage = 0.0f; Real32 Error = 0.02f; };

    Real32            Scale      = 1.0f;
    Real32            Units      = 0.0f;
    Bool              Normals    = true;
    Bool              Tangents   = true;
    Bool              Rig        = true;
    Real32            Rate       = 30.0f;
    Interpolation     Blend      = Interpolation::Linear;
    Sequence<Str>     Once;
    Sequence<Channel> Attributes = Standard();
    Sequence<Detail>  Details    = StandardDetail();
    Real32            Drift      = 1.0f;
    Real32            Density    = 64.0f;

    Bool Wants(ZyGraphic::VertexSlot Slot) const;
    static Sequence<Channel> Standard();
    static Sequence<Detail>  StandardDetail();
    static Bool Parse(Text Description, Ref<Sequence<Channel>> Output);
    static Bool From(ConstRef<Environment> Environment, Ref<Profile> Output);
};
```

From code, `Details` can hold any table (not limited to four levels) as long as the mesh format's limit of 8 levels
(`ZyGraphic::Mesh::kMaxDetail`) holds and the last level has coverage 0. `Profile::From` sets `Drift` to 0 when
`--coverage` is given and `--drift` is not.

### 11.2 `Model::Baker`

```cpp
explicit Baker(Ref<ZyJob::Service> Scheduler);

Bundle Bake(ConstSpan<Byte> Source, Text Type, Text Name, ConstRef<Profile> Profile) const;
Bool   Bake(Text Source, Text Destination, ConstRef<Profile> Profile) const;

struct Bundle
{
    struct Motion { Str Name; Blob Data; };
    Blob             Mesh;       // empty on failure
    Blob             Skeleton;   // empty for a model that never deforms
    Sequence<Motion> Clips;
    Str              Manifest;   // JSON text
};
```

The in-memory `Bake` takes `Name`, the stem the manifest uses for the mesh, skeleton and clip files. Write
`Bundle::Mesh` as `<Name>.msh`, `Skeleton` as `<Name>.skl`, each clip as `<Name>.<Motion::Name>.anm`, and
`Manifest` as `<Name>.mdl`, all in one folder.

### 11.3 Lower-level pieces

| Class | Use |
|---|---|
| `Importer::Import(Scheduler, Source, Type, Profile)` | Read an FBX into a `Scene` |
| `Scene` | Vertices, indices, surfaces, materials, joints, clips; `Describe(VertexSlot)`, `Retarget(Path)`, `Identify(Name)` |
| `Indexer::Unify(Scene)` | Weld within each material and drop repeated triangles |
| `Indexer::Shadow(Scene, Levels)` | Weld what each level casts for shadows |
| `Indexer::Optimize(Scheduler, Scene, Levels)` | Order runs for the vertex cache and for overdraw, and vertices from the coarsest level up |
| `Tangent::Generate(Scene)` | Tangents from the normals and first UV set, splitting vertices whose corners disagree |
| `Simplifier::Generate(Scheduler, Scene, Details, Drift)` | Levels of detail, their checks and their shadow casters |
| `Layout::Pack(Scene, Attributes)` | Streams, bindings and index width |
| `Exporter::ExportMesh`, `ExportSkeleton`, `ExportAnimation`, `ExportModel` | The four writers |

---

## 12. Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Model a hundred times too big or small | Source units kept (`--units 0`) | `--units 1` |
| Outputs appeared next to the FBX | No destination given | Name a `.mdl` destination, or an `--output` folder |
| Textures missing at runtime | Texture bakes not beside the manifest, or named differently | Bake each to `<image stem>.tex` in the manifest's folder |
| Shader shows no roughness or metal | It reads a packed texture, Studio wrote separate slots | Pack the maps and edit the manifest (10.3) |
| No clips, no skeleton | No skin deformer, or `--no-rig` | Bind the mesh to a skin |
| `--once` had no effect | The name was the raw take name | Use the cleaned name (`Armature_Attack`) |
| A clip from another file moves nothing | Bone names differ, or were renamed for duplicates | Keep bone names unique and identical across files |
| Jerky motion at a low rate | `Linear` between sparse samples | Raise `--rate`, or `--interpolation Cubic` |
| Large `.anm` files | `Hermite` triples every changing sample | Use `Cubic` or `Linear` |
| Vertex buffer larger than expected | `Color0` is always written | `--attributes` without `Color0` |
| Separate parts lost their own transforms | Every node is flattened into one mesh | Export parts as separate files |
| Fewer levels than `--lod` asked for | A level shed less than a quarter of the one before, or left more than 5% of the surface out | Nothing to fix. The level before stands in. Loosen `Details[].Error` from code for coarser levels |
| Levels switch at other sizes than the table's | `--drift` (on by default) sets each coverage from the next level's error | `--drift 0`, or `--coverage` |
| No tangent stream | `--no-tangents`, or the scene lacks normals or UVs | Leave `--tangents` on and export normals and UVs |
| Normal map shades wrong | The map was baked against tangents other than MikkTSpace's | Bake the map with MikkTSpace tangents |
| A socket bone is missing | It hangs under a bone whose children compensate its scale | Turn the compensation off on that bone, or hang the socket elsewhere |
| The log counts fewer triangles than the source has | The source repeats faces, and the repeats are dropped (6.6) | Nothing to fix. Remove the doubled faces in the art to keep the source clean |
| A blended model looks inside out | Its triangles are drawn outside first, and it blends without a depth pass | Draw its depth first, then blend against it |

---

## 13. Where it lives

| Path | Holds |
|---|---|
| `Library/Model/Source/Model/Profile.hpp`, `Profile.cpp` | Settings, the default attribute and detail tables, switch reading |
| `Library/Model/Source/Model/Baker.hpp`, `Baker.cpp` | The pipeline, clip naming, clip bounds, the stall and density warnings, file writing |
| `Library/Model/Source/Model/Scene.hpp`, `Scene.cpp` | The in-memory scene, attribute encodings, texture renaming |
| `Library/Model/Source/Model/Import/FBX.hpp`, `FBX.cpp` | ufbx reading, axes, materials, rig, clip resampling |
| `Library/Model/Source/Model/Process/Indexer.cpp` | Welding, dropped triangles, the shadow weld, draw order and vertex order |
| `Library/Model/Source/Model/Process/Tangent.cpp` | Tangent generation and vertex splitting |
| `Library/Model/Source/Model/Process/Simplifier.cpp` | Levels of detail, their checks and shadow casters |
| `Library/Model/Source/Model/Process/Layout.cpp` | Stream packing, index width, weight rounding |
| `Library/Model/Source/Model/Export/Exporter.cpp` | The `.msh`, `.skl`, `.anm` and `.mdl` writers |
