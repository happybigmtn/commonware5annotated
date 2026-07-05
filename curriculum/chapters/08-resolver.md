# Chapter 08 — Resolver: fetch data identified by a key

> How a consensus node fetches the data behind a hash it has heard about.

## The problem

You're a validator in consensus. View 5 gets notarized: `2f+1` nodes voted for
block hash `H`. You missed the actual block contents (you were offline, or
you came in late, or you just couldn't keep up with the leader's broadcast).

Now what? You need the data behind `H` to participate in view 6 (which
might depend on it), to verify it's a valid block, and to vote on its
finalization.

The resolver is the primitive for **"fetch the data behind a key."** Given a
key (typically a hash), it asks peers for the data and delivers it to your
callback when received.

## The trait surface

Open `resolver/src/lib.rs:115-212`. Three core traits:

```rust
pub trait Consumer: Clone + Send + 'static {
    type Key: Span;
    type Value;
    type Subscriber: Clone + Eq + Send + 'static;

    /// Deliver data to the consumer. Returns a Receiver that resolves to true
    /// if the data is valid for the key.
    fn deliver(
        &mut self,
        delivery: Delivery<Self::Key, Self::Subscriber>,
        value: Self::Value,
    ) -> oneshot::Receiver<bool>;
}

pub trait Resolver: Clone + Send + 'static {
    type Key: Span;
    type Subscriber: Clone + Eq + Send + 'static;

    fn fetch<F>(&mut self, key: F) -> Feedback
    where F: Into<Fetch<Self::Key, Self::Subscriber>> + Send;

    fn fetch_all<F>(&mut self, keys: Vec<F>) -> Feedback
    where F: Into<Fetch<Self::Key, Self::Subscriber>> + Send;

    fn retain(
        &mut self,
        predicate: impl Fn(&Self::Key, &Self::Subscriber) -> bool + Send + 'static,
    ) -> Feedback;
}

pub trait TargetedResolver: Resolver {
    type PublicKey: PublicKey;
    fn fetch_targeted(
        &mut self,
        fetch: impl Into<Fetch<Self::Key, Self::Subscriber>> + Send,
        targets: NonEmptyVec<Self::PublicKey>,
    ) -> Feedback;
}
```

The pattern:

1. **You** call `resolver.fetch(key)` with some data you need.
2. **Resolver** asks peers for the data matching that key.
3. **Peers** respond with the data.
4. **Resolver** calls `consumer.deliver(delivery, value)` to hand the data
   to you.
5. **You** verify the data is valid (via the returned `oneshot::Receiver<bool>`).
6. If valid, use the data. If invalid, ignore.

## Subscribers — multiple consumers, same key

`Fetch { key, subscriber, span }` (`resolver/src/lib.rs:21-29`). The
**subscriber** field lets multiple in-flight requests for the same key be
deduplicated.

Say you're fetching block `H` because:
- You missed the broadcast and need to certify it (subscriber A).
- You're also syncing historical state and need it (subscriber B).

Both register `fetch(H)` with different subscribers. The resolver
internally merges them — only one network request goes out. When the
response arrives, both subscribers receive the data.

This is the pattern for avoiding redundant network traffic when multiple
parts of your app want the same data.

## Target hints — "ask this peer first"

`TargetedResolver::fetch_targeted` lets you say "I think peer X has this
data — ask them first." Useful when:

- You just saw a notarization from X. X probably still has the data.
- You know X is well-connected to the data's origin.

The resolver tries the targets first, falls back to broadcasting if they
don't respond.

## Retain — cleaning up subscriptions

`Resolver::retain(predicate)` runs a predicate over all current
subscribers. Subscribers that fail the predicate are canceled (their
`oneshot::Receiver<bool>` may never resolve).

This is how you tell the resolver "I'm done caring about block H now" — the
subscriber for that fetch is dropped, network bandwidth is freed, the
response (if it arrives later) is discarded.

## The relationship to Simplex

In chapter 01's Simplex config, the `Resolver` is a parameter. It's used
by the consensus engine to fetch missing **certificates** (notarizations,
nullifications, finalizations from prior views). Without it, a node that
joins late would be stuck.

The standard construction (provided by `resolver/src/p2p/`) uses the
authenticated P2P layer from chapter 05: it sends a `Get { key }` message
to peers, expects a `Data { value }` message back. The Consumer is your
Simplex engine, which validates the certificate against its known set of
validators and schemes.

## Where to look in the code

- `resolver/src/lib.rs:21-29` — the `Fetch` struct.
- `resolver/src/lib.rs:67-89` — the `Delivery` struct.
- `resolver/src/lib.rs:115-144` — the `Consumer` trait.
- `resolver/src/lib.rs:147-186` — the `Resolver` trait.
- `resolver/src/lib.rs:189-212` — the `TargetedResolver` extension.
- `resolver/src/p2p/` — the P2P-backed construction.

## Content-addressable storage theory

The resolver assumes a foundational property: **two peers that ask for "the
data behind key K" both want the same data**. The key identifies content by
its hash, not by its location or by a name someone picked.

This is **content-addressable storage (CAS)**. The classical CS description
is:

> Data is identified by a deterministic function of its own contents. Two
> peers, given the same content, will compute the same identifier with
> overwhelming probability.

Three properties follow:

1. **Self-certification.** If I tell you "the data behind hash `H` is `D`",
   you can verify by hashing `D` and comparing to `H`. If they match, `D`
   is *the* data behind `H` (modulo collision resistance). No central
   registrar needed.
2. **Deduplication.** Two peers that fetch the same block store the same
   bytes; they can verify by hash without re-fetching from a source of
   truth.
3. **Collision resistance matters.** If a malicious peer could produce
   `D != D'` with `hash(D) == hash(D')`, they could swap content unnoticed.
   Commonware uses SHA-256 (chapter 04), which has 128-bit collision
   resistance — computationally infeasible to break.

CAS shows up everywhere in practice:

| System | How CAS is used |
|---|---|
| Git | Commit IDs; objects stored by SHA-1 of contents |
| IPFS | Files broken into chunks, each addressed by hash |
| BitTorrent | Pieces of a torrent addressed by SHA-1 (info-hash + piece hashes) |
| Commonware | Resolver keys are 32-byte digests (SHA-256 or scheme digests) |

The resolver is the network layer that *finds* content for a CAS
identifier. The hash guarantees correctness; the resolver guarantees
availability.

There's a sharp consequence: **the resolver does not need a name
authority**. There's no "Bob's-block.txt" — just "block-whose-SHA256-is-X."
This makes the resolver trivially open. Anyone who knows the hash can
verify anyone else's reply. The only trust assumption is the collision
resistance of the hash function.

A subtler consequence: the resolver is **idempotent and merge-able**.
Two requests for the same key produce the same bytes. The receiver
can't tell whether the sender fetched it themselves or got it from a
peer. This is good for gossip (chapter 09): a message you forwarded
yesterday and a message a peer forwarded today are the *same* message
by definition.

## Distributed Hash Tables — how peers find data

CAS is "what is the data." The next question is "where is it?" — and in a
network with no fixed topology, that's the job of a **distributed hash
table (DHT)**.

The classic DHT algorithms (Stoica et al., Rowstron & Druschel, etc.):

- **Chord** (2001) — peers form a ring over `[0, 2^160)`. Each peer owns
  a range of keys. Routing table is `O(log N)`; lookup takes `O(log N)`
  hops.
- **Pastry** (2001) — peers form a Plaxton-style tree (prefix routing).
  Used by Past / FreePast.
- **Kademlia** (2002) — peers pick random `k`-bit IDs; routing is via
  XOR distance. Used by libp2p, Mainline DHT (BitTorrent), IPFS.

Commonware's resolver is **not** a DHT. It assumes the set of "peers
that might have the data" is known — typically the active validator
set. This is a deliberate design choice:

- DHT adds latency (`log N` hops per lookup) and complexity (bootstrapping,
  churn handling, attack surface from arbitrary peers).
- Consensus nodes already have a small, authenticated peer set
  (chapter 05). Asking all of them in parallel works fine for `N ≤ ~100`.

But the resolver inherits two **ideas** from DHTs:

1. **Routing by key, not name.** Just as DHTs forward `(key, query)`
   toward the peer that "owns" the key, the resolver forwards
   `(key, Get)` toward a peer likely to have it.
2. **Parallel queries with first-wins.** In Kademlia, a lookup asks
   `k` peers in parallel; the first responder short-circuits the rest.
   `resolver/src/p2p/fetcher.rs` does the same: broadcasts the Get,
   takes whichever Data reply arrives first.

When the peer set is small (~100), a broadcast is cheaper than
`log N` DHT hops. When the peer set is large or unknown (think
"anyone on the internet"), a DHT becomes a better choice.

## BitTorrent's piece-selection legacy

If you came of age in the 2000s, you used BitTorrent every day. It solved
a similar problem — pushing a big file to many peers — and three of its
ideas live on in modern broadcast engines:

**Piece-level addressing.** A torrent's `info_hash` identifies the
metadata; each piece has its own SHA-1 hash in the metainfo file. BitTorrent
clients verify each piece independently. Commonware's resolver encodes
this faithfully: every value has its own digest (chapter 03 `Digestible`),
and verification is per-value, not per-batch.

**Rarest-first.** When choosing which piece to download, a BitTorrent
client picks the piece held by the fewest peers. This reduces the chance
of "all peers needed for piece X already finished their upload slots."
`broadcast::buffered::Engine` doesn't itself implement rarest-first
selection (data is pushed by the sender, not pulled by the receiver),
but the same logic informs reference-counted cache eviction: when the
cache fills, you want to keep messages that are hard to re-fetch.

**Endgame mode.** When a download is 99% done, you request the last few
pieces from *every* peer that has them. The first response "wins";
later responses are discarded. This kills the tail. The resolver does
something equivalent: it accepts the first valid response and cancels
the in-flight peers (Appendix I). The encoding trick is to ignore
redundant responses once the value has been verified.

The pattern is the same: **fan out, take the first valid result, ignore
the rest**. This keeps the worst-case latency close to the slowest peer,
not the slowest *required* peer.

## IPFS bitswap — the wantlist protocol

IPFS builds its data exchange on a protocol called **bitswap**. The
vocabulary transfers directly to Commonware:

