# Cookbook

Complete, self-contained recipes for the jobs a game most often gives a database: accounts with hashed passwords, a
save-game table, a leaderboard, and schema migrations. Each recipe gives the schema for both databases, then the C++
against the library.

All recipes assume:

```cpp
#include <Database/Database.hpp>

using namespace ZyPlugin::Database;
```

and a `Unique<Connection>` that the owning class polls once a frame. Read [connections.md](connections.md) first if
the callback model is new to you.

---

## 1. Schema migrations

Start here: every other recipe creates its tables through a migration.

The library has no migration helper. The pattern below is small and works on both databases:

- a one-row table holds the schema version,
- an ordered list of steps, one SQL statement each, never edited once shipped,
- at start, read the version and run the missing steps **as one batch**, together with the version update. The
  batch is all or nothing, so a failed step leaves the database at the old version, untouched.

### Schema

The version table is created by its own lone query before the read. Both databases accept this text:

```sql
CREATE TABLE IF NOT EXISTS schema_version (id INTEGER PRIMARY KEY, version INTEGER NOT NULL);
INSERT INTO schema_version (id, version) VALUES (1, 0) ON CONFLICT (id) DO NOTHING;
```

### Code

```cpp
class Schema final
{
public:

    explicit Schema(Ref<Connection> Link)
        : mLink { Link }
    {
    }

    void Migrate()
    {
        mLink.Submit(Query(
            "CREATE TABLE IF NOT EXISTS schema_version (id INTEGER PRIMARY KEY, version INTEGER NOT NULL);"
            "INSERT INTO schema_version (id, version) VALUES (1, 0) ON CONFLICT (id) DO NOTHING;"),
            Connection::Reply());

        mLink.Submit(Query("SELECT version FROM schema_version WHERE id = 1"),
                     Connection::Reply([this](ConstRef<Result> Outcome)
        {
            if (!Outcome.IsOk() || Outcome.GetRows() != 1)
            {
                LOG_E("Schema: The version could not be read, {0}", Outcome.GetError());
                mFailed = true;
                return;
            }
            Apply(static_cast<UInt32>(Outcome.GetValue(0, 0).GetInteger()));
        }));
    }

    Bool IsReady() const
    {
        return mReady;
    }

    Bool IsFailed() const
    {
        return mFailed;
    }

private:

    void Apply(UInt32 Current)
    {
        // One statement per entry. Append new steps at the end; never change or reorder a shipped one.
        static constexpr Array<Text, 4> kSteps = {
            "CREATE TABLE account (id INTEGER PRIMARY KEY, name TEXT NOT NULL UNIQUE, salt BLOB NOT NULL, "
            "rounds INTEGER NOT NULL, secret BLOB NOT NULL, created INTEGER NOT NULL)",
            "CREATE TABLE save (account INTEGER NOT NULL REFERENCES account (id) ON DELETE CASCADE, "
            "slot INTEGER NOT NULL, title TEXT NOT NULL, played INTEGER NOT NULL, data BLOB NOT NULL, "
            "checksum BLOB NOT NULL, PRIMARY KEY (account, slot))",
            "CREATE TABLE best (player TEXT PRIMARY KEY, points INTEGER NOT NULL, at INTEGER NOT NULL)",
            "CREATE INDEX best_points ON best (points DESC)",
        };

        if (Current >= kSteps.GetSize())
        {
            mReady = true;
            return;
        }

        Sequence<Query> Steps;

        for (UInt32 Index = Current; Index < kSteps.GetSize(); ++Index)
        {
            Steps.Append(kSteps[Index]);
        }
        Steps.Append("UPDATE schema_version SET version = $1 WHERE id = 1", static_cast<UInt32>(kSteps.GetSize()));

        mLink.Submit(Move(Steps), Connection::Answer([this, Current](ConstSpan<Result> Outcomes)
        {
            if (!Outcomes[0].IsOk())
            {
                LOG_E("Schema: Migration from version {0} was refused, {1}", Current, Outcomes[0].GetError());
                mFailed = true;
                return;
            }
            LOG_I("Schema: Migrated from version {0}", Current);
            mReady = true;
        }));
    }

    Ref<Connection> mLink;
    Bool            mReady  = false;
    Bool            mFailed = false;
};
```

