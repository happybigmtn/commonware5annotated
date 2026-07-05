# Chapter 14 — Ordered Broadcast: a "consensus-lite" for known sequencers

> When you want total order of messages from one sender, without full BFT overhead.

## The problem

Simplex is heavy. Aggregation is for external logs. What if you want
something in between — total order over messages from a **known
sequencer**, but you don't need to elect leaders dynamically or rotate
validator sets?

That's ordered broadcast.

## The concepts

`ordered_broadcast/mod.rs:5-19`:

> The system has two types of network participants: **sequencers** and
> **validators**. Their sets may overlap and are defined by the current
> `epoch`, a monotonically increasing integer.
>
> **Sequencers** broadcast data. The smallest unit of data is a `chunk`.
> Sequencers broadcast `node`s that contain a chunk and a certificate over
> the previous chunk, forming a linked chain of nodes from each sequencer.
>
> **Validators** verify and sign chunks. These signatures can be combined
> to form a quorum certificate, ensuring a quorum verifies each chunk.

So:

- A **sequencer** ships chunks, one at a time, each linked to the previous
  via a certificate.
- A **validator** signs chunks after verifying them.
- A **certificate** over a chunk proves "a quorum saw and verified this chunk."

This is the Autobahn pattern (paper cited in `mod.rs:51`):

