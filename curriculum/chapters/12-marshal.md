# Chapter 12 — Marshal: from consensus certificates to finalized blocks

> The layer that turns "quorum agreed on hash H" into "I have the block behind H, verified it, applied it."

## What Simplex gives you vs what you want

After Simplex chapter 11, your node has:

- Notarizations — "2f+1 voted for hash H in view V"
- Finalizations — "2f+1 voted to finalize hash H in view V"

But you don't have **the block behind H**. You have a 32-byte commitment.

To actually **do** anything with consensus (execute transactions, update
state, etc.), you need the full block. Marshal is the layer that closes
this gap.

## What Marshal does

`consensus/src/marshal/mod.rs:1-15`:

> The core of the module is the unified [`core::Actor`]. It marshals
> finalized blocks into order by:
>
> - Receiving uncertified blocks from a broadcast mechanism
> - Receiving notarizations and finalizations from consensus
> - Reconstructing a total order of finalized blocks
> - Providing a backfill mechanism for missing blocks

So Marshal does five things:

1. **Receive uncertified blocks** from the broadcast layer (chapter 09) or
   coding layer (chapter 07).
2. **Receive notarizations + finalizations** from Simplex.
3. **Match them up.** Notarization for hash H means there's a block at H.
   The block has to be received (from step 1) and verified (against the
   notarization).
4. **Deliver the blocks in order** to your application. **At-least-once**
   delivery (you handle dedup).
5. **Backfill** missing blocks by asking peers via the resolver (chapter 08).

## The trait surface — what Marshal needs

Looking at the imports in `mod.rs:8-18`:

```rust
use crate::Reporter;                          // deliver finalized blocks
use crate::simplex;                            // get consensus messages
use commonware_broadcast::buffered;           // receive uncertified blocks
use coding::shards::Engine;                   // receive erasure-coded shards
use resolver;                                  // backfill mechanism
```

Marshal is a thin glue layer. It doesn't do consensus — that's Simplex. It
doesn't do cryptography — that's the scheme. It doesn't do networking —
that's P2P. It just **matches up blocks and certificates**.

## The two modes: standard vs coding

`marshal/standard/` and `marshal/coding/`:

- **Standard mode** — receives full blocks via the broadcast engine
  (chapter 09). Each peer broadcasts the full block.
- **Coding mode** — receives erasure-coded shards via the `coding::shards::Engine`
  (chapter 07). Each peer broadcasts shards; recipients reconstruct.

Coding mode is for big blocks where bandwidth matters. Standard mode is for
small blocks where simplicity wins.

## The `store` — durable block storage

`marshal/store.rs` — Marshal's storage interface. Two parts:

- **`Certificates`** — stores notarizations and finalizations (so you can
  prove "block H was finalized" later).
- **`Blocks`** — stores the actual block data, indexed by height or digest.

Backends can be in-memory (tests), on-disk (production), or in cloud
storage (`archive` module from chapter 06).

## Backfill

> The actor provides a backfill mechanism for missing blocks. If the actor
> notices a gap in its knowledge of finalized blocks, it will request the
> missing blocks from its peers. (`mod.rs:37-41`)

Say you come online and Simplex has already finalized blocks 1-100. You
have block 50 (from a recent broadcast) but you're missing 1-49. Marshal
sees the gap, calls the resolver for each missing block, peers respond,
you apply them in order.

This is the "state sync" path — fast catch-up without re-running consensus
from genesis.

## The `Start` config — syncing from a height

`marshal::config::Start`. You can configure Marshal to start from a
specific finalized height rather than genesis:

```rust
let cfg = marshal::Config {
    start: marshal::Start::Height(1000, finalization_for_height_1000),
    ...
};
```

This is useful for light clients, validators joining late, or applications
that don't need full block history.

The trade-off (`mod.rs:57-59`):

> Setting a configurable starting height will prevent others from
> backfilling blocks below said height. This feature is only recommended
> for applications that support state sync.

So if you start at height 1000, peers won't backfill you blocks below
1000 — you have to fetch them yourself.

## The ancestry — building the chain

`marshal/ancestry.rs`. Once you have a finalization for height H, you need
the finalization for H-1 (the parent). The `Ancestry` trait iterates
backwards through the chain:

```rust
pub trait Ancestry<B: Block> {
    type Item;
    fn parent(&self) -> Option<Self::Item>;
}
```

This is what lets you verify "block H's parent is block H-1, which was
finalized at view V' — which I have a finalization for — so H is valid."

## The `Application` interface

Marshal interacts with your code through an `Application` trait
(`lib.rs:256-303`):

```rust
pub trait Application<E>: Clone + Send + 'static {
    type SigningScheme: Scheme;
    type Context: Epochable;
    type Block: Block;

    fn propose(&mut self, context: (E, Self::Context),
               ancestry: impl Ancestry<Self::Block>)
        -> impl Future<Output = Option<Self::Block>> + Send;

    fn verify(&mut self, context: (E, Self::Context),
              ancestry: impl Ancestry<Self::Block>)
        -> impl Future<Output = bool> + Send;
}
```

Same shape as Simplex's `Automaton`, but with an `Ancestry` parameter so
you can walk the chain backward to validate.

## Putting it together

```
                  Consensus (Simplex)
                         │
                         ▼ (notarizations, finalizations)
                  ┌───────────────┐
                  │               │
   peers ─────►   │    Marshal    │ ────► Reporter (your code)
                  │               │       (gets finalized blocks in order)
   broadcast ──►  │               │
                  │               │
   resolver ───►  └───────────────┘
                         │
                         ▼
                   store::Blocks, store::Certificates (chapter 06)
```

Marshal is the bridge between "consensus agreed" and "you have the data."

## Where to look in the code

- `consensus/src/marshal/mod.rs:1-65` — the architecture.
- `consensus/src/marshal/core/` — the unified actor.
- `consensus/src/marshal/standard/` — full-block mode.
- `consensus/src/marshal/coding/` — erasure-coded mode.
- `consensus/src/marshal/store.rs` — the storage interface.
- `consensus/src/marshal/ancestry.rs` — the parent-walking trait.
- `consensus/src/marshal/resolver/` — the backfill integration.
- `examples/bridge/` — an example using Marshal + Simplex for cross-chain
  certificates.

## Block propagation strategies — push, pull, hybrid

When a leader finalizes a block, every other node needs the block's
bytes. There are three shapes for getting data from leader to peers:

**Push (broadcast-first).** The leader broadcasts the block
immediately. Peers store it. By the time consensus finalization
arrives, peers already have the data. Cheapest path on the happy
path: `O(N)` for the leader (one outbound copy per peer).

**Pull (fetch-on-demand).** The leader only broadcasts the *digest*
(the proposal). Peers who learn the digest (via gossip or via a
certificate that names the digest) fetch the block from peers. The
leader's egress is `O(1)`. Peers' egress is `O(N × block_size) / N`
(distributed) — but the read pattern means each peer may pull from
multiple sources.

**Hybrid.** Some combination of the two — e.g., broadcast the
digest + a small "block-pointer" packet, and let the broadcast
engine (chapter 09) handle the bytes via codec-sharded
flooding on the back-end.

Commonware's Simplex uses **pull** for blocks:

1. Leader broadcasts a `Proposal { parent, payload }` — the
   payload is the digest, not the bytes.
2. Peers see the digest via the proposal. To verify, they need the
   bytes behind the digest.
3. The Voter fetches the bytes via the Resolver (chapter 08),
   which queries the Broadcast engine (chapter 09) or the Coding
   engine (chapter 07) for the actual data.

A Hybrid equivalent could push the *small* parts (digests,
certificates) and *pull* the large parts (blocks). Commonware
chooses pull because:

- The leader's egress stays bounded (no big broadcasts).
- The Resolver's existing infrastructure handles "fetch by key"
  cleanly.
- Block delivery is asynchronous — peers can fetch the bytes
  *after* receiving the certificate, not in lockstep.

The cost is *some* latency: a peer that just heard the notarization
needs an extra fetch round trip to get the bytes before voting to
finalize. The `CertifiableAutomaton::certify` hook (chapter 11)
keeps this latency budget visible: if certification takes longer
than `certify_timeout`, you nullify the view.

For comparison, push (e.g., Avalanche-style) would mean:

```
leader broadcasts B → peers cache B → leader broadcasts notarization of B
                          ────────────
                          (zero extra fetch)
```

But the broadcast itself is large. For N=100 and a 1 MB block,
the leader pushes 100 MB outbound before the first vote. With
any reasonable uplink (say, 1 Gbps), that's 1 second of full
egress. Pull saves the leader that 1 second.

## Data availability vs validity — and where ZODA fits

Distributed systems distinguish two questions about a piece of
data:

**Validity.** Is the data well-formed? Does it respect the
application's rules? For a block, validity means correct
signatures, balanced Merkle tree, no double-spend.

**Availability (DA).** Can the data be retrieved if you need it?
A piece of data is "available" if a quorum of peers holds it
(so asking them round-robin works). DA is *physical*: the bytes
exist on someone's disk.

