# Queries, values and results

This chapter covers what goes into a connection and what comes back: `Value`, `Query`, `Result`, `Row` and
`Shape`, the `$1` placeholder rule, and how each database's column types map to a `Value`.

## What it is for

A query is a SQL statement plus the values its placeholders take. The values travel separately from the text, so
a player name such as `Robert'); DROP TABLE hero;--` is stored as that literal text and never runs as SQL. Always
put changing data into placeholders, never into the statement text.

## Value

`Value` is one cell, either sent with a query or read back in a row. It holds exactly one of five things:

| Kind | Stored as | Built from | Checked with | Read with |
|------|-----------|------------|--------------|-----------|
| Nothing (SQL `NULL`) | empty | `Value()` | `IsNull()` | — |
| Whole number | `SInt64` | any integral type (`SInt32`, `UInt32`, `UInt64`, `Bool`, …) | `IsInteger()` | `GetInteger()` |
| Real | `Real64` | any real type (`Real32`, `Real64`) | `IsReal()` | `GetReal()` |
| Text | `Str` (copied) | `Text` (and anything that converts to it) | `IsText()` | `GetText()` |
| Bytes | `Sequence<Byte>` (copied) | `ConstSpan<Byte>` | `IsBytes()` | `GetBytes()` |

```cpp
class Value final
{
public:

    using Bytes = Sequence<Byte>;

    Value();                                    // null
    template<IsIntegral Type> Value(Type Number);  // widened to SInt64
    template<IsReal Type>     Value(Type Number);  // widened to Real64
    Value(Text Content);                        // copied
    Value(ConstSpan<Byte> Content);             // copied

    Bool IsNull() const;
    Bool IsInteger() const;
    Bool IsReal() const;
    Bool IsText() const;
    Bool IsBytes() const;

    SInt64          GetInteger() const;
    Real64          GetReal() const;
    Text            GetText() const;
    ConstSpan<Byte> GetBytes() const;
};
```

The getters are forgiving:

| Getter | Holds an integer | Holds a real | Holds text | Holds bytes | Null |
|--------|------------------|--------------|------------|-------------|------|
| `GetInteger()` | the number | the whole part (cut, not rounded) | 0 | 0 | 0 |
| `GetReal()` | widened | the number | 0.0 | 0.0 | 0.0 |
| `GetText()` | empty | empty | the text | empty | empty |
| `GetBytes()` | empty | empty | empty | the bytes | empty |

So `GetInteger()` on a text cell returns 0 rather than parsing the text. When a column could be either, check with
`IsInteger()` / `IsText()` first.

Notes on building values:

- `Bool` is an integral type, so `true` is stored as the integer 1 and `false` as 0. Both databases accept that for
  a boolean column (PostgreSQL reads the text `1` as true).
- `UInt64` above `kMaximum<SInt64>` wraps into a negative `SInt64`. Neither database has an unsigned 64-bit column
  anyway.
- Text and bytes are copied into the value. You may pass a `Text` view of a temporary string.
- `GetText()` and `GetBytes()` return views into the value. They live as long as the value, which for a result cell
  means as long as the `Result`.

### Passing bytes

Wrap fixed-size arrays and byte sequences in `ConstSpan<Byte>` so the intent is plain:

```cpp
const SHA256::Sum Digest = SHA256::Hash(Text("hello").GetBytes());

Link->Submit(Query("INSERT INTO blob_test (data) VALUES ($1)", ConstSpan<Byte>(Digest)), Connection::Reply());
```

### Passing null

Pass a default `Value()`:

```cpp
Link->Submit(Query("UPDATE hero SET guild = $1 WHERE name = $2", Value(), Name), Connection::Reply());
```

## Query

```cpp
class Query final
{
public:

    Query();
    template<typename... Arguments>
    Query(Text Statement, AnyRef<Arguments>... Parameters);

    Text             GetStatement() const;
    ConstSpan<Value> GetArguments() const;
};
```

The constructor copies the statement and turns each parameter into a `Value`, in order. The first parameter is `$1`,
the second `$2`, and so on.

