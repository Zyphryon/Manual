# Math

This chapter teaches `Zyphryon.Math` (namespace `inline ZyMath`, so you write `Vector3` directly) and the scalar
helpers that live in `Zyphryon.Base/Scalar.hpp`: vectors, angles, quaternions, matrices, transforms, boxes and rects,
rays, ranges, colours, easing and tweens, random numbers and noise, and fixed-point packing. It ends with how to keep
positions exact far from the origin. It leans on `base.md` for the type aliases and containers, on `scene.md` for
where transforms live in a world, on `graphics.md` for cameras and projections, and on `network.md` for where
packed values travel.

## 1. Conventions that hold everywhere

- **Y is up.** The ground is the X/Z plane. `Vector3::FromXZ(Ground, Height)` and `Vector3::GetXZ()` move between a
  ground point and a 3D one.
- **Left-handed.** `Matrix4x3::CreateLook` and `Matrix4x4::CreatePerspective` are left-handed; +Z is forward
  (`Quaternion::GetForward()`).
- **Angles are `Angle` values**, not bare reals. An `Angle` stores radians.
- **Everything is a value.** Vectors, angles, boxes and matrices are passed by value (`Matrix4x4` and `Matrix4x3`
  usually by `ConstRef`) and every operation returns a new value. Statics do the work: `Vector2::Normalize(V)`,
  `Vector3::Dot(A, B)`, `Quaternion::Slerp(A, B, T)`.
- **Getters and setters, not public fields:** `V.GetX()`, `V.SetX(1.0f)`, `V.Set(X, Y)`.
- **Debug asserts guard the domain.** Normalizing a zero vector, multiplying an unnormalized quaternion or reflecting
  over a non-unit normal asserts in debug and returns garbage in release.
- **`Real32` loses precision far from the origin.** See section 12.

## 2. Scalars (`Zyphryon.Base/Scalar.hpp`)

These are free functions in `ZyBase`, used unqualified.

| Function | What it does |
|---|---|
| `Min(A, B, ...)`, `Max(A, B, ...)`, `Clamp(V, Lo, Hi)` | Any count of arguments for `Min`/`Max`. |
| `Abs`, `Sign`, `Floor`, `Ceil`, `Round` | As named. |
| `Mod(A, B)` | `fmod` for reals (asserts on a zero divisor). |
| `Wrap(V, Size)` | Into `[0, Size)`, negative values counting down from the top; integers and reals. |
| `Pow`, `Exp`, `Sqrt`, `Cbrt`, `ReciprocalSqrt`, `Log(N, Base = 2)`, `Log10` | Powers and logs. |
| `Sine`, `Cosine`, `Tangent`, `InvSine`, `InvCosine`, `InvTangent(Ratio)`, `InvTangent(X, Y)` | Radians. `InvTangent(X, Y)` is `atan2(Y, X)`: **X first**. |
| `Lerp(A, B, T)`, `InverseLerp(A, B, V)`, `Remap(V, A, B, From, To)` | Interpolation; `Remap` clamps. |
| `Settle(Delta, Time)` | The share of a gap to close this frame so it closes in about `Time` seconds at any frame rate. |
| `IsAlmostZero(V, Tol)`, `IsAlmostEqual(A, B, Tol)`, `IsBetween(V, Lo, Hi)`, `IsNaN`, `IsInf` | Tests. |
| `EncodeNormalized<Int>(V)`, `DecodeNormalized<Int>(I)` | `[0,1]` or `[-1,1]` spread over an integer. |
| `EncodeFixed<Int>(V, Scale)`, `DecodeFixed<Int>(I, Scale)` | Fixed point with `Scale` steps per unit. `EncodeFixed` takes integers of 32 bits or less. |
| `kPI<T>`, `kTwoPI<T>`, `kEpsilon<T>`, `kTolerance<T>`, `kInfinity<T>`, `kNaN<T>`, `kMinimum<T>`, `kMaximum<T>` | Constants. |

Smoothing toward a target at a frame-rate independent pace:

```cpp
// The camera closes 99.9 % of the gap in 0.25 seconds, whatever the frame time.
mZoom = Lerp(mZoom, Target, Settle(Delta, 0.25f));
```

Fixed point for a value that travels or is saved small:

```cpp
const SInt16 Packed = EncodeFixed<SInt16>(Health, 100.0f);   // centi-points, clamped to the SInt16 range
const Real32 Back   = DecodeFixed<SInt16>(Packed, 100.0f);
```

