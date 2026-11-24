# Base

This chapter teaches `Zyphryon.Base`, the vocabulary every engine module and every program written against the
engine uses: the type aliases, the parameter wrappers, the containers, delegates, strings and formatting, JSON, the
binary reader and writer, logging, assertions, the filesystem, and enum and struct reflection. It leans on
`math.md` for vectors and the scalar helpers, on `content.md` for URIs and loading files through mounts, on
`scene.md` for components (which use the reflection described here), and on `runtime.md` for jobs and threads.

Everything in this chapter lives in `inline namespace ZyBase`, so you write `Sequence<UInt32>` and never
`ZyBase::Sequence`. One umbrella header brings it all: `#include <Zyphryon.Base/Base.hpp>`. It is also the
precompiled header of the `ZyEngine` target, so every file of a target that links the engine sees it already.

## 1. Why the engine has its own vocabulary

The engine does not use the C++ standard library in its interfaces. It has its own aliases and containers because:

- Every header reads the same way: `Ref<T>`, `ConstRef<T>`, `Ptr<T>` make ownership and mutability visible at the
  call site.
- The containers are small, inline-heavy and predictable (no hidden allocators, no exceptions; the build turns
  exceptions and RTTI off).
- Every container exposes `GetData()` and `GetSize()`, so one concept (`IsContiguous`) lets any container feed a
  `ConstSpan<T>`, a range-for, the formatter or the binary archive.

Code written against the engine is expected to follow the same style. When an engine API takes `ConstSpan<Text>`
or returns `ConstRetainer<ZyRender::Model3D>`, you pass and hold the engine types, not `std::` ones.

## 2. Type aliases

Defined in `Zyphryon.Base/Primitive.hpp`.

| Alias | Underlying | Use |
|---|---|---|
| `Bool` | `bool` | Flags. |
| `Byte` | `uint8_t` | Raw bytes (blobs, buffers). |
| `Char` | `char` | Characters (UTF-8 text). |
| `Wide` | `wchar_t` | Platform wide char, 16-bit on Windows. Only for OS interop. |
| `SInt8` `SInt16` `SInt32` `SInt64` | `int8_t` ... `int64_t` | Signed integers. |
| `UInt8` `UInt16` `UInt32` `UInt64` | `uint8_t` ... `uint64_t` | Unsigned integers. |
| `SInt` / `UInt` | `intptr_t` / `uintptr_t` | Pointer-sized integers. Sizes and indices of containers are `UInt`. |
| `Real32` / `Real64` | `float` / `double` | Reals. Game math is `Real32`; see `math.md` for positions far from the origin. |
| `Null` | `decltype(nullptr)` | Overloads that take `nullptr` explicitly. |
| `Half` | IEEE binary16 (`Half.hpp`) | Half-precision storage, mostly vertex data. |

Text types are not aliases of `std::string`; see section 6.

## 3. Parameter wrappers

Also in `Primitive.hpp`. They are plain aliases, so they cost nothing; they exist to make signatures read the same
everywhere.

| Wrapper | Means | Typical use |
|---|---|---|
| `Ptr<T>` | `T *` | A nullable pointer to something you may change. |
| `ConstPtr<T>` | `const T *` | A nullable read-only pointer, e.g. what `Entity::Get<const T>()` returns. |
| `Ref<T>` | `T &` | An out or in-out parameter, a non-null handle to a service. |
| `ConstRef<T>` | `const T &` | A read-only parameter for anything bigger than a few words. |
| `AnyRef<T>` | `T &&` | A forwarding or rvalue reference: a parameter the callee takes ownership of. |

Rules for choosing one:

- Small values (numbers, `Text`, `Vector2`, `Angle`, handles) go **by value**.
- Bigger values you only read go as `ConstRef<T>`.
- Bulk reads go as `ConstSpan<T>`.
- A value the callee keeps goes as `AnyRef<T>` and the caller writes `Move(Value)`.

The move helpers are in `Zyphryon.Base/Utility.hpp`:

```cpp
Move(Object)              // std::move
Forward<Type>(Object)     // std::forward
Swap(Left, Right)
Exchange(Left, NewValue)  // returns the old value
AddressOf(Object)         // &Object, also for types that overload &
```

```cpp
void Remember(AnyRef<Str> Name)          // the callee keeps the string
{
    mName = Move(Name);
}

Remember(Str::Print("Wolf {0}", Index)); // a temporary binds directly
Remember(Move(Built));                    // a named string must be moved in
```

## 4. Containers

All containers are in `Zyphryon.Base/Container/`. Each exposes `GetData()` and `GetSize()`, so all of them work with
range-for (free `begin`/`end` in `Adaptor.hpp`) and convert to `ConstSpan<T>`.

### 4.1 `Sequence<T, N>`: the dynamic array

`Sequence<T>` (or `Sequence<T, 0>`) grows on the heap (first allocation 16 elements, then doubling).
`Sequence<T, N>` with `N > 0` stores up to `N` elements inline and never allocates; going past `N` asserts.

| Method | What it does |
|---|---|
| `Append(Args...) -> Ref<T>` | Constructs one element at the end from the arguments, returns it. `Append()` default-constructs. |
| `Append(Container)` | Appends every element of another contiguous container of `T`. |
| `Insert(Index, Args...)` / `Insert(Index, Container)` | Inserts, shifting the rest right. |
| `Remove(Index)` | Removes one, keeping order. |
| `Remove(Index, Length)` | Removes a run, keeping order; `Length` is clamped to what is left. |
| `RemoveFast(Index)` | Moves the last element into the hole. Order is lost, cost is O(1). |
| `RemoveLast()` | Pops the back (asserts on an empty sequence). |
| `RemoveIf(Pred) -> Bool` / `RemoveSomeIf(Pred) -> UInt` | Removes the first / every match, keeping order. |
| `RemoveFastIf` / `RemoveFastSomeIf` | Same, not keeping order. |
| `Erase(Value) -> Bool` / `EraseAll(Value) -> UInt` | Removes by `==`. |
| `DrainIf(Output, Pred) -> UInt` | Moves every match into `Output` and keeps the rest. |
| `Find(Value)` / `Find(Pred) -> SInt` | Index of the first match, or **-1**. |
| `FindLast(Pred, Before = max) -> SInt` | Index of the last match below `Before`, or -1. |
| `Contains(Value)` / `Contains(Pred)` | `Find(...) != -1`. |
| `Sort()` / `Sort(Comparator)` | In place. |
| `Clear()` | Destroys every element, keeps the storage. |
| `Advance(Count)` | Constructs `Count` new elements at the end (trivial types are zeroed). |
| `Fill(Value, Count)` | **Appends** `Count` copies of `Value`. It does not overwrite existing elements. |
| `Resize(Count)` | **Clears, then** constructs `Count` fresh elements. |
| `Expand(Count)` | Grows to at least `Count` elements, **keeping** the contents. Never shrinks. |
| `Reserve(Capacity)` / `Shrink()` | Capacity control (dynamic only). |
| `GetFront()` / `GetBack()` / `operator[]` | Element access; all assert on bad indices in debug. |
| `GetSize()` / `GetCapacity()` / `IsEmpty()` / `IsFull()` | Size queries (`IsFull` on the fixed form). |