The distinction matters because a Byzantine validator could
**publish only the digest of a block** (denying peers the bytes)
and force the chain to halt. The defense is "DA checks": ensure
peers store the data *before* voting to finalize.

Commonware's coding primitive ZODA (`coding/zoda/`) is a **DA
primitive**. It encodes a block into `n` shards such that:

- Any `k` shards reconstruct the original.
- `n - k + 1` shards are needed to *prevent* reconstruction
  (don't publish `n - k + 1` shards, you deny the data).

The protocol:

1. Leader encodes block B into `n` shards.
2. Leader sends a "strong" shard to each of `n - k + 1` peers,
   keeping the weak ones.
3. Leader publishes a `DA_certificate` saying "n - k + 1 peers
   have strong shards."
4. Validators verify: "do I have enough peers to retrieve the
   full block?" — i.e., can I contact `n - k + 1` peers?

ZODA is *not* validity. The reconstructed bytes could still be
invalid. The application (chapter 11) catches validity issues.

In Marshal's coding mode (`marshal/coding/`), the interaction is:

- Marshal receives ZODA shards via broadcast.
- It waits for `k` shards, reconstructs the block.
- It hands the reconstructed block to `Automaton::verify` for
  validity.
- If both pass, Marshal votes to notarize and finalize.

A subtle point: ZODA's threshold is `n = O(k × log N)` for
practical deployment. With `N = 100` and `k = 67` (quorum),
the encoding is `n = 134 → 200` shards. The 33 extra shards
provide redundancy in the network.

## State sync vs block sync

Joining a chain late, you face a choice:

**Block sync.** Download every block from genesis (or your
known floor) to the current height, replay each through the
state machine, end up at the current state. Slow but correct;
reproduces the chain's evolution.

**State sync.** Trust a recent finalized *state root* (Merkle
root of the final state), download just the final state,
verify the state root against the on-chain proof. Fast but
needs an application that can serialize its state.

Kleppmann's DDIA chapter on "stream processing" and "stateful
services" describes this trade-off in detail — block sync is the
"log-compaction" approach, state sync is the "snapshot +
incremental" approach.

Commonware Marshal supports both:

- **Block sync** is the default. You start at a height (or
  genesis), and Marshal backfills blocks one at a time. Each
  block goes through the application (so the application
  processes every transaction in order).
- **State sync** requires the application to expose a
  `set_state(bytes)` method. The application trusts the
  Marshal-provided state root is correct (Marshal has the
  finalization certs for the chain history) and uses it as
  the starting point.

The `Start::Height(h, finalization_at_h)` enum (`marshal/Start`)
is the block-sync starting position. For state sync, the
application sets a `Start` and processes the resulting
`Update::Tip` not as a block to apply but as a snapshot to
trust.

The trade-off table:

| Property | Block sync | State sync |
|---|---|---|
| Time to catch up | O(blocks × block_time) | O(state_size) |
| Trust model | Replay = verify | Trust state root + finalization certs |
| Disk usage | Full block history | Just state + recent blocks |
| Computation | Re-execute every transaction | Single deserialization |
| Backward compatibility | Always works | Application must support |

For a new validator joining an active chain, state sync is
almost always the right choice. For a long-running validator
that's recovering from a crash, block sync from the last-known
floor is right.

## Pruning strategies — discarding old blocks safely

The `Blocks` trait's `prune(below)` method
(`marshal/store.rs`) is the lever for controlling disk usage.
Three pruning strategies you might implement:

**Depth-based.** Keep the last `N` heights of blocks, prune
older. Safe if no peers need backfill below the floor.

**Cost-based.** Keep until disk pressure exceeds a threshold;
prune the oldest first. Cleanest from an ops perspective.

**Explicit floor.** Caller (the application) decides the floor,
Marshal honors it. Useful for "we only need the state after
height 5000" configurations.

The consensus floor — the lowest height for which we still need
certificates — is a separate setting
(`simplex::Config::Floor::Genesis(...)` or specific height).
Marshal's floor can be higher than the consensus floor; that's
a state-sync-style optimization.

Critical invariant for safe pruning: **never prune below the
consensus floor** without a state sync to jumpstart the
recovery. If a peer asks you for a block below the floor, you
must say "I don't have it; backfill me from genesis (or your
own state)".

