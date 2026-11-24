# Content

This chapter teaches `Zyphryon.Content` (namespace `ZyContent`): how the engine names files with URIs, how mounts
map a URI schema onto a folder, a package, bytes compiled into the program or a web server, how typed resources are
loaded, cached, awaited, reloaded and unloaded, how loaders turn bytes into resources, and which resource kinds the
engine knows. It leans on `base.md` for `Blob`, `Retainer`, `Delegate`, JSON and the binary `Reader`, on
`runtime.md` for the kernel and the job lanes, on `graphics.md` for the contents of `.tex`, `.msh`, `.mtl`, `.vfx`
and `.mdl`, on `audio.md` for `.snd`, and on `ai.md` for `.bhv`.

## 1. What the content service is for

Every file the engine reads at run time goes through one object, `ZyContent::Service`. It gives you three things:

1. **One way to name a file.** A file is a `Uri` such as `Resources://Model/Wolf.mdl`. The schema (`Resources`) picks
   a **mount**; the path after `://` is looked up inside it. The same URI works whether a folder on disk, a
   package, bytes compiled into the program or a web server holds the file.
2. **Typed, shared, asynchronous resources.** `Load<ZyRender::Model3D>(Uri)` returns a handle at once. The file is
   read on a background lane, parsed by the **loader** registered for its extension, and finished on the main
   thread. Ten callers asking for the same URI share one resource.
3. **Plain file access.** `Read`, `Write`, `Enumerate`, `Copy` and `Delete` work on raw bytes through the same mounts,
   for data that is not a cached resource (your own data files, saves, settings).

There is one service per process. The kernel registers it (see `runtime.md`); it is ticked every frame, and the
tick is where loads finish.

```
Load<T>(Uri) ---------------> cache: one T per URI (status Idle -> Queued)
                                 |
   mount.Read(path)   <----------+   on the IO lane (disk, archive) or inline (memory, remote)
        |
   loader.Load(Service, Scope, Blob)      still on the reading thread: parse into the resource,
        |                                 and Load<> its dependencies into the scope
        v
   parser list (waits until every dependency of the scope has finished)
        |
   Service::OnTick (main thread)
        resource.OnCreate(Host)           GPU upload, audio decode, anything main-thread
        status Completed or Failed
        OnAssetCompleted / OnAssetFailed broadcast
        one-shot subscriptions for that URI fire
```

## 2. URIs

`ZyContent::Uri` (`Uri.hpp`) holds one URL string and splits it on demand.

| Member | For `Resources://Model/Wolf/Wolf.mdl` |
|---|---|
| `GetUrl()` | `Resources://Model/Wolf/Wolf.mdl` |
| `GetSchema()` | `Resources` |
| `GetPath()` | `Model/Wolf/Wolf.mdl` |
| `GetDirectory()` | `Resources://Model/Wolf` |
| `GetFilename()` | `Wolf.mdl` |
| `GetStem()` | `Wolf` |
| `GetExtension()` | `mdl` |
| `IsValid()`, `HasSchema()`, `HasDirectory()`, `HasExtension()` | tests |
| `Hash()`, `Serialize(Archive)`, `OnFormat(Buffer)` | so a `Uri` is a table key, saves, and prints with `{0}` |
| `Uri::Expand(Relative, Parent)` | joins a schema-less path to the parent's directory |

A `Uri` converts implicitly from a string literal, a `Text` and a `Str`, so every API taking `ConstRef<Uri>` takes a
literal: `Content.Load<ZyAudio::Sound>("Resources://Sound/Bell.snd")`.

**Relative paths.** Inside a file that names other files (a model naming its mesh, a material naming its textures),
a path **without** a schema is relative to the folder of the file naming it:

```
Resources://Model/Wolf/Wolf.mdl  names  "Wolf.msh"                      -> Resources://Model/Wolf/Wolf.msh
Resources://Model/Wolf/Wolf.mdl  names  "Resources://Material/Fur.mtl"  -> unchanged
```

**Traps**

- The cache is keyed by the **whole URL string**. `Resources://Model/Wolf.mdl` and `resources://Model/Wolf.mdl` are
  two different keys, and so are `Model/Wolf.mdl` and `Model//Wolf.mdl`. Mount schemas are matched exactly too:
  `resources://` finds no mount. Build every URI from one constant or one naming rule.
