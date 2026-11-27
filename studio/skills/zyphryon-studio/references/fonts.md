# Fonts

This chapter covers baking a TrueType or OpenType typeface into a `.fnt`: a multi-channel signed distance field
atlas (MTSDF, through msdfgen) plus metrics, glyphs and kerning, so the engine can draw crisp text at any size from
one bake.

---

## 1. What a font bake does

```sh
Studio bake Inter.ttf Content/UI/Inter.fnt --charset latin1,punctuation
```

1. Each typeface is read with stb_truetype. Every codepoint of the charset the face has a glyph for is taken. A
   codepoint the face lacks is **skipped**, not baked blank, so the engine's fallback glyph (`'?'` by default in
   `ZyRender::Font::GetGlyph`) stands in for it at draw time.
2. Fallback faces fill in codepoints the main face lacks. The first face carrying a codepoint keeps it, and a
   later face gives only the codepoints no earlier face gave.
3. Each glyph's outline is turned into an MTSDF field at `--size` pixels per em, with a distance band of `--range`
   texels. Edges are coloured the ink-trap way (msdfgen's `edgeColoringInkTrap`, the reference atlas generator's
   default), and a line or quadratic curve that ends where it starts is left out. Glyphs are generated in parallel.
4. Glyph fields are packed on square pages (shelf packing, tallest first).
5. The `.fnt` is written: metrics, the glyph table, kerning, and one `RGBA8UIntNorm` texture per page.

Everything except the atlas is stored in **em units**, so one bake serves every text size.

---

## 2. Sources

| Extension | Notes |
|---|---|
| `ttf` | TrueType |
| `otf` | OpenType |
| `ttc` | A collection; `--face` picks the typeface by index |

---

## 3. Switches

| Switch | Profile field | Default | Meaning |
|---|---|---|---|
| `--size <px>` | `Size` | `40` | Em size the glyph fields are generated at, in pixels |
| `--range <texels>` | `Range` | `20` | Width of the band the distance fades across, in atlas texels |
| `--angle <radians>` | `Angle` | `3` | Join angle past which an edge counts as a corner (msdfgen ink-trap edge colouring) |
| `--charset <list>` | `Charset` | `ascii` | Codepoints to bake, see 4 |
| `--face <index>` | `Face` | `0` | Typeface to read out of the main source's collection. Every fallback is read at face `0` |
| `--padding <texels>` | `Padding` | `1` | Gap between neighbouring glyphs in the atlas |
| `--limit <texels>` | `Limit` | `8192` | Largest side of an atlas page before another page opens |
| `--underline <em>` | `Underline` | `1.2` | Vertical space an underline takes, in em units (stored as `UnderlineSize`) |
| `--fallback <list>` | (Baker) | none | More typefaces, `path[=charset];...`, see 5 |

`--size` and `--range` must both be positive.

### 3.1 Size versus range

- **`--size`** sets the resolution of the field. Larger glyphs in the atlas mean sharper corners when text is drawn
  large.
- **`--range`** sets how wide the distance band is. It also pads every glyph, and it is how far an outline or a glow
  drawn by the text shader can reach outward.

When glyphs look soft or rounded at large sizes, raise `--size`, not `--range`. Raise `--range` only for thick
outlines or glows. The engine hands the shader `Range` as `Distance / page side` per axis.

### 3.2 Page size

The atlas tries a single page first, starting at the side that could hold the total glyph area and growing by about
3% at a time up to `--limit`. Only when one page at `--limit` still overflows does it spill onto more pages, each
`--limit` on a side. Pages are square and need not be a power of two. One page is cheaper at runtime, so leave
`--limit` high.

A glyph larger than a whole page fails: `a glyph is larger than a whole atlas page; lower the size or the range`.

---

## 4. Charsets

A charset is a comma-separated list of presets, single codepoints and ranges.

| Entry | Meaning |
|---|---|
| `ascii` | `0x20`-`0x7E` |
| `latin1` | `0x20`-`0x7E` and `0xA0`-`0xFF` |
| `punctuation` | `0x2010`-`0x205E` (dashes, quotes, bullets, ellipsis, primes) |
| `65` or `0x41` | One codepoint, decimal unless it starts with `0x` |
| `first-last` | A range, both ends included, such as `0x0400-0x04FF` |

- Preset names are case-insensitive. Spaces around entries are trimmed.
- Entries may overlap or repeat. A codepoint is baked once.
- Anything else fails the bake with exit `2`: `'abc' is not a codepoint, a range, or a known charset`.
- A list that selects nothing fails: `the charset selects no codepoint at all`.

```sh
--charset ascii
--charset latin1,punctuation,0x20AC            # plus the euro sign
--charset "latin1, 0x0400-0x04FF"              # plus Cyrillic
--charset 0x30-0x39                            # digits only
```

Kerning is measured over every pair of glyphs one face gives, spread over every core. The work still grows with the
square of the glyph count, so very large charsets (thousands of glyphs) bake slowly.

---

## 5. Fallback faces

```sh
Studio bake Inter.ttf Content/UI/Inter.fnt --charset latin1 \
    --fallback "NotoSymbols.ttf=0x2190-0x21FF;Icons.ttf=0xE000-0xE0FF"
```

`--fallback` lists more typefaces, separated by **semicolons**. Each entry is a path, optionally followed by `=` and
its own charset.

- An entry with a charset is asked only for that charset. This is how an icon font gives its icons and nothing else.
- An entry without a charset is asked for the main `--charset`.
- Faces are asked in order: the main face first, then each fallback. The first face carrying a codepoint keeps it.
  A fallback gives only the codepoints no earlier face gave.