`EncodeFixed` rounds to the nearest step and clamps to what the integer holds, so 400 health at scale 100 saturates
an `SInt16` at 32767. Pick the integer and scale for the range you need.

## 3. Vectors

### 3.1 The types

| Type | Lanes | Notes |
|---|---|---|
| `Vector2`, `IntVector2`, `UIntVector2` | `AnyVector2<Real32 / SInt32 / UInt32>` | Ground points, screen pixels, grid cells. |
| `Vector3`, `IntVector3`, `UIntVector3` | `AnyVector3<...>` | 3D points and directions. |
| `Vector4` | 4 x `Real32`, SIMD, 16-byte aligned | Used inside matrices and quaternions; rarely in game code. |

Construction: `Vector2(X, Y)`, `explicit Vector2(Scalar)` (both lanes), `Vector3(X, Y, Z)`,
`explicit Vector3(Vector2 XY, Z = 0)`, `Vector3::FromXY(XY, Z)`, `Vector3::FromXZ(XZ, Y)`. Conversion between lane
types is explicit: `IntVector2(Vector2::Floor(P))`, `Vector3(IntVector3 Cell)`.

### 3.2 Members (Vector2 shown; Vector3 matches and adds Z)

| Member | Returns |
|---|---|
| `GetX()`, `GetY()`, `SetX`, `SetY`, `Set(X, Y)` | Lanes (setters return `Ref` for chaining). |
| `GetLength()`, `GetLengthSquared()` | Length. |
| `GetDistance(T)`, `GetDistanceSquared(T)` | Distance to `T`. |
| `GetDistanceManhattan(T)`, `GetDistanceChebyshev(T)`, `GetDistanceOctile(T)` | Grid metrics (Vector2). |
| `GetAngle()` | The direction from +X, in `[-pi, pi]` (Vector2). |
| `GetAngle(Other)` | The angle between the two, in `[0, pi]`. Asserts on a zero vector. |
| `GetMinComponent()`, `GetMaxComponent()` | Lane extremes. |
| `IsAlmostZero()`, `IsAlmostEqual(X, Y)`, `IsNormalized()`, `IsParallel(O)`, `IsUniform()` | Tests. |
| `IsComponentWiseLess(O)`, `IsComponentWiseLessOrEqual(O)`, `IsComponentWiseGreater(O)`, `IsComponentWiseGreaterOrEqual(O)` | Lane-wise tests. |
| `GetXY()`, `GetXZ()` | Vector3 only: drop a lane. |
| `+ - * /` (vector and scalar on the right), unary `-`, `+= -= *= /=`, `== != < <= > >=` | Arithmetic is per lane. |
| `<< >>` | Integer vectors: lane shifts. |

Statics:

| Static | Returns |
|---|---|
| `Zero()`, `One()`, `UnitX()`, `UnitY()`, `UnitZ()` | Constants. |
| `Normalize(V)` | Unit vector. **Asserts on a zero vector.** |
| `Dot(A, B)`, `Cross(A, B)` | Vector2 `Cross` is the scalar z of the 3D cross; `Cross(A, B, C)` is the turn of three points. |
| `Min`, `Max`, `Abs`, `Clamp(V, Lo, Hi)`, `Floor`, `Ceil` | Lane-wise. |
| `Project(Source, Onto)`, `Reflect(I, N)`, `Slide(I, N)`, `Perpendicular(V)` | `Reflect`/`Slide` assert on a non-unit normal. |
| `Toward(From, To, Step)` | Moves at most `Step` toward `To`, landing on it when closer. |
| `Rotate(V, Angle)`, `FromAngle(Angle)` | Vector2: rotate; the unit vector at an angle from +X toward +Y. |
| `Vector3::FromBearing(Bearing, Elevation)` | A unit direction: bearing across the ground from +X toward +Z, elevation up. |
| `EncodeNormalized<Int>`, `EncodeFixed<Int>(V, Scale)`, `EncodeHalf` | Packed lanes as an `Array`. |
| `IntVector2::Walk(From, Into, Visit(Cell, Step) -> Bool)` | Every grid cell a line between two cell centres crosses. |

