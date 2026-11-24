# Connections

`Connection` is the one class your code talks to. This chapter explains its asynchronous model: how `Submit` hands
work away without blocking, how `Poll` brings results back, the order guarantees, batches, threads, shutdown and
what happens when things fail.

## What it is for

A game frame lasts a few milliseconds. A database write can take longer than that: a disk sync, a lock held by
another writer, a round trip to a server on another continent. `Connection` moves that wait off your thread. You
describe the work, hand it over, and pick up the answer on a later frame.

## The model in one picture

```
 your thread                         connection                       worker thread
 -----------                         ----------                       -------------
 Submit(Query, Reply)  ----------->  ++pending, Send() -------------> queue of orders
                                                                         |
                                                                         v
                                                                     run in send order
                                                                         |
                                                                         v
                                     Deliver(): list of arrived  <---- results
 Poll()  <-------------------------  swap the list, call each callback, pending -= count
```

- `Submit` adds one to the pending count and passes the work to the link's `Send`. Both links only append to a
  queue under a lock and wake their worker. `Submit` returns at once.
- The worker runs the work and calls `Deliver`, which stores the callback and results in a list under a lock.
- `Poll` takes the whole list, calls every callback in the order the results arrived, and lowers the pending count.

## The API

```cpp
class Connection
{
public:

    using Reply  = Delegate<void(ConstRef<Result>)>;
    using Answer = Delegate<void(ConstSpan<Result>)>;

    void   Submit(AnyRef<Query> Request, AnyRef<Reply> Callback);
    void   Submit(AnyRef<Sequence<Query>> Requests, AnyRef<Answer> Callback);
    UInt32 Poll();
    UInt32 GetPending() const;
};
```

| Member | What it does |
|--------|--------------|
| `Reply` | The callback of a lone query. It gets the query's `Result`. |
| `Answer` | The callback of a batch. It gets one `Result` per query, in the order the queries were given. |
| `Submit(Query, Reply)` | Sends one query. It runs after everything sent before it. |
| `Submit(Sequence<Query>, Answer)` | Sends a batch, which the database runs whole or not at all. |
| `Poll()` | Calls the callback of every result that arrived since the last call. Returns how many are still out. |
| `GetPending()` | How many queries and batches were sent and not yet handed out by `Poll()`. |

`Reply` and `Answer` are engine `Delegate`s. A lambda of any size fits: small ones are stored inline, larger ones on
the heap. An empty delegate (`Connection::Reply()`, `Connection::Answer()`) means "drop the result".

Wrap a lambda in the delegate type when you pass it. `Submit` has two overloads, and naming the type keeps the call
plain to read:

```cpp
Link->Submit(Query("SELECT 1"), Connection::Reply([](ConstRef<Result> Outcome)
{
    LOG_I("Answer: {0}", Outcome.GetValue(0, 0).GetInteger());
}));
```

## Ordering

The connection promises:

1. **Queries run in the order they were submitted.** A lone query and a batch count the same. If you send a
   `CREATE TABLE` and then an `INSERT` into that table, the insert sees the table.
2. **Callbacks run in the order the results arrived**, which on both links is the order sent. The SQLite worker runs
   one order at a time and delivers it before taking the next. The PostgreSQL server answers a pipeline in order.
3. **A callback runs only inside `Poll()`**, on the thread that called `Poll()`. Never on the worker thread, never
   inside `Submit`.

So you can chain dependent work without waiting:

```cpp
Link->Submit(Query("DELETE FROM pack WHERE owner = $1", Owner), Connection::Reply());
Link->Submit(Query("INSERT INTO pack (owner, item) VALUES ($1, $2)", Owner, "Torch"), Connection::Reply());
Link->Submit(Query("SELECT item FROM pack WHERE owner = $1", Owner), Connection::Reply([](ConstRef<Result> Outcome)
{
    // Sees exactly one row, "Torch", because the delete and insert ran first.
}));
```

Ordering is per connection. Two connections to the same database run independently, and nothing orders one
against the other.

## Batches

A batch is a `Sequence<Query>` sent with an `Answer`:

```cpp
Sequence<Query> Trade;
Trade.Append("UPDATE purse SET gold = gold - $1 WHERE owner = $2", 50, Buyer);
Trade.Append("INSERT INTO pack (owner, item) VALUES ($1, $2)", Buyer, "Sword");

Link->Submit(Move(Trade), Connection::Answer([](ConstSpan<Result> Outcomes)
{
    if (!Outcomes[0].IsOk())
    {
        // Every result failed with the same reason. Nothing the batch did was kept.
        LOG_W("Shop: The trade was refused, {0}", Outcomes[0].GetError());
        return;
    }
    // Outcomes[0].GetAffected() is the number of purses changed, Outcomes[1].GetAffected() the items added.
}));
```