```cpp
Query("SELECT level FROM hero WHERE name = $1", "Aldric");
Query("INSERT INTO score (player, points, at) VALUES ($1, $2, $3)", PlayerId, Points, Timestamp);
Query("CREATE TABLE IF NOT EXISTS hero (name TEXT PRIMARY KEY)");
```

A `Query` is moved into the connection by `Submit`, so build it in place or `Move` it.

### Placeholders

| Placeholder | SQLite | PostgreSQL |
|-------------|--------|------------|
| `$1`, `$2`, … | Yes. `$3` takes the third value wherever it stands. The same `$1` may appear twice. | Yes, native. |
| `?` | Yes, takes values in order | No, a syntax error |
| `:name`, `@name` | Takes values in order (by position) | No |

Write `$N` everywhere. Code then runs unchanged on both databases.

Placeholders stand for **values only**. They cannot name a table, a column, or a SQL keyword:

```cpp
// Wrong: a table name cannot be a placeholder.
Query("SELECT * FROM $1", "hero");

// Right: table names come from your own code, never from players.
Query("SELECT * FROM hero WHERE name = $1", Name);
```

Give exactly as many values as the highest `$N`. SQLite fails a query that has a `$N` beyond the values given with
`The statement has a placeholder no value was given for`. PostgreSQL fails it with the server's own message.

### Text and type inference on PostgreSQL

PostgreSQL receives every value as text with no type attached, and the server infers each placeholder's type from
where it stands. That works for comparisons, inserts and updates against typed columns. Where the server cannot
infer, cast:

```cpp
Query("SELECT $1::bigint + $2::bigint", A, B);
```

SQLite has no such issue; it stores what it is given.

### One statement or several

| Shape | Lone query, no values | Lone query, with values | In a batch |
|-------|-----------------------|-------------------------|------------|
| One statement | Yes | Yes | Yes |
| Several statements (`"A; B"`) | Yes, each runs in turn, and the `Result` is the last statement's | No, refused by both links | No |

Several statements in one text are for schema scripts:

```cpp
Link->Submit(Query(
    "CREATE TABLE IF NOT EXISTS hero (name TEXT PRIMARY KEY, level INTEGER NOT NULL);"
    "CREATE TABLE IF NOT EXISTS pack (owner TEXT NOT NULL, item TEXT NOT NULL);"
    "CREATE INDEX IF NOT EXISTS pack_owner ON pack (owner);"), Connection::Reply([](ConstRef<Result> Outcome)
{
    if (!Outcome.IsOk())
    {
        LOG_E("Schema: {0}", Outcome.GetError());
    }
}));
```

On SQLite each statement of such a text commits on its own, and the run stops at the first one that fails, leaving
earlier ones applied. On PostgreSQL the server runs the whole text as one implicit transaction. To get the same
"all or nothing" on both, write the script inside `BEGIN; ... COMMIT;`.

## Result

```cpp
class Result final
{
public:

    Bool            IsOk() const;
    Text            GetError() const;
    UInt64          GetAffected() const;
    UInt32          GetColumns() const;
    UInt32          GetRows() const;
    Text            GetName(UInt32 Column) const;
    Row             GetRow(UInt32 Index) const;
    ConstRef<Value> GetValue(UInt32 Row, UInt32 Column) const;

    // Used by the links to build a result. Your code does not call these.
    void Fail(Text Reason);
    void SetAffected(UInt64 Affected);
    void SetShape(ConstRetainer<Shape> Columns);
    void AddCell(AnyRef<Value> Cell);
};
```

| Method | Returns |
|--------|---------|
| `IsOk()` | `true` when the query ran, `false` when it was refused |
| `GetError()` | The database's reason, empty when the query ran |
| `GetAffected()` | Rows inserted, updated or deleted. 0 for a query that returns rows, including `INSERT ... RETURNING` |
| `GetColumns()` | Columns per row, 0 for a query that returns no rows |
| `GetRows()` | Number of rows read |
| `GetName(Column)` | The column's name as the database gives it (`level`, or the `AS` alias) |
| `GetRow(Index)` | A `Row` reader over that row |
| `GetValue(Row, Column)` | One cell, as a `Value` |