```cpp
// Steer a hunter toward its prey at 4 units per second, on the ground plane.
const Vector2 Apart = Prey - Hunter;
if (!Apart.IsAlmostZero())
{
    Hunter = Vector2::Toward(Hunter, Prey, 4.0f * static_cast<Real32>(Delta));
}

// Is the prey within the hunter's cone, 60 degrees either side of where it faces?
const Vector2 Facing = Vector2::FromAngle(Heading);
const Bool    Within = Vector2::Dot(Facing, Vector2::Normalize(Apart)) >= Angle::Cosine(Angle::FromDegrees(60.0f));
```

**Traps**

- Put the scalar on the **right**: `Direction * 2.0f`. There is no `Real32 * Vector2` operator.
- `Normalize` of a zero vector asserts. Guard with `IsAlmostZero()` first (the pattern above).
- Dividing an integer vector truncates toward zero, it does not floor. For grid cells of negative coordinates use
  `IntVector2(Vector2::Floor(P / Size))`.

## 4. Angles

`Angle` (`Angle.hpp`) stores radians in a `Real32`.

| Member | Returns |
|---|---|
| `Angle::FromRadians(R)`, `FromDegrees(D)`, `FromTurns(T)` | Construction (constexpr). |
| `FromSine`, `FromCosine`, `FromTangent`, `FromCartesian(X, Y)` | Inverse trigonometry. |
| `GetRadians()`, `GetDegrees()`, `GetTurns()`, `IsValid()` (in `[0, 2pi]`) | Reading. |
| `+ - *(Real32) += -= *=`, `< <= > >=`, `==` (almost equal, not wrapped) | Arithmetic. |
| `Angle::Normalize(A)` | Into `[0, 2pi)`. |
| `Angle::Between(From, To)` | The shortest signed turn, in `[-pi, pi)`. |
| `Angle::Toward(From, To, Step)` | Turns at most `Step` the shortest way, landing on `To` when within it. |
| `Angle::Sine(A)`, `Cosine(A)`, `Tangent(A)` | Trigonometry. |
| `Angle::Encode<UInt>(A)`, `Angle::Decode<UInt>(I)` | One turn spread over an unsigned integer's full range. |

```cpp
// Turn toward the prey at 270 degrees per second.
const Angle Want = Apart.GetAngle();
const Angle Turn = Angle::FromDegrees(270.0f) * static_cast<Real32>(Delta);
Heading = Angle::Toward(Heading, Want, Turn);

// Pack a heading into one byte (256 steps a turn) or two (65536 steps).
const UInt8  Small = Angle::Encode<UInt8>(Heading);
const UInt16 Fine  = Angle::Encode<UInt16>(Heading);
const Angle  Back  = Angle::Decode<UInt16>(Fine);
```

`Encode` normalizes first, so any angle of any sign packs; the step past the last wraps to zero.

**Traps**

- `Angle(Real32)` is an **implicit** constructor taking **radians**. A function taking an `Angle` called with `90.0f`
  turns 90 radians, not 90 degrees. Always write `Angle::FromDegrees(90.0f)`.
- `==` compares radians almost-equal without wrapping: `Angle::FromDegrees(0)` and `Angle::FromDegrees(360)` are not
  equal. Compare `Angle::Between(A, B)` with zero instead.

## 5. Quaternions

`Quaternion` (`Quaternion.hpp`) is a 16-byte aligned SIMD rotation. The default constructor is the identity.

| Member | Returns |
|---|---|
| `Quaternion()`, `Quaternion(X, Y, Z, W)`, `Quaternion(Vector3 Imaginary, Real32 Real)` | Construction. |
| `Quaternion::FromAngles(Angle, Vector3 Axis)` | A turn about a unit axis. |
| `Quaternion::FromAngles(Vector3 Rotation)` | A rotation vector (axis times angle), identity for zero. |
| `Quaternion::FromEulerAngles(Vector3 Angles)` | Radians, X then Y then Z as pitch, yaw, roll. |
| `Quaternion::FromDirection(Forward, Up)` | Turns +Z onto `Forward` and +Y toward `Up` (both normalized). |
| `Quaternion::Slerp(A, B, T)`, `Normalize`, `Conjugate`, `Inverse`, `Dot`, `Rotate(Q, V)` | Operations. |
| `GetForward()` (+Z), `GetBackward()`, `GetUp()`, `GetDown()`, `GetRight()`, `GetLeft()` | Rotated basis vectors. |
| `GetAxis(Ref<Angle> Rotation) -> Vector3` | Axis, angle out. |
| `A * B` | `B` applied first, then `A`. Asserts unless `A` is normalized. |