```cpp
Sequence<UInt32> Ids;
Ids.Append(7);
Ids.Append(9);

Ref<Str> Name = Names.Append("Wolf");        // the new element, constructed in place

for (UInt32 Id : Ids) { ... }                 // range-for works on every container

if (const SInt Found = Ids.Find(9); Found >= 0)
{
    Ids.RemoveFast(static_cast<UInt>(Found));
}

Ids.RemoveSomeIf([](UInt32 Id) { return Id == 0; });
```

**Traps**

- `Resize(N)` does **not** keep the contents. It destroys every element and builds `N` new ones. To grow while
  keeping what is there, call `Expand(N)`. To grow and fill, call `Expand(N)` then write the slots.
- `Fill(Value, N)` appends `N` elements; it is not "set every element to Value".
- `Remove(Index, Length)` asserts when `Index` equals the size, even with `Length` zero. Guard a removal of "the tail
  starting here" with `if (Index < Seq.GetSize())`.
- `Find` returns `SInt` with **-1** for "absent". `Array::Find` (below) returns the count instead. Do not mix them up.
- `Append(Seq[0])` is safe. `Append` builds the new element before the storage grows, so an argument read from the
  same sequence is still valid.
- Any `Append` or `Insert` may reallocate. References and pointers into a dynamic sequence (including the `Ref<T>`
  returned by a previous `Append`) are invalidated by the next growth.
- A fixed `Sequence<T, N>` asserts when full in debug and overruns its storage in release. Size `N` for the worst case.

### 4.2 `Array<T, N>`: the fixed array

A plain fixed-size array (`Container/Array.hpp`). Always holds exactly `N` elements.

| Method | What it does |
|---|---|
| `GetSize()`, `GetData()`, `operator[]`, `GetFront()`, `GetBack()` | Access. |
| `Fill(Value)` | Sets every element. |
| `Find(Value) -> UInt` | Index of the first match, or **`N`** when absent. |
| `Contains(Value)`, `Sort()`, `Sort(Comparator)`, `operator==` | As named. |

A deduction guide lets you write `Array(1, 2, 3)` for `Array<int, 3>`. Engine code uses `Array` for constant tables:

```cpp
static constexpr Array<Text, 3> kLevels =
{
    "Resources://Level/Meadow.lvl",
    "Resources://Level/Forest.lvl",
    "Resources://Level/Ridge.lvl",
};
```

`OnDescribe()` functions (section 13) return `Array(Field..., Field...)` the same way.

### 4.3 `Span<T>` and `ConstSpan<T>`: views

`Span<T>` is a non-owning pointer and size (`Container/Span.hpp`); `ConstSpan<T>` is `Span<const T>`. A span converts
from any contiguous container, from a C array and from a single object (a span of one).

| Method | What it does |
|---|---|
| `GetData()`, `GetSize()`, `GetSizeInBytes()`, `IsEmpty()` | Access. |
| `GetFront()`, `GetBack()`, `operator[]` | Element access. |
| `Contains(Value)` | Linear search. |
| `Slice(Offset)`, `Slice(Offset, Count)` | Sub-views. |

```cpp
UInt32 Sum(ConstSpan<UInt32> Values)
{
    UInt32 Total = 0;
    for (UInt32 Value : Values) { Total += Value; }
    return Total;
}

Sum(Ids);                       // Sequence<UInt32>
Sum(ConstSpan<UInt32>(Fixed));  // Array<UInt32, 4>
```

A span does not keep its source alive. Never return a span over a local container.

### 4.4 `Table<K, V>`: the hash map

A Robin Hood hash map with **dense** storage (`Container/Table.hpp`). Pairs live in one contiguous array of
`Table<K, V>::Pair { First, Second }`, so iterating is a linear walk and `GetData()`/`GetSize()` expose the pairs.

| Method | What it does |
|---|---|
| `Assign(Key, Value) -> Bool` | Inserts or overwrites. `true` when the key was new. |
| `FindOrInsert(Key) -> Ref<V>` | The value, default-constructed when absent. |
| `Find(Key) -> Ptr<V>` | The value, or `nullptr`. |
| `FindOrDefault(Key) -> V` | A copy of the value, or `V {}`. |
| `Contains(Key) -> Bool` | Presence. |
| `Locate(Key) -> UInt32` | The index of the pair in the dense array, or `~0u`. |
| `Erase(Key) -> Bool` | Removes. |
| `Erase(Key, Expected) -> Bool` | Removes only if the key still maps to `Expected`. |
| `EraseIf(Key, Pred)` / `EraseIf(Pred(Key, Value))` | Conditional removal of one key / of every matching pair. |
| `Extract(Key, Output) -> Bool` | Moves the value out and removes the pair. |
| `ForEach(Callback(Key, Value))` | Visits every pair in storage order. |
| `Reserve(Count)`, `Clear()`, `GetSize()`, `IsEmpty()` | As named. |

Keys must be hashable: an integer, a trivially copyable value (hashed by its bytes, including enums), or a type with a
`UInt64 Hash() const` member; and comparable with `==`. Lookups are heterogeneous, so a `Table<Str, V>` is searched
with a `Text` without building a `Str`.

```cpp
Table<Str, UInt32> Bounty;
Bounty.Assign("Wolf", 5);
Bounty.FindOrInsert("Bear") += 20;

if (const Ptr<UInt32> Reward = Bounty.Find(Text("Wolf")))
{
    LOG_I("Game: a wolf is worth {0}", * Reward);
}

Bounty.ForEach([](ConstRef<Str> Name, UInt32 Reward) { ... });
```

**Traps**

- Erasing moves the **last** pair into the hole. Order is not insertion order after an erase, and `Locate` indices
  change.
- `Assign` and `FindOrInsert` may rehash. A `Ptr<V>` from `Find` or a `Ref<V>` from `FindOrInsert` is invalid after
  the next insertion.
- `Hash()` of a `Digest` is its own value: wrap a precomputed hash in `Digest(Value)` when you key a table by hash
  (the content service keys its tables by `Digest(Hash(Uri))`).

### 4.5 `Bitset<N, Word>`

A fixed bitset held inline (`Container/Bitset.hpp`).

| Method | What it does |
|---|---|
| `Test(Index)`, `Set(Index)`, `Set(Index, Length)`, `Reset(Index)`, `Toggle(Index)`, `Reset()` | Bits. |
| `Count()`, `Any()`, `None()`, `All()` | Queries. |
| `ForEach(Callback(Index))` | Every set bit, lowest first. |
| `FindFirstSet()`, `FindFirstClear(Offset = 0)`, `FindLastSet(Offset)`, `FindLastClear()` | Return **`N`** when nothing matches. |
| `GetWord(Index)`, `SetWord(Index, Value)` | Raw words. |
| `&`, `|`, `~`, `&=`, `|=`, `==` | Set algebra. |

