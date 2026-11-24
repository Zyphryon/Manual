# Sounds

This chapter covers baking FLAC, MP3 and WAV files into `.snd`, the three encodings the engine's mixer decodes, and
how to pick one.

---

## 1. What a sound bake does

```sh
Studio bake Hit.wav Content/Audio/Hit.snd --encoding Adaptive
```

1. dr_libs decodes the source into interleaved 32-bit float frames.
2. The frames are resampled to the mixer's rate, **48000 Hz** (`ZyAudio::kMixerFrequency`), with the engine's own
   `ZyAudio::Resampler`. A source already at 48 kHz is kept as it is.
3. The frames are encoded as 16-bit PCM, IMA ADPCM or Opus.
4. The payload is LZ4-compressed when that comes out smaller, and written after a short header.

The engine refuses any `.snd` at another rate, so the bake is the only place resampling happens. At runtime the
decoder only reads.

---

## 2. Sources

| Extension | Decoder |
|---|---|
| `flac` | dr_flac |
| `mp3` | dr_mp3 |
| `wav` | dr_wav |

- **One or two channels only** (`ZyAudio::kMixerStride` is 2). A source with more channels fails:
  `a source of 6 channels at 48000 Hz is not one the mixer plays`. Downmix surround sources first.
- A truncated stream bakes the frames that could be decoded.
- An empty file fails: `source sound is empty`.

---

## 3. Switches

| Switch | Profile field | Default | Meaning |
|---|---|---|---|
| `--encoding <e>` | `Encoding` | `Linear` | `Linear` (16-bit PCM), `Adaptive` (IMA ADPCM) or `Opus` |
| `--bitrate <n>` | `Bitrate` | `96000` | Bits per second Opus aims for, across every channel |
| `--compressed` | `Compress` | on | LZ4-compress the payload when that is smaller |

An unknown `--encoding` is refused with exit `2`: `'<x>' is not an encoding; use Linear, Adaptive or Opus`.

---

## 4. The encodings

`ZyAudio::Encoding`:

| Encoding | Bits per sample | Decoded | Size for 1 s stereo | Good for |
|---|---|---|---|---|
| `Linear` | 16 | Widened on read | 192 000 bytes (before LZ4) | Short effects that must be exact, sounds played many times at once |
| `Adaptive` | 4 | A block at a time | about 48 000 bytes | Most effects: a quarter the size of `Linear`, cheap to decode |
| `Opus` | variable | A packet at a time | `bitrate / 8` bytes (12 000 at 96 kbit/s) | Music, ambience, voice, anything long |

### 4.1 `Linear`

Each float sample is converted to a signed 16-bit integer, interleaved. LZ4 often shrinks silence and simple
waveforms.

### 4.2 `Adaptive` (IMA ADPCM)

- Blocks of **512 frames** (`ZyAudio::Codec::Adaptive::kBlockFrames`), one block per channel, each
  `4 + 512 / 2 = 260` bytes (`kBlockStride`).
- A block's 4-byte header holds the first sample (16-bit little-endian), the step index, and a zero byte. The first
  sample is stored whole. The remaining 511 are 4-bit codes, two per byte, the earlier one in the low nibble.
- The step index carries from one block to the next, so a block boundary does not reset the quantizer.
- Blocks are laid out frame-block-major: block 0 of channel 0, block 0 of channel 1, block 1 of channel 0, ...
- The last block is padded with zero codes.

ADPCM adds a little hiss on quiet, high-frequency material. Compare by ear for soft sounds.

### 4.3 `Opus`

- Encoded with libopus in `OPUS_APPLICATION_AUDIO` mode at 48 kHz, packets of **960 frames** (20 ms).
- `--bitrate` is the total for every channel. 96 kbit/s is transparent for most stereo music. 64 kbit/s is fine for
  voice and ambience.
- The encoder's lookahead is stored as the **pre-skip**, and one lookahead of silence is encoded past the end, so the
  decoder discards exactly the right number of frames at the start and the tail survives.
- The last packet is padded with silence to a whole 960 frames.
- A packet directory (one 16-bit length per packet) comes before the packets, so a seek finds its packet without
  walking the others.

Opus payloads barely compress further, so LZ4 is usually skipped automatically (the raw payload is stored when LZ4
would not shrink it).

---

## 5. Choosing an encoding

