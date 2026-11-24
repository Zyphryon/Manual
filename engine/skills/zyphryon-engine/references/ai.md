# AI: behaviour trees

This chapter teaches `Zyphryon.AI` (namespace `ZyAI`): a small, generic behaviour-tree runtime. The engine gives you
the tree format, the composite and decorator nodes, a per-agent blackboard and the state that resumes running nodes.
Your application gives it the leaves: the actions and checks that touch your game. It leans on `content.md` for
loading resources and registering loaders, and `base.md` for `Variant`, `Table`, `JsonValue` and `Str`.

## 1. What it is for

A behaviour tree decides what an agent does, one tick at a time. The engine part never looks at your world. It only
walks the tree and asks you, through an interface called `Host`, to run the leaves.

```
Behaviour (.bhv resource)  shared by every agent that thinks the same way, read only
   |
Brain                       one per agent: the running state of every node, a blackboard, a clock
   |  Brain.Tick(Tree, Delta, Host)
   v
Host                        your class: runs a leaf by index, halts one, rolls a chance
```

There is no AI service and no system. The application decides when and how often each brain ticks (every frame, a
few times a second, only when the agent is near).

## 2. Concepts

| Name | What it is |
|---|---|
| `Behaviour` | A content resource holding the tree as a flat list of `Node`s in depth-first order, root first. |
| `Behaviour::Node` | One node: `Type`, `Children`, `Span`, `Count`, `Value`, `Arguments`, `Check`, `Name`. |
| `Behaviour::Kind` | The node kinds (section 3). |
| `Behaviour::Test` | What a `Compare` node checks. |
| `Brain` | One agent's run of a tree: memory per node, a `Blackboard`, the sum of ticked time. |
| `Blackboard` | Named values the leaves and the blackboard nodes share. |
| `Host` | The interface you implement: `Tick`, `Halt`, `Roll`. |
| `Status` | `Idle`, `Success`, `Failure`, `Running`. |
| Leaf | A node of kind `Leaf`. It names a host action (`Name`) and may carry JSON `Arguments`. |
| Fresh | A node ticked when its last status was not `Running`. It starts over instead of resuming. |

## 3. Node kinds

| Kind | Children | Reads | What it does |
|---|---|---|---|
| `Sequence` | 1 or more | | Runs children in order. Fails on the first failure; succeeds when all succeed. Resumes from the running child. |
| `Selector` | 1 or more | | Runs children in order. Succeeds on the first success; fails when all fail. Resumes from the running child. |
| `Priority` | 1 or more | | Like `Selector`, but asks again from the first child every tick. If an earlier child succeeds or runs, the later child left running is stopped (its leaves halted). |
| `Parallel` | 1 or more | `Count` | Ticks every unfinished child each tick. Succeeds once `Count` children succeed (`0`, or `Count` at least the child count, means all). Fails once success can no longer be reached. On finishing, stops the children still running. |
| `Invert` | exactly 1 | | Swaps success and failure. `Running` passes through. |
| `Succeed` | exactly 1 | | Turns the child's failure into success. A running child stays `Running`. |
| `Repeat` | exactly 1 | `Count` | After each success, runs the child again, `Count` times in all, or forever at `0`. Returns `Running` between runs. A failure ends it with failure. |
| `Retry` | exactly 1 | `Count` | After each failure, runs the child again, `Count` times in all, or forever at `0`. A success ends it with success. |
| `Cooldown` | exactly 1 | `Value` | Fails, without running the child, until `Value` seconds after the child last finished. |
| `Timeout` | exactly 1 | `Value` | Runs the child; once it has run `Value` seconds, stops it and fails. |
| `Chance` | exactly 1 | `Value` | When fresh, asks `Host::Roll(Value)`. On a pass runs the child (and keeps running it while it runs); otherwise fails. |
| `Leaf` | none | `Name`, `Arguments` | Calls `Host::Tick` for this node. |
| `Compare` | none | `Key`, `Test`, `Value` | Succeeds when the blackboard entry passes the test, fails otherwise. |
| `Set` | none | `Key`, `Value` | Sets the entry to `Value` (a `Real64`) and succeeds. |
| `Clear` | none | `Key` | Removes the entry and succeeds. |
| `Wait` | none | `Value` | Returns `Running` until `Value` seconds have passed since it started, then succeeds. |