```cpp
const Quaternion Yaw   = Quaternion::FromAngles(Angle::FromDegrees(45.0f), Vector3::UnitY());
const Vector3    Ahead = Quaternion::Rotate(Yaw, Vector3::UnitZ());
```

## 6. Matrices

| Type | Shape | Use |
|---|---|---|
| `Matrix3x2` | 2D affine | UI and 2D transforms. |
| `Matrix4x3` | 3D affine, three rows of a 4x4 | World transforms, the listener and emitter poses of the audio service, views. |
| `Matrix4x4` | Full 4x4 | Projections (`CreatePerspective`, `CreateOrthographic`). |

Shared statics (`Matrix4x3` names; the others mirror what applies):

| Static | Returns |
|---|---|
| `Identity()` | The identity. |
| `FromTranslation(V)`, `FromScale(V)`, `FromRotation(Q)`, `FromRotationX/Y/Z(Angle)` | Elementary transforms. |
| `FromTransform(Translation, Scale, Rotation)` | Scale, then rotation, then translation. |
| `FromTransform(Origin, Translation, Scale, Rotation)` | Same, pivoting about `Origin`. |
| `FromAxes(X, Y, Z, Origin)` | From a basis. |
| `CreateLook(Eye, Focus, Up)`, `CreateDirection(Eye, Direction, Up)` | View matrices. |
| `Project(M, Vector3)`, `Project(M, Vector2)`, `ProjectDirection(M, V)` | Transform a point / a direction. |
| `Project(M, ConstSpan<Vector3>, Span<Vector3>)` | Batch. |
| `Inverse(M)`, `Lerp(A, B, T)`, `GetDeterminant(M)` | Operations. |
| `WithTranslation(M, Offset)`, `WithScale(M, V)` | A copy with one part changed. |

`Matrix4x4::CreatePerspective(Angle FovY, Real32 Aspect, Real32 ZNear, Real32 ZFar)` builds a left-handed
projection to match `CreateLook`.

Members: `GetTranslation()`, `GetBasisX/Y/Z()`, `GetRight()`, `GetLeft()` ..., `GetColumn(I)`. `A * B` applies `B`
first: a child's world matrix is `Parent * Local`.

```cpp
// A lantern hanging 1.2 units above and 0.3 ahead of its holder.
const Matrix4x3 Local = Matrix4x3::FromTransform(Vector3(0.0f, 1.2f, 0.3f), Vector3::One(), Quaternion());
const Matrix4x3 World = Holder * Local;
const Vector3   Tip   = Matrix4x3::Project(World, Vector3::UnitZ());
```

## 7. `Transform`

`Transform` (`Transform.hpp`) holds translation, scale and rotation and computes a `Matrix4x3`:
`Transform(Translation, Scale, Rotation)`, `SetTranslation`, `SetScale`, `SetRotation`, `Translate`, `Scale`,
`Rotate(Quaternion)`, `Rotate(Angle, Axis)`, `Rotate(Vector3 EulerAngles)`, `Compute() -> Matrix4x3`,
`Transform::Lerp(A, B, T)`. The mutators return `Ref<Transform>`, so they chain.

```cpp
Transform Prop;
Prop.SetTranslation(Vector3(4.0f, 0.0f, 2.0f));
Prop.Rotate(Angle::FromDegrees(30.0f), Vector3::UnitY());

const Matrix4x3 Matrix = Prop.Compute();
```

A component of your own named `Transform` is visible next to `ZyMath::Transform` everywhere (the namespace is
inline). Qualify one of them where both are in scope.

## 8. Boxes, rects and other shapes

### 8.1 `Box` and `Rect`

`AnyBox<T>` (`Geometry/Box.hpp`) is an axis-aligned 3D box: `Box` (`Real32`), `IntBox` (`SInt32`), `UIntBox`.
`AnyRect<T>` (`Geometry/Rect.hpp`) is the 2D one: `Rect`, `IntRect`, `UIntRect`.

Both are stored as **minimum and maximum corners**, not position and size:

```cpp
const Rect Field(0.0f, 0.0f, 10.0f, 6.0f);           // MinX, MinY, MaxX, MaxY: a 10 x 6 field
const Box  Den(Vector3(-2, 0, -2), Vector3(2, 3, 2)); // Minimum, Maximum
```