```cpp
Bitset<64> Seen;
Seen.Set(3);
if (const UInt Free = Seen.FindFirstClear(); Free < 64) { Seen.Set(Free); }
```

### 4.6 `Unique<T>` and `Retainer<T>`: ownership

`Unique<T>` (`Memory/Unique.hpp`) is an exclusive owner, like `std::unique_ptr`. It moves, never copies.

| Member | What it does |
|---|---|
| `Unique<T>::Create(Args...)` | Allocates and constructs. |
| `Reset(Ptr<T> = nullptr)` | Deletes the current object, optionally takes a new one. |
| `Grab() -> Ptr<T>` | The raw pointer, ownership kept. |
| `Release() -> Ptr<T>` | Gives up ownership without deleting. |
| `->`, `*`, `== nullptr`, `explicit operator Bool` | Access and tests. |

A kernel holds the objects it builds once the services exist this way:

```cpp
class Application final : public ZyRuntime::Kernel
{
protected:

    Bool OnInitialize() override
    {
        mWorld = Unique<Demo::World>::Create(* this);      // needs the services, so not in the constructor
        return true;
    }

    void OnTerminate() override
    {
        mWorld.Reset();                                     // before the services go away
    }

private:

    Unique<Demo::World> mWorld;
};
```

`Retainer<T>` (`Memory/Retainer.hpp`) is a shared, intrusively reference-counted owner. `T` must derive from
`Retainable<T>` (CRTP), which holds the atomic count and deletes the object when it reaches zero. Every content
resource, loader, mount and engine service is held this way.

| Member | What it does |
|---|---|
| `Retainer<T>::Create(Args...)` | Allocates with a count of one. |
| `explicit Retainer(Ptr<Derived>)` | Adopts a heap object (counts it). |
| copy / move | Share / transfer. A `Retainer<Derived>` converts to `Retainer<Base>`. |
| `Retainer<Derived>::Cast(Retainer<Base>)` | Unchecked downcast (a `static_cast`). |
| `->`, `*`, `== nullptr`, `explicit operator Bool`, `explicit operator Ptr<T>` | Access. |
| `Retainable<T>::IsUnique()` | `true` when exactly one retainer holds the object. |

```cpp
class Pack final : public Retainable<Pack>
{
public:
    explicit Pack(UInt32 Size) : mSize { Size } {}
private:
    UInt32 mSize;
};

Retainer<Pack> Wolves = Retainer<Pack>::Create(4);
Retainer<Pack> Shared = Wolves;            // count is two
```

**Trap:** `ConstRetainer<T>` is **`ConstRef<Retainer<T>>`**, a reference to a retainer, not a retainer of a const
object. It is the parameter type (`void AddLoader(ConstSpan<Text>, ConstRetainer<Loader>)`) and a return type of
accessors. Store a `Retainer<T>`, never a `ConstRetainer<T>`, or you hold a dangling reference. Binding a local
`ConstRetainer<T>` to the `Retainer<T>` that `GetService<T>()` returns is fine: the temporary lives as long as the
reference.

### 4.7 `Blob`: an owned run of bytes

`Blob` (`Memory/Blob.hpp`) is a move-only buffer of bytes plus a deleter. File reads hand you one; writes take one.

| Member | What it does |
|---|---|
| `Blob::Allocate<T>(Count)` | Owns a new buffer of `Count * sizeof(T)` bytes. |
| `Blob::Copy<T>(ConstSpan<T>)` | Owns a copy. |
| `Blob::Borrow<T>(Data, Count)` / `Borrow(Span)` | Wraps memory it does **not** own (no-op deleter). |
| `GetData()` / `GetData<T>()`, `GetSize()` | Bytes; `GetSize()` is in bytes. |
| `Copy(Source, Bytes, Offset = 0)` | Writes into the blob. |
| `Clear()`, `== nullptr`, `explicit operator Bool` | Release and tests. |

```cpp
// A file's bytes as text, valid while the blob lives.
const Text Content(Data.GetData<Char>(), Data.GetSize());
```

### 4.8 The rest of the containers

Rarely needed in game code, listed so you do not reinvent them.

| Type | Header | What it is |
|---|---|---|
| `Variant<Types...>` | `Container/Variant.hpp` | Tagged union: `IsHolding<T>()`, `Get<T>()` (asserts on the wrong type), `TryGet<T>()`, `Emplace<T>(...)`, `Visit(...)`, `Reset()`. |
| `Heap<T, N>` | `Container/Heap.hpp` | Binary min-heap (smallest at the front). |
| `Bag<Key>` | `Container/Bag.hpp` | Unordered dense set with fast iteration. |
| `Pool<T, Capacity>` / `Freelist<Capacity>` / `Key<Capacity>` | `Container/Pool.hpp`, `Freelist.hpp`, `Key.hpp` | Fixed-capacity slot allocation with epoch-checked keys. |
| `Spool<T, N>` | `Container/Spool.hpp` | A sequence consumed from the front by a cursor. |
| `Raster<T, W, H>` | `Container/Raster.hpp` | Fixed 2D grid addressed by column and row. |
| `Ring<T, N>` | `Thread/Ring.hpp` | Lock-free single-producer, single-consumer ring. |

## 5. Delegates

`Delegate<R(Args...), InlineSize>` (`Functional/Delegate.hpp`) is a type-erased callable: free function, member
function bound to an object, or lambda. It is copyable and stores small callables inline; bigger ones go on the heap.

`DelegateInlineSize` sets the inline room in pointers: `None` (function pointers only), `Smallest` (1), `Small` (2),
`Standard` (4, the default), `Large` (8), `Largest` (16). A lambda capturing more than `InlineSize * sizeof(void*)`
bytes is heap-allocated.

| Member | What it does |
|---|---|
| `Delegate(Callable)` | Implicit from a lambda or function pointer. |
| `Delegate::Create<&Function>()` | Bound to a free or static function (no storage). |
| `Delegate::Create<&Class::Method>(Ptr<Class>)` | Bound to a member function and an object. |
| `Bind(...)` | Same forms as `Create`, in place. |
| `operator()(Args...)` | Calls. An **empty** delegate returns `R {}` (or does nothing) instead of crashing. |
| `IsEmpty()`, `IsBoundTo<&Function>()`, `IsBoundTo<&Method>(Object)` | Tests. |
| `Consume(Args...)` | Calls once and empties the delegate. |

```cpp
using OnBitten = Delegate<void(ZyScene::Entity Victim, Real32 Damage)>;

OnBitten Handler = [this](ZyScene::Entity Victim, Real32 Damage)
{
    LOG_D("Game: bitten for {0}", Damage);
};

OnBitten Bound = OnBitten::Create<& Wolf::OnBite>(this);   // a member function
Handler(Victim, 4.0f);
```