`Test` values: `Exists`, `Equal`, `NotEqual`, `Less`, `LessEqual`, `Greater`, `GreaterEqual`. Every test but
`Exists` compares a number: a `Real64` entry as it is, a `Bool` entry as `0` or `1`. Any other entry type fails the
test, and so does a missing entry.

Time is the brain's own clock: the sum of every `Delta` passed to `Brain::Tick`. A brain ticked less often sees time
in bigger steps; a brain not ticked does not age.

## 4. The `.bhv` format

A `.bhv` file is a JSON object with one `"Root"` node. Each node is an object:

| Key | Type | Used by | Default |
|---|---|---|---|
| `Type` | string, a `Kind` name | every node | `Leaf` |
| `Children` | array of nodes | composites and decorators | none |
| `Count` | integer | `Parallel`, `Repeat`, `Retry` | 0 |
| `Value` | number | `Cooldown`, `Timeout`, `Chance`, `Wait`, `Compare`, `Set` | 0 |
| `Name` | string | `Leaf` (the host action) | required |
| `Arguments` | object | `Leaf` | none |
| `Key` | string | `Compare`, `Set`, `Clear` (the entry) | required |
| `Test` | string, a `Test` name | `Compare` | `Exists` |

The loader refuses the whole file (`'{0}' has an invalid node`) when:

- `Root` is missing or a node is not an object;
- a `Leaf`, `Compare`, `Set` or `Clear` has children, or has no `Name` / `Key`;
- a `Wait` has children;
- a `Sequence`, `Selector`, `Priority` or `Parallel` has none;
- any other kind does not have exactly one child;
- a leaf's `Arguments` is not an object;
- the tree nests deeper than `kMaxDepth` = 64 or holds more than `kMaxNodes` = 65535 nodes.

A complete file, `Content/AI/Guard.bhv`:

```json
{
  "Root": {
    "Type": "Priority",
    "Children": [
      {
        "Type": "Sequence",
        "Children": [
          { "Type": "Compare", "Key": "Health", "Test": "Less", "Value": 25 },
          { "Type": "Leaf", "Name": "Flee", "Arguments": { "Distance": 12.0 } }
        ]
      },
      {
        "Type": "Sequence",
        "Children": [
          { "Type": "Compare", "Key": "Alarm", "Test": "Equal", "Value": 1 },
          {
            "Type": "Timeout",
            "Value": 10.0,
            "Children": [
              { "Type": "Leaf", "Name": "Chase" }
            ]
          },
          { "Type": "Clear", "Key": "Alarm" }
        ]
      },
      {
        "Type": "Sequence",
        "Children": [
          { "Type": "Leaf", "Name": "Walk", "Arguments": { "Speed": 1.5 } },
          { "Type": "Wait", "Value": 2.0 },
          {
            "Type": "Chance",
            "Value": 0.3,
            "Children": [
              { "Type": "Leaf", "Name": "Whistle" }
            ]
          }
        ]
      }
    ]
  }
}
```

Read it top to bottom: flee when hurt; otherwise chase for at most ten seconds when the alarm is up; otherwise walk,
pause two seconds and sometimes whistle. Since the root is a `Priority`, a guard on a walk drops it the tick the
alarm goes up, and the walk's running leaf is halted.

A node with no `Type` is a leaf. A node naming a `Type` that does not exist, such as a misspelt `"Sequnce"`, is
refused and the whole tree fails to load.

### 4.1 Registering the loader

The engine does not register the `.bhv` loader itself. Add it once at startup, before the first load:

```cpp
#include <Zyphryon.AI/Loader/BHVLoader.hpp>

GetService<ZyContent::Service>()->AddLoader(ZyAI::BHVLoader::kTypes, Retainer<ZyAI::BHVLoader>::Create());
```

Then load trees like any resource:

```cpp
const Retainer<ZyAI::Behaviour> Guard = Content->Load<ZyAI::Behaviour>("Resources://AI/Guard.bhv");
```

Wait for `Guard->HasCompleted()` before ticking. `Brain::Tick` on an empty tree returns `Failure` and does nothing.

## 5. The API

### 5.1 `Brain`

```cpp
Brain();
Status              Tick(ConstRef<Behaviour> Tree, Real32 Delta, Ref<Host> Host);
void                Halt(Ref<Host> Host);
void                Reset();
Status              GetStatus(UInt32 Index) const;
Ref<Blackboard>     GetBlackboard();
ConstRef<Blackboard> GetBlackboard() const;
Real64              GetClock() const;
```

