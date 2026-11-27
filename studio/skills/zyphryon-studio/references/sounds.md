# Sounds

This chapter covers baking FLAC, MP3 and WAV files into `.snd`, the three encodings the engine's mixer decodes, and
how to pick one.

---

## 1. What a sound bake does

```sh
Studio bake Hit.wav Content/Audio/Hit.snd --encoding Adaptive
```

1. dr_libs decodes the source into interleaved 32-bit float frames.
2. The frames are resampled to the mixer's rate, **48000 Hz** (`ZyAudio::kMixerFrequency`), through a
   Kaiser-windowed sinc filter (1.1). A source already at 48 kHz is kept as it is, sample for sample.
3. The frames are encoded as 16-bit PCM, IMA ADPCM or Opus.
4. The payload is LZ4-compressed when that comes out smaller, and written after a short header.

The engine refuses any `.snd` at another rate, so the bake is the only place resampling happens. At runtime the
decoder only reads.

### 1.1 Resampling

`Importer::Resample` is an offline band-limited resampler:

- The kernel is a sinc windowed by a Kaiser window of beta `10` (`kBeta`), which holds the stopband about 100 dB
  down. It spans `64` zero crossings on each side of its centre (`kCrossings`).
- The cutoff is `0.95` (`kBandwidth`) of the lower of the two Nyquist rates, so downsampling filters out what
  48 kHz cannot hold, and upsampling adds no images above the source's band.
- The two rates are reduced to a ratio of whole numbers, so every output frame falls on one of a fixed set of
  phases (160 for 44.1 kHz). Each phase's taps are computed once and sum to one, so a constant level stays level.
- The source reads as silence past both ends.
- The output holds `floor(frames × 48000 / rate)` frames.

---

## 2. Sources

| Extension | Decoder |
|---|---|
| `flac` | dr_flac |
| `mp3` | dr_mp3 |
| `wav` | dr_wav |

- **One or two channels only** (`ZyAudio::kMixerStride` is 2). A source with more channels fails:
  `a source of 6 channels at 48000 Hz is not one the mixer plays`. Downmix surround sources first.
- FLAC and MP3 decode to the end of the stream, so a FLAC with no length in its header bakes, and a truncated FLAC
  or MP3 bakes the frames that could be decoded.
- A WAV whose data is shorter than its header says is refused: `the source is not a readable wav stream`, with
  the extension as the file spells it.
- An empty file fails: `source sound is empty`.

---

## 3. Switches

| Switch | Profile field | Default | Meaning |
|---|---|---|---|
| `--encoding <e>` | `Encoding` | `Linear` | `Linear` (16-bit PCM), `Adaptive` (IMA ADPCM) or `Opus` |
| `--bitrate <n>` | `Bitrate` | `96000` | Bits per second Opus aims for, across every channel |
| `--compressed` | `Compress` | on | LZ4-compress the payload when that is smaller |
| `--loop` | `Loop` | off | Encode an Opus sound as a loop, primed with its own tail, so it repeats without a click (4.3) |

An unknown `--encoding` is refused with exit `2`: `'<x>' is not a value '--encoding' takes`. So is a `--bitrate`
that is not a number. A bitrate the Opus encoder refuses (`0`, say) fails the bake:
`the Opus encoder refused a bitrate of <n> bits per second`.

---

## 4. The encodings

`ZyAudio::Encoding`:

| Encoding | Bits per sample | Decoded | Size for 1 s stereo | Good for |
|---|---|---|---|---|
| `Linear` | 16 | Widened on read | 192 000 bytes (before LZ4) | Short effects that must be exact, sounds played many times at once |
| `Adaptive` | 4 | A block at a time | about 48 000 bytes | Most effects: a quarter the size of `Linear`, cheap to decode |
| `Opus` | variable | A packet at a time | `bitrate / 8` bytes (12 000 at 96 kbit/s) | Music, ambience, voice, anything long |

### 4.1 `Linear`

Each float sample is scaled by 32768 (what the decoders divide by), rounded and clamped to a signed 16-bit integer,
interleaved. A sample that does not already sit on a 16-bit step gets triangular dither first, from a fixed seed, so
a source always bakes to the same bytes. A 16-bit 48 kHz source bakes to its own samples. LZ4 often shrinks silence
and simple waveforms.

