# Chapter 09 — Broadcast: wide-area data dissemination

> How to ship a big message to a lot of peers efficiently.

## The problem

Chapter 05 (P2P) lets you send a message to a set of peers. But if the
message is **big** (a block, a snapshot, a transaction batch), and you have
**many peers**, you have the same problem as chapter 07 — the leader's
egress becomes the bottleneck.

The naive broadcast:

```
Sender: 1 × big message → N-1 peers = N-1 × bandwidth
```

If N=100 and message=10MB, that's 990MB outbound. Same as the block
dissemination problem from chapter 07.

## The trick — coded broadcast

Open `broadcast/src/lib.rs:14-31`:

```rust
pub trait Broadcaster: Clone + Send + 'static {
    type Recipients;
    type Message: Codec + Clone + Send + 'static;
    fn broadcast(&self, recipients: Self::Recipients, message: Self::Message) -> Feedback;
}
```

The trait is **tiny** — one method. That's because the cleverness is all in
the implementation: `broadcast::buffered::Engine` uses erasure coding
(chapter 07) under the hood.

The pattern:

1. Sender encodes the message into `n` shards (chapter 07).
2. Sender sends each shard to a different peer.
3. Peers exchange shards among themselves (each peer keeps what they got
   from the sender, sends to others who need it).
4. Once a peer has `k` shards, it reconstructs the message.

This is the **Coded Broadcast** pattern. Each peer's egress drops from
`(N-1) × message` to `k × shard = message`. Total network bandwidth is
~`N × message` (the data itself), distributed evenly. **Optimal.**

## The `buffered::Engine`

Open `broadcast/src/buffered/mod.rs:1-23`:

> The core of the module is the [Engine]. It is responsible for:
> - Accepting and caching messages from other participants
> - Broadcasting messages to all peers
> - Serving cached messages on-demand

Three jobs:

1. **Receive + cache.** When a peer broadcasts a message, you cache it. The
   cache is bounded per-peer (`CACHE_SIZE` in tests, configurable in
   production). When the cache fills, the oldest message is evicted.

2. **Broadcast.** When you have a message to share, broadcast it (using
   coding).

3. **Serve.** When a peer fetches a message from you (via the resolver from
   chapter 08), check your cache and serve it.

## Why "buffered"?

Because the engine **buffers messages from peers** before you ask for them.
If peer X broadcasts 100 messages but you only ask for message 73, the
engine has been storing 1-72 and 74-100 the whole time (up to the cache
limit). You get message 73 immediately.

This is different from the "fetch on demand" model. The broadcast layer
**assumes you might want everything eventually** and pre-stages data in
the cache.

## Deduplication across peers

If peer A and peer B both broadcast the same message (because of
redundancy / coding), the engine deduplicates. A message referenced by
multiple senders stays cached until the last per-sender deque that
references it is evicted.

Storage cost: O(unique messages) not O(unique references). For consensus
this matters — if 67% of validators broadcast the same notarization, you
store it once, not 67 times.

## Peer set tracking

`buffered/mod.rs:21-24`:

> Only peers in `latest.primary` may buffer messages. When a peer is no
> longer in `latest.primary`, its buffered messages are evicted unless
> buffered by any other primary peer.

This is the active peer set from chapter 05. Only primary peers'
contributions count. Secondary peers (light clients, followers) get data
gossiped to them but don't contribute to the buffer.

When a peer is removed from primary, its cache entries are evicted (but
only if no other primary peer also referenced them — reference counting).

## The relationship to Simplex's `Marshal`

Chapter 12 covers Marshal in detail, but here's the preview:

Simplex gives you **certificates** (notarizations, finalizations) that
contain hashes. Marshal is the layer that turns hashes into downloaded,
verified, finalized blocks. The `broadcast::buffered::Engine` is the
default backing store for Marshal: when Marshal needs a block, it asks the
broadcast layer, which has been caching it from peers.

This is why broadcast lives at the substrate level: it's how all "give me
block X" requests flow.

## The `buffered::Config`

The config knobs (skim `broadcast/src/buffered/config.rs`):

- `cache_size` — how many messages to keep per peer
- `message_backlog` — how many pending messages can be queued for delivery
- `priority_proposals`, `priority_free` — priority flags for the mailbox

These tune the trade-off between "use lots of memory for fast access" and
"use less memory but slower access."

## Where to look in the code

- `broadcast/src/lib.rs:14-31` — the `Broadcaster` trait.
- `broadcast/src/buffered/mod.rs:1-24` — what `Engine` does.
- `broadcast/src/buffered/engine.rs` — the implementation.
- `broadcast/src/buffered/ingress.rs` — message handling.
- `docs/blogs/scaling-throughput-with-ordered-broadcast.html` — Brendan Chou's
  post on the design rationale.

## Epidemic algorithms — the math behind gossip