The steps above use SQLite types. For PostgreSQL, write the steps with `BIGINT GENERATED ALWAYS AS IDENTITY` for
the `account.id` key and `BYTEA` for the byte columns; see the schemas in the next recipes. If one program must run
on both, keep two step lists and choose by the address you opened.

`Array<Text, N>` is the engine's fixed array; `GetSize()` gives its length, so adding a step updates the target
version by itself.

### Using it

Do not send queries that need the new tables until `IsReady()` is true. Queries sent earlier run before the
migration batch, because the connection runs in send order.

```cpp
void Game::Tick()
{
    mLink->Poll();

    if (mSchema.IsFailed())
    {
        ShowFatal("The save database could not be upgraded.");
    }
    else if (mSchema.IsReady() && !mStarted)
    {
        mStarted = true;
        LoadProfile();
    }
}
```

### Rules for steps

| Rule | Why |
|------|-----|
| One statement per step | A batch query holds one statement |
| Never edit a shipped step | Databases already past it will not run it again |
| Add columns with `ALTER TABLE ... ADD COLUMN ... DEFAULT ...` | Existing rows need a value |
| Prefer new tables over rewriting old ones | SQLite's `ALTER TABLE` is limited (it cannot drop a constraint) |
| Keep data fixes in steps too (`UPDATE ...`) | They then run exactly once, in order |

On SQLite, `PRAGMA user_version` is a built-in alternative to the version table. The table works on both databases,
so the recipe uses it.

---

## 2. Accounts with hashed passwords

A server stores accounts: a unique name, a salt, a round count and a PBKDF2 key. See [digest.md](digest.md) for why
each piece is there.

### Schema

SQLite:

```sql
CREATE TABLE account (
    id      INTEGER PRIMARY KEY,
    name    TEXT    NOT NULL UNIQUE,
    salt    BLOB    NOT NULL,
    rounds  INTEGER NOT NULL,
    secret  BLOB    NOT NULL,
    created INTEGER NOT NULL
);
```

PostgreSQL:

```sql
CREATE TABLE account (
    id      BIGINT  GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name    TEXT    NOT NULL UNIQUE,
    salt    BYTEA   NOT NULL,
    rounds  INTEGER NOT NULL,
    secret  BYTEA   NOT NULL,
    created BIGINT  NOT NULL
);
```

### Code

```cpp
class Accounts final
{
public:

    /// The callback of a sign-up or login, given the account id or 0 when refused.
    using Verdict = Delegate<void(SInt64)>;

    explicit Accounts(Ref<Connection> Link)
        : mLink { Link }
    {
    }

    void Register(Text Name, Text Password, SInt64 Now, AnyRef<Verdict> Done)
    {
        Array<Byte, 16> Salt { };

        if (!Entropy::Gather(Salt))
        {
            Done(0);
            return;
        }

        const SHA256::Sum Secret = SHA256::Derive(Password.GetBytes(), Salt, kRounds);

        // A taken name inserts nothing, and RETURNING then gives no row.
        mLink.Submit(Query("INSERT INTO account (name, salt, rounds, secret, created) VALUES ($1, $2, $3, $4, $5) "
                           "ON CONFLICT (name) DO NOTHING RETURNING id",
                           Name, ConstSpan<Byte>(Salt), kRounds, ConstSpan<Byte>(Secret), Now),
                     Connection::Reply([Done](ConstRef<Result> Outcome)
        {
            Done(Outcome.IsOk() && Outcome.GetRows() == 1 ? Outcome.GetValue(0, 0).GetInteger() : 0);
        }));
    }

    void Login(Text Name, Text Password, AnyRef<Verdict> Done)
    {
        mLink.Submit(Query("SELECT id, salt, rounds, secret FROM account WHERE name = $1", Name),
                     Connection::Reply([Done, Password = Str(Password)](ConstRef<Result> Outcome)
        {
            if (!Outcome.IsOk() || Outcome.GetRows() != 1)
            {
                Done(0);
                return;
            }

            Row Entry = Outcome.GetRow(0);

            const SInt64          Id     = Entry.Read<SInt64>();
            const ConstSpan<Byte> Salt   = Entry.Read<ConstSpan<Byte>>();
            const UInt32          Rounds = Entry.Read<UInt32>();
            const ConstSpan<Byte> Stored = Entry.Read<ConstSpan<Byte>>();

            Done(Match(Password, Salt, Rounds, Stored) ? Id : 0);
        }));
    }

private:

    static constexpr UInt32 kRounds = 200000;

    static Bool Match(Text Password, ConstSpan<Byte> Salt, UInt32 Rounds, ConstSpan<Byte> Stored)
    {
        if (Stored.GetSize() != SHA256::kSize || Rounds == 0)
        {
            return false;
        }

        const SHA256::Sum Key = SHA256::Derive(Password.GetBytes(), Salt, Rounds);

        Byte Difference = 0;

        for (UInt32 Index = 0; Index < SHA256::kSize; ++Index)
        {
            Difference |= static_cast<Byte>(Key[Index] ^ Stored[Index]);
        }
        return Difference == 0;
    }

    Ref<Connection> mLink;
};
```