Commonware's `Blocks::prune(below)` returns `Result<()>`, so a
backend can refuse to prune (e.g., "you told me to keep the
last 1000, I have 1500 left"). The application decides
how to handle that error — typically by logging it and
continuing (it'll get pruned next time).

## The two modes of Marshal — Standard vs Coding

`consensus/src/marshal/standard/` and
`consensus/src/marshal/coding/` differ in *how* blocks move
from peer to peer:

**Standard mode (`marshal/standard/`).**

- Receives full blocks via `broadcast::buffered::Engine`
  (chapter 09).
- Each peer has a per-peer cache (`buffered::Engine::deques`).
- A peer asks for block `H`, the Resolver finds a peer with
  `H` cached, sends the bytes.

Pros: simplest possible mental model ("blocks are broadcast,
cached, served on demand"). Best for *small* blocks (a few KB
to a few hundred KB) where the per-peer cache can hold recent
blocks comfortably.

Cons: a single peer's egress is `O(blocks × N)` for one round
of delivery (everyone asks, everyone responds). Bandwidth
unfavorable for big blocks.

**Coding mode (`marshal/coding/`).**

- Receives shards via ZODA (`coding/zoda/`).
- Each peer gets one *strong* shard from the leader and
  accumulates weak shards from gossip.
- Reconstruct when `k` shards are available.
- Hand the reconstructed block to `Automaton::verify`.

Pros: leader egress is `O(N/k)` per round (much smaller for
small `k`); per-peer cache holds shards not whole blocks;
DA is built-in.

Cons: more complex; needs a coding library that aligns with
block semantics; the per-shard overhead (signature per shard,
encoding/decoding) means small blocks have higher
relative overhead.

The standard choice:

| Block size | Mode |
|---|---|
| < 100 KB | Standard (simplicity wins) |
| 100 KB – 10 MB | Either — Standard is fine if peer set is small, Coding if large |
| > 10 MB | Coding (bandwidth wins) |

The default in `examples/alto` (Alto chain) is Coding. Small
chains (community experiments, subnets) often use Standard.

A key implementation detail: the two modes share the **same
trait surface** (`marshal::Application`). The application
doesn't care which mode it gets. Marshal's `Config` picks
the mode at startup.

## The Store interface — `Certificates` and `Blocks`

Marshal's storage abstraction is two traits
(`consensus/src/marshal/store.rs`):

### `Certificates`

```rust
trait Certificates {
    async fn put_notarization(&mut self, h: Height, cert: Notarization);
    async fn put_finalization(&mut self, h: Height, cert: Finalization);
    async fn get_notarization(&self, h: Height) -> Option<Notarization>;
    async fn get_finalization(&self, h: Height) -> Option<Finalization>;
    async fn last_finalized(&self) -> Option<(Height, Finalization)>;
}
```

This is **the cryptographic proof layer**. A certificate is
signed by `2f+1` validators; it's a self-contained proof you can
verify independent of Marshal. The store persists them
durably so that even after a crash, you can prove "block H was
finalized at view V" from the bytes alone.

**Backends:**

- **In-memory** (`tests`): `BTreeMap<Height, Cert>`.
- **On disk** (`storage::archive::Archive`): append-only with
  periodic compaction.
- **Cloud storage**: backed by S3 or GCS, used in production.
  Slow access, but reliable and large.

The `last_finalized` operation is the critical one for
recovery — it's where Marshal looks after a crash to figure out
where to resume.

### `Blocks`

```rust
trait Blocks<B: Block> {
    async fn put(&mut self, h: Height, block: B) -> Result<()>;
    async fn get(&self, h: Height) -> Result<Option<B>>;
    async fn get_by_hash(&self, hash: Digest) -> Result<Option<B>>;
    async fn prune(&mut self, below: Height) -> Result<()>;
}
```

The actual block bytes. Two access patterns:

- **By height** (sequential replay).
- **By hash** (random fetches).

Two writes: `put` (called when Marshal gets a block and finalizes).
One prune: `prune(below)` (caller-driven).

The Marshal actor never owns these — it's the user / application
that provides concrete implementations. The Commonware examples
use `storage::archive::Archive` for both, sometimes with two
different archive instances (one for certs, one for blocks).

## Backfill in detail

When Marshal notices a gap in its finalized blocks — say it has
finalizations for heights 100 and 200, but not 101-199 — it
initiates backfill:

```
1. Marshal observes finalization for height 200.
2. Walks the floor: max contiguous finalized height = 100.
3. Range (101, 200) is the gap.
4. For each missing height, request via Resolver:
     resolver.fetch(MarshalKey::Height(h))
5. Peers respond with the missing blocks (and their finalizations).
6. Marshal verifies each, stores them, and processes them in order.
```

The Resolver keys for Marshal are typically `Height` or `Digest`
(depending on the marshal backend). The `examples/bridge/`
implementation shows a custom key type that wraps both.

**Backfill is bounded.** A marshal that lags 10,000 blocks won't
ask for all 10,000 at once (network implosion). Commonware uses
a sliding window:

```rust
const MAX_INFLIGHT_BACKFILLS: usize = 64;

async fn check_for_gap(&mut self) {
    let last = self.finalized.keys().rev().next();
    if let Some(last_height) = last {
        let outstanding = self.outstanding_fetches.len();
        let budget = MAX_INFLIGHT_BACKFILLS.saturating_sub(outstanding);
        // Fetch up to `budget` heights...
    }
}
```

You can see hints of this in `marshal/core/actor.rs`'s gap-detection
loop (Appendix A).

**Backfill interacts with Simplex.** If you're catching up via
backfill, you also need the *certificates* for the views you
backfilled through. Simplex's Voter fetches missing
notarizations / nullifications on demand (chapter 11 main text,
"Forced Inclusion"). Marshal's backfill and Simplex's recovery
are independent paths to the same goal: catch up.

## The `Start` mechanism — bootstrap from anywhere

`marshal::config::Start` is the lever for starting from a
specific position:

```rust
pub enum Start {
    Default,                                  // Use config floor; genesis or last-known.
    Height(Height, Finalization),             // Start at this height; finalization is the proof.
}
```

The `Start::Height(h, f)` variant does several things:

1. **Skips pre-`h` work.** Marshal doesn't try to verify
   `f.certificates` against the application — it just trusts
   `f` as the historical floor.
2. **Triggers a fetch of block at `h`.** Before processing
   anything, Marshal asks the Resolver for the block at height
   `h`. This is so the application has a starting state.
3. **Rejects backfill below `h`.** If a peer tries to backfill
   you `h-1`, you reject (you've declared you don't care).
4. **Hooks the application** if it cares. Some applications use
   the optional `Start` to load a saved state, verify against
   the provided finalization, and resume at `h+1`.

This is the **light-client entrypoint**. A joining validator
can declare "I trust state root `R` at height `h`; give me
everything since." The `Examples::bridge` shows this pattern:
the bridge downloads a finalization cert from the source chain,
hands it to Marshal as `Start::Height(h, cert)`, and Marshal
processes subsequent blocks.

## Finalization acknowledgment — blocking until applied

Once Marshal decides "block H is finalized," it hands H to your
application via the `Reporter`:

```rust
trait Reporter {
    type Activity;
    fn report(&mut self, activity: Self::Activity) -> Feedback;
}
```

`Activity` for Marshal is `Update<B>` (the variant
`Update::Block(block, ack)` in `marshal/mod.rs:135-152`).

The `ack` is an `Acknowledgement` that the application signals
once it's done processing:

```rust
async fn on_finalized_block(&mut self, block: B) {
    // Execute transactions
    apply_transactions(&block).await?;
    // Update state root
    self.commit_state(block.parent()).await?;
    // Acknowledge (signals Marshal that we're done with this block)
    self.ack.signal(()).unwrap();
}
```

Without this acknowledgment, Marshal would either (a) never
finalize another block, or (b) advance past blocks before the
application has applied them. The two failure modes are bad in
opposite ways: stuck, or inconsistent.

**Idempotency required.** At-least-once delivery means the
application may see the same block twice (e.g., it crashed
between applying and acknowledging, then Marshal re-delivers).
The application must handle dedup — typically by hashing the
block and short-circuiting if already applied.

A useful Rust pattern in this loop:

```rust
let (tx, rx) = oneshot::channel();
// Marshal: send ack via tx.send
// Application: do work, then ack_complete.signal(()).unwrap();
```

The channel pairs the *delivery* (Marshal → application) with
the *acknowledgment* (application → Marshal). If the
application crashes before acking, Marshal holds the block as
un-acked and re-delivers on restart. The application's
idempotency check prevents double-execution.

## The `Application` interface — and the `impl Future` pattern

The Marshal `Application` trait (`consensus/src/lib.rs:256-303`)
is shaped like Simplex's `Automaton`:

```rust
pub trait Application<E>: Clone + Send + 'static {
    type SigningScheme: Scheme;
    type Context: Epochable;
    type Block: Block;

    fn propose(&mut self, context: (E, Self::Context),
               ancestry: impl Ancestry<Self::Block>)
        -> impl Future<Output = Option<Self::Block>> + Send;

    fn verify(&mut self, context: (E, Self::Context),
              ancestry: impl Ancestry<Self::Block>)
        -> impl Future<Output = bool> + Send;
}
```

Two Rust patterns worth highlighting:

**`impl Future` returns.** Both methods return `impl Future<...> +
Send` rather than a concrete type. Why?

- **No exposed future type.** A complex verify might use
  `select!`, channels, or even store state in a future-local
  struct. Returning `impl Future` lets the implementer pick any
  combinator they want.
- **Static dispatch.** No vtable, no heap allocation for the
  future itself (the trait returns whatever it wants, sized at
  compile time).
- **Send + 'static.** The future is `Send + 'static` because
  Marshal needs to spawn it on its actor task. The bound is in
  the trait return type, not the implementor — neat.

Without `impl Future`, you'd write `Box<dyn Future<...> + Send>`
which loses the static-dispatch advantage.

**`oneshot::channel` for propose / verify.** Why does propose
return `Option<Block>` wrapped in a future? Because the propose
decision is *non-trivial*:

- The application might need to fetch missing transactions.
- It might need to wait for a previous proposal to finish.
- It might be unable to propose (still syncing — return `None`).
- It might be ready immediately.

`oneshot::channel` with `Sender<Option<Block>>` lets the
application close the channel at the right time:

```rust
fn propose(&mut self, ctx: ..., ancestry: ...) -> impl Future<...> {
    let (tx, rx) = oneshot::channel();
    // Spawn async work:
    self.spawner.spawn("propose", async move {
        let block = self.build_block(ctx, ancestry).await;
        tx.send(block);  // None = can't propose
    });
    rx
}
```

Marshal awaits `rx`. If it resolves to `Some(block)`, great. If
`None`, Marshal nullifies the view (chapter 11 main text).

For `verify`, the pattern is the same shape, but the answer is
`bool`. Marshal awaits a future that yields the verdict. If
`true`, vote to notarize. If `false`, treat as invalid and
nullify (chapter 11's "Treat local verification failure as
immediate timeout").

The `oneshot::Receiver<bool>` (or `Option<Block>`) pattern is
used throughout Commonware's actor interfaces — anywhere a
caller wants async, possibly-deferred, single-shot results.

---


## Deepening 1 — Block propagation strategies in depth

Marshal accepts blocks at the protocol layer; how those blocks move
across the network is a separate question. There are three families
of strategies, each with different cost profiles.

### Push (also called "broadcast-first")

The producer immediately broadcasts the full block to all peers.
This is the "gossip everything" approach used by Bitcoin and
Ethereum's older designs.

Cost:
- **Producer egress: O(N × B)** — one outbound copy per peer, per
  block. For N=100 and B=1MB, that's 100 MB egress per block.
- **Total bandwidth: O(N² × B)** — every peer re-broadcasts to its
  N-1 neighbors, so each block traverses the network multiple times.
  With deduplication, effective bandwidth is closer to O(N × B) at
  each peer.
- **Latency: 1 hop** — once the producer broadcasts, every peer has
  the block within one round trip.

The unique benefit: peers have the block *before* seeing any
certificate. By the time the certificate arrives, peers are ready to
vote `finalize` immediately.

### Pull (also called "fetch-on-demand")

The producer broadcasts only the digest (a 32-byte commitment).
Peers who learn the digest fetch the block from any peer who has it.

Cost:
- **Producer egress: O(B)** — broadcasts the block to one peer (the
  first requester), or uses coding to send shards.
- **Per-peer egress: O(B)** — each peer serves the block to a few
  others.
- **Total bandwidth: O(N × B)** at each peer, distributed.
- **Latency: 1 + extra hop** — peers need an additional round trip
  to fetch the block after seeing the digest.

Simplex uses pull (chapter 11 main text, also chapter 12 "Backfill").
The benefit is bounded producer egress, important for big blocks.

### Hybrid

Many systems combine both. Commonware's `broadcast::buffered::Engine`
(chapter 09) is a hybrid: it uses codec-sharded flooding for
small-to-medium data and pull-on-failure for large data.

Simplex's recommendation for `Relay` (chapter 11 main text on the
`Relay` trait) uses this hybrid: small block metadata is push (the
digest and notarization message), the block bytes are pull from a
peer's broadcast cache.

### Decision matrix

| Block size | Push cost | Pull cost | Best |
|---|---|---|---|
| < 100 KB | O(N×100KB) ~ 10 MB | O(100KB) | Pull |
| 100 KB - 1 MB | 100 MB egress | 1 MB egress | Pull |
| 1 MB - 10 MB | 1 GB egress | 10 MB egress | Pull + Coding |
| > 10 MB | prohibitive | 100 MB egress | Coding |

For all reasonable block sizes, pull (with coding for very large) beats
push. The "leader bandwidth" bottleneck that affects PoW systems is
eliminated.

### The leader's egress budget

In practice, a single producer's uplink caps at some byte/sec. For
1 Gbps uplink: ~125 MB/s. For 100 peers and 1 MB blocks, push
saturates the uplink at < 1 block/sec. With pull, the leader sends
the block once and peers fan out — the leader's egress is bounded by
peer *requests*, not by N.

The pull-based architecture is what makes Simplex viable at scale.
A 100-validator, 10 MB block deployment has the leader sending ~10 MB
per block (~80 ms at 1 Gbps), not 100 × 10 MB = 1 GB.

### Implementation in Commonware

The flow:

1. Leader `propose()` returns a digest.
2. Voter broadcasts `notarize(digest, view)` over P2P.
3. Peers receive the `notarize`, add it to votes (chapter 11).
4. Once 2f+1 votes accumulate, `Notarization(digest, view)` is formed.
5. Peers see the digest on the certificate.
6. Peers `resolver::fetch(digest)` to get the block bytes.
7. The resolver asks the `broadcast::buffered::Engine` for the bytes.
8. Once peers have the bytes, they `Automaton::verify` and `Automaton::certify`.

The two-round-trip structure (digest, then bytes) is what pull
imposes. The benefit: bounded producer egress.

---

## Deepening 2 — Data availability vs validity

These two concepts are **independent** and commonly confused. A
block can be available (peers have the bytes) but invalid (state
transitions break invariants). Or available but unvalidatable (peers
have the bytes but the application can't process them without more
data, e.g., a missing parent).

### Data availability (DA)

DA is the question: "Can I retrieve the bytes of the block if I
need them?" If yes, the block is **available** (peers can fetch it
on demand). If no (peers have only the digest, not the bytes), the
block is **not available**, and you cannot process it.

DA is **physical**: it's about whether the bytes exist on disk in
peers' storage. A DA check asks "do I know that enough peers hold
the bytes that I can fetch one?"

The classical "DA problem" in distributed ledgers:

- A leader broadcasts a notarization for digest D.
- D's bytes are available only on the leader's machine.
- Peers cannot fetch D because no peer has it.
- The block is notarized but not "really" in the chain.

DA primitives defend against this.

### Validity

Validity is the question: "Are the bytes correct according to the
protocol's rules?" For a block:

- Signatures are correct.
- State transitions are well-formed.
- The state root matches.
- No double-spend, no invalid transactions.

Validity is **logical**: it's about whether the bytes conform to the
protocol's semantics.

### ZODA: a DA primitive

`commonware-coding` has ZODA, the **Zero-overhead Data Availability**
primitive (chapter 07). ZODA encodes a block into `n` shards such
that:

- Any `k` shards reconstruct the original.
- ZODA's security argument: to *prevent* reconstruction, the leader
  must withhold shards from at least `n - k + 1` peers. The leader
  has to do this deliberately; it's costly in network bandwidth.
- The construction uses Reed-Solomon codes over a large Galois field
  (typically BLS12-381's scalar field).

The interaction with consensus:

1. Leader encodes block B into `n` shards.
2. Leader sends strong shards to `n - k + 1` peers.
3. Recipients broadcast weak shards (gossip).
4. After enough shards accumulate at any validator, reconstruction
  is possible.
5. The reconstructed B is verified by the application.

The DA check is implicit: if you can get `k` shards from peers, the
data is available. The leader's attempt to withhold is detectable
because the protocol designed for `n - k + 1` redundancy.

### The DA problem in detail (for the curious)

The classical DA problem (in nakamoto consensus, Ethereum 2.0
research): how do *light clients* verify a block is available without
downloading the whole block?

The answer: a light client samples a few random shards and verifies
they're consistent. With high probability, if the leader withheld
data, the light client will detect inconsistency across samples.

ZODA is a full-data primitive (designed for full nodes, not light
clients), but the same principles apply: erasure-coded redundancy
makes withholding detectable.

### Where DA meets consensus

In Simplex + Marshal:

1. The Voter votes to notarize a *digest*, not the bytes.
2. The Notarization certificate says "2f+1 saw this digest."
3. The Marshal actor uses the digest to fetch the bytes via
   broadcast or coding.

If the leader withholds the bytes (DA failure): the Notarization
forms, but peers can't verify or certify. They `nullify` because
`verify` doesn't return. The view times out.

If the bytes are available but invalid (validity failure): peers
verify and reject. They `nullify` because `verify` returns false.

Both are caught at the Marshal + Voter boundary.

### The interplay with stateless clients

A stateless client (chapter 17) only cares about the digest and the
validity proof (e.g., a Merkle proof of a state transition). It
doesn't care about DA — the stateless client's view is consistent
because it trusts the consensus to decide what bytes are "in" the
chain. DA matters only to *full* nodes who need to execute the
transactions.

Simplex's protocol is DA-agnostic: it commits to digests only, lets
DA be a downstream concern.

---

## Deepening 3 — State sync vs block sync

Joining a chain late has two paths: replay every block (`block
sync`), or download just the final state (`state sync`).

### Block sync

Download blocks from a known starting height (or genesis) to the
current tip. For each block, replay the state transitions, ending
up at the current state.

Cost:
- **Time**: O(N × B) where N = number of blocks, B = block size.
  For Alto's testnet with 1,000,000 blocks at 100 KB each, that's
  100 GB of data.
- **Disk**: same as time-to-download. Plus working disk for
  intermediate state.
- **CPU**: O(N × T) where T = transactions per block × verification
  cost. 1,000,000 blocks × 1000 tx/block × 1 µs/tx = 1000 seconds
  of CPU.
- **Trust**: zero. You replay; you verify every transition.

### State sync

Download a recent finalized *state root* (Merkle root of the
state) plus the application-defined *state bytes*. Verify the state
root against the consensus's certificate.

Cost:
- **Time**: O(B) where B = state size. For Alto's state of 50 GB,
  that's a 50 GB download.
- **Disk**: state + recent blocks (for state root verification).
- **CPU**: O(1) per block (just verify the certificate against the
  state root).
- **Trust**: trust the certificate you downloaded. If the
  certificate is valid, the state root is valid; the state bytes
  are self-verifying.

The trust model is the same as the rest of consensus: the
certificate is the consensus-level proof. State sync is just a
different way to use that proof.

### Trust and verification

State sync's trust model is subtle. You're trusting:

1. The certificate's validity (verifiable by signature check).
2. The state's correctness *relative to the certificate's height*.
3. The application's logic for serializing state into bytes and
   recomputing the state root.

If the application correctly serializes its state, the state bytes
reconstruct the same state root. If the application is buggy
(serializes non-canonically), the state may not match even though the
state root is valid.

### Trade-offs in Commonware

| Property | Block sync | State sync |
|---|---|---|
| Time to catch up | O(N × B) | O(state) |
| Trust model | Replay = verify | Trust certificate |
| Disk | Full block history | State + recent blocks |
| Computation | O(transactions) | O(serialize/deserialize) |
| App support required | Always works | App must support snapshotting |

The Marshal API exposes both. Use `Start::Height(h, finalization)` for
block sync from a known floor; use `Start::Height(h, finalization)`
plus a `set_state(state_bytes, state_root)` call for state sync.

### Why state sync is faster in practice

For a 6-month-old chain with 100 GB of state, block sync might take
10 hours (1 GB/s = 100 GB / 10 hours). State sync might take 30
minutes (if you trust the certificate and the application supports
it).

The trust trade-off is what makes state sync controversial. The
certificate proves that the consensus agreed on the state root; the
state bytes must match that root. There's no replay-based
verification — you're trusting the prior consensus.

### Failure modes for state sync

1. **State root mismatch**: the application computes the state root
   from the bytes and gets a different value. Fail the state sync,
   fall back to block sync.
2. **Application doesn't support snapshotting**: state sync is
   unavailable for that application.
3. **No recent certificate available**: the joining validator needs
   to find a peer willing to provide both the certificate and the
   state bytes.

Marshal handles failure mode (3) by waiting — if no peer has the
state, you can't state-sync. Block sync always works as a fallback.

### When to use which

| Scenario | Best |
|---|---|
| Joining a new chain for the first time | State sync (fast) |
| Recovering from a crash | Block sync from last-known floor |
| Testing state sync correctness | State sync first, then verify via block sync once caught up |
| Resource-constrained device | State sync (less CPU) |
| Strict verification required | Block sync (zero trust) |

In production chains, the default is state sync for new validators,
with block sync reserved for crash recovery.

### The "state root" trust argument

A final note: state sync's trust isn't "trust the joining peer."
It's "trust the consensus that produced this certificate." That's
the same trust as block sync (you trust the certificate too; you
just *also* replay to verify). State sync skips the replay but
holds the certificate's signature check.

The trust *delta* between block sync and state sync is the replay.
Replay is what makes block sync safe against a malicious chain; the
replay detects any inconsistency. State sync's equivalent of "detect
inconsistency" is "the certificate is signed by 2f+1 honest
validators, so the state root is the consensus's choice" — that
requires the joining validator to trust the consensus's honesty.

If the consensus was honest (which is what you sign up for by
joining), state sync is safe. If the consensus was dishonest, no
matter how you join, you have a problem.

---


## Deepening 4 — Pruning strategies in depth

The `Blocks::prune(below)` call discards old blocks. Three strategies
the application can implement:

### Depth-based pruning

Keep the last `D` blocks (e.g., `D = 1000`); prune the rest. Simple
implementation: a counter that increments on each `put`.

```rust
async fn put(&mut self, h: Height, b: Block) -> Result<()> {
    self.blocks.insert(h, b);
    if h > self.max_kept {
        let to_prune = h - self.max_kept;
        for old_h in self.blocks.keys().take_while(|&&k| k < to_prune).cloned().collect::<Vec<_>>() {
            self.blocks.remove(&old_h);
        }
        self.max_kept = h;
    }
    Ok(())
}
```

Cost: O(blocks pruned) per `put`. With `D = 1000`, that's 1000
removals per block put — fast.

Operational tuning: how big is `D`? Should be at least the depth
your application needs for reorgs (5-10 blocks typical) plus some
buffer for state rollback (a few hundred blocks for fork recovery).

### Cost-based pruning

Keep blocks until disk pressure exceeds threshold; prune the oldest
first. Requires a `Storage::available()` check.

```rust
async fn maybe_prune(&mut self) -> Result<()> {
    let available = self.disk.available_bytes().await?;
    if available < self.config.min_free_bytes {
        // Prune oldest blocks first
        let oldest = self.blocks.keys().next().cloned();
        if let Some(h) = oldest {
            self.blocks.remove(&h);
            self.disk.sync().await?;
        }
    }
    Ok(())
}
```

Advantage: adapts to deployment. Disadvantage: prune timing is
non-deterministic, which makes testing harder.

### Explicit floor

The application specifies a floor and Marshal honors it. Useful for
"we only need blocks after height 5000."

```rust
let cfg = marshal::Config {
    start: marshal::Start::Height(5000, finalization_at_5000),
    prune_below: 5000,
    ...
};
```

This is essentially the depth strategy with the depth set
externally.

### The consensus floor

A separate setting in `simplex::Config::Floor`:

```rust
floor: simplex::config::Floor::Height(1000, notarization_at_1000)
```

This tells Simplex "trust the notarization at height 1000 as the
floor; don't try to verify earlier." Same effect as Marshal's start
but for consensus.

If `marshal::start >= simplex::floor`, block sync skips blocks
between the floors (the consensus has already accepted them; no need
to replay).

### The cost-of-keeping math

Each 1 KB block costs:

- 1 KB on disk.
- ~64 bytes in the index (height → digest mapping).
- ~64 bytes in the certificates (Notarization, Finalization).
- ~16 bytes/journal overhead for the WAL entry.

Total: ~1.2 KB per block.

For a chain with 1,000,000 blocks: ~1.2 GB. For 10,000,000: ~12 GB.
For a billion-block chain: ~1.2 TB.

Pruning is essential beyond the first million blocks for any
deployable system.

### Pruning safety: the hard invariant

**Never prune below the highest recorded certificate's parent.** If
you have a Finalization at height 1000, you must keep blocks up to
height 1000 (because a peer's request for block 1000 must succeed).

Marshal enforces this via the `prune(below)` method: the
implementation can refuse to prune if doing so would orphan
certificates.

If you're memory-constrained and must prune aggressively, you also
need to implement state sync to fill the gaps after pruning. Most
chains opt for "conservative pruning" — keep 1000-10000 blocks of
history — rather than aggressive pruning with state-sync fillin.

---

## Deepening 5 — Standard vs Coding modes in depth

`marshal::standard` and `marshal::coding` differ in how blocks move.
This section goes deeper on the trade-offs.

### Standard mode

Receives full blocks via `broadcast::buffered::Engine` (chapter 09).
Each peer caches recent blocks in a `HashMap<(Digest, BTreeMap<PK,
Bytes>)>` — actually a `VecDeque<Bytes>` per peer, with a bounded
size to prevent unbounded memory.

The flow:

```
Leader broadcasts block B (full bytes) via broadcast::buffered.
Peers cache B in their deques.
A peer needs block B for notarization or verification.
  -> asks Resolver for digest(B)
  -> Resolver asks broadcast::buffered::Engine
  -> Engine forwards the request to peers who have B cached
  -> First peer to respond wins; bytes returned
```

Cost per block delivery:
- Leader: 1 outbound per peer (× N).
- Each peer: 1 cache write; later, 1 cache read on request.
- For N peers: O(N) total bandwidth at the leader's egress,
  O(N) at each peer's egress (when they serve to others).

### Coding mode

Receives shards via ZODA. Each peer gets one *strong* shard
directly from the leader; the rest accumulate via gossip.

The flow:

```
Leader encodes B into n shards.
Leader sends shard 0 to peer 0, shard 1 to peer 1, ..., shard n-1 to peer n-1.
Leader broadcasts weak shards (hash-only "I have shard K" proofs).
Peers accumulate shards until they have k of them.
Reconstruction: k peers' shards -> original block.
```

Cost per block delivery:
- Leader: 1 outbound per peer (one shard each).
- Each peer: stores its shard; gathers more from gossip until k.
- Total bandwidth: O(N) at leader's egress (same as standard).
  But each peer's *egress* when serving shards is O(N/k) — much smaller
  in expectation than standard mode for large N.

### Size implications

A 10 MB block in standard mode: each peer's cache holds one 10 MB
copy. For 100 peers, that's 1 GB of cache total across the cluster.

A 10 MB block in coding mode (k=33, n=67): each peer's cache holds
one shard of ~150 KB, plus accumulated weak shards. For 100 peers,
that's ~15 MB total — 67× less.

For memory-constrained deployments, coding wins.

### Bandwidth implications

Standard mode: to serve the block to many peers, the leader's egress
is dominant. But coding mode distributes: each peer serves its
shard, so no single peer's egress becomes a bottleneck.

For a geo-distributed deployment, coding mode's distribution lets
peers in different regions serve each other (low latency) rather
than waiting for the leader's regional peers.

### When each is right

| Scenario | Best |
|---|---|
| Small blocks (< 100 KB) | Standard |
| Big blocks (> 1 MB) | Coding |
| Geo-distributed peers | Coding |
| Limited memory | Coding |
| Many peers (> 50) | Coding |
| Few peers (< 20) | Standard |
| Testing simplicity | Standard |

### The decision in source code

In `consensus/src/marshal/coding/` vs `consensus/src/marshal/standard/`:
the same `Actor` runs in both modes, but with different
sub-engines. The choice is at `marshal::Config::mode`:
`MarshalMode::Standard` or `MarshalMode::Coding`.

The default in `examples/alto` is Coding, reflecting the
production-deployment assumption that blocks will be moderately
sized (1-10 MB) and the network is geo-distributed.

### The unified Application interface

A key design: both modes implement the same `marshal::Application`
trait (chapter 12 main text + Appendix). The application's code
doesn't know or care which mode is selected.

This means a Standard-mode application can be switched to Coding
mode by changing one config field. No application code changes.

The only application-level concern: `Automaton::certify` returns
`false` if the data is unavailable. In Coding mode, this might mean
"shards haven't arrived yet" (transient) or "shards will never
arrive" (leader malfeasance). The application distinguishes
via the timeout (chapter 11, Appendix D).

---

## Deepening 6 — The Store interface in depth

Marshal's durable storage abstraction has two distinct traits:
`Certificates` for proofs, `Blocks` for content. This split
reflects different access patterns and lifetimes.

### `Certificates`

Stores notarizations and finalizations, indexed by height. The
purpose: prove "block at height H was finalized at view V."

```rust
#[async_trait]
trait Certificates {
    async fn put_notarization(&mut self, h: Height, cert: Notarization) -> Result<()>;
    async fn put_finalization(&mut self, h: Height, cert: Finalization) -> Result<()>;
    async fn get_notarization(&self, h: Height) -> Result<Option<Notarization>>;
    async fn get_finalization(&self, h: Height) -> Result<Option<Finalization>>;
    async fn last_finalized(&self) -> Result<Option<(Height, Finalization)>>;
    async fn prune(&mut self, below: Height) -> Result<()>;
}
```

Implementations:

- **In-memory** (`BTreeMap<Height, Cert>`): tests, validators with
  ephemeral storage.
- **Append-only archive** (`storage::archive::Archive`):
  durable, periodic compaction, fast for sequential reads.
- **Cloud storage** (S3, GCS): archivable, slow access, very
  large scale.

### `Blocks`

Stores block bytes, indexed by height OR by digest:

```rust
#[async_trait]
trait Blocks {
    async fn put(&mut self, h: Height, b: Block) -> Result<()>;
    async fn get(&self, h: Height) -> Result<Option<Block>>;
    async fn get_by_hash(&self, hash: Digest) -> Result<Option<Block>>;
    async fn prune(&mut self, below: Height) -> Result<()>;
}
```

The two access patterns matter:

- **By height**: sequential replay (state transitions in order).
- **By hash**: random fetch (cross-references, e.g., backfill).

Some implementations optimize one over the other. The test
implementation uses `BTreeMap<Height, Block>` + `BTreeMap<Digest,
Height>`. Production uses `storage::archive::Archive` with digest
indices.

### Why two traits, not one?

Because access patterns are different. Certificates:

- Append-heavy (one per vote/cert).
- Rarely read (only on verification, replay, or proof generation).
- Small (~1 KB each).
- Long-lived (kept forever for proof purposes).

Blocks:

- Receive-heavy (one per block).
- Often read (every verification, every backfill).
- Variable size (KB to MB).
- Prunable (after some depth).

A single store could implement both, but its APIs would have to
expose both access patterns, leading to confusion. Two traits make
the access pattern explicit at the type level.

### The storage layout in production

Commonware uses `storage::archive::Archive` for both. The archive is
a log-structured key-value store with:

- **Append-only writes**: amortized O(1) per write, fast fsync.
- **Periodic compaction**: rebuilds index, reclaims space.
- **Range reads**: efficient for sequential block replay.
- **Random reads**: indexed for fast lookup.

The archival format (chapter 16) stores the entries with their
metadata (height, key, etc.) plus a periodic index snapshot.

### The cost-benefit analysis

| Store type | Read latency | Disk cost | Crash recovery |
|---|---|---|---|
| In-memory BTreeMap | ns | high (per-entry malloc) | zero (lost on crash) |
| Append-only archive | µs | low (sequential) | fast (replay) |
| Cloud storage | 100 ms | very low | none (provider-managed) |

Production chains use append-only archives (RocksDB-like) for both.

---


## Deepening 7 — Backfill in depth

Marshal's backfill mechanism recovers missing blocks when the
consensus has finalized heights the local node hasn't seen. This
section walks through the full flow.

### When backfill triggers

Three scenarios:

1. **Joining late**: node comes up, has no blocks past height H,
   consensus has progressed past H+1.
2. **Behind on events**: node missed notarizations for some
   heights (e.g., network partition during those views).
3. **State sync succeeded**: node has state at height H, needs to
   fetch blocks from H+1 to current.

In all three, Marshal notices a gap: it has certificates at heights
{H-1, H+5} but not {H, H+1, H+2, H+3, H+4}.

### The gap detection algorithm

Marshal tracks `finalized_heights: BTreeMap<Height, Finalization>`.
On every new finalization, it checks for gaps:

```rust
async fn check_for_gap(&mut self) -> Option<Range<Height>> {
    let heights: Vec<_> = self.finalized_heights.keys().copied().collect();
    if heights.len() < 2 { return None; }
    for w in heights.windows(2) {
        if w[1].value() - w[0].value() > 1 {
            // Gap from w[0].value() + 1 to w[1].value() - 1
            return Some(w[0].value() + 1 .. w[1].value());
        }
    }
    None
}
```

A linear scan is fine because the number of heights is bounded (you
have the certificates, not unbounded).

### The fetch flow

For each missing height:

```rust
async fn backfill_height(&mut self, h: Height) -> Result<()> {
    // 1. Request the block via Resolver
    let key = MarshalKey::Height(h);
    let rx = self.resolver.fetch(key).await;

    // 2. Wait for delivery (with timeout)
    let block = timeout(self.config.backfill_timeout, rx).await??;

    // 3. Verify against the certificate
    let cert = self.finalized_heights.get(&h).expect("missing cert");
    if block.digest() != cert.payload {
        return Err(Error::BlockDigestMismatch(h));
    }

    // 4. Store
    self.blocks.put(h, block.clone()).await?;

    // 5. Hand to application for processing
    self.apply_block(block).await?;

    Ok(())
}
```

The `Resolver::fetch` returns `oneshot::Receiver<Block>`. The peer
is selected by the Resolver's mechanism (chapter 08), which rates
limits to avoid overwhelming any one peer.

### The bounded in-flight window

Marshal doesn't fetch all missing heights at once (network implosion).
It bounds the in-flight fetches:

```rust
const MAX_INFLIGHT_BACKFILLS: usize = 64;

async fn process_backfills(&mut self) -> Result<()> {
    while self.inflight_backfills.len() < MAX_INFLIGHT_BACKFILLS
          && self.has_more_to_backfill()
    {
        let next_h = self.next_backfill_height().unwrap();
        let rx = self.backfill_height(next_h).await?;
        self.inflight_backfills.insert(next_h, rx);
    }
    Ok(())
}
```

This caps at 64 concurrent fetches; once any completes, another is
launched. The window size is configurable in `marshal::Config`.

### The interactions with Simplex's recovery

While Marshal is fetching missing blocks, Simplex's Voter is also
recovering (chapter 11). The two are independent:

- **Voter's recovery**: ensures the consensus state is correct
  (votes, certificates, current view).
- **Marshal's backfill**: ensures the block data is present locally.

A node can finalize a block whose bytes it doesn't have locally
(certificate comes first, bytes later). Marshal ensures the bytes
arrive before the application needs them for processing.

### Failure modes and recovery

Backfill can fail for several reasons:

1. **No peer has the block**: every peer responds with "not found."
   This shouldn't happen in a healthy cluster (the leader
   definitely had it; some peer should have cached it). If it does,
   the node fails the backfill and waits longer.
2. **Peer times out**: the peer was slow or offline. Try another peer.
3. **Block doesn't match certificate**: data corruption or
   Byzantine peer. Mark the peer for slashing (chapter 05),
   try another.

Each failure mode triggers a different retry strategy. The `Resolver`
handles retries internally.

### Backfill traffic shaping

Under high backfill pressure (large gap), Marshal should not
starve the consensus traffic. The implementation typically
prioritizes consensus over backfill:

```rust
let backfill_budget = if self.consensus_busy() {
    16   // smaller window when consensus is busy
} else {
    64   // larger window when consensus is idle
};
```

This adaptive shaping prevents a huge backfill from blocking
consensus progress.

### The backfill completion invariant

Marshal's correctness property: **once backfill completes, the node's
local chain is a prefix of the consensus's chain.**

Proof: the certificates Marshal holds are signed by 2f+1 honest
validators (or f Byzantine; either way the certificates are valid).
The blocks Marshal fetches have digests matching the certificates'
payloads (verified on fetch). Therefore the local chain is byte-for-byte
equal to the consensus's chain up to the highest fetched height.

This property is what makes Marshal safe for state sync: the node can
trust its local view matches the consensus's view.

---

## Deepening 8 — Start mechanism in depth

`marshal::Start` is the lever for joining mid-chain. Three variants:

### `Start::Default`

Start from genesis (or from the consensus's `Floor`). The node
processes every block from the beginning.

Use case: testing, fresh deployments.

### `Start::Height(h, finalization)`

Start at a specific height. The `finalization` is the cryptographic
proof that the consensus agreed at height h.

Use case: late joiners, state sync entry point.

### `Start::Genesis`

Like `Default` but explicit. Sometimes useful for clarity.

### The `SyncPlan` lifecycle

When `Start::Height(h, f)` is given, Marshal generates an internal
`SyncPlan`:

```rust
struct SyncPlan {
    target_height: Height,
    target_finalization: Finalization,
    start_height: Height,
    start_finalization: Finalization,
    backfill_window: Range<Height>,
}
```

The plan says: "bring me from `start_height` to `target_height` by
fetching the blocks in `backfill_window`."

### The interaction with the application

When `Start::Height` is given, Marshal calls the application *before*
fetching any block:

```rust
async fn start_with_height(&mut self, h: Height, f: Finalization) {
    // 1. Tell the application about the floor
    self.application.on_start(h, f.clone()).await;

    // 2. Begin backfill from h+1 onwards
    self.start_backfill(h + 1).await;

    // 3. Resume normal consensus participation
}
```

The application's `on_start` (if defined) can do state-sync setup,
log the new floor, etc.

### Why the application must know the floor

If the application's state machine processes blocks sequentially
(from genesis), starting at height H requires either:

1. Replay blocks 1..H to get to H's state.
2. Trust the state root at H.

Option 2 = state sync. Marshal's `Start::Height` doesn't tell the
application which to pick; the application decides in `on_start`.

The recommended pattern:

```rust
async fn on_start(&mut self, h: Height, f: Finalization) {
    if self.has_state_at_height(h) {
        // State sync — trust the consensus's state root, load bytes
        self.load_state().await?;
    } else {
        // Block sync — replay from genesis
        // (Marshal's `Start::Default` or fetched-through approach)
    }
}
```

### The "start from genesis" path

If the application picks block sync from genesis, Marshal's `Start::Default`
kicks in. The node fetches the genesis finalization from peers,
validates it, then begins replay.

This works automatically — no special application code needed.

### The "resume from local state" path

If the application has its own durable state (chapter 17
`metastore`), it can resume without any consensus at all. Marshal's
`Start::Height(h, ...)` provides the cryptographic anchor (the
finalization at h); the application loads its state from its
metastore.

### The bootstrap sequence

For a new validator:

```
1. Network discovery (find peers).
2. Get the latest finalization cert from any peer.
3. Decide: trust the state at that height (state sync) or replay
   from genesis (block sync)?
4. If state sync:
   a. Download the state bytes.
   b. Verify against the finalization's state root.
   c. Initialize Marshal with Start::Height(h, cert).
5. If block sync:
   a. Start at genesis (Start::Default).
   b. Or start at a recent cert (Start::Height(h, cert)) and replay.
6. Begin participating in consensus.
```

The cost-benefit: state sync is faster but trusts consensus; block
sync is slower but trustless.

### Trust boundaries

`Start::Height(h, cert)` requires trusting the certificate. The
"trust" here is "trust the consensus has produced a valid certificate
for h." That's the same trust as participating in consensus; there's
no extra trust.

If the certificate is invalid (signatures don't verify, the 2f+1
quorum isn't met), Marshal refuses to start. The application's
`Start::Height` is gated on certificate validity.

---

## Deepening 9 — Finalization acknowledgment in depth

Marshal delivers finalized blocks to the application via the
`Reporter` trait. The application's `Ack` signal tells Marshal when
the application is done with the block.

### The flow

```rust
async fn marshal_main_loop(&mut self) {
    loop {
        // 1. Decide which block to deliver next
        let next_h = self.next_block_to_deliver().await;

        // 2. Fetch and validate the block
        let block = self.blocks.get(next_h).await?.expect("gap");
        let cert = self.finalized_heights.get(&next_h).unwrap();

        // 3. Hand to the application via Reporter
        let (activity, ack_rx) = Reporter::activity_for(block.clone());
        self.reporter.report(activity).await;
        let ack = ack_rx.await.expect("application must ack");

        // 4. Application processed; advance to next block
        self.last_delivered = next_h;
        self.delivered_heights.insert(next_h, ack);
    }
}
```

The `activity_for` returns a tuple `(activity, oneshot::Receiver)`.
The application processes the block, then signals the receiver.

### Why acks matter

Without acks:

- Marshal doesn't know when it's safe to advance to the next block.
- The application might not have written the state changes to disk.
- Crashes between "Marshal delivered" and "application wrote" cause
  inconsistency on restart.

With acks:

- Marshal knows the application has fully processed the block.
- Crashes can be cleanly recovered (re-deliver the same block).
- The application's progress is observably bounded by Marshal.

### The at-least-once delivery guarantee

Marshal delivers each block *at least once*. The application may
see duplicate deliveries (e.g., Marshal crashes after delivery but
before advancing `last_delivered`). The application's idempotency
check handles this.

The idempotency check is typically:

```rust
async fn on_block(&mut self, block: Block) {
    if self.applied_heights.contains(&block.height()) {
        return; // Already processed; skip
    }
    self.apply_transactions(&block).await?;
    self.applied_heights.insert(block.height());
    // Acknowledge...
}
```

The `applied_heights` set tracks what's done. On restart, this is
durable (saved in the application's own state).

### The back-pressure

If the application is slow, Marshal naturally slows down. The
`oneshot::Receiver` blocks Marshal's main loop until the application
acks.

This is **good** for consistency but **bad** for throughput if the
application is much slower than consensus. A solution: parallel
processing of multiple blocks in the application (Marshal delivers
in order, but the application can pipeline state writes).

In practice, application-level pipelining is rare; it's more common
to keep Marshal's throughput bounded by the application's per-block
processing time.

### The shutdown handling

When Marshal is shutting down, it stops fetching new blocks but
still waits for in-flight acks. The shutdown sequence:

```rust
async fn shutdown(&mut self) {
    // 1. Stop accepting new finalizations from consensus
    self.consensus_sender.cancel().await;

    // 2. Wait for in-flight block deliveries to ack
    self.wait_for_pending_acks().await;

    // 3. Persist state (last_delivered, applied_heights)
    self.persist_state().await?;

    // 4. Clean shutdown
}
```

A clean shutdown prevents the application from having "unfinished"
block processing that's lost on next restart.

### The `Ack` as a Rust pattern

The `Ack` is a `oneshot::Sender<()>`, possibly wrapped in `Result`.
The pairing with `oneshot::Receiver` is the Rust convention for
one-shot async handshakes. Commonware uses this in many places
(not just Marshal):

- `Marshal -> Application`: block delivery (this section).
- `Voter -> Application`: verify / certify verdicts (chapter 11).
- `Resolver -> Voter`: block fetched (chapter 08).

The pattern: producer returns `Receiver<T>`; consumer returns
`Sender<T>` after completing work.

---

## Exercises — Marshal

A set of exercises covering Marshal's deeper behaviors.

### Exercise 12.1 — Implement the gap-detection

Write the gap-detection algorithm in Rust:

```rust
fn detect_gaps(finalized: &BTreeMap<Height, Finalization>) -> Vec<Range<Height>> {
    let heights: Vec<Height> = finalized.keys().copied().collect();
    let mut gaps = Vec::new();
    for w in heights.windows(2) {
        let gap_start = w[0].value() + 1;
        let gap_end = w[1].value() - 1;
        if gap_start <= gap_end {
            gaps.push(gap_start..gap_end + 1);
        }
    }
    gaps
}
```

Test: with `finalized = {1, 2, 5, 6, 9}`, the result should be
`[3..4, 7..8]`.

### Exercise 12.2 — Backfill traffic budget

For a 10-block gap with `MAX_INFLIGHT_BACKFILLS = 64`, how many
parallel fetches occur at peak? Hint: not all 10 — Marshal
limits by the window.

For a 1000-block gap, what's the steady-state concurrency? The
traffic pattern? (A: 64 in flight until ~50% complete, then declining.)

### Exercise 12.3 — Marshal on the Alto deployment

Look at `examples/alto` (chapter 19, chapter 21). Find:

1. The `marshal::Config` (mode, peer, retry).
2. The application's `on_start` implementation.
3. The `Reporter` implementation.
4. The `MarshalMode` (Standard or Coding).

Why those choices? Document your reasoning.

### Exercise 12.4 — Demonstrate pull vs push bandwidth

For a deployment with 5 peers, 1 MB blocks, 100 blocks:

1. Push bandwidth: leader egress = 5 × 1 MB × 100 = 500 MB.
2. Pull bandwidth: leader egress = 1 MB × 100 = 100 MB.

Calculate the savings. How does it scale to 100 peers?

### Exercise 12.5 — DA primitive trade-offs

For a 10 MB block with `n = 100, k = 67`:

1. Coding-mode cache per peer: 10 MB / 67 ≈ 150 KB.
2. Standard-mode cache per peer: 10 MB.
3. Total cluster cache: coding ≈ 15 MB, standard ≈ 1 GB.

Why is coding so much more memory-efficient? What's the trade-off?

### Exercise 12.6 — Trace one finalization's flow

For one block at height H:

1. Consensus emits `Finalization(c, v)` to Marshal.
2. Marshal matches `c` to a stored block (or fetches it).
3. Marshal verifies `c == block.digest`.
4. Marshal emits `Update::Block(block, ack)` to the Reporter.
5. The application processes the block and acks.
6. Marshal advances.

What's the latency at each step? Where might back-pressure
emerge?

---


## Appendix A — The Marshal Actor, in detail

`consensus/src/marshal/core/`. The `Actor` is the unified orchestrator.

### State

```rust
struct MarshalActor<S, B, R, ...> {
    // Configuration
    config: Config,
    scheme: S,
    automaton: R,
    // ...

    // Storage
    blocks: store::Blocks<B>,           // blocks indexed by height
    certificates: store::Certificates, // notarizations + finalizations

    // In-memory state
    pending_blocks: BTreeMap<Digest, Block>,    // received but not yet matched
    pending_finalizations: BTreeMap<Height, Finalization>,
    finalized: BTreeMap<Height, (Block, Finalization)>,
    outstanding_fetches: BTreeSet<Digest>,
    // ...

    // Mailboxes (channels)
    mailbox: Mailbox<MarshalMessage>,
    resolver: ResolverHandle,
    broadcast: BroadcastHandle,
    // ...
}
```

This is a lot of state. Marshal tracks everything that's "in flight" —
blocks being fetched, certificates being assembled, finalizations
awaiting application processing.

### The main loop

```rust
async fn run(mut self) {
    loop {
        select! {
            // New notarization or finalization from consensus
            cert = self.consensus_receiver.recv() => {
                self.on_certificate(cert).await;
            }

            // New block from the broadcast layer
            block = self.broadcast_receiver.recv() => {
                self.on_block(block).await;
            }

            // Block delivery from the resolver
            delivery = self.resolver_receiver.recv() => {
                self.on_resolver_delivery(delivery).await;
            }

            // Application asking for the next block
            req = self.application_receiver.recv() => {
                self.on_application_request(req).await;
            }

            // Application finished processing a block
            ack = self.application_ack.recv() => {
                self.on_application_ack(ack).await;
            }

            // Shutdown
            _ = self.stop_signal => break,
        }
    }
}
```

Six branches, all non-blocking. The select! polls them all.

## Appendix B — Standard mode (full blocks)

`consensus/src/marshal/standard/`. Two variants:

### `Inline`

The application **embeds the consensus context** in the block:

```rust
struct Block {
    parent: Digest,
    payload: Bytes,           // application data
    consensus_context: ConsensusContext,  // view, height, etc.
}
```

When Marshal receives a notarization for hash H, the consensus context is
in the notarization. The block, once received, has a matching context.
Marshal verifies the match.

Pros: simple, one-pass verification.
Cons: changing consensus metadata requires a block format change.

### `Deferred`

The application **commits to the context via the block's digest**, but
the context is provided separately:

```rust
struct Block {
    parent: Digest,
    payload: Bytes,
}

struct BlockWithContext {
    block: Block,
    context: ConsensusContext,
}
```

Pros: block format can stay stable across consensus changes.
Cons: requires the application to remember the context.

### `Deferred` is more flexible

For applications that evolve (like Alto), `Deferred` is recommended.
Changing the consensus metadata doesn't require re-encoding all old
blocks.

## Appendix C — Coding mode (erasure-coded shards)

`consensus/src/marshal/coding/`. Uses ZODA (chapter 07) for dissemination.

### The shards Engine

`consensus/src/marshal/coding/shards/`. The shards engine wraps the
broadcast layer to add coding-specific logic:

```rust
struct ShardsEngine {
    // ...
    shard_stores: HashMap<Digest, ShardStore>,  // per-block shard store
    commitments: HashMap<Digest, ZodaCommitment>,
    // ...
}
```

When the leader proposes a block B:

1. Encode B into shards (chapter 07).
2. Send one strong shard to each validator via the shards engine.
3. Validators weaken their shard and gossip weak shards to each other.
4. Each validator reconstructs B from k shards.

### The certification

When a notarization forms, validators check: "can I reconstruct the
block behind this notarization?" The `CertifiableAutomaton::certify`
hook (chapter 11) is where this happens in Simplex. Marshal's coding
mode coordinates with that hook.

## Appendix D — The Store interface

`consensus/src/marshal/store.rs`. Two abstractions:

### `Certificates`

Stores notarizations and finalizations durably:

```rust
trait Certificates {
    async fn put_notarization(&mut self, height: Height, cert: Notarization);
    async fn put_finalization(&mut self, height: Height, cert: Finalization);
    async fn get_notarization(&self, height: Height) -> Option<Notarization>;
    async fn get_finalization(&self, height: Height) -> Option<Finalization>;
    async fn last_finalized(&self) -> Option<(Height, Finalization)>;
}
```

Implemented via `storage::archive` (chapter 06) on disk, or in-memory for
tests.

### `Blocks`

Stores the actual block data:

```rust
trait Blocks<B: Block> {
    async fn put(&mut self, height: Height, block: B) -> Result<()>;
    async fn get(&self, height: Height) -> Result<Option<B>>;
    async fn get_by_hash(&self, hash: Digest) -> Result<Option<B>>;
    async fn prune(&mut self, below: Height) -> Result<()>;
}
```

`prune` deletes blocks below the given height. Used for state sync
startup: once you have a finalized floor, you can prune everything below.

## Appendix E — The ancestry walking

`consensus/src/marshal/ancestry.rs`. The `Ancestry` trait:

```rust
pub trait Ancestry<B: Block> {
    type Item;
    fn parent(&self) -> Option<Self::Item>;
}
```

Provides iteration from a block backward through its ancestors:

```rust
impl<B: Block> Ancestry<B> for B {
    type Item = (B, Finalization);
    fn parent(&self) -> Option<Self::Item> {
        // Look up parent's finalization
        // Return block + finalization for parent
    }
}
```

Used by `Application::verify` (chapter 11) to walk the chain backward,
validating each block against its ancestry.

## Appendix F — The Backfill machinery

When Marshal notices a gap (e.g., it has finalization for height 100
and 200, but not 101-199), it requests backfill:

```rust
fn check_for_gap(&mut self) {
    let last_finalized = self.finalized.keys().max().copied();
    if let Some(last) = last_finalized {
        for height in (self.floor + 1)..last {
            if !self.finalized.contains_key(&height) {
                self.request_backfill(height);
            }
        }
    }
}
```

Backfill uses the resolver (chapter 08):

```rust
fn request_backfill(&mut self, height: Height) {
    let key = BackfillKey::Height(height);
    self.resolver.fetch(key);
}
```

Peers respond with the missing blocks (and their finalizations).

## Appendix G — The `Start` mechanism

`consensus/src/marshal/config.rs::Start`. Allows starting Marshal at a
specific height, not genesis:

```rust
pub enum Start {
    /// Start from genesis (or wherever Marshal was previously).
    Default,
    /// Start from a specific finalized block.
    Height(Height, Finalization),
}
```

When using `Start::Height`, Marshal:

1. Initializes its state at the given height.
2. Fetches the block for that height via the resolver.
3. Begins processing from `height + 1` onward.
4. Refuses to backfill blocks below the starting height.

This is the **light client / late joiner** pattern. New validators
don't need to replay from genesis.

## Appendix H — The finalization acknowledgment

After Marshal delivers a finalized block to the application, it waits for
acknowledgment:

```rust
async fn on_finalization(&mut self, height: Height, block: Block, cert: Finalization) {
    // Send to application
    let ack = self.application_tx.send((height, block, cert));

    // Don't acknowledge to consensus yet — wait for application
    self.pending_acks.insert(height, ack);
}

async fn on_application_ack(&mut self, height: Height) {
    // Application finished processing — now acknowledge to consensus
    self.pending_acks.remove(&height);
    self.ack_consensus(height).await;
}
```

The application processes the block (executes transactions, updates
state, etc.) and then signals "done." Marshal then acknowledges to
consensus (via Reporter or similar).

If the application crashes between block delivery and acknowledgment,
Marshal will re-deliver on restart. **At-least-once** semantics — the
application must handle duplicates.

## Appendix I — The reporter integration

Marshal reports to the application's Reporter (chapter 11):

```rust
impl Reporter for MarshalReporter {
    type Activity = FinalizedBlock;
    fn report(&mut self, activity: Self::Activity) -> Feedback {
        // Pass to the application's Reporter chain
        self.inner.report(activity)
    }
}
```

The application's Reporter is the **single observation point** for
"block X was finalized." Use it for metrics, slashing, syncing
downstream systems.

## Appendix J — Test patterns

```rust
fn test_block_delivery() {
    // Setup: 4 validators, Marshal wired up
    // Block N is proposed, notarized, finalized
    // Verify: each validator receives the block
    // Verify: each validator processes it in order
    // Verify: each validator acknowledges
}

fn test_backfill() {
    // Setup: 4 validators
    // Validator A misses blocks 50-100 (simulated downtime)
    // Validator A restarts, sees finalization for 200
    // Verify: A requests backfill for 50-199
    // Verify: A receives blocks 50-199
    // Verify: A processes them in order
}

fn test_standard_vs_coding() {
    // Run same scenario with Standard and Coding modes
    // Verify: same finalization order
    // Verify: same application-visible behavior
}
```

## Appendix K — Common gotchas

### Not handling duplicate block delivery

Marshal delivers at-least-once. If your application isn't idempotent,
you'll double-execute. Make sure state transitions are idempotent
(transaction nonces, etc.).

### Forgetting to wait for `ack`

If Marshal doesn't wait for your application's ack, it'll fall behind.
Always wait for the ack before processing the next block.

### Mismatched digest types

If your `Block::Digest` type doesn't match the consensus scheme's
digest type, you get cryptic errors. Use the scheme's digest type
throughout.

## Where to look in the code (expanded)

- `consensus/src/marshal/mod.rs:1-65` — the architecture.
- `consensus/src/marshal/core/` — the unified actor.
- `consensus/src/marshal/standard/` — full-block mode.
- `consensus/src/marshal/coding/` — erasure-coded mode.
- `consensus/src/marshal/store.rs` — the storage interface.
- `consensus/src/marshal/ancestry.rs` — the parent-walking trait.
- `consensus/src/marshal/resolver/` — the backfill integration.
- `consensus/src/marshal/application/` — the ApplicationMode glue.
- `examples/bridge/` — an example using Marshal + Simplex.

## If you only remember three things

1. **Marshal = "consensus says H is finalized; here's the block for H."** It matches certificates to data.
2. **Two modes: standard (full blocks) and coding (shards).** Coding wins for big blocks.
3. **Backfill lets you catch up.** Marshal notices gaps and asks peers for missing blocks via the resolver.

→ Next: **Chapter 13 — Aggregation**. A different consensus-flavored
primitive: instead of ordering messages, it collects signatures to form
certificates over an externally-synchronized stream.