`buffered::Engine` is fundamentally a **gossip protocol**: messages
spread peer-to-peer, with each peer passing along what it has to
newcomers. The mathematical analysis of gossip predates Cassandra by
decades (Demers et al., "Epidemic Algorithms for Replicated Database
Maintenance", 1987) and the conclusions generalize to broadcast engines.

The two main flavors Commonware's approach blends:

**Anti-entropy (push-pull).** Every round (every message arrival, in
practice), a peer "exchanges state" with a randomly-chosen peer.
Both sides send their digests; both sides request messages the
other has that they don't. After `O(log N)` rounds, all peers
converge (each peer sees the full message set). This is **pull** —
you ask what the other has and fetch what you're missing.

**Rumor mongering.** Once a peer has new information, it pushes it
to `k` other peers (the "fan-out factor"). After `O(log N)` rounds
on average, all peers have heard. This is **push** — you send
without being asked.

**Push-pull beats either alone.** Pure push underperforms at the
tail — late-arriving peers never get the message. Pure pull underperforms
in the head — initial dissemination is slow. Push-pull combines
push's propagation with pull's tail-catchup.

The broadcast engine blurs these two. The `deque_size` cap
mechanically limits how much each peer is willing to "remember" for
gossip; the reference counts (`buffered::Engine::counts`)
deduplicate so the *effective* fan-out is per-message, not per-peer.

**The math.** With fan-out `k`, the fraction of peers that have
heard a rumor by round `r` is roughly:

```
fraction(r) ≈ 1 - (1 - k/N)^r  (push, well-mixed)
log₁/(₁₋ₖ/ₙ)(N/N₀) ≈ 7-10 rounds (push only, 100 nodes, k=2)
```

For 100 validators and `k = 2`, push alone takes ~7 rounds to reach
99%. Push-pull halves that. Commonware doesn't sit on a randomized
peering model — it's a fixed peer set — but the math explains why
gossip is "good enough" for low-stakes messages and why
quorum-reliable broadcast (chapter 11) is needed for consensus
messages.

**Rumors and "birthday paradox" effects.** In rumor mongering with
random peering, two messages launched simultaneously in small
networks can collide — one might never propagate. With 100 nodes
and fan-out 2, the probability of a message living only one round is
`((N-k)/N)^(N-k) ≈ 23%` — surprisingly high. This is why
consensus messages use deterministic peer sets (chapter 11
certificates), not pure gossip.

## Plausible eventual consistency

The gossip model gives you a different kind of guarantee than a
totally-ordered broadcast. Demers et al. called it **plausible
eventual consistency**:

> If gossip keeps running long enough, eventually all nodes converge
> to the same state.

The word "plausible" is doing a lot of work. The property holds with
high probability but not with certainty. A separator that prevents
gossip forever (network partition that never heals) breaks the
property. The *eventual* part means there's no time bound.

`buffered::Engine` is stronger than this in two ways:

1. **Bounded peer set.** For an `N`-node consensus cluster, gossip
   converges in `O(log N)` rounds deterministically (well, modulo
   packet loss). No probability in the guarantee.
2. **Reference counts.** When a peer's deque drops the message, the
   message *might still be cacheable* if another peer still has it.
   So the "liveness" of a message is its reference count, not its
   first-arrival ordering.

But for the message-stability story as seen by the application, the
right mental model is "plausible eventual" with a fast tail. In
adversarial networks with packet loss (chapter 05 P2P), the engine
might temporarily have a peer that has a message while another peer
doesn't — that's fine, because consensus messages have *additional*
delivery guarantees (Quorum intersection in chapter 01) that broadcast
alone doesn't provide.

## The buffered engine's reference counting

The reference-counted dedup in `broadcast::buffered::Engine` is the
secret sauce that lets the engine handle redundancy cheaply. Let's
unpack it formally.

Two data structures cooperate (`broadcast/src/buffered/engine.rs:84-99`):

```rust
struct Engine<...> {
    /// All cached messages by digest.
    items: BTreeMap<M::Digest, M>,

    /// Per-peer LRU of digests (at most deque_size per peer).
    deques: BTreeMap<P, VecDeque<M::Digest>>,

    /// Number of peer's deques containing each digest.
    counts: BTreeMap<M::Digest, usize>,
}
```

When a message `m` arrives from peer `X`:

```
1. items[m.digest()] = m      // (or no-op if already present)
2. counts[m.digest()] += 1
3. deques[X].push_back(m.digest())
4. If deques[X].len() > deque_size:
     dropped = deques[X].pop_front()
     counts[dropped] -= 1
     if counts[dropped] == 0:
         items.remove(dropped)  // last reference, drop the bytes
```

The invariant:

> **`items` contains a digest iff some peer's deque currently references
> it** (`counts[digest] > 0`).

This is reference counting via a `BTreeMap`, not via `Rc`/`Arc`. Why?
Three reasons:

1. **`Rc`/`Arc` doesn't survive threads.** You'd need `Arc`, but then
   you'd also need a `Mutex` around the items map to mutate, which
   serializes every read.
2. **The data is stored once anyway.** The whole point is to *avoid*
   cloning the bytes. `Arc<M>` works but the indirection adds latency.
3. **Counts let you know when memory is truly free.** A peer that
   drops a message doesn't actually free the memory until all other
   peers drop their references, too.

The same trick handles peer set transitions (below) and message
eviction under pressure (any kind of eviction that decrements a
reference).

There's a subtle wrinkle: what if the same digest *with different
content* arrives? This shouldn't happen if `Digestible` is genuinely
content-derived (chapter 04), but a Byzantine peer could try. The
engine treats the first-arrival's bytes as canonical; subsequent
arrivals with the same digest are accepted (incrementing the count)
but the bytes don't change. The downstream protocol must verify the
content matches the digest on its own — which Marshal (chapter 12)
does with `CertifiableAutomaton::certify` plus the broadcast
byte-for-byte equality check.

## Priority flag — when latency matters more than throughput

The `broadcast::buffered::Mailbox::broadcast(...)` takes a `priority`
boolean. Behind the scenes, that flag floats through to two layers:

1. **The P2P send loop** (`p2p` chapter 05). Priority messages are
   placed in a separate queue that is drained ahead of normal-priority
   messages.
2. **The mailbox itself** (`broadcast::buffered/engine.rs:60-64`). The
   `priority` field controls whether the mailbox yield/check is
   eager or lazy.

The high-level question: **when should you mark a broadcast as
priority?**

| Use case | Priority? | Why |
|---|---|---|
| Block proposal | Yes | Time-sensitive; peers need to vote soon |
| Finalization cert | Yes | Same reason |
| Gossip (catch-up) | No | Best-effort; failure isn't catastrophic |
| Large snapshot | No | Bulk transport; latency doesn't matter |
| Pruning batch | No | Idempotent, low urgency |

The trade-off: priority messages can starve normal messages if you
spam them. P2P quotas (chapter 05) impose rate caps — priority
*jumps the queue*, it doesn't *bypass* the quota.

A subtle point: priority is local to your sender, not the network.
"Your peer votes on time" doesn't mean "their queue puts it first
too." But for time-critical consensus messages, your local
priority-bump is the main lever you have.

## Peer set transitions — buffering through churn

Consensus validator sets rotate (epoch changes, chapter 01). When
they do, `buffered::Engine` must handle two things:

1. **Old peers' contributions.** If validator X used to be primary but
   is now demoted to secondary (or removed), their buffered
   messages should no longer be canonical. The engine's contract
   (`broadcast/src/buffered/mod.rs:21-24`):

   > Only peers in `latest.primary` may buffer messages. When a peer
   > is no longer in `latest.primary`, its buffered messages are
   > evicted unless buffered by any other primary peer.

2. **In-flight broadcasts.** A message you sent to the old peer set
   may still be in transit. The engine discards pending references
   to demoted peers; the message stays cached as long as some
   *current primary* peer still references it.

Reference counting makes both clean: a demoted peer's deque is
removed, its references decrement, and the underlying items
disappear when the last reference is gone. This is the same code
path as ordinary eviction (previous section), driven by a different
event (`peer_provider` updates vs. a deque overflow).

There's a delay window: between `track(N, new_set)` and your next
broadcast, the engine is reading from the *old* primary set via
`latest.primary`. The P2P layer's `Provider::track` updates trigger a
notification, and the engine processes it on its next mailbox
event. Tests like `test_peer_set_update_evicts_disconnected_peer_buffers`
verify this transition is correct (`broadcast/src/buffered/tests.rs`).

For an in-flight broadcast *broadcast*, the engine code does:

```rust
fn on_peer_set_change(new_primary: &Set<PublicKey>) {
    let old_primary = std::mem::replace(&mut self.primary, new_primary.clone());
    for peer in old_primary.difference(&new_primary) {
        if let Some(deque) = self.deques.remove(peer) {
            for digest in deque {
                let entry = self.counts.entry(digest).and_modify(|c| *c -= 1);
                if entry.or_insert(0) == &mut 0 {
                    self.items.remove(&digest);
                }
            }
        }
    }
}
```

This is O(K log K) where K = the old peer's deque size.

## Bandwidth model — why Commonware simulates, not just measures

Consensus is a real-time system. You don't get to "wait until network
conditions improve." That makes bandwidth a first-class concern. Commonware's
`p2p::simulated::Network` doesn't just inject latency; it enforces a
**bandwidth budget per link**, like a real WAN connection.

The simulation knobs are in `p2p/src/simulated/` (chapter 05):

```rust
struct Link {
    latency: Duration,
    jitter: Duration,
    success_rate: f64,    // packet loss 0..1
    bandwidth: ???        // bytes per second
}
```

When a message of `B` bytes is sent over a link with bandwidth `bw`,
the simulated arrival time is `latency + B/bw`. For a 1 MB block
across a 100 Mbps link, that's 80 ms of *transfer time* on top of
the latency — realistic for a wide-area network.

**Why this matters for broadcast engines.** A naive broadcast
(`O(N)` egress per sender) saturates the sender's uplink with N=100
peers and 1 MB blocks. With bandwidth simulation, you'd see the
sender's queue back up, the simulated time jump forward, and the
engine start dropping messages at the application layer. The
broadcast engine with coding (chapter 07) keeps egress at `O(k/N)
× message_size = O(1) × message_size` per sender, which fits
comfortably in any realistic uplink.

**Max-min fairness.** A real network is shared. Commonware's P2P
quotas (`Quota::per_second(N)`) implement **max-min fairness**:
each peer gets an equal share of bandwidth up to their quota; if
a peer doesn't use theirs, it doesn't get distributed to others
(sticky allocations). This is the standard fair-queueing policy
at L4/L7 routers and a good default for consensus — you don't
want one chatty peer to starve out the actual votes.

For consensus work specifically, the engine optimizes for:
- Sender egress bounded (chapter 07's coding approach).
- Receiver bandwidth bounded by `peers × per_peer_bandwidth`.
- Time-to-deliver deterministic (engine doesn't back off on slow
  peers; it tries harder).

## HTTP/3 datagrams and QUIC unreliability

`broadcast::buffered::Engine` runs over the P2P layer (chapter 05),
which itself sits on top of the `stream` crate (QUIC). A natural
question: **why not send broadcast messages as QUIC streams (TCP-like
reliable) instead of datagrams (UDP-like unreliable)?**

The answer depends on the layer:

| Layer | Reliability choice |
|---|---|
| Transport (QUIC) | Reliable streams + unreliable datagrams (RFC 9221) |
| P2P (chapter 05) | Streams for ordered multi-message flows; messages for one-shot sends |
| Broadcast | The engine itself provides retries on top of unreliable delivery |

For consensus broadcast, the broadcast engine *wants* unreliable
delivery plus a retry layer above it. That's exactly what
epidemic replication provides (above). If the transport were always
reliable, you'd waste bandwidth on retransmissions of messages
that have already arrived at the recipient (over-counted in your
"delivery guarantees"). With unreliable + gossip retries, the
end-to-end behavior is reliable-enough and the bandwidth cost
is bounded.

QUIC's **datagram extension** (RFC 9221) gives exactly this
primitive: send a packet, no retransmit on loss. Commonware's
stream crate uses it for one-shot messages. Without datagram mode,
QUIC would force every send to be a stream, dragging along
head-of-line blocking and retransmission logic.

For broadcast specifically, datagram mode means:

```
send(m, priority) → QUIC datagram → peer → process(m)
```

Three properties (per Kurose & Ross, chapter 4):

1. **No retransmission** — lost datagrams are lost. The peer-broadcast
   layer above handles retries.
2. **No ordering** — datagram arrival order is not preserved.
   The broadcast engine doesn't need ordering (certificates do, in
   chapter 11).
3. **No congestion control** (in datagram mode) — QUIC's DCB-style
   congestion control doesn't apply. Your application sets the
   pacing.

For high-throughput finality machinery, this is the right choice.
For genuinely loss-tolerant workloads (gossip without consensus),
it's even better.

## Rust patterns in the broadcast engine

Three Rust idioms specific to broadcast:

**`mpsc::channel` for fan-out.**
The broadcast engine receives one `Broadcast(M)` from a caller
and turns it into N network sends. The actor mailbox models this
internally:

```rust
let (broadcast_tx, broadcast_rx) = mailbox::bounded(N);
// caller side:
broadcast_tx.broadcast(recipients, msg).await?;
// engine loop:
loop {
    let msg = broadcast_rx.recv().await?;
    match msg {
        Broadcast(m) => self.fan_out(m).await,
        Get(key) => self.serve(key).await,
        // ...
    }
}
```

The fan-out happens asynchronously inside `fan_out`, so the
caller doesn't block waiting for N TCP sends to drain.

**`Arc` for shared payloads.**
When the engine receives message M from peer X and decides to
cache it for Y, the bytes themselves are shared. The standard
trick: `Bytes` (the `bytes` crate) for zero-copy slices.

```rust
type Message = Bytes;  // shared, cheaply cloneable
// or for richer messages:
struct Message {
    header: Header,                    // small, copied
    body: Arc<[u8]>,                   // shared buffer
}
```

`Bytes` keeps a refcount inside the slice itself; clone of a `Bytes`
of length `n` is `O(1)` regardless of `n`. This is the common
zero-copy pattern for network buffers in Commonware
(`broadcast/src/buffered/engine.rs` uses `Codec` for encoding,
which produces `Bytes`).

**`BTreeMap` for ordered iteration.**
The engine uses `BTreeMap<M::Digest, M>` for `items` and
`BTreeMap<P, VecDeque<M::Digest>>` for `deques`
(`broadcast/src/buffered/engine.rs:85, 92`). Why BTreeMap (sorted)
not HashMap (random)?

Three reasons:

1. **Deterministic iteration** — the runtime auditor (chapter 02)
   hashes every event in iteration order. `HashMap` order varies
   between runs; `BTreeMap` doesn't.
2. **Reproducible tests** — `test_ref_count_across_peers` and
   friends assert exact membership. Sorted iteration makes those
   tests robust.
3. **Compression-friendly serialization** — the conformance harness
   (`conformance` crate) re-encodes engine state to verify format
   stability; sorted keys produce stable output.

The cost is `O(log N)` per lookup vs. `O(1)` for `HashMap`. For
the broadcast engine's workload (a few hundred messages in flight
per peer), the difference doesn't matter.