`MulticastDelegate<R(Args...)>` holds a list: `Add(Delegate)`, `AddFunction<&F>()`, `AddMethod<&C::M>(Object)`,
`AddLambda(Lambda)`, `Remove(...)`, `RemoveFunction`, `RemoveMethod`, `Clear()`, `IsEmpty()`, `Broadcast(Args...)`
(calls all), and `Propagate(Args...)` (calls in order until one returns a truthy value, and returns it). The content
service's `OnAssetCompleted` and `OnAssetFailed`, and every input event (`runtime.md`), are multicast delegates.

**Traps**

- A delegate holding `[this]` keeps nothing alive. Remove it before the object dies, typically in the
  `OnTerminate` of the kernel that added it.
- A `mutable` lambda is allowed (the delegate calls it in place, so state it changes persists between calls). The
  content service's own read callbacks are `mutable` lambdas that move their captured blob out.
- Copying a delegate copies the callable and its captures. A delegate bound with `Create<&Class::Method>(this)`
  copies only the pointer, so every copy calls the same object.

## 6. Strings and text

### 6.1 `Text`: the string view

`Text` (`Lexical/Text.hpp`) is a pointer and a size over characters it does not own. Every API that only reads a
string takes `Text` by value. It converts implicitly from a string literal, from any `Str`/`StrN`, and from any
contiguous container of `Char`.

| Member | What it does |
|---|---|
| `GetData()`, `GetSize()`, `IsEmpty()`, `GetBytes()` | Access. |
| `GetFront()`, `GetBack()`, `operator[]` | Characters. |
| `Slice(Offset)`, `Slice(Offset, Count)` | Sub-views; assert when the range runs past the end. |
| `Hash()`, `==`, `!=` | Hash and comparison. |
| `Text::Empty()` | The empty view. |

Literals: `"Bite"_Text` is a compile-time `Text`; `"Bite"_Hash` is a compile-time `UInt64` hash.

**Traps**

- `Text` is **not null-terminated**. A slice's `GetData()` keeps going past its end. Never hand `Text::GetData()` to
  a C API; build a `Str` first.
- `Text` borrows. A `Text` read out of a JSON document, a `Reader`, a `Blob`, the command line or a temporary `Str`
  dies with it.
- A data member of type `Text` is refused by reflection (`Field::Property` static-asserts) for the same reason. Hold a
  `Str` and hand out a `Text` from a getter.

### 6.2 `Str` and the fixed strings

`String<Capacity>` (`Lexical/String.hpp`) owns its characters and is always null-terminated.

| Alias | Storage |
|---|---|
| `Str` | Heap, grows without limit. |
| `Str16`, `Str32`, `Str48`, `Str64`, `Str128` | Inline, at most 14, 30, 46, 62, 126 characters. |

Common members: `GetData()`, `GetSize()`, `GetCapacity()`, `IsEmpty()`, `Clear()`, `Reserve(N)`, `Append(Char)`,
`Append(Char, Count)`, `Append(Text)`, `AppendCodepoint(UInt32)`, `AppendInteger(...)`, `AppendReal(...)`,
`Insert(Index, ...)`, `Remove(Index, Length)`, `RemoveLast()`, `RemoveLast(Count)`, `Truncate(Size)`,
`Replace(Char, Char) -> UInt`, `Reverse()`, `Slice(...) -> Text`, `Format(Pattern, Args...)`, `Hash()`, comparisons
against `Text`, `+=`, and `+` with a `Text` on either side. Statics: `Str::Print(Pattern, Args...)`,
`Str::Join(Left, Delimiter, Right)`, `Str::Unescape(Text)`, `Str::ConvertFromUTF16(ConstSpan<Wide>)`. The statics
exist on every `String<N>`, so `Filesystem::Path::Join(...)` builds a fixed path the same way.

```cpp
Str  Path  = Str::Join("Resources://Model", '/', "Wolf.mdl");  // "Resources://Model/Wolf.mdl"
Str  Title = Str::Print("{0} the {1}", Name, Role);
Str32 Tag  = "Wolf";                                            // inline, no heap
Tag += ".Alpha";
```

**Trap:** a fixed `StrN` silently **truncates** text that does not fit (`Append(Text)`, assignment, `Format`), and
`Append(Char)` on a full one asserts. Pick `Str` for anything a user or a data file can make long.

### 6.3 String helpers

Free functions in `Lexical/Algorithm.hpp`, all taking `Text`. Positions are `SInt` with **-1** for "not found".

| Helper | Returns |
|---|---|
| `StrStartsWith(Value, Prefix)`, `StrEndsWith(Value, Suffix)`, `StrContains(Value, Search)` | `Bool` |
| `StrFind(Value, Char or Text)`, `StrFindLast(...)`, `StrFindAny(Value, Chars, Start = 0)` | `SInt`, -1 when absent |
| `StrFindCaseInsensitive`, `StrContainsCaseInsensitive` | as above, ignoring case |
| `StrBefore`, `StrBeforeLast`, `StrAfter`, `StrAfterLast` (by `Char` or `Text`) | `Text` |
| `StrTrim`, `StrTrimLeft`, `StrTrimRight` (whitespace, or a given `Char`) | `Text` |
| `StrCompare`, `StrCompareCaseInsensitive` | `SInt32` (<0, 0, >0) |
| `StrEqualCase`, `StrEqualCaseInsensitive` | `Bool` |
| `StrCount(Value, Char)`, `StrLength(Ptr)` | `UInt` |
| `StrLowercase(Char)`, `StrUppercase(Char)` | `Char` |
| `StrIsDigit`, `StrIsHexDigit`, `StrIsSpace`, `StrIsAlphabetic`, `StrIsAlphanumeric`, `StrIsIdentifier` | `Bool` |
| `StrSplit(Value, Delimiter, Callback(Text) -> Bool or void)` | walks tokens; returning `false` stops |
| `StrSkip`, `StrSkipWhitespace`, `StrConsume(Content, Cursor, Char or Text)` | cursor parsing |
| `StrExtractChar`, `StrExtractNumber<Base, Type>`, `StrExtractNumber<Real>`, `StrExtractBool`, `StrExtractText` | cursor parsing |

**Trap:** `StrBefore`, `StrBeforeLast`, `StrAfter` and `StrAfterLast` return an **empty** view when the delimiter is
absent, not the whole input. `StrAfterLast("Wolf.mdl", '/')` is empty. The URI helpers are built on them, which is
why a URI needs a schema and a folder (see `content.md`).

Parsing a number from text uses a cursor:

```cpp
UInt         Cursor = 0;
const UInt32 Level  = StrExtractNumber<10, UInt32>("12 wolves", Cursor);   // 12, Cursor is now 2
UInt         At     = 0;
const Real32 Speed  = StrExtractNumber<Real32>("3.5", At);                 // 3.5
```

`StrExtractNumber` returns `0` and leaves the cursor alone when no digit is there; compare the cursor before and
after when `0` is a valid value.

## 7. Formatting

`Str::Print`, `String::Format`, the `LOG_*` macros and `ZyFormat::Processor::Format` all share one pattern language:

| Pattern | Means |
|---|---|
| `{0}`, `{1}` ... | The argument at that position. A position past the last argument writes nothing. |
| `{{` / `}}` | A literal brace. |
| `{}`, or a `{` never closed | Written as it stands. |
| `{0:spec}` | `[align][sign][0][width][.precision][type]` |

- align: `<` left (default), `>` right, `^` centre.
- sign: `+` always, ` ` a space before positives (default writes `-` only).
- `0` pads with zeros instead of spaces; width is the minimum field width.
- `.N` writes a real with `N` decimals in fixed notation (it implies type `f`).
- type: `x`/`X` hex, `b`/`B` binary, `c` the character an integer stands for, `f` fixed reals.

```cpp
Str::Print("{0} bites for {1:.1} ({2}%)", Name, Damage, Percent);   // "Wolf bites for 4.5 (30%)"
Str::Print("{0:>6}|{1:06}", "hp", 42);                              // "    hp|000042"
Str::Print("{0:x}", 255);                                           // "ff"
Str::Print("{0}", Ids);                                             // "[7, 9]" for a Sequence
```

What an argument may be:

| Argument | Written as |
|---|---|
| integers, `Bool`, `Real32`, `Real64` | numbers; a `Bool` as a flag |
| `Text`, `Str`, `StrN`, any contiguous `Char` container | the characters |
| a type with `template<typename Output> void OnFormat(Ref<Output> Buffer) const` | whatever it writes (`Uri`, `JsonValue`, `JsonError`, vectors, `Angle`) |
| a contiguous container | `[a, b, c]`; a `Table` writes `{"key": value, ...}` |
| an enum | its name, or its number when the value has no name |

**Traps**

- Always number the placeholders. `{}` does not mean "the next argument". It is written as `{}`.
- A formatted argument is captured by address. Arguments are fine as temporaries inside the call, but do not build
  a `ZyFormat::Argument` yourself and keep it.

To make your own type printable, give it `OnFormat`:

```cpp
struct Wound final
{
    UInt32 Amount = 0;

    template<typename Output>
    ZY_INLINE constexpr void OnFormat(Ref<Output> Buffer) const
    {
        ZyFormat::Processor::Format(Buffer, "Wound({0})", Amount);
    }
};
```

## 8. Logging, assertions and profiling

### 8.1 Logs

`Telemetry/Log.hpp` defines four macros. Each takes a **string literal** pattern (it is token-pasted with `_Text`,
so a `Text` variable does not compile) and the format arguments:

```cpp
LOG_D("Game: wolf {0} picked a target", Id);       // debug
LOG_I("Game: started");                             // info
LOG_W("Game: no level named '{0}'", Name);
LOG_E("Game: the save file is damaged");
```

- Prefix every message with its module (`"Game: "`); the engine's own logs do (`"Kernel: "`, `"Content: "`,
  `"Graphics: "`).
- Messages print to stdout with a coloured `[DEBUG]`, `[INFO]`, `[WARNING]` or `[ERROR]` prefix (logcat on Android,
  the browser console on the web). Warnings and errors flush at once; the kernel flushes the rest at shutdown.
- `LOG_D` is **not** compiled out in release. Do not log every frame.
- A message is formatted into a 4096-character thread-local buffer and cut there.
- Do not build a message up front "in case": pass the pieces as arguments.

### 8.2 Assertions

`ZY_ASSERT(Condition, "Message")` (in `Compiler.hpp`) breaks into the debugger when the condition fails in a debug
build. Under `NDEBUG` it expands to nothing and **the condition is not evaluated**. Never put work with side effects
inside it. Engine containers assert on out-of-range access, so a debug build catches index bugs that a release build
turns into memory corruption.

### 8.3 Profiling

`Telemetry/Profiler.hpp` has zone macros that compile to nothing unless the build uses the Tracy backend:
`ZY_PROFILE` (a zone for the function), `ZY_PROFILE_SCOPE("Name")`, `ZY_PROFILE_NAMED(Data, Size)`,
`ZY_PROFILE_INTEGER("Name", Value)`, `ZY_PROFILE_REAL("Name", Value)`, `ZY_PROFILE_FRAME`, `ZY_PROFILE_THREAD("Name")`.
`runtime.md` covers turning Tracy on.

```cpp
void Pack::Hunt()
{
    ZY_PROFILE_SCOPE("Game::Pack::Hunt");
    ...
}
```

## 9. Compiler macros

From `Compiler.hpp`, the ones application code meets:

| Macro | Use |
|---|---|
| `ZY_INLINE` | Force-inline. Every function body in an engine header carries it. |
| `ZY_EXPORT_C` | `extern "C"` plus export; marks an entry point of a plugin library (see `runtime.md`). |
| `ZY_API` | Import/export of engine symbols; empty in a plain static build. Application code does not need it. |
| `ZY_PLATFORM_WINDOWS`, `ZY_PLATFORM_LINUX`, `ZY_PLATFORM_MACOS`, `ZY_PLATFORM_ANDROID`, `ZY_PLATFORM_WEB`, `ZY_PLATFORM_POSIX` | Platform tests. |
| `ZY_HAS_THREADS` | Defined unless building for the web without pthreads. |
| `ZY_MODE_HEADLESS` | Defined in a headless build (see `runtime.md`). |
| `ZY_ALIGN(N)` | `alignas(N)`. |

## 10. JSON

`Marshal/Json/` has a DOM parser and writer. The engine's text formats (`.bhv` trees, `.mtl` materials, `.vfx`
techniques, `.mdl` models, the startup settings, input bindings) are JSON read through these classes, and so can
your own data files.

### 10.1 The pieces

| Type | Role |
|---|---|
| `JsonValue` | One node: null, `Bool`, number (always `Real64` inside), `Str`, array (`Sequence<JsonValue>`) or object (`Table<Str, JsonValue>`). Owns its children. |
| `JsonDocument` | Static `Parse` and `Dump`. |
| `JsonObject` | A non-owning wrapper over an object node, with typed getters and setters. |
| `JsonArray` | A non-owning wrapper over an array node. |
| `JsonError` | Why and where a parse failed: `Message`, `Line`, `Column`, `Path`. It formats itself. |

```cpp
static JsonValue JsonDocument::Parse(Text Content);                                  // null on failure
static Bool      JsonDocument::Parse(Text Content, Ref<JsonValue> Output, Ref<JsonError> Error);
static Str       JsonDocument::Dump(ConstRef<JsonValue> Value, Text Indent = "  "); // "" for one line
```

A leading UTF-8 byte order mark is accepted.

`JsonValue`: `IsNull()`, `IsBool()`, `IsNumber()`, `IsString()`, `IsArray()`, `IsObject()`, `GetBool()`,
`GetNumber<T>()`, `GetString() -> Text`, `GetArray()`, `GetObject()`, `SetBoolean(Bool)`, `SetNumber(T)`,
`SetString(Text)`, `SetArray() -> Ref<Array>`, `SetObject() -> Ref<Object>`. The raw getters **assert** on the wrong
kind; use the wrappers below, which return defaults instead.

