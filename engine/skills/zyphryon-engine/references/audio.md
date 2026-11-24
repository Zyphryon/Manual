# Audio

This chapter teaches `Zyphryon.Audio` (namespace `ZyAudio`): the audio service, the output device, sounds and the
`.snd` format, playbacks (voices), the category submixes and the master volume, 3D placement with emitters and a
listener, the low-pass filter, and what happens when every voice is taken. It leans on `runtime.md` for the kernel
and the startup file, `content.md` for mounts and asynchronous loading, `math.md` for `Matrix4x3`, `Vector3` and
`Angle`, and `base.md` for `Retainer`, `Delegate` and the binary `Reader`.

## 1. What the module is for

The audio module plays baked sounds through the system's output device. It owns one in-house mixer that runs on the
device's thread, and a service on the main thread that the application talks to. There is no streaming from disk:
a sound is loaded whole into memory (compressed or not) and decoded while it plays.

```
application (main thread)
   |  Play, Stop, Pause, SetPlaybackPose, SetSubmixVolume, SetListenerPose, Subscribe ...
   v
ZyAudio::Service              engine subsystem, ticked every frame
   |  commands -> lock-free queue (kMaxCommands = 4096)
   v
ZyAudio::Mixer                runs inside the device callback, 48 kHz stereo, 256-frame blocks
   |  per voice: Decoder -> fold/deinterleave -> Filter -> gains -> sum -> master + limiter
   v
ZyAudio::Driver               WASAPI (Windows), ALSA (Linux), WebAudio worklet (web)
```

What travels back from the mixer is one thing: the list of playbacks that ended, and why. The service drains it on
its tick, releases each ended playback's sound and calls the subscriber, if any.

## 2. Concepts and names

| Name | What it is |
|---|---|
| `Service` | The engine subsystem. The only class an application calls. |
| `Sound` | A content resource (`.snd`): frame count, rate, channels, and the encoded bytes. Shared by every playback of it. |
| `Object` | A playback handle (`Key<kMaxInstances>`). `0`/invalid means "nothing played". Generation-checked, so a stale handle names nothing. |
| `Voice` | The mixer's state for one playback: decoder, gains, filter memory, flags. Internal. |
| `Category` | The submix a playback is routed through: `Music`, `Effect`, `Voice`, `Ambient`, `Interface`. |
| `Emitter` | The spatial settings of a source: attenuation curve, inner and outer radius, cone, starting cutoff. A plain value. |
| `Spatializer` | Turns a source position against the listener into left and right gains. Internal. |
| `Filter` | A second-order (Butterworth) low-pass section used to muffle a voice. |
| `Decoder` | Reads one sound's bytes as float frames. One per playback. `Linear`, `Adaptive`, `Opus`. |
| `Encoding` | How a `.snd` stores samples: `Linear`, `Adaptive`, `Opus`. |
| `Reason` | Why a playback ended: `Completed`, `Stopped`, `Superseded`. |
| `Description` | `Backend`, `Adapter`, `Endpoints`, as the driver reports them. |
| `Resampler` | A linear-interpolation rate converter. The playback path does not use it (sounds are baked at 48 kHz). |

Constants (`Types.hpp`):

| Constant | Value | Meaning |
|---|---|---|
| `kMixerFrequency` | 48000 | The only rate the mixer runs at, and the only rate a `.snd` may have. |
| `kMixerStride` | 2 | Output is interleaved stereo. A sound has 1 or 2 channels. |
| `kMixerBlock` | 256 | Frames mixed per internal sub-block. |
| `kMixerCutoff` | 20000 | A cutoff at or above this, in hertz, is "open": the voice is not filtered. |
| `kMaxInstances` | 256 | Playbacks alive at once. |
| `kMaxCommands` | 4096 | Capacity of the command queue to the mixer and of the completion queue back. |

## 3. The service

### 3.1 Getting it

The runtime kernel registers `ZyAudio::Service` only when the application has a window and the build is not
headless (`ZY_MODE_HEADLESS`). It then calls `Initialize` with the adapter named in the startup file. A failure is
not fatal: the kernel logs `Kernel: Failed to initialize audio service, continuing without sound` and the service
stays registered with no device.