- `Tick` adds `Delta` to the clock, walks from the root and returns the root's status. A root that finished
  (`Success` or `Failure`) starts over on the next tick, so a tree loops by itself.
- `Halt` stops every running node and calls `Host::Halt` for every running leaf. Call it when the agent stops
  thinking (it dies, is despawned, is switched to another tree), so your host can let go of what the leaves held.
- `Reset` forgets the running state without calling any halt. The blackboard is kept.
- `GetStatus` gives the status each node was left in, for a debug view. Indices are positions in `GetNodes()`.
- When the tree's nodes are replaced (a hot reload) the brain notices and restarts its state on the next tick.

A `Brain` is a plain value. Keep one per agent, next to whatever else the agent holds.

### 5.2 `Host`

```cpp
class Host
{
public:
    virtual ~Host() = default;
    virtual Status Tick(UInt32 Index, Bool Fresh, Real64 Elapsed, Ref<Blackboard> Board) = 0;
    virtual void   Halt(UInt32 Index) = 0;
    virtual Bool   Roll(Real32 Chance) = 0;
};
```

| Method | Called | Must |
|---|---|---|
| `Tick` | For a leaf, every tick it is reached. `Fresh` is `true` on its first tick of a run, `Elapsed` the seconds since that tick. | Return `Success`, `Failure`, or `Running` to be ticked again next time. |
| `Halt` | For a leaf left `Running` that will not be ticked again (a priority switch, a timeout, a parallel finishing, `Brain::Halt`). | Undo or stop what the leaf started. |
| `Roll` | When a `Chance` node starts. | Return `true` with probability `Chance`. Use your own random source, so runs can be reproduced. |

`Index` is the leaf's position in `Behaviour::GetNodes()`. Read the leaf's `Name` from that node and its arguments
with `Behaviour::GetArguments(Index)` (a JSON object, or `nullptr`).

A host usually maps names to handlers once per tree and keeps the agent it is ticking for in a member set before
each `Tick` call. The host is not stored by the brain, so one host object can serve many agents in turn.

### 5.3 `Blackboard`

```cpp
using Name  = String<13>;
using Value = Variant<Bool, Real64, UInt64, Vector2, Vector3, Name>;

template<typename Type> void           Set(Text Key, Type Data);
template<typename Type> ConstPtr<Type> Find(Text Key) const;           // nullptr when missing or another type
template<typename Type> Type           Get(Text Key, Type Default = Type()) const;
ConstPtr<Value>                        Lookup(Text Key) const;
Bool                                   Has(Text Key) const;
Bool                                   Erase(Text Key);
void                                   Clear();
template<typename Function> void       ForEach(AnyRef<Function> Callback) const;
```

- Keys are stored as `Str32`, so keep them short.
- Values are one of six types. A number is `Real64`, never `Real32`: `Set("Health", 40.0)`. Store entity ids as
  `UInt64`, positions as `Vector3`, short labels as `Name` (up to 13 characters).
- `Compare` reads only `Real64` and `Bool` entries. A `UInt64`, vector or name entry can be tested with `Exists` only.
- `Set` in a tree writes a `Real64`.

The blackboard belongs to the brain, not to the tree. Each agent has its own.

## 6. Worked example: guards that patrol, chase and flee

A standalone application with three guards on a line. It hosts the four leaves the tree above names: `Flee`,
`Chase`, `Walk` and `Whistle`. It writes `Health` and `Alarm` into each guard's blackboard from outside the tree,
the way a game's damage and perception code would.

