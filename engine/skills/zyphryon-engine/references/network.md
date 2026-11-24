# Networking and replication

This chapter teaches the two engine modules a networked program stands on: `Zyphryon.Network` (namespace
`ZyNetwork`: transports, endpoints, connections, delivery) and `Zyphryon.Replication` (namespace `ZyReplication`:
the service that copies a scene from a serving side to the sides following it). It covers their real API, the wire
protocol, how changes are found and sent, persistent replicas, owner-only data, and the rules that keep a follower
safe against a hostile server and a server safe against hostile clients. It leans on `scene.md` for entities,
components, pull lists and deferral, on `base.md` for `Writer`, `Reader`, `Archive` and `Retainer`, and on
`runtime.md` for the kernel, the command line and headless starts.

## 1. What the two modules are for

```
your program
   |  Replicate<T>, Serve, Join, Watch, Possess, Send, Find, Listener callbacks
   v
ZyReplication::Service          one per process, registered by the kernel after the scene
   |-- Publisher   (serving side)   reads the scene's pull lists, builds one frame stream per peer
   |-- Subscriber  (following side) applies frames to its own scene
   |-- Manifest                     the table of component types that travel
   v
ZyNetwork::Service              endpoints and connections, run on a thread of their own
   |-- TCP: Stream, Pipe            reliable ordered frames, ping and timeout
   |-- UDP: Session, Sentry         unreliable datagrams behind an address challenge
   v
Driver: Winsock / Posix / Emscripten (WebSocket, TCP only)
```

Use `ZyNetwork` alone for anything that is not a shared world: a login service, a chat relay, a tool talking to a
running game. Use `ZyReplication` when one process owns a scene and others must see it.

The kernel (`Zyphryon.Runtime/Kernel.cpp`) creates both services for every program, headless or not. It registers
`ZyNetwork::Service` and calls `Initialize()`; when that fails it unregisters it and logs
`Kernel: Failed to initialize network service`, and replication then refuses to open a session. It registers
`ZyReplication::Service` after `ZyScene::Service`, so replication ticks after the scene each frame: the world moves,
then the publisher sends what changed.

## 2. Zyphryon.Network

### 2.1 Names

| Name | What it is |
|---|---|
| `Service` | The engine subsystem. Opens endpoints, sends, closes. Its `OnTick` hands the program everything the network thread heard since the last frame. |
| Endpoint | One listening or connecting socket set, named by a `Connection` with no peer half. At most `kMaxEndpoints` = 128. |
| `Connection` | A handle naming an endpoint, or one peer of an endpoint. `GetValue()` packs it into a `UInt64` (endpoint high 32 bits, peer low 32). `IsValid()`, `IsPeer()`, `GetEndpoint()`. |
| `Handler` | What the program implements to hear one endpoint: `OnConnect`, `OnDisconnect`, `OnMessage`. |
| `Address` | `{ Str Host; UInt16 Port; }`. An empty host listens on every interface (`IsAny()`). A zero port is refused (`IsValid()`). |
| `Transport` | `TCP` or `UDP`. |
| `Delivery` | `Unreliable`, `Unordered`, `Reliable`. |
| `Reason` | Why a connection ended: `None`, `Closed`, `Refused`, `Unreachable`, `Timeout`, `Protocol`, `Congested`. |
| `Statistics` | `Latency` (ms), `Loss` (0..1), `Pending`, `Sent`, `Received` (bytes). |
| `Signal<Type, Opcode>` | Helper for a value one side keeps telling the other, newest wins (section 2.6). |

### 2.2 The Service API

```cpp
// Zyphryon.Network/Service.hpp
Bool       Initialize();
Connection Listen(ConstRef<Address> Address, Transport Transport, ConstRetainer<Handler> Listener);
Connection Connect(ConstRef<Address> Address, Transport Transport, ConstRetainer<Handler> Listener);
void       Close(Connection Link);
void       SetTimeout(Connection Link, Real64 Seconds);
void       Send(Connection Link, Delivery Delivery, ConstSpan<Byte> Message);
void       Broadcast(Connection Link, Delivery Delivery, ConstSpan<Byte> Message);
```

- `Listen` and `Connect` return the endpoint's `Connection` at once, or an invalid one when the address names no
  port or no slot is left (both logged). The address resolves away from the program's thread. Success or failure
  arrives later as `OnConnect` or `OnDisconnect`.
- `Close` on a peer ends it once what it was sent over TCP has gone out. `Close` on the endpoint ends it at once.
- `SetTimeout` sets how long a peer (or every peer of an endpoint) may say nothing before it is taken as gone. The
  default is `Channel::kMaxSilence` = 10 s. Zero never gives up.
- `Send` copies the payload, so the caller may free it at once. A payload over `kMaxFrame` (1 MiB) is dropped with
  `Network: a message of {0} byte(s) was dropped for being too large`.
- `Broadcast` sends to every peer of an endpoint.

Every callback runs on the program's thread, inside `Service::OnTick`. The sockets run on the `Executor`'s thread,
which wakes at least every `kMaxIdle` = 100 ms. Nothing in a handler needs locking.

### 2.3 Handler

```cpp
class Handler : public Retainable<Handler>
{
public:
    virtual void OnConnect(Connection Link, ConstRef<Address> Origin);
    virtual void OnDisconnect(Connection Link, Reason Reason);
    virtual void OnMessage(Connection Link, Delivery Delivery, ConstSpan<Byte> Message);
};
```

On a listening endpoint, `OnConnect` fires once for the endpoint itself (`Link.IsPeer()` is `false`) and once per
peer that reaches it. On a connecting endpoint it fires once the peer is reached. `Message` is valid only during the
call; copy what you keep.

### 2.4 TCP

TCP is the transport to use for anything that must arrive, and the only one that carries `Reliable` today.

- Each message is a frame: a 4-byte length (`Pipe::kHeader`) then a tag byte, `Message`, `Ping` or `Pong`, then the
  payload. Every message arrives `Reliable`, whatever delivery was asked for.