From an application (a `ZyRuntime::Kernel` subclass) or any subsystem:

```cpp
const Retainer<ZyAudio::Service> Audio = GetService<ZyAudio::Service>();   // null in a headless build

if (Audio)
{
    Audio->SetSubmixVolume(ZyAudio::Category::Music, 0.6f);
}
```

The startup file section (see `runtime.md`):

```json
{
  "Audio": { "adapter": "", "pause": true }
}
```

| Key | Default | Meaning |
|---|---|---|
| `adapter` | empty | The output device to open, matched by name. Empty takes the system default. |
| `pause` | `false` | `true` suspends the device while the window is out of focus and restores it on focus. |

### 3.2 Device and lifetime

```cpp
Bool   Initialize(Text Device);              // opens the device, registers the .snd loader
void   Probe(Ref<Description> Output) const; // backend, adapter, endpoints
void   Suspend();                            // stops pulling audio from the mixer
void   Restore();                            // resumes it
void   OnTick(Real64 Delta) override;        // driven by the kernel, never called by hand
```

`Initialize` logs `Audio: Backend '{0}' on device '{1}'` and one `Audio: Found endpoint '{0}'` line per endpoint.
On failure it logs `Audio: Failed to open output device '{0}'` and returns `false`. The `.snd` loader is registered
only on success, so in a process with no device a `.snd` load finds no loader.

| Platform | Backend name | Thread |
|---|---|---|
| Windows | `WASAPI` | Its own render thread, woken by the device event. |
| Linux | `ALSA` | Its own worker thread. An unknown device name falls back to `default`. |
| Web | `WebAudio` | An audio worklet thread. |

`Driver::Advance` runs on the main thread inside `OnTick`; only a backend without a thread of its own does work there.

### 3.3 Volumes

```cpp
void   SetMasterVolume(Real32 Volume);                      // 0..1, asserts outside
Real32 GetMasterVolume() const;
void   SetMasterLimiter(Bool Limited);                      // default true
Bool   HasMasterLimiter() const;
void   SetSubmixVolume(Category Category, Real32 Volume);   // 0..1, asserts outside
Real32 GetSubmixVolume(Category Category) const;
```

All volumes are linear gains, read by the mixer every block, so a change is heard within one block. Each start at
`1.0`.

- **Submix.** Every voice's gain is multiplied by its category's submix volume. There are exactly five submixes, one
  per `Category`. There are no user-made buses, sends or effects.
- **Master.** Multiplied into the summed mix. A change ramps across one block, so it never clicks.
- **Limiter.** With the limiter on, a peak above full scale pulls the gain down at once to hold it at 1.0, then the
  gain climbs back over about 0.15 s. With it off, samples are clamped to [-1, 1] (hard clipping). The final output is
  clamped either way.

A voice's own gain ramps from its last value to its new target across each block, so moving a source or changing a
volume does not click either.

### 3.4 Playing

```cpp
Object Play(Category             Category,
            ConstRetainer<Sound> Sound,
            Real32               Volume,                 // 0..1
            ConstPtr<Emitter>    Emitter   = nullptr,    // both null: a flat stereo playback
            ConstPtr<Matrix4x3>  Transform = nullptr);   // both set:  a spatial playback

void SetPlaybackLooping(Object Handle, Bool Looping);
void SetPlaybackVolume(Object Handle, Real32 Volume);
void SetPlaybackPose(Object Handle, ConstRef<Matrix4x3> Transform);   // spatial playbacks
void SetPlaybackCutoff(Object Handle, Real32 Cutoff);                 // hertz, 0 = unfiltered
void Stop(Object Handle);
void Pause(Object Handle);
void Resume(Object Handle);
void Silence(ConstRef<Sound> Asset);                                  // stops every playback of one sound
```

`Play` asserts that the sound has completed loading, that it is baked at `kMixerFrequency`, that the volume is in
[0, 1], and that `Emitter` and `Transform` are both given or both null. It returns `Object()` (false) when the sound
gives no decoder, when every slot is taken by something more important (section 5), or when the command queue is full.