`Sequence::Append` constructs the `Query` in place, so its arguments are the `Query` constructor's arguments.

The rules of a batch:

- **Whole or not at all.** SQLite wraps the batch in a `SAVEPOINT` and rolls it back on the first refusal. PostgreSQL
  sends the whole batch closed by one sync message, so the server runs it as one implicit transaction.
- **All failed with one reason.** When the batch is rolled back, every `Result` in the span is failed with the same
  error text, even the queries that ran before the one that was refused. So checking `Outcomes[0].IsOk()` is enough
  to know whether the batch was kept.
- **One statement per query.** Each `Query` of a batch must hold one SQL statement. A text such as `"A; B"` is for a
  lone `Submit(Query, Reply)` with no values.
- **One result per query, in order.** `Outcomes.GetSize()` equals the number of queries you sent.

A lone `Submit(Query, Reply)` is a batch of one underneath: the base class wraps it in a `Sequence<Query>` of one
and adapts the `Reply` to an `Answer`. A lone query is not wrapped in a savepoint on SQLite; a single statement is
atomic by itself.

### Batch or transaction statements?

You may also write `BEGIN` and `COMMIT` yourself as separate lone queries. Because the connection runs everything in
order on one link, the statements between them run inside that transaction. Prefer a batch:

- A batch cannot be left half open. If your program sends `BEGIN`, an `UPDATE`, and then crashes before sending
  `COMMIT`, the link stays inside a transaction until it is closed.
- A batch answers once, with all results together.
- On PostgreSQL a batch goes out as one write (see [postgres.md](postgres.md)).

## Threads

| Who | What runs there |
|-----|-----------------|
| The thread that calls `Submit` | Appending the work to the link's queue. Nothing else. |
| The link's worker thread | Running SQL (SQLite), or the socket, TLS and protocol (PostgreSQL). |
| The thread that calls `Poll` | Every callback. |

`Submit` takes a lock and bumps an atomic counter, so sending from several threads does not corrupt anything. The
order between two threads that submit at the same time is whichever got the lock first. If order matters, submit
from one thread.

`Poll` is meant to be called from one thread. Callbacks may call `Submit` again; the new work goes to the back of the
queue and its result arrives on a later `Poll`.

```cpp
Link->Submit(Query("SELECT id FROM account WHERE name = $1", Name), Connection::Reply([Raw = AddressOf(* Link)](ConstRef<Result> Outcome)
{
    if (Outcome.IsOk() && Outcome.GetRows() == 1)
    {
        const SInt64 Id = Outcome.GetValue(0, 0).GetInteger();
        Raw->Submit(Query("SELECT slot, data FROM save WHERE account = $1", Id), Connection::Reply());
    }
}));
```

## Capturing state in callbacks

The callback runs on a later frame. Anything it reads must still be alive then.

| Capture | Safe? |
|---------|-------|
| A value copied into the lambda (`[Id]`, `[Name = Str(Name)]`) | Yes |
| `this` of an object that owns the connection and is the only one that polls it | Yes |
| A reference or pointer to a local variable | No, it is gone by the next frame |
| A `Text` view of a temporary string | No, copy it into a `Str` |
| `this` of an object that may be destroyed while the query is out | No; use an identifier and look the object up |

A pattern for game objects that may disappear (`Session` and `mSessions` are your own player types):

```cpp
const UInt32 Player = Session.GetIdentifier();

Link->Submit(Query("SELECT gold FROM purse WHERE owner = $1", Player), Connection::Reply([this, Player](ConstRef<Result> Outcome)
{
    if (const Ptr<Session> Found = mSessions.Find(Player))
    {
        Found->SetGold(Outcome.GetRows() == 1 ? Outcome.GetValue(0, 0).GetInteger() : 0);
    }
}));
```

## Waiting (there is no `Fetch`)

The library has no call that blocks until one answer is in. A command-line tool, a test, or the shutdown of a
server that must be sure its last writes landed can loop on `Poll()`:

```cpp
// Blocks the calling thread until every query sent on this link was answered. Not for use inside a frame.
static void Drain(Ref<Connection> Link)
{
    while (Link.Poll() > 0)
    {
        // Spins. The library offers no sleep or wait on the connection.
    }
}
```

