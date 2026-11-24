# Scene (ECS)

This chapter teaches `Zyphryon.Scene` (namespace `ZyScene`): the world, entities and archetypes, declaring
components, reading and writing them, systems and the phases an application declares for them, change lists,
deferral, the clock, and saving worlds. It leans on `base.md` for the engine types (`Ref<T>`, `Sequence<T>`,
`Writer`, `Reader`, `Archive`), on `runtime.md` for the kernel that creates the scene service and ticks it, on
`math.md` for `Vector3` and friends, and on `network.md` for replicating a scene to other processes.

## 1. What the scene is for

The scene is the single store of game state. Everything that exists in a game, a den, a wolf, the player, the score,
is an **entity** holding **components** (plain C++ values). Code that changes the world each frame is a **system**:
a callback the scene runs over every entity that holds the components the callback asks for. Systems are grouped
into **phases**, which the application declares and which run in a fixed order once per frame.

The frame, from the top:

```
ZyRuntime::Kernel::Tick
  -> Kernel::OnTick(Delta)                       the application's own per-frame code
  -> ZyEngine::Subsystem::Host::Tick(Delta)       every service ticks, in the order it was registered
       -> ZyScene::Service::OnTick(Delta)
            Clock.Tick(Delta)                    the world's clock advances (in place, no change recorded)
            World::Progress()                    every phase, in creation order
              phase "Game::Think"                every system in it, in creation order
              phase "Game::Move"
              phase "Game::Clean"
```

`ZyScene::Service` is both the engine subsystem and the `World` (it derives from `ZyEngine::Subsystem` and
`ZyScene::World`). The runtime kernel registers one per process. An application reaches it from its kernel:

```cpp
Ref<ZyScene::Service> Scene = * GetService<ZyScene::Service>();
```

The scene is used one way, and the API is shaped around it:

1. Entities are made from **archetypes** (templates) and their archetype is fixed for life.
2. Their **components change** over time, and those changes are read by other systems (and replicated, see
   `network.md`).
3. Entities **spawn** and **despawn**.

Changing what an entity *is* (its archetype, or its base archetype) is a despawn plus a spawn, never an edit.

## 2. Concepts and names

| Name | What it is |
|---|---|
| `World` | The store: entities, systems, phases, queries, saves. `ZyScene::Service` is the one you use. |
| `Entity` | A small value (world pointer + 64-bit id) naming one entity. Valid only while that entity lives. |
| Handle / ID | `Entity::GetID()`: slot in the low 32 bits, generation in the high 32. A reused slot gets a new generation. |
| World entity | `Scene.GetWorld()`, never destroyed. It holds every **singleton**. |
| Archetype | An entity tagged `Prefab` that no walk sees. Instances read components from it. |
| Part | An archetype under another archetype (tag `Part`), copied beneath every instance of the whole. |
| Component | Any C++ type declared with `Declare<T>()`. A type with no data members is a **tag**. |
| Singleton | A component declared `.Singleton()`. Its one value lives on the world entity. |
| System | A callback a phase runs each frame, built with `Scene.System(name, phase)`. |
| Phase | A named group of systems, made with `Scene.Phase(name)`, run in creation order. |
| Walk | One run of a system or query over the entities it matches. |
| Pull list | The per-component list of Added, Changed and Removed entries a system can read with `Reads<...>()`. |
| Deferral | While a walk runs, structural changes (add, remove, destroy, spawn) wait and are applied when it ends. |
| Module | What `Scene.Import(name, callback)` records. `Scene.Discard(handle)` takes its systems out again. |

Built-in tags and components (`Types.hpp`, `Clock.hpp`):

| Type | Meaning |
|---|---|
| `ZyScene::Prefab` | Carried by every archetype. Keeps it out of walks unless asked (`With<Prefab>` or `Also<Prefab>`). |
| `ZyScene::Asleep` | A sleeping entity. Every walk leaves it out unless asked. Removal readers still see it. |
| `ZyScene::Transient` | Leaves the entity, and everything beneath it, out of every save. |
| `ZyScene::Part` | Marks a part archetype and its copies. |
| `ZyScene::Named` | The name (`Str Value`) an entity is found by among its siblings. Use `SetName`/`GetName`. |
| `ZyScene::Clock` | The world's clock singleton, declared and set by `ZyScene::Service` itself. |

A tag is written `ZyScene::Tag<"Name">` or as any empty struct:

```cpp
using Hostile = ZyScene::Tag<"Game::Hostile">;   // a tag named by a string literal
```

## 3. Entities

### 3.1 The `Entity` API

All methods are `const` (an `Entity` is a name, not the data). Methods that return `Entity` return the same entity
so calls chain.

| Method | Use |
|---|---|
| `UInt64 GetID() const` | The id to store or send. `Scene.GetEntity(Id)` turns it back into an `Entity`. |
| `Bool IsAlive() const` | `false` once destroyed. Always check an `Entity` you stored before using it. |
| `Bool IsArchetype() const`, `Bool IsPart() const` | What kind of entity it is. |
| `void Destruct() const` | Destroys it and everything beneath it, at the end of the running walk if there is one. |
| `Entity Set(Value) const` | Adds the component or writes over the one it holds. Records Added or Changed. |
| `Entity Add<T>() const` | Adds a default `T` unless it holds or inherits one. The usual way to put a tag. |
| `Entity Remove<T>() const` | Takes off a component the entity holds itself. It then reads its archetype's, if any. |
| `Bool Has<T>() const` | Carries `T`, held **or inherited** from the archetype. |
| `Bool Owns<T>() const` | Holds `T` **itself**, not inherited. |
| `Ptr<T> Get<T>() const` | Writable pointer to its **own** value, `nullptr` when it only inherits it. |
| `Get<const T>() const` | Read pointer, own or inherited. `nullptr` when missing (and always for a tag). |
| `T GetOrDefault<T>() const` | A copy of the value (own or inherited) or a default `T`. |
| `Ref<T> Ensure<T>() const` | Its own value, adding a default or copying the inherited one first. Never inside a walk. |
| `Entity Modify<T>(Callback) const` | Edits part of a component in place and records a change. Safe inside a walk. |
| `Entity Notify<T>() const` | Records a change after you wrote the value through a pointer yourself. |
| `Bool Await<T>(Content, Asset) const` | `true` when a resource has loaded, otherwise calls `Notify<T>()` once it does. |
| `Bool Attach(Entity Parent) const` | Moves it under a new parent (or to the root with `Entity()`). |
| `Entity GetParent() const`, `Bool IsParent() const` | Hierarchy reads. |
| `void Children(Callback) const`, `void Descendants(Callback) const` | Visit what hangs beneath. |
| `Entity Find<T>(Bool Inclusive = false) const` | Nearest ancestor carrying `T`. |
| `Bool IsDescendantOf(Entity, Bool Inclusive = false) const` | Up-the-chain test. |
| `Entity GetArchetype() const` | The archetype it was made from, fixed for its whole life. |
| `void Heirs(Callback) const` | Every entity made straight from this archetype. |
| `Bool SetName(Text) const`, `Text GetName() const` | Name among siblings. `SetName` fails if a sibling holds it. |
| `Entity GetAssembly() const` | The outermost whole a part was built into. |
| `void Each(Callback) const` | Visits every component it holds itself, as `ZyScene::Component`. |