A playback starts not looping. To loop, call `SetPlaybackLooping(Handle, true)` right after `Play`: the command is
queued behind the play and applied in the same mixer pass, before any audio is produced.

The service retains the `Sound` for as long as the playback lives, so the application may drop its own `Retainer`
right after `Play`.

Every per-playback call is a queued command. A handle that has already ended (or was never valid) is ignored by
the mixer, so calling `Stop` on an old handle is safe.

### 3.5 Hearing a playback end

```cpp
using Callback = Delegate<void(Object, Reason)>;

void Subscribe(Object Handle, Callback Callback);
void Unsubscribe(Object Handle);
```

- The callback runs on the main thread, inside the audio service's tick, once, then the subscription is dropped.
- `Completed`: the sound reached its end and was not looping.
- `Stopped`: `Stop` or `Silence` ended it.
- `Superseded`: another `Play` took its slot (section 5). This one is called **before that `Play` returns**, since
  the slot changes hands there. A callback that plays a new sound from inside is safe: the slot was already secured.
- A looping playback never completes on its own. Only `Stop`, `Silence` or being superseded ends it.

Subscribe right after `Play`. A handle subscribed after its playback already ended is never called back, and its
entry stays in the table until `Unsubscribe`.

## 4. 3D placement

### 4.1 The listener

```cpp
void SetListenerPose(ConstRef<Matrix4x3> Transform);
void SetListenerCone(Angle InnerAngle, Angle OuterAngle, Real32 OuterGain);
```

The listener is one per process. `SetListenerPose` reads the transform's translation, its forward axis
(`GetForward()`, the +Z basis) and its right axis (`GetRight()`, the +X basis). Pass the camera's world transform
each frame, or the transform of whatever the player hears through. Until then it sits at the origin facing +Z.

The listener cone makes sources behind the listener quieter. The default is a full 360 degree cone (no effect).

### 4.2 Emitters

`ZyAudio::Emitter` is a value you fill and pass to `Play`. The mixer copies it, so it can be a temporary.

| Setter | Default | Meaning |
|---|---|---|
| `SetAttenuation(Attenuation)` | `Inverse` | The curve the gain falls along between the two radii. |
| `SetRadius(Inner, Outer)` | 1, 100 | Full volume within `Inner`; silent at and past `Outer`. Asserts `0 <= Inner <= Outer`. |
| `SetCone(InnerAngle, OuterAngle, OuterGain)` | 360, 360, 0 | Full gain inside the inner cone, `OuterGain` outside the outer one, blended between. |
| `SetCutoff(Hertz)` | 0 | The low-pass cutoff the playback starts at. 0 leaves it unfiltered. |

Each has a getter (`GetAttenuation`, `GetInnerRadius`, `GetOuterRadius`, `GetInnerAngle`, `GetOuterAngle`,
`GetOuterGain`, `GetCutoff`).

The cone angles are full widths. Angles are `Angle` values (`Angle::FromDegrees(90.0f)`) and must lie in [0, 2π].

### 4.3 How the gains are computed

For a spatial playback, every block:

1. **Distance.** `d` is the distance from the listener. Inside `Inner` the gain is 1. Past it, with `d` clamped
   to `Outer`:

   | `Attenuation` | Raw value | Then |
   |---|---|---|
   | `Linear` | falls in a straight line from 1 at `Inner` to 0 at `Outer` | used as is |
   | `Inverse` | `Inner / d` | rescaled so `Outer` lands exactly at 0 |
   | `Exponential` | `(Inner / d)^2` | rescaled so `Outer` lands exactly at 0 |

   Every curve reaches silence at the outer radius. Nothing is heard past it.
2. **Emitter cone.** The angle between the source's forward axis and the direction to the listener, against the
   emitter's cone.
3. **Listener cone.** The angle between the listener's forward axis and the direction to the source, against the
   listener's cone.
4. **Pan.** Constant power: `Pan = (dot(direction, listenerRight) + 1) / 2`, left gain `sqrt(1 - Pan)`, right
   `sqrt(Pan)`. A source on top of the listener plays centred.