> [Autobahn](https://arxiv.org/abs/2401.10369) provided the insight that a
> succinct proof-of-availability could be produced by linking sequencer
> broadcasts.

## The four schemes — same as before

`ordered_broadcast/mod.rs:21-37` — supports the same four signature schemes
from chapter 04 (ed25519, secp256r1, bls12381_multisig, bls12381_threshold).
Same trade-offs.

## Why this is "lighter" than Simplex

Simplex: dynamic leader election, view timeouts, certification, view
transitions, finalize/nullify dance.

Ordered broadcast: **a fixed sequencer broadcasts chunks in order**, each
one linked to the last. Validators sign. No leader election. No view
transitions. No nullifications.

If you trust the sequencer (or have external slashing for sequencer
misbehavior), you get total order without BFT overhead. This is what
makes it useful for "consensus-lite" deployments.

## Crash recovery — the journal

`ordered_broadcast/mod.rs:19-21`:

> Network participants persist any new nodes to a journal. This enables
> recovery from crashes and ensures that sequencers do not broadcast
> conflicting chunks and that validators do not sign them.

Same pattern as chapter 06. Journal everything; on restart, replay; never
sign two conflicting chunks.

## The Engine's responsibilities

`ordered_broadcast/mod.rs:41-45`:

> The core of the module is the Engine. It is responsible for:
> - Broadcasting nodes (if a sequencer)
> - Signing chunks (if a validator)
> - Tracking the latest chunk in each sequencer's chain
> - Assembling certificates from a quorum of signatures
> - Notifying other actors of new chunks and certificates

Five things. The flow for a validator:

```
Sequencer broadcasts Node { chunk, parent_cert } over P2P (chapter 05).
  │
  ▼
Validator receives Node, verifies parent_cert (from chapter 04).
  │
  ▼
Validator verifies chunk (application-defined logic).
  │
  ▼
If valid: Validator signs chunk and multicasts Ack.
  │
  ▼
Other validators receive Ack, verify, add to in-progress set for chunk.
  │
  ▼
Once 2f+1 Acks: form Certificate over chunk.
  │
  ▼
Notify application via Reporter. Sequencer can advance to next chunk.
```

The sequencer waits for the certificate on chunk N before broadcasting
chunk N+1. That's how the chain stays linked.

## The `tip_manager` and `ack_manager`

Two internal components:

- `tip_manager` — tracks the latest chunk per sequencer. Decides what
  counts as "the chain tip."
- `ack_manager` — tracks in-progress acknowledgments per chunk. Decides
  when a quorum has signed.

Both are pure data-structure work; no networking or crypto inside. Easy to
unit test.

## Why link chunks with certificates?

Without the linking, a sequencer could broadcast "chunk A" then "chunk A
again" with different contents (equivocation). Validators can't tell which
to certify.

With linking, each chunk references its parent's certificate. The
sequencer can broadcast two different chunks at the same height, but
each will reference different (or the same) parent cert. Validators
**only sign the first one they see** — once a chunk at height N is
signed, no other chunk at height N gets signed.

That's the equivocation prevention. Combined with the journal's
crash-safety, you get "a sequencer can equivocate once per crash, but the
crashed sequencer can't equivocate again on restart."

## Use cases

Where would you use this?

- **MEV-resistant transaction ordering.** A known sequencer (e.g., a
  threshold-encrypted mempool) broadcasts encrypted transactions in a
  fixed order; validators sign.
- **Cross-chain bridges.** The bridge operator is the sequencer; validators
  are the bridge committee.
- **Oracle data feeds.** A known data provider is the sequencer; validators
  sign each price update.

Anywhere you want **a single trusted (slashable) party's output to be
collectively attested**, ordered broadcast is the right tool.

## Where it lives in the stack

```
Sequencer's local data
        │
        ▼
Ordered Broadcast Engine (sequencer role)
        │
        ▼ broadcast Node
        │
   Validators' Ordered Broadcast Engines (validator role)
        │
        ▼ sign + multicast Ack
        │
        ▼ 2f+1 Acks → Certificate
        │
        ▼ Reporter → Application
```

## Where to look in the code

- `consensus/src/ordered_broadcast/mod.rs:1-52` — the architecture.
- `consensus/src/ordered_broadcast/engine.rs` — the main loop.
- `consensus/src/ordered_broadcast/ack_manager.rs` — tracking Acks.
- `consensus/src/ordered_broadcast/tip_manager.rs` — tracking tips.
- `consensus/src/ordered_broadcast/scheme.rs` — the signature schemes.

## Total order broadcast — the formal problem

The primitive in this chapter is, in the formal literature, called
**total order broadcast** (or atomic broadcast): every honest node
delivers every message in the same order.

Three equivalent definitions appear across the literature; Commonware's
implementation matches all three:

1. **Agreement**: if any honest node delivers message m at index i, then
   every honest node delivers m at index i.
2. **Total order**: if any honest node delivers message m1 before
   message m2, then every honest node delivers m1 before m2.
3. **Gap-free**: each honest node's delivered sequence is the same
   contiguous sequence — no skips, no duplicates.

The original formalization (Cristian, Aghili, Strong, 1985, "Atomic
Broadcast of Messages over Noisy Channels") introduced the *Uniform vs
Non-uniform* distinction: uniform guarantees hold even if Byzantine
nodes deliver messages; non-uniform only guarantees for honest nodes.
Commonware's ordered broadcast is uniform — once a chunk has been
certified, the certificate acts as proof *to anyone*, including the
Byzantine nodes.

### The FLP impossibility and partial synchrony

Dwork and Lynch's 1988 paper ("Consensus in the Presence of Partial
Synchrony") establishes the model this chapter lives in. The headline
result: in a fully asynchronous model with even one Byzantine node, no
deterministic protocol can guarantee agreement. This is the FLP
impossibility (Fischer, Lynch, Paterson, "Impossibility of Distributed
Consensus with One Faulty Process", 1985).

Ordered broadcast (and Simplex) bypass FLP with **partial synchrony**:
the system is asynchronous for some stretches (no guaranteed message
delay bound) and synchronous for others (with a known upper bound on
message delay that holds "eventually, after some Global Stabilization
Time `GST`"). The protocols in Commonware are designed to be **safe at
all times** (no two honest nodes ever disagree on the order) and
**live after GST** (the system keeps making progress once the network
behaves).

Ordered broadcast is **cheaper than Simplex** for two reasons:

1. The sequencer role is pre-assigned, so there's no leader election.
2. The "linked-chunk" structure provides the safety property directly:
   validators agree on chunk N+1's content by first agreeing on chunk
   N's certificate (and signing chunk N+1 only after seeing chunk N's
   certificate).

The "linked chunks" pattern is a form of **proof-of-publication** in
the data-availability sense: every node that signs chunk N+1 has
*seen* chunk N's certificate, hence can be told the contents of N if
needed.

## Sequencer designs: centralized, semi-centralized, decentralized

The "known sequencer" is a sliding scale.

| Design | Description | Trust assumption | Cost |
|---|---|---|---|
| **Single sequencer** | One well-known party broadcasts | Trust the sequencer (or trust external slashing) | Cheapest protocol |
| **Round-robin sequencer** | Sequencers take turns per epoch | Trust the union of all sequencers (no minority equivocates) | One rotation per epoch |
| **VRF-elected sequencer** | VRF picks a sequencer per round | Honest supermajority of stake/registration | One VRF per round |
| **PoW-style sequencer** | Whoever finds a nonce publishes | Honest majority of compute | Continuous hashing |

This chapter covers the **first** design — known-sequencer ordered
broadcast, with one designated sequencer per epoch. Commonware's
`ordered_broadcast` module does not include round-robin or VRF logic
because those are policy decisions the application should make. You
can layer them on: pick a sequencer per epoch (out of band), configure
that sequencer's pubkey in the engine's `Config`, and the rest of the
protocol is identical.

If the sequencer is **malicious**, two things can go wrong:

1. **Equivocation**: the sequencer broadcasts two different chunks at
   the same height. Commonware's journal-backed recovery (next
   section) plus the validator-side first-seen-wins rule prevents
   validators from signing both.
2. **Censorship**: the sequencer refuses to broadcast chunk N+1.
   Validators will see "no new chunk" — the chain stalls. Ordered
   broadcast has no built-in timeout-driven view-change. Applications
   that want censorship resistance must add an outer layer (e.g.,
   Timed-Rotating-Sequencer or fallback-to-VRF).

## DAG-based consensus: Narwhal, Bullshark, Tusk

A different answer to "total order over a stream" comes from Mysten
Labs (Sui): DAG-based consensus (Narwhal mempool + Bullshark / Tusk
consensus). Instead of a single sequencer broadcasting a chain,
everyone broadcasts "vertices" forming a DAG (directed acyclic graph);
each round advances; the consensus rule is "transitively finalize
vertices that have 2f+1 causal support."

The contrast:

| | Ordered broadcast (this chapter) | DAG-based consensus (Narwhal/Bullshark) |
|---|---|---|
| Producer role | Single sequencer | Every validator submits |
| Topology | Linked list (chain) | DAG |
| Bandwidth per validator | O(1) (chunks signed) | O(N) (causal references) |
| Finality latency | One-hop certification | ~3 rounds with Bullshark |
| Liveness under partial synchrony | Single sequencer can stall | Producers can take over for stuck validators |
| Implementation cost | ~2 KLOC | ~10+ KLOC |

Commonware's choice of "single sequencer + linked chain" is a
deliberate engineering simplification. If you need DAG semantics or if
no single sequencer is acceptable, you want a different library
(Sui, Aptos-BFT, Narwhal). Ordered broadcast is the right primitive
when **one trusted (slashable) sequencer** is acceptable.

## The Engine, in full

The ordered broadcast `Engine` runs once per validator. It has two
**concurrent roles** in the same loop:

```
Sequencer role          Validator role
--------------          ---------------
broadcast chunk N       receive Node
wait for cert on N      verify parent_cert + chunk
broadcast chunk N+1     sign + multicast Ack
```

A validator that is also the sequencer for some epochs plays both
roles; a validator that is only-a-validator plays just the second.

The lifecycle per epoch:

```
Epoch start:
  - Sequencer: load chain state from journal (last certified height)
  - Validator: load signed-by-me set from journal

Sequencer-side:
  - Application submits chunk via Sender<Chunk>
  - Engine wraps chunk into Node { chunk, parent_cert }
  - Sequencer signs Node and multicasts to peers
  - Sequencer waits for Certificate on chunk (via Reporter::Activity)

Validator-side:
  - Receive Node from network
  - Verify sequencer's signature on chunk
  - Verify chunk builds on parent's certificate (tip_manager check)
  - Look up (sequencer, height) in local signed-by-me set
  - If already-signed, reject (equivocation prevention)
  - Verify chunk's contents (application-defined logic)
  - Sign Ack and multicast
  - When 2f+1 acks arrive → assemble Certificate → emit to Reporter
```

The same engine struct plays both roles because the same `select!`
loop serves both: it has a channel from "the local chunk producer
application," a channel for "Node messages from peers," and a channel
for "Ack messages from peers." The current role is just metadata.

`consensus/src/ordered_broadcast/engine.rs::run` is the
`select!` loop; the per-role logic is in private methods `on_new_chunk`,
`on_node`, and `on_ack` (roughly).

## The `AckManager`, in detail

`consensus/src/ordered_broadcast/ack_manager.rs`. Tracks the
in-progress acks for each `(sequencer, height)` pair.

```rust
struct AckManager {
    pending: HashMap<(PK, Height), PendingAck>,
    acks: HashMap<(PK, Height), BTreeMap<PK, Ack>>,
}

struct PendingAck {
    chunk: Chunk,
    parent_cert: Certificate,
    started_at: SystemTime,
}
```

Methods:

- `add_ack(sequencer, height, ack)` — verify the signature, insert
  into the inner `BTreeMap`. If `len() >= QUORUM`, return
  `Some(Certificate)` and remove the pending entry.
- `prune_completed(sequencer, height)` — remove bookkeeping for a
  chunk that's been certified (cert was successfully emitted).
- `pending_count()` — for metrics, how many chunks are mid-ack.

The `BTreeMap<PK, Ack>` ordering matters: when assembling a
Certificate, the iteration is stable (sorted by validator pubkey).
BLS aggregation is order-sensitive — adding the same signatures in
different orders still produces the same aggregate (curve-point addition
is commutative), but **multisig-with-attribution** is not: a swap of
indices breaks signature verification.

## The `TipManager`, in detail

`consensus/src/ordered_broadcast/tip_manager.rs`. Tracks the latest
chunk per sequencer. The verification rule:

```
update_tip(sequencer, chunk, cert):
    current = tips.get(sequencer)
    if current is None:
        accept (this is the genesis chunk for this sequencer)
    else:
        if chunk.parent != current.cert.digest():
            reject (chunk does not extend the chain)
        else:
            accept
    tips.insert(sequencer, Tip { chunk, cert, height })
```

A chunk whose parent certificate digest doesn't match the current
tip's certificate digest is **rejected silently** (returning None and
incrementing a metric). It is not destructive; it just isn't adopted.

`tips: BTreeMap<PK, Tip>` is the per-sequencer map. The BTreeMap
ordering matters because `find_peaks` (chapter 16) walks it.

## Equivocation prevention

The cryptographic mechanism is the same journaled signature rule as
chapter 13's Aggregation, with one extra wrinkle from the chain
structure.

Three facts together prevent equivocation across crashes:

1. **The chain link.** Each node carries `parent_cert`. To broadcast
   two different chunks at height `N+1`, the Byzantine sequencer must
   reference the *same* parent (the certificate from height N). The
   two chunks are siblings, both extending a single canonical tip.

2. **First-seen-wins.** When the validator's engine sees chunk-A at
   height N+1, it signs and journals. When it later sees chunk-B at
   the same height, the journal lookup `signed.contains((seq, N+1))`
   is non-empty; the engine refuses to sign B.

3. **Crash-safe journal.** The journal is the **on-disk record of every
   signature the validator has emitted.** On restart, the validator
   replays the journal and rebuilds the `signed` set before processing
   any new nodes. This is why equivocation prevention survives crashes.

A consequence of (2) is that the validator might be first to see
**the wrong chunk** — if A signs and the chain happens to be
proceeding on B, the chain forks. The application decides which
to treat as canonical: usually "the chunk with the first
Certificate assembled by the network."

## Gossip protocol: pull, push, hybrid

The chunk gossip is a standard structured-gossip pattern with a
capability twist. Default in Commonware: pull-based gossip using the
broadcast cache from chapter 09.

```
Pull-based (default):
  push (chunk, parent_cert) to a random subset of peers on first sight
  recipients cache if signature is valid
  on restart, request missing chunks by parent-cert digest from any peer who advertised it
  on the critical path (parent cert chain), request synchronously
```

Why pull beats push for chunks:

1. **Bandwidth.** Push-everything-on-every-gossip at 100 validators
   means each chunk is re-sent ~99 times before it's universally
   known. Pull only re-sends on demand.
2. **Resilience to churn.** New validators join by pulling recent
   chunks; they don't need to play catch-up via a leader.
3. **Caching.** The broadcast engine's cache + dedup logic (chapter 09)
   applies whether you push or pull.

The bandwidth tradeoff: chunks are large, but Acks are small. So
gossip for **chunks** is pull; gossip for **acks** is push. Acks fit
into a single UDP-equivalent packet; pulling each one would multiply
the round-trip cost.

The gossip semantics for *certificates* are like Acks — small, push,
repeatedly gossiped with dedup.

## The Chunk types, full enum

```rust
pub enum Message {
    // Sequencer → Validators
    Node(Node),
    // Validators → Sequencer (and other Validators)
    Ack(SignedAck),
    // Used internally for state sync
    ChunkRequest { sequencer: PK, height: Height },
    ChunkResponse { chunk: Chunk, parent_cert: Certificate },
}

pub struct Node {
    pub chunk: Chunk,
    pub parent_cert: Certificate,
    pub sequencer_signature: Signature,
}

pub struct Chunk {
    pub sequencer: PK,
    pub height: Height,
    pub payload: Bytes,
    pub timestamp: SystemTime,
    pub namespace: Bytes,
}

pub struct Ack<S: Scheme> {
    pub height: Height,
    pub digest: Digest,
    pub signer: PK,
    pub signature: S::Signature,
}

pub struct Certificate<S: Scheme> {
    pub height: Height,
    pub digest: Digest,
    pub certificate: S::Certificate,
}
```

The relationship to "blocks": a Chunk is *not* a block. A block
(in Simplex terms) is the consensus output — a finalized unit of
state transitions. A Chunk is the **data unit** the sequencer ships.
The application decides what the payload bytes mean; a typical app
embeds the block's content in the payload, but the ordered_broadcast
primitive doesn't know or care.

## Certificate construction: the threshold math

When `2f+1` acks arrive for `(sequencer, height)`:

```rust
fn assemble(sequencer: PK, height: Height, acks: BTreeMap<PK, Ack<S>>) -> Certificate<S> {
    // All acks must agree on the digest (else equivocation: reject)
    let digests: HashSet<_> = acks.values().map(|a| a.digest).collect();
    assert_eq!(digests.len(), 1, "equivocation: acks disagree on digest");

    // Aggregate the signatures via the scheme
    let sigs = acks.values().map(|a| &a.signature);
    let agg = S::assemble_certificate(sigs).expect("scheme assembly failed");

    Certificate {
        height,
        digest: digests.into_iter().next().unwrap(),
        certificate: agg,
    }
}
```

For BLS12-381 multisig: `S::assemble_certificate` returns the
aggregate point plus a bitfield of which indices participated. For
threshold: `S::assemble_certificate` returns a single aggregate point
with **no attribution.** For Ed25519 / secp256r1:
`S::assemble_certificate` returns a vector of size 2f+1.

The mathematical fact: any 2f+1-sized intersection of two
honest-majority quorums has at least `f+1` honest signatures, and
the **f+1-th** honest signature is **enough to make the certificate
indistinguishable from one assembled by an honest-only quorum.** This
is the safety arg from chapter 01.

A downstream verifier doesn't see the 2f+1 sigs individually (for BLS
threshold) — they see one aggregate point. They check: `e(agg_sig,
g2) == e(aggregated_pubkey, H(namespace || height || digest))`. One
pairing. Independent of quorum size.