Rows and columns count from 0. `GetName`, `GetRow` and `GetValue` do not check their indices: an index past
`GetRows()` or `GetColumns()` reads out of bounds. Loop to `GetRows()` and `GetColumns()`.

A failed result has no rows and no columns, whatever it had read before the failure.

### Reading every cell

```cpp
for (UInt32 Index = 0; Index < Outcome.GetRows(); ++Index)
{
    for (UInt32 Column = 0; Column < Outcome.GetColumns(); ++Column)
    {
        ConstRef<Value> Cell = Outcome.GetValue(Index, Column);

        if (Cell.IsNull())
        {
            LOG_I("{0} = NULL", Outcome.GetName(Column));
        }
        else if (Cell.IsInteger())
        {
            LOG_I("{0} = {1}", Outcome.GetName(Column), Cell.GetInteger());
        }
        else if (Cell.IsReal())
        {
            LOG_I("{0} = {1}", Outcome.GetName(Column), Cell.GetReal());
        }
        else if (Cell.IsText())
        {
            LOG_I("{0} = '{1}'", Outcome.GetName(Column), Cell.GetText());
        }
        else
        {
            LOG_I("{0} = {1} bytes", Outcome.GetName(Column), Cell.GetBytes().GetSize());
        }
    }
}
```

### Columns by name

`Result` has no lookup by column name. Name the columns in your `SELECT` and read them by position. If you need a
name lookup, build it once from `GetName`:

```cpp
static UInt32 Column(ConstRef<Result> Outcome, Text Name)
{
    for (UInt32 Index = 0; Index < Outcome.GetColumns(); ++Index)
    {
        if (Outcome.GetName(Index) == Name)
        {
            return Index;
        }
    }
    return kMaximum<UInt32>;
}
```

`GetName` walks the shape's names from the start each call, so call it once per column, not once per row.

## Row

`Row` reads a row's cells one after another, from the first column to the last, converting each to the type you ask
for:

```cpp
class Row final
{
public:

    explicit Row(ConstSpan<Value> Cells);

    template<typename Type>
    Type Read();
};
```

| `Read<T>()` with T | Gives |
|--------------------|-------|
| `Bool` | `GetInteger() != 0` |
| any integral type (`UInt8` … `SInt64`) | `GetInteger()` cast to T |
| any real type | `GetReal()` cast to T |
| `ConstSpan<Byte>` | `GetBytes()`, a view |
| `Text` | `GetText()`, a view |
| `Str` | a copy of `GetText()` |

Every call moves to the next column. Reading past the last column trips an assertion.

```cpp
Link->Submit(Query("SELECT name, level, gold, portrait FROM hero WHERE name = $1", Name),
             Connection::Reply([this](ConstRef<Result> Outcome)
{
    if (!Outcome.IsOk() || Outcome.GetRows() == 0)
    {
        return;
    }

    Row Hero = Outcome.GetRow(0);

    mName  = Hero.Read<Str>();
    mLevel = Hero.Read<UInt32>();
    mGold  = Hero.Read<UInt64>();

    const ConstSpan<Byte> Portrait = Hero.Read<ConstSpan<Byte>>();
    mPortrait.Clear();
    mPortrait.Append(Portrait);
}));
```

The order of `Read` calls must match the order of columns in the `SELECT`. Write the column list out; avoid
`SELECT *` with `Row`, because adding a column to the table would shift every read after it.

A `Row` holds a view of the result's cells, so it must not outlive the `Result`.

## Shape

`Shape` is the list of column names of a statement's rows. The links build one when a statement first returns rows
and share it, by `Retainer<Shape>`, with every later result of that statement. Your code meets it only through
`Result::GetColumns()` and `Result::GetName()`.

```cpp
class Shape final : public Retainable<Shape>
{
public:

    void   AddColumn(Text Name);
    UInt32 GetColumns() const;
    Text   GetName(UInt32 Column) const;
};
```

## Type mapping

What a column comes back as, per database.

### SQLite

SQLite gives each cell its own storage class, whatever the column's declared type:

| Storage class | `Value` kind |
|---------------|--------------|
| `INTEGER` | integer |
| `REAL` | real |
| `TEXT` | text |
| `BLOB` | bytes |
| `NULL` | null |