```cpp
#include <Zyphryon.AI/Brain.hpp>
#include <Zyphryon.AI/Loader/BHVLoader.hpp>
#include <Zyphryon.Content/Mount/Disk.hpp>
#include <Zyphryon.Content/Service.hpp>
#include <Zyphryon.Runtime/Entry.hpp>
#include <Zyphryon.Runtime/Kernel.hpp>

/// \brief Represents one guard: where it is, and how it thinks.
struct Guard final
{
    /// The position along the line, in metres.
    Real32      Position = 0.0f;

    /// The running state of the guard's tree.
    ZyAI::Brain Brain;
};

/// \brief Runs the leaves of the guard tree for whichever guard is being ticked.
class Leaves final : public ZyAI::Host
{
public:

    /// \brief Binds the guard and the tree the next tick runs for.
    void Bind(Ref<Guard> Agent, ConstRef<ZyAI::Behaviour> Tree)
    {
        mAgent = AddressOf(Agent);
        mTree  = AddressOf(Tree);
    }

    ZyAI::Status Tick(UInt32 Index, Bool Fresh, Real64 Elapsed, Ref<ZyAI::Blackboard> Board) override
    {
        ConstRef<ZyAI::Behaviour::Node> Node      = mTree->GetNodes()[Index];
        const ConstPtr<JsonValue>       Arguments = mTree->GetArguments(Index);

        if (Node.Name == "Walk")
        {
            mAgent->Position += static_cast<Real32>(Read(Arguments, "Speed", 1.0)) * kStep;
            return Elapsed >= 3.0 ? ZyAI::Status::Success : ZyAI::Status::Running;
        }
        if (Node.Name == "Chase")
        {
            mAgent->Position += 4.0f * kStep;
            return ZyAI::Status::Running;          // runs until the Timeout above it stops it
        }
        if (Node.Name == "Flee")
        {
            if (Fresh)
            {
                Board.Set("Home", static_cast<Real64>(mAgent->Position) - Read(Arguments, "Distance", 10.0));
            }
            mAgent->Position -= 5.0f * kStep;
            return mAgent->Position <= Board.Get<Real64>("Home") ? ZyAI::Status::Success : ZyAI::Status::Running;
        }
        if (Node.Name == "Whistle")
        {
            LOG_I("Guard: whistles at {0} m", mAgent->Position);
            return ZyAI::Status::Success;
        }

        LOG_W("Guard: no leaf named '{0}'", Node.Name);
        return ZyAI::Status::Failure;
    }

    void Halt(UInt32 Index) override
    {
        LOG_I("Guard: stopped '{0}'", mTree->GetNodes()[Index].Name);
    }

    Bool Roll(Real32 Chance) override
    {
        mSeed = mSeed * 1664525u + 1013904223u;
        return static_cast<Real32>(mSeed >> 8) / 16777216.0f < Chance;
    }

    /// The fixed step every guard thinks at, in seconds.
    static constexpr Real32 kStep = 0.1f;

private:

    /// \brief Reads a number from a leaf's arguments, or a default when the leaf has none or the field is missing.
    static Real64 Read(ConstPtr<JsonValue> Arguments, Text Key, Real64 Default)
    {
        if (Arguments)
        {
            if (const ConstPtr<JsonValue> Field = Arguments->GetObject().Find(Key); Field && Field->IsNumber())
            {
                return Field->GetNumber<Real64>();
            }
        }
        return Default;
    }

    Ptr<Guard>                    mAgent = nullptr;
    ConstPtr<ZyAI::Behaviour>     mTree  = nullptr;
    UInt32                        mSeed  = 12345;
};

class Watch final : public ZyRuntime::Kernel
{
protected:

    Bool OnInitialize() override
    {
        const Retainer<ZyContent::Service> Content = GetService<ZyContent::Service>();
        Content->AddMount("Resources", Retainer<ZyContent::Disk>::Create("Content"));
        Content->AddLoader(ZyAI::BHVLoader::kTypes, Retainer<ZyAI::BHVLoader>::Create());

        mTree = Content->Load<ZyAI::Behaviour>("Resources://AI/Guard.bhv");

        for (Ref<Guard> Each : mGuards)
        {
            Each.Brain.GetBlackboard().Set("Health", 100.0);
        }
        return true;
    }

    void OnTick(Real64 Delta) override
    {
        if (!mTree->HasCompleted())
        {
            return;
        }

        // Think ten times a second, whatever the frame rate.
        for (mPending += Delta; mPending >= Leaves::kStep; mPending -= Leaves::kStep)
        {
            mClock += Leaves::kStep;

            // The outside world: an alarm at 5 s for the first guard, a wound at 12 s for the second.
            if (mClock >= 5.0 && mClock < 5.0 + Leaves::kStep)
            {
                mGuards[0].Brain.GetBlackboard().Set("Alarm", true);
            }
            if (mClock >= 12.0 && mClock < 12.0 + Leaves::kStep)
            {
                mGuards[1].Brain.GetBlackboard().Set("Health", 10.0);
            }

            for (Ref<Guard> Each : mGuards)
            {
                mLeaves.Bind(Each, * mTree);
                Each.Brain.Tick(* mTree, Leaves::kStep, mLeaves);
            }
        }
    }

    void OnTerminate() override
    {
        for (Ref<Guard> Each : mGuards)
        {
            mLeaves.Bind(Each, * mTree);
            Each.Brain.Halt(mLeaves);
        }
    }

private:

    Retainer<ZyAI::Behaviour> mTree;
    Array<Guard, 3>           mGuards;
    Leaves                    mLeaves;
    Real64                    mPending = 0.0;
    Real64                    mClock   = 0.0;
};

ZY_APPLICATION(Watch)
```