- A connect attempt that takes longer than `Stream::kMaxHandshake` = 10 s ends as `Unreachable`.
- A live stream pings every third of its timeout. Only a whole frame proves the peer alive, so a peer trickling a
  frame byte by byte still times out (`Timeout`).
- A frame that announces more than `kMaxFrame` (1 MiB), or carries no tag byte, ends the peer with `Protocol`.
- A peer that stops reading while more than `kMaxBacklog` (1 MiB) waits to go out ends with `Congested`.
- A listening endpoint keeps at most `Server::kMaxInvites` = 32 accepts in flight and up to `kMaxPeers` = 4096 peers.
- In a browser build (Emscripten driver) the socket is a WebSocket. UDP is refused there:
  `Network: the browser carries no datagrams, so a socket for them cannot be opened`.

### 2.5 UDP

UDP exists for unreliable traffic and is not yet a full transport:

- A datagram is at most `kMaxDatagram` = 1200 bytes, envelope included. A message must fit one datagram
  (`Session::kPayload`).
- A stranger must answer a challenge before the endpoint writes anything down about it. `Sentry` derives the cookie
  from the address under a key that rotates every `Sentry::kRotation` = 15 s, which stops spoofed floods.
- Only `Delivery::Unreliable` is carried yet. A `Reliable` or `Unordered` send is refused with
  `Network: a message that must arrive was refused, which this build cannot promise yet`.

Use TCP for anything replication or gameplay needs. Use UDP only for traffic where losing a packet is fine, such as a
voice stream or a position hint, and pair it with `Signal`.

### 2.6 Signal

```cpp
template<typename Type, typename Opcode = UInt8>
class Signal
{
public:
    static constexpr Real64 kRepeat = 0.25;

    explicit Signal(Opcode Code);
    void           Set(ConstRef<Type> Value);
    ConstRef<Type> Get() const;
    Bool           Poll(Ref<Type> Value);
    Bool           Tick(Real64 Delta, Ref<Writer> Output);
    Bool           Receive(ConstSpan<Byte> Message);
};
```

The sender calls `Set` whenever the value changes and `Tick` every frame. `Tick` writes `[opcode][UInt16
sequence][value]` when the serialized bytes changed or `kRepeat` passed, and returns `true` when it wrote. Send that
unreliably. The receiver hands every message to `Receive`, which keeps the value only when its sequence is newer, and
reads it with `Poll` (`true` when a newer value arrived). The value is serialized with `Archive`, so `Type` needs
`Serialize` or must be trivially copyable.

```cpp
ZyNetwork::Signal<Vector3> mAim { 1 };

// Sender, every frame.
mAim.Set(Aim);

Writer Output;
if (mAim.Tick(Delta, Output))
{
    Network.Send(mLink, ZyNetwork::Delivery::Unreliable, ConstSpan<Byte>(Output.GetData(), Output.GetSize()));
}

// Receiver, in OnMessage.
if (mAim.Receive(Message) && mAim.Poll(Aim))
{
    // Aim holds the newest value
}
```

### 2.7 Worked example: a chat relay over bare TCP

A complete program. Run it with `--serve` to relay, or with `--join 127.0.0.1` to connect; each line a client sends
goes to every client. Messages are a length-prefixed text written with `Writer::WriteText`.

```cpp
// Chat/Main.cpp
#include <Zyphryon.Network/Service.hpp>
#include <Zyphryon.Runtime/Entry.hpp>

namespace Chat
{
    /// \brief Represents the side of the chat this process plays, relay or speaker.
    class Peer final : public ZyNetwork::Handler
    {
    public:

        /// \brief Constructs the handler over the network service.
        ///
        /// \param Network The network service, which outlives the handler.
        explicit Peer(Ref<ZyNetwork::Service> Network)
            : mNetwork { Network }
        {
        }

        /// \copydoc ZyNetwork::Handler::OnConnect
        void OnConnect(ZyNetwork::Connection Link, ConstRef<ZyNetwork::Address> Origin) override
        {
            if (Link.IsPeer())
            {
                LOG_I("Chat: {0}:{1} joined", Origin.Host, Origin.Port);
            }
            mLink = Link;
        }

        /// \copydoc ZyNetwork::Handler::OnDisconnect
        void OnDisconnect(ZyNetwork::Connection Link, ZyNetwork::Reason Reason) override
        {
            LOG_I("Chat: a connection ended ({0})", static_cast<UInt32>(Reason));
        }

        /// \copydoc ZyNetwork::Handler::OnMessage
        void OnMessage(ZyNetwork::Connection Link, ZyNetwork::Delivery, ConstSpan<Byte> Message) override
        {
            Reader Input(Message);
            const Text Line = Input.ReadText();

            // A message that is not exactly one line of at most 255 characters ends the peer.
            if (Line.IsEmpty() || Line.GetSize() > 255 || Input.GetAvailable() != 0)
            {
                mNetwork.Close(Link);
                return;
            }

            if (Link.IsPeer() && mServing)
            {
                // The relay passes the line, as it was sent, to every peer of its endpoint.
                mNetwork.Broadcast(Link.GetEndpoint(), ZyNetwork::Delivery::Reliable, Message);
            }
            else
            {
                LOG_I("Chat: {0}", Line);
            }
        }

        /// \brief Sends one line to the relay.
        ///
        /// \param Line The text, at most 255 characters.
        void Say(Text Line)
        {
            Writer Output;
            Output.WriteText(Line);
            mNetwork.Send(mLink, ZyNetwork::Delivery::Reliable, ConstSpan<Byte>(Output.GetData(), Output.GetSize()));
        }

        /// \brief Sets whether this process relays.
        ///
        /// \param Serving `true` to relay what arrives, `false` to print it.
        void SetServing(Bool Serving)
        {
            mServing = Serving;
        }

    private:

        Ref<ZyNetwork::Service> mNetwork;
        ZyNetwork::Connection   mLink;
        Bool                    mServing = false;
    };

    /// \brief Represents the program.
    class Application final : public ZyRuntime::Kernel
    {
    protected:

        /// \copydoc ZyRuntime::Kernel::OnConfigure
        void OnConfigure(Ref<ZyRuntime::Startup> Startup) override
        {
            // A relay opens no window.
            Startup.SetHeadless(GetEnvironment().Contains("serve"));
        }

        /// \copydoc ZyRuntime::Kernel::OnInitialize
        Bool OnInitialize() override
        {
            const Retainer<ZyNetwork::Service> Network = GetService<ZyNetwork::Service>();

            if (!Network)
            {
                return false;
            }
            mPeer = Retainer<Peer>::Create(* Network);

            if (GetEnvironment().Contains("serve"))
            {
                mPeer->SetServing(true);
                return Network->Listen({ .Port = 9000 }, ZyNetwork::Transport::TCP, mPeer).IsValid();
            }

            const ZyNetwork::Address Server { .Host = GetEnvironment().GetText("join", "127.0.0.1"), .Port = 9000 };
            return Network->Connect(Server, ZyNetwork::Transport::TCP, mPeer).IsValid();
        }

    private:

        Retainer<Peer> mPeer;
    };
}

ZY_APPLICATION(Chat::Application)
```