## Epidemic algorithms in full — Demers et al. and the math

The epidemic section above gave the intuition. Let me unpack the
formal model from Demers et al., "Epidemic Algorithms for Replicated
Database Maintenance" (1987) — the paper that started it all.

### The model

A replicated database has `N` sites. Each site has the full database
(or a relevant subset). The sites gossip about updates.

Three flavors of gossip:

1. **Anti-entropy (push-pull)**: every site periodically picks a
   random peer and exchanges state. Both sides send their digests,
   both sides request missing updates.
2. **Rumor mongering (push only)**: a site with a new update pushes
   it to `k` random peers. Those peers push to `k` more, etc.
3. **Pure pull**: a site polls `k` random peers for updates. (Less
   common in practice.)

Demers et al. showed that anti-entropy with push-pull converges
**fastest**, while push-only is simpler but slower.

### The math of how epidemics spread

Define `s(t)` = fraction of sites that have heard a particular update
by time `t`. The differential equation for push-only:

```
ds/dt = (1 - s) × k/N × β
```

where `k` is the fan-out, `N` is the total sites, and `β` is the
gossip rate per site.

Solution:

```
s(t) = 1 - e^(-βkt/N)
```

For `N = 100, k = 2, β = 1` (one gossip per second):

```
s(1s) = 1 - e^(-2/100) ≈ 0.0198  (2% have heard)
s(7s) = 1 - e^(-14/100) ≈ 0.13    (13%)
s(50s) = 1 - e^(-100/100) ≈ 0.63  (63%)
s(100s) = 1 - e^(-200/100) ≈ 0.86 (86%)
s(500s) = 1 - e^(-1000/100) ≈ 0.99995 (99.995%)
```

So push-only takes ~500 seconds (8 minutes) to reach 99.995% of
100 sites. Too slow for consensus.

### Push-pull is faster

With push-pull, both sides exchange state. The equation becomes:

```
ds/dt = (1 - s) × (2βk/N) × s
```

(Same as before, but the rate doubles because both sides send, AND
there's an `s` factor because pulling is more useful when more sites
have the update.)

Solution:

```
s(t) = e^(2βkt²/N) / (1 + e^(2βkt²/N))
```

Wait, that's not right. Let me redo this.

The push-pull model (from Demers et al.) is:

```
ds/dt = β × (1 - s) × (k/N) × s × 2
```

Where the `s` factor captures the value of pull (when only 1 site has
the update, pull is useless; when 50 sites have it, pull is very
useful).

Solving: `s(t) = e^(βkt²/N) / (1 + e^(βkt²/N))` (logistic curve).

For `N = 100, k = 2, β = 1`:

```
s(1s) = 0.50
s(2s) = 0.88
s(3s) = 0.98
s(5s) = 0.9997
```

So push-pull reaches 99.97% in 5 seconds vs. 500 seconds for push-only.
**100x speedup**.

### The "tail" problem

Even with push-pull, gossip has a long tail. After 5 seconds, 99.97%
have heard, but 0.03% (3 out of 10000 sites) haven't. Those last
sites may never hear the rumor — they've been "missed" by the random
peering.

Demers et al. called this the **permanent gap**. For replicated
databases, the solution is to **layer multiple gossip protocols**:
a fast push-pull for the bulk of dissemination, plus a slow, broader
push for the tail.