| Member | Returns |
|---|---|
| `GetMinimum()`, `GetMaximum()`, `GetMinimumX()` ..., `SetMinimum`, `SetMaximum`, `Set(...)` | Corners. |
| `GetWidth()`, `GetHeight()`, `GetDepth()`, `GetSize()`, `GetCenter()`, `GetExtents()` (Box), `GetPosition()` | Measures. |
| `GetArea()`, `GetVolume()` (Box), `GetPerimeter()` (Rect), `GetNearest(Point)` | Measures. |
| `Contains(Point)`, `Contains(Other)`, `Test(Other)` | Containment and overlap. |
| `Expand(Amount)`, `Contract(Amount)`, `+ / -` a vector (shift) | Derived boxes. |
| `IsValid()` (min <= max), `IsAlmostZero()`, `IsEmpty()` (Rect) | Tests. |
| `GetXY()`, `GetXZ()` | Box only: the 2D footprint. |
| `Invalid()`, `Zero()`, `One()`, `FromXY`, `FromXZ(Rect, MinY, MaxY)` | Constructors. |
| `Union`, `Intersection`, `Overlaps`, `Min`, `Max`, `Canonicalize`, `Floor`, `Ceil` | Set operations. |
| `Enclose<Int>(Box)` | The integer box that covers it. |
| `Quantize<Int>(Box, Cell)` | The cells it touches. |
| `Transform(Box, Matrix4x3)` | The box around a transformed box. |
| Rect: `Anchor(Rect, Pivot2D)`, `Within(Region, Share)`, `ForEachRectDiff(A, B, Action)` | Layout helpers. |

`Box::Invalid()` is the empty starting value for growing a bound with `Box::Union`:

```cpp
Box Bound = Box::Invalid();

for (ConstRef<Box> Part : Parts)
{
    Bound = Box::Union(Bound, Part);
}
```

**Trap:** `Rect::Contains(Point)` is **half-open**: a point on the maximum edge is outside. That is right for grid
cells and wrong for "is the cursor on this button's last pixel"; expand by an epsilon if you need the edge.

### 8.2 Circles, spheres, cylinders, lines, polygons

| Type | Header | Notes |
|---|---|---|
| `Circle` | `Geometry/Circle.hpp` | `Circle(Center, Radius)`, `Contains`, `Test(Circle)`, `Test(Rect)`, `GetDistance`, `Merge`. |
| `Sphere` | `Geometry/Sphere.hpp` | `Sphere(Center, Radius)`, `Contains`, `Test`, `Merge`, `Sphere::Enclose(Box)`, `GetXZ()`. |
| `Cylinder` | `Geometry/Cylinder.hpp` | Upright cylinders, a common body shape. |
| `Line`, `Quad`, `Polygon` | `Geometry/` | Segments, quads, convex polygons. |
| `Frustum` | `Geometry/Frustum.hpp` | Camera culling (`Test(Box)`). |
| `Shape2D` / `Shape3D` | `Collision/Shape.hpp` | `Variant<Rect, Circle>` / `Variant<Box, Cylinder>`. |
| `Collide::Test(A, B, Ref<Manifold3D>)`, `Sweep::Test(A, B, Motion, Ref<Manifold3D>)` | `Collision/` | Contact and swept tests between shapes, filling a contact manifold. |

## 9. Rays

`Ray` (`Geometry/Ray.hpp`): `Ray(Origin, Direction)` with a direction that **must already be normalized**, or
`Ray::Between(From, To)`.

| Member | Returns |
|---|---|
| `GetOrigin()`, `GetDirection()`, `GetPoint(Distance)`, `GetNearest(Point)`, `GetDistance(Point)` | Reading. |
| `Ray::Test(Ray, Box, Limit = inf)`, `Ray::Test(Ray, Sphere, Limit)` | Hit or miss. |
| `Ray::Intersects(Ray, Box, Ref<Real32> Distance, Limit)` | The entry distance. |
| `Ray::Traverse(Ray, Box, Ref<Real32> Entry, Ref<Real32> Exit, Limit)` | The interval inside. |
| `Ray::Transform(Ray, Matrix4x3)` | Into another space. |

```cpp
Real32 Hit = 0.0f;
if (Ray::Intersects(Ray::Between(Eye, Target), Den, Hit, 50.0f))
{
    const Vector3 Where = Ray::Between(Eye, Target).GetPoint(Hit);
}
```

To test a ray against a model's box, move the ray into the model's space with
`Ray::Transform(Ray, Matrix4x3::Inverse(World))` and test the local box.

