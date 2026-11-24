# PostgreSQL

The PostgreSQL link talks to a PostgreSQL server with the library's own client. It does not use `libpq`: it speaks
the server's wire protocol (version 3.0) over the engine's network sockets, encrypts with the engine's TLS class,
and logs in with SCRAM-SHA-256 or a clear-text password. This chapter covers its address, login, TLS, the way it
pipelines queries, how it reconnects, and the traps particular to it.

## What it is for

- A game server that keeps accounts, characters, inventories and leaderboards in one shared database.
- Several server processes that must see the same data.
- Anything that outgrows one file on one machine.

## Address

```
postgres://user:password@host:port/name?option&option
```

| Part | Required | Default | Notes |
|------|----------|---------|-------|
| `user` | Yes | — | The login role |
| `:password` | No | empty | Everything after the first `:` up to the last `@`; may itself hold `:` and `@` |
| `@host` | Yes | — | A host name or address the engine can resolve |
| `:port` | No | `5432` | |
| `/name` | No | the user name | The database to open |
| `?tls=on` | No | plain | Encrypt, accept any certificate |
| `?tls=verified` | No | plain | Encrypt, refuse a certificate that is untrusted or for another host name |
| `&ca=file` | No | the system's trusted authorities | A PEM file of the only authorities trusted, read when the connection starts |

Examples:

```cpp
Database::Open("postgres://game:secret@127.0.0.1/game");
Database::Open("postgres://game:secret@db.example.com:6432/game?tls=on");
Database::Open("postgres://game:secret@db.example.com/game?tls=verified");
Database::Open("postgres://game:secret@db.example.com/game?tls=verified&ca=certs/db-ca.pem");
```

Parsing rules to know:

- The text is split at the **last** `@`. Without an `@`, the whole text is read as host, port and database, and the
  user name is empty. The server then refuses the login. Always include `user@`.
- The user name ends at the first `:`. A user name cannot contain `:`.
- Nothing is percent-decoded. Write the password as it is; `%40` stays `%40`.
- Options are split at `&`. An unknown option logs `Database: The option '...' is not known, so the link is opened
  without it`, and the link opens anyway. A typo such as `tls=yes` therefore gives a plain link.
- `ca=` is read relative to the working directory, through the engine's `Filesystem::Read`. A file that cannot be
  read logs `Database: The authority file '...' could not be read` and the link falls back to the system's
  authorities.
- `ca=` alone does not turn TLS on. Pair it with `tls=verified`.

Keep the password out of source code. Read it from the environment or a configuration file and build the address at
start-up:

```cpp
Str Address("postgres://game:");
Address.Append(Password);
Address.Append("@db.example.com/game?tls=verified");

Unique<Connection> Link = Database::Open(Address);
```

## Life of the link

```
Reaching ──connect──> (tls?) Securing ──handshake──> Greeting ──login──> Ready
    ^                                                                       |
    |                                                                       | socket error, server closed,
    └──────────── Mend: wait 1 s, 2 s, 4 s … up to 30 s ─── Broken <───────┘ bad message, login refused
```

| State | What happens |
|-------|--------------|
| `Reaching` | The host is resolved and a TCP connection is posted. |
| `Securing` | With `tls=on` or `tls=verified`, the link asks the server for encryption (`SSLRequest`). The server must answer `S`. Then the TLS handshake runs. |
| `Greeting` | The startup message names `user`, `database`, `client_encoding = UTF8` and `bytea_output = hex`. The server asks for a login method; the link answers. |
| `Ready` | Queries go out and answers come back. |
| `Broken` | Every batch in the air or waiting fails with the reason. New queries fail at once with the same reason. After a wait, the link starts again at `Reaching`. |

The wait before a new attempt starts at 1 second and doubles on each failed attempt, up to 30 seconds. A successful
login resets it to 1 second and logs `Database: The link to '<host>' is back`.

**While the link is broken, queries are not held back for later. They fail.** Your code must decide whether to send
them again. A server that has just started is the common case:

```cpp
void Server::Store(Query Request)
{
    mLink->Submit(Move(Request), Connection::Reply([](ConstRef<Result> Outcome)
    {
        if (!Outcome.IsOk())
        {
            // A broken link and a refused query look the same here. Log, and retry only what is safe to repeat.
            LOG_W("Store: {0}", Outcome.GetError());
        }
    }));
}
```

Queries sent while the link is still `Reaching`, `Securing` or `Greeting` the first time are held and go out once it
is `Ready`.

## Login

| Server asks for | Supported | Notes |
|-----------------|-----------|-------|
| Nothing (`trust`) | Yes | |
| Clear-text password | Yes | Sends the password as it is. Use only over TLS. |
| SCRAM-SHA-256 | Yes | The default of modern servers. The password never crosses the wire. No channel binding. |
| MD5 | No | Fails with `The server asks for a way of authenticating this link does not speak` |
| GSSAPI, SSPI, Kerberos, certificates | No | Same failure |