5. The result is multiplied by the playback volume and the submix volume.

A spatial playback is **mono**: a stereo sound is folded to one channel (the average of left and right) before it is
panned. Play music and interface sounds flat.

There is no Doppler shift, no head-related filtering, no reverb and no occlusion test. To muffle a source behind a
wall, run your own test and call `SetPlaybackCutoff`.

### 4.4 Moving a source

A spatial playback holds its own position. Nothing follows an entity by itself. Call `SetPlaybackPose` whenever the
source moves (once a frame for a moving one). The forward axis is the transform's `GetForward()`. A playback never
given a pose faces +Z.

## 5. When every voice is taken

There are `kMaxInstances` = 256 playback slots. When `Play` finds none free, it compares the newcomer against what is
playing and takes the slot of the least important voice, if that voice ranks lower. The rank is a category floor plus
the gain the voice reaches the listener at (capped at 1):

| Category | Floor |
|---|---|
| `Music` | 6 |
| `Interface` | 5 |
| `Voice` | 4 |
| `Effect` | 2 |
| `Ambient` | 1 |

A category always outranks the ones below it whatever the gains, so music is never pushed out by effects and a
footstep never takes the slot of dialogue. Within a category, the quieter voice (after attenuation) loses. If the
newcomer ranks lowest, `Play` returns `Object()` and nothing plays.

The loser's owner hears `Reason::Superseded`. The mixer takes the slot over in silence.

A voice whose gain is below a 16-bit step (silent, for example past its outer radius) is not decoded. Its decoder is
skipped forward instead, so it stays in time and costs almost nothing, but it still holds a slot.

## 6. The low-pass filter

Each voice can run through one second-order low-pass section (`Filter`, Butterworth, `kQuality` = 0.7071).

- A cutoff of 0, or at or above `kMixerCutoff` (20 kHz), means open: no filter runs and nothing is paid.
- `Filter::kMinimumCutoff` = 20 Hz is the lowest the section accepts.
- `SetPlaybackCutoff` sets a target. The cutoff glides toward it (about 30 ms), so occlusion fades in and out instead
  of switching.
- `Emitter::SetCutoff` sets where a spatial playback starts.

Typical values: 800 to 2000 Hz for a wall, 300 to 500 Hz for underwater.

## 7. Sounds and the `.snd` format

### 7.1 Loading

`.snd` files are content resources. Load them through the content service from a mount (see `content.md`):

```cpp
const Retainer<ZyContent::Service> Content = GetService<ZyContent::Service>();
const Retainer<ZyAudio::Sound>     Bell    = Content->Load<ZyAudio::Sound>("Resources://Audio/Bell.snd");
```

The load runs in the background. Check `Bell->HasCompleted()` (or `HasFailed()`) before passing it to `Play`.

`Sound` getters:

| Method | Value |
|---|---|
| `GetFrames()` | Total frames. |
| `GetFrequency()` | Hertz (always 48000 for a loaded `.snd`). |
| `GetStride()` | Channels: 1 or 2. |
| `GetDuration()` | Microseconds. |
| `Decode()` | A new decoder at the start of the stream. The service calls it once per playback. |

### 7.2 Encodings

| `Encoding` | Storage | Decode cost | Use for |
|---|---|---|---|
| `Linear` | Signed 16-bit samples | None | Short effects that play often. |
| `Adaptive` | IMA ADPCM, 4 bits per sample, blocks of 512 frames | Low | Effects and ambience at a quarter of the size. |
| `Opus` | Opus packets with a length directory | Highest | Music and long ambience. |

### 7.3 File layout

All fields little-endian.

```
UInt32  magic      'ZSND' (bytes 'Z' 'S' 'N' 'D')
UInt16  version    1
UInt8   encoding   0 = Linear, 1 = Adaptive, 2 = Opus
UInt16  stride     channels, 1 or 2
UInt32  frequency  must be 48000
UInt64  frames     total frames
UInt32  size       payload size once unpacked
UInt32  stored     payload bytes that follow
Byte[]  payload    LZ4 data, or the raw payload when stored == size
```

