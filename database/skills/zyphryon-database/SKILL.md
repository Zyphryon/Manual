---
name: zyphryon-database
description: Use when a program built on the Zyphryon engine stores or loads data through Zyphryon Database (ZyDatabase, ZyPlugin::Database) — opening `sqlite://` or `postgres://` addresses with Database::Open, sending a Query with `$1` placeholders through Connection::Submit, polling replies with Connection::Poll, reading a Result, Row or Value, running a batch as one transaction, keeping save games, accounts, leaderboards or a schema migration, hashing passwords with SHA256::Derive, or debugging a query that failed, never answered or blocked a frame.
---

# Zyphryon Database

Zyphryon Database is a small storage library for programs built on the Zyphryon engine. It gives a game or a server
one way to talk to two databases:

- **SQLite**, a database kept in one file, compiled into your program. Good for save games, settings, a local
  cache, or a small single-machine server.
- **PostgreSQL**, a database behind a server, reached over the network with the library's own client. Good for a
  shared game server with accounts, characters and leaderboards.

Both sit behind one class, `Connection`, so the code that sends queries does not care which one it talks to.

The library has one rule above all others: **sending a query never blocks the caller.** You hand the query to the
connection, keep running your frame, and later a call to `Poll()` hands you the result through a callback. A slow
disk or a slow server never costs you a frame.

## How the pieces fit

```
Database::Open("sqlite://save.db")           Database::Open("postgres://user:pw@host/name")
          |                                             |
          v                                             v
   SQLite : Connection                          Postgres : Connection
   (worker thread, sqlite3 compiled in)         (worker thread, wire protocol over ZyNetwork sockets,
                                                 optional TLS, SCRAM-SHA-256 login)
          \                                             /
           \______________ Connection ________________/
                 Submit(Query, Reply)          one query
                 Submit(Sequence<Query>, Answer)   a batch, whole or not at all
                 Poll()                         hands out every result that arrived
                 GetPending()                   how many are still out
                              |
                              v
                Result  ->  Row / Value  (+ Shape, the column names)
```

| Type | Header | Role |
|------|--------|------|
| `Database` | `Source/Database/Database.hpp` | `Database::Open(Text Address)` picks the link by address prefix |
| `Connection` | `Module/Database.Core/Source/Database.Core/Connection.hpp` | The base link: `Submit`, `Poll`, `GetPending`, `Reply`, `Answer` |
| `Query` | `.../Database.Core/Query.hpp` | A statement and its placeholder values, `$1` for the first |
| `Value` | `.../Database.Core/Value.hpp` | One cell: null, `SInt64`, `Real64`, text or bytes |
| `Result` | `.../Database.Core/Result.hpp` | Rows read, rows changed, or the error |
| `Row` | `.../Database.Core/Row.hpp` | Reads one row's cells in column order with `Read<T>()` |
| `Shape` | `.../Database.Core/Shape.hpp` | Column names shared by every result of one statement |
| `SQLite` | `Module/Database.SQLite/Source/Database.SQLite/SQLite.hpp` | The file database link |
| `Postgres` | `Module/Database.Postgres/Source/Database.Postgres/Postgres.hpp` | The server database link |
| `Scram` | `Module/Database.Postgres/Source/Database.Postgres/Scram.hpp` | SCRAM-SHA-256 login, internal to `Postgres` |

Everything lives in the namespace `ZyPlugin::Database`. You include one header:

```cpp
#include <Database/Database.hpp>

using namespace ZyPlugin::Database;
```

## The smallest program

```cpp
const Unique<Connection> Link = Database::Open("sqlite://save.db");

Link->Submit(Query("CREATE TABLE IF NOT EXISTS hero (name TEXT PRIMARY KEY, level INTEGER NOT NULL)"),
             Connection::Reply());

Link->Submit(Query("INSERT INTO hero (name, level) VALUES ($1, $2)", "Aldric", 12), Connection::Reply());

Link->Submit(Query("SELECT name, level FROM hero WHERE level >= $1", 10),
             Connection::Reply([](ConstRef<Result> Outcome)
{
    if (!Outcome.IsOk())
    {
        LOG_E("Save: {0}", Outcome.GetError());
        return;
    }
    for (UInt32 Index = 0; Index < Outcome.GetRows(); ++Index)
    {
        Row Hero = Outcome.GetRow(Index);

        const Str    Name  = Hero.Read<Str>();
        const UInt32 Level = Hero.Read<UInt32>();
        LOG_I("{0} is level {1}", Name, Level);
    }
}));

// Once a frame, on the thread that should run the callbacks.
Link->Poll();
```