Notes:

- `INSERT ... ON CONFLICT (name) DO NOTHING RETURNING id` works on both databases (SQLite has `RETURNING` since
  3.35). A taken name gives a successful result with no rows, so you never parse an error message.
- `Login` copies the password into the lambda (`Password = Str(Password)`): the caller's `Text` may be gone when the
  callback runs.
- A wrong name and a wrong password both give 0, so a client cannot tell which accounts exist.
- `kRounds` is an example. Measure `Derive` on your server and choose (see [digest.md](digest.md)). At high counts,
  move `Derive` (in `Register` and in the callback of `Login`) onto a worker thread; it is the slow part of both.
- The password must reach the server over an encrypted network link. Hashing it on the client does not help.

### Raising the round count later

Because each row stores its own `rounds`, a login that succeeds with an old count can rehash in place:

```cpp
if (Rounds < kRounds)
{
    Array<Byte, 16> Salt { };
    Entropy::Gather(Salt);

    const SHA256::Sum Secret = SHA256::Derive(Password.GetBytes(), Salt, kRounds);

    Link.Submit(Query("UPDATE account SET salt = $1, rounds = $2, secret = $3 WHERE id = $4",
                      ConstSpan<Byte>(Salt), kRounds, ConstSpan<Byte>(Secret), Id), Connection::Reply());
}
```

---

## 3. A save-game table

A single-player game keeps several save slots per profile in one SQLite file. Each save is a title for the menu, the
play time, and one blob of game state with a checksum.

### Schema

```sql
CREATE TABLE save (
    slot     INTEGER PRIMARY KEY,
    title    TEXT    NOT NULL,
    played   INTEGER NOT NULL,
    data     BLOB    NOT NULL,
    checksum BLOB    NOT NULL
);
```

(For PostgreSQL write `BIGINT` and `BYTEA`; for one save table per account, add an `account` column to the key as in
the migration recipe.)

### Code

How you turn your game state into bytes is your own serialization; the recipe takes a `Sequence<Byte>`.