- The URI helpers return **empty** when their delimiter is absent (they are built on `StrAfterLast` and friends, see
  `base.md`). A URI with no schema reaches no mount: the read fails and the log says
  `Content: Failed to read asset '<path>' = '<result>'`.
- `Uri::Expand` does not collapse `..`. A disk mount hands `Model/Wolf/../Fur.tex` to the operating system, which
  resolves it; the memory mount looks the path up literally and fails. Prefer paths that go down, not up.
- The loader is chosen by the **extension**, so a file must keep its engine extension (`.mdl`, `.tex`, ...). An
  unknown extension logs `Content: Unknown file format '<ext>'` and the resource fails.

## 3. Mounts

A mount (`Mount.hpp`) implements `IsAsynchronous`, `Enumerate`, `Read`, `Write`, `Copy` and `Delete` for one schema.

| Mount | Header | Backing | Asynchronous | Writes |
|---|---|---|---|---|
| `ZyContent::Disk(Text Path)` | `Mount/Disk.hpp` | a host folder (`Path` joined with `/` and the URI path) | no (runs on the IO lane) | yes |
| `ZyContent::Memory(ConstSpan<Entry>)` | `Mount/Memory.hpp` | bytes compiled into the binary; each `Entry` is a `Text Path` and a `ConstSpan<Byte> Data` | yes (inline) | refused (`Denied`) |
| `ZyContent::Archive(Text Path, Signer Key = Signer())` | `Mount/Archive.hpp` | a package file (`Package.hpp`, magic `ZPAK`) | no | refused |
| `ZyContent::Remote(Text Address)` | `Mount/Remote.hpp` | HTTP fetches; web builds only | yes | refused |

```cpp
Ref<ZyContent::Service> Content = * GetService<ZyContent::Service>();

Content.AddMount("Resources", Retainer<ZyContent::Disk>::Create("Content"));
Content.RemoveMount("Resources");
```

A synchronous mount (`Disk`, `Archive`) runs every operation as a job on `ZyJob::Lane::IO`; an asynchronous one
(`Memory`, `Remote`) runs it inline and answers when it can. Outside a web build, `Remote` refuses every read with
`Denied`. A mount's buffers (`Memory` entries) must outlive the mount.

### 3.1 Choosing schemas

The engine names only one schema itself, `Embedded://`. Every other schema is yours. A common layout:

| Schema | Mount | Points at |
|---|---|---|
| `Resources://` | `Disk` in development, `Archive` in a shipped build, `Remote` on the web | the game's baked content |
| `Save://` | `Disk` over `Filesystem::GetDataFolder(Organization, Application)` | the player's saves and settings |
| `Embedded://` | `Memory`, through the generated `ZyRegisterEmbedded` | files compiled into the program |

```cpp
Bool OnInitialize() override
{
    Ref<ZyContent::Service> Content = * GetService<ZyContent::Service>();

    ZyRegisterEmbedded(Content);                                        // Embedded://

#if defined(ZY_PLATFORM_WEB)
    Content.AddMount("Resources", Retainer<ZyContent::Remote>::Create("Content"));
#else
    const Filesystem::Path Root = Filesystem::GetRootFolder();
    Content.AddMount("Resources", Retainer<ZyContent::Disk>::Create(Filesystem::Path::Join(Root, '/', "Content")));
#endif

    const Filesystem::Path Saves = Filesystem::GetDataFolder("Acme", "Hello");

    if (!Saves.IsEmpty())
    {
        Filesystem::MakeAll(Saves);
        Content.AddMount("Save", Retainer<ZyContent::Disk>::Create(Saves));
    }
    return true;
}
```

The schema is the name you pass to `AddMount`, without `://`.

### 3.2 `Embedded://`

`Embedded://` serves the engine's own resources (its shaders under `Embedded://Shader/` and its built-in techniques
such as `Embedded://Technique/PostEffect/Bloom.vfx`) and anything you register. Folders are registered at CMake
configure time with `ZyRegisterResources(TARGET ... DIRECTORY ... [PREFIX ...])` and materialized into the
application by `ZyApplyResources(<target>)`, which generates `<target>.Embedded.cpp` with a `Memory` mount and
`void ZyRegisterEmbedded(Ref<ZyContent::Service> Service)`. Call that function once in `OnInitialize`; the renderer's
post effects need it.