| IPFS bitswap term | Commonware equivalent |
|---|---|
| `wantlist` | Resolver's in-progress fetches (`HashMap<K, PendingFetch<S>>` in `resolver/src/p2p/inflight.rs`) |
| `have` message | "Yes, I have it" (resolver's response) |
| `dont_have` message | "I don't have it" (resolver skips this peer) |
| `cancel` peer message | `resolver.retain(...)` |
| `block` transfer | `Data { key, value }` (in `resolver/src/p2p/wire.rs`) |

The **lifecycle of a fetch** in bitswap form:

```
1. Want(K)             — "I want the block for K"
2. Peers respond with have(K) or dont_have(K)
3. If have(K): request from any peer; first valid wins
4. Block arrives: verify hash, deliver
5. Cancel(K) when no longer needed
```

This is exactly the resolver's flow:
1. `resolver.fetch(K)` — add to in-progress set, send `Get` to peers
   (`resolver/src/p2p/engine.rs`).
2. Peers respond with `Data { key, value }` or no response.
3. First valid response delivers to all subscribers.
4. `resolver.retain(predicate)` cancels.

Bitswap adds one idea Commonware doesn't currently use: the **peer's
inventory** (their `have` set) is gossiped lazily. Commonware assumes
peers are either serving or not, with retries until satisfied. For a
consensus peer set this is fine.

## CDN patterns — caches, edges, hierarchies

The resolver's cache layer is a tiny CDN (content-delivery network).
A real CDN has three dimensions Commonware's resolver doesn't:

1. **Geographic hierarchy.** A CDN puts caches close to users
   (edge → regional → origin). The resolver is "all caches close
   together" — consensus nodes are usually in a few data centers.
2. **Pull vs push.** CDNs typically *pull* content on user demand.
   The resolver *pushes* (via the broadcast layer, chapter 09) and
   fetches on demand (the resolver itself). This is a hybrid.
3. **Cache invalidation.** CDNs invalidate by TTL or by explicit
   purge. The resolver invalidates by timestamp: messages are
   evicted from per-peer deques when fresh messages arrive
   (`buffered::Engine::deques`, `broadcast/src/buffered/engine.rs:90`).

The simplest CDN topology is **pull-cache-on-miss**:

```
Client → Edge (cache? yes → return; no → fetch from origin → cache → return)
```

The resolver, told to `fetch(K)`, does the same:
- Check your local cache (`pending[C]`). If you already have it,
  deliver. If not, send `Get(K)` to peers. First valid response
  is delivered.

A more sophisticated CDN supports **push** (the origin writes
new content to edges proactively). The broadcast layer is exactly
this: the leader pushes a new block to all peers before anyone
asks for it. By the time the resolver is invoked, the peers may
already have it cached.

In Commonware's design, **resolver + broadcast together play the
role of the CDN**: the broadcast layer is the proactive push
("origin writes to edges"), and the resolver is the on-miss pull
("edge fetches from a peer"). Each block has a digest key, not a
location, so the two layers interleave cleanly.

## Cache eviction policies

`broadcast::buffered::Engine` uses a simple per-peer ring buffer
(FIFO) for incoming messages. The resolver maintains a separate
cache (typically LRU). Both choices deserve unpacking.

The three policies you meet most often in real systems:

**LRU (Least Recently Used).** Evict the entry that hasn't been
read in the longest time. Implemented with a doubly-linked list
or a hash-table + linked-list combo. `O(1)` access and eviction.
Strong in workloads with temporal locality — if you needed K
recently, you'll probably need it again.

**LFU (Least Frequently Used).** Evict the entry that's been read
the fewest times. Better than LRU when access patterns have *long*
temporal locality — think reference data accessed for years. Needs
extra bookkeeping (frequency counts); `O(1)` is achievable with
cleverness but most implementations are slower.

**ARC (Adaptive Replacement Cache).** IBM's 2003 algorithm —
maintains two LRU lists (one for "recently seen once" and one for
"seen multiple times") and dynamically balances them based on
hit ratio. Resists cache-thrashing better than LRU; rarely worth
the complexity.

For Commonware's consensus use case, **LRU is fine** because:

- Block requests are driven by consensus progress: you fetch B1,
  then B2, then B3. The next block is the next one, not a random
  one from history.
- The cache size is bounded by `cache_size × num_peers`, which is
  small enough that policy differences don't matter much.
- Predictability beats cleverness for consensus — you don't want
  the cache to evict a block you need just because someone else
  has been hammering a different block.

The resolver's cache (in `resolver/src/p2p/engine.rs`) is a
simple `HashMap<K, Bytes>` plus subscriber tracking. The broadcast
engine's cache uses a `BTreeMap<M::Digest, M>` plus the
`deques` ring buffers per peer.

When memory matters more than hit rate, the manager can shrink
the cache and accept cache misses (the resolver re-fetches).
When latency matters more, expanding the cache eliminates most
fetches at the cost of memory.

A neat trick specific to consensus: the broadcast layer uses
**reference counting across peers** (`broadcast/src/buffered/engine.rs:96-99`).
A message from peer X and peer Y is stored once but referenced
twice. Only when *all* references are gone is it actually freed.
This is the same idea as a small-reference LRU with shared
objects, but it falls out of the design naturally.

## Lifecycle of a fetch — from a distributed-systems perspective

From a distributed-systems perspective, a fetch has five
phases. Each phase maps cleanly to source code.

**Phase 1 — Request issuance.** Your node decides it needs the
data behind key K. You construct `Fetch { key, subscriber, span }`
(`resolver/src/lib.rs:22-29`). The `subscriber` is your identity
("I want this data because of reason X"); the `span` carries
tracing context across the actor boundary.

```rust
let fetch = Fetch {
    key: K::encode(&cert_proposal_digest),  // 32-byte hash
    subscriber: MyConsumer::Me,             // typed identifier
    span: tracing::info_span!("resolver.fetch"),
};
resolver.fetch(fetch).await?;
```

**Phase 2 — Routing.** The resolver looks up `key` in its
in-progress map (`resolver/src/p2p/inflight.rs`).

```
key → not present:  send Get(K) to peers in latest.primary,
                     register in-flight entry with our subscriber.
key → already present: register our subscriber with that entry
                        (no new network traffic).
key → blocked:        a peer lied earlier; try others.
```

**Phase 3 — Peer query.** A `Get(K)` reaches peers. Each peer
receives it on the "request" channel (registered in
`resolver/src/p2p/config.rs`). The peer's responder looks up
its local cache:

```rust
impl Handler for Responder {
    fn process(&mut self, origin: PublicKey, request: Request) {
        match request {
            Request::Get { key } => match self.cache.get(&key) {
                Some(value) => { let _ = self.responder.send(...); }
                None => { /* skip silently or queue */ }
            }
        }
    }
}
```

If the peer has K, they reply with `Data(key, value)`. If not,
the message expires without a reply (the resolver times out and
tries another peer).

**Phase 4 — Verification.** When the requester receives
`Data(key, value)`, it:

1. Checks `Data::key == request::key` (defense against confused-peer
   attacks — a peer can't substitute a different value).
2. Verifies `Data::value` itself (e.g., block hash matches digest;
   certificate signatures check out) — `Consumer::deliver` is
   what does this.
3. If valid, deliver. If invalid, mark the peer as blocked and
   retry.

**Phase 5 — Delivery and cleanup.** Successful delivery resolves
all subscribers' `oneshot::Receiver<bool>` channels. The inflight
map drops the entry. If `deliver` returned `false` (we accepted
the value but no subscribers want it), we still drop the entry.

```rust
let delivery = Delivery { key, subscribers: NonEmptyVec::of((Me, span)) };
let accepted = consumer.deliver(delivery, value).await?;
// accepted == false is normal — application rejected (e.g., shutdown)
```

Lifecycle timing, end-to-end:

```
T+0     : resolver.fetch(K) issued
T+0     : inflight[K] = {subscribers: [Me], peers_tried: []}
T+0.001 : Get(K) sent to 5 peers (broadcast)
T+0.020 : peer X replies with Data(K, V)
T+0.020 : verify hash(V) == K
T+0.020 : deliver to subscribers
T+0.020 : inflight.remove(K)
T+0.020 : Me's Receiver<bool> resolves
```

If no peer responds by `T+0.5`, retry (in Commonware, the resolver
keeps trying until `cancel` or quorum of bad responses).

## Rust patterns in the resolver

Three Rust patterns you'll see repeatedly in this crate:

**`oneshot::channel` for response delivery.** When a peer responds
to your Get, the resolver hands the value to your `Consumer` and
gets a verdict back:

```rust
let (tx, rx) = oneshot::channel();
self.pending.insert(*key, tx);
// ... later, on response ...
let _ = tx.send(accepted);  // accepts bool
// ... on consumer side ...
let accepted = rx.await?;
```

Why oneshot? Because the verdict is binary ("this fetch is good /
not good") and fires exactly once. If the consumer's `verify` future
is dropped (consumer cancelled), `tx.send` returns `Err(value)` —
the resolver silently discards. No retries, no bookkeeping.

**`BTreeMap` vs `HashMap` — the determinism / speed trade-off.**
The resolver uses `HashMap` internally because it cares about
throughput, not reproducibility. The broadcast engine uses
`BTreeMap` (`broadcast/src/buffered/engine.rs:133`) for *iteration
order* — when a test loops over the cache and hashes the entries,
a non-deterministic order would give different hashes on different
runs. The deterministic runtime (chapter 02) needs `BTreeMap`
semantics for replay tests to match bit-for-bit.

Rule of thumb in Commonware source:

- Internal indexing at runtime hot path → `HashMap`.
- Persistent state, test fixtures, anything that gets serialized →
  `BTreeMap`.
- Per-peer ordered queues (deques) → `VecDeque` or `BTreeMap` of `VecDeque`s.

**`tokio::select!` cancellation safety (or: `select_loop!`).**
Commonware's pattern for actor mailboxes is the `select_loop!` macro
(`commonware-macros::select_loop`) in chapter 15. The idea: an actor
parses a stream of incoming events concurrently, and any branch
finishing first wins. Cancellation safety is crucial: dropping a
future mid-execution must not corrupt state.

The resolver uses `select_loop!` to:
- Receive from `mailbox_receiver` for new fetches / cancels / serves.
- Receive from `inflight` timers (timeouts).
- Stop on shutdown signal.

If a future is cancelled (e.g., the resolver is shutting down and
the in-flight peer query needs to stop), the state it mutated so
far must be safe to leave as-is. Two specific places this matters
in the resolver:

1. `subscribe()` — registering a subscriber with an in-flight
   fetch. The future can be cancelled mid-registration. We use a
   lock so the registered state is either fully written or not
   written.
2. `serve()` — looking up a cache entry. Read-only; trivially
   cancellation-safe.

## Content-addressable storage: history and deep theory

The CAS section above gave the core idea. Let me dig deeper into the
history, the security model, and the connections to Merkle DAGs.

### A brief history of CAS

CAS predates Commonware by decades. The pattern appeared in:

1. **Plan 9 / Venti (Bell Labs, 2002).** Venti was a network storage
   system where every block was identified by its SHA-1 hash. New
   writes only succeeded if no existing block had the same hash (an
   early deduplication mechanism). Venti's `venti.conf` configuration
   file format, the `arenas` data structure, and the `isect` index
   all derive from the CAS abstraction.

   Venti's data model was a flat namespace of blocks, indexed by hash.
   The OS file system layer (Fossil) layered a directory tree on top.
   Venti showed that an entire operating system could be stored in a
   CAS without losing performance — at least for read-mostly
   workloads.

2. **Git (Linus Torvalds, 2005).** Git objects (commits, trees, blobs)
   are identified by their SHA-1 hash. The entire repository is a
   Merkle DAG keyed by content. If you have the SHA-1 of a commit,
   you can verify the entire history underneath (assuming you trust
   the root).

   Git's data model is more structured than Venti's: commits, trees,
   and blobs have different types and are linked in a DAG. But the
   addressing is pure CAS.

3. **IPFS (Juan Benet, 2014).** IPFS generalizes Git's model to
   arbitrary content. Files are broken into chunks (typically 256 KB),
   each chunk addressed by its multihash (a hash with a self-describing
   algorithm prefix). Files are Merkle DAGs of chunks; directories
   are Merkle DAGs of file references.

   IPFS's addressing is `CID` (Content Identifier), a self-describing
   hash that includes the algorithm and length. This makes IPFS
   CIDs forward-compatible: a CID can say "I'm a SHA-256 of 32 bytes"
   without needing out-of-band knowledge.

4. **BitTorrent (Bram Cohen, 2001).** BitTorrent's torrent files
   include SHA-1 hashes of each piece. The `info_hash` (which
   identifies the torrent itself) is also a hash of structured data.
   So BitTorrent has CAS-like properties for pieces and torrent
   metadata, though the swarm model is different from a global CAS.

5. **Commonware (2024+).** Commonware's resolver uses CAS keys as
   the request identifier. The 32-byte digests come from SHA-256 or
   from scheme-defined digests (e.g., ZODA commitments).

### The Plan 9 / Venti lineage in detail

Venti's design (from Quinlan's 2002 paper) is worth understanding:

```
                    +-------------------+
                    |       Venti       |
                    |  (CAS storage)    |
                    +---------+---------+
                              |
                  +-----------+-----------+
                  |           |           |
              +---v---+   +---v---+   +---v---+
              | arena |   | arena |   | arena |
              |  (raw |   |  (raw |   |  (raw |
              |  data |   |  data |   |  data |
              +-------+   +-------+   +-------+
```

Each arena is a fixed-size file on disk. Blocks are written
sequentially; once written, never modified. The arena has a header
pointing to a "log" of block additions and a pointer to the current
write position.

The addressing: SHA-1 hash of the block contents. To read a block,
compute its disk location from the hash (via a hash-to-arena mapping)
and read the corresponding sector.

Venti's deduplication is automatic: if you try to write a block whose
hash matches an existing block, the write is a no-op. The block is
already there.

The connection to BFT: Venti's model is **immutable, append-only,
content-addressed**. BFT block storage has the same properties.
Commonware's `storage::journal` (chapter 06) is conceptually similar:
append-only logs of immutable items.

### Git's CAS model in detail

Git's object types:

- **Blob**: the contents of a file.
- **Tree**: a list of (mode, name, blob-or-tree-hash) entries.
- **Commit**: a snapshot of the project at a point in time, with
  metadata (author, committer, message, parent commit hashes).

Each object is stored in `.git/objects/xx/yyyyyy...` where `xx` is the
first 2 hex chars of the SHA-1 hash and `yyyyyy...` is the rest.
The object itself is `<type> <size>\0<contents>` (zlib-compressed).

Example: the commit object for a simple change might be:

```
tree 4b825dc642cb6eb9a060e54bf8d69288fbee4904
parent 8c2a...  (previous commit)
author Alice <alice@example.com> 1700000000 +0000
committer Alice <alice@example.com> 1700000000 +0000

Initial commit
```

The SHA-1 of this string is the commit hash. If you change any byte
(author, timestamp, message), you get a different hash, and the
commit is "different" by definition.

This is **content-derived identity**. Two repositories with the same
content have the same hashes. You can verify a repository's integrity
by checking that the root commit's hash matches what you expected.

### The connection to Merkle DAGs

A Merkle DAG (Directed Acyclic Graph) is a graph where each node is
identified by the hash of its contents, and each edge points from a
parent to a child whose hash is part of the parent's contents.

Git commits form a Merkle DAG (each commit references its parent).
IPFS files form a Merkle DAG (each chunk references its children).
Commonware's resolver keys form a Merkle DAG in a more limited sense:
each block references its parent block (via the consensus chain),
and the chain is a Merkle DAG (via MMR, chapter 06).

The key property of a Merkle DAG: **a hash uniquely identifies a
subgraph**. If you know the root hash, you can verify every node
underneath by following the references and checking hashes.

This is the foundation of CAS: the addressing IS the verification.

### The collision-resistance requirement, formally

For CAS to be secure, the hash function `H` must satisfy:

> Computational collision resistance: no PPT algorithm can find
> `x ≠ y` with `H(x) = H(y)` with non-negligible probability.

SHA-256 has 128-bit collision resistance (the underlying block cipher
is 256-bit, but birthday attacks reduce effective resistance to half).

For BFT, 128-bit collision resistance is more than enough. The
practical collision-finding attacks on SHA-256 (if any) would require
computational resources far exceeding the entire Bitcoin network.

A more subtle issue: **second pre-image resistance**. Given `x`,
finding `y ≠ x` with `H(y) = H(x)` is harder than finding any collision.
SHA-256 has 256-bit second pre-image resistance.

Commonware uses SHA-256 (chapter 04) for the resolver keys. Both
properties are easily satisfied.

### What if a collision IS found?

Hypothetically: if an attacker finds `x ≠ y` with `H(x) = H(y)`:

- They could store `x` at the address `H(x)`, then present `y` as
  "the data behind key `H(x)`." The verifier, who knows the address
  `H(x)`, would accept `y` as valid (since `H(y) = H(x)`).
- This is a "data swap" attack. The attacker replaces legitimate
  content with their own at the same hash.

Defenses:

- **Use a stronger hash.** SHA-3 (Keccak) has different security
  properties from SHA-2.
- **Use a multi-hash scheme** (IPFS's `multihash`). The address
  includes the algorithm; if SHA-256 is broken, you can verify with
  SHA-3 instead.
- **Use a Merkle path.** Even if two chunks have the same hash,
  their Merkle paths in the larger DAG would differ (because their
  contents differ). The path binds them to the DAG structure.

Commonware's resolver relies on the hash alone, not on a Merkle path.
This is fine for BFT because the blocks are large (10 MB+) and the
hash space is 256-bit — finding a collision in a 10 MB block would
require ~`2^128` operations.

### The relationship to Merkle DAGs (revisited)

For BFT block storage, Commonware's resolver isn't quite a Merkle DAG.
The keys are hashes of individual blocks, not of subgraphs. The
"graph" structure comes from the consensus chain (block N references
block N-1).

If you wanted a true Merkle DAG over the consensus state, you'd
construct an MMR (chapter 06) of all blocks and address blocks by
their Merkle path. Commonware does this in `storage::qmdb` (chapter
16), where each block's hash includes its position in the MMR.

The resolver at the substrate level uses CAS-by-content-hash; the
MMR-based addressing is layered on top by applications that need it.

## Distributed Hash Tables in full — Chord, Kademlia, Pastry

The DHT section above gave the names. Let me explain the algorithms
properly.

### Chord (Stoica et al., 2001)

Chord is the canonical DHT. The protocol:

**Address space**: identifiers are `m`-bit integers (typically
`m = 160`). Both nodes and keys live in this space.

**Hashing**: nodes have IDs `H(node_address)`. Keys have IDs
`H(key)`. Both are uniform random in `[0, 2^m)`.

**Successor**: each key is "owned" by the node whose ID is the
smallest ≥ the key's ID (modulo `2^m`). The **successor pointer**
of a node is the next node clockwise on the ring.

**Routing**: each node maintains a **finger table** with `m` entries:
```
finger[i] = successor(node_id + 2^(i-1))
```
for `i = 1, 2, ..., m`. The finger table enables `O(log N)` lookups:
at each hop, you jump to the node whose ID is closest to the target
without going past it.

**Lookup**:
```
lookup(key):
    if key in (node_id, successor]:
        return successor
    else:
        next = closest_preceding_node(key)
        return next.lookup(key)
```

**Join**: a new node `n` finds its successor via any existing node's
`lookup(n.id)`, then updates the ring (pred/succ pointers) and
finger tables of affected nodes.

**Leave**: a node `n` notifies its predecessor and successor; they
re-link. Finger tables of distant nodes may need re-stabilization.

**Stabilization**: Chord has a periodic stabilization protocol that
fixes failed pointers. The protocol is eventually consistent — after
`O(log N)` rounds of stabilization, the ring is correct.

Chord's properties:

- **`O(log N)` hops per lookup**, even with churn.
- **`O(log N)` state per node** (the finger table).
- **Eventually consistent**: lookups during churn may be incorrect
  but eventually succeed.

Chord is the basis for many production systems (CFS, Past, etc.),
but **Commonware doesn't use it** for the reasons above (small peer
set, broadcast is simpler).

### Kademlia (Maymounkov-Mazières, 2002)

Kademlia is the most widely deployed DHT (used by Mainline DHT for
BitTorrent, IPFS, Ethereum's discv4/discv5). Its key innovation is
the **XOR distance metric**.

**Address space**: 160-bit IDs (same as Chord).

**Distance**: `d(x, y) = x XOR y`. The XOR metric has a useful
property: it's symmetric (`d(x, y) = d(y, x)`) and unidirectional
(`d(x, y) < d(x, z)` implies either `d(y, x) < d(y, z)` or `d(y, x)
> d(y, z)` for any third node `y`).

**Routing table**: each node maintains a **k-bucket** for each
distance range. The k-bucket at distance range `[2^i, 2^(i+1))` holds
up to `k` (typically 20) nodes with IDs in that range. The k-buckets
together cover all `m` bits of distance, so the routing table is
`O(m × k) = O(log N × k)`.

**Lookup**: Kademlia's `FIND_NODE` and `FIND_VALUE` operations work
in parallel:
```
lookup(target):
    closest = α closest nodes to target (typically α = 3)
    queried = {}
    while closest changes:
        for node in closest \ queried:
            queried.add(node)
            response = node.find_node(target)
            closest.merge(response)
            closest.trim_to(k)
    return closest
```

The lookup converges in `O(log N)` rounds, with `α` parallel queries
per round.

**Why Kademlia won**: the XOR distance gives natural load balancing
(no "hot spots" near common keys), and the k-bucket design resists
eclipse attacks (an attacker has to fill all `k` slots in a bucket
to influence routing).

Kademlia in production:

- **Mainline DHT** (BitTorrent's bootstrap DHT) — millions of nodes.
- **IPFS** — uses a modified Kademlia for content routing.
- **Ethereum discv4** — peer discovery for Ethereum nodes.

### Pastry (Rowstron & Druschel, 2001)

Pastry uses **prefix routing**. Each node has a 128-bit ID; routing
proceeds by matching longer and longer prefixes.

**Routing table**: each node has rows for prefixes of length 0, 1,
2, ..., `m/4`. Each row has `2^4 - 1 = 15` entries (the possible
next-prefix digits, excluding the node's own). At row `i`, the entries
have IDs that share the first `i` digits with the node.

**Lookup**: at each hop, find a node whose ID shares a longer prefix
with the target. Eventually the IDs match.

**Leaf set**: each node maintains a small set of nodes with IDs
numerically close (the leaf set). Lookups can terminate at the leaf
set if the target is in the same numerical range.

**Properties**:

- **`O(log N)` hops per lookup**, with `O(log N)` state per node.
- **Locality-aware**: the routing table can be biased toward
  network-proximity neighbors.
- **Self-organizing**: handles churn via periodic stabilization.

Pastry was used in **PAST** (a distributed archival storage system)
and **SCRIBE** (a publish-subscribe system).

### Tapestry (Zhao et al., 2001)

Tapestry is similar to Pastry but with a focus on **fault tolerance
and locality**. Each node has a 160-bit ID; routing is by suffix
matching (vs. Pastry's prefix matching).

**Routing**: at each hop, find a node whose ID matches the next
suffix of the target. Like Pastry, `O(log N)` hops.

**Properties**:

- **Locality**: routing tables biased toward network-proximity.
- **Fault tolerance**: multiple paths through the routing mesh.
- **Self-repair**: nodes can detect and repair broken links.

Tapestry was used in **OceanStore**, a global persistent storage
system.

### Why Simplex replaces DHT with broadcast — a deliberate choice

Commonware's resolver assumes a **known, bounded peer set**: the
validators of the current epoch. For `N = 100`, broadcasting to all
validators is cheaper than a `log N`-hop DHT lookup.

The trade-offs:

| Property | DHT | Broadcast (Commonware's choice) |
|---|---|---|
| Peer set | Open (anyone can join) | Closed (validator set) |
| Lookup latency | `O(log N)` hops | `O(1)` hops (single broadcast) |
| Per-node state | `O(log N)` | `O(N)` (peer set) |
| Fault tolerance | High (redundant paths) | `2f+1` BFT assumption |
| Security | Eclipse attacks possible | Trusted peer set |
| Complexity | High (DHT maintenance) | Low (just broadcasting) |

For BFT consensus, the peer set is small, authenticated, and known.
DHT complexity buys nothing here.

For open systems (file sharing, public blockchains, content
distribution), DHTs are essential. Commonware's `resolver` is **not
a DHT** and shouldn't be used as one.

### What the resolver inherits from DHTs

Even without using DHT routing, the resolver borrows two ideas:

1. **Routing by key, not name.** Just as DHTs forward `(key, query)`
   toward the peer that "owns" the key, the resolver forwards
   `(key, Get)` toward a peer likely to have it.

2. **Parallel queries with first-wins.** In Kademlia, a lookup asks
   `k` peers in parallel; the first responder short-circuits the
   rest. `resolver/src/p2p/fetcher.rs` does the same: broadcasts the
   Get, takes whichever Data reply arrives first.

These are stylistic borrowings, not architectural dependencies.

## BitTorrent's piece-selection in full

The BitTorrent section above mentioned rarest-first and endgame
mode. Let me cover them in more detail.

### The BitTorrent data model

A torrent has:
- **`info`**: metadata about the files (names, sizes, piece length).
- **`info_hash`**: SHA-1 of the `info` (uniquely identifies the
  torrent).
- **Trackers**: servers that coordinate peers (or DHT for trackerless
  torrents).
- **Pieces**: chunks of the file, each ~256 KB to 1 MB.
- **Hashes**: SHA-1 hash of each piece, embedded in the `info`.

When you join a swarm, you:
1. Get the `info` (from a `.torrent` file or magnet link).
2. Connect to peers (via tracker or DHT).
3. Exchange `bitfield` messages (which pieces each peer has).
4. Request pieces using rarest-first selection.
5. Verify each piece's SHA-1 hash on receipt.

### Rarest-first in depth

The rarest-first algorithm:

```
def select_piece(my_pieces, peer_pieces):
    """Select the piece to request next."""
    available = peer_pieces - my_pieces  # pieces peer has that I don't
    if not available:
        return None

    # Count how many peers have each available piece
    counts = {p: 0 for p in available}
    for peer in all_peers:
        for piece in peer.pieces:
            if piece in counts:
                counts[piece] += 1

    # Pick the piece with the lowest count
    return min(available, key=lambda p: counts[p])
```

The goal: download pieces that are **scarce** first. This maximizes
the chance that other peers will want to download from you (because
you have rare pieces). It's tit-for-tat: you share what others want.

For a 100-peer swarm with a 1 GB file and 1 MB pieces (1000 pieces),
after a few seconds of activity, each peer has a roughly uniform
distribution. Rarest-first converges quickly.

### Endgame mode

When a download is ~99% complete, the algorithm switches to endgame
mode:

```
def endgame_mode():
    outstanding = {piece: set() for piece in my_missing_pieces}
    for peer in all_peers:
        for piece in outstanding:
            if peer.has(piece):
                outstanding[piece].add(peer)
                peer.request(piece)
                break

    while any outstanding:
        # Wait for any response
        piece, peer = wait_for_response()
        outstanding[piece].discard(peer)
        if outstanding[piece] and piece not in my_pieces:
            # Re-request from another peer
            new_peer = next(iter(outstanding[piece]))
            new_peer.request(piece)
```

In endgame mode, the same piece is requested from **every** peer that
has it. The first response "wins"; later responses are discarded.

Why: without endgame, a single slow peer could hold up the entire
download (the last piece might be held only by them, with high
latency). Endgame parallelizes the tail.

### Seeding vs leeching

- **Leeching**: downloading. You share pieces as you get them
  (tit-for-tat).
- **Seeding**: complete; you're uploading only.

The "seeder ratio" (your upload / your download) determines your
"reputation" in the swarm. Private trackers enforce minimum ratios;
public trackers don't.

For Commonware: there's no leeching/seeding distinction because
peers are persistent (validators don't churn frequently). The
broadcast engine doesn't have a "ratio" metric.

### What we borrow for peer-to-peer data dissemination

Three BitTorrent ideas that influenced broadcast engines:

1. **Piece-level verification.** Each piece has a hash. The receiver
   verifies on receipt. Commonware's `Digestible` trait (chapter 03)
   enforces the same: each value has a digest; verification is
   per-value.

2. **Parallel requests with first-wins.** In BitTorrent, you request
   from multiple peers; the first response wins. The resolver does
   the same (broadcast Get, first Data wins).

3. **Rarest-first influence on cache eviction.** BitTorrent's
   rarest-first is a download-time strategy; the broadcast engine's
   reference-counted eviction (chapter 09) is the symmetric
   cache-time strategy. The principle is the same: keep hard-to-
   replace data.

### What we don't borrow

- **Tracker-based coordination.** Commonware has a fixed peer set
  (the validators); no tracker needed.
- **Piece selection per-peer.** The broadcast engine doesn't pick
  pieces; it accepts whatever the sender pushed.
- **Tit-for-tat.** No upload/download ratio in Commonware; all peers
  are equal.

## IPFS bitswap protocol in full

The IPFS section above showed the lifecycle. Let me dig into bitswap
specifically.

### Bitswap's wire protocol

Bitswap uses protobuf-encoded messages on top of libp2p. The message
types:

- **`Wantlist`**: blocks the peer wants.
- **`Have`**: peer has block X.
- **`DontHave`**: peer doesn't have block X.
- **`Block`**: here's the data for block X.
- **`Cancel`**: I no longer want block X.

A typical exchange:

```
A → B:  Wantlist([X, Y])
B → A:  Have(X)
A → B:  Block(X)  (B sends the data unsolicited)
A → B:  Have(Y)
B → A:  Block(Y)
A → B:  Cancel(X)  (A got X, no longer wants it)
```

In practice, the messages are interleaved and the protocols are
multi-peer.

### The wantlist lifecycle

A wantlist entry has three states:

- **Wanted**: the block is needed.
- **Don't-have received**: a peer said they don't have it; the block
  is still wanted from other peers.
- **Have received**: a peer said they have it; send a request.

A block transitions out of the wantlist when:
- The data arrives and verifies.
- All peers have said "don't have" (give up).
- The wantlist is canceled by the application.

Commonware's resolver has a similar lifecycle in `inflight.rs`:

```
fetch(K) → pending[K] = { subscribers: [Me] }
              ↓
         send Get(K) to peers
              ↓
         response: Data(K, V)  →  deliver to subscribers
              ↓
         pending.remove(K)
```

The state machine is simpler than bitswap's (one peer set, no
wantlist priority), but the principle is the same.

### The connection to the Resolver pattern

Commonware's resolver is bitswap-inspired but simplified:

| Property | IPFS bitswap | Commonware Resolver |
|---|---|---|
| Peer set | Open (DHT-discovered) | Closed (validator set) |
| Wantlist | Multi-block, prioritized | Single block, single subscriber |
| Have list | Gossiped lazily | Implicit (peer responds or doesn't) |
| Block transfer | Request/response | Request/response |
| Verification | Hash on receipt | Hash on receipt + app validation |
| Cancel | Explicit Cancel message | `retain(predicate)` |

The simplification is the BFT assumption: peers are trusted
(authenticated), the peer set is small, and the application validates
responses.

### What we don't borrow from bitswap

- **Wantlist priorities.** Commonware's resolver doesn't prioritize
  within a single fetch (it's one block, one priority).
- **Have list gossip.** Commonware's resolver doesn't gossip "I have
  X" — peers respond when asked, or they don't.
- **Multi-hop forwarding.** Commonware's resolver is one hop (you
  ask a peer directly).

For a BFT peer set, the simplifications are appropriate.

### The "peer inventory" concept

Bitswap's "have list" is a peer's inventory of blocks. Commonware
doesn't maintain an explicit inventory; the broadcast engine's cache
*is* the inventory.

When a peer receives a `Get(K)`, the broadcast engine checks its
cache:

```rust
fn on_get(key: K) -> Option<V> {
    self.cache.get(&key).cloned()
}
```

If the cache has the key, the peer responds. If not, the request is
ignored (or silently dropped). This is bitswap's `Have` /
`DontHave` behavior, but implicit.

For Commonware's BFT use case, this is fine. For a CDN or a
content-distribution network, an explicit inventory might be
worthwhile (so peers can advertise their caches proactively).

## CDN patterns in full

The CDN section above gave the basics. Let me dig into more CDN
patterns.

### The CDN hierarchy

A real CDN has multiple cache tiers:

```
              Origin (data center, all data)
                 │
                 ▼
           Regional cache (continent-level)
            │      │      │
            ▼      ▼      ▼
          Edge   Edge   Edge  (city-level, close to users)
            │      │      │
            ▼      ▼      ▼
          Users  Users  Users
```

Each tier has more capacity but higher latency. The user's request
goes to the nearest edge; if the edge doesn't have it, the request
goes up to the regional cache; if not there, to the origin.

For Commonware: there's no regional tier (all validators are in a
small number of data centers). The "origin" is the leader; the
"edges" are the other validators.

### Pull vs push

- **Pull** (cache-on-miss): the cache fetches from the origin only
  when a user requests it.
- **Push** (origin writes): the origin proactively writes to caches.

Commonware combines both:
- The leader **pushes** the block via broadcast (chapter 09).
- Peers **pull** via the resolver if they missed the broadcast.

The combination is sometimes called "push-pull" or "speculative
push."

### Origin shielding

A common CDN pattern: an **origin shield** is a dedicated server in
front of the origin. The shield aggregates requests, deduplicates,
and applies rate limits. Without it, a viral event can overwhelm
the origin.

Commonware's broadcast engine plays a similar role: peers
deduplicate messages they've already cached (via reference counting,
chapter 09), so a popular block doesn't trigger N parallel fetches.

### Edge servers and PoPs

CDN edges are deployed in **Points of Presence (PoPs)** — geographic
locations close to users. The cost: each PoP needs hardware,
network, and operations.

Commonware doesn't have PoPs — the validators are the network. But
the principle is the same: minimize the latency between the data
and the consumer.

### Cache invalidation

CDNs invalidate caches by:
- **TTL** (Time To Live): entries expire after a fixed time.
- **Explicit purge**: an operator purges a specific entry.
- **Versioning**: each entry has a version; new versions invalidate
  old.

Commonware's resolver invalidates by **timestamp**: messages are
evicted from per-peer deques when fresh messages arrive
(`buffered::Engine::deques`, `broadcast/src/buffered/engine.rs:90`).
There's no explicit purge — old messages just age out.

For BFT, this is appropriate: the protocol is eventually consistent,
and old messages (e.g., from a prior epoch) shouldn't be used.

### When to use a CDN vs the resolver

CDN: for content delivery to many users (think: video streaming,
software downloads).

Resolver: for content delivery to known validators in a BFT system.

If you have a hybrid system (e.g., a blockchain with a public RPC
endpoint), you might combine:
- Resolver for inter-validator data.
- CDN for public-facing data (the same data, served via HTTP).

## Cache eviction policies in full

The cache eviction section above mentioned LRU, LFU, and ARC. Let me
go deeper.

### LRU (Least Recently Used)

The classic. On access, move the entry to the "most recent" position.
On eviction, remove the "least recent" entry.

Implementation: a doubly-linked list + hash map. The hash map gives
`O(1)` lookup; the linked list gives `O(1)` reordering.

```
class LRUCache<K, V>:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}       # K -> Node
        self.head = Node()    # sentinel
        self.tail = Node()    # sentinel
        self.head.next = self.tail
        self.tail.prev = self.head

    def get(self, key):
        if key in self.cache:
            node = self.cache[key]
            self._move_to_front(node)
            return node.value
        return None

    def put(self, key, value):
        if key in self.cache:
            node = self.cache[key]
            node.value = value
            self._move_to_front(node)
        else:
            if len(self.cache) >= self.capacity:
                # Evict LRU
                lru = self.tail.prev
                self._remove(lru)
                del self.cache[lru.key]
            node = Node(key, value)
            self.cache[key] = node
            self._add_to_front(node)
```

**Properties**:

- `O(1)` access and eviction.
- Predictable: the eviction order is deterministic given the access
  pattern.
- Works well for workloads with temporal locality.

**When LRU fails**: cache-thrashing. If the working set is larger
than the cache, every access evicts something you'll need again
soon. LRU's worst case is much worse than random.

### LFU (Least Frequently Used)

Each entry has a counter. On access, increment. On eviction, remove
the entry with the lowest count.

Implementation: a hash map + a min-heap on count. The heap gives
`O(log N)` eviction.

**Properties**:

- Better than LRU for stable working sets (long-term references).
- Worse for shifting workloads (an old entry with high count sticks
  around even after it's no longer useful).

### ARC (Adaptive Replacement Cache)

IBM's 2003 algorithm. Maintains two LRU lists:
- **T1**: recently-seen-once entries.
- **T2**: seen-multiple-times entries.
- **B1, B2**: ghost lists (metadata about evicted entries, to detect
  recency vs. frequency shifts).

The algorithm dynamically adjusts the partition between T1 and T2
based on hit ratios. ARC resists cache-thrashing better than LRU.

**Properties**:

- Self-tuning: no manual configuration of T1/T2 balance.
- More complex than LRU: 4 lists + adaptive logic.
- Rarely worth the complexity outside high-throughput systems.

### W-TinyLFU and modern variants

Modern cache libraries (Caffeine in Java, rcache in Rust) use
**W-TinyLFU**: a window LRU + a main SLRU (Segmented LRU) +
frequency sketches. Designed for high-throughput web caches.

For Commonware's consensus use case, the overhead is unjustified.
LRU is fine.

### The right cache for consensus

For BFT consensus, the right cache is **LRU with bounded size**:

- **Bounded size**: prevents unbounded memory growth.
- **LRU policy**: simple, predictable, handles consensus workload
  (sequential block access).
- **Per-peer partitioning**: each peer has its own deque (FIFO);
  the broadcast engine deduplicates across peers via reference
  counting.

The resolver's cache is `HashMap<K, V>` with implicit LRU (when
the map is full, old entries are evicted). The broadcast engine's
cache is `BTreeMap<M::Digest, M>` + per-peer `VecDeque` (FIFO).

### When LRU might NOT be fine

- **Very long histories** (e.g., 1 million blocks): LRU might evict
  a block you need for verification. Solution: a separate "pinned"
  cache for must-keep items.
- **Random access patterns** (e.g., serving RPC queries for any
  block): LRU might evict what users are currently asking for.
  Solution: ARC or a hot-key detector.
- **Bursty workloads** (a sudden flood of requests for old blocks):
  LRU evicts to make room. Solution: a separate overflow cache.

For BFT consensus, none of these typically apply. The workload is
sequential and the history is bounded.

## Exercises

These exercises build intuition for the resolver's design choices.

### Exercise 1 — CAS collision probability

Implement SHA-256 in your language of choice (or use a library) and
compute:

1. How many random inputs do you need to find a collision (via
   birthday paradox)? Formula: `√(π/2 × 2^256) ≈ 2^128`.
2. For 32-byte inputs (matching Commonware's resolver keys), how
   many inputs to expect a collision?
3. If an attacker can compute `10^18` hashes per second (a
   state-of-the-art mining rig), how long to find a collision?
   Answer: `2^128 / 10^18 ≈ 10^20` seconds, or `3 × 10^12` years.

**Take-away**: SHA-256 collision resistance is enough for any
practical BFT deployment.

### Exercise 2 — Chord finger table

Implement a Chord node with `m = 8` (256-node space) and verify:

1. `lookup(key)` returns the correct successor.
2. After adding 10 nodes, lookups still work (with `O(log N)` hops).
3. After removing a node, lookups still work (stabilization).

**Stretch goal**: simulate a churn event (10 nodes joining/leaving
in quick succession) and verify the ring stabilizes.

### Exercise 3 — Rarest-first simulation

Implement a BitTorrent-style rarest-first simulator:

1. 100 peers, 1000 pieces, each peer has a random subset of 100
   pieces.
2. Each peer wants to download all 1000 pieces.
3. Each round: each peer selects a piece (rarest-first) and
   "downloads" it (instant).
4. After 100 rounds, what's the download distribution? Are rare
   pieces still rare?

**Stretch goal**: add latency (each download takes 1-10 rounds) and
verify rarest-first is still optimal.

### Exercise 4 — Cache eviction comparison

Implement LRU, LFU, and ARC caches in Rust:

1. LRU: doubly-linked list + HashMap.
2. LFU: HashMap + min-heap on frequency.
3. ARC: two LRU lists + ghost lists + adaptive partitioning.

Benchmark on three workloads:

- Sequential access (1, 2, 3, ..., 100, 1, 2, ...).
- Random access (uniform random keys).
- Working-set fit (random access, but the working set fits).

For each, measure the hit rate and the operations-per-second.

**Take-away**: ARC wins on random access with a small working set,
but LRU is competitive on most practical workloads.

### Exercise 5 — Bitswap-lite

Implement a simplified bitswap protocol:

1. Two peers (A and B) connected by an unreliable channel.
2. A sends `Wantlist([K1, K2, K3])`.
3. B responds with `Block(K1, data1)`.
4. A verifies the hash of `data1` matches K1.
5. A sends `Cancel(K2)` (no longer wants K2).

This is the basic bitswap flow. Compare to Commonware's resolver
flow (one Get, one Data, no Cancel message — use `retain`).

### Exercise 6 — CDN hierarchy simulation

Simulate a 3-tier CDN (origin, regional, edge):

1. 1000 users issuing requests uniformly across 1000 blocks.
2. Origin has all blocks; regional has 10% (random); edge has 1%.
3. Each request: edge → regional → origin if needed.

Measure:

- Cache hit rate at each tier.
- Average request latency.
- Load on the origin (requests served).

**Stretch goal**: add "push" — when origin adds a new block, it
writes to all regional caches. Measure the change in hit rate.

## Where to look in the code (intermediate)

These pointers bridge the new sections to the implementation.

- `resolver/src/lib.rs:21-213` — the traits and core types.
- `resolver/src/p2p/engine.rs` — the engine that drives the resolver.
- `resolver/src/p2p/inflight.rs` — in-flight fetch tracking.
- `resolver/src/p2p/config.rs` — config (cache size, timeouts).
- `resolver/src/p2p/wire.rs` — wire format (Get, Data messages).
- `resolver/src/ingress.rs` — incoming requests.
- `resolver/src/subscribers.rs` — subscriber tracking.
- `resolver/src/delivery.rs` — delivery bookkeeping.
- `resolver/src/opaque.rs` — type-erased resolver (for dynamic
  dispatch when needed).

## If you only remember three things (expanded)

1. **Resolver = "fetch data for key K."** Same key requested by N
   callers → one network request → N deliveries via subscribers.
   The CAS model guarantees correctness without a name authority.

2. **TargetedResolver = "ask X first, then broadcast."** Useful when
   you have a hint about who has the data (e.g., a peer that just
   signed a certificate).

3. **`retain` cleans up subscriptions.** Otherwise you're holding
   network resources for fetches nobody cares about anymore. And
   `retain(predicate)` is the cancel mechanism — use it carefully.

## Appendix A — The full fetch lifecycle

`resolver/src/ingress.rs`, `resolver/src/p2p/`, `resolver/src/subscribers.rs`,
`resolver/src/delivery.rs`. When you call `resolver.fetch(key)`:

### Step 1 — the Fetch struct

```rust
pub struct Fetch<K, S = ()> {
    pub key: K,
    pub subscriber: S,            // who wants this data?
    pub span: tracing::Span,      // trace span across actor boundaries
}
```

The `subscriber` is your **identity** in the resolver. Multiple callers
asking for the same key with the same subscriber collapse to one fetch.
The `span` is the tracing context (chapter 02) that crosses actor
boundaries — the receiver of the data sees the same trace.

### Step 2 — registration

The resolver checks if `key` is already in flight:

- **Yes**: register your subscriber with the in-progress fetch.
- **No**: start a new fetch. Send `Get(key)` to peers.

### Step 3 — peer query

For `TargetedResolver::fetch_targeted(key, targets)`:
1. Try the targets first (in order).
2. If no response within `target_timeout`, fall back to broadcasting.

For `Resolver::fetch(key)`:
1. Broadcast `Get(key)` to all active peers.
2. Wait for first response.

### Step 4 — response arrival

A peer responds with `Data(key, value)`. The resolver:

1. Verifies the response matches the key (defense against confused peer).
2. Validates the value (application-specific check via `Consumer::deliver`).
3. If valid: deliver to all registered subscribers. Mark fetch complete.
4. If invalid: ignore this peer, retry with another (or blacklist).

### Step 5 — delivery

```rust
let delivery = Delivery {
    key,
    subscribers: NonEmptyVec<(subscriber, span)>,
};
let accepted = consumer.deliver(delivery, value).await?;
```

`accepted` is `true` if the consumer (you) wants the value; `false` if
everyone has dropped interest.

### Step 6 — cleanup

If `accepted`: remove the fetch from the in-progress map.
If `!accepted`: also remove (no one wants it).

The subscribers are released; their `oneshot::Receiver<bool>` resolves
with the result.

## Appendix B — The subscriber mechanics

`resolver/src/subscribers.rs`. The internal subscriber tracking:

```rust
struct Subscription<S> {
    predicate: Option<Box<dyn Fn(&S) -> bool + Send>>,
    subscribers: HashMap<K, Vec<S>>,
}
```

The resolver groups subscribers **by key**. Same key → same `Vec<S>` of
subscribers. When the data arrives, all subscribers receive it.

`retain(predicate)` runs `predicate` over each subscriber; those that
fail are removed:

```rust
fn retain(&mut self, predicate: impl Fn(&K, &S) -> bool + 'static) {
    for (key, subscribers) in &mut self.subscribers {
        subscribers.retain(|s| predicate(key, s));
    }
    // Remove empty entries
    self.subscribers.retain(|_, subs| !subs.is_empty());
}
```

After `retain`, any fetch with no remaining subscribers is **canceled**.
The resolver drops the in-progress state, stops asking peers, ignores
future responses for that key.

## Appendix C — The `TargetedResolver` internals

```rust
pub trait TargetedResolver: Resolver {
    type PublicKey: PublicKey;
    fn fetch_targeted(
        &mut self,
        fetch: impl Into<Fetch<Self::Key, Self::Subscriber>> + Send,
        targets: NonEmptyVec<Self::PublicKey>,
    ) -> Feedback;
}
```

The `targets` is a `NonEmptyVec` — you must provide at least one
target. The resolver tries them in order; if all fail, it falls back to
broadcast (if implemented).

The implementation can also **merge** targets with in-progress
fetches. If you call `fetch_targeted(K, [X])` and someone else already
started `fetch(K)` (broadcast), the resolver can decide:
- Add your target X to the existing fetch.
- Use your target X instead of broadcast.
- Ignore your target X and rely on the existing broadcast.

Commonware's P2P-backed resolver (`resolver/src/p2p/`) does the simple
thing: target hints are passed to the first query, then fall back to
broadcast if no response.

## Appendix D — The `Delivery` type in depth

`resolver/src/lib.rs:67-89`:

```rust
pub struct Delivery<K, S> {
    pub key: K,
    pub subscribers: NonEmptyVec<(S, tracing::Span)>,
}
```

`NonEmptyVec` — there must be at least one subscriber. (If there were
zero, the fetch wouldn't exist.) The `tracing::Span` per subscriber is
the **originating trace span** — used to preserve the trace context
across the actor boundary.

```rust
impl<K: PartialEq, S: PartialEq> PartialEq for Delivery<K, S> {
    fn eq(&self, other: &Self) -> bool {
        self.key == other.key
            && self.subscribers.len() == other.subscribers.len()
            && self.subscribers.iter()
                .zip(other.subscribers.iter())
                .all(|((a, _), (b, _))| a == b)
    }
}
```

PartialEq ignores the spans — comparison is on identity, not on
context. Spans are for tracing only.

## Appendix E — The `Subscriber` trait, in detail

```rust
pub trait Resolver: Clone + Send + 'static {
    type Subscriber: Clone + Eq + Send + 'static;
    // ...
}
```

The `Subscriber` type is whatever you want it to be, as long as it's
`Clone + Eq`. Common uses:
- A simple unit type `()` when there's only one consumer per process.
- An actor handle (`Mailbox<S>`) when multiple actors might fetch.
- An enum variant per app domain.

The resolver uses `Subscriber` purely as an **identifier** — it groups
fetches by `(key, subscriber)` and dispatches to all subscribers when
data arrives.

## Appendix F — The P2P integration, internals

`resolver/src/p2p/`. This is the standard construction:

### Channels

The resolver uses two P2P channels:
- **Channel 0** (sender): for `Get(key)` requests.
- **Channel 1** (receiver): for incoming `Data(key, value)` responses.

Each is registered with appropriate quotas:

```rust
let (requester, responder) = network.register(
    0,                                  // channel ID
    Quota::per_second(NZU32!(10)),      // 10 requests/sec/peer
    256,                                // 256 messages in flight
);
```

### Request format

```rust
pub enum Request<K, V> {
    Get { key: K },
}
```

Encode with codec (chapter 03):

```rust
let encoded = Request::Get { key: K::encode(&key) }.encode();
sender.send(Recipients::All, encoded, false).await?;
```

### Response format

```rust
pub enum Response<K, V> {
    Data { key: K, value: V },
}
```

Encoded similarly.

### Handling incoming requests

A node receiving a `Get(key)` looks up its local cache. If found,
respond with `Data(key, value)`. If not found, ignore (or queue for
later; common implementations just drop if not in cache).

```rust
impl Handler for Responder {
    fn process(&mut self, origin: PublicKey, request: Request<K, V>) {
        match request {
            Request::Get { key } => {
                if let Some(value) = self.cache.get(&key) {
                    let _ = self.responder.send(Response::Data { key, value });
                }
            }
        }
    }
}
```

The `oneshot::Sender<Response>` is provided by the resolver's P2P
machinery. If the sender drops (e.g., the requester disconnected), the
response is silently dropped.

## Appendix G — Caching and pruning

The resolver maintains an internal cache:

```rust
struct ResolverState<K, V, S> {
    cache: HashMap<K, V>,                    // resolved data
    pending: HashMap<K, PendingFetch<S>>,  // in-progress fetches
    blocked: HashSet<PublicKey>,            // peers that lied
}
```

`cache` is bounded — typically LRU with a fixed size. When full, oldest
entries are evicted.

`pending` is also bounded — typically by `fetch_concurrent` config. If
too many fetches are in flight, new fetches are delayed or rejected.

`blocked` is the **blocker list** — peers that returned bad data. The
resolver marks them; subsequent fetches skip them.

## Appendix H — Common patterns in production

### The "lazy fetch" pattern

```rust
// In the Voter:
async fn on_notarization(&mut self, cert: Notarization) {
    // Mark the block as needed
    self.needed_blocks.insert(cert.proposal.payload);

    // Don't fetch immediately — wait until we actually need it
    if self.needed_for_current_view(cert) {
        self.resolver.fetch(cert.proposal.payload).await?;
    }
}
```

Only fetch blocks you actually need. Saves bandwidth.

### The "fetch ahead" pattern

```rust
// In the Voter, when entering a new view:
async fn enter_view(&mut self, view: View) {
    // Pre-fetch blocks we'll need
    for height in self.needed_blocks() {
        self.resolver.fetch(height).await?;
    }
    // ... enter view ...
}
```

Start fetches early so data arrives by the time you need it.

### The "block misbehaving peers" pattern

```rust
impl Consumer for MyConsumer {
    async fn deliver(&mut self, delivery, value) -> bool {
        if !self.validate(&value) {
            self.blocker.block(delivery.subscriber);
            return false;
        }
        // ... use value ...
        true
    }
}
```

If a peer returns bad data, block them. The resolver marks them; future
fetches skip them.

## Appendix I — Edge cases and gotchas

### Subscriber dedup is `Eq`-based

The resolver uses `==` to check if two subscribers are the same. If your
subscriber type has more state than identity matters, you'll get
incorrect dedup. Use a key-only type (enum variant, struct with just an
ID).

### The `NonEmptyVec<S>` invariant

`Delivery::subscribers` is `NonEmptyVec` — must have at least one
subscriber. If you implement a custom `Consumer`, ensure your `deliver`
implementation never creates an empty `Delivery`.

### Retain with no predicate

`retain(|_, _| false)` cancels all fetches. Useful for cleanup, but
**don't call it accidentally** — you'll cancel everything.

### Cancellation semantics

When a fetch is canceled (via `retain` or `cancel`), any in-progress
`deliver` future is **dropped**. The peer may have already sent the
data, but the response is discarded. The resolver doesn't pay for
unwanted deliveries.

## Where to look in the code (expanded)

- `resolver/src/lib.rs:21-213` — the traits and core types.
- `resolver/src/delivery.rs` — delivery bookkeeping.
- `resolver/src/ingress.rs` — incoming requests.
- `resolver/src/p2p/` — the P2P-backed construction.
- `resolver/src/subscribers.rs` — subscriber tracking.
- `resolver/src/opaque.rs` — opaque (type-erased) resolver.

## If you only remember three things

1. **Resolver = "fetch data for key K."** Same key requested by N callers → one network request → N deliveries via subscribers.
2. **TargetedResolver = "ask X first, then broadcast."** Useful when you have a hint about who has the data.
3. **`retain` cleans up subscriptions.** Otherwise you're holding network resources for fetches nobody cares about anymore.

→ Next: **Chapter 09 — Broadcast**. The wide-area data dissemination primitive
that uses coding under the hood.