```cpp
class Slots final
{
public:

    Slots()
        : mLink { Database::Open("sqlite://profile.db") }
    {
        mLink->Submit(Query("CREATE TABLE IF NOT EXISTS save (slot INTEGER PRIMARY KEY, title TEXT NOT NULL, "
                            "played INTEGER NOT NULL, data BLOB NOT NULL, checksum BLOB NOT NULL)"),
                      Connection::Reply());
    }

    void Save(UInt32 Slot, Text Title, UInt64 Played, ConstSpan<Byte> State)
    {
        const SHA256::Sum Checksum = SHA256::Hash(State);

        mLink->Submit(Query("INSERT INTO save (slot, title, played, data, checksum) VALUES ($1, $2, $3, $4, $5) "
                            "ON CONFLICT (slot) DO UPDATE SET title = excluded.title, played = excluded.played, "
                            "data = excluded.data, checksum = excluded.checksum",
                            Slot, Title, Played, State, ConstSpan<Byte>(Checksum)),
                      Connection::Reply([Slot](ConstRef<Result> Outcome)
        {
            if (!Outcome.IsOk())
            {
                LOG_E("Save: Slot {0} was not saved, {1}", Slot, Outcome.GetError());
            }
        }));
    }

    void List()
    {
        // The menu needs no blobs; leave them out of the read.
        mLink->Submit(Query("SELECT slot, title, played FROM save ORDER BY slot"),
                      Connection::Reply([this](ConstRef<Result> Outcome)
        {
            mMenu.Clear();

            for (UInt32 Index = 0; Index < Outcome.GetRows(); ++Index)
            {
                Row Cursor = Outcome.GetRow(Index);

                Ref<Entry> Line = mMenu.Append();
                Line.Slot   = Cursor.Read<UInt32>();
                Line.Title  = Cursor.Read<Str>();
                Line.Played = Cursor.Read<UInt64>();
            }
        }));
    }

    void Load(UInt32 Slot)
    {
        mLink->Submit(Query("SELECT data, checksum FROM save WHERE slot = $1", Slot),
                      Connection::Reply([this, Slot](ConstRef<Result> Outcome)
        {
            if (!Outcome.IsOk() || Outcome.GetRows() != 1)
            {
                LOG_W("Save: Slot {0} is empty", Slot);
                return;
            }

            Row Saved = Outcome.GetRow(0);

            const ConstSpan<Byte> State  = Saved.Read<ConstSpan<Byte>>();
            const ConstSpan<Byte> Stored = Saved.Read<ConstSpan<Byte>>();
            const SHA256::Sum     Actual = SHA256::Hash(State);

            if (Stored.GetSize() != SHA256::kSize || !Equal(Actual, Stored))
            {
                LOG_E("Save: Slot {0} is corrupted", Slot);
                return;
            }

            mLoaded.Clear();
            mLoaded.Append(State);   // Copy out: the view dies with the result.
        }));
    }

    void Erase(UInt32 Slot)
    {
        mLink->Submit(Query("DELETE FROM save WHERE slot = $1", Slot), Connection::Reply());
    }

    void Tick()
    {
        mLink->Poll();
    }

private:

    struct Entry
    {
        UInt32 Slot   = 0;
        Str    Title;
        UInt64 Played = 0;
    };

    static Bool Equal(ConstSpan<Byte> Left, ConstSpan<Byte> Right)
    {
        for (UInt32 Index = 0; Index < SHA256::kSize; ++Index)
        {
            if (Left[Index] != Right[Index])
            {
                return false;
            }
        }
        return true;
    }

    Unique<Connection> mLink;
    Sequence<Entry>    mMenu;
    Sequence<Byte>     mLoaded;
};
```

Notes:

- The upsert (`ON CONFLICT (slot) DO UPDATE`) keeps one row per slot and works on both databases.
- The checksum catches a truncated or edited blob. It does not stop a determined player from editing their own
  save; nothing on the player's machine can.
- Saving while the player keeps playing is fine: `Save` returns at once and the write happens on the worker.
- On quit, drain the connection so the last save is confirmed (see [connections.md](connections.md)).

---

## 4. A leaderboard

A server keeps each player's best score and answers "top N" and "my rank".

### Schema

SQLite:

```sql
CREATE TABLE best (player TEXT PRIMARY KEY, points INTEGER NOT NULL, at INTEGER NOT NULL);
CREATE INDEX best_points ON best (points DESC);
```

PostgreSQL:

```sql
CREATE TABLE best (player TEXT PRIMARY KEY, points BIGINT NOT NULL, at BIGINT NOT NULL);
CREATE INDEX best_points ON best (points DESC);
```

### Code