```cmake
# Your own files, served as Embedded://Hello/...
ZyRegisterResources(TARGET ${PROJECT_NAME} DIRECTORY "Embedded" PREFIX "Hello")
ZyApplyResources(${PROJECT_NAME})
```

Register before `ZyApplyResources` runs. Adding a file to an embedded folder needs a reconfigure, not just a
rebuild: the bytes are read when CMake configures.

### 3.3 Packages

A package is one file holding a folder's files, read by name where they lie in memory. Build one with
`ZyContent::Package::Pack(Folder, Output, Key = Signer())`, which writes the same bytes for the same folder and
leaves hidden files out, and mount it with `ZyContent::Archive(Path, Key)`. A package sealed under a key opens only
for a reader holding that key. Lookups inside a package ignore case and the direction of slashes.

```cpp
// A build step or a tool: pack the baked content once.
ZyContent::Package::Pack("Build/Content", "Build/Content.zpak");

// The shipped program.
Content.AddMount("Resources", Retainer<ZyContent::Archive>::Create("Content.zpak"));
```

## 4. Resources

### 4.1 The base classes

`ZyContent::Resource` (`Resource.hpp`) is the base of every loadable thing. It derives from `Retainable<Resource>`,
so it is held by `Retainer<T>`.

| Member | What it does |
|---|---|
| `GetKey() -> ConstRef<Uri>` | The URI it was loaded from. |
| `GetStatus()` | `Status::Idle`, `Queued`, `Completed` or `Failed`. |
| `HasFinished()` | `Completed` or `Failed`. |
| `HasCompleted()`, `HasFailed()` | Each one. |
| `GetFootprint()`, `SetFootprint(Bytes)` | The memory it accounts for. |
| `GetPolicy()`, `SetPolicy(Policy)` | `Managed` (shared through the cache) or `Exclusive` (owned by one parent, e.g. a material written inline in a model). |
| `virtual Bool OnCreate(Ref<Host>)` | Main-thread finish (GPU upload ...). Returning `false` fails the resource. |
| `virtual void OnDelete(Ref<Host>)` | Main-thread release. |
| `virtual void OnReload(Ref<Host>)` | Before a reload, to reload what it references. |

`ZyContent::AbstractResource<T>` adds the per-type cache: `T::GetCache()` is a process-wide `ZyContent::Cache<T>`,
one per resource type, keyed by URI.

### 4.2 The resource kinds the engine loads

| Extension | Resource class | Module | Stored as | Loader |
|---|---|---|---|---|
| `.tex` | `ZyGraphic::Image` | Graphic | baked binary, magic `ZTEX` | `TEXLoader` |
| `.msh` | `ZyGraphic::Mesh` | Graphic | baked binary, `ZMSH` | `MSHLoader` |
| `.mtl` | `ZyGraphic::Material` | Graphic | JSON | `MTLLoader` |
| `.vfx` | `ZyGraphic::Technique` | Graphic | JSON naming shader stages | `VFXLoader` |
| `.glsl`, `.hlsl` | `ZyGraphic::Shader` | Graphic | shader source | `SHDLoader` |
| `.mdl` | `ZyRender::Model3D` | Render | JSON naming mesh, skeleton, materials, clips | `MDLLoader` |
| `.skl` | `ZyRender::Skeleton3D` | Render | baked binary, `ZSKL` | `SKLLoader` |
| `.anm` | `ZyRender::Animation3D` | Render | baked binary, `ZANM` | `ANMLoader` |
| `.fnt` | `ZyRender::Font` | Render | baked binary, `ZFNT` | `FNTLoader` |
| `.sh2` | `ZyRender::Sheet2D` | Render | JSON | `SH2Loader` |
| `.snd` | `ZyAudio::Sound` | Audio | baked binary, `ZSND` | `SNDLoader` |
| `.bhv` | `ZyAI::Behaviour` | AI | JSON | `BHVLoader`, **not registered by default** |

The graphic, render and audio services register their loaders when they start (`Zyphryon.Graphic/Service.cpp`,
`Zyphryon.Render/Service.cpp`, `Zyphryon.Audio/Service.cpp`). Nothing registers `BHVLoader`; a program that loads
behaviour trees adds it (see `ai.md`). In a headless run, which does not start the graphic, render and audio
services, none of their formats load.