`JsonObject`:

| Member | Returns |
|---|---|
| `JsonObject(Ref<JsonValue> Node)` | Wraps a node that **must already be an object** (asserts otherwise). |
| `IsValid()`, `IsNull()`, `IsEmpty()`, `GetSize()`, `Contains(Key)` | Tests. |
| `GetBool(Key, Default = false)` | the flag, or the default when missing or not a flag |
| `GetNumber<T>(Key, Default = {})` | the number cast to `T`, or the default |
| `GetString(Key, Default = "")` | a `Text` into the document, or the default |
| `GetEnum<E>(Key, Default = {})` | the enumerator named by a string (case-insensitive), or the default |
| `GetArray(Key)`, `GetObject(Key)` | a wrapper, null when missing or of another kind |
| `GetValue(Key) -> ConstPtr<JsonValue>` | the raw node, or `nullptr` |
| `SetBool`, `SetNumber`, `SetString`, `SetEnum` (writes the name), `SetArray(Key)`, `SetObject(Key)` | writers |
| `ForEach(Callback(ConstRef<Str> Key, Ref<JsonValue> Value))` | every member |

`JsonArray`: `IsNull()`, `IsEmpty()`, `IsNullOrEmpty()`, `GetSize()`, `GetBool/GetNumber<T>/GetString/GetEnum<E>(Index,
Default)`, `GetArray(Index)`, `GetObject(Index)`, `AddBool`, `AddNumber`, `AddString`, `AddEnum`, `AddArray()`,
`AddObject()`.

### 10.2 Reading a file

Parse with the error overload, check the root is an object, then read through wrappers:

```cpp
Bool Bestiary::Read(Text Content)
{
    JsonValue Root;

    if (JsonError Problem; !JsonDocument::Parse(Content, Root, Problem) || !Root.IsObject())
    {
        LOG_E("Game: Bestiary.json is not a JSON object ({0})", Problem);
        return false;
    }

    const JsonObject Top(Root);
    const JsonArray  List = Top.GetArray("Creatures");

    if (Top.GetNumber<UInt32>("Version") != 1 || List.IsNull())
    {
        LOG_E("Game: Bestiary.json needs \"Version\": 1 and a list of \"Creatures\"");
        return false;
    }

    for (UInt Index = 0; Index < List.GetSize(); ++Index)
    {
        const JsonObject Creature = List.GetObject(Index);
        const Text       Name     = Creature.GetString("Name");
        const Real32     Health   = Creature.GetNumber<Real32>("Health", 10.0f);
        ...
    }
    return true;
}
```

```json
{
    "Version": 1,
    "Creatures":
    [
        { "Name": "Wolf", "Health": 25 },
        { "Name": "Crow" }
    ]
}
```

Where the bytes come from is the content service's business (`content.md`, `Service::Read`), or `Filesystem::Read`
before the services exist.

### 10.3 Writing a file

```cpp
JsonValue Root;
Root.SetObject();

JsonObject Top(Root);
Top.SetNumber("Version", 1);

JsonArray  Creatures = Top.SetArray("Creatures");
JsonObject Wolf      = Creatures.AddObject();
Wolf.SetString("Name", "Wolf");
Wolf.SetNumber("Health", 25);

const Str Output = JsonDocument::Dump(Root);         // pretty, two-space indent
const Str Packed = JsonDocument::Dump(Root, "");     // one line
```

**Traps**

- `JsonObject(Root)` on a root that is not an object asserts. Check `Root.IsObject()` first; `JsonDocument::Parse`
  returns null for bad text.
- Wrappers point into the tree. `JsonArray::AddObject` appends to a `Sequence`, and `JsonObject::SetX` inserts into a
  `Table`; either may reallocate and invalidate **earlier wrappers into the same container**. Finish one child before
  adding the next, or re-fetch it with `GetObject(Index)`.
- Every `Text` you read (`GetString`) and every wrapper die with the `JsonValue` that owns the tree.
- Numbers are `Real64`. `GetNumber<UInt32>` of `-1` or `1e12` is a plain cast with no range check. Validate ranges
  yourself when the file is user-written.
- `GetEnum` matches the enumerator's **name**, case-insensitively, and falls back to the default on anything else,
  including numbers.

## 11. Binary: `Writer`, `Reader` and `Archive`

`Marshal/Binary/` holds the binary serializer every save, every baked asset and the network use.

### 11.1 `Writer`

A growing byte buffer.

| Member | What it writes |
|---|---|
| `Write<T>(Value)` | A trivially copyable value, raw (host byte order). |
| `Write<T>(Ptr, Bytes)` | Raw bytes. |
| `WriteUInt<T>(Value)` / `WriteInt<T>(Value)` | A varint (7 bits per byte) / a zigzag varint. |
| `WriteText(Text)` | A varint length, then the characters. |
| `WriteBlock<Header>(ConstSpan<T>)` | A `Header`-typed **element count**, then the elements raw. |
| `WriteBlock<Header>(Function(Ref<Writer>))` | A `Header`-typed **byte length** patched in after the function wrote the body. |
| `WriteBlockCompressed<Header>(ConstSpan<Byte>, Level = 0)` | The original size, then an LZ4 block (raw when packing does not shrink it). |
| `Ensure(Bytes)` / `Advance(Bytes)` | Reserve room and write in place. |
| `Detach() -> Blob` | Hands the bytes over. |
| `GetData()`, `GetSize()`, `GetCapacity()`, `Clear()` | Access. |

### 11.2 `Reader`

A cursor over bytes it does not own. `Reader(ConstPtr<Byte>, UInt32 Size)` or `Reader(ConstSpan<Byte>)`.

| Member | What it reads |
|---|---|
| `Read<T>()` / `Peek<T>()` | A value; past the end it returns `T {}` and does not move. |
| `ReadUInt<T>()`, `ReadInt<T>()` | Varints. |
| `ReadText() -> Text` | A view **into the buffer**, empty if truncated. |
| `ReadBlock<Header, T>() -> ConstSpan<T>` | A span into the buffer, empty (and the rest skipped) if the count is too large. |
| `ReadBlockCompressed<Header>() -> Blob` | The expanded bytes, empty when malformed. |
| `Skip(Bytes)`, `Split(Bytes) -> Reader` | Move on; split off a sub-reader. |
| `GetAvailable()`, `GetOffset()`, `GetSize()`, `GetData()` | Position. |

The reader never reads past the end; truncated input reads as zeros and empties. That is what makes it safe on
network bytes, but it also means a format mismatch shows up as zeros rather than an error. Check `GetAvailable()`
or a magic number (`ZyContent::Loader::Expect` reads a four-character magic and a version, see `content.md`).

**Trap:** the two `WriteBlock` overloads prefix different things. The span form writes an element count, which
`ReadBlock<Header, T>` reads back. The function form writes the bytes behind their count, which
`ReadBlock<Header, Byte>` reads back, or `Reader Body = Input.Split(Input.Read<Header>())` to read the body as values.

### 11.3 `Archive<Reader>` and `Archive<Writer>`