## 10. Ranges, colours, curves and motion

### 10.1 `Range` and `IntRange`

`AnyRange<T>` (`Range.hpp`): `Range(Min, Max)` sorts its two ends; `Range(Value)` is fixed. Members: `GetMinimum()`,
`GetMaximum()`, `GetLength()`, `IsFixed()`, `Contains(V)`, `Clamp(V)`, `Lerp(Amount)`, `Pick(Ref<Random>)`.
Use it for "a value somewhere between", never for an ordered pair (a birth and a death), since it swaps the ends.

```cpp
const Range Lifetime(0.8f, 1.4f);
const Real32 Seconds = Lifetime.Pick(mRandom);
```

### 10.2 Colours

`Color` is `AnyColor<Real32>`, `IntColor8` is `AnyColor<UInt8>` (`Color.hpp`). Construction
`Color(R, G, B, A = 1)`, `Color::FromHexadecimal("#FF8800")`, `Color::FromHSV(Vector3, Alpha)`,
`Color::FromColor8(IntColor8)`. Named: `Black()`, `White()`, `Gray()`, `Red()`, `Green()`, `Blue()`, `Yellow()`,
`Cyan()`, `Magenta()`, `Transparent()`. Operations: `Lerp`, `Modulate`, `Fade(Color, Opacity)`, `WithAlpha`,
`Saturate`, `Invert`, `Clamp`.

### 10.3 Easing and tweens

`enum class Easing` (`Motion/Easing.hpp`) names the curves (`Linear`, `InQuad`, `OutQuad`, `InOutQuad`, the cubic,
sine, expo, circ, elastic, bounce and back families, `Smoothstep`). `Ease(Easing, T)` applies one to `T` in `[0, 1]`;
each curve is also a free function (`OutCubic(T)`).

`Tween<T>` (`Motion/Tween.hpp`) eases a value over time: `Tween(Start, End, Seconds, Easing = Linear, Delay = 0)`,
`Tick(Delta) -> T`, `GetValue()`, `Aim(End, Seconds)` (restart from where it is), `Rearm(End)`, `IsIdle()`,
`IsComplete()`. `T` is anything with a static `Lerp` (scalars, vectors, `Color`, `Transform`) or `Slerp`
(`Quaternion`).

```cpp
Tween<Real32> Fade(0.0f, 1.0f, 0.4, Easing::OutCubic);
...
const Real32 Alpha = Fade.Tick(Delta);
```

Other motion helpers: `Track` and `Playback` (keyed curves), `Flipbook` (frame sequences), `Cadence` (fixed-step
timing), `Interpolate`, and `Bezier`, `Polynomial` and `Ramp` (`Curve/`).

## 11. Random numbers and noise

### 11.1 `Random`

`Random` (`Random.hpp`) is xoshiro256++.

| Member | Returns |
|---|---|
| `Random(UInt64 Seed = 0)`, `Random::Seeded()` (from OS entropy) | Construction. |
| `NextInteger<T>()`, `NextInteger<T>(Min, Max)` | Both ends **inclusive**. |
| `NextReal<T>()`, `NextReal<T>(Min, Max)` | `[0, 1)` and `[Min, Max)`. |
| `NextAngle()` | `[0, 2pi)`. |
| `NextPoint(Radius) -> Vector2` | Uniform in a disc. |
| `Pick(ConstSpan<T>) -> ConstPtr<T>` | One element, `nullptr` for an empty span. |
| `Random::Mix(UInt64)`, `Random::Mix(IntVector2)` | Stateless hashes, e.g. a stable value per grid cell. |