The baked binary formats are produced from source art by the Zyphryon Studio baker; the JSON ones are written by hand
or by your tools (`graphics.md` for `.mtl`, `.vfx` and `.mdl`, `ai.md` for `.bhv`, `audio.md` for sounds).

Your own data files (a level list, a bestiary) are usually **not** resources. Read them as raw bytes with
`Content.Read` and parse them yourself (section 5.3), or give them a loader (section 6) when you want caching,
sharing and reloads.

## 5. The service API

`ZyContent::Service` (`Service.hpp`).

### 5.1 Loading typed resources

```cpp
template<typename T> Retainer<T> Load(ConstRef<Uri> Key, Ptr<Scope> Parent = nullptr);
template<typename T> void        Reload(ConstRetainer<T> Asset);   // only once it has finished
template<typename T> void        Unload(ConstRetainer<T> Asset);   // out of the cache, then OnDelete, once loaded
template<typename T> void        Prune(Bool Force);                 // drop unreferenced T, or every finished T when forced
template<typename T> UInt64      GetMemoryUsage();
```

- `Load` returns at once. The handle is valid immediately; its status is `Queued` until the main-thread tick after
  the file was read and parsed.
- Loading a URI that is already in the cache returns the same object and queues nothing (only an `Idle` resource is
  queued).
- `Parent` is the `Scope` of a resource being parsed. Passing it makes the new resource a dependency, so the parent's
  `OnCreate` runs only after the child has finished, and it expands a relative URI against the parent's folder.
- A failed resource stays in the cache as `Failed`. Loading the same URI again returns the failed one; fix the file
  and `Reload` it.
- Nothing prunes automatically. Resources stay cached until you call `Unload` or `Prune<T>`.

### 5.2 Waiting for a resource

```cpp
using Callback = Delegate<void(Ref<Resource>)>;

void Subscribe(ConstRef<Uri> Key, AnyRef<Callback> Function);   // fires once, when it finishes
void Unsubscribe(ConstRef<Uri> Key);
template<typename Handle> Bool Await(ConstRef<Handle> Asset, AnyRef<Callback> Function);

MulticastDelegate<void(Ref<Resource>)> OnAssetCompleted;   // every resource that completes
MulticastDelegate<void(Ref<Resource>)> OnAssetFailed;      // every resource that fails
UInt32 GetBacklog() const;                                  // loads and lane reads still in flight
```

`Await(Asset, Callback)` returns `true` when the resource has already finished; otherwise it subscribes the callback
and returns `false` (also `false`, with nothing subscribed, for an empty handle). It never calls the callback for an
already-finished resource, so the usual shape is:

```cpp
mBell = Content.Load<ZyAudio::Sound>("Resources://Sound/Bell.snd");

if (Content.Await(mBell, [this](Ref<ZyContent::Resource> Asset) { OnBellReady(); }))
{
    OnBellReady();
}
```

Inside the scene there is a shorter form for a component that names an asset: `Entity::Await<Component>(Content,
Asset)` marks the component changed once the asset lands, so the systems reading it run again (see `scene.md`).

**Traps**

- Subscriptions are one-shot and keyed by URI. `Unsubscribe(Key)` removes **every** callback waiting on that URI,
  including other code's. Do not unsubscribe a URI you share; let the one-shot fire and ignore it.
- "Finished" includes `Failed`. Check `HasCompleted()` inside the callback before using the resource.
- `GetBacklog()` counts in-flight work, so a loading screen can wait for it to reach zero.

### 5.3 Raw file access

```cpp
void Read(ConstRef<Uri> Key, AnyRef<Mount::OnRead> Callback);                    // Delegate<void(Filesystem::Result, Blob)>
void Read(ConstRef<Uri> Key, ZyJob::Lane Lane, AnyRef<Mount::OnRead> Callback);  // the callback runs on that lane
void Write(ConstRef<Uri> Key, AnyRef<Blob> Data, AnyRef<Mount::OnResult> Callback);
void Copy(ConstRef<Uri> Source, ConstRef<Uri> Destination, AnyRef<Mount::OnResult> Callback);  // same schema
void Delete(ConstRef<Uri> Key, AnyRef<Mount::OnResult> Callback);
void Enumerate(ConstRef<Uri> Key, AnyRef<Mount::OnEnumerate> Callback);         // (Result, Sequence<Filesystem::Record>)
```