`Archive<Object>` wraps a `Reader` or a `Writer` so one `Serialize` method both saves and loads. `Serialize(Value)`
handles:

- numbers: raw;
- enums: as their underlying integer;
- `Str`/`StrN`/`Text` targets: through `WriteText`/`ReadText`;
- contiguous containers with an `Element` type: a varint count when resizable (`Sequence`), then the elements
  (one raw block when trivially copyable);
- `Table`: a varint count, then the pairs;
- any type with `template<typename Serializer> void Serialize(Serializer Archive)`: calls it;
- any other trivially copyable type: raw bytes.

`Archive::IsReader` and `Archive::IsWriter` are `static constexpr Bool` for code that must branch.

```cpp
struct Ledger final
{
    UInt32           Gold = 0;
    Str              Owner;
    Sequence<UInt32> Items;

    template<typename Serializer>
    ZY_INLINE void Serialize(Serializer Archive)
    {
        Archive.Serialize(Gold);
        Archive.Serialize(Owner);
        Archive.Serialize(Items);
    }
};

// Save.
Writer Output;
Archive<Writer>(Output).Serialize(Saved);
Blob Bytes = Output.Detach();

// Load.
Reader Input(Bytes.GetData(), Bytes.GetSize());
Ledger Loaded;
Archive<Reader>(Input).Serialize(Loaded);
```

Components use the same `Serialize` shape for world saves and for replication (see `scene.md` and `network.md`).

### 11.4 LZ4

`Compression/LZ4.hpp` exposes `LZ4Bound`, `LZ4Encode` and `LZ4Expand` directly; most code uses
`WriteBlockCompressed`/`ReadBlockCompressed` instead.

## 12. Filesystem

`System/Filesystem.hpp` is a static class over the host filesystem with blocking calls. Results are a
`Filesystem::Result`: `Success`, `Invalid`, `Inexistent`, `Denied`, `Incomplete`, `Unknown`.

| Function | What it does |
|---|---|
| `GetRootFolder() -> Path` | The executable's folder on Windows, the working folder elsewhere. |
| `GetDataFolder(Organization, Application) -> Path` | The per-user data folder joined with both names, empty where there is none. |
| `Read(Path, Ref<Blob>)`, `Write(Path, ConstSpan<Byte>)`, `Write(Path, Text)` | Whole files. `Write` does not create folders. |
| `Enumerate(Path, Callback(ConstRef<Record>) -> Bool)` | A folder's entries (`Record` has `Name`, `Type`, `Size`, `Time`). |
| `Make(Path)`, `MakeAll(Path)` | A folder; a folder and every missing parent. |
| `Ensure(FilePath)` | The parent folders of a file path. |
| `Copy`, `CopyAll`, `Rename`, `Delete`, `DeleteAll` | Files and folders. |
| `Open`, `Tell`, `Read(Handle, Offset, Span)`, `Write(Handle, Offset, Span)`, `Close` | Explicit-offset file I/O. |
| `Map(Path, Ref<View>)`, `Unmap(View)` | Memory-mapped read. |

`Filesystem::Path` is a `String<320>` and `Filesystem::Name` a `String<128>`.

Once the services are up, prefer the content service and its URIs (`Resources://Data/Bestiary.json`) for game
data: the same URI works from a folder, an archive, bytes compiled into the program or a web server, and the calls
do not block the frame (see `content.md`). `Filesystem` is the right tool before the services exist (the settings
file read in `OnConfigure`, see `runtime.md`) and in tools.

## 13. Reflection: enums, fields and schemas

### 13.1 Enums (`ZyEnum`)

`Pattern/Enumerator.hpp` reflects any `enum`/`enum class` at compile time, with no registration.

| Function | Returns |
|---|---|
| `ZyEnum::GetName(Value) -> Text` | The enumerator's name, or empty for an undeclared value. |
| `ZyEnum::Cast<E>(Name, Default) -> E` | The enumerator a name names (case-insensitive), or `Default`. |
| `ZyEnum::Find<E>(Name, Ref<E> Output) -> Bool` | Same, with a found flag. |
| `ZyEnum::Cast(Value)` | The underlying integer. |
| `ZyEnum::GetValues<E>()` | Every declared value, ascending. |
| `ZyEnum::Count<E>() -> UInt` | How many are declared. |

`ZY_DEFINE_BITWISE_ENUM(Enum)` (at namespace scope) or `ZY_DEFINE_BITWISE_FRIEND_ENUM(Enum)` (inside a class) adds
`& | ^ ~ &= |= ^=` for flag enums. `HasBit(Field, Mask)` (`Bit.hpp`) tests one.

```cpp
enum class Temper : UInt8
{
    Calm,
    Wary,
    Hostile,
};

LOG_I("Game: the wolf is {0}", ZyEnum::GetName(Temper::Wary));     // "Wary"
const Temper Read = ZyEnum::Cast("hostile", Temper::Calm);          // Temper::Hostile
```

**Trap:** discovery probes underlying values **0 to 128** only (`ZyEnum::Range<E>::Min`/`Max`). An enumerator with a
negative value or one above 128 (for example a flag `= 256`) has no name: `GetName` returns empty, and JSON
`GetEnum`/`SetEnum` cannot read or write it. Specialize the range for such an enum:

```cpp
template<>
struct ZyEnum::Range<Game::Mark>
{
    static constexpr SInt64 Min = 0;
    static constexpr SInt64 Max = 512;
};
```

### 13.2 Fields: `OnDescribe`

A struct or component lists its fields with a static `OnDescribe()` returning an `Array` of
`ZyReflection::Field` (`Reflection/Field.hpp`). `Schema` (below) and tools such as an inspector read these.

| Factory | Describes |
|---|---|
| `Field::Property<&T::Member>("Name")` | A data member of a scalar, enum, text or flat type. |
| `Field::Property<&T::Get, &T::Set>("Name")` | A value read through a getter and written through a setter. |
| `Field::Nested<&T::Member>("Name")` | A member that is itself described (the walk steps into it). |
| `Field::Asset<&T::Get, &T::Set>("Name", ".mdl")` | A path or id a tool picks, filtered (a `Uri` to a model file here). |
| `Field::List<&T::Member>("Name")` | A run of elements (`Sequence`, `Array`). |
| `Field::Custom<&T::Member>("Name", "Tag")` | A value only a tool registered under `Tag` understands. |

Modifiers return a copy: `.Between(Min, Max)`, `.Stepped(Step)` (numbers), `.Masked()` (an integer that is a set of
bits), `.Notifying()` (tell the owner after a write), `.Frozen()` (shown, not writable).

Names must be string literals (the factories take `const Char (&)[N]`).