What happens:

1. Until 5 s every guard runs the third branch: `Walk` returns `Running` for three seconds, then `Wait` two, then a
   30% `Whistle`. The root `Priority` checks `Health` and `Alarm` first on every tick, and both fail.
2. At 5 s the first guard's `Alarm` is `true`. `Compare` reads a `Bool` as `1`, so `Equal 1` passes. The priority
   takes the second branch, stops the walk branch, and the host's `Halt` logs `stopped 'Walk'` if the walk was
   running. `Chase` runs for ten seconds, then the `Timeout` halts it and fails, so `Clear` never runs. The alarm
   stays and the next tick starts a fresh ten-second chase. Wrap the timeout in `Succeed` so `Clear` runs after one
   chase.
3. At 12 s the second guard's `Health` drops to 10. The first branch wins: `Flee` stores a `Home` entry when fresh
   and runs until it gets there. The root then succeeds, starts over, and since `Health` is still low the guard
   flees again from where it stands, until the game raises `Health`.

## 7. Common mistakes and limits

| Mistake | What happens | Do instead |
|---|---|---|
| Loading a `.bhv` without adding `BHVLoader` | No loader for the extension; the resource fails. | `AddLoader(ZyAI::BHVLoader::kTypes, ...)` at startup. |
| Ticking before the tree has loaded | `Tick` returns `Failure` on an empty tree. | Check `HasCompleted()`. |
| `Board.Set("Speed", 2.0f)` | `Real32` is not one of the value types. | Use `Real64` (`2.0`). |
| Comparing a `UInt64` or `Vector3` entry | Every test but `Exists` fails. | Keep compared values as `Real64` or `Bool`. |
| A leaf that starts something and never cleans up | A priority switch or timeout leaves it behind. | Undo it in `Host::Halt(Index)`. |
| Dropping an agent without `Brain::Halt` | Its running leaves are never halted. | `Halt` before the agent goes. |
| One `Brain` shared by several agents | They overwrite each other's running state and blackboard. | One brain per agent. The `Behaviour` is what is shared. |
| Using a global random generator in `Roll` | Results change with tick order. | Seed per agent or per run. |
| A misspelt `Type` | The tree fails to load. | Check names against section 3. |
| Expecting a finished root to stop | The next tick starts it over. | Gate the tick yourself, or end the tree in a `Wait`/`Running` leaf. |

Limits:

- At most 65535 nodes and 64 levels deep per tree.
- Leaves are identified by node index and name only. Argument parsing is the host's job; the loader keeps the JSON.
- No sub-trees or references between `.bhv` files, no events or interrupts besides `Priority`, no services or
  observers that wake a tree. A tree reacts on the tick after the blackboard changes.
- `Brain` and `Host` are single-threaded per agent. Separate brains can tick on separate threads if the host and
  leaves allow it.

## 8. Where it lives

| Path | Holds |
|---|---|
| `Engine/Source/Zyphryon.AI/Behaviour.hpp` | `Behaviour`, `Kind`, `Test`, `Node`, `kMaxNodes`, `kMaxDepth`, `GetNodes`, `GetArguments`. |
| `Engine/Source/Zyphryon.AI/Brain.hpp`, `Brain.cpp` | The tick, every node kind's behaviour, `Halt`, `Reset`, compare rules. |
| `Engine/Source/Zyphryon.AI/Host.hpp` | The leaf interface. |
| `Engine/Source/Zyphryon.AI/Blackboard.hpp` | The blackboard and its value types. |
| `Engine/Source/Zyphryon.AI/Status.hpp` | `Status`. |
| `Engine/Source/Zyphryon.AI/Loader/BHVLoader.hpp`, `BHVLoader.cpp` | The `.bhv` reader and its validation. |