What you send is stored as sent: an integer `Value` goes in as `INTEGER`, a real as `REAL`, text as `TEXT`, bytes as
`BLOB`. Column affinity may still convert (an integer into a `TEXT` column is stored as text). Declare columns with
the type you mean: `INTEGER`, `REAL`, `TEXT`, `BLOB`.

### PostgreSQL

The server sends every cell as text, and the link converts by the column's type number:

| PostgreSQL type | Type number | `Value` kind |
|-----------------|-------------|--------------|
| `boolean` | 16 | integer, 1 for true and 0 for false |
| `smallint`, `integer`, `bigint` | 21, 23, 20 | integer |
| `oid` | 26 | integer |
| `real`, `double precision` | 700, 701 | real |
| `numeric` | 1700 | real (may lose digits) |
| `bytea` | 17 | bytes |
| anything else (`text`, `varchar`, `timestamp`, `date`, `uuid`, `json`, `jsonb`, arrays, …) | — | text, as the server prints it |

Consequences:

- A `numeric` money column comes back as a `Real64`. Store currency as `bigint` cents instead.
- A `timestamp` comes back as text such as `2026-11-23 18:04:00`. Store times as `bigint` seconds or milliseconds if
  you want numbers, or convert in SQL: `SELECT extract(epoch FROM at)::bigint`.
- A `SELECT COUNT(*)` is `bigint`, so it comes back as an integer. `SELECT SUM(points)` over `integer` is `bigint` too,
  but over `bigint` it is `numeric`, which arrives as a real. Cast it: `SUM(points)::bigint`.

### Portable column types

| You want | SQLite | PostgreSQL |
|----------|--------|------------|
| A whole number | `INTEGER` | `BIGINT` (or `INTEGER`) |
| An automatic id | `INTEGER PRIMARY KEY` | `BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY` |
| Text | `TEXT` | `TEXT` |
| Bytes | `BLOB` | `BYTEA` |
| A real | `REAL` | `DOUBLE PRECISION` |
| A flag | `INTEGER` (0/1) | `BOOLEAN` |
| Time | `INTEGER` (seconds) | `BIGINT` (seconds) |

The DML (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `INSERT ... ON CONFLICT ... DO UPDATE SET x = excluded.x`,
`RETURNING`) is the same on both. Usually only the `CREATE TABLE` statements differ.

## Common mistakes and limits

| Mistake | Effect | Fix |
|---------|--------|-----|
| Building SQL with string concatenation | SQL injection, broken quotes | Use `$N` values |
| `?` placeholders | Works on SQLite, fails on PostgreSQL | Use `$N` |
| Several statements with values in one text | Refused by both links. SQLite fails it with `The statement has several statements, which take no values`, PostgreSQL with the server's message | One statement per query when it has values |
| `Read` calls out of column order | Values land in the wrong fields | Match the `SELECT` list |
| Keeping a `Text` from a result past the callback | Dangling view | Copy into `Str` |
| `GetValue` past `GetRows()` | Out-of-bounds read | Loop to `GetRows()` |
| `GetInteger()` on a text cell | Returns 0 silently | Check `IsInteger()`, or cast in SQL |
| Expecting `GetAffected()` from `INSERT ... RETURNING` | It is 0 | Count the returned rows with `GetRows()` |

## Where it lives

| Path | Holds |
|------|-------|
| `Module/Database.Core/Source/Database.Core/Value.hpp` | `Value` |
| `Module/Database.Core/Source/Database.Core/Query.hpp` | `Query` |
| `Module/Database.Core/Source/Database.Core/Result.hpp` | `Result` |
| `Module/Database.Core/Source/Database.Core/Row.hpp` | `Row::Read<T>` |
| `Module/Database.Core/Source/Database.Core/Shape.hpp` | `Shape` |
| `Module/Database.SQLite/Source/Database.SQLite/SQLite.cpp` | `SQLite::Bind` and `SQLite::Step`, the SQLite mapping |
| `Module/Database.Postgres/Source/Database.Postgres/Postgres.cpp` | `Postgres::Put(ConstRef<Value>)` and `Postgres::Read`, the PostgreSQL mapping |