### 4.2 `Adaptive` (IMA ADPCM)

- Blocks of **512 frames** (`ZyAudio::Codec::Adaptive::kBlockFrames`), one block per channel, each
  `4 + 512 / 2 = 260` bytes (`kBlockStride`).
- A block's 4-byte header holds the first sample (16-bit little-endian), the step index, and a zero byte. The first
  sample is stored whole. The remaining 511 are 4-bit codes, two per byte, the earlier one in the low nibble.
- The step index carries from one block to the next, so a block boundary does not reset the quantizer.
- Blocks are laid out frame-block-major: block 0 of channel 0, block 0 of channel 1, block 1 of channel 0, ...
- The last block is padded with zero codes.
- Samples are scaled by 32768 like `Linear`, without dither, whose one step the codec's far coarser ones would
  swallow.

ADPCM adds a little hiss on quiet, high-frequency material. Compare by ear for soft sounds.

### 4.3 `Opus`

- Encoded with libopus in `OPUS_APPLICATION_AUDIO` mode at 48 kHz, packets of **960 frames** (20 ms), at
  complexity 10 with unconstrained VBR, since a file has no buffer to stay within.
- `--bitrate` is the total for every channel. 96 kbit/s is transparent for most stereo music. 64 kbit/s is fine for
  voice and ambience.
- The encoder's lookahead is stored as the **pre-skip**, and one lookahead of silence is encoded past the end, so the
  decoder discards exactly the right number of frames at the start and the tail survives.
- The last packet is padded with silence to a whole 960 frames.
- With `--loop`, the sound is read round and round. The first packet opens with the sound's own tail, a prime of
  `959` frames less the lookahead, and the pre-skip covers the lookahead and the prime, `959` frames in all. What is
  encoded past the end, and the padding of the last packet, is the sound's own start instead of silence. The codec
  then sees no edge at the seam, so the loop repeats without a click. `--loop` changes nothing for `Linear` and
  `Adaptive`.
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
| Long ambience loop | `Opus`, `--bitrate 64000 --loop` |
| Dialogue | `Opus`, `--bitrate 48000` to `64000` |

Keep loops sample-accurate in the source. Opus pads its last packet, but the header's frame count is the source's
exact frame count, which is what a decoder plays to. Bake a looping Opus sound with `--loop`, or its seam may click.

---

## 6. Batch baking

`bake` takes several sources or folders and bakes them in parallel. A folder gives every file in it a baker reads,
sorted by name, but not its subfolders. Each result goes beside its source, or into `--output <folder>` under the
source's name. The run ends with `Studio: N baked, M failed`, and exits `1` if any failed.

```sh
# Every sound in Art/Audio/Effects as ADPCM, into Content/Audio/Effects
Studio bake Art/Audio/Effects --output Content/Audio/Effects --encoding Adaptive

# Music as Opus
Studio bake Art/Audio/Music --output Content/Audio/Music --encoding Opus --bitrate 128000
```

Every file of one run bakes with the same switches, so keep a folder to one kind of sound. Two sources with the same
name would write one file, so a run that holds them is refused before anything bakes.

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
    Bool              Loop     = false;

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

`Exporter::Export` does not resample. Hand it a `Sample` at 48000 Hz. A sample at another rate is refused with
`a sample at <n> Hz is not at the mixer's rate`, and an empty blob comes back.

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
| `'<x>' is not a value '--encoding' takes` | Misspelt `--encoding` | `Linear`, `Adaptive` or `Opus` |
| `a sample at N Hz is not at the mixer's rate` | Your own code handed `Exporter::Export` a sample at another rate | Resample to 48000 Hz before `Exporter::Export` |
| `the Opus encoder refused a bitrate of N bits per second` | `--bitrate` of 0 | A bitrate such as `96000` |
| A looping Opus sound clicks at its seam | Baked without `--loop` | `--loop` |
| `the source is not a readable wav stream` | A WAV cut short, its data shorter than its header says | Export the WAV again |
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
| `Library/Sound/Source/Sound/Export/Exporter.cpp` | Linear, ADPCM and Opus encoding, dither, loops, the header |