The payload, once unpacked:

| Encoding | Payload |
|---|---|
| `Linear` | `frames * stride` interleaved `SInt16` samples. |
| `Adaptive` | Blocks of `4 + 512 / 2` = 260 bytes per channel, each holding 512 frames. |
| `Opus` | `UInt16` preskip, `UInt16` frames per packet, `UInt32` packet count, then one `UInt16` length per packet, then the packets back to back. |

The loader refuses, with a warning naming the file:

| Case | Warning |
|---|---|
| Wrong magic or version | `'{0}' is not a ZSND file (bad magic)` / `has an unsupported ZSND version` |
| `stride` 0 or more than 2 | `declares {1} channels, which the mixer cannot consume` |
| `frequency` not 48000 | `is baked at {1} Hz, which is not the mixer's rate` |
| Payload fails to unpack | `has an invalid sound payload` |
| Fewer frames in the payload than declared | `declares more frames than its payload holds` |
| An Opus directory that does not fit | `has an Opus directory that does not fit its payload` |
| Unknown encoding | `has an unknown encoding` |

### 7.4 Making `.snd` files

Sounds are baked offline. The Zyphryon Studio baker (a separate repository) reads common source formats, resamples to
48 kHz and writes `.snd`:

```
Studio bake Sources/Bell.wav Content/Audio/Bell.snd --encoding Adaptive
Studio bake Sources/Theme.flac Content/Audio/Theme.snd --encoding Opus --bitrate 96000
```

Its switches: `--encoding` (`Linear`, `Adaptive` or `Opus`, default `Linear`), `--bitrate` (Opus, default 96000) and
`--compressed` (LZ4 when it shrinks the file, on by default). Since the engine never converts rates at load time, a
48 kHz bake is the only requirement any other tool writing `.snd` must meet.

## 8. Worked example: a bell, a theme and a moving campfire

A complete application. It mounts a `Content` folder as `Resources://`, plays a looping theme, rings a flat bell
every 5 seconds, and moves a crackling campfire in a circle around the
listener so it pans left and right.

```
Content/
  Audio/
    Theme.snd       Opus, stereo
    Bell.snd        Adaptive, mono
    Fire.snd        Adaptive, mono
```

```cpp
#include <Zyphryon.Audio/Service.hpp>
#include <Zyphryon.Content/Mount/Disk.hpp>
#include <Zyphryon.Content/Service.hpp>
#include <Zyphryon.Runtime/Entry.hpp>
#include <Zyphryon.Runtime/Kernel.hpp>

class Campfire final : public ZyRuntime::Kernel
{
protected:

    Bool OnInitialize() override
    {
        const Retainer<ZyContent::Service> Content = GetService<ZyContent::Service>();
        Content->AddMount("Resources", Retainer<ZyContent::Disk>::Create("Content"));

        mTheme = Content->Load<ZyAudio::Sound>("Resources://Audio/Theme.snd");
        mBell  = Content->Load<ZyAudio::Sound>("Resources://Audio/Bell.snd");
        mFire  = Content->Load<ZyAudio::Sound>("Resources://Audio/Fire.snd");

        mAudio = GetService<ZyAudio::Service>();
        return true;
    }

    void OnTick(Real64 Delta) override
    {
        if (!mAudio || !mTheme->HasCompleted() || !mBell->HasCompleted() || !mFire->HasCompleted())
        {
            return;   // no device in this build, or the sounds are still loading
        }

        mClock += Delta;

        // The listener's world transform: at the origin, right along +X, up along +Y, forward along +Z.
        // Pass a world transform (an object's or the camera's), never a view matrix.
        mAudio->SetListenerPose(Matrix4x3::Identity());

        if (!mMusic)
        {
            mAudio->SetSubmixVolume(ZyAudio::Category::Music, 0.5f);

            mMusic = mAudio->Play(ZyAudio::Category::Music, mTheme, 1.0f);
            mAudio->SetPlaybackLooping(mMusic, true);
        }

        // The fire circles the listener at four metres, so it is heard on the left, in front, on the right, behind.
        const Real32    Turn  = static_cast<Real32>(mClock) * 0.5f;
        const Matrix4x3 Place = Matrix4x3::FromTranslation(Vector3(Cosine(Turn) * 4.0f, 0.0f, Sine(Turn) * 4.0f));

        if (!mBurning)
        {
            ZyAudio::Emitter Emitter;
            Emitter.SetAttenuation(ZyAudio::Attenuation::Inverse);
            Emitter.SetRadius(1.0f, 20.0f);

            mBurning = mAudio->Play(ZyAudio::Category::Ambient, mFire, 0.8f, AddressOf(Emitter), AddressOf(Place));
            mAudio->SetPlaybackLooping(mBurning, true);

            // An ambient loop is the first to go when the voices run out, so play it again when it does.
            mAudio->Subscribe(mBurning, [this](ZyAudio::Object, ZyAudio::Reason Reason)
            {
                mBurning = ZyAudio::Object();
            });
        }
        else
        {
            mAudio->SetPlaybackPose(mBurning, Place);
        }

        // A flat interface ring every five seconds.
        if (mClock >= mNextBell)
        {
            mNextBell = mClock + 5.0;
            mAudio->Play(ZyAudio::Category::Interface, mBell, 0.9f);
        }
    }

    void OnTerminate() override
    {
        if (mAudio)
        {
            mAudio->Stop(mMusic);
            mAudio->Stop(mBurning);
        }
    }

private:

    Retainer<ZyAudio::Service> mAudio;
    Retainer<ZyAudio::Sound>   mTheme;
    Retainer<ZyAudio::Sound>   mBell;
    Retainer<ZyAudio::Sound>   mFire;
    ZyAudio::Object            mMusic;
    ZyAudio::Object            mBurning;
    Real64                     mClock    = 0.0;
    Real64                     mNextBell = 0.0;
};

ZY_APPLICATION(Campfire)
```