| Sound | Suggestion |
|---|---|
| UI click, footstep, hit | `Adaptive` |
| A short sound that must be bit-exact, or very quiet detail | `Linear` |
| Music track | `Opus`, `--bitrate 128000` for high quality, `96000` default |
| Long ambience loop | `Opus`, `--bitrate 64000` |
| Dialogue | `Opus`, `--bitrate 48000` to `64000` |

Keep loops sample-accurate in the source. Opus pads its last packet, but the header's frame count is the source's
exact frame count, which is what a decoder plays to.

---

## 6. Batch baking

Studio bakes one file per `bake`. Loop in the shell:

```sh
# Every WAV in Art/Audio/Effects as ADPCM, mirrored into Content/Audio/Effects
for f in Art/Audio/Effects/*.wav; do
    name=$(basename "$f" .wav)
    Studio bake "$f" "Content/Audio/Effects/$name.snd" --encoding Adaptive || echo "failed: $f"
done

# Music as Opus
for f in Art/Audio/Music/*.flac; do
    Studio bake "$f" "Content/Audio/Music/$(basename "$f" .flac).snd" --encoding Opus --bitrate 128000
done
```

Or keep the settings in a presets file:

```json
{
    "effect": { "encoding": "Adaptive" },
    "music":  { "encoding": "Opus", "bitrate": 128000 },
    "voice":  { "encoding": "Opus", "bitrate": 64000 }
}
```

```sh
Studio bake Art/Audio/Theme.flac Content/Audio/Theme.snd --presets Audio.json --preset music
```

---

## 7. Library API

Headers under `Library/Sound/Source/Sound/`.

```cpp
struct Profile final
{
    ZyAudio::Encoding Encoding = ZyAudio::Encoding::Linear;
    SInt32            Bitrate  = 96000;
    Bool              Compress = true;

    static Profile From(ConstRef<Environment> Environment);
};

class Baker final
{
public:
    Blob Bake(ConstSpan<Byte> Source, Text Type, ConstRef<Profile> Profile) const;
    Bool Bake(Text Source, Text Destination, ConstRef<Profile> Profile) const;
};
```

`Sound::Baker` takes no scheduler. Lower-level:

| Class | Use |
|---|---|
| `Importer::Import(ConstSpan<Byte>, Text Type)` | Decode and resample into a `Sample` |
| `Sample` | Interleaved float frames: `GetSamples()`, `GetFrequency()`, `GetStride()`, `GetFrames()` |
| `Exporter::Export(ConstRef<Sample>, ConstRef<Profile>)` | Encode a `Sample` already at 48 kHz into `.snd` bytes |

`Exporter::Export` does not resample. Hand it a `Sample` at 48000 Hz, or the engine will refuse the file.

```cpp
// Generate a one-second 440 Hz tone and bake it.
const UInt64 Frames  = 48000;
Blob         Samples = Blob::Allocate<Real32>(Frames);
for (UInt64 Index = 0; Index < Frames; ++Index)
{
    Samples.GetData<Real32>()[Index] = 0.25f * Sine(6.2831853f * 440.0f * Index / 48000.0f);
}

const Studio::Sound::Sample Tone(Move(Samples), 48000, 1, Frames);
Studio::Sound::Profile      Settings;
Settings.Encoding = ZyAudio::Encoding::Adaptive;

const Blob Bytes = Studio::Sound::Exporter::Export(Tone, Settings);
Filesystem::Write("Content/Audio/Tone.snd", Bytes);
```

---

## 8. Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| `... channels ... is not one the mixer plays` | More than two channels | Downmix to stereo or mono |
| `'<x>' is not an encoding` | Misspelt `--encoding` | `Linear`, `Adaptive` or `Opus` |
| Engine logs `is baked at N Hz, which is not the mixer's rate` | A `.snd` written by your own code at another rate | Resample to 48000 Hz before `Exporter::Export` |
| Hiss on a quiet sound | ADPCM quantization | `Linear` or `Opus` |
| Music file is huge | Baked with the default `Linear` | `--encoding Opus` |

---

## 9. Where it lives

| Path | Holds |
|---|---|
| `Library/Sound/Source/Sound/Profile.hpp`, `Profile.cpp` | Settings and switch reading |
| `Library/Sound/Source/Sound/Baker.hpp`, `Baker.cpp` | Decode, encode, write |
| `Library/Sound/Source/Sound/Sample.hpp` | Decoded frames |
| `Library/Sound/Source/Sound/Import/Importer.cpp` | dr_libs decoding and resampling |
| `Library/Sound/Source/Sound/Export/Exporter.cpp` | Linear, ADPCM and Opus encoding, the header |