The relay closes any peer that sends something that is not one short line: that check is the program's half of the
safety rules in section 10.

## 3. Zyphryon.Replication: the model

### 3.1 One serving side, many following sides

A session has one side that **serves** (owns the world and decides it) and any number that **follow** (apply what
it says). `Service::IsServing()` and `IsFollowing()` say which this process is. Both are `false` before a session
opens. The serving side runs a `Publisher`, a following side a `Subscriber`. Both are internal (`Service.hpp` only
forward-declares them).

### 3.2 Names

| Name | Header | What it is |
|---|---|---|
| `Service` | `Service.hpp` | The subsystem a program talks to. |
| `Manifest` | `Manifest.hpp` | The table of component types that travel, with their audience, rate and adapter. |
| `Locus` | `Locus.hpp` | Component: the interest key an entity travels under. Setting it makes the entity a replica. `Locus()` (key 0) means "under my root's key". |
| `Replica` | `Replica.hpp` | Component: the identifier a replica travels under on every side. Bit 63 (`kPersistent`) marks one both sides load from disk. |
| `Owner` | `Owner.hpp` | Component: the peer that owns a replica and alone receives its `Owner` components. |
| `Possessed` | `Common.hpp` | Tag: on a following side, the replica this side owns and steers. |
| `Remote` | `Common.hpp` | Tag: on a following side, a replica another side decides for. |
| `Server`, `Client` | `Common.hpp` | Tags that keep an entity to one side. Neither ever travels. |
| `Audience` | `Common.hpp` | `Everyone`, `Owner`, `Others`. |
| `Link` | `Link.hpp` | A reference to another replica, held inside a replicated component. |
| `Ledger<T>` | `Ledger.hpp` | A list of plain slots that marks each slot it changes, so a delta carries only those. |
| `Listener` | `Listener.hpp` | What the program implements to hear joins, leaves, messages and waits. |
| Interest key | | A `UInt64` the program chooses. Peers watch keys. `kGlobal` (0) and `kPrivate` (max) are reserved. |
| Section | | The part of a frame carrying one key's messages. |
| Catch-up | | Everything under a key, written whole for a peer that starts watching it. |

`Service`'s constructor declares `Replica` and `Locus` as `.Serializable().Local()` (saves carry them, instances never
take them from an archetype), `Owner` as `.Local()` (no save outlives a live peer), and the four tags as `.Local()`.

### 3.3 Keys, roots and children

- A **root** is a replica whose `Locus` holds a non-zero key. It is a member of that key's channel.
- A **child** is a replica with `Locus()` under some ancestor that has a key. It travels under its nearest keyed
  ancestor's key, in that root's channel, and arrives after its parent.
- Parts (`Entity::IsPart()`) never travel on their own. They come with the follower's own archetype.
- Keys mean nothing to the protocol. The program picks them: one per level, one per room, one per cell of a grid
  over a large map. A peer sees a replica only while it watches the replica's key.

### 3.4 Persistent and runtime replicas

| | Persistent | Runtime |
|---|---|---|
| Identifier | Authored or computed, bit 63 set | Given by the publisher, counting up from 1, never reused |
| How a follower gets it | Loads it from its own disk | `Enter` on the wire |
| What travels | Only components that diverged from disk, as `Update` | Everything replicated, then changes |
| Can it `Enter`, `Forget`, move key? | Never. A frame claiming so refuses the publisher | Yes |
| Can it have an `Owner`? | No (asserts) | Yes |
| Saved? | `Replica` writes its identifier | `Replica` writes zero |

Persistent replicas fit content both sides ship with, such as the doors and chests of a level: the follower loads the
level itself, and only what the server changed travels. The publisher remembers, per persistent replica, every
component that changed or was removed since load (its diverged mask). A peer that starts watching its key gets those
components in its catch-up, so it ends up where the server is, not where the disk was.

A world loaded while a session is open must be loaded inside `Service::Restore`:

```cpp
Replication.Restore([&]
{
    // Every side loads the same level from disk here, and marks its root persistent under the level's key.
    Scene.Merge(Bytes, Level);
    Level.Set(ZyReplication::Replica(ZyReplication::kPersistent | Key));
    Level.Set(ZyReplication::Locus(Key));
});
```

`Restore` reads pending changes first, runs the load, then reads again with the load's values counting as the
baseline (no change). On a follower it then takes in the new persistent replicas and applies what waited for them.

## 4. The Service API

### 4.1 Declaring what travels

```cpp
template<typename Type>
void Replicate(Audience Audience, UInt8 Rate = 1);
void Replicate(UInt32 Identifier, Audience Audience, UInt8 Rate, ConstRef<Manifest::Adapter> Adapter);
void Forget(UInt32 Identifier);
```

- Call it alike on both sides **before** a session opens. `Serve` and `Join` seal the table, and registering after
  that asserts.