## Rust pattern — `Pin<&mut Self>` for self-referential async actors

The most subtle Rust pattern in this chapter is the use of `Pin` for
the Engine struct. Why?

The Engine has fields like:

```rust
struct Engine<S, P> {
    mailbox_receiver: Receiver<Message>,        // borrows from self
    chunk_sender: Sender<Chunk>,                // borrows from self
    ack_manager: AckManager<S>,                 // owns state
    broadcast: Broadcast,                       // owns state
    ...
}
```

When you write the engine's loop as `async fn run(self)`, Rust tries
to move `self` into the future. But the `Receiver` borrows from
`self.mailbox` — that's a self-referential struct. Self-referential
structs are **not allowed in safe Rust** because `&self.field` cannot
outlive the move.

The fix is to **pin the engine and take `Pin<&mut Self>`**:

```rust
async fn run(self: Pin<&mut Self>) {
    let this = self.project();   // returns Pin<&mut FieldType> for each field
                                   // (works because of `pin-project` crate or hand-rolled
                                   // `unsafe` `Pin::get_unchecked_mut`)
    select! {
        msg = this.mailbox_receiver.recv() => { ... }
        msg = this.chunk_receiver.recv() => { ... }
    }
}
```

The actual code in `consensus/src/ordered_broadcast/engine.rs` uses
either the `pin-project` crate or `tokio::pin!` depending on the
runtime choice.

### `Box::pin` vs `Arc<Pin<Box<...>>>`

When spawning:

```rust
// Box::pin: pinned Box, owned by the spawned task
let engine = Engine::new(...);
tokio::spawn(Box::pin(engine.run()));

// Arc<Pin<Box<T>>>: pinned Box, shared across multiple owners (rare)
// Used when multiple tasks need to send messages to the engine
let engine = Arc::new(Pin::new(Box::new(Engine::new(...))));
```

`Box::pin` is the standard. It heap-allocates the future and pins it
in place, which is necessary for any future that wants to borrow
from itself (which async tasks almost always do). The spawned task
takes ownership of the Box; when the future ends, the Box is freed.

The `Arc<Pin<Box<T>>>` form is rarer and usually a sign of "I need
multiple handles to the pinned future, because multiple senders all
hold this Engine pointer." Commonware's mailbox pattern does not
need this — the actor owns its own state and exposes senders via
`Sender`. The actor is moved into its task; the senders live outside.

### Why this matters for correctness

Without `Pin`, the compiler rejects async functions that borrow
across `.await` points where the borrow lasts into a future that
moves `self`. The check protects against creating a `&self.foo`
reference, moving `self` to a new memory location, and then using the
now-dangling reference. `Pin<&mut Self>` contracts that the engine
will never move after pinning — so any `&self.foo` is stable for the
lifetime of the future.

This is why Commonware's actor types are written
`async fn run(self: Pin<&mut Self>)` and not
`async fn run(self)`. The difference is invisible at the type signature
level but fatal for the borrow checker.

## Where to look in the code (expanded preview)

The appendices that follow go deeper on each topic above. When you
see a `file:line` reference in main body, the appendix walks it
line by line — but the surrounding structure and motivation live
here.

---


## Deepening 1 — Total order broadcast theory in full

Ordered Broadcast implements **Total Order Broadcast (TOB)** — a
classic distributed-systems primitive. This section formalizes
the problem, the impossibility result, and the partial-synchrony
escape.

### The formal problem

Three equivalent definitions appear across the literature;
Commonware's Ordered Broadcast matches all three:

1. **Agreement**: if any honest node delivers message m at index
   i, then every honest node delivers m at index i.
2. **Total order**: if any honest node delivers message m1
   before message m2, then every honest node delivers m1 before m2.
3. **Gap-free**: each honest node's delivered sequence is the same
   contiguous sequence — no skips, no duplicates.

For Ordered Broadcast in Commonware, the "index" is `(sequencer,
height)`. Node A delivers (seq = S, height = 5) → all honest nodes
deliver (S, 5). Each S has its own monotonic sequence.

### The Uniform vs Non-uniform distinction