## Chapters

Read the one that matches the task.

| Chapter | Read it when |
|---------|--------------|
| [getting-started.md](references/getting-started.md) | Adding ZyDatabase to a CMake program, the build options, opening a first connection, the frame loop |
| [connections.md](references/connections.md) | Understanding Submit, Poll, ordering, batches, callbacks, threads, shutdown and failure |
| [queries.md](references/queries.md) | Writing a `Query`, choosing `Value` types, reading `Result`, `Row` and `Shape`, type mapping per database |
| [sqlite.md](references/sqlite.md) | Using `sqlite://` addresses, files, `:memory:`, pragmas, statement caching, SQLite-only traps |
| [postgres.md](references/postgres.md) | Using `postgres://` addresses, login, TLS, reconnects, pipelining, PostgreSQL-only traps |
| [digest.md](references/digest.md) | Hashing passwords and tokens with the engine's `SHA256` (`Hash`, `Sign`, `Derive`) and `Entropy` |
| [cookbook.md](references/cookbook.md) | Accounts with hashed passwords, a save-game table, a leaderboard, schema migrations |

## Top traps

1. **Nothing arrives without `Poll()`.** Results wait inside the connection until you call `Poll()`. If you never
   poll, no callback ever runs. Call it once a frame.
2. **Callbacks run on the polling thread, later.** Do not capture references to stack values that will be gone by the
   next frame. Capture by value, or capture something that outlives the connection.
3. **A failed query does not throw or log for you.** Check `Result::IsOk()` and `Result::GetError()` in every
   callback that matters. An empty `Connection::Reply()` drops the result, errors included.
4. **There is no blocking fetch.** The library has no call that waits for one answer. Tools and shutdown code that
   must wait loop on `Poll()` until `GetPending()` is 0 (see [connections.md](references/connections.md)).
5. **Use `$1`, `$2`, … placeholders, never string concatenation.** `$N` works on both databases. `?` works only on
   SQLite.
6. **A batch must hold one statement per query.** `Submit(Sequence<Query>, Answer)` runs the queries as one
   transaction. A text with several statements (`"A; B"`) belongs in a lone `Submit(Query, Reply)` with no values.
   Given values, both links refuse it.
7. **Pass fixed-size byte arrays as `ConstSpan<Byte>(...)`.** A `SHA256::Sum` or `Array<Byte, N>` stored as a blob is
   clearest when wrapped in `ConstSpan<Byte>` before it goes into a `Query`.
8. **Text and bytes read from a `Result` live as long as the `Result`.** `Row::Read<Text>()`, `Value::GetText()` and
   `Value::GetBytes()` are views. Copy into `Str` or `Sequence<Byte>` to keep them after the callback.
9. **SQLite is built with `SQLITE_DQS=0`.** Double quotes are identifiers only. Write string literals with single
   quotes, or better, pass them as `$N` values.
10. **A PostgreSQL address needs a user.** Without an `@`, `postgres://localhost/game` reaches `localhost` with an
    empty user name, and the server refuses the login. Always write `user:password@host`.

## Gaps in the current code

The library is young. These are not there yet; do not look for them:

- No blocking `Fetch` or `Wait` call. Waiting means looping on `Poll()`.
- No migration helper. The cookbook shows how to write one with a version table.
- No SQLSTATE error codes. `Result::GetError()` is the server's message text only.
- No column lookup by name on `Result`; columns are read by index (`GetName(Column)` gives the name).
- No date, time, UUID or JSON value types; those arrive as text.
- No PostgreSQL `LISTEN`/`NOTIFY`, `COPY`, or MD5 password login.
- No connection pool. One `Connection` is one link; open several if you want parallel work.

## Where it lives

| Path | Holds |
|------|-------|
| `CMakeLists.txt` | The `ZyDatabase` target, options `ZY_DATABASE_SQLITE` and `ZY_DATABASE_POSTGRES` |
| `Source/Database/` | `Database::Open` |
| `Module/Database.Core/` | `Connection`, `Query`, `Value`, `Result`, `Row`, `Shape` |
| `Module/Database.SQLite/` | The SQLite link and the compiled-in SQLite amalgamation |
| `Module/Database.Postgres/` | The PostgreSQL link and its SCRAM login |
| `README.md` | Short setup and address table |