Reading a JSON file and handling it on the main thread:

```cpp
Content.Read("Resources://Data/Bestiary.json", ZyJob::Lane::Main,
    [this](Filesystem::Result Result, Blob Data)
{
    if (Result != Filesystem::Result::Success)
    {
        LOG_W("Game: no file at 'Resources://Data/Bestiary.json'");
        return;
    }

    const Text Content(Data.GetData<Char>(), Data.GetSize());
    mBestiary.Read(Content);     // see base.md for JSON
});
```

Writing a save (the disk mount creates any missing parent folders first):

```cpp
Writer Output;
Archive<Writer>(Output).Serialize(Ledger);

Content.Write("Save://Slot1.sav", Output.Detach(), [](Filesystem::Result Result)
{
    if (Result != Filesystem::Result::Success)
    {
        LOG_W("Game: the save did not write ({0})", ZyEnum::GetName(Result));
    }
});
```

Listing a folder:

```cpp
Content.Enumerate("Save://Slots", [](Filesystem::Result Result, Sequence<Filesystem::Record> Entries)
{
    for (ConstRef<Filesystem::Record> Entry : Entries)
    {
        LOG_I("Game: found save '{0}' ({1} bytes)", Entry.Name, Entry.Size);
    }
});
```

**Traps**

- The one-argument `Read` calls back **on whatever thread the mount answered on**: an IO worker for the disk mount,
  the calling thread for the memory mount. Touching the scene or game state there is a data race. Use the `Lane`
  overload with `ZyJob::Lane::Main` unless the callback only parses into its own buffers. `Write`, `Copy`, `Delete`
  and `Enumerate` have no lane overload: their callbacks run where the mount answered, so hop to the main thread
  with `ZyJob::Service::Dispatch` before touching shared state (see `runtime.md`).
- A lambda in a callback must stay valid until the call returns. Capturing `this` of an object that may be destroyed
  in between calls into freed memory; keep the object alive until `GetBacklog()` reaches zero, or guard it.
- Writes to `Embedded://` (a memory mount), to an archive and to a remote mount are refused with `Denied`.

## 6. Writing a loader

A new file format is a resource class plus a loader, registered for one or more extensions. The shape, copied from
`Zyphryon.AI/Behaviour.hpp` and `Zyphryon.AI/Loader/BHVLoader.cpp`:

```cpp
// Lore.hpp: a resource holding the lines a village elder can say.
namespace Game
{
    class Lore final : public ZyContent::AbstractResource<Lore>
    {
    public:

        explicit Lore(AnyRef<ZyContent::Uri> Key)
            : AbstractResource { Move(Key) }
        {
        }

        ZY_INLINE void SetLines(AnyRef<Sequence<Str>> Lines)
        {
            mLines = Move(Lines);
        }

        ZY_INLINE ConstSpan<Str> GetLines() const
        {
            return mLines;
        }

    private:

        Sequence<Str> mLines;
    };
}
```

```cpp
// LORLoader.hpp
namespace Game
{
    class LORLoader final : public ZyContent::Loader
    {
    public:

        static constexpr Text kTypes[] = { "lore" };

        Bool Load(Ref<ZyContent::Service> Service, Ref<ZyContent::Scope> Scope, AnyRef<Blob> Data) override
        {
            const Retainer<Lore> Asset = Retainer<Lore>::Cast(Scope.GetResource());

            JsonValue Document;

            if (!Interpret(Data, Asset->GetKey(), "lore", Document))
            {
                return false;
            }

            const JsonObject Root(Document);
            const JsonArray  List = Root.GetArray("Lines");

            Sequence<Str> Lines;

            for (UInt Index = 0; Index < List.GetSize(); ++Index)
            {
                Lines.Append(List.GetString(Index));
            }

            Asset->SetLines(Move(Lines));
            return true;
        }
    };
}
```

```cpp
// In OnInitialize, before anything loads a .lore file.
GetService<ZyContent::Service>()->AddLoader(Game::LORLoader::kTypes, Retainer<Game::LORLoader>::Create());

// Anywhere afterwards.
Retainer<Game::Lore> Elder = Content.Load<Game::Lore>("Resources://Data/Elder.lore");
```