- `--face` picks the main typeface out of its collection. Every fallback is read at its first face, index `0`.
- **Metrics** (ascender, descender, underline) come from the first face that gives any glyph, normally the main face.
- **Kerning** is read from each face for the glyphs it gives, and only among them. A fallback never changes the
  kerning of the main face's glyphs, and a pair across two faces has no kerning.
- A fallback charset that cannot be read exits `2` before anything bakes.
- The split on `=` is at the last `=`, so a path may contain `=` as long as a charset follows.
- Quote the list. `;` ends a command in most shells.
- Every face is scaled to the same em size. A fallback designed with a different em box may sit higher or larger.

---

## 6. What is stored

### 6.1 Metrics (`ZyRender::Font::Metrics`)

| Field | Value |
|---|---|
| `Size` | `--size`, the em size the fields were made at, in pixels |
| `Distance` | `--range`, in atlas texels |
| `Ascender` | Typeface ascent, em units |
| `Descender` | Typeface descent, em units (negative below the baseline) |
| `UnderlineOffset` | From the `post` table, em units, or `-0.1` when the face leaves it unfilled |
| `UnderlineSize` | `--underline` |
| `UnderlineThickness` | From the `post` table, em units, or `0.05` when unfilled |

### 6.2 Glyphs (`ZyRender::Font::Glyphs`, a `Table<UInt32, Glyph>` keyed by codepoint)

| Field | Value |
|---|---|
| `Local` | The quad's rectangle around the pen, em units, origin on the baseline, Y up |
| `Atlas` | The quad's rectangle on its page, as fractions of the page side, inset by half a texel |
| `Advance` | How far the pen moves, em units |
| `Page` | The atlas page |

A blank (a space) has zero rectangles and only an advance.

### 6.3 Kerning (`ZyRender::Font::Kerning`, a `Table<UInt64, Real64>`)

Keyed by `(first << 32) | second` codepoint, valued in em units. Both glyphs of a pair come from the same face. Read
from `GPOS` if the face has one, else from the older `kern` table. Written only when not empty.

### 6.4 Pages

Each page is a native texture inside the `.fnt`: `Texture2D`, `RGBA8UIntNorm`, one level, LZ4 at level 9. The three
colour channels hold the multi-channel distance, alpha the true distance (MTSDF). The engine wraps each page in a
material with the page under `Albedo` and the range under `Range`.

Byte layout: `formats.md`.

---

## 7. Library API

Headers under `Library/Font/Source/Font/`.

```cpp
using Interval = AnyRange<UInt32>;

struct Profile final
{
    Real32             Size      = 40.0f;
    Real32             Range     = 20.0f;
    Real32             Angle     = 3.0f;
    Real32             Underline = 1.2f;
    UInt32             Face      = 0;
    UInt32             Padding   = 1;
    UInt32             Limit     = 8192;
    Sequence<Interval> Charset;

    static Profile From(ConstRef<Environment> Environment);
    static Bool    Parse(Text Description, Ref<Sequence<Interval>> Output);
};

class Baker final
{
public:
    struct Source   { ConstSpan<Byte> Data; Text Type; Sequence<Interval> Charset; };
    struct Fallback { Text Path; Text Charset; };

    explicit Baker(Ref<ZyJob::Service> Scheduler);

    Blob Bake(ConstSpan<Source> Sources, ConstRef<Profile> Profile) const;
    Bool Bake(Text Source, Text Destination, ConstRef<Profile> Profile, ConstSpan<Fallback> Fallbacks = { }) const;
};
```

`Profile::From` reads `--charset` with `ascii` as the default and leaves `Charset` empty when it cannot be read. A
default-constructed `Profile` has an **empty** charset, which the baker refuses, so fill it from code:

```cpp
Studio::Font::Baker   Fonts(* Scheduler);
Studio::Font::Profile Settings;
Settings.Size = 48.0f;
Studio::Font::Profile::Parse("latin1,punctuation", Settings.Charset);

const Studio::Font::Baker::Fallback Icons[] = { { "Art/Fonts/Icons.ttf", "0xE000-0xE0FF" } };
Fonts.Bake("Art/Fonts/Inter.ttf", "Content/UI/Inter.fnt", Settings, Icons);
```

---

## 8. Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Some characters draw as `?` | Not in the charset, or no face has them | Add them to `--charset`, or a `--fallback` face |
| Soft corners at large sizes | Field resolution too low | Raise `--size` |
| Outline or glow cut off | The distance band is too narrow | Raise `--range` |
| `a glyph is larger than a whole atlas page` | `--size` plus `--range` exceed `--limit` | Lower them, or raise `--limit` |
| Several pages where one was expected | Charset too large for `--limit` | Raise `--limit`, or shrink `--size` |
| `index N does not exist` | `--face` past the main collection's faces | Pick a lower index |
| A fallback collection gives the wrong typeface | Fallbacks are always read at face `0` | Save that typeface as a file of its own |
| Shell error on `--fallback` | Unquoted `;` | Quote the list |
| Very slow bake | Huge charset, kerning is pairwise | Bake only what the game shows |

---

## 9. Where it lives

| Path | Holds |
|---|---|
| `Library/Font/Source/Font/Profile.hpp`, `Profile.cpp` | Settings, charset presets and parsing |
| `Library/Font/Source/Font/Baker.hpp`, `Baker.cpp` | Face order, glyph table, the profile's metrics |
| `Library/Font/Source/Font/Import/Importer.cpp` | stb_truetype reading, the codepoints each face gives, metrics, kerning |
| `Library/Font/Source/Font/Process/Generator.cpp` | msdfgen field generation |
| `Library/Font/Source/Font/Process/Atlas.cpp` | Page sizing and shelf packing |
| `Library/Font/Source/Font/Export/Exporter.cpp` | The `.fnt` writer |
| `Application/Source/Console.cpp` | `BakeFont`, `--fallback` parsing |