Commonware's broadcast engine doesn't have this two-layer design
(it's a fixed peer set, not an open system), but the principle
applies: if you need 100% dissemination, run gossip long enough.

### Rumor mongering with adversarial peers

The "rumor mongering" model assumes all peers are honest. What if
some are Byzantine?

A Byzantine peer can:
- **Drop rumors**: refuse to forward to others. Reduces dissemination.
- **Forward fake rumors**: send fake updates. Other peers accept
  without verification.

Defenses:
- **Cryptographic signatures** on updates (chapter 04). Peers
  verify signatures before forwarding.
- **Quorum intersection** for consensus: even if some peers drop,
  the gossip reaches enough to form a quorum.
- **Detecting droppers**: track which peers have forwarded each
  update; mark non-forwarders as suspicious.

Commonware's broadcast engine assumes an authenticated peer set
(chapter 05), so fake rumors are caught at the cryptographic layer.
Droppers are detected by missing-data patterns but not actively
punished (the consensus protocol handles that via nullification).

### The birthday paradox in gossip

The "rumors and birthday paradox effects" section above noted that
two messages launched simultaneously can collide. Let me make this
concrete.

In a 100-node network with fan-out 2, the probability that a message
lives exactly one round is:

```
P(message lives 1 round) = ((N - k) / N)^(N - k)
                        = (98/100)^98
                        ≈ 0.144
```

So ~14% of messages die after one round. The probability of surviving
`r` rounds is `(k/N)^r` — geometric decay.

For **two** messages launched simultaneously, the probability that
they collide (i.e., neither survives beyond one round because they
"steal" each other's forwarders) is non-trivial. Roughly:

```
P(both die) ≈ 0.144^2 × something
```

The exact math is intricate (see Demers et al. Section 4), but the
take-away: simultaneous messages can collide. For consensus, this
means **deterministic peer selection** (chapter 11 certificates) is
needed for high-stakes messages.

### Epidemic algorithms in modern systems

The epidemic model has been used in:

- **Cassandra** (2008): anti-entropy for cluster-wide state
  propagation.
- **DynamoDB** (2007): gossip for membership and failure detection.
- **Riak**: similar to DynamoDB.
- **Bitcoin** (2008): block propagation via gossip (with some
  optimizations like Compact Blocks).
- **Ethereum** (2015): gossip for transactions and blocks.

For BFT consensus specifically, gossip is used for:

- **Pre-consensus data dissemination**: the broadcast engine.
- **Post-finality sync**: peers catching up on historical blocks.
- **Peer discovery**: gossip for membership.

Commonware's broadcast engine is the pre-consensus variant.

## Gossip protocols in distributed databases — DynamoDB and Cassandra

The gossip section above mentioned DynamoDB and Cassandra. Let me
unpack what they actually do.

### DynamoDB (DeCandia et al., 2007)

Amazon's DynamoDB uses gossip for two things:

1. **Membership**: each node gossips its view of the cluster
   membership every second. After a few gossip rounds, all nodes
   converge on the same membership list.
2. **Failure detection**: indirect gossip ("node X says node Y is
   down"). After enough reports, node Y is marked failed.

The gossip protocol:

```
gossip_round():
    peer = random_alive_peer()
    send peer: local_membership_view
    receive peer: peer_membership_view
    merge views
```

Each node maintains a "membership view" with versioning (so updates
propagate correctly even if the gossip is out-of-order).

For consensus: DynamoDB doesn't use BFT consensus (it uses eventual
consistency + quorum reads/writes). But the gossip layer is similar
to what Commonware uses for peer discovery.

### Cassandra (Lakshman & Malik, 2010)

Cassandra's gossip is more sophisticated. Each node maintains:

- **Endpoint state**: the node's address, generation, heartbeat.
- **Application state**: per-keyspace state (schema, tokens, etc.).
- **Failure detection**: phi accrual failure detector (a probabilistic
  failure detector that outputs a "suspicion level" instead of a
  binary up/down).

The gossip protocol uses three message types:

- **`GossipDigestSyn`**: a node initiates a gossip round.
- **`GossipDigestAck`**: the responder sends a digest of its state.
- **`GossipDigestAck2`**: the initiator sends the full state for any
  divergent entries.

This three-message handshake is similar to Kademlia's `PING` /
`PONG` / `STORE` sequence.

Cassandra's gossip runs **continuously** (every second), unlike
Dynamo's periodic rounds. This gives faster convergence but more
overhead.

### Failure detection — phi accrual

Cassandra's phi accrual failure detector (Hayashibara et al., 2004)
is a more nuanced alternative to heartbeat-based detection.

The idea: instead of "node X is up if it sent a heartbeat in the
last 5 seconds," use a probabilistic model.

For each node Y, maintain a sliding window of inter-arrival times
of heartbeats. Compute:

```
phi = -log10(P_later(heartbeat))
```

where `P_later` is the probability of a heartbeat arriving later
than the time elapsed since the last heartbeat.

Higher phi = more suspicion. Threshold at phi = 8 (i.e., the heartbeat
is 10^8 times more delayed than expected) marks the node as failed.

For BFT: Commonware uses simpler timeouts (chapter 11). The
benefit of phi accrual is more nuanced failure detection; the cost
is complexity.

### The eventual consistency model

Cassandra and DynamoDB both provide **eventual consistency**:

> If no new updates are made to a key, eventually all accesses to
> that key will return the last updated value.

This is weaker than linearizability or serializability. It allows
the system to remain available during partitions (CAP theorem's
"A" in the AP direction).

Commonware's broadcast engine is "plausible eventual consistency"
(Demers et al.) — even weaker, because gossip doesn't guarantee
all peers converge in a bounded time.

For BFT consensus, neither eventual nor plausible eventual is
sufficient. Consensus requires **strong consistency** (every honest
node agrees on the order of finalized blocks). The gossip layer
helps with pre-consensus data dissemination, but the consensus
protocol itself is what guarantees consistency.

### Comparison: DynamoDB gossip vs Commonware broadcast

| Property | DynamoDB | Cassandra | Commonware broadcast |
|---|---|---|---|
| Purpose | Membership + failure detection | Same, plus schema | Data dissemination |
| Frequency | 1s/round | 1s/round continuous | On-message |
| Peer set | Fixed (within data center) | Fixed (within cluster) | Validators |
| Security | Authenticated | Authenticated | Authenticated |
| Consistency | Eventual | Eventual | Plausible eventual |

The Commonware broadcast engine is closer to Cassandra's gossip in
spirit but specialized for block dissemination.

## The buffered Engine in full

The reference-counting section above described the structure. Let me
dig into more details: the priority flag's interaction with the
mailbox, the deduplication algorithm in detail, and the peer set
transition logic.

### The reference counting model, in detail

The engine uses three BTreeMaps (chapter 09 main text):

```rust
struct Engine<...> {
    items: BTreeMap<M::Digest, M>,           // all cached messages
    deques: BTreeMap<P, VecDeque<M::Digest>>, // per-peer LRU
    counts: BTreeMap<M::Digest, usize>,        // refcount per message
}
```

The invariant:

> `items` contains a digest iff some peer's deque references it
> (`counts[digest] > 0`).

When `on_message(m, from)`:

```
1. items[m.digest()] = m      // or no-op if already present
2. counts[m.digest()] += 1
3. deques[from].push_back(m.digest())
4. If deques[from].len() > deque_size:
      dropped = deques[from].pop_front()
      counts[dropped] -= 1
      if counts[dropped] == 0:
          items.remove(dropped)
```

Three subtleties:

**Subtlety 1: what if `items` already has `m.digest()`?** The
`items.insert(m)` is a no-op; the `m` we have is canonical (first
arrival). Subsequent arrivals with the same digest but different
content are **silently ignored**. The downstream protocol must
verify content matches digest (Marshal's `CertifiableAutomaton`
does this; chapter 12).

**Subtlety 2: what if `counts[digest]` was 0 and now becomes 1?**
This shouldn't happen via the `on_message` path (we increment from
0 to 1 when adding the message). But it can happen via... actually,
no: `counts[digest]` is incremented only when `items[m.digest()]`
is already set or just got set. So counts and items stay in sync.

**Subtlety 3: what about race conditions?** The engine is an actor
(chapter 15); all access is serialized via the mailbox. No races.

### Eviction policy in detail

The per-peer deque is a FIFO ring buffer. When `deques[from].len()
> deque_size`:

```
dropped = deques[from].pop_front()
counts[dropped] -= 1
if counts[dropped] == 0:
    items.remove(dropped)
```

The "dropped" message might still be in another peer's deque
(`counts[dropped] > 0` before decrement). If so, the message stays
in `items` but is no longer in `from`'s deque.

This is **reference-counted eviction**: a message stays as long as
anyone references it.

### Priority flag in detail

The `priority` flag on `Mailbox::broadcast` does two things:

1. **P2P send priority** (chapter 05): priority messages are placed
   in a separate queue that's drained ahead of normal messages.
2. **Mailbox processing priority**: priority messages are processed
   eagerly (without yielding to other mailbox events).

The second is the key for the broadcast engine: a priority broadcast
gets the engine's attention **before** other events. A normal-priority
broadcast may wait if there's a backlog of higher-priority events.

When to use priority:

| Use case | Priority? | Why |
|---|---|---|
| Block proposal | Yes | Time-sensitive |
| Finalization cert | Yes | Time-sensitive |
| Gossip catch-up | No | Best-effort |
| Snapshot | No | Bulk transport |
| Pruning batch | No | Idempotent, low urgency |

The cost: priority messages can starve normal messages if abused.
The P2P layer's quota system prevents priority from bypassing
network rate limits (priority jumps the queue, doesn't bypass the
quota).

### Peer set transitions in detail

When the active peer set changes (e.g., validator rotation):

```
1. New primary set P' received from peer_provider.
2. For each peer p in old primary P - P':
       deques.remove(p)   // p is no longer primary
       for digest in deques[p]:
           counts[digest] -= 1
           if counts[digest] == 0:
               items.remove(digest)
3. Update self.primary = P'.
```

The transition is atomic from the engine's perspective (single actor
mailbox event).