Most methods also have an overload taking a runtime `ZyScene::Component` instead of a template type (`Add(Type)`,
`Remove(Type)`, `Has(Type)`, `Owns(Type)`, `Get(Type)`, `Ensure(Type)`, `Notify(Type)`), for tools that handle
components they do not know at compile time.

### 3.2 Own versus inherited values

An instance reads components its archetype **lends** without copying them. This decides how you read and write:

```cpp
// The archetype "Wolf" carries a Prowl declared .Inheritable(). A fresh wolf only inherits it.
const ZyScene::Entity Wolf = Scene.Spawn(Scene.GetArchetype("Wolf"), ZyScene::Entity());

Wolf.Has<Game::Prowl>();                    // true, inherited counts
Wolf.Owns<Game::Prowl>();                   // false, the wolf holds no copy yet
Wolf.Get<Game::Prowl>();                    // nullptr, a writable pointer never points into the archetype
Wolf.Get<const Game::Prowl>();              // the archetype's value
Wolf.GetOrDefault<Game::Prowl>();           // a copy of the archetype's value

Wolf.Ensure<Game::Prowl>().SetSpeed(9.0f);  // copies the archetype's value onto the wolf, then writes it
```

Whether an archetype's component is lent or copied depends on its declaration: `.Inheritable()` (or `.Authored()`)
lends it, anything else is copied onto each instance when it spawns. `.Local()` components never reach an instance.

### 3.3 Hierarchy

Every entity can hang under a parent. A parent is how a game groups things that come and go together (a den and its
wolves, a level and everything in it): destroying the parent destroys the subtree, and saving the parent saves it.
Names are unique among siblings, so `SetName` and `Attach` refuse a clash. Root entities may share names.

```cpp
if (!Pup.Attach(Den))
{
    LOG_W("Game: the den already holds a child named '{0}'", Pup.GetName());
}

Den.Children([](ZyScene::Entity Child)
{
    Child.Add<Game::Hostile>();          // the callback may detach or destroy the child
});

// The den a wolf belongs to, found up the chain.
const ZyScene::Entity Home = Wolf.Find<Game::Lair>();
```

## 4. Archetypes and spawning

### 4.1 Making archetypes

An archetype is an entity no walk sees, which instances are made from. An application makes its archetypes at start
(by hand, or by loading a save that holds them, section 14) and finds them by name later:

```cpp
// Storage, which World and Service derive from.
Entity CreateArchetype(Entity Base, Entity Parent);   // Base: archetype it reads from. Parent: makes it a part.
Entity CloneArchetype(Entity Source, Entity Parent);

// World.
Entity GetArchetype(Text Name);                       // a ROOT archetype by its Named value
void   ForEachArchetype(Callback);                    // every archetype, parts included
```

```cpp
const ZyScene::Entity Wolf = Scene.CreateArchetype(ZyScene::Entity(), ZyScene::Entity());
Wolf.SetName("Wolf");
Wolf.Set(Game::Prowl(4.0f));
Wolf.Add<Game::Hostile>();

// An archetype that reads from Wolf, and overrides one value.
const ZyScene::Entity Alpha = Scene.CreateArchetype(Wolf, ZyScene::Entity());
Alpha.SetName("Alpha");
Alpha.Set(Game::Prowl(6.0f));
```

`CreateArchetype` places the archetype at once, even during a walk (but asserts inside a spread walk).
`GetArchetype` asserts inside a spread walk too.

### 4.2 `World::Spawn`

```cpp
template<typename... Types>
Entity Spawn(AnyRef<Types>... Values);                                   // a plain root entity

template<typename... Types>
Entity Spawn(Entity Archetype, Entity Parent, AnyRef<Types>... Values);  // from an archetype, under a parent
```

The entity lands in one move with its archetype's copied components, its parts, its own values and its parent.
Inside a walk the slot is taken at once (the returned `Entity` is valid and can be stored) but it joins queries only
when the walk ends.

```cpp
const ZyScene::Entity Den = Scene.Spawn(Game::Lair { Vector3(0.0f, 0.0f, 0.0f), 8.0f });

// The values given are the ones the instance holds itself, over the archetype's.
const ZyScene::Entity Pup = Scene.Spawn(Scene.GetArchetype("Wolf"), Den, Game::Prowl(2.0f), ZyScene::Transient());
```

Despawn with `Entity.Destruct()`. A common pattern is a "dispose" tag that any system may add and one system in the
last phase destroys, so every system in between still sees the entity this frame (section 15 does this).

### 4.3 Changing what an entity is

The archetype of an entity, and the base of an archetype, are fixed when it is made. There is no `SetArchetype`.
To turn a pup into an alpha, despawn the pup and spawn the new one, carrying over what should survive:

```cpp
void Game::Pack::Promote(Ref<ZyScene::Service> Scene, ZyScene::Entity Pup)
{
    const ZyScene::Entity Parent   = Pup.GetParent();
    const Game::Position  Position = Pup.GetOrDefault<Game::Position>();

    Pup.Destruct();
    Scene.Spawn(Scene.GetArchetype("Alpha"), Parent, Position);
}
```

Adding and removing components on a living entity is fine and common (tags such as `Hostile`, a `Bleeding`
marker). What is not supported is swapping the template it reads from.

## 5. Components

### 5.1 What a component type needs

Any default-constructible, movable C++ type is a component. What else it needs depends on how it is declared:

| Needs | When |
|---|---|
| `static constexpr Text kName` | Optional. The name saves know it by, used when `Declare` is given no name. |
| `template<typename Serializer> void Serialize(Serializer Archive)` | When `.Serializable()` or `.Authored()`. Tags need none. |
| Copy constructor | When it sits on an archetype and is copied to instances. |
| `static constexpr auto OnDescribe()` | When `.Reflect()` lays its fields out for tools. |
| `static void OnDeclare(Ref<ZyScene::World> Scene)` | Lets `Scene.Register<A, B, C>()` declare several at once. |

A component that keeps its declaration on the type:

```cpp
#pragma once

#include <Zyphryon.Scene/World.hpp>

namespace Game
{
    /// \brief Represents how fast a wolf roams.
    class Prowl final
    {
    public:

        /// \brief Constructs a prowl at four metres per second.
        ZY_INLINE Prowl()
            : mSpeed { 4.0f }
        {
        }

        /// \brief Constructs a prowl with a speed.
        ///
        /// \param Speed The speed, in metres per second.
        ZY_INLINE explicit Prowl(Real32 Speed)
            : mSpeed { Speed }
        {
        }

        /// \brief Sets how fast the wolf roams.
        ///
        /// \param Speed The speed, in metres per second.
        ZY_INLINE void SetSpeed(Real32 Speed)
        {
            mSpeed = Speed;
        }

        /// \brief Gets how fast the wolf roams.
        ///
        /// \return The speed, in metres per second.
        ZY_INLINE Real32 GetSpeed() const
        {
            return mSpeed;
        }

        /// \brief Serializes the state of the object to or from the specified archive.
        ///
        /// \param Archive The archive to serialize the object with.
        template<typename Serializer>
        ZY_INLINE void Serialize(Serializer Archive)
        {
            Archive.Serialize(mSpeed);
        }

    public:

        /// \brief Declares the component to the scene.
        ///
        /// \param Scene The world the component is declared into.
        ZY_INLINE static void OnDeclare(Ref<ZyScene::World> Scene)
        {
            Scene.Declare<Prowl>("Game::Prowl").Authored().Describe("Game").Reflect();
        }

        /// \brief Lays out the fields of the component for tools.
        ///
        /// \return The fields of the component, in the order they are shown.
        ZY_INLINE static constexpr auto OnDescribe()
        {
            return Array(ZyReflection::Field::Property<&Prowl::mSpeed>("Speed").Between(0.0f, 20.0f));
        }

    private:

        Real32 mSpeed;
    };

    /// \brief Represents a tag for a body that attacks on sight.
    struct Hostile final
    {
        /// The name the tag registers under.
        static constexpr Text kName = "Game::Hostile";

        /// \brief Declares the tag to the scene.
        ///
        /// \param Scene The world the tag is declared into.
        ZY_INLINE static void OnDeclare(Ref<ZyScene::World> Scene)
        {
            Scene.Declare<Hostile>();
        }
    };
}
```

Then, once at start:

```cpp
Scene.Register<Game::Prowl, Game::Hostile>();
```

### 5.2 `Declare<T>(Name)` and its options

```cpp
template<typename Type>
Declaration<Type> Declare(Text Name = Text());   // empty name: T::kName, or the compiler's name for T
```

The declaration is a temporary. Chain its options in one statement; the world is told when the statement ends.

| Option | Effect |
|---|---|
| `.Serializable()` | Saves carry it. Needs `Serialize` (tags need none). |
| `.Inheritable()` | Instances read it from their archetype instead of holding a copy. |
| `.Authored()` | `Serializable` + `Inheritable`. The normal choice for data set on archetypes. |
| `.Local()` | Never reaches an instance made from an archetype (archetype-only data). |
| `.Singleton()` | The world entity holds the one value. Walks read it as a field. |
| `.Keeps()` | A removal keeps the value that left, so `Removed<T>` readers can take `(Entity, ConstRef<T>)`. |
| `.Server()` / `.Client()` | Records which side of a session keeps it. The scene records this trait and never acts on it. |
| `.Implies<A, B...>()` | Wherever this is added, `A` and `B` come along. |
| `.Describe(Group, Icon = 0, Policy = Authoring::Anywhere)` | Tells tools where to list it and where it may be put. |
| `.Reflect()` | Lays its fields out for tools from `OnDescribe()`. |

`ZyScene::Authoring` (for `Describe`) is `Derived`, `Archetype`, `Instance`, `World`, `Anywhere`
(`Archetype | Instance`). `Server` and `Client` together assert.

Declarations, for shape:

```cpp
Scene.Declare<Game::Lair>("Game::Lair").Serializable().Keeps();
Scene.Declare<Game::Prowl>("Game::Prowl").Authored();
Scene.Declare<Game::Body>("Game::Body").Authored().Implies<Game::Position>();
Scene.Declare<Game::Tally>("Game::Tally").Serializable().Singleton();
Scene.Declare<Game::Hostile>();                          // a tag, named by its kName
```

Choosing:

- Set on archetypes and shared by every instance: `.Authored()`.
- Runtime state that must survive a save (cooldowns, counters): `.Serializable()` without `Inheritable`.
- Runtime state rebuilt every session (caches, handles to GPU objects): declare it plain.
- One value for the whole world (game rules, score): `.Singleton()`.

A save keeps a component it does not know as opaque bytes, and reads it back once some code declares the type. So a
module that is not loaded does not lose its data in a save.

### 5.3 Singletons

A singleton is declared once and set on the world entity:

```cpp
Scene.Declare<Game::Tally>("Game::Tally").Serializable().Singleton();

if (!Scene.GetWorld().Has<Game::Tally>())
{
    Scene.GetWorld().Set(Game::Tally());
}

const ConstPtr<Game::Tally> Tally = Scene.GetWorld().Get<const Game::Tally>();
```

A system that names a singleton among its fields reads it from the world entity, whatever entity it is walking.
If the singleton has never been set, the walk **does not run at all** (it has nothing to hand the callback).

## 6. Writing values, and which writes are seen

Other systems learn about changes through pull lists (section 9). Whether a write lands in a list depends on how it
was made:

| Write | Recorded as a change |
|---|---|
| `Entity.Set(Value)` | Yes: Added when new, Changed when it held one. |
| `Entity.Add<T>()` / `Remove<T>()` / `Destruct()` | Yes: Added / Removed. |
| `Entity.Modify<T>([](Ref<T> Value) { ... })` | Yes: Changed. |
| `*Entity.Get<T>() = ...` then `Entity.Notify<T>()` | Yes, from `Notify`. |
| `*Entity.Get<T>() = ...` alone | **No.** |
| `Ref<T>` / `Ptr<T>` written inside a walk | **No**, unless the system declared `.Writes<T>()`. |