For anything that must replay the same way (a saved world, a simulation shared over the network), seed one
`Random` from a stored seed and draw every gameplay roll from it, in a fixed order. Keep separate generators for
cosmetics (a particle's wobble) so they never shift the gameplay sequence. A spread scene walk runs rows on several
workers in no fixed order, so it must not share a generator; derive a value per entity with `Random::Mix` instead.

### 11.2 `Noise`

`Noise` (`Noise.hpp`) is seeded 2D simplex noise: `Noise(Seed)`, `Sample(Point) -> [-1, 1]`,
`Fractal(Point, Octaves, Lacunarity = 2, Gain = 0.5) -> [-1, 1]`, `Ridged(...) -> [0, 1]`.

```cpp
const Noise  Ground(1234);
const Real32 Height = 4.0f * Ground.Fractal(Vector2(X, Z) * 0.05f, 5);
```

## 12. Far from the origin

A `Real32` keeps 24 bits of mantissa. At 1,000 units from the origin it still resolves about 0.00006 of a unit; at a
million units only about 0.06. Movement, distances and collision computed from large absolute `Real32` positions
silently snap. If your world stays within a few thousand units, ignore this section. If it does not:

- Keep **absolute** positions as an integer cell plus a small `Real32` offset (`IntVector3` plus `Vector3`), or in
  `Real64`.
- Do the math in a **local** frame: subtract the two cells as integers first, then add the offsets, and only then
  convert to `Real32`.
- Render relative to the camera: give the GPU positions with the camera's cell already subtracted.

```cpp
struct Spot final
{
    /// The whole-unit cell.
    IntVector3 Cell;

    /// The offset inside the cell, each lane in [0, 1).
    Vector3    Offset;
};

// The exact offset from A to B, small whenever the two are near each other wherever they are.
Vector3 Toward(ConstRef<Spot> A, ConstRef<Spot> B)
{
    return Vector3(B.Cell - A.Cell) + (B.Offset - A.Offset);
}
```

What not to write when positions can be large:

```cpp
// Wrong: both sides are already rounded before the subtraction.
const Vector3 Apart = Vector3(static_cast<Real32>(B.X), static_cast<Real32>(B.Y), static_cast<Real32>(B.Z))
                    - Vector3(static_cast<Real32>(A.X), static_cast<Real32>(A.Y), static_cast<Real32>(A.Z));
```

## 13. Packing values small

When a value is saved or sent many times a second, pack it (`network.md` shows where):

| Value | Pack with |
|---|---|
| A heading | `Angle::Encode<UInt8>` (1.4 degree steps) or `Angle::Encode<UInt16>` |
| A share in `[0, 1]` (health fraction, a blend) | `EncodeNormalized<UInt8>` / `DecodeNormalized` |
| A bounded real (centimetres, centi-points) | `EncodeFixed<SInt16>(V, 100.0f)` / `DecodeFixed` |
| A small 2D offset (within about ±127 units) | `Vector2::EncodeFixed<SInt16>(Offset, 256.0f)` |
| A direction | `Vector3::EncodeNormalized<SInt8>` |

`EncodeFixed` works in `Real64`, so the limits of a 32-bit integer clamp exactly. A 64-bit integer does not compile.
The input is still a `Real32`, so a 32-bit result carries only about 7 significant digits.

## 14. Common mistakes

1. Passing `90.0f` where an `Angle` is expected: radians, through the implicit constructor.
2. `2.0f * Direction`: does not compile; write `Direction * 2.0f`.
3. `Vector2::Normalize` of a zero offset (two things on the same spot).
4. `Rect(X, Y, Width, Height)`: the constructor takes corners.
5. Comparing angles with `==` across the wrap.
6. Large absolute positions in `Real32`, subtracted after rounding.
7. `InvTangent(Y, X)` out of habit: the engine's two-argument form takes X first.
8. A gameplay roll from a cosmetic generator, or one generator shared by a spread walk.
9. Two `Transform` types in scope without qualifying one.
10. A `Ray` built from an unnormalized direction (use `Ray::Between`).

## 15. Where it lives

| What | Path |
|---|---|
| Scalars | `Engine/Source/Zyphryon.Base/Scalar.hpp` |
| Vectors | `Engine/Source/Zyphryon.Math/Vector2.hpp`, `Vector3.hpp`, `Vector4.hpp` |
| Angle, quaternion, matrices, transform | `Engine/Source/Zyphryon.Math/Angle.hpp`, `Quaternion.hpp`, `Matrix3x2.hpp`, `Matrix4x3.hpp`, `Matrix4x4.hpp`, `Transform.hpp` |
| Pivots | `Engine/Source/Zyphryon.Math/Pivot2D.hpp`, `Pivot3D.hpp` |
| Shapes and rays | `Engine/Source/Zyphryon.Math/Geometry/` |
| Collision | `Engine/Source/Zyphryon.Math/Collision/` |
| Colour, range, random, noise | `Engine/Source/Zyphryon.Math/Color.hpp`, `Range.hpp`, `Random.hpp`, `Noise.hpp` |
| Easing, tweens and motion | `Engine/Source/Zyphryon.Math/Motion/` |
| Curves | `Engine/Source/Zyphryon.Math/Curve/` |