This loop burns a core while it waits. For a tool that is acceptable. Never do it in a game frame.

A one-shot query in a tool:

```cpp
Unique<Connection> Link = Database::Open("sqlite://save.db");
Str Version;

Link->Submit(Query("SELECT sqlite_version()"), Connection::Reply([& Version](ConstRef<Result> Outcome)
{
    Version = Outcome.GetValue(0, 0).GetText();
}));

Drain(* Link);
LOG_I("SQLite {0}", Version);
```

Capturing `Version` by reference is safe here only because `Drain` runs every callback before the function goes on.

## Shutdown

Destroying the connection (letting the `Unique<Connection>` go) closes the link:

| Link | What the destructor does |
|------|--------------------------|
| SQLite | Stops the worker after it **runs** every query still queued, then closes the file. |
| PostgreSQL | Waits until every batch queued or in the air is answered (or failed, when the link is broken), then closes the socket. It waits at most 10 seconds, then fails what is left with `The link was closed before the server answered`. |

In both cases the results that arrive while the connection is dying are **dropped**: no callback runs after the
destructor starts. Writes still happen. If your program must act on the last answers (for example to log that the
save completed), call `Drain` first:

```cpp
void Game::Quit()
{
    mSaves->Submit(BuildFinalSave(), Connection::Answer([](ConstSpan<Result> Outcomes)
    {
        LOG_I("Save: Final save {0}", Outcomes[0].IsOk() ? "kept" : "refused");
    }));

    Drain(* mSaves);
    mSaves = nullptr;
}
```

The PostgreSQL destructor waits for the server, but not forever. A server that accepts the link and then never
answers holds the destructor for at most 10 seconds, after which what is left fails with `The link was closed before
the server answered`. A server that closes or drops the link fails the batches and lets it finish at once.

## Failure

A failed query does not throw. Its `Result` says so:

```cpp
if (!Outcome.IsOk())
{
    LOG_E("Save: {0}", Outcome.GetError());
}
```

Errors you can meet, by where they come from:

| Error text | From | Means |
|------------|------|-------|
| The database's own message (for example `no such table: hero`, `duplicate key value violates unique constraint ...`) | Either link | The database refused the SQL |
| `The query failed` | `Result::Fail` with an empty reason | The database refused and gave no text |
| `The database is not open` | SQLite | The file could not be opened when the connection was made |
| `The statement has a placeholder no value was given for` | SQLite | The text has `$3` and you passed fewer than three values |
| `The statement has several statements, which take no values` | SQLite | A text such as `"A; B"` was given values |
| `The server could not be resolved or a link to it opened`, `The server refused the link or could not be reached`, `The server closed the link`, `The link to the server was lost while writing`, ... | PostgreSQL | The link broke; see [postgres.md](postgres.md) |

The error text is the database's message only. There is no error code, so do not branch on exact messages if you
can avoid it. Prefer SQL that reports the outcome by its effect, such as `INSERT ... ON CONFLICT DO NOTHING` and a
check of `GetAffected()` (see [cookbook.md](cookbook.md)).

## Common mistakes

| Mistake | Effect | Fix |
|---------|--------|-----|
| Expecting the result right after `Submit` | The variable is still empty | Use the value inside the callback, or on a later frame |
| Passing `Connection::Reply()` for a query that may fail | The error is lost silently | Pass a callback that checks `IsOk()` |
| Capturing a local by reference | Crash or garbage on a later frame | Capture by value |
| Two statements in one batch query | The batch fails | One statement per query in a batch |
| Calling `Poll` from two threads | Callbacks race with each other | Poll from one thread |
| Busy-looping `Poll` in a frame | The frame stalls | Poll once a frame and continue |

## Where it lives

| Path | Holds |
|------|-------|
| `Module/Database.Core/Source/Database.Core/Connection.hpp` | `Connection`, `Reply`, `Answer`, `Submit`, `Poll`, `GetPending` |
| `Module/Database.Core/Source/Database.Core/Connection.cpp` | How a lone query becomes a batch of one, `Poll`, `Deliver`, `Refuse` |
| `Module/Database.SQLite/Source/Database.SQLite/SQLite.cpp` | SQLite worker loop and savepoint batches |
| `Module/Database.Postgres/Source/Database.Postgres/Postgres.cpp` | PostgreSQL worker loop and pipelined batches |