The last rule is deliberate. Marking every walk's writes would make unrelated readers redo their work every frame, so
writes are opt-in. A walk that always writes declares it; a walk that writes only sometimes calls `Notify` on the
entities it changed:

```cpp
// Always writes Position: declare it, and every visited entity's page is marked changed.
Scene.System("Game::Wolf::Roam", mMove)
    .Writes<Game::Position>()
    .Each([](ConstRef<ZyScene::Clock> Clock, ConstRef<Game::Stride> Stride, Ref<Game::Position> Position)
    {
        Position.Value += Stride.Velocity * static_cast<Real32>(Clock.GetDelta());
    });

// Writes only on some frames: tell readers per entity.
Scene.System("Game::Wolf::Howl", mThink)
    .Each([](ZyScene::Entity Wolf, ConstRef<ZyScene::Clock> Clock, Ref<Game::Hunger> Hunger)
    {
        if (Clock.GetAbsolute() >= Hunger.Due)
        {
            Hunger.Due = Clock.GetAbsolute() + 5.0;
            Wolf.Notify<Game::Hunger>();
        }
    });
```

`.Writes<T>()` marks whole storage pages, so a `Set<T>` reader then sees every entity the walk visited, not only
the ones whose value really changed. Use it for walks that change every value they touch.

Replication sends what pull lists record, so this rule is also the rule for what reaches other processes. See
`network.md`.

`Entity::Await<T>(Content, Asset)` is a convenience for resources: it returns `true` when the asset is loaded, and
otherwise calls `Notify<T>()` on the entity once it loads, so a `Set<T>` system runs again (see `content.md`).

## 7. Systems

### 7.1 The builder

```cpp
Builder<System> System(Text Name, UInt32 Phase);   // Name is what profiles show
```

Chain any of these on the temporary, then end with `.Each(callback)`:

| Call | Meaning |
|---|---|
| `.With<A, B...>()` | Matches must carry `A`, `B` (not handed to the callback). Naming `Prefab` or `Asleep` lets only those in. |
| `.Not<A, B...>()` | Matches must neither hold nor inherit `A`, `B`. |
| `.Or<A, B...>()` | Matches must carry at least one of them. |
| `.Also<Prefab>()` / `.Also<Asleep>()` | Lets archetypes or sleepers in alongside everything else. |
| `.Up<T>()` | A `ConstPtr<T>` field reads `T` from the nearest ancestor carrying it. |
| `.Cascade<T>()` | Like `Up`, walking parents before children, spread. For hierarchy transforms. |
| `.Writes<T...>()` | Marks `T` changed for every visited entity (section 6). |
| `.Rate(UInt16 Ticks)` | Runs once every `Ticks` progresses. |
| `.Spread()` | Lets the walk run on the job workers when it pays. |
| `.Reads<List>()` | Turns the system into a reader of a pull list (section 9). |

There is no system handle. A system lives until its module is discarded (section 12) or the world ends.

### 7.2 The callback is the query

The callback's parameters say what the walk matches and how each field is handed:

| Parameter | Field |
|---|---|
| `ZyScene::Entity Actor` (first only) | The entity being walked. |
| `ConstRef<T>` or `T` by value | Required, read, own or inherited. |
| `Ref<T>` | Required, writable, and must be the entity's **own** value. |
| `ConstPtr<T>` | Optional, read. `nullptr` when missing. Read from an ancestor with `.Up<T>()`. |
| `Ptr<T>` | Optional, writable. |
| A singleton type in any form | Read from the world entity. |
| No parameters at all, no terms | Not a walk. The callback is just called once per frame. |

Two consequences matter:

- A `Ref<T>` field matches only entities that **own** `T`. An instance that only inherits `T` from its archetype is
  not visited. Take `ConstRef<T>` to read inherited values, or give each instance its own copy (declare `T` without
  `Inheritable`, so it is copied at spawn).
- The parameter types are the match. There is no separate "query description" to keep in sync.

Systems, for shape:

```cpp
// Optional fields, an ancestor field, a spread walk.
Scene.System("Game::Wolf::Leash", mMove)
    .Up<Game::Lair>()
    .Spread()
    .Each([](ZyScene::Entity           Wolf,
             ConstRef<Game::Position>  Position,
             Ref<Game::Stride>         Stride,
             ConstPtr<Game::Lair>      Lair)
    {
        if (Lair && Position.Value.GetDistance(Lair->Spot) > Lair->Range)
        {
            Stride.Velocity = Vector3::Normalize(Lair->Spot - Position.Value) * Stride.Velocity.GetLength();
            Wolf.Notify<Game::Stride>();
        }
    });

// A tag filter, sleepers let in, a structural change made inside the walk.
Scene.System("Game::Dispose", mClean)
    .With<Game::Dispose>()
    .Also<ZyScene::Asleep>()
    .Each([](ZyScene::Entity Actor)
    {
        Actor.Destruct();
    });

// A plain call once per frame.
Scene.System("Game::Score::Print", mClean)
    .Each([this]
    {
        Print();
    });
```

### 7.3 Spread walks

`.Spread()` lets the scene hand rows to the job workers (`ZyJob::Lane::Compute`) when the measured cost says it pays
(about 12 microseconds of work, `kSpreadSeconds` in `Execution/Walk.hpp`). A cheap walk stays on the main thread. The
callback must be safe to call from many threads at once:

- Write only the fields handed to you, in place.
- `Set`, `Add`, `Remove`, `Destruct`, `Notify` on any entity are allowed. They queue per share and apply in order
  when the walk ends.
- `World::Spawn`, `CreateArchetype`, `CloneArchetype` and `GetArchetype` **assert** in a spread walk. Wrap the
  spawn in `Postpone`, and find archetypes before the walk.
- No shared mutable state without your own synchronization. Randomness must not depend on which worker runs a row:
  derive it from the entity (its `GetID()`), not from a shared generator.
- A spread walk cannot start inside another spread walk.