- `Rate` is the number of ticks between two reads of the component's changes (at least 1). Changes merge in the
  pull list until the component's turn. Places in the table spread the turns, so components with the same rate do
  not all read on the same tick.
- The component must save and load itself (a `Serialize(Archive)` member, or trivially copyable), or be a tag. It
  must be declared to the scene first (`scene.md`).
- Code in another module (a library loaded at runtime) passes `Manifest::Bind<Type>()` made where the type is known.
  `Forget` takes a type back out before sealing, as such a library unloads.
- At most `kMaxComponents` = 128 types.

Sealing sorts the table by type name, so the order of declaration never reaches the wire. The hash both sides
compare is built from each entry's name, size and audience. Rate is not part of it.

### 4.2 Opening and closing

```cpp
Bool Serve(ConstRef<ZyNetwork::Address> Address, ZyNetwork::Transport Transport, ConstRetainer<Listener> Listener);
Bool Join(ConstRef<ZyNetwork::Address> Address, ZyNetwork::Transport Transport, ConstRetainer<Listener> Listener,
          ConstSpan<Byte> Ticket);
void Close();
```

Both return `false` when a session is open already, the network service is missing, or the endpoint could not
open. `Join` shows `Ticket` (at most `kTicket` = 1024 bytes) before anything else. `Close` drops the publisher or
subscriber and closes the endpoint. A null listener is replaced by a silent one.

### 4.3 Serving

```cpp
void Watch(UInt64 Peer, ConstSpan<UInt64> Keys);
void Possess(UInt64 Peer, ZyScene::Entity Actor);
void Expel(UInt64 Peer, Text Reason = Text::Empty());
ConstPtr<ZyNetwork::Address> GetAddress(UInt64 Peer) const;
void Send(UInt64 Peer, ConstSpan<Byte> Message);
void Send(ZyScene::Entity Actor, Audience Audience, ConstSpan<Byte> Message);
Bool IsWatched(ZyScene::Entity Actor);
```

| Call | What it does |
|---|---|
| `Watch` | Replaces the keys a peer watches. `kGlobal` and `kPrivate` are implied and skipped. New keys get their catch-up in the next frame, dropped keys get a `Leave`. |
| `Possess` | Sets `Owner(Peer)` on the entity, or removes it for peer 0. The publisher reads the change and makes the replica arrive again whole, with its owner part to the new owner. |
| `Expel` | Sends `Expel` with the reason (cut at 255 characters) and closes the peer once it went out. The listener hears `OnLeave` when the connection closes. |
| `Send(Peer, ...)` | A message in that peer's private section of its next frame, after the world changes before it. Up to 32 KiB. |
| `Send(Actor, Audience, ...)` | A message in the channel of the key the entity stands under, so it arrives in order with the entity's changes. `Owner` sends it privately to the owner, `Others` tells the owner's side to pass over it. Pass the world entity to reach every peer. An entity never published reaches nobody. |
| `IsWatched` | `true` when some peer watches the key the replica was published under. Use it to skip building messages nobody reads. |

### 4.4 Following

```cpp
void Send(ConstSpan<Byte> Message);    // to the serving side, reliable and in order
```

A following side has one route up: its messages arrive on the server as `Listener::OnMessage(Peer, ...)`.
Everything else a follower learns arrives as frames.

### 4.5 Both sides

```cpp
ZyScene::Entity Find(UInt64 Identifier);
ZyScene::Entity Resolve(Ref<Link> Value);
Bool IsServing() const;
Bool IsFollowing() const;
template<typename Callable> void Restore(AnyRef<Callable> Load);
```

`Find` returns nothing while no session is open. On a follower it also lets go of an identifier whose entity the
program destroyed.

### 4.6 Listener

```cpp
class Listener : public Retainable<Listener>
{
public:
    virtual Bool OnKnock(UInt64 Peer, ConstSpan<Byte> Ticket);   // serving: let the peer in?
    virtual void OnJoin(UInt64 Peer);                            // serving: peer joined; following: server reached (0)
    virtual void OnLeave(UInt64 Peer, Text Reason);              // serving: peer left; following: server gone (0)
    virtual void OnMessage(UInt64 Peer, ConstSpan<Byte> Message);// a message, from a peer or from the server (0)
    virtual void OnAwait(UInt64 Identifier);                     // following: own body waits for this persistent parent
};
```

- `OnKnock` sees the ticket before the peer is sent anything. Returning `false` tells the peer `Ticket` and closes
  it. Only the first message of a waiting peer is a ticket. A peer that never got past it is never heard leaving.
