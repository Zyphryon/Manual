# SQLite

The SQLite link keeps a database in one file, with SQLite compiled into your program. It is the right choice for
save games, settings, local caches and small single-machine servers. This chapter covers its addresses, what it sets
up when it opens a file, how it runs queries on its own thread, the statement cache, and the traps particular to it.

## What it is for

- A single-player game's saves: one file per profile or one file for all.
- Tool data: a level editor's asset index, a baked-content catalog.
- A small server that runs on one machine and does not need a separate database process.

It is not for many processes writing the same file at once. One program, one connection per file, is the intended
use.

## Addresses

```cpp
Unique<Connection> Saves  = Database::Open("sqlite://save.db");
Unique<Connection> Nested = Database::Open("sqlite://Saves/profile-1.db");
Unique<Connection> Memory = Database::Open("sqlite://:memory:");
```

| After `sqlite://` | Opens |
|-------------------|-------|
| A path | That file, relative to the process's working directory, created when missing |
| `:memory:` | A private database in memory, gone when the connection is destroyed |

The path is handed to `sqlite3_open` as it is. Notes:

- The folder must exist. SQLite creates the file, not the folders above it.
- The path is UTF-8.
- SQLite URI file names (`file:save.db?mode=ro`) are not turned on; such a text is read as a plain file name.
- Each `:memory:` connection is its own database. Two `Database::Open("sqlite://:memory:")` calls do not share data.

If the file cannot be opened, the connection still exists. It logs `Database: Unable to open '<path>'` and fails
every query with `The database is not open`.

## What happens on open

The constructor opens the file and runs three pragmas:

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA foreign_keys = ON;
```

| Pragma | Effect for you |
|--------|----------------|
| `journal_mode = WAL` | Writes go to a write-ahead log. The database is three files while open: `save.db`, `save.db-wal`, `save.db-shm`. Copy all three, or close the connection first, when you back up a save. |
| `synchronous = NORMAL` | A commit syncs to disk less often. A power loss can lose the last commits but does not corrupt the file. |
| `foreign_keys = ON` | `REFERENCES` constraints are enforced, and `ON DELETE CASCADE` works. |

You can change any of them with your own query after opening:

```cpp
Link->Submit(Query("PRAGMA synchronous = FULL"), Connection::Reply());
```

Then the constructor starts the worker thread and returns.

## How queries run

```
Submit -> Send: append the order under a lock, signal the gate
Worker: wait on the gate -> take every order -> for each: Perform, Deliver -> wait again
```

- One worker thread per connection runs everything. SQLite itself is only ever called from that thread.
- Orders are taken in the order sent and run one at a time. A long query delays every query behind it on the same
  connection.
- A lone query runs as it is. A statement is atomic by itself.
- A batch runs between `SAVEPOINT batch` and `RELEASE batch`. On the first refused query the link runs
  `ROLLBACK TO batch; RELEASE batch` and fails every result with that query's reason. If `RELEASE` itself fails, the
  batch is rolled back with SQLite's message.

Because a batch is a savepoint, a batch sent while you hold your own `BEGIN` becomes part of your transaction, and
rolling the batch back leaves your outer transaction open.

## The statement cache

The link keeps compiled statements so a statement sent many times is compiled once.

| Rule | Detail |
|------|--------|
| What is kept | A text holding exactly one statement |
| When, with values | From its first use |
| When, without values | From its second use; a text without values sent only once is compiled, run and let go |
| How many | Up to 512 statements; past that the one used longest ago is let go |
| Keyed by | The exact statement text |

The rule about texts without values keeps one-off schema statements and pragmas from filling the cache. The link
remembers up to 4096 such texts as "seen once" before it forgets them all and starts counting again.

What this means for your code:

- Write the same statement text every time and pass the changing parts as values. `"SELECT ... WHERE id = $1"` with
  different ids is one cache entry; building `"... WHERE id = 7"`, `"... WHERE id = 8"` would compile every one.
- Whitespace differences make different entries. Keep statement texts as constants.

Column names (the `Shape`) are kept with the statement and shared by every result, so reading many results of one
statement does not copy the names each time.

## Values and types

Binding:

| `Value` | Bound with |
|---------|------------|
| null | `sqlite3_bind_null` |
| integer | `sqlite3_bind_int64` |
| real | `sqlite3_bind_double` |
| text | `sqlite3_bind_text64`, UTF-8, copied |
| bytes | `sqlite3_bind_blob64`, copied |

Reading: each cell becomes the `Value` kind of its storage class (`INTEGER`, `FLOAT`, `TEXT`, `BLOB`, `NULL`). See
the type tables in [queries.md](queries.md).

`GetAffected()` is `sqlite3_changes64()` for a statement that returns no rows, and 0 for one that does.

Placeholders: `$N` takes the N-th value wherever it stands, and may repeat. Any other placeholder form (`?`,
`:name`, `@name`) takes values by its position among the placeholders.

## How SQLite is built

The amalgamation is compiled with:

| Definition | What it changes for you |
|------------|-------------------------|
| `SQLITE_THREADSAFE=2` | Multi-thread mode. Safe because only the worker thread touches the database handle. |
| `SQLITE_DQS=0` | Double-quoted text is **only** an identifier. `WHERE name = "Aldric"` fails with `no such column: Aldric`. Use `'Aldric'`, or a `$N` value. |
| `SQLITE_DEFAULT_MEMSTATUS=0` | Memory statistics are off. |
| `SQLITE_OMIT_DEPRECATED` | Deprecated C functions are gone (not reachable from this library anyway). |
| `SQLITE_OMIT_SHARED_CACHE` | No shared-cache mode. |
| `SQLITE_OMIT_LOAD_EXTENSION` | No loading of extension libraries. |

SQLite's own SQL features are all there: `INSERT ... ON CONFLICT ... DO UPDATE`, `RETURNING`, window functions,
`json_extract` and the JSON functions, `PRAGMA user_version` and so on.

## Worked example: a settings store

A small key-value store for player settings, kept beside the save.

```cpp
class Settings final
{
public:

    Settings()
        : mLink { Database::Open("sqlite://settings.db") }
    {
        mLink->Submit(Query("CREATE TABLE IF NOT EXISTS setting (key TEXT PRIMARY KEY, value TEXT NOT NULL)"),
                      Connection::Reply());

        mLink->Submit(Query("SELECT key, value FROM setting"), Connection::Reply([this](ConstRef<Result> Outcome)
        {
            for (UInt32 Index = 0; Index < Outcome.GetRows(); ++Index)
            {
                Row Entry = Outcome.GetRow(Index);

                Str Key     = Entry.Read<Str>();
                Str Content = Entry.Read<Str>();
                mValues.Assign(Move(Key), Move(Content));
            }
            mLoaded = true;
        }));
    }

    void Set(Text Key, Text Content)
    {
        mValues.Assign(Str(Key), Str(Content));

        mLink->Submit(Query("INSERT INTO setting (key, value) VALUES ($1, $2) "
                            "ON CONFLICT (key) DO UPDATE SET value = excluded.value", Key, Content),
                      Connection::Reply());
    }

    void Tick()
    {
        mLink->Poll();
    }

private:

    Unique<Connection> mLink;
    Table<Str, Str>    mValues;
    Bool               mLoaded = false;
};
```

Read everything once at start, keep it in memory, and write each change through without waiting. `mLoaded` tells
the rest of the game when the stored values are in.

## Worked example: a save file per slot

```cpp
Unique<Connection> OpenSlot(UInt32 Slot)
{
    Str64 Address;
    Address.Format("sqlite://Saves/slot-{0}.db", Slot);

    return Database::Open(Address);
}
```

Each slot is its own file, so deleting a save is deleting its three files (after its connection is destroyed).

## Common mistakes and limits

| Mistake | Effect | Fix |
|---------|--------|-----|
| `"..."` string literals in SQL | `no such column` | Single quotes or `$N` |
| Copying only `save.db` while it is open | The copy misses the latest commits in `save.db-wal` | Close the connection first, or copy all three files, or run `VACUUM INTO 'backup.db'` |
| Opening the same file twice in one program | Two worker threads compete for the write lock; one may fail with `database is locked` | One connection per file |
| A folder that does not exist in the path | `Unable to open`, every query fails | Create the folder first |
| A slow query in the frame's connection | Every query after it waits | Put heavy reports on a second connection |
| A multi-statement text with values | Refused with `The statement has several statements, which take no values` | One statement per query when it has values |
| Expecting the destructor to call back | Callbacks of the last queries are dropped (the queries still run) | Drain with `Poll()` first, see [connections.md](connections.md) |

Not there:

- No busy timeout is set, so a file locked by another process fails with `database is locked` at once. Set one
  yourself if you share a file: `PRAGMA busy_timeout = 2000`.
- No `IsOpen()` through `Connection`. The `SQLite` class has `IsOpen()`, but `Database::Open` returns the base class;
  learn of a failed open from the first query's result.
- No online backup API. Use `VACUUM INTO 'file'`.

## Where it lives

| Path | Holds |
|------|-------|
| `Module/Database.SQLite/Source/Database.SQLite/SQLite.hpp` | `SQLite`, its constants `kKeep` (512) and `kSeen` (4096) |
| `Module/Database.SQLite/Source/Database.SQLite/SQLite.cpp` | Open and pragmas, the worker loop, savepoint batches, the cache, binding and stepping |
| `Module/Database.SQLite/CMakeLists.txt` | The SQLite download and its compile definitions |
| `Source/Database/Database.cpp` | The `sqlite://` prefix |
