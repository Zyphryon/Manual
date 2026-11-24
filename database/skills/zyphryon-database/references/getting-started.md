# Getting started

This chapter adds Zyphryon Database to a CMake program, explains the build options, opens a first connection and
shows where `Poll()` goes in a frame loop.

## What you need

- A C++ program that builds against the Zyphryon engine (`ZyEngine` target). The library uses the engine's base
  types (`Str`, `Text`, `Sequence`, `Unique`, `Delegate`, `Mutex`, `Thread`), its logger (`LOG_I`, `LOG_E`), and, for
  PostgreSQL, its network module (`ZyNetwork::Proactor`, `ZyNetwork::Socket`) and TLS class.
- CMake 3.26 or newer (the library's `CMAKE_MINIMUM_REQUIRED`).
- Network access at configure time. CMake downloads the engine (when you have not provided it) and the SQLite
  amalgamation.

## Adding the library

The library is a normal CMake project named `ZyDatabase`. The simplest way in is `FetchContent`:

```cmake
INCLUDE(FetchContent)

FetchContent_Declare(ZyDatabase GIT_REPOSITORY https://github.com/Zyphryon/Database GIT_TAG main)
FetchContent_MakeAvailable(ZyDatabase)

TARGET_LINK_LIBRARIES(MyGame PRIVATE ZyDatabase)
```

Linking `ZyDatabase` brings everything else in through `PUBLIC` links:

```
ZyDatabase            (Source/Database, Database::Open)
 ├── ZyDatabase_Core      (Connection, Query, Value, Result, Row, Shape) -> ZyEngine
 ├── ZyDatabase_SQLite    (when ZY_DATABASE_SQLITE)   -> ZyDatabase_SQLite_Engine (sqlite3.c, private)
 └── ZyDatabase_Postgres  (when ZY_DATABASE_POSTGRES)
```

### Using your own copy of the engine

The library's `CMakeLists.txt` declares the engine itself:

```cmake
FetchContent_Declare(External_ZyEngine
        GIT_REPOSITORY https://github.com/Zyphryon/Engine
        GIT_TAG        main
        GIT_SHALLOW    TRUE)
FetchContent_MakeAvailable(External_ZyEngine)
```

With CMake's `FetchContent`, the first declaration of a name wins. If your program already builds the engine from
a local folder or a fixed tag, declare `External_ZyEngine` yourself **before** you make `ZyDatabase` available, so
both use the same engine:

```cmake
FetchContent_Declare(External_ZyEngine SOURCE_DIR ${CMAKE_SOURCE_DIR}/External/Engine)
FetchContent_MakeAvailable(External_ZyEngine)

FetchContent_Declare(ZyDatabase GIT_REPOSITORY https://github.com/Zyphryon/Database GIT_TAG main)
FetchContent_MakeAvailable(ZyDatabase)
```

If your program fetches the engine under another name, CMake would see two copies of the `ZyEngine` target. Use the
name `External_ZyEngine` for yours.

### Build options

| Option | Default | Effect |
|--------|---------|--------|
| `ZY_DATABASE_SQLITE` | `ON` | Builds `Module/Database.SQLite` with SQLite 3.54.1 compiled in, and defines `ZY_DATABASE_SQLITE` for users |
| `ZY_DATABASE_POSTGRES` | `ON` | Builds `Module/Database.Postgres`, and defines `ZY_DATABASE_POSTGRES` for users |

Turn off what you do not use. A single-player game that only keeps saves needs SQLite alone:

```cmake
SET(ZY_DATABASE_POSTGRES OFF CACHE BOOL "" FORCE)

FetchContent_Declare(ZyDatabase GIT_REPOSITORY https://github.com/Zyphryon/Database GIT_TAG main)
FetchContent_MakeAvailable(ZyDatabase)
```

A dedicated server that only talks to PostgreSQL can turn SQLite off the same way and skip the SQLite download.

The two definitions are `PUBLIC`, so your own code can test them:

```cpp
#ifdef    ZY_DATABASE_SQLITE
    // The program can open sqlite:// addresses.
#endif // ZY_DATABASE_SQLITE
```

### What gets downloaded

| What | From | Pinned by |
|------|------|-----------|
| Zyphryon engine | `https://github.com/Zyphryon/Engine`, branch `main` | `GIT_TAG main` (not pinned) |
| SQLite amalgamation 3.54.1 | `https://www.sqlite.org/2026/sqlite-amalgamation-3540100.zip` | `URL_HASH SHA3_256=...` |

SQLite is compiled with these definitions (see [sqlite.md](sqlite.md) for what they change):

```
SQLITE_THREADSAFE=2
SQLITE_DQS=0
SQLITE_DEFAULT_MEMSTATUS=0
SQLITE_OMIT_DEPRECATED
SQLITE_OMIT_SHARED_CACHE
SQLITE_OMIT_LOAD_EXTENSION
```

## The one header

```cpp
#include <Database/Database.hpp>

using namespace ZyPlugin::Database;
```

`Database/Database.hpp` includes `Database.Core/Connection.hpp`, which brings `Query`, `Value`, `Result`, `Row` and
`Shape`. You never include the SQLite or PostgreSQL headers yourself; `Database::Open` picks the link.

## Opening a connection

```cpp
static Unique<Connection> Open(Text Address);
```

`Database::Open` reads the start of the address:

| Address | Opens |
|---------|-------|
| `sqlite://save.db` | A database in the file `save.db`, created when missing, relative to the working directory |
| `sqlite://:memory:` | A database that lives only as long as the connection |
| `postgres://user:password@host:5432/name` | A PostgreSQL database over a plain link |
| `postgres://user:password@host/name?tls=on` | The same, encrypted, any certificate accepted |
| `postgres://user:password@host/name?tls=verified&ca=ca.pem` | Encrypted, refusing a server the authority in `ca.pem` did not sign |

It returns:

- a `Unique<Connection>` that owns the link, or
- an empty `Unique<Connection>` when no built link answers the prefix (for example `postgres://` with
  `ZY_DATABASE_POSTGRES` off, or a typo such as `sqlite:/save.db`). It logs `Database: No database answers the
  address '...'`.

```cpp
Unique<Connection> Link = Database::Open("sqlite://save.db");

if (!Link)
{
    LOG_E("Save: The save database could not be opened");
    return;
}
```

Opening is itself non-blocking in practice: both links start a worker thread and return at once.

- A SQLite file that cannot be opened still gives you a connection. It logs `Database: Unable to open '...'`, and
  every query you send fails with the error `The database is not open`.
- A PostgreSQL server that cannot be reached also gives you a connection. It keeps trying in the background, and
  queries sent meanwhile fail with the reason the link broke. See [postgres.md](postgres.md).

So the test `if (!Link)` only catches a wrong prefix. The first query's `Result` tells you whether the database is
really there.

## Your first queries

```cpp
#include <Database/Database.hpp>

using namespace ZyPlugin::Database;

class Saves final
{
public:

    Saves()
        : mLink { Database::Open("sqlite://save.db") }
    {
        mLink->Submit(Query("CREATE TABLE IF NOT EXISTS hero ("
                            "  name  TEXT    PRIMARY KEY,"
                            "  level INTEGER NOT NULL,"
                            "  gold  INTEGER NOT NULL DEFAULT 0)"), Connection::Reply([](ConstRef<Result> Outcome)
        {
            if (!Outcome.IsOk())
            {
                LOG_E("Saves: Could not create the hero table, {0}", Outcome.GetError());
            }
        }));
    }

    void Store(Text Name, UInt32 Level, UInt32 Gold)
    {
        mLink->Submit(Query("INSERT INTO hero (name, level, gold) VALUES ($1, $2, $3) "
                            "ON CONFLICT (name) DO UPDATE SET level = excluded.level, gold = excluded.gold",
                            Name, Level, Gold), Connection::Reply());
    }

    void Load(Text Name)
    {
        mLink->Submit(Query("SELECT level, gold FROM hero WHERE name = $1", Name),
                      Connection::Reply([this](ConstRef<Result> Outcome)
        {
            if (Outcome.IsOk() && Outcome.GetRows() == 1)
            {
                Row Hero = Outcome.GetRow(0);
                mLevel = Hero.Read<UInt32>();
                mGold  = Hero.Read<UInt32>();
            }
        }));
    }

    void Tick()
    {
        mLink->Poll();
    }

private:

    Unique<Connection> mLink;
    UInt32             mLevel = 0;
    UInt32             mGold  = 0;
};
```

Points to notice:

- `CREATE TABLE` is sent and not waited for. The `INSERT` sent after it runs after it, because a connection runs
  queries in the order sent.
- `Store` passes `Connection::Reply()`, an empty callback: the result is dropped. That is fine for a write you do not
  need to confirm, but it also drops any error.
- `Load` captures `this`. That is safe because callbacks run only inside `Poll()`, and only `Saves::Tick` calls
  `Poll()` on this connection. Once `Saves` is being destroyed nobody polls, so no callback can reach a dead `this`.
  The rule is in [connections.md](connections.md): a callback runs only inside your own `Poll()` call.

## Where `Poll()` goes

Call `Poll()` once a frame on the thread that owns the game state the callbacks touch. A typical loop:

```cpp
while (Running)
{
    Input();
    Update(Delta);

    // Hand out every database result that arrived since the last frame.
    Saves.Tick();

    Render();
}
```

`Poll()` returns the number of queries and batches still out, which is handy for a loading screen:

```cpp
const UInt32 Left = Link->Poll();

if (Left == 0 && mLoading)
{
    mLoading = false;
    EnterWorld();
}
```

`Poll()` is cheap when nothing arrived: it takes a lock, swaps an empty list, and returns.

## Common mistakes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Callbacks never run | `Poll()` is never called | Call `Poll()` once a frame |
| `Database::Open` returns empty | Wrong prefix, or that link was built off | Check the address starts with `sqlite://` or `postgres://` and the option is `ON` |
| Every query fails with `The database is not open` | SQLite could not open the file (bad folder, no rights) | Check the path and the log line `Database: Unable to open` |
| Link error about two `ZyEngine` targets | Your program fetched the engine under another name | Declare it as `External_ZyEngine` before `ZyDatabase` |
| A string literal in SQL fails with "no such column" | SQLite is built with `SQLITE_DQS=0` | Use single quotes, or pass the text as `$N` |

## Where it lives

| Path | Holds |
|------|-------|
| `CMakeLists.txt` | Project `ZyDatabase`, engine fetch, options |
| `Module/Database.Core/CMakeLists.txt` | `ZyDatabase_Core`, linked to `ZyEngine` |
| `Module/Database.SQLite/CMakeLists.txt` | SQLite download, compile definitions, `ZyDatabase_SQLite` |
| `Module/Database.Postgres/CMakeLists.txt` | `ZyDatabase_Postgres` |
| `Source/Database/Database.hpp`, `Database.cpp` | `Database::Open` and the address prefixes |