```cpp
// mPup is the pup archetype, found once at start with Scene.GetArchetype("Wolf").
Scene.System("Game::Den::Breed", mThink)
    .Spread()
    .Each([this](ZyScene::Entity Den, ConstRef<ZyScene::Clock> Clock, Ref<Game::Litter> Litter)
    {
        if (Clock.GetAbsolute() < Litter.Due)
        {
            return;
        }
        Litter.Due = Clock.GetAbsolute() + 30.0;

        // Spawning asserts here, so it waits until the walk is over, in order with the walk's other changes.
        mScene.Postpone([this, Den]
        {
            if (Den.IsAlive())
            {
                mScene.Spawn(mPup, Den, Game::Position(), Game::Stride());
            }
        });
    });
```

## 8. Phases: declaring the frame's order

A phase is a number returned by `World::Phase(Text Name)`. Phases run in the order they were made; systems inside a
phase run in the order they were declared. **The engine declares no phases itself.** An application (or each
module it is built from) declares the ones it needs, once, before it declares systems:

```cpp
UInt32 Phase(Text Name);                 // the new phase, never zero, run after every phase made before it
void   Switch(UInt32 Phase, Bool Active); // skip or resume every system in a phase
void   Progress();                        // run every phase once (the scene service does this each tick)
```

A layout that works for most games, from the top:

| Phase | What runs there |
|---|---|
| `Game::React` | Readers of what was set or removed last frame (input, network, loads) take it in. |
| `Game::Think` | Minds choose what bodies do (behaviour trees, see `ai.md`). |
| `Game::Decide` | Rules decide: timers, spawners, damage. |
| `Game::Move` | Bodies move. |
| `Game::Derive` | World transforms are composed from local ones and parents (`.Cascade<T>()`). |
| `Game::Present` | What draws or sounds reads final positions. |
| `Game::Clean` | Marks are cleared; disposed entities leave. |

```cpp
// In the kernel's OnInitialize, before any system.
mThink   = Scene.Phase("Game::Think");
mMove    = Scene.Phase("Game::Move");
mPresent = Scene.Phase("Game::Present");
mClean   = Scene.Phase("Game::Clean");

Scene.System("Game::Pack::Hunt", mThink) ...
```

Keep the phase numbers where the systems are declared (members of the class that declares them, or a small struct a
game shares between its pieces, or a singleton component holding them). The number of a phase depends only on the
order phases were made, so declare them in one place.

`Switch(Phase, false)` holds a phase. A level editor or a pause menu uses it to stop the phases that decide and
move while the world is shown but not played, so the rules declared there do not run until play resumes. Pausing
time alone is `Scene.SetTimescale(0.0f)` (section 13), which keeps every system running with a zero delta.

Pick the phase by what the system reads and what reads it. A rule that decides damage goes in Decide; a rule that
reacts to a component a network message just set goes in React; something that needs final world positions goes in
Present (or later than Derive).

## 9. Change pulls: `Reads<Added|Set|Removed<T>>`

A system can read only the entities whose component changed, instead of walking every match:

| List | Entries |
|---|---|
| `ZyScene::Added<T>` | Entities that came to see `T` (added, or spawned holding or inheriting it). |
| `ZyScene::Set<T>` | Added entries first, then entities whose `T` was written (as section 6 records). |
| `ZyScene::Removed<T>` | Entities that stopped seeing `T`, destroyed ones included. |

```cpp
// Fields like a walk. Terms (With, Not, Or) filter the entries.
Scene.System("Game::Wolf::OnHunger", mPresent)
    .Reads<ZyScene::Set<Game::Hunger>>()
    .With<Game::Hostile>()
    .Each([](ZyScene::Entity Wolf, ConstRef<Game::Hunger> Hunger)
    {
        LOG_I("Game: wolf {0} howls, {1} time(s)", Wolf.GetID(), Hunger.Howls);
    });

// Removals take (Entity), or (Entity, ConstRef<T>) when T is declared .Keeps().
Scene.System("Game::Den::OnLairRemoved", mClean)
    .Reads<ZyScene::Removed<Game::Lair>>()
    .Each([](ZyScene::Entity, ConstRef<Game::Lair> Lair)
    {
        LOG_I("Game: the den at range {0} is gone", Lair.Range);
    });
```

Rules of reading:

- Each reader has its own position. It sees what was recorded **since it was made**, so a reader declared after
  entities already exist does not see them as Added. Declare readers at start, before the world loads.
- A removal reader is handed an entity that may already be destroyed. Check `IsAlive()` before touching it. Its
  terms are matched against the components the entity had before the change.
- Taking `(Entity, ConstRef<T>)` for a removal of a type not declared `.Keeps()` asserts.
- A reader reads no ancestor (`Up`/`Cascade` assert).
- What the callback changes waits like in any walk, and changes it causes are read on the next run.
- Writing a `Ref<T>` field inside a reader records nothing, the same as in a walk.

This is the scene's whole observer mechanism. There are no callbacks fired on add or remove; a system reading a
pull list in the right phase is the observer.

`ZyScene::Cursor<List>` is the reader object behind `Reads`. It can be constructed directly from the world
(`ZyScene::Cursor<ZyScene::Set<T>>(Scene)`) and drained with `Each(callback)` from code that is not a system
(replication does this). Prefer a system.

## 10. Deferral: `Defer` and `Postpone`

While a walk runs, structural changes (add, remove, destroy, spawn, attach) cannot move entities between storage
chunks under the walk's feet. They queue and apply, in order, when the outermost walk ends. Values written in place
with `Set` on an entity that already holds the component land at once.

| Call | Use |
|---|---|
| `Scene.Defer(Callback)` | Runs the callback with every structural change it makes held until it returns. Batches many changes. |
| `Scene.Postpone(Callback)` | Runs the callback after the running walk ends, in order with its changes, or at once outside a walk. |
| `Scene.Postpone(ConstSpan<T> Values, Callback)` | Same, with a copy of trivially copyable values handed as a span. |

Consequences an author meets:

- An entity spawned inside a walk is valid and storable, but no query sees it until the walk ends.
- `Ensure<T>()` asserts inside any walk (it may move the entity). Use `Set` or `Modify`.
- `Purge<Tag>()` (takes a tag off every entity) asserts inside a walk. Call it from a system with no fields.
- `World::Load` and `Merge` assert inside a walk.
- An entity whose change is queued has its later in-place writes queued behind it, so the order you wrote in holds.

## 11. Queries outside systems

A query walks on demand, from any code, not tied to a phase. Keep it as a member when it is walked often; it keeps
its matches current as chunks are made, instead of matching again:

```cpp
// The types listed are the terms: writable (non-const) ones must be owned, const ones may be inherited.
ZyScene::Query mWolves = Scene.Query<Game::Position, const Game::Prowl>().With<Game::Hostile>();

mWolves.Run([](Ref<Game::Position> Position, ConstRef<Game::Prowl> Prowl)
{
    // ...
});
```

The query builder takes `With`, `Not`, `Or`, `Also` and `Writes` like a system; converting the builder to
`ZyScene::Query` makes the query. A query keeps the fields of the first callback it is run with; running it later
with a different callback signature asserts. `Matches()` returns how many entities it matches now. Structural
changes inside `Run` are held until it ends, like any walk. Queries are moved, never copied.

```cpp
const ZyScene::Query Wolves = Scene.Query<const Game::Prowl>().With<Game::Hostile>();

UInt32 Count = 0;
Wolves.Run([&Count](ConstRef<Game::Prowl>)
{
    ++Count;
});
```

## 12. Modules

`World::Import(Name, Callback)` runs the callback and records every component, phase and system it declares under a
module handle. `World::Discard(Handle)` takes the module's systems and phases out and keeps its components' values
on their entities as the bytes they save to, so importing the module again reads them back. A value that cannot be
saved is lost with its code.

This is what lets an application load and unload a piece of game code (a mod, a hot-reloaded library) without
restarting the world:

```cpp
const UInt32 Package = Scene.Import("Wolves", [&]
{
    const UInt32 Hunt = Scene.Phase("Wolves::Hunt");

    Scene.Register<Game::Prowl, Game::Hostile>();
    Scene.System("Wolves::Chase", Hunt).Each([](ConstRef<Game::Prowl>) { });
});

// Later, before the library holding that code goes away:
Scene.Discard(Package);
```

Imports do not nest (a second `Import` inside the callback asserts).

## 13. The clock

`ZyScene::Clock` is a singleton the scene service declares and sets itself. `Service::OnTick` ticks it in place
before the phases run, so reading it never produces a change.

| Method | Value |
|---|---|
| `Real64 GetDelta() const` | Scaled seconds of the last tick. Use for movement and timers. |
| `Real64 GetAbsolute() const` | Scaled seconds since the clock started. Use for deadlines (`Due = Absolute + Span`). |
| `Real64 GetElapsed() const` | Real seconds, ignoring the timescale and pauses. |
| `Bool IsEvery(Real64 Period) const` | `true` when the last tick crossed a multiple of a real-time period, even while paused. |
| `Real32 GetMultiplier() const` / `SetMultiplier(Real32)` | The timescale. Zero pauses the world. |
| `void Follow(ConstRef<Clock> Leader)` | Eases toward a leader's time (a client following the server). |

On the service: `Scene.GetTime()` (absolute), `Scene.SetTimescale(Real32)`, `Scene.GetTimescale()`.

Read it as a field in a system, or from the world entity elsewhere:

```cpp
Scene.System("Game::Wolf::Hunger", mThink)
    .Each([](ConstRef<ZyScene::Clock> Clock, ConstRef<Game::Hunger> Hunger)
    {
        if (Clock.GetAbsolute() >= Hunger.Due)
        {
            // due: act on it
        }
    });

const ConstPtr<ZyScene::Clock> Clock = Scene.GetWorld().Get<const ZyScene::Clock>();
const Real64 Due = Clock->GetAbsolute() + 3.0;
```

Store deadlines as absolute times (`Real64`), not as counters decremented each frame. They survive saves, rate
changes and timescale changes, and a system can skip entities that are not due.

## 14. Saving and loading

| Method | Use |
|---|---|
| `void Save(Ref<Writer> Output, ConstSpan<Entity> Roots)` | Writes each root and everything beneath it. Pass `GetWorld()` for singletons. |
| `Sequence<Entity> Load(ConstSpan<Byte> Input)` | Makes each entity from the archetype it names. Returns the roots. |
| `Bool Merge(ConstSpan<Byte> Input, Entity Root)` | Reads the first root over an existing entity, keeping its place, parent and archetype. |
| `Sequence<Entity> Migrate(Ref<World> Target, ConstSpan<Entity> Roots, ConstSpan<Component> Exclude)` | Copies trees into another world as a save would. |
| `static Bool Strip(Input, Output, Exclude, Filter)` | Copies a save's bytes without some components, with no world. |

What a save holds: the format is ZSCN version 2 (magic `ZSCN`, a `TYPE` chunk naming each component once, then an
`ENTS` chunk with the records, parents before children). Only components declared `Serializable` (or `Authored`)
are written. Entities tagged `Transient`, and everything beneath them, are left out. Archetypes keep their slot, so
instances in another save find them again by slot. Unknown component names load as opaque bytes.

```cpp
// The world's singletons and one den with everything in it.
const ZyScene::Entity Roots[] = { Scene.GetWorld(), Den };

Writer Output;
Scene.Save(Output, ConstSpan<ZyScene::Entity>(Roots));

// Later, or in another run, after the same components and archetypes are declared.
const Sequence<ZyScene::Entity> Loaded = Scene.Load(ConstSpan<Byte>(Output.GetData(), Output.GetSize()));
```

Writing the bytes to disk and reading them back is `base.md` (files) and `content.md` (mounts).

To keep runtime state out of saves, spawn it `Transient`, or declare the component without `Serializable`.

Archetypes can be saved and loaded the same way: save each root archetype as a root, and load that file before any
save that holds instances of them. Since an instance names its archetype by slot, load archetypes in the same order
every run.

## 15. Worked example: wolves around a den

A complete program. One den, a wolf archetype, four wolves that roam and turn back at the edge of the den's range,
howl when hungry, and leave after three howls; a singleton counts the wolves that left.