For SCRAM, the link also checks the server's final signature. A server that cannot prove it knows the password
breaks the link with `The server could not prove it knows the password`.

If the server refuses the login (wrong password, unknown database, no `pg_hba.conf` entry), the link breaks with
the server's message, and retries on the backoff schedule above. A wrong password therefore produces a log line
every 30 seconds until fixed.

Server set-up for a game role, run once as an administrator:

```sql
CREATE ROLE game LOGIN PASSWORD 'secret';
CREATE DATABASE game OWNER game;
```

With `password_encryption = 'scram-sha-256'` (the server default in recent versions), the role logs in with SCRAM.

## TLS

TLS uses the engine's `TLS` class, which is built on Windows (Schannel) and Linux (OpenSSL). On other platforms the
engine does not build it.

| Option | Encrypted | Certificate checked | Host name checked |
|--------|-----------|---------------------|-------------------|
| none | No | — | — |
| `tls=on` | Yes | No | No |
| `tls=verified` | Yes | Yes, against the system's trusted authorities | Yes, against `host` |
| `tls=verified&ca=file` | Yes | Yes, against the authorities in `file` only | Yes |

`tls=on` stops someone reading the traffic, not someone pretending to be the server. Use `tls=verified` across any
network you do not own. With `tls=verified` the `host` in the address must be the name on the server's certificate;
an IP address works only if the certificate lists it.

Failures:

| Error | Cause |
|-------|-------|
| `The server does not take an encrypted link` | The server answered the encryption request with `N` (TLS off on the server) |
| `The link could not be encrypted, or the server's certificate was refused` | Handshake failed, or verification failed |
| `The encrypted link broke while writing` | Encryption of outgoing bytes failed |

## Pipelining and batches

The link does not wait for one answer before sending the next query. It writes many queries ahead, up to a window of
512 queries and batches in the air. When the window fills, it waits until half are answered, then tops it up. That
means a server across the internet answers a thousand small inserts in a few round trips, not a thousand.

A batch goes out as one group of messages closed by one `Sync`. The server runs it as one implicit transaction:
if any statement fails, the server rolls back the whole batch and skips the rest. Every `Result` of the batch is
failed with the server's message.

A lone query is a batch of one, so it also runs in its own implicit transaction, unless you have sent `BEGIN`
yourself.

### How a statement is sent

| Query | Sent as |
|-------|---------|
| Has values | Prepared on the server once and kept, then bound and executed |
| No values, first time | A simple query; the server compiles it, runs it and forgets it |
| No values, second time and after | Prepared and kept, like one with values |
| No values, holds a `;` before its end | Always a simple query (may be several statements) |

Kept statements are named on the server `s0`, `s1`, …. Up to 512 are kept; past that the least recently used is
closed on the server too. Statements still used by a batch in the air are not closed, so the bound can be outgrown
briefly.

Column names and types of a kept statement are asked for once, then reused for every later result.

Consequences:

- Like SQLite, keep statement texts constant and pass changing parts as values.
- A text with a `;` inside a string literal and no values is sent as a simple query. That is harmless.
- A text with several statements **and** values is refused by the server: a prepared statement holds one command.

### Values on the wire

Every value goes as text, with no type attached:

| `Value` | Sent as |
|---------|---------|
| null | SQL `NULL` |
| integer | Its decimal digits |
| real | 17 significant digits, enough to read back the same `Real64` |
| text | The text, UTF-8 |
| bytes | `\x` followed by two hexadecimal digits per byte |

The server infers each placeholder's type from where it stands. When it cannot (for example `SELECT $1 + $2`), cast
in SQL: `$1::bigint`.

Bytes sent this way are read correctly into a `bytea` column. Sent into a `text` column they are stored as the
literal text `\x...`.

Results come back as text and are converted by column type; the table is in [queries.md](queries.md). A `bytea`
column is read in the `hex` output format. The startup message asks the server for `bytea_output = hex`, so bytes
come back right whatever the server's own setting is.

## Reconnect and state

When the link mends, it is a new session to the server:

- Every kept statement is forgotten and prepared again on first use.
- Session settings you made with `SET` are gone. Put them in the role instead: `ALTER ROLE game SET timezone = 'UTC'`.
- A transaction you opened with `BEGIN` is gone; the server rolled it back when the old link dropped.
- Temporary tables are gone.

Batches avoid all of this: they never span two sessions.

## Worked example: a chat history server

A server keeps the last messages of each room.

