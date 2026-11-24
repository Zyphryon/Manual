# Digests: SHA-256, HMAC and PBKDF2

A database that holds accounts needs password hashes, and a server that hands out session tokens needs random bytes
and signatures. Zyphryon Database does not ship its own hashing: it uses the engine's `SHA256` class for its
PostgreSQL login, and your code uses the same class. This chapter documents that class and `Entropy`, and how to use
them for passwords and tokens stored through the library.

Both types are part of the engine's Base module, which every program linking `ZyDatabase` already has. They are in
the engine's inline namespace `ZyBase`, so no `using` is needed.

## SHA256

```cpp
class SHA256 final
{
public:

    static constexpr UInt32 kSize  = 32;   // bytes in a finished digest
    static constexpr UInt32 kBlock = 64;   // bytes worked through at a time

    using Sum = Array<Byte, kSize>;

    SHA256();
    void Update(ConstSpan<Byte> Data);
    Sum  Finish();

    static Sum Hash(ConstSpan<Byte> Data);
    static Sum Sign(ConstSpan<Byte> Key, ConstSpan<Byte> Data);
    static Sum Derive(ConstSpan<Byte> Password, ConstSpan<Byte> Salt, UInt32 Rounds);
};
```

| Member | Is | Use it for |
|--------|----|------------|
| `Hash(Data)` | SHA-256 | A fingerprint of a file or a save blob, to spot changes or corruption |
| `Sign(Key, Data)` | HMAC-SHA-256 | Proving a message or token came from someone who holds `Key` |
| `Derive(Password, Salt, Rounds)` | PBKDF2-HMAC-SHA-256, one 32-byte block | Turning a password into a key that is slow to guess |
| `SHA256()`, `Update`, `Finish` | Streaming SHA-256 | Hashing data that arrives in pieces |
| `Sum` | `Array<Byte, 32>` | The result of all of the above |

`Finish` ends the digest. Do not call `Update` after it.

Turning text into bytes: `Text::GetBytes()` gives a `ConstSpan<Byte>` over the text's UTF-8 bytes.

```cpp
const SHA256::Sum Digest = SHA256::Hash(Text("hello").GetBytes());
```

Streaming:

```cpp
SHA256 Digest;
Digest.Update(Header);
Digest.Update(Body);

const SHA256::Sum Total = Digest.Finish();
```

A `Sum` converts to `ConstSpan<Byte>`, so it can be passed to the next call or to a `Query`:

```cpp
const ConstSpan<Byte> Bytes = Total;
```

## Entropy

```cpp
class Entropy final
{
public:

    static Bool   Gather(Span<Byte> Output);
    static UInt64 Draw();
};
```

| Member | Gives |
|--------|-------|
| `Gather(Output)` | Fills `Output` with bytes from the operating system's secure random source. Returns `false` if the system had none, leaving the bytes untouched. |
| `Draw()` | One random `UInt64`, or 0 when the system had none. |

Use `Entropy` for salts and tokens, never a game random number generator. Always check `Gather`'s result.

```cpp
Array<Byte, 16> Salt { };

if (!Entropy::Gather(Salt))
{
    LOG_E("Accounts: The system gave no random bytes");
    return;
}
```

## Storing a password

Never store a password, and never store a plain `Hash` of one. A fast hash lets an attacker who copies the table
try billions of guesses a second. Store:

1. a random **salt**, different for every account,
2. the **round count** used,
3. the result of `Derive(Password, Salt, Rounds)`.

```cpp
struct Secret
{
    Array<Byte, 16> Salt { };
    UInt32          Rounds = 0;
    SHA256::Sum     Key { };
};

static Bool Seal(Text Password, UInt32 Rounds, Ref<Secret> Output)
{
    if (!Entropy::Gather(Output.Salt))
    {
        return false;
    }
    Output.Rounds = Rounds;
    Output.Key    = SHA256::Derive(Password.GetBytes(), Output.Salt, Rounds);
    return true;
}
```

Checking a password reads the stored salt and rounds, derives again, and compares in constant time, so the time the
check takes does not tell an attacker how many bytes matched:

```cpp
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
```

The library's own SCRAM code compares the server's signature the same way.

### Choosing the round count