Cristian, Aghili, and Strong (1985, "Atomic Broadcast of Messages
over Noisy Channels") introduced the Uniform vs Non-uniform split:

- **Non-uniform**: agreement holds only for honest nodes.
- **Uniform**: agreement holds for *all* nodes, including
  Byzantine ones.

Commonware's Ordered Broadcast is **uniform** — once a chunk has
been certified, the certificate acts as proof *to anyone*, including
potentially Byzantine nodes. A Byzantine node can be shown the
Certificate and must accept it.

The reason Uniform matters: cross-chain bridges. A bridge's
verifier chain might be a Byzantine node (compromised validator).
The Certificate must be unforgeable even to that Byzantine node.

### The FLP impossibility again

Fischer, Lynch, Paterson (1985, "Impossibility of Distributed
Consensus with One Faulty Process") proved the headline result:

> In a fully asynchronous system with even one Byzantine node, no
> deterministic protocol can guarantee consensus.

This is FLP. The proof: consider the FLP scenario — a network with
n ≥ 2 nodes where one is Byzantine. Construct an execution where
the network messages are delivered in an adversarial order. The
Byzantine node can hide votes until the last moment. The
honest nodes are partitioned in their decision.

The headline result: there's no "always-make-progress" guarantee
in pure asynchrony.

### The partial-synchrony escape

Dwork and Lynch (1988, "Consensus in the Presence of Partial
Synchrony") showed the escape: assume the network is *eventually*
synchronous. Specifically:

> There exists a Global Stabilization Time (GST) and a known bound
> Δ such that all messages sent at time t are delivered by time
> t + Δ for all t ≥ GST.

Under partial synchrony:

- **Safety always**: no two honest nodes ever disagree on the
  total order.
- **Liveness eventually**: after GST, progress is guaranteed.

Commonware's Ordered Broadcast is partial-synchronous. The
protocol is *safe at all times*; *live after network behaves*.

### Why Ordered Broadcast is "lighter" than Simplex

Ordered Broadcast skips two mechanisms Simplex needs:

1. **Leader election**: a fixed sequencer is known at the
   configuration time. No per-view election.
2. **View-change protocol**: when the sequencer stalls, Ordered
   Broadcast stalls. There's no view-change within the protocol
   (the application may add one outside).

The "linked chunks" pattern provides safety directly: validators
agree on chunk N+1's content by first agreeing on chunk N's
certificate. The chain is implicit in the certificate linkage.

### The "linked chunks" as proof-of-publication

Each chunk references its parent's certificate. To form a quorum
on chunk N+1, validators must have seen (and verified) chunk N's
certificate. This is **proof-of-publication** in the
data-availability sense: every node that signs chunk N+1 has
*seen* chunk N's certificate, hence can be told the contents of
N if needed.

In a sense, the chain link is a Merkle-root-like structure: the
parent certificate's digest commits to all the chain state.

### Total Order Broadcast = Atomic Broadcast

TOB is sometimes called **Atomic Broadcast** (AB). The two terms
are interchangeable in the literature. The difference is purely
historical: AB comes from Cristian et al. 1985 (atomic = all nodes
do the same thing); TOB comes from Birman-Schiper-Stephenson
1991 ("Total Order Multicast"). The semantics are identical.

In Commonware's naming, both map to Ordered Broadcast.

### The agreement property in detail

> If any honest node delivers message m at index i, then every
> honest node delivers m at index i.

This is the strongest possible agreement property — once a single
honest node accepts m at i, *all* honest nodes will. They might
not have seen m yet, but they're committed to delivering it at
position i.

The implication for Ordered Broadcast: chunk N is either
delivered by every honest node at the same height, or by no
honest node. There's no "split."

### Equivocation and the link

A naive TOB has a Byzantine sequencer broadcasting two different
chunks at the same height. The link rule prevents this from
silently forking:

1. Sequencer broadcasts chunk A and chunk B (different content) at
   height N+1. Both reference the same parent cert (height N).
2. Validators receive both. Each validator's first-seen rule
   makes them sign *one* of A or B. They journal the signed chunk.
3. Across validators, the network reaches a quorum on one of A or
   B (depending on which arrived first at enough validators).
4. The other chunk is *ignored* — no quorum forms. Validators who
   signed the ignored chunk are Byzantine (or unlucky).

The journal makes this recovery-safe: a validator that crashes
replays the journal and knows what it signed.

---

## Deepening 2 — Sequencer designs in depth

The "known sequencer" assumption is a sliding scale.

### Single sequencer

One well-known party broadcasts chunks. The simplest design.

Trust: trust the sequencer. If the sequencer equivocs, the
network forks; if the sequencer stalls, the chain stops.

Counter-measures (external to Ordered Broadcast):
- Slashing: detect equivocation post-hoc, slash the sequencer's
  stake, rotate.
- Watchtowers: monitor the chain for forks, alert on detection.

Commonware's `ordered_broadcast` module supports this directly
via the `sequencer` config field.

### Round-robin sequencer

Sequencers rotate per epoch. Each epoch has one designated
sequencer (the "current" one); when the epoch changes, a new
sequencer is selected.

Trust: trust the union of all sequencers. No minority
equivocators.

Mechanism: the application maintains a `current_epoch: u64`
and `sequencer_for_epoch(e): Option<PK>`. The Ordered Broadcast
config picks the active sequencer per epoch.

Cost: rotation ceremony per epoch. For 100 sequencers and
1-day epochs: 100 rotations/year, manageable.

### VRF-elected sequencer

VRF picks a sequencer per round (similar to Simplex but without
the safety-critical notarization logic; here, even equivocation
just halts the chain).

Trust: honest supermajority of stake. Approximately the same
as Simplex.

Cost: one VRF per round per validator. O(N) VRF computations
per round.

### PoW-style sequencer

Whoever finds a nonce can publish. No predetermined sequencer;
any party who proves work can submit.

Trust: honest majority of compute (e.g., PoW assumption).

Cost: continuous hashing. Energy-intensive.

### Comparison

| Design | Trust | Cost | Liveness under asynchrony |
|---|---|---|---|
| Single | Sequencer alone | O(1) | No (stalls) |
| Round-robin | Union of sequencers | O(N) rotation ceremony | Eventually (per epoch) |
| VRF-elected | Honest supermajority | O(N) VRF per round | Eventually (per round) |
| PoW | Honest majority compute | O(N) continuous | Eventually (per PoW cycle) |

Commonware's Ordered Broadcast is design-agnostic at the
sequencer level — it accepts any sequencer PK. The application
implements the rotation policy separately.

### The rotation mechanics

For round-robin:

```rust
pub fn sequencer_for_epoch(epoch: u64, sequencer_set: &[PK]) -> PK {
    sequencer_set[epoch as usize % sequencer_set.len()].clone()
}
```

For VRF:

```rust
pub fn sequencer_for_round(round: u64, validators: &[(PK, VrfPK)]) -> Option<PK> {
    let mut best: Option<(PK, VrfOutput)> = None;
    for (pk, vrf_pk) in validators {
        let input = encode_round(round);
        let (output, _) = vrf_pk.verify_prehash(&input);
        if best.as_ref().map_or(true, |(_, b)| output < *b) {
            best = Some((pk.clone(), output));
        }
    }
    best.map(|(pk, _)| pk)
}
```

The Validator's epoch-state holds the current sequencer PK. On
epoch transition, the new sequencer PK is loaded into config.

### Failure modes and defenses

A malicious sequencer can:

1. **Equivocate**: broadcast two chunks at the same height. The
   link rule + journal + first-seen-wins prevent the network from
   silently forking. The misbehavior is *detectable* (every honest
   validator sees both and rejects one), even if not auto-punished.
2. **Censor**: refuse to broadcast specific chunks. The chain
   stalls at that height.
3. **Stall**: refuse to broadcast anything. Same as censor.

For Ordered Broadcast, the defenses are external:

- **Watchtower**: a separate service watches the chain for forks
  and slashing evidence.
- **Fallback**: after a stall, switch to a different sequencer
  (the application chooses the new one).
- **External slashing**: post-hoc evidence collection, then
  punish the sequencer.

These mechanisms are application-level; Ordered Broadcast just
provides the protocol.

---

## Deepening 3 — DAG-based consensus in depth

A different answer to "total order over a stream" comes from
Mysten Labs (Sui): DAG-based consensus (Narwhal mempool +
Bullshark / Tusk consensus).

### The DAG structure

Instead of a single sequencer broadcasting a chain, everyone
broadcasts "vertices" forming a DAG (directed acyclic graph). Each
round advances; the consensus rule is "transitively finalize
vertices that have 2f+1 causal support."

The graph looks like:

```
        v_5 -- v_6
       / |    / |
      /  |   /  |
   v_1   v_2  v_3
       \ |    /
        \|   /
         v_4
```

Each vertex v references multiple parents (the "previous round's"
vertices). A vertex is **finalized** if it has 2f+1 causal support
(multiple paths back from future vertices).

### Bullshark consensus

Bullshark (Danezis, et al. 2022, "Bullshark: DAG BFT Protocols
Made Practical") is the practical consensus layer. The
algorithm:

1. Each round: every validator broadcasts a vertex with parents
   from the previous round.
2. After 2 rounds: vertices with 2f+1 ancestors are "candidates."
3. After 3 rounds: candidates become ordered.
4. The DAG structure provides the chain ordering implicitly.

Tusk (the modern variant) provides even faster finality: ~3
rounds for honest case, sub-second finality.

### Contrast with chain-based

| | Ordered Broadcast (OB) | DAG-based (Narwhal/Bullshark) |
|---|---|---|
| Producer role | Single sequencer | Every validator submits |
| Topology | Linked list (chain) | DAG |
| Bandwidth per validator | O(1) (chunks signed) | O(N) (causal references) |
| Finality latency | One-hop certification | ~3 rounds with Bullshark |
| Liveness under partial synchrony | Single sequencer can stall | Producers can take over for stuck validators |
| Implementation cost | ~2 KLOC | ~10+ KLOC |

### Why Commonware chose OB over DAG

Commonware's choice of "single sequencer + linked chain" is a
deliberate engineering simplification. Rationale:

- **Simplicity**: ~2 KLOC vs ~10+ KLOC. Smaller attack surface.
- **Single trust point**: easier to reason about.
- **No causal history**: a chain doesn't need to track "who
  referenced whom" — just "what's the parent cert."
- **Limited use cases**: if you have one trusted sequencer, OB is
  the right tool. DAGs shine when no single sequencer is
  acceptable.

If you need DAG semantics or if no single sequencer is acceptable,
use a different library (Sui, AptosBFT, Narwhal). Ordered Broadcast
is the right primitive when one trusted (slashable) sequencer is
acceptable.

### When DAGs win

DAGs have these advantages:

- **Parallel block production**: all validators can submit
  concurrently without losing total order.
- **Censorship resistance**: a single sequencer can censor, but
  in a DAG, anyone can add vertices.
- **Throughput scaling**: more vertices per round = more
  transactions per round.

For a *consensus* network (where any validator should be able to
add transactions), DAGs are better. For an *application-level*
chain (where one party produces and others verify), OB is better.

### When OB wins

OB has these advantages:

- **Simplicity**: fewer moving parts.
- **Lower bandwidth**: O(1) per validator for Acks, vs O(N) per
  vertex for DAGs.
- **Predictable performance**: single leader means predictable
  throughput.
- **Single-trust model**: slashing is straightforward.

For an application like a "bridge witness" (where a known bridge
operator is the sequencer), OB is the right choice.

---

## Deepening 4 — The Engine in full

Ordered Broadcast's Engine plays two roles in the same loop:
sequencer and validator. This section walks through the engine
in detail.

### The lifecycle per epoch

```rust
struct Engine<S, P> {
    config: Config<S>,
    // ... mailbox, journal, ack_manager, tip_manager, etc.
}

impl<S, P> Engine<S, P> {
    pub async fn run(self: Pin<&mut Self>) {
        let this = self.project();
        loop {
            select! {
                // Local command: application submitted a chunk.
                msg = this.command_receiver.recv() => {
                    if this.is_sequencer(this.config.epoch) {
                        this.handle_sequencer_msg(msg).await;
                    } else {
                        // Validator doesn't process commands
                    }
                }
                // Incoming Node from a sequencer.
                node = this.node_receiver.recv() => {
                    this.handle_node_msg(node).await;
                }
                // Incoming Ack from a peer validator.
                ack = this.ack_receiver.recv() => {
                    this.handle_ack_msg(ack).await;
                }
                // Stop signal.
                _ = this.stop_signal => break,
            }
        }
    }
}
```

The two roles share the same loop but branch on
`config.sequencer_pk == our_pk`.

### Sequencer-side flow

When the application submits a chunk to the sequencer:

1. **Receive chunk** from the application's `Sender<Chunk>`.
2. **Read current height** from the journal: `h = last_cert_height + 1`.
3. **Read parent certificate** from the journal: `parent_cert =
   last_certificate`.
4. **Sign the Node**: `node_signature = sign(sequencer_sk, encode(node))`.
5. **Append to journal**: WAL for crash-recovery.
6. **Multicast Node** to peers over P2P.
7. **Wait for certificate**: subscribe to `Reporter::Activity` for
   certificate events.
8. **On certificate received**: update local chain state, advance
   last_cert_height, advance last_certificate.
9. **Repeat** for the next chunk.

A validator can also run a sequencer's flow (if the validator is
also the configured sequencer). In that case, both branches of
the select are exercised.

### Validator-side flow

When a peer validator receives a Node:

1. **Verify Node signature** against the sequencer's PK for the
   current epoch.
2. **Verify parent** is the current tip in `TipManager`. If the
   parent cert doesn't extend the chain, reject.
3. **Check signed-by-me**: does `(sequencer, height)` already have
   our signature? If yes, reject (equivocation).
4. **Verify chunk content** (application-defined logic): call
   `Application::verify(chunk)`.
5. **If valid: sign and multicast Ack**.

When the validator receives an Ack:

1. **Verify Ack signature** against the validator's PK for the epoch.
2. **Add to AckManager's set** for `(sequencer, height)`.
3. **If quorum reached** (2f+1 acks on same digest):
   - **Assemble Certificate** via `S::assemble_certificate`.
   - **Emit to Reporter**: `Activity::Certificate(cert)`.
   - **Update TipManager** with the new cert.

### The wiring

The Engine is wired into the application through:
- `Sender<Chunk>` for outgoing chunks (sequencer only).
- `Reporter` for incoming certificates (both roles).
- `Sender<Node>` / `Receiver<Node>` for P2P (both roles).
- `Sender<Ack>` / `Receiver<Ack>` for Ack multicast (both roles).

In a common deployment:

```rust
let (chunk_tx, chunk_rx) = mailbox::bounded(64);
let (cert_tx, cert_rx) = mailbox::bounded(64);

let cfg = ordered_broadcast::Config {
    sequencer_pk: my_pk_if_im_sequencer_else_some_other,
    namespace: my_namespace,
    scheme: bls_threshold,
    epoch: Epoch::new(0),
    // ... etc
};
let engine = ordered_broadcast::Engine::new(context, cfg);

engine.start(chunk_rx, cert_tx);
```

The application uses `chunk_tx` to submit chunks (sequencer only)
and `cert_rx` to receive certificates (both roles).

### The role toggle

A node may be the sequencer for some epochs (e.g., epoch 0 to 10)
and only a validator for others (epoch 11+). The `Config.sequencer_pk`
changes per epoch.

At an epoch boundary, the engine reloads config:
1. Reads new sequencer PK from the trusted config source.
2. If new PK == our PK: enable sequencer-side submission.
3. If new PK != our PK: disable sequencer-side submission.

The transition is graceful (the engine reads from the journal
to recover state).

---


## Deepening 5 — AckManager in depth

The `AckManager` tracks in-progress acks for each `(sequencer,
height)` pair. This section walks the data structure and the
algorithm.

### The data structure

```rust
struct AckManager<S: Scheme> {
    pending: HashMap<(PK, Height), PendingAck<S>>,
    acks: HashMap<(PK, Height), BTreeMap<PK, Ack<S>>>,
}

struct PendingAck<S: Scheme> {
    chunk: Chunk,
    parent_cert: Certificate<S>,
    started_at: SystemTime,
}
```

Two maps:

- `pending`: keyed by `(sequencer, height)`. Holds the chunk being
  acked and metadata.
- `acks`: keyed by `(sequencer, height)`. Holds the in-progress
  signed Acks.

### Why a `BTreeMap<PK, Ack<S>>` and not a `HashMap`?

For the inner map, ordering matters:

1. **Stable iteration when assembling a Certificate**: BLS threshold
   aggregation is order-independent (sum is commutative), but
   multisig aggregation depends on the order of bits in a
   bitfield. A `BTreeMap` gives a stable PK order.
2. **Determinism for tests**: deterministic iteration ensures the
   `Auditor::state` hash is reproducible.
3. **Fast lookups by PK**: `BTreeMap` is O(log n), fine for inner
   maps of at most 100 entries.

For the outer `pending` map, `HashMap` is used because lookups by
sequencer+height don't need ordering.

### The algorithm

```rust
impl<S: Scheme> AckManager<S> {
    fn add_ack(&mut self, sequencer: PK, height: Height, signer: PK, ack: Ack<S>)
        -> Option<Certificate<S>>
    {
        // Check signature first.
        let valid = S::verify(&epoch_pk_for(signer), &ack_namespace, &payload, &ack.signature);
        if !valid { return None; }

        // Add to the ack set.
        let entry = self.acks.entry((sequencer.clone(), height)).or_default();
        if entry.contains_key(&signer) {
            return None;  // double-counting
        }
        entry.insert(signer.clone(), ack.clone());

        // Quorum check.
        if entry.len() >= 2 * self.f + 1 {
            // All acks agree on the digest?
            let digest: HashSet<_> = entry.values().map(|a| a.digest).collect();
            if digest.len() != 1 {
                return None;  // fork detected
            }

            // Assemble certificate.
            let dig = digest.into_iter().next().unwrap();
            let sigs = entry.values().map(|a| &a.signature);
            let cert = S::assemble_certificate(sigs).expect("scheme assembly failed");
            Some(Certificate { height, digest: dig, certificate: cert })
        } else {
            None
        }
    }

    fn prune_completed(&mut self, sequencer: PK, height: Height) {
        self.pending.remove(&(sequencer, height));
        self.acks.remove(&(sequencer, height));
    }

    fn pending_count(&self) -> usize {
        self.pending.len()
    }
}
```

The quorate check happens *eagerly*: as soon as the 2f+1-th ack
arrives, the Certificate is formed.

### The equivocation check

The "all acks agree on digest" check is critical: if a Byzantine
sequencer sent two different chunks at the same height, validators
who received chunk A signed acks for A; validators who received
chunk B signed acks for B. The two sets don't overlap in digest.

The check `digest.len() != 1` rejects this case: no Certificate
is formed, and the chain doesn't advance. A subsequent view
change (or fallback) handles the equivocation at the application
level.

### The certificate emit

When a Certificate is formed, the engine emits to the Reporter:

```rust
self.reporter.report(Reporter::Activity::Certificate(cert)).await;
```

The application's Reporter implementation might:
- Forward to downstream consumers (chapter 17).
- Update local state.
- Persist for crash-recovery.

The Certificate is durable once it's been emitted.

---

## Deepening 6 — TipManager in depth

The `TipManager` tracks the latest chunk per sequencer and
verifies chain extension.

### The data structure

```rust
struct TipManager {
    tips: BTreeMap<PK, Tip>,
    certs: BTreeMap<Digest, Certificate>,
}

struct Tip {
    sequencer: PK,
    height: Height,
    chunk_digest: Digest,
    cert_digest: Digest,    // digest of the certificate proving chunk's inclusion
}
```

Two maps:

- `tips`: keyed by sequencer PK. Holds the latest known chunk per
  sequencer.
- `certs`: keyed by certificate digest. Holds recent certificates
  (cache for verification).

### The chain-extension rule

When a new chunk arrives:

```rust
fn update_tip(&mut self, sequencer: PK, chunk: Chunk, cert: Certificate)
    -> Result<(), Error>
{
    let current = self.tips.get(&sequencer);
    match current {
        None => {
            // Genesis: accept the chunk.
            let tip = Tip {
                sequencer: sequencer.clone(),
                height: chunk.height,
                chunk_digest: chunk.digest(),
                cert_digest: cert.digest(),
            };
            self.tips.insert(sequencer, tip);
            Ok(())
        }
        Some(tip) => {
            // Check that chunk extends current.
            let expected_parent = tip.cert_digest.clone();
            if chunk.parent != expected_parent {
                return Err(Error::NotAnExtension);
            }
            // Reject if height isn't tip.height + 1
            if chunk.height != tip.height + 1 {
                return Err(Error::HeightMismatch);
            }
            // Accept and update.
            let new_tip = Tip {
                sequencer,
                height: chunk.height,
                chunk_digest: chunk.digest(),
                cert_digest: cert.digest(),
            };
            self.tips.insert(tip.sequencer.clone(), new_tip);
            Ok(())
        }
    }
}
```

The rule is "chunk's parent digest == current tip's cert digest,
and chunk's height == current tip's height + 1."

### The "catch up" path

A new node joining the chain needs to fetch historical chunks. The
`TipManager` doesn't directly support `get_chunks_after(height)`,
but it provides the *boundary* knowledge: the current tip. The
resolver (chapter 08) fetches missing chunks based on the
boundary.

```rust
async fn catch_up(&mut self, from_height: Height, to_height: Height) -> Result<()> {
    for h in from_height..to_height {
        let key = MarshalKey::Height(sequencer, h);
        let chunk = self.resolver.fetch(key).await?;
        let cert = self.certs.get(&chunk.parent).ok_or(Error::MissingParent)?;
        self.update_tip(sequencer.clone(), chunk, cert.clone())?;
    }
    Ok(())
}
```

This catches up the local view to the network's view.

### Edge cases

1. **Concurrent chains at the same height**: two different
   `update_tip` calls for the same `(sequencer, height)` would
   conflict. The `update_tip` is called within a single-threaded
   section (the engine's task), so it's safe.
2. **Gaps in the chain**: the validator missed chunks at heights
   N+1, N+2. When N+3 arrives with parent = N's cert, the check
   `chunk.height != N + 3` will fail (it's N+1 expected). The
   validator must catch up via the resolver.
3. **Double-height chunk**: a chunk at height N+1 is verified
   against current tip = N. If a chunk at height N+2 arrives
   first (without N+1), `update_tip` sees height mismatch and
   rejects. The validator must fetch N+1 first.

The TipManager is strict. It doesn't accept "jumps" in the chain.

### The interaction with marshal

Marshal uses TipManager's view for backfill decisions: when
gaps appear, fetch the missing chunks. The TipManager's strict
chain-extension rule is what makes backfill deterministic.

---

## Deepening 7 — Equivocation prevention in depth

Equivocation (a Byzantine actor signing two conflicting messages)
is prevented by three mechanisms working together.

### Mechanism 1: the chain link

Each Node carries `parent_cert`. To broadcast two different chunks
at height N+1, the Byzantine sequencer must reference the *same*
parent (the certificate from height N). The two chunks are
siblings, both extending a single canonical tip.

Equivocation means: chunk A and chunk B, both with parent = cert
of N, both have height N+1, but A ≠ B (different content).

### Mechanism 2: first-seen-wins

When the validator's engine sees chunk A at height N+1, it signs
and journals. When it later sees chunk B at the same height, the
journal lookup `signed.contains((seq, N+1))` is non-empty; the
engine refuses to sign B.

The journal-on-sign rule:

```rust
fn on_chunk(&mut self, chunk: Chunk, parent_cert: Certificate) -> Result<()> {
    // 1. Verify signature.
    self.scheme.verify(&sequencer_pk, &chunk_namespace, &chunk_payload, &chunk.sig)?;

    // 2. Check journal: have we already signed?
    let key = (chunk.sequencer, chunk.height);
    if self.signed.contains(&key) {
        return Err(Error::AlreadySigned);
    }

    // 3. Verify chain extension.
    self.tip_manager.update_tip(...)?
        .map_err(|_| Error::ChainViolation)?;

    // 4. Verify content.
    let valid = self.application.verify(&chunk).await?;
    if !valid { return Err(Error::InvalidChunk); }

    // 5. Sign and journal.
    let ack = self.sign_ack(&chunk)?;
    self.journal.append(...)?;  // WAL write
    self.signed.insert(key);

    // 6. Broadcast ack.
    self.sender.send(Recipients::All, ack.encode(), false).await?;
    Ok(())
}
```

The first-seen-wins rule is **the validator's local choice**. A
Byzantine validator could break it (not journal, sign multiple).
But it'd be detected (the journal is replayed on restart, the double-
sign shows up at restart time).

### Mechanism 3: crash-safe journal

The journal is the **on-disk record of every signature the validator
has emitted**. On restart, the validator replays the journal and
rebuilds the `signed` set before processing any new nodes. This is
why equivocation prevention survives crashes.

The order: verify (off-journal) → journal-write (WAL sync) →
broadcast. Same pattern as Simplex's WAL-then-broadcast rule
(chapter 11).

### The consequence

A consequence of (2) is that the validator might be first to see
**the wrong chunk**. If validator A sees chunk X at height 5 first
and signs it, but the network's actual canonical chain is on chunk
Y at height 5, the validator A is now "stuck" on chunk X.

This is a known issue. The application decides which to treat as
canonical:

- "The chunk with the first Certificate assembled by the network"
  — typically the chunk that got 2f+1 acks first.
- "The chunk chosen by the sequencer's signing key" — assumes
  you have a way to know.

If the application is wrong (e.g., signs X but should have waited
for Y), it's a slashing event under most designs.

### The triple-signature check

The validator's local state has three pieces of evidence per
chunk:

1. **The signature**: signed by the validator's key.
2. **The sequence number**: (sequencer, height) is unique.
3. **The chunk content**: the bytes that were signed.

Any combination of two is ambiguous; the third provides the binding.
Together they form a "triple signature" that uniquely identifies
the validator's commitment:

```
validator_commits_to(sequencer, height, chunk)
```

A Byzantine validator who has two such triples (same sequencer+height,
different chunks) is provably Byzantine.

The journal's record of "I signed (S, H) at content C" is the
proof. Any peer who requests the journal can confirm.

---

## Deepening 8 — Chunk types in full

The full enum and encoding for chunk-related messages.

### The Message enum

```rust
pub enum Message<S: Scheme> {
    Node(Node),
    Ack(Ack<S>),
    ChunkRequest { sequencer: PK, height: Height },
    ChunkResponse { chunk: Chunk, parent_cert: Certificate<S> },
}
```

Four variants:
- `Node` is broadcast by the sequencer.
- `Ack` is broadcast by validators.
- `ChunkRequest` / `ChunkResponse` are pull-style fetches for
  backfill.

### The Node struct

```rust
pub struct Node<S: Scheme> {
    pub chunk: Chunk,
    pub parent_cert: Certificate<S>,
    pub sequencer_signature: S::Signature,
}
```

The `sequencer_signature` is over `encode(chunk, parent_cert)`
(or however the protocol defines the signed payload). This binds
the chunk to its parent, preventing relay attacks.

### The Chunk struct

```rust
pub struct Chunk {
    pub sequencer: PK,
    pub height: Height,
    pub payload: Bytes,
    pub timestamp: SystemTime,
    pub namespace: Bytes,    // for cross-chain routing
}
```

The `payload` is the actual data (typically a block, a bridge
message, an oracle update). The `timestamp` records when the
sequencer created the chunk.

### The Ack struct

```rust
pub struct Ack<S: Scheme> {
    pub height: Height,
    pub digest: Digest,
    pub signer: PK,
    pub signature: S::Signature,
}
```

The `signature` is over `encode(height, digest)` in the ack
namespace. Validators sign `(height, digest)` to commit to having
seen the chunk.

### The Certificate struct

```rust
pub struct Certificate<S: Scheme> {
    pub height: Height,
    pub digest: Digest,
    pub certificate: S::Certificate,   // BLS aggregate or multisig or list
}
```

The `certificate` field's structure depends on the scheme:
- BLS threshold: 1 curve point.
- BLS multisig: 1 curve point + bitfield (which validators
  contributed).
- Ed25519 / secp256r1: list of individual signatures.

### The encoding

All these types use Commonware's Codec (chapter 03). The
encoding:

```rust
impl<S: Scheme> Encode for Message<S> {
    fn encode(&self) -> Bytes {
        match self {
            Message::Node(n) => encode_node(n),
            Message::Ack(a) => encode_ack(a),
            Message::ChunkRequest { sequencer, height } => {
                encode_chunk_request(sequencer, height)
            }
            Message::ChunkResponse { chunk, parent_cert } => {
                encode_chunk_response(chunk, parent_cert)
            }
        }
    }
}
```

The wire format is type-distinguished:

```
1 byte | variant discriminator |
rest   | variant payload
```

For ChunkRequest:

```
variant_byte: 3
sequencer:    32-96 bytes (Ed25519 or BLS)
height:       varint
```

For ChunkResponse:

```
variant_byte:    4
chunk:           Encode of Chunk
parent_cert:     Encode of Certificate<S>
```

### The digest computation

A `Chunk`'s digest is the SHA-256 of its encoded form:

```rust
impl Chunk {
    pub fn digest(&self) -> Digest {
        let encoded = self.encode();
        let hash = Sha256::hash(&encoded);
        Digest(hash.into())
    }
}
```

Same for `Node` digest (over `encode(node)`), `Certificate` (over
its encoded form), etc.

The digest is what's referenced in `Ack`s and `Certificate`s —
they sign `digest`, not the full bytes.

---

## Deepening 9 — Certificate construction in depth

Forming a Certificate from 2f+1 Acks requires:

1. **Equivocation check**: all Acks must agree on the digest.
2. **Aggregation**: combine the signatures via the scheme.
3. **Encoding**: store the Certificate with aggregated
   signatures.

### The threshold math

For BLS threshold (t = 2f+1, n = 3f+1):

- Any 2f+1 subset's signatures can produce a valid certificate.
- The Lagrange coefficients fold the subset choice away.
- The resulting certificate is *indistinguishable* from one made
  by any other 2f+1 subset.

For BLS multisig (no threshold):

- Any 2f+1 signatures from the *same* multisig pubkey set produce
  a valid certificate.
- The multisig includes a bitfield of which validators
  contributed.

For Ed25519 / secp256r1:

- Any 2f+1 individual signatures, all from different validators.
- The Certificate is a `Vec<(PK, Sig)>` (no aggregation).

### The equivocation check

```rust
fn assemble_certificate<S: Scheme>(acks: &[Ack<S>]) -> Option<Certificate<S>> {
    if acks.len() < 2 * f + 1 { return None; }

    // All acks must agree on the digest.
    let first = &acks[0].digest;
    if !acks.iter().all(|a| a.digest == *first) {
        // Fork detected: acks disagree on digest.
        return None;
    }

    // Aggregate signatures.
    let sigs = acks.iter().map(|a| &a.signature);
    let cert = S::assemble_certificate(sigs).expect("scheme assembly failed");

    Some(Certificate {
        height: acks[0].height,
        digest: first.clone(),
        certificate: cert,
    })
}
```

The equivocation check fails if the validator signed for one chunk
and another validator signed for a different chunk at the same
height. In that case, the network's honest validators formed their
groups on different chunks, and no Certificate can be assembled.

### The verification

A downstream verifier of a Certificate:

```rust
fn verify_certificate<S: Scheme>(cert: &Certificate<S>, expected_pubkey: &S::PublicKey) -> bool {
    let payload = encode_for_signing(&namespace, cert.height, &cert.digest);
    S::verify_cert(expected_pubkey, &namespace, &payload, &cert.certificate)
}
```

For BLS threshold:

```rust
fn verify_cert(pk_group: &PublicKey, namespace: &[u8], msg: &[u8], cert: &Certificate) -> bool {
    // One pairing check.
    e(cert.aggregate_point, g2) == e(H(msg), pk_group)
}
```

One pairing, regardless of how many signers contributed.

For multisig:

```rust
fn verify_cert(pk_set: &[PublicKey], namespace: &[u8], msg: &[u8], cert: &MultiSigCert) -> bool {
    // One pairing + bitfield check.
    let aggregate_pk = sum_with_bitfield(&cert.bitfield, pk_set);
    e(cert.aggregate_point, g2) == e(H(msg), &aggregate_pk)
}
```

For Ed25519:

```rust
fn verify_cert(pk_set: &[PublicKey], namespace: &[u8], msg: &[u8], cert: &VecCert) -> bool {
    if cert.signatures.len() < 2 * f + 1 { return false; }
    cert.signatures.iter().all(|(pk, sig)| {
        Ed25519::verify(pk, namespace, msg, sig)
    })
}
```

### The Certificate's lifetime

A Certificate is **immutable**: once formed, it never changes. The
Certificate serves as proof and is persisted in Marshal's
`Certificates` store (chapter 12).

A node that doesn't trust the network can verify the Certificate
independently. The Certificate is a portable proof.


## Exercises — Ordered Broadcast

A set of exercises covering Ordered Broadcast's deeper behaviors.

### Exercise 14.1 — Trace a chunk's lifecycle

For a chunk B at height N:

1. Sequencer builds Node { B, parent_cert, sequencer_signature }.
2. Sequencer multicasts Node to peers.
3. Each peer receives Node, verifies signature, signs Ack.
4. Each peer multicasts Ack.
5. Once 2f+1 Acks arrive at the engine, Certificate forms.
6. Reporter emits Certificate.
7. TipManager updates; sequencer waits for Certificate before B+1.

Sketch the messages exchanged for N=5 peers.

### Exercise 14.2 — Equivocation exercise

Suppose a Byzantine sequencer sends two chunks A and B at height 5.
Both reference the same parent.

1. How many Acks does A get? How many does B get? In a 5-validator
   cluster (4 honest, 1 Byzantine), the worst case is 2 for each.
2. Does the Certificate form? (Answer: no — neither reaches 2f+1 =
   3 Acks.)
3. What happens? (Answer: the chain stalls at height 4. The
   Byzantine sequencer is detected (it can be proven to have sent
   conflicting chunks; the application can slash.))

### Exercise 14.3 — Sequencer rotation

For a 3-validator cluster with round-robin sequencers [A, B, C]:

1. Epoch 0 sequencer = A.
2. Epoch 1 sequencer = B.
3. Epoch 2 sequencer = C.

What happens when an epoch boundary crosses?

- Old sequencer stops being able to submit new chunks.
- New sequencer takes over.
- The chain continues; the new sequencer's first chunk references
  the old sequencer's last Certificate.

### Exercise 14.4 — DAG comparison

List 3 concrete applications where Ordered Broadcast is the right
choice and 3 where DAG-based consensus would be better.

OB wins: bridge witnesses, oracle data feeds, MEV-resistant
sequencers.
DAG wins: high-throughput L1 chains, multi-producer
applications, censorship-resistant chains.

### Exercise 14.5 — Certificate verification cost

For a downstream verifier:

1. BLS threshold: 1 pairing = ~50 µs.
2. BLS multisig: 1 pairing + bitfield = ~55 µs.
3. Ed25519 multisig: N signatures verified independently = ~5 ms
   for N=100.

Why is BLS preferred for bridge witness verification?

### Exercise 14.6 — Crash recovery

A validator crashes mid-acknowledgment and restarts. What does the
journal contain?

The journal has:
- All Acks the validator sent (so it doesn't re-send).
- All Certificates the validator formed (so it can re-emit).
- Any in-progress Node *received* (so it can reprocess).

The validator rebuilds `signed` set from the journal, then resumes
participation. Recovery is automatic and doesn't lose state.

---


## Appendix A — The Ordered Broadcast Engine, in detail

`consensus/src/ordered_broadcast/engine.rs`. The Engine has two roles:

### As a sequencer

```rust
struct SequencerState {
    chain: VecDeque<Chunk>,                  // pending chunks to broadcast
    last_broadcast: Option<Digest>,          // last chunk's certificate digest
    pending_certificate: Option<oneshot::Sender<Certificate>>,  // waiting on cert
}

async fn sequencer_loop(&mut self) {
    loop {
        select! {
            // New chunk to broadcast
            chunk = self.chunk_receiver.recv() => {
                self.broadcast_next(chunk).await;
            }

            // Certificate arrived (signed by validators)
            cert = self.cert_receiver.recv() => {
                self.on_certificate(cert).await;
            }

            // Shutdown
            _ = self.stop_signal => break,
        }
    }
}
```

### As a validator

```rust
struct ValidatorState {
    chains: BTreeMap<PK, ChainState>,         // per-sequencer chain tracking
    journal: Journal,                        // WAL for signed chunks
}

struct ChainState {
    latest_chunk: Option<(Chunk, Certificate)>,
    pending_acks: BTreeSet<Height>,           // heights waiting for ack collection
    // ...
}
```

## Appendix B — The `ack_manager`

`consensus/src/ordered_broadcast/ack_manager.rs`. Tracks in-progress
acknowledgments:

```rust
struct AckManager {
    pending: HashMap<(PK, Height), PendingAck>,
    acks: HashMap<(PK, Height), BTreeMap<PK, Ack>>,
}

struct PendingAck {
    chunk: Chunk,
    parent_cert: Certificate,
    started_at: SystemTime,
}

impl AckManager {
    fn add_ack(&mut self, sequencer: PK, height: Height, ack: Ack) {
        let entry = self.acks.entry((sequencer, height)).or_insert_with(BTreeMap::new);
        entry.insert(ack.signer, ack);

        if entry.len() >= QUORUM {
            // Assemble Certificate
            let cert = assemble_certificate(entry.values());
            // Notify the chain manager
            self.on_certificate_ready(sequencer, height, cert);
        }
    }
}
```

## Appendix C — The `tip_manager`

`consensus/src/ordered_broadcast/tip_manager.rs`. Tracks the latest
chunk per sequencer:

```rust
struct TipManager {
    tips: BTreeMap<PK, Tip>,                  // sequencer → latest tip
    pending_tips: BTreeMap<PK, BTreeSet<Digest>>,  // chunks awaiting tip update
}

struct Tip {
    chunk: Chunk,
    certificate: Certificate,
    height: Height,
}

impl TipManager {
    fn update_tip(&mut self, sequencer: PK, chunk: Chunk, cert: Certificate) {
        // Only accept if this chunk builds on the current tip
        let current = self.tips.get(&sequencer);
        if let Some(t) = current {
            if chunk.parent != t.certificate.digest() {
                // Not a valid extension; ignore
                return;
            }
        }

        // Advance the tip
        self.tips.insert(sequencer, Tip { chunk, certificate: cert, height: ... });
    }
}
```

Sequencer links chunks via the parent certificate. A chunk that doesn't
extend the current tip is rejected.

## Appendix D — The Equivocation prevention

From `ordered_broadcast/mod.rs:19-21`:

> Network participants persist any new nodes to a journal. This enables
> recovery from crashes and ensures that sequencers do not broadcast
> conflicting chunks and that validators do not sign them.

The validator's journal records every chunk it has signed. On restart:

```rust
fn on_restart(journal: Journal) {
    let replayed = journal.replay();

    for (sequencer, chunk, ack) in replayed {
        // Mark this (sequencer, height) as already signed
        self.signed.insert((sequencer, chunk.height), ack);

        // Reject any new chunk at this height from this sequencer
    }
}
```

If a sequencer tries to send two different chunks at the same height,
the validator:
1. Sees chunk A, signs it, journal-records the signature.
2. Sees chunk B at the same height. Looks up the journal: "I already
   signed A for this height." Refuses to sign B.
3. The chain forks; A is the canonical, B is rejected.

This works **across crashes**: even if you crash and restart, the
journal remembers your signature.

## Appendix E — The gossip protocol

The Engine gossips **chunk nodes** to peers:

```
Node {
    chunk: Chunk,
    parent_cert: Certificate,
    sequencer_signature: Signature,
}
```

A node is a chunk + its parent's certificate + the sequencer's signature
over the chunk. Validators verify:
1. Sequencer signature is valid (only the designated sequencer signs).
2. Parent certificate is valid (links to the chain).
3. Chunk builds on the parent's tip.

If valid, the validator signs and multicasts an Ack.

## Appendix F — The Chunk type

```rust
pub struct Chunk {
    pub sequencer: PK,
    pub height: Height,
    pub payload: Bytes,
    pub timestamp: SystemTime,
    pub namespace: Bytes,
}

pub struct Ack<S: Scheme> {
    pub height: Height,
    pub digest: Digest,        // hash of the chunk
    pub signer: PK,
    pub signature: S::Signature,
}

pub struct Certificate<S: Scheme> {
    pub height: Height,
    pub digest: Digest,
    pub certificate: S::Certificate,  // aggregated
}
```

The `Chunk` is the data. `Ack` is one validator's signature. `Certificate`
is the quorum-aggregated form.

## Appendix G — The Certificate construction

When `2f+1` acks arrive for `(sequencer, height)`:

```rust
fn assemble_certificate(sequencer: PK, height: Height, acks: BTreeMap<PK, Ack>) -> Certificate {
    Certificate {
        height,
        digest: acks.values().next().unwrap().digest,    // all must match
        certificate: S::assemble_certificate(acks.values().map(|a| &a.signature))?,
    }
}
```

All acks for the same `(height, digest)` must produce the same digest.
If they don't, you have equivocation — abort and report.

## Appendix H — Test patterns

```rust
fn test_single_sequencer() {
    // Setup: 1 sequencer, 4 validators
    // Sequencer broadcasts 10 chunks
    // Verify: each validator has 10 certificates
    // Verify: each validator has the chunks in order
}

fn test_sequencer_rotation() {
    // Setup: 4 validators, rotate sequencer every 5 chunks
    // Verify: chains continue across rotation
    // Verify: validators track each sequencer's chain separately
}

fn test_equivocation_prevention() {
    // Setup: 4 validators, 1 Byzantine sequencer
    // Sequencer broadcasts two different chunks at height 5
    // Verify: validators only sign the first one they see
    // Verify: chain continues from the first chunk, second is rejected
}
```

## Appendix I — Common gotchas

### Confusing height with sequencer-relative index

Each sequencer has its **own** chain. Sequencer A's chunk at "height 5"
is different from sequencer B's "height 5." Use `(sequencer, height)`
as the tuple key.

### The chunk digest must be unique

The digest is `hash(chunk)`. If two different chunks have the same
digest, you've got a hash collision. SHA-256 makes this infeasible,
but use `Digest` (which is sized for collision resistance).

### Missing the sequencer signature

A chunk without a sequencer signature is rejected. Don't skip it.

## Where to look in the code (expanded)

- `consensus/src/ordered_broadcast/mod.rs:1-52` — the architecture.
- `consensus/src/ordered_broadcast/engine.rs` — the main loop.
- `consensus/src/ordered_broadcast/ack_manager.rs` — ack tracking.
- `consensus/src/ordered_broadcast/tip_manager.rs` — tip tracking.
- `consensus/src/ordered_broadcast/scheme.rs` — signature schemes.
- `consensus/src/ordered_broadcast/types.rs` — message types.
- `consensus/src/ordered_broadcast/config.rs` — config knobs.

## If you only remember three things

1. **Sequencer broadcasts a linked chain of chunks; validators sign each chunk.** A certificate on chunk N lets the sequencer broadcast chunk N+1.
2. **No leader election, no view transitions, no nullifications.** Lighter than Simplex because the sequencer is known.
3. **Journals prevent equivocation across crashes.** Once you sign chunk N, no other chunk at height N ever gets signed — even if you crash and restart.

→ Next: **Chapter 15 — Actor**. Mailboxes, supervision, how all these
primitives communicate safely without deadlocking.