```sql
CREATE TABLE IF NOT EXISTS message (
    id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room   TEXT   NOT NULL,
    author TEXT   NOT NULL,
    body   TEXT   NOT NULL,
    at     BIGINT NOT NULL
);
CREATE INDEX IF NOT EXISTS message_room_at ON message (room, at DESC);
```

```cpp
class History final
{
public:

    explicit History(Text Address)
        : mLink { Database::Open(Address) }
    {
        mLink->Submit(Query(kSchema), Connection::Reply([](ConstRef<Result> Outcome)
        {
            if (!Outcome.IsOk())
            {
                LOG_E("History: {0}", Outcome.GetError());
            }
        }));
    }

    void Post(Text Room, Text Author, Text Body, SInt64 At)
    {
        mLink->Submit(Query("INSERT INTO message (room, author, body, at) VALUES ($1, $2, $3, $4)",
                            Room, Author, Body, At), Connection::Reply());
    }

    void Recent(Text Room, UInt32 Peer)
    {
        mLink->Submit(Query("SELECT author, body, at FROM message WHERE room = $1 ORDER BY at DESC LIMIT 50", Room),
                      Connection::Reply([this, Peer](ConstRef<Result> Outcome)
        {
            for (UInt32 Index = 0; Index < Outcome.GetRows(); ++Index)
            {
                Row Line = Outcome.GetRow(Index);

                const Text   Author = Line.Read<Text>();
                const Text   Body   = Line.Read<Text>();
                const SInt64 At     = Line.Read<SInt64>();
                SendToPeer(Peer, Author, Body, At);
            }
        }));
    }

    void Tick()
    {
        mLink->Poll();
    }

private:

    static constexpr Text kSchema =
        "CREATE TABLE IF NOT EXISTS message ("
        "  id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,"
        "  room   TEXT   NOT NULL,"
        "  author TEXT   NOT NULL,"
        "  body   TEXT   NOT NULL,"
        "  at     BIGINT NOT NULL);"
        "CREATE INDEX IF NOT EXISTS message_room_at ON message (room, at DESC);";

    void SendToPeer(UInt32 Peer, Text Author, Text Body, SInt64 At);

    Unique<Connection> mLink;
};
```

`SendToPeer` is your own network code. The schema text holds two statements and no values, so it goes as one simple
query, which the server runs as one transaction.

## Common mistakes and limits

| Mistake | Effect | Fix |
|---------|--------|-----|
| `postgres://host/db` with no user | The user name is empty and the server refuses the login | `postgres://user:password@host/db` |
| `tls=yes`, `ssl=on`, `sslmode=require` | Logged as not known and left out, so the link is plain | `tls=on` or `tls=verified` |
| Expecting queries to wait for the link | They fail at once while it is broken | Retry from the callback, or check the link first with a `SELECT 1` |
| Server set to MD5 passwords | Login fails | Use SCRAM: `password_encryption = 'scram-sha-256'` and reset the password |
| `numeric` columns for money | Arrive as `Real64` | `bigint` cents |
| `BEGIN` … `COMMIT` across frames | Lost if the link drops in between | Use a batch |
| `SET` session settings | Lost on reconnect | `ALTER ROLE … SET …` |
| A destructor on a hung server | Waits up to 10 seconds, then fails what is left with `The link was closed before the server answered` | Drain with `Poll()` before closing, and treat a refused last write as lost |

Not there:

- No error codes (SQLSTATE). `GetError()` is the server's message text (`M` field) only.
- No notices: server `NOTICE` and `WARNING` messages are ignored.
- No `LISTEN`/`NOTIFY`, `COPY`, or cancel requests.
- No binary result format; everything is text and converted.
- No connection pool; one `Connection` is one session.
- No timeout on a single query. Use `SET statement_timeout` on the role (`ALTER ROLE game SET statement_timeout =
  '5s'`).

## Where it lives

| Path | Holds |
|------|-------|
| `Module/Database.Postgres/Source/Database.Postgres/Postgres.hpp` | `Postgres`, `State`, constants `kWindow` (512), `kIdle` (100 ms), `kPause` (1 s), `kLimit` (30 s), `kChunk` (64 KiB), `kKeep` (512), `kSeen` (4096) |
| `Module/Database.Postgres/Source/Database.Postgres/Postgres.cpp` | `Parse` (address), `Greet`, `Authenticate`, `Encode` (pipelining), `Handle` (server messages), `Break` and `Mend` (reconnect), `Read` (type mapping) |
| `Module/Database.Postgres/Source/Database.Postgres/Scram.hpp`, `Scram.cpp` | The SCRAM-SHA-256 exchange |
| Engine `Engine/Source/Zyphryon.Base/Cryptography/TLS.hpp` | The TLS client used for `tls=` |
| Engine `Engine/Source/Zyphryon.Network/` | `Proactor`, `Socket`, `Endpoint` used for the socket |