`Derive` runs HMAC-SHA-256 `Rounds` times. More rounds cost an attacker more per guess, and cost your server the
same per login. Current guidance for PBKDF2-HMAC-SHA-256 is in the hundreds of thousands of rounds. Measure
`Derive` on your server hardware and pick the largest count that keeps a login within your budget. Store the count
with each account, so you can raise it later and rehash accounts as their owners log in.

`Derive` is not free. At high round counts it takes long enough to be felt in a frame, so do not run it on the
thread that renders. On a server, run it on a worker (an engine job or thread) and hand the result back to the game
thread.

`Derive` produces one 32-byte block, which is the right length for a password key.

## Session tokens

A login can hand the client a token it sends back on later requests. Two ways:

**Random token stored in the database.** Simple and easy to revoke:

```cpp
Array<Byte, 32> Token { };
Entropy::Gather(Token);

// Store only a hash of the token, so a copied table does not give away live sessions.
const SHA256::Sum Stored = SHA256::Hash(Token);

Link->Submit(Query("INSERT INTO session (token, account, expires) VALUES ($1, $2, $3)",
                   ConstSpan<Byte>(Stored), AccountId, Expires), Connection::Reply());
```

When the client presents the token, hash it and look the hash up.

**Signed token, nothing stored.** The server keeps one secret key and signs the account id and expiry:

```cpp
static SHA256::Sum SignTicket(ConstSpan<Byte> ServerKey, SInt64 Account, SInt64 Expires)
{
    Array<Byte, 16> Fields { };

    for (UInt32 Index = 0; Index < 8; ++Index)
    {
        Fields[Index]     = static_cast<Byte>(static_cast<UInt64>(Account) >> (56 - Index * 8));
        Fields[Index + 8] = static_cast<Byte>(static_cast<UInt64>(Expires) >> (56 - Index * 8));
    }
    return SHA256::Sign(ServerKey, Fields);
}
```

The client sends back the account, the expiry and the signature. The server signs again and compares in constant
time. Revoking such a ticket before it expires needs a database check anyway, so prefer the stored token unless you
have a reason.

## Storing digests in the database

Store digests and salts as bytes:

| Database | Column type |
|----------|-------------|
| SQLite | `BLOB` |
| PostgreSQL | `BYTEA` |

Pass them as `ConstSpan<Byte>`:

```cpp
Link->Submit(Query("UPDATE account SET salt = $1, rounds = $2, secret = $3 WHERE id = $4",
                   ConstSpan<Byte>(Sealed.Salt), Sealed.Rounds, ConstSpan<Byte>(Sealed.Key), Id),
             Connection::Reply());
```

Read them back with `Row::Read<ConstSpan<Byte>>()` or `Value::GetBytes()`. The cookbook has the full account flow.

## How the library itself uses SHA256

The PostgreSQL link's `Scram` class logs in with SCRAM-SHA-256. It uses `Derive` to stretch the password with the
server's salt and round count, `Sign` for the client and server keys and the proof, `Hash` for the stored key, and
`Entropy::Gather` for its 18-byte nonce. You do not call `Scram` yourself; it is listed here so you know the hashing
you use is the same code the library trusts for its own login.

## Common mistakes

| Mistake | Why it is wrong | Instead |
|---------|-----------------|---------|
| Storing `Hash(Password)` | Fast to guess, same for equal passwords | `Derive` with a per-account salt |
| One salt for all accounts | Equal passwords give equal keys | `Entropy::Gather` a new salt per account |
| Comparing digests with `==` or a loop that returns early | Timing leaks the match length | Constant-time loop as above |
| Ignoring `Gather`'s result | A failed gather leaves zeros | Check it, fail the operation |
| Running `Derive` with high rounds on the render thread | A visible stall | Run on a worker |
| Hashing a password client-side and sending the hash | The hash becomes the password | Send the password over TLS; derive on the server |

## Where it lives

| Path | Holds |
|------|-------|
| Engine `Engine/Source/Zyphryon.Base/Cryptography/SHA256.hpp` | `SHA256`, `Hash`, `Sign`, `Derive` |
| Engine `Engine/Source/Zyphryon.Base/System/Entropy.hpp` | `Entropy::Gather`, `Entropy::Draw` |
| `Module/Database.Postgres/Source/Database.Postgres/Scram.cpp` | The library's own use, for the PostgreSQL login |