```json
{
    "Lines":
    [
        "The wolves came down from the ridge again.",
        "Ask the healer if you are bitten."
    ]
}
```

The two `protected static` helpers every loader gets:

| Helper | Does |
|---|---|
| `Interpret(ConstRef<Blob> Data, ConstRef<Uri> Key, Text Kind, Ref<JsonValue> Document) -> Bool` | Parses the bytes as JSON and requires an object root; logs `'<key>' is not valid JSON: <error>` or `'<key>' is not a valid <kind>`. |
| `Expect(Ref<Reader> Input, ConstRef<Uri> Key, Text Magic, UInt16 Version) -> Bool` | Reads a four-character magic and a `UInt16` version at the start of a baked binary file, leaving the reader past them; logs `bad magic` or `unsupported version`. |

A binary format of your own follows the same shape with `Expect`:

```cpp
Bool Load(Ref<ZyContent::Service> Service, Ref<ZyContent::Scope> Scope, AnyRef<Blob> Data) override
{
    const Retainer<Game::Heightmap> Asset = Retainer<Game::Heightmap>::Cast(Scope.GetResource());

    Reader Input(Data.GetData(), Data.GetSize());

    if (!Expect(Input, Asset->GetKey(), "ZHGT", 1))
    {
        return false;
    }

    const UInt16 Width  = Input.Read<UInt16>();
    const UInt16 Height = Input.Read<UInt16>();

    // Untrusted bytes: refuse a header the payload cannot back.
    if (Width == 0 || Height == 0 || Input.GetAvailable() < UInt32(Width) * Height * sizeof(Real32))
    {
        return false;
    }

    Sequence<Real32> Samples;
    Samples.Reserve(UInt32(Width) * Height);

    for (UInt32 Index = 0; Index < UInt32(Width) * Height; ++Index)
    {
        Samples.Append(Input.Read<Real32>());
    }

    Asset->SetSamples(Width, Height, Move(Samples));               // your resource's own setter
    return true;
}
```

Rules for a loader:

- `Load` runs **on the thread that read the file** (an IO worker for disk files). Fill the resource's CPU-side data
  only. Anything that needs the main thread (GPU objects, audio voices, scene access) goes in the resource's
  `OnCreate`, which the service calls on the main thread.
- Load dependencies through the scope: `Service.Load<ZyGraphic::Image>(Path, AddressOf(Scope))`. The resource then
  waits for them, and a relative `Path` resolves against the file's folder.
- Return `false` on anything malformed. The resource becomes `Failed` and `OnAssetFailed` fires; nothing crashes.
- Treat file bytes as untrusted: check sizes and counts before reading (the `Reader` already returns zeros past the
  end; `Behaviour` caps its depth and node count).
- Remove a loader with `RemoveLoader(kTypes)` only when nothing will load that format again.

## 7. Naming an asset from a component: `Proxy<T>`

`ZyContent::Proxy<T>` (`Proxy.hpp`) is a URI plus the resource once resolved, the way a component names an asset:

| Member | Does |
|---|---|
| `Proxy(Uri)`, `Proxy(ConstRetainer<T>)` | From a path, or from a resource already held (both explicit). |
| `SetKey(Uri)` | Changes the path and drops the held resource. |
| `GetKey()` | The path (or the held resource's key when no path was set). |
| `GetResource() -> ConstRetainer<T>` | Empty until resolved, then possibly still loading. |
| `Resolve(Ref<Service>)` | Loads the path when it is valid. |
| `Serialize(Archive)` | Saves and loads the URI only. |

Its reflection name is `Content.Asset` (a flat type standing for its URI), and a component exposes it with
`ZyReflection::Field::Asset<&Get, &Set>("Model", ".mdl")`, which tells a tool to offer a file picker filtered by
extension (see `base.md`, section 13).

```cpp
struct Look final
{
    /// The model drawn for the entity.
    ZyContent::Proxy<ZyRender::Model3D> Model;

    template<typename Serializer>
    ZY_INLINE void Serialize(Serializer Archive)
    {
        Archive.Serialize(Model);
    }
};

// A reader of every Look that was added or set resolves its proxy, and Entity::Await marks the Look set again once
// the model has finished, so the same reader runs a second time for it with the model ready.
Scene.System("Game::Look::Resolve", Update)
    .Reads<ZyScene::Set<Look>>()
    .Each([Assets = AddressOf(Content)](ZyScene::Entity Actor, ConstRef<Look> Current)
    {
        if (!Current.Model.GetResource())
        {
            Actor.Get<Look>()->Model.Resolve(* Assets);         // a write through the pointer records nothing
        }

        if (Actor.Await<Look>(* Assets, Actor.Get<const Look>()->Model.GetResource()))
        {
            // Finished (completed or failed): build whatever draws it.
        }
    });
```

Change pulls, `Entity::Get`, `Entity::Await` and which writes are recorded are covered in `scene.md`.

## 8. Reloading and unloading

There is no file watcher. A resource is reloaded when something calls `Service::Reload(Asset)`:

- `Reload` works only on a finished resource. It calls `OnReload` (so a material reloads its images, a technique its
  shaders), releases the resource through `OnDelete`, sets it back to `Idle` and queues it again under the same
  URI. Every holder keeps its `Retainer` and sees the new contents once the reload finishes.
- A development build can bind a key to reload what it is iterating on:

```cpp
if (Input->IsKeyPressed(ZyInput::Key::F5) && mMaterial->HasFinished())
{
    Content.Reload(mMaterial);
}
```

`Unload(Asset)` takes the resource out of the cache and releases it through `OnDelete`; holders keep their `Retainer`,
but the next `Load` of the URI builds a new resource. `Unload` on a resource still loading does not cut it off
mid-load. It waits and unloads it once it lands, finished or failed. `Prune<T>(false)` drops every finished `T` no
one else holds, and `Prune<T>(true)` drops every finished `T`, held or not. Neither touches a resource still loading.

## 9. Common mistakes

1. Using a resource right after `Load` as if it were ready. Check `HasCompleted()` or `Await` it.
2. A URI with the wrong case or a doubled slash: a second cache entry, or no mount at all.
3. A path without a schema outside a file that names it: nothing resolves it.
4. Doing main-thread work inside a loader's `Load`.
5. Handling a one-argument `Read` callback, or a `Write` callback, as if it ran on the main thread.
6. `Unsubscribe(Uri)` wiping out another caller's wait.
7. Expecting a headless run to load models, textures or sounds: its graphic, render and audio services are not
   running, so their loaders are not registered.
8. Writing to `Embedded://`, an archive or a remote mount.
9. Loading `.bhv` files without adding `BHVLoader`.
10. Forgetting `ZyRegisterEmbedded`, so `Embedded://` reaches no mount and the renderer's post effects fail.
11. Expecting `Remote` to fetch outside a web build: it refuses every read with `Denied`.

## 10. Where it lives

| What | Path |
|---|---|
| Service | `Engine/Source/Zyphryon.Content/Service.hpp`, `Service.cpp` |
| URI | `Engine/Source/Zyphryon.Content/Uri.hpp` |
| Resource, cache, scope, proxy | `Engine/Source/Zyphryon.Content/Resource.hpp`, `Cache.hpp`, `Scope.hpp`, `Proxy.hpp` |
| Loader base | `Engine/Source/Zyphryon.Content/Loader.hpp`, `Loader.cpp` |
| Mounts and packages | `Engine/Source/Zyphryon.Content/Mount/` (`Disk`, `Memory`, `Archive`, `Remote`), `Package.hpp` |
| Embedded resources | `CMake/CMakeApplyResources.cmake` (`ZyApplyResources`), `CMake/CMakeRegisterResources.cmake` (`ZyRegisterResources`), `Engine/Resources/` |
| A small JSON loader to copy | `Engine/Source/Zyphryon.AI/Loader/BHVLoader.cpp`, `Engine/Source/Zyphryon.AI/Behaviour.hpp` |
| A loader with dependencies | `Engine/Source/Zyphryon.Render/Model/Loader/MDLLoader.cpp` |
| Loader registration | `Engine/Source/Zyphryon.Graphic/Service.cpp`, `Engine/Source/Zyphryon.Render/Service.cpp`, `Engine/Source/Zyphryon.Audio/Service.cpp` |