```cpp
// Game/Pack.hpp
#pragma once

#include <Zyphryon.Scene/Service.hpp>

namespace Game
{
    /// \brief Represents a tag for a body that attacks on sight.
    using Hostile = ZyScene::Tag<"Game::Hostile">;

    /// \brief Represents a tag for an entity that leaves the world at the end of the frame.
    using Dispose = ZyScene::Tag<"Game::Dispose">;

    /// \brief Represents a den, the spot its wolves roam around and how far they stray.
    struct Lair final
    {
        /// The spot the wolves return to, in world space.
        Vector3 Spot;

        /// The distance they stray from the spot, in metres.
        Real32  Range = 6.0f;

        /// \brief Serializes the state of the object to or from the specified archive.
        ///
        /// \param Archive The archive to serialize the object with.
        template<typename Serializer>
        ZY_INLINE void Serialize(Serializer Archive)
        {
            Archive.Serialize(Spot);
            Archive.Serialize(Range);
        }
    };

    /// \brief Represents where a body is, in world space.
    struct Position final
    {
        /// The point, in metres.
        Vector3 Value;

        /// \brief Serializes the state of the object to or from the specified archive.
        ///
        /// \param Archive The archive to serialize the object with.
        template<typename Serializer>
        ZY_INLINE void Serialize(Serializer Archive)
        {
            Archive.Serialize(Value);
        }
    };

    /// \brief Represents how a body moves.
    struct Stride final
    {
        /// The velocity, in metres per second.
        Vector3 Velocity = Vector3(1.0f, 0.0f, 0.0f);

        /// \brief Serializes the state of the object to or from the specified archive.
        ///
        /// \param Archive The archive to serialize the object with.
        template<typename Serializer>
        ZY_INLINE void Serialize(Serializer Archive)
        {
            Archive.Serialize(Velocity);
        }
    };

    /// \brief Represents a wolf's hunger, the absolute time it howls next and how often it has.
    struct Hunger final
    {
        /// The absolute clock time of the next howl, in seconds.
        Real64 Due   = 0.0;

        /// The howls so far.
        UInt32 Howls = 0;

        /// \brief Serializes the state of the object to or from the specified archive.
        ///
        /// \param Archive The archive to serialize the object with.
        template<typename Serializer>
        ZY_INLINE void Serialize(Serializer Archive)
        {
            Archive.Serialize(Due);
            Archive.Serialize(Howls);
        }
    };

    /// \brief Represents the count of wolves that left, kept on the world.
    struct Tally final
    {
        /// The wolves that left since the world was made.
        UInt32 Wolves = 0;

        /// \brief Serializes the state of the object to or from the specified archive.
        ///
        /// \param Archive The archive to serialize the object with.
        template<typename Serializer>
        ZY_INLINE void Serialize(Serializer Archive)
        {
            Archive.Serialize(Wolves);
        }
    };
}
```

```cpp
// Game/Valley.cpp
#include "Game/Pack.hpp"
#include <Zyphryon.Runtime/Entry.hpp>

namespace Game
{
    /// \brief Represents the program, a valley with one den of wolves.
    class Valley final : public ZyRuntime::Kernel
    {
    protected:

        /// \copydoc ZyRuntime::Kernel::OnInitialize
        Bool OnInitialize() override
        {
            Ref<ZyScene::Service> Scene = * GetService<ZyScene::Service>();

            // The frame's order, declared once, before any system.
            const UInt32 Think   = Scene.Phase("Game::Think");
            const UInt32 Move    = Scene.Phase("Game::Move");
            const UInt32 Present = Scene.Phase("Game::Present");
            const UInt32 Clean   = Scene.Phase("Game::Clean");

            // Components.
            Scene.Declare<Lair>("Game::Lair").Serializable();
            Scene.Declare<Position>("Game::Position").Serializable();
            Scene.Declare<Stride>("Game::Stride").Serializable();
            Scene.Declare<Hunger>("Game::Hunger").Serializable();
            Scene.Declare<Tally>("Game::Tally").Serializable().Singleton();
            Scene.Declare<Hostile>();
            Scene.Declare<Dispose>();

            Scene.GetWorld().Set(Tally());

            // Systems, in the order their phases run.
            Scene.System("Game::Wolf::Howl", Think)
                .Not<Dispose>()
                .Each([](ZyScene::Entity Wolf, ConstRef<ZyScene::Clock> Clock, Ref<Hunger> Hunger)
                {
                    if (Clock.GetAbsolute() < Hunger.Due)
                    {
                        return;
                    }
                    Hunger.Due = Clock.GetAbsolute() + 5.0;
                    ++Hunger.Howls;

                    // The write above changed a field in place, so readers of Hunger are told.
                    Wolf.Notify<Game::Hunger>();

                    if (Hunger.Howls >= 3)
                    {
                        Wolf.Add<Dispose>();
                    }
                });

            Scene.System("Game::Wolf::Roam", Move)
                .Up<Lair>()
                .Writes<Position>()
                .Spread()
                .Each([](ConstRef<ZyScene::Clock> Clock, Ref<Stride> Stride, Ref<Position> Position, ConstPtr<Lair> Lair)
                {
                    Position.Value += Stride.Velocity * static_cast<Real32>(Clock.GetDelta());

                    // Past the edge of the den's range, turn back toward it.
                    if (Lair && Position.Value.GetDistance(Lair->Spot) > Lair->Range)
                    {
                        Stride.Velocity = Vector3::Normalize(Lair->Spot - Position.Value) * Stride.Velocity.GetLength();
                    }
                });

            Scene.System("Game::Wolf::OnHowl", Present)
                .Reads<ZyScene::Set<Hunger>>()
                .Each([](ZyScene::Entity Wolf, ConstRef<Hunger> Hunger)
                {
                    LOG_I("Game: wolf {0} howls ({1})", Wolf.GetID(), Hunger.Howls);
                });

            Scene.System("Game::Wolf::OnLeave", Clean)
                .Reads<ZyScene::Removed<Hunger>>()
                .Each([Scene = & Scene](ZyScene::Entity)
                {
                    Scene->GetWorld().Modify<Tally>([](Ref<Tally> Tally)
                    {
                        ++Tally.Wolves;
                    });
                });

            Scene.System("Game::Dispose", Clean)
                .With<Dispose>()
                .Also<ZyScene::Asleep>()
                .Each([](ZyScene::Entity Actor)
                {
                    Actor.Destruct();
                });

            // The archetype every wolf is made from.
            const ZyScene::Entity Wolf = Scene.CreateArchetype(ZyScene::Entity(), ZyScene::Entity());
            Wolf.SetName("Wolf");
            Wolf.Add<Hostile>();

            // One den with four wolves, each with its own position, stride and hunger.
            const ZyScene::Entity Den = Scene.Spawn(Lair { Vector3(0.0f, 0.0f, 0.0f), 8.0f });

            for (UInt32 Index = 0; Index < 4; ++Index)
            {
                const Real32 Offset = static_cast<Real32>(Index);

                Scene.Spawn(Wolf, Den,
                    Position { Vector3(Offset, 0.0f, 0.0f) },
                    Stride   { Vector3(1.0f + Offset, 0.0f, 0.5f) },
                    Hunger   { 2.0 + Offset, 0 });
            }
            return true;
        }
    };
}

ZY_APPLICATION(Game::Valley)
```