The delay window: between `track(N, P')` and the next mailbox event
processing the new set, the engine uses the **old** primary set. This
is by design — the P2P layer notifies the engine asynchronously, and
the engine processes notifications on its mailbox schedule.

For correctness: a message from a peer that's about to be demoted is
still accepted; when the demotion is processed, the message gets
evicted if no other primary peer has it.

### Interaction with peer set size

The engine's memory usage is `O(cache_size × N)` where `N` is the
number of primary peers. For `N = 100, cache_size = 100`, that's
10,000 message slots. At 1 KB each, 10 MB. Reasonable.

When `N` grows (e.g., during validator rotation), memory pressure
increases. Commonware's engine doesn't dynamically resize `cache_size`
— it's a config knob. Operators tune it based on expected `N`.

A subtler issue: if `N` shrinks (some peers leave), the engine's
cache size per peer stays the same. Memory usage might be over-
provisioned. Not a correctness issue, just inefficiency.

### The deduplication algorithm, in detail

When a message `m` arrives from peer X:

```
fn on_message(m: M, from: P) {
    let digest = m.digest();

    // Step 1: ensure m is in items
    if !self.items.contains_key(&digest) {
        self.items.insert(digest, m);
    }
    // else: items already has m; the existing m is canonical

    // Step 2: increment refcount
    *self.counts.entry(digest).or_insert(0) += 1;

    // Step 3: add to peer X's deque
    let deque = self.deques.entry(from.clone()).or_insert_with(VecDeque::new);
    deque.push_back(digest);

    // Step 4: enforce size limit
    if deque.len() > self.deque_size {
        let dropped = deque.pop_front().unwrap();
        let count = self.counts.entry(dropped).or_insert(0);
        *count -= 1;
        if *count == 0 {
            self.counts.remove(&dropped);
            self.items.remove(&dropped);
        }
    }
}
```

The dedup property: if two peers send the same message, the message
is in `items` once but referenced from both `deques[X]` and
`deques[Y]`. Eviction from one doesn't drop the message.

### Time complexity analysis

- **`on_message`**: `O(log N + log M)` per insertion (`log N` for the
  peer map, `log M` for the digest map). `M` is the number of unique
  cached messages.
- **Eviction**: `O(log M)` per eviction.
- **Lookup by digest**: `O(log M)`.
- **Serve request**: `O(log M)` (look up digest in `items`).

For `N = 100, M = 1000`, this is ~30 ops per operation. Fast.

### Common pitfalls

1. **Forgetting to update counts on peer removal.** If you remove a
   peer from `deques` without iterating the digests, you leak
   refcounts.
2. **Inserting before checking `items`**. Always insert in `items`
   before incrementing `counts`. Otherwise the invariant breaks.
3. **Concurrent access without locking.** The engine is an actor;
   no concurrent access. If you refactor to share state, add a lock.

## HTTP/3 and QUIC in depth

The HTTP/3 section above introduced QUIC datagram mode. Let me dig
deeper.

### RFC 9000 — QUIC, the transport

QUIC is a UDP-based reliable transport. Key features:

1. **Stream multiplexing**: many independent streams over one
   connection, no head-of-line blocking between streams.
2. **Built-in TLS 1.3**: encryption is part of the protocol, not
   layered on top.
3. **0-RTT handshake**: clients can send data on the first round
   trip (with caveats).
4. **Connection migration**: a connection can survive the client
   changing IP addresses (e.g., switching from Wi-Fi to cellular).
5. **Congestion control**: pluggable (Cubic, BBR, etc.).

For BFT: QUIC is the right transport for inter-validator
communication. The multiplexing lets us run control (certificates)
and data (blocks) over the same connection.

### RFC 9221 — HTTP/3 datagrams

HTTP/3 (RFC 9114) builds on QUIC. RFC 9221 adds **HTTP/3 datagrams**:
the ability to send unreliable datagrams within an HTTP/3 connection.

Why unreliable datagrams? HTTP/3 inherits HTTP's request/response
model, which is reliable. But for real-time or broadcast traffic,
reliability is wasteful (retransmitting a message everyone already
has).

The datagram extension (RFC 9221) gives exactly this: send a packet,
no retransmit on loss. The application layer handles retries if
needed.

### Commonware's use of QUIC datagrams

Commonware's stream crate (used by the P2P layer) uses QUIC streams
for reliable communication and QUIC datagrams for unreliable one-shot
messages.

The P2P layer (chapter 05) uses:
- **Streams**: for multi-message flows (e.g., handshake, sync).
- **Datagrams**: for broadcast (chapter 09), since broadcasts don't
  need reliability beyond what the gossip layer provides.

A broadcast over reliable streams would force retransmissions on
loss, even if the recipient has the message from another peer. With
datagrams, lost messages are lost, and the gossip layer fills in
the gaps via retries.

### Why datagrams matter for gossip

The gossip layer's semantics:

- Sender pushes a message to N peers.
- Each peer independently forwards to K others.
- Eventually all peers have the message.

With reliable transport:
- If peer X misses the message, the sender retransmits.
- Peer X is now "ahead" of peers that received the first try.
- Network bandwidth is wasted on retransmits.

With unreliable transport + gossip:
- Sender pushes to N peers once.
- Each peer independently retries (e.g., via its own fan-out).
- Network bandwidth is used once per peer per hop.

The math: with reliable transport and 5% packet loss, the sender
sends `N × 1.05` messages (on average). With unreliable + gossip,
the sender sends `N` and the recipients fill in the gaps.

For Commonware's broadcast engine: unreliable datagrams + gossip
retries is the right model. The gossip layer provides end-to-end
reliability; the transport provides per-packet unreliability.

### TCP vs QUIC for broadcast

| Property | TCP | QUIC |
|---|---|---|
| Multiplexing | No (head-of-line blocking) | Yes (per-stream) |
| Encryption | Optional (TLS overlay) | Built-in |
| Connection setup | 1-RTT + TLS handshake | 0-1 RTT |
| Reliability | Always | Stream: yes; datagram: no |
| Mobile-friendly | No (no migration) | Yes (connection migration) |
| Deployment | Ubiquitous | Growing (HTTP/3, MASQUE) |