What happens, step by step:

1. `OnInitialize` runs after every service is up. The three loads start in the background.
2. Each `OnTick` returns early until the three sounds have loaded.
3. The theme starts flat, in the `Music` submix at half volume, and loops. Nothing ever supersedes it, since music
   ranks above every other category.
4. The fire starts as a spatial `Ambient` playback. Each frame, `SetPlaybackPose` moves it; the mixer recomputes its
   gains every block and ramps toward them, so the circle sounds smooth.
5. The bell is fire-and-forget. Its handle is dropped. The service keeps the sound alive until the ring ends.
6. If 256 other sounds ever crowd the mixer, the fire is the first to be superseded. Its callback clears the handle
   and the next tick starts it again.

To muffle the fire while it is behind the listener, add after `SetPlaybackPose`:

```cpp
mAudio->SetPlaybackCutoff(mBurning, Sine(Turn) < 0.0f ? 1200.0f : 0.0f);   // muffled while z < 0
```

### 8.1 Following an entity of the scene

With the scene module (see `scene.md`), keep the playback handle in a component and move it from a system that
reads the entity's world transform. The app declares both components itself:

```cpp
struct Spot final
{
    /// The world transform of the entity.
    Matrix4x3 World = Matrix4x3::Identity();
};

struct Crackle final
{
    /// The playback of the entity's loop, or invalid while none plays.
    ZyAudio::Object Playback;
};

Scene.Declare<Spot>("Demo::Spot");
Scene.Declare<Crackle>("Demo::Crackle");

// Present is a phase the app made earlier with Scene.Phase("Demo::Present"), and Audio its audio service.
Scene.System("Demo::Audio::Follow", Present)
    .Each([Audio](ConstRef<Spot> Spot, ConstRef<Crackle> Crackle)
    {
        if (Crackle.Playback)
        {
            Audio->SetPlaybackPose(Crackle.Playback, Spot.World);
        }
    });
```

Stop the playback when the entity goes, with a `Removed<Crackle>` reader on a component declared `.Keeps()`, so the
callback still sees the handle (see `scene.md`).

## 9. Common mistakes and limits