How the target is built and how `ZY_APPLICATION` finds the engine's modules is in `getting-started.md` and
`runtime.md`.

Step by step, what happens:

1. `OnInitialize` runs once, after every service exists. It declares four phases, the components, the singleton's
   value, five systems, one archetype and five entities. Outside a walk, every spawn lands at once.
2. Each frame the kernel ticks the scene service: the clock advances, then the phases run in the order they were
   made.
3. In `Game::Think`, `Howl` walks every entity that owns `Hunger` and has no `Dispose`. When a wolf is due it writes
   in place, calls `Notify`, and on the third howl adds `Dispose`; the add is queued and applied when the walk ends.
4. In `Game::Move`, `Roam` reads the den through `.Up<Lair>()`, moves each wolf and turns it back at the edge. It
   declared `.Writes<Position>()`, so a `Set<Position>` reader (a renderer, replication) sees every wolf moved. It may
   spread over the workers; it touches only its own fields.
5. In `Game::Present`, `OnHowl` is handed only the wolves whose `Hunger` was notified this frame.
6. In `Game::Clean`, `OnLeave` reads last frame's removals first (none on the first frame), then `Dispose` destroys
   the tagged wolves. Next frame, `OnLeave` is handed each destroyed wolf and bumps the singleton with `Modify`,
   which records a change, so a scoreboard reading `Set<Tally>` updates.

`OnLeave` is declared before `Dispose` in the same phase, so it reads removals one frame late. Declare it after
`Dispose` (or in a later phase) to read them the same frame.

## 16. Common mistakes and limits

| Mistake | What happens | Do instead |
|---|---|---|
| Writing a `Ref<T>` field and expecting a `Set<T>` reader or a peer to see it | Nothing is recorded. | `.Writes<T>()` on the system, or `Notify<T>()` per changed entity. |
| `*Entity.Get<T>() = Value;` with no `Notify` | Same: silent write. | `Entity.Set(Value)` or `Entity.Modify<T>(...)`. |
| A walk taking `Ref<T>` over instances that inherit `T` | They are not visited. | `ConstRef<T>` to read, or declare `T` non-inheritable so each instance owns a copy. |
| `Get<T>()` (non-const) on an inheriting instance | Returns `nullptr`. | `Get<const T>()` to read, `Ensure<T>()` (outside walks) to own. |
| Swapping an entity's archetype | No API exists. | Despawn plus spawn. |
| A component that looks up other entities in its methods | Hidden coupling, unsafe in spread walks. | Components hold data and their own behaviour; the system looks entities up and passes them in. |
| Storing `Entity` across frames without checking | The slot may hold another entity later. | `IsAlive()` first. Store `GetID()` for saves and messages. |
| `Spawn` in a `.Spread()` walk | Asserts. | `Scene.Postpone(...)`. |
| `Ensure<T>()` inside a walk | Asserts. | `Set` or `Modify`. |
| A system naming a singleton that was never set | The walk silently never runs. | Set the singleton at start (`GetWorld().Set(...)`) if absent. |
| An `Added<T>` reader declared late | Misses entities that already existed. | Declare at start, or walk existing ones once with a query. |
| Reading a removed entity's other components in a `Removed<T>` reader | It is usually gone. | Declare `T` `.Keeps()` and take `(Entity, ConstRef<T>)`. |
| Running a query with a second callback signature | Asserts. | One query per field set. |
| Two pieces of code declaring the same component name | The registry is keyed by name. | Prefix names (`Game::Lair`). |
| Declaring phases in several places in a varying order | Systems run in a different order between runs. | Declare every phase in one place, once. |
| Expecting the engine to provide phases | `System` needs a phase number; there is none until you make one. | `Scene.Phase(...)` at start. |
| Counting frames for timers | Breaks with timescale and rate changes. | Absolute deadlines on the clock. |

Limits worth knowing: at most 64,510 archetypes per world (slots 1,025 to 65,534, `kMaxCountArchetypes`); entities
start at slot 65,535 (`kMinRangeEntities`); an entity's archetype is fixed; a reader cannot read ancestors; imports do
not nest.

## 17. Where it lives

| Path | Holds |
|---|---|
| `Engine/Source/Zyphryon.Scene/World.hpp` | `World`: `Spawn`, `GetArchetype`, `ForEachArchetype`, `Query`, `System`, `Phase`, `Switch`, `Progress`, `Declare`, `Register`, `Import`, `Discard`, `Save`, `Load`, `Merge`, `Migrate`, `Strip`. |
| `Engine/Source/Zyphryon.Scene/Storage/Storage.hpp` | `GetWorld`, `GetEntity`, `CreateArchetype`, `CloneArchetype`, `Purge`, `Defer`, `Postpone`. |
| `Engine/Source/Zyphryon.Scene/Entity.hpp` | The `Entity` API. |
| `Engine/Source/Zyphryon.Scene/Builder.hpp` | `With`, `Not`, `Or`, `Also`, `Up`, `Cascade`, `Writes`, `Rate`, `Spread`, `Reads`, `Each`. |
| `Engine/Source/Zyphryon.Scene/Execution/Plan.hpp` | How callback parameters become fields. |
| `Engine/Source/Zyphryon.Scene/Execution/Walk.hpp` | Walks, spreading and its thresholds. |
| `Engine/Source/Zyphryon.Scene/Type/Declaration.hpp` | `Declare` options. |
| `Engine/Source/Zyphryon.Scene/Type/Metatype.hpp`, `Type/Reflection.hpp` | `Trait`, `Authoring`, `Side`. |
| `Engine/Source/Zyphryon.Scene/Types.hpp` | Slot limits, `Pull`, `Added`, `Set`, `Removed`, `Tag`, `Prefab`, `Asleep`, `Transient`, `Part`, `Named`. |
| `Engine/Source/Zyphryon.Scene/Query.hpp`, `Cursor.hpp` | On-demand queries and pull-list readers. |
| `Engine/Source/Zyphryon.Scene/Clock.hpp`, `Service.hpp` | The clock and the scene service. |
| `Engine/Source/Zyphryon.Runtime/Kernel.cpp` | Where the scene service is registered and ticked. |