```cpp
struct Bleed final
{
    /// The damage dealt each tick, in health points.
    Real32 Damage = 2.0f;

    /// The seconds between ticks.
    Real32 Period = 1.0f;

    /// The temper the wound puts its bearer in.
    Temper Mood   = Temper::Wary;

    ZY_INLINE static constexpr auto OnDescribe()
    {
        return Array(
            ZyReflection::Field::Property<&Bleed::Damage>("Damage").Between(0.0f, 100.0f),
            ZyReflection::Field::Property<&Bleed::Period>("Period").Between(0.1f, 10.0f).Stepped(0.1f),
            ZyReflection::Field::Property<&Bleed::Mood>("Mood"));
    }
};
```

A type can also name itself to reflection with `static constexpr auto OnClassify()` returning
`ZyReflection::Presentation { .Name = "Game.Bleed", .Flat = false }`. `Flat = true` means the type stands in for its
one part (as `Uri` stands for its `Url` string and `Angle` for its radians).

### 13.3 Schemas

`ZyReflection::Schema` (`Reflection/Schema.hpp`) is the run of fields of one described type:

```cpp
constexpr ZyReflection::Schema Schema = ZyReflection::Schema::Of<Game::Bleed>();

Schema.GetSize();                         // number of fields
Schema.Find("Damage");                    // ConstPtr<Field>, nullptr when absent
Schema.Decode(JsonPayload, AddressOf(B)); // writes the fields a JSON object gives, keeps the others
Schema.Write(AddressOf(B), Output);       // binary, field by field
Schema.Read(Input, AddressOf(B));         // binary back, refusing out-of-bounds values
```

`Field::Write(Instance, ZyReflection::Value)` writes one field through reflection, e.g.
`Schema.Find("Damage")->Write(AddressOf(B), ZyReflection::Value::FromReal(5.0f))`.

`Schema::Decode` is the shortest way to read a data file into a struct: the JSON keys are exactly the field names.

```cpp
// { "Damage": 4, "Mood": "Hostile" }  ->  Bleed { Damage = 4, Period = 1 (kept), Mood = Hostile }
Bleed Wound;
ZyReflection::Schema::Of<Bleed>().Decode(Root, AddressOf(Wound));
```

`Decode` returns `false` when a given key held the wrong kind of value (a string for a number, an unknown enum
name); the other fields are still written. Like `JsonObject::GetEnum`, it matches enum names ignoring case. A null
`Root` decodes to `true` and changes nothing.

## 14. Threads (brief)

Application code runs on the main thread unless a callback says otherwise. When you do share state with a job or a
loader:

- `Mutex` and `Guard` (`Thread/Mutex.hpp`): `const Guard Lock(mMutex);` holds it for the scope.
- `Atomic<T>` (`Thread/Atomic.hpp`) is `std::atomic<T>`.
- Prefer the job service lanes (`ZyJob::Lane::Main`, `Compute`, `IO`) over raw threads (see `runtime.md`).

## 15. Other helpers in Base

| Header | What it gives |
|---|---|
| `Utility.hpp` | `Move`, `Forward`, `Swap`, `Exchange`, `Align`, `Construct`, `Destruct`, `Copy`, `Zero`, `Allocate`, `Free`, `Sort`, `Find`, `Count`, `Accumulate`. |
| `Scalar.hpp` | `Min`, `Max`, `Clamp`, `Abs`, `Sign`, `Floor`, `Ceil`, `Round`, `Wrap`, `Lerp`, `Remap`, `Settle`, `IsAlmostZero`, `EncodeFixed`... (taught in `math.md`). |
| `Hash.hpp` | `Hash(Value)`, `HashCombine(Values...)`, `Hash<Type>()` (a type's name hash), `Digest`. |
| `Bit.hpp` | `HasBit`, bit counting (`CountTrailingZeros`, `CeilBit` ...). |
| `Functional/Journal.hpp` | A small command journal for deferred operations. |
| `System/Entropy.hpp` | Bytes from the OS's entropy source. |
| `System/Environment.hpp` | The command line (see `runtime.md`). |
| `System/Library.hpp` | Loading shared libraries (see `runtime.md`). |
| `Cryptography/` | `SHA256`, `Signer`, `TLS` (used by packages and the network). |
| `Lexical/Unicode.hpp` | UTF-8 and UTF-16 decoding. |

## 16. Common mistakes, in one list

1. `Seq.Resize(N)` to grow a list you meant to keep. Use `Expand(N)`.
2. `Seq.Remove(Seq.GetSize(), 0)` or any removal at the size: asserts.
3. Holding a `Ref<T>` from `Append` or a `Ptr<V>` from `Table::Find` across another insertion.
4. Storing a `ConstRetainer<T>` (a reference) instead of a `Retainer<T>`.
5. Storing a `Text` that points into a JSON document, a blob, the command line or a temporary `Str`.
6. Passing `Text::GetData()` to a C function expecting a null terminator.
7. `LOG_I(MyText)` with a variable pattern: the macros only take literals.
8. Side effects inside `ZY_ASSERT`.
9. Wrapping a non-object JSON root in `JsonObject`.
10. An enum value above 128 that must round-trip through JSON or reflection.
11. Reading a `WriteBlock(Function)` block with `ReadBlock<Header, T>` for a `T` other than `Byte` (byte length
    versus element count).
12. `Filesystem::Write` into a folder that does not exist yet. Call `Filesystem::Ensure(Path)` first. A content
    write through the `Disk` mount does this itself.

## 17. Where it lives

| What | Path |
|---|---|
| Aliases and wrappers | `Engine/Source/Zyphryon.Base/Primitive.hpp`, `Utility.hpp`, `Compiler.hpp` |
| Containers | `Engine/Source/Zyphryon.Base/Container/` (`Sequence.hpp`, `Array.hpp`, `Table.hpp`, `Bitset.hpp`, `Span.hpp`, `Variant.hpp` ...) |
| Ownership | `Engine/Source/Zyphryon.Base/Memory/` (`Unique.hpp`, `Retainer.hpp`, `Retainable.hpp`, `Blob.hpp`) |
| Delegates | `Engine/Source/Zyphryon.Base/Functional/Delegate.hpp` |
| Text and formatting | `Engine/Source/Zyphryon.Base/Lexical/` (`Text.hpp`, `String.hpp`, `Algorithm.hpp`, `Format/`) |
| JSON | `Engine/Source/Zyphryon.Base/Marshal/Json/` |
| Binary | `Engine/Source/Zyphryon.Base/Marshal/Binary/` (`Reader.hpp`, `Writer.hpp`, `Archive.hpp`), `Compression/LZ4.hpp` |
| Logs, profiler | `Engine/Source/Zyphryon.Base/Telemetry/` |
| Filesystem, command line, libraries | `Engine/Source/Zyphryon.Base/System/` (`Filesystem.hpp`, `Environment.hpp`, `Library.hpp`) |
| Reflection | `Engine/Source/Zyphryon.Base/Pattern/Enumerator.hpp`, `Reflection/` (`Field.hpp`, `Schema.hpp`, `Access.hpp`, `Value.hpp`) |
| Small JSON readers to copy | `Engine/Source/Zyphryon.Runtime/Startup.cpp` (`Startup::Load`), `Engine/Source/Zyphryon.AI/Loader/BHVLoader.cpp` |