| Mistake | What happens | Do instead |
|---|---|---|
| `Play` before the sound finished loading | Asserts in debug. | Check `HasCompleted()`, or wait for the content callback. |
| A `.snd` at 44.1 kHz or with more than 2 channels | The loader refuses it with a warning; nothing plays. | Bake at 48 kHz, mono or stereo. |
| An emitter without a transform, or the reverse | Asserts. | Pass both for a spatial playback, neither for a flat one. |
| A volume above 1 to make something louder | Asserts. | Volumes are gains in [0, 1]. Lower the others or the submix instead. |
| A stereo sound played spatially, expecting stereo width | It is folded to mono and panned. | Bake spatial sources mono; play stereo beds flat. |
| A spatial loop that never moves although its entity does | Sources do not follow anything. | `SetPlaybackPose` each frame it moves. |
| Hearing a sound past its outer radius | Every curve reaches silence there. | Raise the outer radius. |
| Never setting the listener | The listener stays at the origin facing +Z. | `SetListenerPose` from the camera each frame. |
| `Subscribe` after the playback already ended | Never called; the entry lingers. | Subscribe right after `Play`, or `Unsubscribe`. |
| Keeping a handle after it ended and comparing it later | The slot may be reused with a new generation. | Clear it in the end callback; never compare raw slots. |
| Assuming `Play` always succeeds | It returns `Object()` when crowded out or the queue is full. | Check the handle before storing it. |
| Calling `GetService<ZyAudio::Service>()` in a headless build | Returns null. | Check before use. |
| Thousands of commands in one frame | The 4096-command queue fills and later commands are dropped. | Move sources once per frame, not per sub-step. |

Limits worth knowing:

- One output device, stereo only. No surround, no capture (microphone) input.
- No streaming from disk: a sound lives whole in memory, compressed by its encoding.
- No pitch, playback rate or seek in the service API. `Decoder::Seek` exists but the service never exposes it.
- No Doppler, reverb, HRTF or per-submix effects. Five fixed submixes.
- No sample-accurate scheduling: a `Play` starts at the mixer's next pass after the command arrives.
- `Resampler` is a utility class; nothing in the playback path converts rates.

## 10. Where it lives

| Path | Holds |
|---|---|
| `Engine/Source/Zyphryon.Audio/Service.hpp`, `Service.cpp` | The service: device, volumes, listener, play and playback control, slot ranking. |
| `Engine/Source/Zyphryon.Audio/Types.hpp` | Constants, `Attenuation`, `Category`, `Reason`, `Encoding`, `Description`, `Object`, `Completion`. |
| `Engine/Source/Zyphryon.Audio/Mixer.hpp`, `Mixer.cpp` | The command queue, voice pool, ranking, mixing, master ramp and limiter. |
| `Engine/Source/Zyphryon.Audio/Voice.hpp` | One playback's mixer state. |
| `Engine/Source/Zyphryon.Audio/Emitter.hpp` | The spatial settings value. |
| `Engine/Source/Zyphryon.Audio/Spatializer.hpp`, `Spatializer.cpp` | Distance curves, cones and pan. |
| `Engine/Source/Zyphryon.Audio/Filter.hpp`, `Filter.cpp` | The low-pass section. |
| `Engine/Source/Zyphryon.Audio/Sound.hpp`, `Sound.cpp` | The sound resource. |
| `Engine/Source/Zyphryon.Audio/Decoder.hpp`, `Decoder/Linear.hpp`, `Decoder/Adaptive.hpp`, `Decoder/Opus.hpp` | The decoders. |
| `Engine/Source/Zyphryon.Audio/Loader/SNDLoader.hpp`, `SNDLoader.cpp` | The `.snd` reader. |
| `Engine/Source/Zyphryon.Audio/Resampler.hpp` | The rate converter utility. |
| `Engine/Source/Zyphryon.Audio/Driver.hpp`, `Driver/Windows/Driver.inl`, `Driver/Linux/Driver.inl`, `Driver/Emscripten/Driver.inl` | WASAPI, ALSA and WebAudio backends. |
| `Engine/Source/Zyphryon.Runtime/Kernel.cpp`, `Startup.hpp` | Where the service is created and initialized, and the `Audio` startup section. |