For broadcast: TCP forces reliability, which is the wrong semantic.
QUIC's datagram mode is better.

For control plane (certificates, votes): TCP-style reliability via
QUIC streams is fine — these messages need guaranteed delivery.

Commonware uses both: QUIC streams for control, QUIC datagrams for
data.

### Latency considerations

QUIC's connection setup is faster than TCP + TLS (0-RTT in some
cases). For BFT, where validators maintain persistent connections,
this matters less. But for first-time connections (e.g., new
validator joining), it's a win.

QUIC's stream multiplexing also helps latency: opening a new stream
is `O(1)` (no system call, no handshake), vs. opening a new TCP
connection which is `O(RTT)`.

For broadcast, latency is dominated by data transfer time, not
handshake. So QUIC's connection-setup win is small here.

### Bandwidth considerations

QUIC's overhead per packet: ~30-40 bytes (vs. TCP's ~20 bytes). For
small messages, this overhead matters. For large blocks (10 MB),
it's negligible.

For BFT block dissemination: large messages, low overhead impact.
QUIC is fine.

### HTTP/3 vs raw QUIC

Commonware uses raw QUIC (via the `stream` crate), not HTTP/3. HTTP/3
adds request/response framing, headers, and content negotiation —
none of which are needed for BFT.

If you wanted to expose the broadcast engine to web clients (e.g.,
for a public RPC endpoint), HTTP/3 would be useful. For inter-
validator communication, raw QUIC is leaner.

### Where to look for QUIC details

- **RFC 9000**: QUIC transport.
- **RFC 9001**: QUIC + TLS 1.3.
- **RFC 9221**: HTTP/3 datagrams.
- **RFC 9298**: HTTP/3 connection carriage.

Commonware's `stream` crate implements the QUIC subset Commonware
needs: reliable streams + unreliable datagrams.

## Bandwidth modeling in depth

The bandwidth section above introduced the concepts. Let me dive
deeper.

### Max-min fairness, formally

Max-min fairness allocates bandwidth to flows such that:

1. No flow gets more than its demand.
2. Any flow that gets less than its demand can only do so because
   all other flows also get less than their demands.

Mathematically, an allocation `(x_1, ..., x_n)` is max-min fair if
for every `i`:

```
x_i ≤ demand_i
```

and for every `j ≠ i`:

```
if x_j < demand_j:
    x_i ≥ x_j
```

(If peer j is bandwidth-limited, peer i shouldn't be able to get
more than j.)

The standard algorithm: water-filling. Start with all flows at 0;
increase all flows at the same rate until one hits its share; freeze
that flow; continue with the rest.

For Commonware's P2P quotas: `Quota::per_second(N)` gives each peer
a per-second rate cap. If a peer doesn't use its share, the share
isn't distributed (sticky allocation).

### Why Commonware simulates, not just measures

Real-world testing of BFT consensus is expensive:
- Spinning up 100 validators takes real hardware.
- Network conditions (latency, loss) are hard to reproduce.
- Bugs that only appear under load are missed.