```cpp
class Board final
{
public:

    explicit Board(Ref<Connection> Link)
        : mLink { Link }
    {
    }

    void Submit(Text Player, SInt64 Points, SInt64 Now)
    {
        // Only a better score replaces the stored one.
        mLink.Submit(Query("INSERT INTO best (player, points, at) VALUES ($1, $2, $3) "
                           "ON CONFLICT (player) DO UPDATE SET points = excluded.points, at = excluded.at "
                           "WHERE excluded.points > best.points",
                           Player, Points, Now),
                     Connection::Reply([Name = Str(Player)](ConstRef<Result> Outcome)
        {
            if (Outcome.IsOk() && Outcome.GetAffected() == 1)
            {
                LOG_I("Board: New best for {0}", Name);
            }
        }));
    }

    void Top(UInt32 Count)
    {
        mLink.Submit(Query("SELECT player, points FROM best ORDER BY points DESC, at ASC LIMIT $1", Count),
                     Connection::Reply([this](ConstRef<Result> Outcome)
        {
            mTop.Clear();

            for (UInt32 Index = 0; Index < Outcome.GetRows(); ++Index)
            {
                Row Line = Outcome.GetRow(Index);

                Ref<Place> Entry = mTop.Append();
                Entry.Player = Line.Read<Str>();
                Entry.Points = Line.Read<SInt64>();
            }
        }));
    }

    void Rank(Text Player)
    {
        mLink.Submit(Query("SELECT COUNT(*) + 1 FROM best "
                           "WHERE points > (SELECT points FROM best WHERE player = $1)", Player),
                     Connection::Reply([this, Name = Str(Player)](ConstRef<Result> Outcome)
        {
            if (Outcome.IsOk() && Outcome.GetRows() == 1)
            {
                LOG_I("Board: {0} is ranked {1}", Name, Outcome.GetValue(0, 0).GetInteger());
            }
        }));
    }

private:

    struct Place
    {
        Str    Player;
        SInt64 Points = 0;
    };

    Ref<Connection> mLink;
    Sequence<Place> mTop;
};
```

Notes:

- `ON CONFLICT ... DO UPDATE ... WHERE excluded.points > best.points` works on both databases. A worse score
  changes nothing and `GetAffected()` is 0.
- On PostgreSQL, `GetAffected()` comes from the server's `INSERT 0 n` tag; on SQLite from `sqlite3_changes64`. Both
  report 1 for an insert or an accepted update and 0 for a skipped one.
- `COUNT(*)` is a `bigint` on PostgreSQL and an integer on SQLite; both arrive as an integer `Value`.
- A player with no score gets rank 1 from the `Rank` query, because the subquery is `NULL` and no row compares
  greater. Check that the player exists first if that matters.
- `LIMIT $1` works on both; PostgreSQL infers `bigint` for the placeholder.
- For a board of millions of players, `Rank` scans every better row. Keep it for small boards, or cache ranks
  periodically.

### Submitting many scores at the end of a match

Send them as one batch so the board updates at once:

```cpp
Sequence<Query> Scores;

for (ConstRef<Finisher> Each : Match.GetFinishers())
{
    Scores.Append("INSERT INTO best (player, points, at) VALUES ($1, $2, $3) "
                  "ON CONFLICT (player) DO UPDATE SET points = excluded.points, at = excluded.at "
                  "WHERE excluded.points > best.points", Each.Name, Each.Points, Now);
}

Link.Submit(Move(Scores), Connection::Answer());
```

`Match` and `Finisher` are your own types. The statement text is the same for every query, so each link compiles it
once.

---

## Recipe checklist

| Check | Recipe |
|-------|--------|
| Tables are created by a migration batch, never ad hoc | 1 |
| Every changing value is a `$N`, never concatenated | all |
| Passwords are derived with a per-row salt and stored rounds | 2 |
| Bytes go in as `ConstSpan<Byte>` and come out copied | 2, 3 |
| "Exists already" is learned from `RETURNING` or `GetAffected()`, not error text | 2, 4 |
| Related writes go in one batch | 1, 4 |
| The connection is drained before quitting when the last write matters | 3 |