- `OnJoin` runs with the key the peer is known by from then on (the connection's `GetValue()`, never 0).
- `OnMessage` on the server only hears admitted peers.
- `OnAwait` fires when a replica this side owns arrives under a persistent parent not loaded yet. Load the level
  that parent lives in (inside `Restore`), and the held arrival replays once it lands.

## 5. How the publisher finds and sends changes

### 5.1 The tick

`Service::OnTick` calls `Publisher::Publish` once per frame while serving:

1. **Read.** Walk the pull lists. `Locus` removals retire replicas (they `Forget` on the wire), `Locus` sets place
   them (a new runtime replica gets its identifier here, at once, so a `Link` written this tick names it), `Owner`
   changes mark replicas to arrive again, and each replicated component's `Set` and `Removed` lists mark dirty and
   dropped bits on the tracker of each entity. A change under a key nobody watches marks nothing (a persistent
   replica still records it as diverged).
2. **Process.** Each queued replica is written into its channel:
   - first publish under a key: an arrival, which writes `Enter` for it and every child below it,
   - key changed: its pending changes go out on the old key first, then `Forget(id, new key)` on the old key, then
     an arrival under the new one,
   - parent or root status changed, or owner changed: pending changes first, then `Enter` again in the same channel,
   - otherwise: an `Update` with what changed and what was dropped, shared components into the key's stream and
     `Owner` components into the owner's private stream.
3. **Singletons.** Changes on the world entity go out as `Update` with identifier 0 in the global channel.
4. **Deliver.** Catch-ups are built once per key that a peer started watching, then each peer's frames are
   assembled on the job workers: greeting (first frame), global section, `Leave` for dropped keys, catch-ups for new
   keys, each watched key's stream, and the private section.
5. Channels nobody watches and nothing stands under are dropped.

A peer with nothing new still gets an empty frame every `kHeartbeat` = 1 s, which carries the clock.

### 5.2 Deltas

A component may travel as a delta instead of whole by giving its `Serialize` a second parameter:

```cpp
template<typename Serializer>
ZY_INLINE void Serialize(Serializer Archive, Bool Delta = false)
{
    mValues.Serialize(Archive, Delta);
}
```

`Manifest::Bind` detects it (`IsIncremental<Type>`) and fills the adapter's emit and apply functions. Only updates
write deltas, for components the peers already hold and that were not dropped this tick. `Enter`, catch-ups and the
owner's whole copy always write the component whole (`Delta = false`). A whole write keeps the marks, since peers
that hold the value still wait for the delta.

`Ledger<T>` is the ready-made delta container. It marks each slot changed through `Set` (only when the bytes
differ), `Edit`, `Append`, `Remove` or `Resize`, and its delta write sends the count, then each marked slot behind its
index, then forgets the marks. `IsDirty()` says whether any slot is marked. `T` must be trivially copyable. A ledger
holds at most 4096 slots (`kLimit`).

The contract: a delta carries absolute values of what changed, and a fresh instance counts everything as changed.
Writing a component in place still needs a notify (section 5.3), or the publisher never looks at it.

### 5.3 What counts as a change

Only what reaches the component's `Set` or `Removed` pull list:

- `Actor.Set(Value)`, `Add`, `Remove`, a spawn or a destroy,
- `Actor.Notify<T>()` after an in-place write (inside a spread walk the notify is queued and recorded when the walk
  ends),
- a walk that declared `.Writes<T>()`, which marks every row it visited.

A plain `Ref<T>` write in a walk records nothing, so it never travels. See `scene.md`, section 6.

## 6. The wire protocol

Every message a publisher sends a peer is a **frame**:

```
Frame    [UInt8 Opcode::Frame][Real64 clock][Real32 multiplier] message message ...
Packed   [UInt8 Opcode::Packed][UInt32 unpacked size][LZ4 bytes]       unpacks to one Frame
```

A frame is cut at `kMaxChunk` = 32 KiB, always between two sections, and packed with LZ4 when that saves more than
the 5 bytes the packed header costs.

| Opcode | Body | Meaning |
|---|---|---|
| `Frame` (0) | clock, multiplier | Opens a frame. The follower's world clock follows it. |
| `Hello` (1) | `UInt64` manifest hash, `UInt64` peer key | First message a peer ever gets, followed by the global catch-up. |
| `Leave` (2) | varint key | The peer no longer sees the key. |
| `Section` (3) | varint key | The messages up to the next section belong to this key (or `kGlobal`, `kPrivate`). |
| `Enter` (4) | varint id, varint parent id, varint archetype slot, varint owner peer, set | A replica whole: create it from that archetype under that parent, or overwrite the one held. |
| `Update` (5) | varint id (0 = world), set | What changed and what was removed. |
| `Forget` (6) | varint id, varint new key or 0 | The replica left this key (for the new key), or was destroyed (0). |
| `Message` (7) | varint skipped peer, varint size, bytes | A program message. The skipped peer (an `Others` message's owner) passes over it. |
| `Packed` (8) | size, LZ4 | Only ever a whole frame, never inside one. |
| `Knock` (9) | ticket bytes | Follower's first message to the server. |
| `Expel` (10) | text | Server's last message to a follower. |

A **set** is one length-prefixed block (`UInt32` length):

```
[place of each absent component ...][0xFF]   components to remove
[place | 0x80 if delta][bytes] ...           components present, Others components last
```

The place is the component's index in the sealed table. The `Others` components come last so the owner's side stops
reading there.

Following side to server, over the same connection: the `Knock` first, then whatever the program sends with
`Service::Send`, unframed, handed to the server's `Listener::OnMessage`.

## 7. How the subscriber applies frames

- **Enter.** A runtime replica it does not hold is spawned from the archetype slot under its parent, with its
  `Replica` set, and gets `Locus(section key)` as a root or `Locus()` as a child. A replica it holds already keeps its
  state unless it moved parent (the publisher sent its changes before re-entering). Its owner is claimed (below).
  Children that arrived before it are replayed.
- **Parent not held.** An `Enter` whose parent this side does not hold (a persistent parent not loaded yet) is held.
  When it is this side's own body, the listener's `OnAwait(Parent)` asks the program to load it.
- **Update.** Applied to the replica, or to the world for id 0. An update for a persistent replica not loaded yet, or
  for one whose arrival is held, is held too.
- **Forget.** Destroys the replica, unless the new key is one this side watches (it arrives there next) or it is the
  side's own body.
- **Leave.** Destroys every runtime root under the key and drops what was held for it. The side's own body survives.
- **Claim.** On every arrival: `Owner(Peer)` is set or removed. The owner's own copy gets `Possessed` (and loses
  `Remote`). Every other copy gets `Remote` and loses its `Owner`-audience components, so no client keeps another
  player's private data.
- **Others.** The owner's copy reads past `Others` components in updates and in the arrival that hands it the
  body, since it predicts them itself.
- **Persistent replicas** loaded from disk get `Remote` (unless tagged `Client`) as the subscriber takes them in.
- **Hello** starts the session over: runtime replicas go, held messages go, the peer key is learned.

A follower's systems that decide (move bodies, apply rules) filter `.Not<ZyReplication::Remote>()`, so they run only
on what this side owns; the server's values for everything else arrive in frames.

## 8. Owner-only data

`Audience::Owner` components travel in each peer's private section. They go:

- whole on every arrival of a replica the peer owns,
- whole when the peer starts watching a key holding replicas it owns,
- as updates (or deltas), only to the owner and only while the owner watches the replica's key.

`Send(Actor, Audience::Owner, ...)` rides the same private section. State that must stay hidden (a player's
inventory, cooldowns, quest log) belongs here. Everything other players may see goes through `Everyone` components or
messages. `Audience::Others` is for values the owner predicts itself (its own position), so the owner skips them.

## 9. Links: references between replicas

An `Entity` is meaningless on another side. A replicated component that names another replica holds a `Link`:

```cpp
struct Leash final
{
    /// The wolf's master, which every peer resolves on its own side.
    ZyReplication::Link Master;

    /// \brief Serializes the state of the object to or from the specified archive.
    ///
    /// \param Archive The archive to serialize the object with.
    template<typename Serializer>
    ZY_INLINE void Serialize(Serializer Archive)
    {
        Archive.Serialize(Master);
    }
};

// Server: point it at an entity.
Wolf.Set(Leash { ZyReplication::Link(Player) });

// Any side: find what it names now, or nothing while the target is not held here.
const ZyScene::Entity Master = Replication.Resolve(Wolf.Get<Leash>()->Master);
```

A link caches the entity it found and asks again once that entity is gone. It writes the live target's identifier,
or remembers a persistent one after the target unloads. A runtime identifier whose target died writes as zero.
Identifiers are never reused, so a link never names the wrong replica.

## 10. Safety rules for reading peer bytes

Replication reads bytes from the network on both sides. The engine's rule is that a corrupted or malicious peer can
cause no crash, out-of-bounds read, endless loop or unbounded allocation.

### 10.1 What the subscriber refuses

A refused publisher has nothing more applied until a new `Hello` starts a session over. Each refusal logs a line
starting `Replication:`.

| Input | Result |
|---|---|
| A message that is not a frame | Refused (`A message arrived outside a frame`). |
| A frame shorter than its clock, or a clock that is NaN, infinite or has a negative multiplier | Refused. |
| A packed frame naming size 0 or over 1 MiB (`kMaxUnpacked`), that does not unpack whole, or that unpacks to another packed frame | Refused. |
| An unknown opcode, or anything before `Hello` | Refused. |
| A `Hello` whose manifest hash differs | Refused (`The publisher replicates a different set of components than this build`). |
| An `Enter` with id 0, outside a key section, for a persistent id, or attaching a replica beneath itself | Refused. |
| A `Forget` of a persistent id | Refused. |
| A `Message` longer than what is left of the frame | Refused. |
| A set whose length runs past the frame, an index past the table, or a delta for a tag or a missing component | Refused. Each set is read through its own sub-reader, so a bad entry cannot reach the messages after it. |
| Held messages (updates and arrivals waiting for parents) over 4 MiB (`kMaxHeld`) | Refused. Held arrivals replay from a worklist, never by recursion. |
| An archetype slot this world does not hold | Logged, and the replica is spawned without an archetype. |

### 10.2 What the serving side refuses

| Input | Result |
|---|---|
| A waiting peer's first message that is not a `Knock`, or a ticket over 1024 bytes | Told `Ticket`, closed. |
| Any later message from a peer still waiting | Ignored. |
| A message from a peer not admitted | Never reaches the listener. |
| A TCP frame over 1 MiB, or a frame without a tag | Connection ends with `Protocol`. |
| A peer that stops reading | Connection ends with `Congested` past 1 MiB of backlog. |

Above that, the program's own message handling decides. A good server expels a peer whose message is too short,
too long, names something it may not touch, or is not read whole.

### 10.3 Rules for your own replicated components and messages

These keep your code inside the engine's guarantee:

- **Bound every count you read.** A component that reads a list from the wire caps it, as `Ledger` does (at most
  4096 slots, and never more than the bytes left could hold). An unbounded `Sequence` read from a peer is an
  allocation a hostile peer controls.
- **Expect short input.** `Reader::Read<T>()` past the end returns a zero value and moves the cursor to the end. It
  never reads out of bounds, but a zero may be a valid value. Check `GetAvailable()` before reading fixed fields when
  that matters. `ReadText` returns an empty text when the bytes do not hold it whole.
- **Validate ids.** An identifier from the wire may name nothing (`Find` returns an entity that is not alive), or a
  table entry you do not have. Check before indexing anything with it.
- **Validate enums and ranges.** A `UInt8` read as an enum may be past its last value.
- **Never trust a client's claim about the world.** It may name any replica. Range, ownership and cost are checked on
  the server, where the world is.
- **Read the message whole.** Treat leftover bytes as a malformed message.

## 11. Worked example: a replicated arena

One program, two roles. `--serve` runs a headless server; with no switch it joins `127.0.0.1` and opens a window.
The server keeps every peer under one key, hands each joining peer a body, and moves bodies on the asks peers send;
each body's `Health` travels to everyone and its `Purse` only to its owner. A chat line sent by any peer reaches
every peer in order with the world.

```cpp
// Arena/Arena.hpp
#pragma once

#include <Zyphryon.Replication/Service.hpp>
#include <Zyphryon.Scene/Service.hpp>

namespace Arena
{
    /// \brief Represents where a body is, in world space.
    struct Position final
    {
        /// The point, in metres.
        Vector3 Value;

        /// \brief Serializes the state of the object to or from the specified archive.
        ///
        /// \param Archive The archive to serialize the object with.
        template<typename Serializer>
        ZY_INLINE void Serialize(Serializer Archive)
        {
            Archive.Serialize(Value);
        }
    };

    /// \brief Represents what a body can still take, which every peer sees.
    struct Health final
    {
        /// The hit points left, from 0.
        UInt16 Points = 100;

        /// \brief Serializes the state of the object to or from the specified archive.
        ///
        /// \param Archive The archive to serialize the object with.
        template<typename Serializer>
        ZY_INLINE void Serialize(Serializer Archive)
        {
            Archive.Serialize(Points);
        }
    };

    /// \brief Represents the coins a body carries, which only its owner sees.
    struct Purse final
    {
        /// The coins carried.
        UInt32 Coins = 0;

        /// \brief Serializes the state of the object to or from the specified archive.
        ///
        /// \param Archive The archive to serialize the object with.
        template<typename Serializer>
        ZY_INLINE void Serialize(Serializer Archive)
        {
            Archive.Serialize(Coins);
        }
    };

    /// \brief Specifies the asks a peer may send the server, the first byte of each message.
    enum class Ask : UInt8
    {
        Step, ///< Moves the peer's body one metre along a direction.
        Say,  ///< Sends a line of chat to every peer.
    };

    /// \brief Represents both sides of the arena's session.
    class Session final : public ZyReplication::Listener
    {
    public:

        /// \brief The one key everything stands under in this small game.
        static constexpr UInt64 kArena = 1;

        /// \brief Declares what travels, alike on both sides, before the session opens.
        ///
        /// \param Scene       The scene service, which outlives the session.
        /// \param Replication The replication service, which outlives the session.
        Session(Ref<ZyScene::Service> Scene, Ref<ZyReplication::Service> Replication)
            : mScene       { Scene },
              mReplication { Replication }
        {
            Scene.Declare<Position>("Arena::Position");
            Scene.Declare<Health>("Arena::Health");
            Scene.Declare<Purse>("Arena::Purse");

            Replication.Replicate<Position>(ZyReplication::Audience::Everyone);
            Replication.Replicate<Health>(ZyReplication::Audience::Everyone, 4);
            Replication.Replicate<Purse>(ZyReplication::Audience::Owner);

            // Both sides make the archetype first, so it holds the same slot on both.
            mBody = Scene.CreateArchetype(ZyScene::Entity(), ZyScene::Entity());
            mBody.SetName("Body");
            mBody.Set(Health());
        }

        /// \copydoc ZyReplication::Listener::OnJoin
        void OnJoin(UInt64 Peer) override
        {
            if (!mReplication.IsServing())
            {
                return;
            }

            // A root under the arena's key, which the peer is told to watch.
            const ZyScene::Entity Body = mScene.Spawn(mBody, ZyScene::Entity(), Position(), Purse { 10 });
            Body.Set(ZyReplication::Locus(kArena));

            const UInt64 Keys[] = { kArena };
            mReplication.Watch(Peer, ConstSpan<UInt64>(Keys));
            mReplication.Possess(Peer, Body);
            mBodies.Assign(Peer, Body.GetID());
        }

        /// \copydoc ZyReplication::Listener::OnLeave
        void OnLeave(UInt64 Peer, Text Reason) override
        {
            if (const ConstPtr<UInt64> Found = mBodies.Find(Peer))
            {
                if (const ZyScene::Entity Body = mScene.GetEntity(* Found); Body.IsAlive())
                {
                    Body.Destruct();
                }
                mBodies.Erase(Peer);
            }
        }

        /// \copydoc ZyReplication::Listener::OnMessage
        void OnMessage(UInt64 Peer, ConstSpan<Byte> Message) override
        {
            Reader Input(Message);

            if (!mReplication.IsServing())
            {
                // From the server: a chat line it relayed.
                LOG_I("Arena: {0}", Input.ReadText());
                return;
            }

            // From a peer: every ask is checked here, whole, before anything acts on it.
            const ConstPtr<UInt64> Found = mBodies.Find(Peer);
            const ZyScene::Entity  Body  = Found ? mScene.GetEntity(* Found) : ZyScene::Entity();

            if (!Body.IsAlive() || Input.GetAvailable() < 1)
            {
                mReplication.Expel(Peer, "Malformed ask");
                return;
            }

            switch (static_cast<Ask>(Input.Read<UInt8>()))
            {
            case Ask::Step:
            {
                const UInt8 Direction = Input.Read<UInt8>();

                if (Direction > 3 || Input.GetAvailable() != 0)
                {
                    mReplication.Expel(Peer, "Malformed step");
                    return;
                }

                const Vector3 Steps[] = {
                    Vector3( 1.0f, 0.0f,  0.0f), Vector3(-1.0f, 0.0f,  0.0f),
                    Vector3( 0.0f, 0.0f,  1.0f), Vector3( 0.0f, 0.0f, -1.0f) };

                // Modify records the change, so it travels in the next frame.
                Body.Modify<Position>([&](Ref<Position> Position)
                {
                    Position.Value += Steps[Direction];
                });
                break;
            }
            case Ask::Say:
            {
                const Text Line = Input.ReadText();

                if (Line.IsEmpty() || Line.GetSize() > 255 || Input.GetAvailable() != 0)
                {
                    mReplication.Expel(Peer, "Malformed line");
                    return;
                }

                Writer Output;
                Output.WriteText(Line);

                // Through the world entity, every peer hears it, in order with the world's changes.
                mReplication.Send(mScene.GetWorld(), ZyReplication::Audience::Everyone,
                    ConstSpan<Byte>(Output.GetData(), Output.GetSize()));
                break;
            }
            default:
                mReplication.Expel(Peer, "Unknown ask");
                break;
            }
        }

        /// \brief Asks the server to move this side's body, on a following side.
        ///
        /// \param Direction The direction, from 0 to 3.
        void Step(UInt8 Direction)
        {
            Writer Output;
            Output.Write<UInt8>(static_cast<UInt8>(Ask::Step));
            Output.Write<UInt8>(Direction);
            mReplication.Send(ConstSpan<Byte>(Output.GetData(), Output.GetSize()));
        }

    private:

        Ref<ZyScene::Service>       mScene;
        Ref<ZyReplication::Service> mReplication;
        ZyScene::Entity             mBody;
        Table<UInt64, UInt64>       mBodies;
    };
}
```

```cpp
// Arena/Main.cpp
#include "Arena/Arena.hpp"
#include <Zyphryon.Runtime/Entry.hpp>

namespace Arena
{
    /// \brief Represents the program, server or client by its command line.
    class Application final : public ZyRuntime::Kernel
    {
    protected:

        /// \copydoc ZyRuntime::Kernel::OnConfigure
        void OnConfigure(Ref<ZyRuntime::Startup> Startup) override
        {
            Startup.SetHeadless(GetEnvironment().Contains("serve"));
            Startup.SetRate(30);
        }

        /// \copydoc ZyRuntime::Kernel::OnInitialize
        Bool OnInitialize() override
        {
            Ref<ZyScene::Service>       Scene       = * GetService<ZyScene::Service>();
            Ref<ZyReplication::Service> Replication = * GetService<ZyReplication::Service>();

            mSession = Retainer<Session>::Create(Scene, Replication);

            if (GetEnvironment().Contains("serve"))
            {
                return Replication.Serve({ .Port = 7777 }, ZyNetwork::Transport::TCP, mSession);
            }

            const ZyNetwork::Address Server { .Host = GetEnvironment().GetText("join", "127.0.0.1"), .Port = 7777 };
            return Replication.Join(Server, ZyNetwork::Transport::TCP, mSession, ConstSpan<Byte>());
        }

    private:

        Retainer<Session> mSession;
    };
}

ZY_APPLICATION(Arena::Application)
```

What happens:

1. Both processes declare the same three components, replicate them with the same audiences, and make the `Body`
   archetype first, so its slot matches. The manifest hashes agree.
2. The client knocks with an empty ticket. The default `OnKnock` lets it in; the server's `OnJoin` spawns a body
   under key 1, has the peer watch key 1, and possesses the body for it.
3. On the next server tick the publisher reads the `Locus` set, gives the body an identifier, and sends the client a
   frame: `Hello`, then the arena's catch-up, `Enter` for every body there. Every other peer watching key 1 gets an
   `Enter` for the new body.
4. The client spawns each body from the `Body` archetype. Its own gets `Possessed` and its `Purse`; every other gets
   `Remote` and no `Purse`.
5. `Session::Step` sends an ask. The server checks it whole and moves the body with `Modify`, which records a change,
   so the next frame carries an `Update` of `Position` to every peer. `Health` (rate 4) is read every fourth tick.
6. A peer that leaves loses its body, which `Forget`s on the wire.

The client would draw `Position` each frame with a system reading `Set<Position>` (see `scene.md` and
`graphics.md`).

## 12. Common mistakes and limits

- **Different tables on the two sides.** Any `Replicate` call missing on one side, or a component whose size differs
  between builds, fails the greeting's hash and refuses the server.
- **Archetypes made in a different order on the two sides.** An `Enter` names the archetype by slot, so the client
  builds the wrong thing. Make or load archetypes identically on both sides before the session opens.
- **Registering after the session opened.** The table is sealed in `Serve`/`Join`; later calls assert.
- **Expecting a write to travel without a notify.** See section 5.3.
- **Treating a persistent replica like a runtime one.** It never moves key, never has an owner, and is never
  forgotten over the wire. To make something both loaded from disk and movable, despawn it and spawn a runtime
  replica.
- **Holding `Entity` in a replicated component.** Use `Link`.
- **Giving a root key 0.** `Locus()` means "child". A root under no key is never published.
- **Large messages.** A replication message over 32 KiB is dropped and logged
  (`Replication: A message of {0} bytes is larger than a frame takes`). A network payload over 1 MiB is dropped too.
- **UDP for reliable traffic.** Not carried yet. Use TCP.
- **Deltas without marks.** A delta-capable component that writes in place without marking its ledger sends an empty
  delta. Change slots through `Set`/`Edit`.
- **Trusting a peer's message.** Check length, range and ownership of everything a peer sends, and expel on a
  malformed message.
- A session is one server and its followers. There is no host migration, no peer-to-peer, and no relay.

## 13. Where it lives

| Path | What |
|---|---|
| `Engine/Source/Zyphryon.Network/Service.hpp`, `Service.cpp` | Endpoints, send, close, timeouts. |
| `Engine/Source/Zyphryon.Network/Common.hpp` | Limits, `Delivery`, `Transport`, `Reason`, `Address`, `Statistics`. |
| `Engine/Source/Zyphryon.Network/Handler.hpp`, `Connection.hpp` | The callbacks and the handle. |
| `Engine/Source/Zyphryon.Network/Executor.hpp`, `Executor.cpp`, `Mailbox.hpp` | The network thread and the queue back to the program. |
| `Engine/Source/Zyphryon.Network/Protocol/TCP/` | `Stream`, `Pipe`, `Server`: framing, ping, timeout, congestion. |
| `Engine/Source/Zyphryon.Network/Protocol/UDP/` | `Session`, `Sentry`, `Server`, `Client`. |
| `Engine/Source/Zyphryon.Network/Signal.hpp` | Newest-wins value over unreliable messages. |
| `Engine/Source/Zyphryon.Network/Driver/` | Winsock, Posix, Emscripten sockets. |
| `Engine/Source/Zyphryon.Replication/Service.hpp`, `Service.cpp` | The service and its connection gate (knock, ticket, expel). |
| `Engine/Source/Zyphryon.Replication/Manifest.hpp`, `Manifest.cpp` | The table, its hash, set writing and reading. |
| `Engine/Source/Zyphryon.Replication/Publisher.hpp`, `Publisher.cpp` | Change tracking, channels, catch-ups, frames. |
| `Engine/Source/Zyphryon.Replication/Subscriber.hpp`, `Subscriber.cpp` | Applying frames, holds, refusals. |
| `Engine/Source/Zyphryon.Replication/Wire.hpp` | Opcodes, `kGlobal`, `kPrivate`. |
| `Engine/Source/Zyphryon.Replication/Common.hpp` | `kPersistent`, `kTicket`, `Audience`, the tags. |
| `Engine/Source/Zyphryon.Replication/Ledger.hpp`, `Link.hpp`, `Locus.hpp`, `Owner.hpp`, `Replica.hpp`, `Listener.hpp` | The value types. |
| `Engine/Source/Zyphryon.Runtime/Kernel.cpp` | Where both services are created. |