Simulation (chapter 02's deterministic executor) gives:
- **Reproducibility**: same seed, same result, every run.
- **Speed**: simulation runs faster than real-time (often 10-100x).
- **Control**: inject faults (drop, delay, partition) at will.
- **Coverage**: explore parameter space systematically.

For bandwidth: the simulator needs an accurate model of the network,
or tests will miss real-world issues.

### The simulated network model

Commonware's `p2p::simulated::Network` models:

- **Latency**: per-link, configurable (e.g., 50 ms cross-region).
- **Jitter**: ±10 ms uniform random.
- **Loss**: configurable drop rate (e.g., 0.1% per link).
- **Bandwidth**: bytes per second per link (e.g., 100 Mbps).
- **Reordering**: configurable reorder probability.

When a message of `B` bytes is sent over a link with `bw` bandwidth:

```
transfer_time = B / bw
arrival_time = now + latency + jitter + transfer_time
```

If a drop is scheduled, the message is silently dropped (no
notification to the sender).

### How bandwidth modeling affects tests

A naive broadcast test (without bandwidth simulation) might pass
because the network is "infinite." With bandwidth simulation:

- Sender saturates its uplink → messages queue → tests must handle
  queueing.
- Peers with limited bandwidth become bottlenecks → tests must
  verify message delivery despite slow peers.
- The 10 MB block test takes longer than the small-message test
  (proportional to `B / bw`) → tests take time proportional to
  realistic conditions.

For Commonware: the broadcast engine must work under realistic
bandwidth constraints. Tests verify this.

### Bandwidth and the broadcast engine's design

The engine's design choices are bandwidth-aware:

1. **Bounded per-peer cache**: prevents memory from blowing up when
   receiving lots of data.
2. **Reference counting**: deduplicates, so high-bandwidth senders
   don't waste receiver memory.
3. **Priority flag**: lets time-sensitive messages bypass the queue.
4. **Coded broadcast**: cuts sender egress from `O(N)` to `O(1)`
   (chapter 07).

Without these, the engine would either run out of memory, lose
messages, or back up the network.

### The economics of bandwidth

Bandwidth has a real cost. For 100 validators across multiple
regions, the inter-region bandwidth is the most expensive. A 1 Gbps
cross-region link costs ~$1000/month (AWS Direct Connect, GCP
interconnect, etc.).

For BFT: minimizing inter-region bandwidth is a key design goal.
Coded broadcast helps (chapter 07); the broadcast engine's design
assumes the network is a bottleneck.

### Why "simulated bandwidth" matters for testing

A real bandwidth test (deploying to 100 VMs and measuring) is
expensive and flaky. A simulated bandwidth test:
- Runs in CI (fast, reproducible).
- Tests edge cases (saturation, congestion, asymmetric links).
- Verifies that the engine doesn't crash under realistic load.

Commonware's test suite uses simulated bandwidth extensively
(chapter 02's deterministic executor + simulated network).

### Bandwidth in production deployment

For a production BFT deployment, the bandwidth budget should be
allocated as follows (rough rules of thumb):

| Tier | Bandwidth | Notes |
|---|---|---|
| Inter-validator (control) | ~10 Kbps per peer | Certificates, votes |
| Inter-validator (data) | ~100 Mbps per peer | Blocks (coded) |
| Intra-region | ~1 Gbps | Validator cluster |
| Cross-region | ~100 Mbps | WAN links |

The "100 Mbps per peer for data" assumes coded broadcast with
`k = 67, n = 100`: each peer receives `B / 67 ≈ 150 KB` per block.
At 1 block per second, that's 1.2 Mbps per peer — well within
budget.

For larger blocks (10 MB), the per-peer rate is 12 Mbps. Still
manageable.

## Plausible eventual consistency, formally

The "plausible eventual consistency" section above gave the
intuition. Let me state it formally.

### The model

A replicated state machine with `N` replicas. Each replica has a
local state. Updates are propagated via gossip.

**Definition (Plausible Eventual Consistency, Demers et al.)**:

> A system is plausible eventually consistent if, for every pair of
> non-faulty replicas `A, B` and every update `u`, the probability
> that `B` eventually receives `u` approaches 1 as time goes to
> infinity, assuming the gossip continues.

The "plausible" qualifier: this is not a hard guarantee. A separator
(network partition that never heals) breaks the property. But under
"normal" conditions, convergence is overwhelming.

### Why this matters for Commonware

Commonware's broadcast engine is plausible eventually consistent:

- It uses gossip (push-pull anti-entropy).
- The peer set is the validator set (closed).
- Updates are messages (data disseminated via broadcast).

For BFT, plausible eventual consistency is **not strong enough**.
Consensus needs **strong consistency**: every honest replica agrees
on the order of finalized blocks.

So Commonware uses broadcast for *pre-consensus* data dissemination
(the block is in flight) and consensus for *post-consensus* ordering
(the block is finalized). The two layers are separate.

### The CAP theorem context

Plausible eventual consistency sits in the **AP corner** of CAP:
the system is **available** during partitions but provides only
**eventual consistency** (vs. strong consistency in the CP corner).

For BFT: the system needs **CP** (consistency + partition tolerance).
This is what Simplex provides (chapter 11).

The broadcast engine is AP-style. Consensus is CP-style. They're
layered: broadcast disseminates data; consensus orders it.

### Comparison: PEC vs other consistency models

| Model | Guarantee | Use case |
|---|---|---|
| Strict (linearizable) | Reads see the latest write | Banking, locks |
| Causal | Reads see causally-preceding writes | Social feeds |
| Eventual | Reads eventually see all writes | DNS, caches |
| Plausible eventual | Eventual + probabilistic | Gossip-based systems |

For BFT consensus, the broadcast engine's PEC is fine for the data
layer; Simplex's linearizability is what guarantees consensus.

### The probability of convergence

Demers et al. showed that for push-pull gossip with random peering,
the probability of a single update being lost forever decays
exponentially with the number of gossip rounds:

```
P(lost after r rounds) ≤ e^(-c × r²)
```

For `N = 100, k = 2, c = some constant`, after 10 rounds, the
probability of permanent loss is `e^(-100c) ≈ 0` for `c ≥ 0.05`.

For BFT: 10 gossip rounds × 1 second per round = 10 seconds. Plenty
of time to converge.

But "permanent loss" requires the message to die out completely —
for a BFT block, even a single replica having it is enough (the
consensus doesn't require all replicas to have the block pre-
finalization).

## Exercises

These exercises build intuition for the broadcast engine's design.

### Exercise 1 — Epidemic spreading simulation

Simulate a push-pull epidemic protocol:

1. `N = 100` nodes in a graph (fully connected).
2. Each round: each node picks `k = 2` random peers, exchanges
   state.
3. Track the fraction of nodes that have heard each update.
4. Plot `s(t)` over `t = 0..50`.

Compare to:
- Push-only: each node sends to `k` random peers, doesn't pull.
- Pull-only: each node asks `k` random peers.

Verify that push-pull is fastest.

**Stretch goal**: add a churn rate (5% of nodes leave/join per
round) and verify convergence still happens.

### Exercise 2 — Reference-counted dedup

Implement the broadcast engine's reference-counted dedup:

1. `HashMap<Digest, Message>` for items.
2. `HashMap<Peer, VecDeque<Digest>>` for deques.
3. `HashMap<Digest, usize>` for refcounts.

Test:
- Send the same message from 3 peers; verify refcount = 3.
- Evict from 2 peers' deques; verify refcount = 1.
- Evict from the last peer; verify the message is gone.

**Stretch goal**: add peer-set changes (one peer leaves mid-stream);
verify the refcount is correctly decremented.

### Exercise 3 — Max-min fairness simulation

Simulate a 10-flow network with max-min fairness:

1. Each flow has a demand (e.g., 10 Mbps).
2. Total link capacity: 50 Mbps.
3. Use water-filling: allocate evenly until one flow is satisfied;
   freeze it; redistribute.

Verify the allocation is fair: all unsatisfied flows get equal
bandwidth, satisfied flows get their demand.

### Exercise 4 — QUIC vs TCP broadcast

Implement a simple broadcast over both TCP and QUIC (datagram mode):

1. Sender pushes 1000 messages of 1 KB each.
2. Measure: total time, bytes sent (including retransmits), bytes
   received by all peers.

Compare:
- TCP: every message retransmitted on loss.
- QUIC datagram: lost messages not retransmitted; gossip retries
  at the application layer.

Verify that QUIC datagram mode uses less total bandwidth.

### Exercise 5 — Plausible eventual consistency simulator

Simulate a gossip system with partitions:

1. 100 nodes in two halves (50-50 split).
2. Each half gossips internally; cross-half gossip is blocked.
3. After 100 rounds, heal the partition; resume gossip.

Verify:
- During partition: convergence within each half (100% of nodes in
  each half have heard the update).
- After healing: convergence across both halves (100% of all nodes).

**Stretch goal**: add an adversarial peer that drops all gossip;
verify the system still converges via other paths.

### Exercise 6 — Bandwidth-bounded broadcast

Implement a broadcast engine with bandwidth limits:

1. Sender's uplink: 10 Mbps.
2. Each peer's downlink: 1 Mbps.
3. Broadcast a 1 MB block to 100 peers.

Measure:
- Sender's egress: should be ~10 MB (the block, not 100 MB).
- Each peer's ingress: should be ~150 KB (1/67 of the block, with
  coding).
- Time to deliver: should be ~1 second (10 MB / 10 Mbps + peer
  delivery).

If any of these is wildly off, there's a bug.

## Where to look in the code (intermediate)

These pointers bridge the new sections to the implementation.

- `broadcast/src/lib.rs:14-31` — `Broadcaster` trait.
- `broadcast/src/buffered/mod.rs:1-24` — `Engine` overview.
- `broadcast/src/buffered/engine.rs` — the implementation.
- `broadcast/src/buffered/ingress.rs` — mailbox and message types.
- `broadcast/src/buffered/config.rs` — config knobs.
- `broadcast/src/buffered/metrics.rs` — Prometheus metrics.
- `broadcast/src/buffered/mocks.rs` — mocks for testing.
- `consensus/src/marshal/standard.rs` — how Marshal uses broadcast.

## If you only remember three things (expanded)

1. **`Broadcaster::broadcast(recipients, msg)` is the API.** Tiny
   trait; big cleverness inside (coding + gossip + dedup +
   reference counting + priority).

2. **Peers cache messages before you ask.** Bounded per-peer;
   evicted when full. Different from "fetch on demand." Reference
   counting means a message from multiple peers is stored once.

3. **Reference-counted dedup across senders.** Same message from
   multiple peers = stored once. Eviction waits for the last
   reference. This is the secret sauce that lets the engine handle
   redundancy cheaply.

## Appendix A — The `buffered::Engine` internals

`broadcast/src/buffered/engine.rs`. The engine maintains:

```rust
struct BufferedEngine<M: Codec + Clone> {
    // Per-peer ring buffers of recent messages
    buffers: BTreeMap<PublicKey, VecDeque<M>>,
    // Dedup: which messages came from which peers
    references: BTreeMap<M, BTreeSet<PublicKey>>,
    // The cache size (per peer)
    cache_size: NonZeroUsize,
    // Channel for outgoing broadcasts
    sender: Sender,
    // Channel for incoming requests (serving cached messages)
    receiver: Receiver,
}
```

Let me unpack each field.

### The `buffers` — per-peer ring buffers

`BTreeMap<PublicKey, VecDeque<M>>`. Each peer has a bounded `VecDeque`
holding its recent messages. New messages from peer X push to the
**back** of X's deque. When X's deque is full, the **front** message is
evicted.

The `BTreeMap` (not `HashMap`) gives you **deterministic iteration order**
(sorted by public key) — important for reproducibility in tests.

### The `references` — dedup counter

`BTreeMap<M, BTreeSet<PublicKey>>`. For each cached message, the set of
peers who have contributed it. A message stays cached as long as **any
peer's buffer still references it**.

This is reference counting via sets, not via atomic counters. Cleaner
than `Rc<M>` (which can't be sent across threads) and avoids the ABA
problem.

## Appendix B — The dedup algorithm

When a message `m` arrives from peer `X`:

```rust
fn on_message(m: M, from: PublicKey) {
    // Step 1: add X to the references for m
    let refs = self.references.entry(m.clone()).or_insert_with(BTreeSet::new);
    refs.insert(from.clone());

    // Step 2: add m to X's buffer (evict oldest if full)
    let buf = self.buffers.entry(from.clone()).or_insert_with(|| VecDeque::with_capacity(self.cache_size.into()));
    if buf.len() >= self.cache_size.get() {
        // Evict the oldest message
        let evicted = buf.pop_front().unwrap();
        self.remove_reference(&evicted, &from);
    }
    buf.push_back(m);
}

fn remove_reference(m: &M, from: &PublicKey) {
    if let Some(refs) = self.references.get_mut(m) {
        refs.remove(from);
        if refs.is_empty() {
            // Last reference — actually delete the entry
            self.references.remove(m);
        }
    }
}
```

The dedup property: if peer X and peer Y both send message `m`,
`references[m] = {X, Y}`. The message is in both `buffers[X]` and
`buffers[Y]`. If X's buffer evicts `m`, only Y's reference remains. The
message stays cached. When Y also evicts, the message is removed.

This is the **BFE** (Bloom-filter equivalent) optimization — wait, that's
not right. It's just reference counting. Reference-counted dedup is
O(refs) per eviction, which is fine for small reference sets.

## Appendix C — The eviction policy

When a peer's buffer is full, the oldest message is evicted:

```rust
fn evict_oldest_from(peer: &PublicKey) {
    let buf = self.buffers.get_mut(peer).unwrap();
    let evicted = buf.pop_front().unwrap();
    drop(buf);
    self.remove_reference(&evicted, peer);
}
```

The "oldest" is the message that arrived first (FIFO). This is the
**ring buffer** eviction policy.

Other policies are possible (LRU on access, priority-based) but FIFO is
the simplest and most predictable.

## Appendix D — Serving cached messages

When the engine receives a request (via the resolver or direct query):

```rust
fn on_get(key: Key) -> Option<M> {
    // Look up the message by key
    self.references.get(&key).map(|peers| {
        // The message is cached if anyone still references it
        // (No need to fetch from any specific peer)
        // ... return the cached value ...
    })
}
```

Actually, the broadcast engine doesn't index messages by key. The
resolver pattern would be: peer X has message `m`; you want `m`; you
ask X. X's broadcast engine has `m` cached (because X was one of the
peers who contributed it). X serves `m` directly.

The broadcast layer is **stateless with respect to requests**. It just
caches incoming broadcasts and serves them when asked.

## Appendix E — Memory accounting

The engine's memory usage is bounded by:

```
Total memory = (cache_size × avg_message_size) × num_active_peers
```

For `cache_size = 100`, avg_message = 1 KB, 100 peers = 10 MB. Manageable.

But with dedup, the actual storage is less because redundant messages
are shared. Effective memory = `cache_size × avg_message_size ×
(unique_messages) / (per_peer_storage)`.

## Appendix F — The `Mailbox` interface

`broadcast/src/buffered/ingress.rs`. The mailbox:

```rust
pub struct Mailbox<M> {
    sender: UnboundedSender<Message<M>>,
}

pub enum Message<M> {
    Broadcast(M),
    Get(Key),
}
```

`UnboundedSender` — internal use, no rate limiting. The mailbox sends
to the engine's actor task.

The engine's main loop processes mailbox messages in order:
1. `Broadcast(M)` — receive and cache.
2. `Get(Key)` — fetch from cache and respond.

## Appendix G — Peer set transitions

`buffered/mod.rs:21-24`:

> Only peers in `latest.primary` may buffer messages. When a peer is no
> longer in `latest.primary`, its buffered messages are evicted unless
> buffered by any other primary peer.

The implementation tracks the active peer set. On transition:

```rust
fn on_peer_set_change(new_primary: &Set<PublicKey>) {
    let old_primary = std::mem::replace(&mut self.primary, new_primary.clone());

    for peer in old_primary.difference(&new_primary) {
        // Peer was demoted from primary
        if let Some(buf) = self.buffers.remove(peer) {
            for m in buf {
                self.remove_reference(&m, peer);
            }
        }
    }
}
```

Messages from a peer demoted from primary are **removed from buffers and
references**. If another primary peer still has the same message, it
stays cached. Otherwise, it's gone.

This is reference counting via sets — same logic as eviction, just at a
different granularity.

## Appendix H — The `priority` flag

```rust
fn broadcast(&self, recipients: Recipients<Self::PublicKey>,
             message: Self::Message, priority: bool) -> Unreliable<Feedback>;
```

The `priority` flag lets the caller mark a message as urgent. The
implementation can choose to:
- Send priority messages first.
- Use a separate queue for priority messages.
- Increase the rate limit for priority messages.

In `buffered::Engine`, the priority flag is passed through to the P2P
layer's priority handling. By default, the P2P layer's TCP send loop
may send priority messages before normal ones.

For consensus: notarizations are priority (they're time-critical);
gossip is normal.

## Appendix I — Why "buffered" and not "cache"

The name "buffered" emphasizes the pre-staging nature: messages are
buffered **before** you ask. The engine doesn't wait for explicit fetch
requests; it absorbs everything broadcast to it.

This is a different design philosophy from "on-demand cache" (e.g., a
CDN). Buffered = proactive (everything received is cached for some
window). On-demand = reactive (cache only what's been fetched).

For consensus: you want buffered. Block broadcasts are continuous; you
might want any recent block at any moment. Pre-staging them all is
much faster than fetching on demand.

## Appendix J — Test patterns

From `broadcast/src/buffered/tests.rs` (look there for the full set):

```rust
async fn test_message_propagation() {
    // Setup: 4 peers, each with a buffered engine
    // Peer A broadcasts message M
    // Wait for all other peers to receive M via the simulated network
    // Verify all 3 others have M cached
    // Verify references[M] = all 4 peers
}

async fn test_dedup_across_peers() {
    // Setup: 4 peers
    // Peer A broadcasts M
    // Peer B broadcasts M (same message)
    // Peer C's buffer gets M (via gossip from A or B)
    // Peer C evicts M from its buffer (cache_size = 1, with older M evicted)
    // Verify: M still cached in A or B's buffer; references[M] still has C
}
```

The dedup test verifies the **reference counting** behavior — eviction
from one buffer doesn't delete the message if another buffer still
holds it.

## Appendix K — Common gotchas

### Cache size too small

If `cache_size = 10` and you have 100 active peers broadcasting
constantly, you evict old messages quickly. Late-joining peers (or
peers catching up after a partition) can't fetch what they need from
your cache.

**Rule of thumb**: `cache_size` should be ≥ `messages_per_minute ×
expected_minutes_to_catch_up`. For consensus: `cache_size ≥ 100` is a
reasonable minimum.

### Forgetting to register channels

The engine doesn't auto-register channels. You (the caller) must:

```rust
let (sender, receiver) = network.register(
    0,                              // channel ID
    Quota::per_second(NZU32!(10)),  // rate limit
    256,                            // mailbox size
);
// Pass sender/receiver to the engine
let engine = Engine::new(context, config, sender, receiver);
```

If you forget, broadcasts go nowhere silently.

### Mixing broadcast with gossip

If you also have a separate gossip layer broadcasting the same messages,
you'll see them twice. The dedup catches this (same message → same
entry in references), but you're wasting bandwidth upstream. Pick one
broadcast layer per channel.

## Where to look in the code (expanded)

- `broadcast/src/lib.rs` — the trait.
- `broadcast/src/buffered/mod.rs` — the entry point.
- `broadcast/src/buffered/engine.rs` — the Engine implementation.
- `broadcast/src/buffered/ingress.rs` — mailbox and message types.
- `broadcast/src/buffered/config.rs` — config knobs.
- `broadcast/src/buffered/metrics.rs` — Prometheus metrics.
- `broadcast/src/buffered/mocks.rs` — mocks for testing.
- `consensus/src/marshal/standard.rs` — how Marshal uses broadcast.

## If you only remember three things

1. **`Broadcaster::broadcast(recipients, msg)` is the API.** Tiny trait; big cleverness inside (coding + gossip + dedup).
2. **Peers cache messages before you ask.** Bounded per-peer; evicted when full. Different from "fetch on demand."
3. **Reference-counted dedup across senders.** Same message from multiple peers = stored once. Eviction waits for the last reference.

→ Next: **Chapter 10 — Collector**. The third member of the data-movement
trio: "send a request to N peers, gather responses, commit when quorum is met."