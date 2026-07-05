# Chapter 10 — Collector: gather committable responses

> How a leader sends a request to many peers and knows when to commit.

## The problem

Different from the resolver or broadcast. In those:

- **Resolver**: I want some data. Fetch it.
- **Broadcast**: I have data. Send it to others.

The **collector** is a third pattern:

- **Collector**: I send a request to many peers. They respond. I want to
  know when I have enough responses to act (e.g., commit, finalize, advance).

This is the request/response pattern, but with **quorum semantics**. You
don't need ALL responses — you need a quorum (some agreed-upon count, often
`2f+1` from chapter 01).

## The trait surface

Open `collector/src/lib.rs:21-79`. Three traits:

```rust
pub trait Originator: Clone + Send + 'static {
    type PublicKey: PublicKey;
    type Request: Committable + Digestible + Codec;
    fn send(&mut self, recipients: Recipients<Self::PublicKey>, request: Self::Request) -> Feedback;
    fn cancel(&mut self, commitment: <Self::Request as Committable>::Commitment) -> Feedback;
}

pub trait Handler: Clone + Send + 'static {
    type PublicKey: PublicKey;
    type Request: Committable + Digestible + Codec;
    type Response: Committable<Commitment = <Self::Request as Committable>::Commitment>
                   + Digestible<Digest = <Self::Request as Digestible>::Digest>
                   + Codec;
    fn process(&mut self, origin: Self::PublicKey, request: Self::Request,
               response: oneshot::Sender<Self::Response>);
}

pub trait Monitor: Clone + Send + 'static {
    type PublicKey: PublicKey;
    type Response: Committable + Digestible + Codec;
    fn collected(&mut self, handler: Self::PublicKey, response: Self::Response, count: usize);
}
```

The three roles:

- **Originator** — the requester. Sends requests, collects responses,
  decides when quorum is met.
- **Handler** — the responder. Receives a request, optionally produces a
  response.
- **Monitor** — observability. Called for each response, gets the count of
  responses received so far for the same commitment.

## The `Committable` constraint

`Request: Committable + Digestible + Codec` — three things the request type
must be:

- **`Committable`** — has an associated `Commitment` type (think: hash or
  ID). Two requests with the same commitment can have their responses
  merged.
- **`Digestible`** — has a `Digest`. For deduplication and lookup.
- **`Codec`** — can be encoded/decoded (chapter 03).

The response has a **matching** commitment (`Commitment = <Request as
Committable>::Commitment`) and matching digest. This is what lets the
collector verify a response goes with a specific request.

## The Monitor pattern

`Monitor::collected(handler, response, count)` is called for **every**
response received. The `count` argument tells you how many responses have
been collected for this commitment so far (including this one).

```rust
impl Monitor for MyMonitor {
    fn collected(&mut self, handler: PublicKey, response: Response, count: usize) {
        if count == QUORUM {
            // we've got enough! commit.
            self.commit(response);
        }
        // Otherwise: keep waiting.
    }
}
```

The Monitor is your application's view into the response collection. The
collector internally tracks responses; the Monitor is the observer.

## The `cancel` mechanism

`Originator::cancel(commitment)` stops caring about a request. Any future
responses for that commitment are ignored.

Why cancel?
- You got your quorum early — no need to keep waiting for stragglers.
- The request is no longer relevant (you've moved on).
- The originator is shutting down.

Without cancellation, you'd accumulate pending response tracking
forever.

## Where the Collector shows up

In Simplex consensus, the Collector pattern is **how the Batcher collects
votes**. Look at the architecture diagram from chapter 01:

```
+------------+          +++++++++++++++
|            +--------->+             +
|  Batcher   |          +    Peers    +
|            |<---------+             +
+-------+----+          +++++++++++++++
```

The Batcher is essentially a Collector: validators send `notarize(c, v)`
messages, the Batcher collects them, and when it has `2f+1` it produces a
`Notarization(c, v)` certificate.

So Collector is so fundamental to consensus that it's woven into Simplex
directly, not bolted on. But the standalone `commonware-collector` crate
provides the pattern for application-level request/response with quorum.

## Collector vs Resolver vs Broadcast — the trio

| Primitive | Direction | Cardinality | Quorum? |
|---|---|---|---|
| Resolver | Request data | 1 request, 1 response | No |
| Broadcast | Send data | 1 broadcast, N receives | No |
| Collector | Request with quorum | 1 request, M responses | **Yes** |

- **Resolver**: "give me block H" → "here's block H" (when any peer responds)
- **Broadcast**: "I'm broadcasting block B" → all peers get it
- **Collector**: "I'm asking for votes on block B" → when 67% have voted, I'm done

## A worked example

Imagine you're running a price oracle. Each round, you ask 100 data feeds
for the current BTC/USD price. You commit when 67 (2f+1 with f=33) agree.

```rust
// Define request and response types
struct PriceRequest { round: u64, asset: Asset }
struct PriceResponse { round: u64, asset: Asset, price: u64, signature: Signature }

impl Committable for PriceRequest {
    type Commitment = u64;  // round number
    fn commitment(&self) -> u64 { self.round }
}

// Set up the originator (sends requests)
let mut originator = MyOriginator::new(p2p_network);
originator.send(Recipients::All, PriceRequest { round: 42, asset: BTC });

// Set up the handler (responds)
impl Handler for PriceFeed {
    fn process(&mut self, origin: PublicKey, request: PriceRequest,
               responder: oneshot::Sender<PriceResponse>) {
        // Fetch price, sign it, send back
        let price = self.fetch_price(request.asset);
        let response = PriceResponse { round: 42, asset: BTC, price, signature: ... };
        responder.send(response);
    }
}

// Monitor (gets called per response)
impl Monitor for PriceAggregator {
    fn collected(&mut self, handler: PublicKey, response: PriceResponse, count: usize) {
        if count == QUORUM {
            // We have enough — commit the median price
            self.commit(response);
        }
    }
}
```

The collector internally:
- Tracks which responses have arrived for each commitment (round)
- Deduplicates responses from the same handler
- Calls `monitor.collected` for each new response
- Cleans up when `cancel` is called or the originator drops

## Where to look in the code

- `collector/src/lib.rs:21-79` — the three traits.
- `collector/src/p2p/` — the P2P-backed implementation.
- `consensus/src/simplex/actors/batcher.rs` — how Simplex uses this pattern
  for vote collection.

## The collector pattern in distributed systems

The collector is one of the oldest patterns in distributed systems,
predating it by decades in human terms: assemble a quorum of
agreeing parties before acting. The digital version is the same:
ask a question of many peers, count the responses, declare success
when enough agree.

The canonical references:

- **Nakagawa's quorum systems** (the formal model, 1984).
- **Gifford's weighted voting** (1979) — read quorums with arbitrary
  per-peer weights (we'll encounter this in `storage::archive`).
- **Byzantine quorum systems** (Malkhi & Reiter, 1998) — quorums
  that survive up to `f` Byzantine failures.

In Commonware, the collector pattern is what makes `2f+1` votes
become a certificate. In storage, it's what makes "read from `R` of
`N` replicas, write to `W`, with `R + W > N`" work
(consistent reads — "Quorum Reads" in Kleppmann's DDIA, chapter 5).

The collector is **simpler than a full consensus protocol**. It
doesn't decide *what* to do (that's Simplex's job); it just
aggregates *who said what*. Your code is the policy layer that
turns the response set into an action.

Three things to internalize:

1. **Quorum size is a tunable policy**, not a fixed property.
   `2f+1` for Byzantine safety with `N = 3f+1` validators is the
   Classic answer. Other choices (`f+1` for non-Byzantine, weighted
   thresholds for stake-weighted voting) apply.
2. **A quorum is a *subset*, not a fixed count.** Any subset of size
   `q` is a quorum. The collector doesn't promise a *specific* peer
   is in the response set, just that the set has `q` distinct ones.
3. **The collector is cancellation-aware.** Peers may never respond
   (network partition, crash, Byzantine). The collector must accept
   partial completion.

## Quorum systems — the formal model

A **quorum system** over a peer set `S` is a family `Q ⊆ 2^S` of
subsets (the *quorums*) such that:

- (Intersection) `∀ q₁, q₂ ∈ Q: q₁ ∩ q₂ ≠ ∅` (any two quorums share a
  peer).
- (Or, in the Byzantine case) `∀ q₁, q₂ ∈ Q: |q₁ ∩ q₂| ≥ f + 1` if
  there are `f` Byzantine failures.

The intersection property is what makes "split-brain" impossible:
two concurrent operations, each assembling a quorum, must share a
peer, and that shared peer must enforce a single order.

For Byzantine quorums specifically, you need `|Q| ≥ 2f + 1` for `N = 3f + 1`.
That's the classic `2f+1` threshold. The `f+1` overlap is the safety
margin: with `f` Byzantine peers, even the worst case of `|q₁ ∩ q₂|`
contains an honest peer.

```
N = 4        →  f = 1
quorum size = 3 (2f+1)
intersection guarantee = 2 (always ≥ f+1)

N = 7        →  f = 2
quorum size = 5 (2f+1)
intersection guarantee = 3 (≥ f+1)

N = 10       →  f = 3
quorum size = 7
intersection guarantee = 4
```

Commonware's runtime anywhere `N = 3f+1` works (because it's the
tightest setting), but it also works for `N > 3f+1` (more redundancy,
less aggregation efficiency).

**Weighted quorums.** Each peer has a *weight*. A quorum is a subset
whose weights sum to at least `W`. In some chains, weight is stake.
In others, it's hash power. Commonware lets you put any weight into
the `Commitment`/`Digest` types, so you can implement weighted
versions easily.

**Read/write overlap.** Read quorum size `R`, write quorum size `W`.
For consistency on reads:

```
R + W > N     (write → at least one read quorum, read → at least one write quorum)
```

In Byzantine systems: `R + W > N + f`. Commonware's collector treats
*quorum counting* uniformly — the application chooses the threshold
in its `Monitor` or directly in its `send` call.

## The Monitor pattern for observability

`Monitor::collected(handler, response, count)` is called once per
response (chapter 10 main text). The `count` argument is the running
total of distinct handlers who have responded for this commitment.

For a real Byzantine consensus system, you want to instrument the
collector. Three categories of metrics:

**Per-response metrics.** Every response:

```
collector_responses_total{view, source} ++
collector_response_latency_seconds{view, source}.observe(elapsed)
```

The `source` label tells you if the response came from a specific
peer (`One(pk)`) or a broadcast. High `response_latency_seconds`
on multiple sources signals an unhealthy network.

**Per-commitment metrics.** When a commitment's `count` crosses
the quorum threshold:

```
collector_quorum_reached_total{commitment_class="notarize|nullify|finalize"} ++
collector_quorum_reach_seconds{commitment_class}.observe(elapsed_from_send)
```

This is the SLA for quorum reach — the time from when you sent
the request to when you had enough. For notarizations, in a healthy
100-validator cluster over WAN, this should be < 500ms (one round-trip).

**Per-failure metrics.** When a commitment's `count` never reaches
quorum:

```
collector_quorum_failed_total{reason="timeout|canceled|peer_down"} ++
```

Combine with the per-response metrics: if 5 of 67 peers are
unresponsive, you don't fail — you just take longer.

A typical implementation:

```rust
impl Monitor for MyMonitor {
    fn collected(&mut self, handler: PK, response: Response, count: usize) {
        let elapsed = self.start_time.elapsed();
        METRICS.responses_total.inc();
        METRICS.response_latency.observe(elapsed.as_secs_f64());
        if count >= self.quorum {
            METRICS.quorum_reached_total.inc();
            METRICS.quorum_reach_seconds.observe(elapsed.as_secs_f64());
            self.commit(response);
        }
    }
}
```

A common pitfall: putting slow work inside `Monitor::collected`.
The Monitor is called from the Originator's mailbox processing
loop. If your monitor does I/O, you block the Originator. Move
heavy work to a separate task:

```rust
fn collected(&mut self, handler: PK, response: Response, count: usize) {
    let metrics_clone = self.metrics.clone();
    self.spawner.spawn("metric-task", async move {
        metrics_clone.heavy_increments(response).await;
    });
}
```

## Cancel semantics and the lost-quorum problem

`Originator::cancel(commitment)` stops waiting for a commitment.
But what happens if a quorum forms *after* you canceled?

The current collector API doesn't deliver late responses — once you
cancel, late responses are silently dropped. Your `oneshot` receivers
(`Delivery::subscribers`) won't be awakened.

That's almost always fine. If you canceled because you had quorum,
you've already committed. If you canceled because you decided to
move on, the late data is junk.

There's a more subtle case: **partial quorum**. Imagine you call
`send(req)`. After 1 second, you have 50 responses, not the 67
you need. You call `cancel(req)`. Then 200 ms later, 17 more peers
finally respond — forming a quorum you abandoned.

This is the "lost quorum" problem. For consensus, it rarely bites
because:

- Notarization attempts to reach `2f+1` within a strict time
  budget (timer `t_a`, chapter 11). If you're under 67 in time,
  you weren't going to make it.
- The view transitions to `v+1` after a nullification; the old
  commitment is dead.
- The application layer (Simplex's voter) tracks which views are
  live and ignores responses to dormant commitments.

Where it *does* bite: **slow followers**. If 17 peers are slow but
still alive, canceling the request means those 17 peers' responses
are dropped without ever being processed — those peers might have
contributed a useful piece of data (e.g., a certificate for an
earlier view the originator needs for ancestry). The safe fix:
don't cancel aggressively unless you're sure other paths cover
the same data.

In collector terms, the formal advice is:

> Only call `cancel` for commitments where you've already
> established `|response set| < quorum` and you're transitioning
> away. Don't cancel preemptively "to free memory" if there's any
> chance the responses would have been useful.

## Deadline math — when to send, when to give up

Say you have `n` peers. You want a quorum of `q`. Each round-trip
takes time `τ`. How long should you wait before giving up?

The probabilistically correct answer depends on the network:

```
Expected response time per peer = τ              (with σ_latency jitter)
Expected time for k-th response ≈ τ + (k-1) × τ/N  (well-mixed parallel)
```

For consensus over a 100-validator wide-area network with `τ ≈ 100 ms`,
the time for the *67th* response (quorum) is `100 ms` (one round-trip
in parallel). So a `t_a` of 300 ms is generous.

The actual deadline you choose is a tunable policy knob.
Commonware's `Config::leader_timeout` and `activity_timeout`
(chapter 11 Appendix K) capture these:

| Knob | Meaning | Typical value |
|---|---|---|
| `leader_timeout` (`t_l`) | How long to wait for leader's proposal before nullifying | `2 × round_trip = 200 ms` |
| `activity_timeout` (`t_a`) | How long to wait for 2f+1 votes before nullifying | `3 × round_trip = 300 ms` |
| `timeout_retry` (`t_r`) | How often to re-broadcast our vote to overcome packet loss | `1 × round_trip = 100 ms` |

The relationship between timeouts and round trips is intentional: a
view should produce a notarization in `2-3 round trips` under good
conditions.

A deeper model — the **errant-leader recovery time**. If the leader is
Byzantine and never proposes, the worst case is:
- `t_l` elapses, you nullify the view.
- Next view's leader (different) gets selected.
- `t_l` × `f+1` views worst-case (the leader is stuck in a 1-of-N cycle
  where every leader happens to be Byzantine).

For `N=3f+1` and expected-fault-free operation, the recovery time is
`O(f)` views × `t_l`. The SkipTimeout mechanism (chapter 11) lets
honest leaders skip ahead, reducing this to `O(1)` for benign
non-proposing leaders.

The math says: **timeouts are sized for the round trip, not for the
network**. A peer set in two regions (50 ms intra-region, 100 ms
cross-region) needs `t_l = 200 ms` to give the cross-region latency a
chance. A peer set in one region needs `t_l = 20 ms` for sub-100ms
view latency.

## Common Collector variants — Batchers and generic collectors

Commonware has two main instantiations of the collector pattern:

**`collector::p2p::Originator`** (this chapter). The generic,
network-backed version. You provide a `Request`, a `Response`,
peers to send to, and a `Monitor`. The originator sends the request,
collects responses, calls the monitor per response, fires the
commit when quorum is met.

**`simplex::actors::batcher::Batcher`** (chapter 11). The
consensus-internal version. It collects *votes* (`notarize(c, v)`,
`nullify(v)`, `finalize(c, v)`) and assembles certificates
(`notarization`, `nullification`, `finalization`). Same pattern,
different protocol.

The Batcher isn't a `collector::Originator` because:

1. **Per-view partitioning.** Votes are scoped to a view, and votes
   for different views can arrive interleaved. The Batcher routes
   by view and rotates state.
2. **Lazy verification.** The Originator verifies each response as
   it arrives (or doesn't — depends on your Monitor). The Batcher
   batch-verifies only when quorum is met (chapter 04 BatchVerifier).
3. **Bisect-on-failure.** If batch verification fails, the Batcher
   bisects to find the bad voter, blocks them, and retries.

The shared shape:

```
in-flight map:       Commitment → PendingSet
signature verifier:  handle response, accumulate, finalize
monitor hook:        per-response callback
cancel mechanism:    drop in-flight state
```

A third pattern, "collection without quorum", shows up in
`bridge` (chapter 12 / `examples/bridge`) — collecting *proofs* of
state for cross-chain messages. This uses the collector for
aggregation but with a different threshold.

When to use what:

| Need | Use |
|---|---|
| Generic request/response with quorum | `collector` |
| Consensus votes → certificates | `simplex::Batcher` |
| Cross-chain proofs | `bridge` machinery |
| One-shot fetch (no quorum) | `resolver` (chapter 08) |

## Rust patterns in collector code

The collector has a richer generic signature than the resolver
because of the `Response: Committable<Commitment = <Request as Committable>::Commitment>`
constraint. Two specific patterns:

**Generic over the message type with associated types.** The
trait shape:

```rust
pub trait Originator: Clone + Send + 'static {
    type PublicKey: PublicKey;
    type Request: Committable + Digestible + Codec;
    type Response: Committable<Commitment = <Self::Request as Committable>::Commitment>
                   + Digestible<Digest = <Self::Request as Digestible>::Digest>
                   + Codec;
    // ...
}
```

This is **not `Box<dyn Trait>`**. It's static dispatch via generics.
Why?

- **No allocation per request.** `Box<dyn Trait>` would heap-allocate
  the response; generics inline it.
- **Type-level pairing.** The associated types guarantee at compile
  time that the request and response share commitment and digest
  types. A typo in the implementation is a compile error, not a
  runtime `unreachable!()`.
- **Inlining.** The compiler can monomorphize and inline the
  per-response callback. With `dyn`, every call is a vtable lookup.

The cost: **code size**. Each unique combination of types generates a
new copy of the trait impls. For Commonware's small set of request
/ response types (a few per app), this is fine. For a hot-loop library
that handled user types, it would matter.

When to switch to `dyn Trait`:

- **Many implementations of the same trait** that a single binary
  uses (you'd generate N copies otherwise).
- **Plugin architectures** where the type isn't known at compile
  time.
- **Recursive types** that would unbox to infinite size.

For the collector, none of these apply; static dispatch wins.

**`Box<dyn Trait>` for the Monitor.** The `Monitor` trait in
collector is a single method:

```rust
fn collected(&mut self, handler: PK, response: Self::Response, count: usize);
```

Some implementations pass `dyn Monitor` to the originator because
the same monitor might handle commitments from many sources. This
is a niche optimization; the default is generics.

**Trait objects vs generics for trait method receivers.** A subtle
Rust idiom: `&dyn Originator` vs `&Originator<Bla>`. The first is
dynamic dispatch; the second is static. The collector's
construction in `collector/src/p2p/originator.rs` takes the
generic form because there's exactly one Originator per
Originator *implementation*. The Monitor and Handler are wrapped
in `Arc<dyn ...>` for the same reason.

## The collector pattern in distributed systems — a deep look

The "collector pattern in distributed systems" section above gave the
intuition. Let me dig deeper into where the pattern shows up, its
formal definition, and the relationship to other patterns.

### The collector pattern in MapReduce

MapReduce's **gather** phase is a kind of collector. The pattern:

```
Map: (K1, V1) -> [(K2, V2)]    # per-input
Shuffle: group by K2             # framework-managed
Reduce: (K2, [V2]) -> V3         # per-group
```

The shuffle-and-reduce phase is a collector: the framework collects
all values for each K2 and groups them for the reducer. The reducer
sees a complete group (or fails).

But there's no **quorum** in MapReduce's gather — the framework
collects **all** values, not a quorum. The collector pattern in
Commonware is closer to a **threshold-gather**: stop when enough
values have arrived.

### The collector pattern in consensus

In every consensus algorithm (Paxos, Raft, PBFT, Simplex, HotStuff,
Tendermint), there's a step where votes are collected and a quorum
is checked:

- **Paxos**: the proposer collects promises from a quorum of
  acceptors; if a quorum is reached, the value is chosen.
- **Raft**: the leader collects votes; once a quorum is reached,
  the entry is committed.
- **PBFT**: the primary collects pre-prepare/prepare/commit
  messages; quorum of each phase enables the next.
- **Simplex**: each voter collects notarization votes; a quorum
  forms a Notarization certificate.

The collector pattern is **the core of consensus**. Without quorum
collection, you can't decide.

### The collector pattern in distributed locks

A distributed lock (e.g., Google's Chubby, Apache ZooKeeper) needs
quorum agreement:

- **Lock acquisition**: collect "I grant you the lock" responses
  from a quorum of lock servers. If a quorum responds positively,
  you have the lock.
- **Lock release**: collect "I release the lock" responses from a
  quorum. (Or: the lock has a TTL; the holder can extend it.)

Chubby uses Paxos internally for replication. The lock service is
a quorum of replicas.

### The collector pattern in replicated state machines

A replicated state machine (RSM) replicates a deterministic service
across multiple replicas. To execute a command:

1. Client sends command to one replica (or all).
2. Replicas agree on the command (consensus).
3. Replicas execute the command (deterministic, so all replicas
   produce the same result).
4. Replicas send results back to the client.
5. Client waits for a quorum of matching results.

Step 5 is the collector pattern: collect results from multiple
replicas, decide based on the quorum.

### The formal definition

A **collector** is a protocol for gathering responses from a peer
set with quorum semantics:

- **Initiator**: the entity collecting responses.
- **Request**: a message sent to peers asking for a response.
- **Response**: a peer's reply, identified by commitment.
- **Quorum threshold**: the minimum number of responses needed to
  declare success.
- **Predicate**: the application-defined check on responses
  (typically "do they agree on a value?").

The collector's interface:

```
collect(request) -> Receiver<Option<Response>>
```

The returned receiver resolves with `Some(response)` if a quorum is
reached, or `None` if the collector is canceled.

### The relationship to gather-scatter (MPI)

MPI's `MPI_Gather` collects data from all processes to one root.
MPI's `MPI_Allgather` collects from all to all. The collector pattern
is similar, but with quorum semantics (don't need all responses).

For BFT, the all-to-all model is too expensive. The quorum model is
right.

### The relationship to aggregation (chapter 13)

Chapter 13 covers the **aggregation** crate, which is a specialized
collector for BLS signature aggregation. The aggregation crate:

1. Sends requests to validators asking for BLS signatures.
2. Collects responses, verifying each signature.
3. When a quorum of valid signatures is collected, aggregates them
   into one BLS aggregate signature.

The aggregation crate uses the collector pattern under the hood.
The difference: aggregation has built-in signature verification,
while the generic collector leaves that to the application.

### The collector pattern in real systems

| System | Use of collector pattern |
|---|---|
| Paxos | Quorum of acceptor promises |
| Raft | Quorum of follower votes |
| PBFT | Three phases, each with quorum |
| Simplex | Notarization/finalization certificates |
| Chubby | Lock service via Paxos |
| ZooKeeper | Zab consensus with quorum |
| etcd | Raft consensus |
| Bitcoin | Block validation (no quorum; longest chain) |
| Ethereum | Casper FFG (quorum of validators) |

The collector pattern is **the** fundamental primitive for consensus.

### Why it's not called "quorum gather"

"Quorum gather" is descriptive but unwieldy. Commonware's term,
"collector," is shorter and emphasizes the data-flow direction.

Alternative names in the literature:
- **Quorum call** (some papers)
- **Quorum read** (if reading state)
- **Quorum write** (if committing state)
- **Threshold aggregation** (if the result is aggregated, like BLS)

Commonware's "collector" is a generalization: any operation that
gathers responses with quorum semantics.

## Quorum systems in full

The "quorum systems" section above gave the basic definitions. Let me
go deeper.

### The formal model

A **quorum system** over a peer set `S = {s_1, ..., s_N}` is a
family `Q ⊆ 2^S` of subsets (the *quorums*) satisfying:

- **(Non-triviality)**: every quorum is non-empty.
- **(Intersection)**: any two quorums intersect: `∀ q₁, q₂ ∈ Q:
  q₁ ∩ q₂ ≠ ∅`.

For Byzantine quorums:

- **(Byzantine intersection)**: `∀ q₁, q₂ ∈ Q: |q₁ ∩ q₂| ≥ f + 1`,
  where `f` is the number of Byzantine faults.

The intersection property is the **safety** guarantee: any two
operations that each assemble a quorum share a peer, and that shared
peer must enforce a single order.

### Majorities: the simple case

A **majority quorum system** has quorums of size `> N/2`:

```
Q = {q ⊆ S : |q| > N/2}
```

For `N = 3f + 1`:

- Quorum size: `2f + 1`.
- Intersection size: at least `f + 1` (two majorities overlap in at
  least one peer minus the difference).
- With `f` Byzantine faults: any two quorums share at least one
  honest peer.

Majority quorums are the simplest and most common. They give
**optimal load** (every peer is in roughly the same number of
quorums) and **optimal availability** (the system tolerates the
maximum number of faults).

### Weighted quorums

In a weighted system, each peer has a weight, and a quorum's total
weight exceeds a threshold:

```
Q = {q ⊆ S : Σ_{s ∈ q} w(s) ≥ W}
```

For example, in a proof-of-stake system:

- Each validator has weight = stake.
- Quorum threshold: `W = 2/3 × total_stake`.

Weighted quorums allow:

- **Variable validator power**: more stake = more influence.
- **Heterogeneous validators**: some have more capacity than others.
- **Decentralization policies**: cap individual weights.

Commonware's collector supports weighted quorums via the
`Monitor::collected` callback: the application decides when a quorum
is met based on the responses received.

### Byzantine quorums (Malkhi-Reiter, 1998)

Byzantine quorum systems extend the model to handle `f` Byzantine
peers. The constraints:

- **(Byzantine intersection)**: any two quorums share at least
  `f + 1` peers.
- **(Byzantine availability)**: the system can survive `f` Byzantine
  peers and still form a quorum.

The classic construction: `N = 3f + 1` peers, quorum size `2f + 1`.
This is **optimal** — you can't do better (smaller `N` or smaller
quorums).

Other Byzantine quorum systems:

- **Masks**: each peer is in a subset of quorums; not all peers
  participate in every quorum.
- **Threshold quorums**: based on threshold cryptography (e.g.,
  `t-of-n` signatures).
- **BFT-CRaft** (Duan et al., 2018): uses a different trade-off
  between quorum size and round complexity.

Commonware uses the classic `N = 3f + 1, quorum = 2f + 1` construction.

### The load-availability trade-off

Two metrics for quorum systems:

- **Load**: the probability that the busiest peer is accessed,
  assuming uniform access.
- **Availability**: the probability that the system can form a
  quorum, given peer failures.

Lower load = better work distribution. Higher availability = better
resilience.

For majority quorums:
- **Load**: `O(1/√N)` (by the birthday paradox, the busiest peer is
  in `O(√N)` quorums out of `N` total).
- **Availability**: tolerates `f = ⌊(N-1)/3⌋` Byzantine peers.

For specialized quorum systems (e.g., "grid" or "tree" structures):
- **Load**: can be `O(1/N)` (every peer in equal number of quorums).
- **Availability**: worse; tolerates fewer failures.

The trade-off: more specialized quorum systems have lower load but
worse availability. Majority quorums are the sweet spot for most
practical systems.

### Optimality results

Naor and Wool (1998) proved:

- **Lower bound on load**: every quorum system has load ≥
  `O(1/√N)` for crash failures, `O(1/N)` for Byzantine failures.
- **Lower bound on availability**: at least one quorum is needed,
  and the size of the smallest quorum matters.

For BFT consensus, the classic `N = 3f + 1, quorum = 2f + 1` is
optimal.

### Read / Write quorum overlap

For replicated storage:

- **Read quorum** `R`: the minimum number of peers to read.
- **Write quorum** `W`: the minimum number of peers to write.

For consistency: `R + W > N`.

For BFT: `R + W > N + f`.

Commonware's storage layer (`storage::archive`, chapter 06) supports
configurable `R` and `W`. The default is `R = W = 2f + 1` (with
`N = 3f + 1`), satisfying `R + W = 4f + 2 > 3f + 1 = N`.

### The CAP theorem context

In CAP (Consistency / Availability / Partition tolerance), you can
have at most two. Quorum systems give different trade-offs:

- **CP**: quorums require consensus; partitions block until a
  quorum is available. (Paxos, Raft, BFT.)
- **AP**: weaker consistency; availability even during partitions.
  (DynamoDB, Cassandra, optimistic replication.)

For BFT consensus, CP is the right choice. Quorums must intersect;
during a partition, you can't make progress.

### Quorum systems in Commonware

Commonware uses the **majority Byzantine quorum system**:

- `N = 3f + 1` validators.
- Quorum size: `2f + 1`.
- Byzantine intersection: at least `f + 1` honest peer shared.

This is the simplest and most-studied construction. For specialized
needs (e.g., weighted by stake), the application can override the
quorum threshold in the `Monitor`.

## Flex Paxos and quorum intersection

Flex Paxos (Howard et al., 2016) generalizes Paxos by allowing
different quorum sizes for different phases. The result: smaller
quorums for some phases, larger for others.

### The classic Paxos quorums

Paxos has two phases:

- **Phase 1 (prepare)**: the proposer picks a proposal number `n`
  and sends `prepare(n)` to a quorum of acceptors. Each acceptor
  responds with the highest-numbered proposal it's seen.
- **Phase 2 (accept)**: the proposer sends `accept(n, value)` to a
  quorum. Each acceptor accepts if `n` is the highest it's seen.

Both phases use a quorum of size `> N/2`. The two quorums intersect
in at least one acceptor, ensuring no conflicting values are
chosen.

### Flex Paxos

In Flex Paxos:

- **Phase 1 quorum**: `Q1` (typically `> N/2`).
- **Phase 2 quorum**: `Q2` (can be smaller, as long as `Q1 ∩ Q2 ≠ ∅`).

If `Q2` is smaller, the proposer can commit with fewer acceptors,
reducing latency and bandwidth. The trade-off: a smaller `Q2` means
less redundancy in Phase 2.

For example, with `N = 5`:

- Classic Paxos: `Q1 = Q2 = 3`.
- Flex Paxos: `Q1 = 4, Q2 = 2`. (Phase 1 needs more; Phase 2
  needs less.)

The intersection `Q1 ∩ Q2` is non-empty (at least 1 acceptor), so
safety holds.

### How Raft does it differently

Raft is a "leader-based" consensus algorithm. The leader handles
proposals; followers vote.

Quorum in Raft: every log entry is replicated to a majority of
followers. The leader commits an entry when a quorum of followers
has acknowledged it.

```
log replication:
    leader sends AppendEntries to all followers
    each follower appends to its log
    leader waits for majority acks
    when majority ack: commit
```

Raft's quorum is the same as Paxos's (`> N/2`). The difference:
Raft has a designated leader, simplifying recovery.

For Simplex (chapter 11), the structure is similar: designated
leader per view, quorum of votes for notarization.

### The common pattern

All these algorithms share the **quorum intersection** property:

> Any two operations that each assemble a quorum share at least one
> peer. That shared peer enforces a single order.

This is the **safety** foundation of BFT consensus. Without it,
two operations could proceed in parallel and produce inconsistent
results.

For Commonware's collector: the application chooses the quorum size
and threshold, but the intersection property is the user's
responsibility. The collector doesn't enforce it; it just counts
responses.

### Why this matters for Simplex

Simplex's safety relies on quorum intersection. The notarization
quorum (`2f + 1`) and the finalization quorum (`2f + 1`) are the
same set of peers, so they intersect trivially.

The nullification quorum (a different set in some protocols)
intersects with both. Simplex's design ensures all quorums overlap.

## The Monitor pattern in observability

The "Monitor pattern for observability" section above gave a basic
treatment. Let me dig deeper.

### What to monitor in a collector

For a production BFT collector, you want metrics for:

1. **Response latency per peer.** Time from sending the request to
   receiving the response.

2. **Quorum formation time.** Time from sending the request to
   reaching quorum.

3. **Response count.** Number of distinct peers that responded.

4. **Failure rate.** Number of timeouts, cancellations, and
   misbehaving peers.

5. **Quorum reach rate.** Fraction of requests that successfully
   reach quorum.

These metrics feed into dashboards and alerts.

### Prometheus metric definitions

For a typical BFT collector:

```rust
use prometheus::{Counter, Histogram, IntCounterVec};

struct CollectorMetrics {
    /// Total responses received, by view.
    responses_total: IntCounterVec,

    /// Response latency, in seconds, by view and source peer.
    response_latency: HistogramVec,

    /// Number of times quorum was reached, by view.
    quorum_reached_total: IntCounterVec,

    /// Time to reach quorum, in seconds, by view.
    quorum_reach_seconds: HistogramVec,

    /// Number of times quorum failed (timeout, cancel).
    quorum_failed_total: IntCounterVec,
}

impl CollectorMetrics {
    fn new() -> Self {
        Self {
            responses_total: IntCounterVec::new(
                "collector_responses_total",
                "Total responses received"
            ).unwrap(),
            response_latency: HistogramVec::new(
                HistogramOpts::new(
                    "collector_response_latency_seconds",
                    "Response latency"
                ).buckets(vec![0.01, 0.05, 0.1, 0.5, 1.0, 5.0]),
                &["view", "source"]
            ).unwrap(),
            quorum_reached_total: IntCounterVec::new(
                "collector_quorum_reached_total",
                "Number of times quorum was reached"
            ).unwrap(),
            quorum_reach_seconds: HistogramVec::new(
                HistogramOpts::new(
                    "collector_quorum_reach_seconds",
                    "Time to reach quorum"
                ).buckets(vec![0.05, 0.1, 0.5, 1.0, 5.0, 10.0]),
                &["commitment_class"]
            ).unwrap(),
            quorum_failed_total: IntCounterVec::new(
                "collector_quorum_failed_total",
                "Number of times quorum failed"
            ).unwrap(),
        }
    }
}
```

### Dashboards for collector metrics

A typical dashboard has these panels:

1. **Response rate** (`rate(collector_responses_total[5m])`): how
   many responses per second. Healthy: steady rate matching the
   consensus rate.

2. **Quorum reach time** (p99 of `collector_quorum_reach_seconds`):
   time from request to quorum. Healthy: < 500ms for BFT.

3. **Quorum failure rate** (`rate(collector_quorum_failed_total[5m])`):
   how often quorums fail. Healthy: < 1% (rare).

4. **Response latency by peer** (heatmap): which peers are slow?
   Heat: red = slow, green = fast.

5. **Responses per commitment** (distribution): how many responses
   per request? Healthy: matches the validator count.

### Common anomalies

- **Slow responses from one peer**: that peer has a network or
  compute issue. Check the peer's CPU, memory, network.
- **Quorum reach time spikes**: network congestion. Check
  inter-region bandwidth.
- **Quorum failure rate spikes**: leader is faulty, or peers are
  crash-looping. Look at peer health.
- **Response count below expected**: some peers are not responding.
  Are they partitioned? Out of sync?

### Alerting rules

For production:

```yaml
- alert: CollectorHighLatency
  expr: histogram_quantile(0.99, rate(collector_quorum_reach_seconds_bucket[5m])) > 1.0
  for: 5m
  annotations:
    summary: "Collector p99 latency above 1s"

- alert: CollectorQuorumFailures
  expr: rate(collector_quorum_failed_total[5m]) > 0.1
  for: 5m
  annotations:
    summary: "Collector quorum failure rate above 10%"
```

### The "heavy Monitor" anti-pattern

A common pitfall: putting slow work inside `Monitor::collected`:

```rust
// BAD: blocks the collector's mailbox
impl Monitor for MyMonitor {
    fn collected(&mut self, handler: PK, response: Response, count: usize) {
        self.metrics_database.save(response).await; // <- blocks!
        if count >= QUORUM { self.commit(response); }
    }
}
```

The fix: spawn a separate task for the heavy work:

```rust
// GOOD: spawns a task, returns immediately
impl Monitor for MyMonitor {
    fn collected(&mut self, handler: PK, response: Response, count: usize) {
        let metrics = self.metrics.clone();
        self.spawner.spawn("metric-task", async move {
            metrics.save(response).await;
        });
        if count >= QUORUM { self.commit(response); }
    }
}
```

### Per-commitment metrics

For a Simplex voter, the "commitment" is the view number. Metrics
should label by view:

```rust
fn collected(&mut self, handler: PK, response: Response, count: usize) {
    let view = response.view();
    self.metrics.responses_total
        .with_label_values(&[&view.to_string()])
        .inc();
    // ...
}
```

This lets you see which views are slow or failing.

## Cancel semantics in depth

The "cancel semantics" section above gave the basics. Let me dig
deeper.

### The "lost quorum" problem

The lost quorum problem: you cancel a request that was about to
reach quorum.

Scenario:
```
T+0.0s: send request R to 100 peers
T+0.1s: 50 responses received (not quorum)
T+0.5s: cancel R (move to next view)
T+0.7s: 17 more responses arrive (would have made 67 = quorum)
T+0.7s: late responses are dropped
```

The quorum was **almost** formed, but the cancel killed it. The
work to send and receive the 17 late responses was wasted.

For BFT consensus: this happens during view transitions. A view
that doesn't reach quorum is nullified, and the next view starts.
The current view's responses are dropped (they're for the old view).

### When canceling is correct

Canceling is correct when:

1. **Quorum was reached early.** You've already committed; cancel
   to free resources.

2. **Quorum is impossible.** All peers have responded with
   incompatible data; no quorum will form.

3. **View is obsolete.** The view has advanced; the request is for
   the old view.

Canceling is **incorrect** when:

1. **You're in a tight memory situation.** Don't cancel just to
   free memory; let the deadline expire.

2. **You don't know if a quorum will form.** Let the deadline
   decide.

3. **The responses might still be useful.** If responses contain
   information needed by other parts of the system (e.g.,
   certificates for an earlier view), canceling loses that info.

### The timeout hierarchy

Commonware's collector uses a timeout hierarchy:

```
deadline (request-level)
    ↓
    sub-timeout (per-peer)
        ↓
        retry (per-peer)
```

The request-level deadline is the absolute time after which the
collector gives up. The per-peer sub-timeout is how long to wait
for a single peer. Retries happen within the sub-timeout.

For Simplex (chapter 11):
- `t_a` (activity timeout): how long to wait for notarization
  quorum.
- `t_l` (leader timeout): how long to wait for the leader's
  proposal before nullifying.
- `t_r` (retry timeout): how often to re-broadcast the vote.

### The retry strategy

A retry sends the request again to peers that haven't responded.
Two strategies:

1. **Bounded retries**: retry N times, then give up.
2. **Time-bounded retries**: retry until the deadline.

For BFT: time-bounded is more common. The collector retries every
`t_r` until `t_a` expires.

### The "lost quorum" math

For a request with deadline `t_a` and per-peer latency `τ`:

```
Expected responses by time t:  t/τ × N  (parallel)
```

For quorum `q = 2f + 1`:

```
Time for quorum ≈ q/N × t_a
```

Canceling before `q/N × t_a` is **premature**. The collector should
wait at least this long before giving up.

For `N = 100, q = 67, τ = 0.1s`:

```
Time for quorum ≈ 67/100 × 0.5s = 0.335s
```

So canceling at `T+0.2s` is too early. Canceling at `T+0.5s` (after
the deadline) is correct.

### The "dropping responses" question

When the collector drops responses (because of cancellation), the
work to receive them is wasted. Is there a way to avoid this?

**Option 1: Don't send to peers that won't respond.** Use the
peer-provider to skip unresponsive peers. But "unresponsive" is
dynamic; you might miss a peer that recovers.

**Option 2: Pre-cancel before sending.** Decide the quorum threshold
in advance and only send to enough peers to reach quorum. But
this doesn't help with adversarial peers (who respond late to
stall).

**Option 3: Use the deadline as the cancel signal.** The collector
auto-cancels at the deadline, regardless of count. This is what
Commonware does.

### The "false quorum" attack

An adversary controls `f` peers. They respond to your request with
matching signatures, hoping you'll declare quorum early.

For BFT: the adversary can only contribute `f` signatures to a
quorum. If the quorum threshold is `2f + 1`, the adversary alone
can't form a quorum. So the false quorum attack requires
collusion with `f+1` additional peers (which is beyond the BFT
assumption).

For non-BFT (e.g., permissioned systems): the adversary might
control more than `f` peers. Use stronger quorum thresholds or
Byzantine fault detection.

## Deadline math in depth

The "deadline math" section above gave the basics. Let me dig into
more nuanced analysis.

### The optimal strategy, formally

Given:
- `N` peers.
- Quorum threshold `q = 2f + 1`.
- Round-trip time per peer: `τ` (with jitter).
- Failure probability per peer: `p_fail`.

What's the optimal time `t*` to wait before giving up?

For a successful quorum:
- Need at least `q` peers to respond.
- Each peer responds with probability `1 - p_fail`.
- Expected number of responses by time `t`: `(1 - p_fail) × N ×
  (1 - e^(-t/τ))` (exponential assumption).

Set this equal to `q` and solve for `t`:

```
t* = -τ × ln(1 - q / (N × (1 - p_fail)))
```

For `N = 100, q = 67, τ = 0.1s, p_fail = 0`:

```
t* = -0.1 × ln(1 - 67/100) = -0.1 × ln(0.33) ≈ 0.111s
```

So waiting 111 ms gives ~67 expected responses. Canceling earlier
risks missing quorum.

### The "tail latency" problem

The exponential model assumes a single time constant `τ`. Real
networks have **tail latency**: most responses arrive quickly, but
a few take much longer.

Empirically, network latency follows a **long-tail distribution**
(e.g., Weibull with shape < 1). The 99th percentile latency can
be 10x the median.

For BFT quorum: you need `2f+1` responses. If the 67th response
arrives at the 99th percentile of latency, your "quorum time" is
99th percentile.

Implication: timeouts should be sized for the **tail**, not the
median. A `t_a` of 100 ms might catch only 50% of quorums; 500 ms
catches 99%.

### The deadline-vs-quorum-size trade-off

You can choose:

- **Tight deadlines** (small `t_a`): fast views, but some peers
  miss quorums (and the view is nullified).
- **Loose deadlines** (large `t_a`): reliable views, but slow.

The right balance depends on the application:

| Application | Recommended `t_a` |
|---|---|
| High-frequency trading | 10 ms (tight) |
| Payment processing | 100 ms (moderate) |
| Cross-chain settlement | 1 s (loose) |
| Archival consensus | 5 s (very loose) |

For Commonware's Simplex: `t_a` is typically 3× round-trip time.
For a 100-validator cluster with 50 ms RTT, `t_a = 150 ms`.

### The adaptive timeout

A more sophisticated approach: **adaptive timeouts** that adjust
based on observed latency.

```
adaptive_timeout():
    if observed_p50_latency < 0.1s:
        return 0.5s   # tight
    elif observed_p50_latency < 0.5s:
        return 1.0s   # moderate
    else:
        return 5.0s   # loose
```

The collector tracks recent latencies and adjusts `t_a` accordingly.
This is more complex but handles variable network conditions.

Commonware's Simplex doesn't currently use adaptive timeouts, but
they're a reasonable extension.

### The "back-pressure" interaction

If the collector is slow to process responses (e.g., because of
expensive verification), the queue grows. The collector should
implement back-pressure: pause sending new requests until the queue
drains.

Commonware's collector uses a bounded mailbox (chapter 15):
`mailbox_size` is a config knob. When full, the sender blocks.

### The "parallel quorums" case

Sometimes a node needs to collect quorums for multiple requests
concurrently (e.g., different views). The math is more complex:

- For `k` concurrent requests, the expected time to complete all
  is `t × (1 + log k)` (Amdahl-like).
- For high `k`, the network becomes the bottleneck.

For Simplex, only one view is active at a time, so `k = 1`. But
during view transitions, multiple views might be in flight. The
collector handles this via per-commitment tracking.

### The "Byzantine" addition

For Byzantine peers, the model changes:

- A Byzantine peer might respond **slowly** (just slow, not
  malicious).
- A Byzantine peer might respond **incorrectly** (signature
  doesn't verify).
- A Byzantine peer might respond **equivocating** (different
  responses to different peers).

For each case, the collector needs to handle. Commonware's
collector treats incorrect responses as "not received" (they
don't count toward quorum). Equivocation is detected by the
application (signature verification).

### The "network partition" case

If a partition isolates `f + 1` honest peers from the rest, the
collector on the "outside" can't form quorum (only `N - f - 1`
peers reachable). The collector waits for the partition to heal.

For BFT: this is correct behavior. Progress requires a quorum;
without it, the system stalls.

## Common Collector variants in depth

The "common Collector variants" section above compared the
collector to Simplex's Batcher. Let me dig deeper into specific
variants.

### The Batcher (Simplex-internal)

Simplex's Batcher (chapter 11) is a specialized collector for
votes:

```
struct Batcher<...> {
    /// Per-view pending votes.
    votes: BTreeMap<View, BTreeMap<PublicKey, Vote>>,
    /// Quorum threshold (typically 2f + 1).
    quorum: usize,
}

impl Batcher {
    fn on_vote(&mut self, vote: Vote) {
        let view = vote.view();
        let pending = self.votes.entry(view).or_default();
        pending.insert(vote.signer(), vote);
        if pending.len() >= self.quorum {
            let cert = assemble_certificate(&pending);
            self.emit(Notarization::new(view, cert));
            self.votes.remove(&view);
        }
    }
}
```

Key features:

- **Per-view partitioning.** Votes are scoped to a view; votes for
  different views are tracked separately.
- **Lazy verification.** Votes are batch-verified when quorum is
  met (chapter 04 BatchVerifier), not on receipt.
- **Bisect-on-failure.** If batch verification fails, bisect to
  find the bad voter, block them, retry.

The Batcher is **more efficient** than the generic collector for
consensus because of the lazy verification. But it's less general.

### The generic Collector

The generic `collector::p2p::Originator` (this chapter's focus)
provides a flexible framework:

```
struct Originator<...> {
    pending: HashMap<Commitment, PendingRequest<...>>,
    sender: Sender,
    mailbox_size: NonZeroUsize,
}

impl Originator {
    fn send(&mut self, recipients: Recipients, request: Request) -> Feedback {
        // Send and track
    }
    
    fn on_response(&mut self, from: PK, response: Response) {
        // Verify, accumulate, fire Monitor
    }
}
```

Key features:

- **Generic over Request/Response types.** Any application can use
  it.
- **Immediate verification.** Each response is verified on receipt
  (via `Consumer::deliver`).
- **Per-commitment tracking.** Multiple requests can be in flight,
  each with its own commitment.

The generic collector is **less efficient** than the Batcher (no
lazy verification) but **more flexible**.

### When to use which

| Need | Use |
|---|---|
| Generic request/response with quorum | Generic collector |
| Consensus votes → certificates | Batcher |
| BLS signatures → aggregate | `aggregation` crate |
| Cross-chain proofs | `bridge` machinery |
| One-shot fetch (no quorum) | Resolver (chapter 08) |

For consensus work, the Batcher is right. For application-level
work (e.g., an oracle service), the generic collector.

### The Resolver as a degenerate Collector

The Resolver (chapter 08) is a degenerate Collector:

- Single request per commitment.
- Quorum size = 1 (any peer response).
- No cancellation needed (response delivers).

If you generalize the Resolver: `q = 1, cancel = always-on-arrival`.
That gives the Resolver.

### The aggregation crate

The aggregation crate (chapter 13) is a specialized collector for
BLS signatures:

```
struct Aggregator {
    sigs: HashMap<Commitment, BTreeMap<PublicKey, Signature>>,
    threshold: usize,
}

impl Aggregator {
    fn on_signature(&mut self, from: PK, sig: Signature) {
        if !verify_bls(&sig, ...) { return; }
        // ... accumulate ...
        if count >= threshold {
            let agg = aggregate(&sigs);
            self.emit(agg);
        }
    }
}
```

The aggregation crate is even more specialized than the Batcher:
it knows about BLS signatures and aggregation specifically.

### A unified mental model

All these patterns share a core structure:

```
in-flight map:  Commitment → { responses: Vec<Response>, deadline: T }
verifier:       (response) -> bool    # per-response check
quorum check:   |responses| >= q      # commit if so
cancel:         drop the in-flight state
```

The differences are in:

- What's verified and how (per-response vs batch).
- How the request is sent (broadcast vs targeted).
- What the response represents (votes, data, signatures).
- How the cancellation works (manual vs deadline).

Commonware's code base has multiple implementations of this core
pattern. The collector crate is the generic version; the Batcher
and Aggregator are specialized.

### Trade-offs in the patterns

| Property | Generic collector | Batcher | Aggregator |
|---|---|---|---|
| Generic over types | Yes | No (votes only) | No (signatures only) |
| Verification | Per-response | Batch on quorum | Per-response (BLS) |
| Flexibility | High | Low | Low |
| Performance | Moderate | High | High (BLS verify) |
| Code complexity | Low | Medium | Medium |

For consensus, the Batcher's batch verification is a significant
win. For other applications, the generic collector's flexibility
matters more.

## Exercises

These exercises build intuition for the collector's design.

### Exercise 1 — Quorum intersection

Implement a simple quorum intersection test:

1. `N = 7` peers.
2. Generate 100 random quorums of size 5.
3. Verify: any two quorums share at least 3 peers (≥ `f+1 = 3`).

Verify the constraint holds for majority Byzantine quorums
(`N = 3f + 1, q = 2f + 1`).

**Stretch goal**: try `N = 10, q = 7` and verify the intersection
property. Try a non-majority quorum (e.g., `N = 10, q = 4`) and
show the property fails (two disjoint quorums of size 4).

### Exercise 2 — Quorum formation time

Simulate quorum formation under varying network conditions:

1. `N = 100` peers.
2. Each peer has a latency drawn from an exponential distribution
   with mean `τ = 50 ms`.
3. Send a request; collect responses; measure time to reach quorum
   `q = 67`.
4. Repeat 1000 times; plot the distribution.

Compare:
- Tight deadline (`t_a = 200 ms`): how many trials reach quorum?
- Loose deadline (`t_a = 1000 ms`): how many?

**Stretch goal**: add tail latency (some peers have 5x mean
latency). Show that tight deadlines fail more often.

### Exercise 3 — Monitor pattern implementation

Implement a collector with metrics:

1. Use a metrics library (e.g., `prometheus` for Rust).
2. Track: responses per commitment, time to quorum, failures.
3. Run a simulated workload: 1000 requests, varying latencies.
4. Plot the metrics.

Verify:
- Response count per commitment matches the validator count.
- Time to quorum is bounded by `3 × τ` for healthy networks.

### Exercise 4 — Cancel semantics

Implement the lost-quorum scenario:

1. Send a request to 100 peers.
2. At T+0.2s, cancel the request (simulating a view transition).
3. Continue receiving responses (don't drop them yet).
4. Count: how many responses would have completed a quorum?
5. Compute the "wasted work" = bytes received after cancel.

Verify: tight cancels waste more work; loose cancels waste less.

**Stretch goal**: vary the cancel time and plot wasted work vs.
cancel time.

### Exercise 5 — Weighted quorum

Implement a weighted quorum collector:

1. 5 validators with weights [10, 20, 30, 40, 50] (total 150).
2. Quorum threshold: 80 (majority of total weight).
3. Generate random responses; compute the running weight.
4. Declare quorum when weight ≥ 80.

Verify: the weight-based quorum is different from count-based.

**Stretch goal**: add Byzantine peers (some validators are
adversarial). Show that the Byzantine intersection property
holds for weighted quorums if the threshold is `> 2W/3`.

### Exercise 6 — Quorum reach latency analysis

Simulate a 100-validator consensus network:

1. Round-trip times are normal distribution: mean 50 ms, stddev 10 ms.
2. Quorum threshold: 67.
3. Each round: send request, collect responses, measure quorum time.
4. Plot the latency distribution (p50, p90, p99).

Verify: p99 quorum time is ~3x the median (the rule of thumb).

## Where to look in the code (intermediate)

These pointers bridge the new sections to the implementation.

- `collector/src/lib.rs:21-79` — the traits.
- `collector/src/p2p/originator.rs` — originator loop.
- `collector/src/p2p/handler.rs` — handler loop.
- `consensus/src/simplex/actors/batcher/` — Simplex's Batcher.
- `consensus/src/simplex/actors/batcher/batch.rs` — batch
  verification.
- `aggregator/src/lib.rs` — the aggregation crate (BLS
  signatures).

## If you only remember three things (expanded)

1. **Collector = request with quorum semantics.** Send to N peers,
   commit when M respond. The "M" is a policy choice; the
   `Monitor` enforces it.

2. **Monitor is the observability hook.** Called per-response with
   the running count. Keep it fast — heavy work goes in a separate
   task.

3. **Committable + Digestible + Codec** — the type bound that ties
   everything together. The commitment is the key; the digest is
   the content hash; the codec lets you serialize.

## Appendix A — The `Originator` internals

`collector/src/lib.rs:22-40`. The Originator state:

```rust
struct OriginatorState<Req, Res> {
    pending: HashMap<Commitment, PendingRequest<Req, Res>>,
    sender: Sender,
    mailbox_size: NonZeroUsize,
}

struct PendingRequest<Req, Res> {
    request: Req,
    recipients: Recipients,
    responses: HashMap<PublicKey, Res>,    // per-handler
    deadline: Option<SystemTime>,
    monitor: Option<MonitorHandle>,
}
```

When `send(recipients, request)` is called:

```rust
fn send(&mut self, recipients, request) -> Feedback {
    let commitment = request.commitment();
    if self.pending.contains_key(&commitment) {
        // Already pending — merge recipients
        self.merge_recipients(commitment, recipients);
        return Feedback::Ok;
    }

    // New pending
    let entry = PendingRequest { ... };
    self.pending.insert(commitment, entry);
    self.sender.send(recipients, request.encode(), false);
}
```

The `commitment` is the **key**. Two requests with the same commitment
merge into one pending request.

## Appendix B — The `Handler` interface

`collector/src/lib.rs:43-64`. When a handler receives a request:

```rust
trait Handler {
    type PublicKey;
    type Request: Committable + Digestible + Codec;
    type Response: Committable<Commitment = Request::Commitment>
                   + Digestible<Digest = Request::Digest>
                   + Codec;

    fn process(&mut self, origin: PublicKey, request: Self::Request,
               response: oneshot::Sender<Self::Response>);
}
```

The handler gets the request AND a `oneshot::Sender` for the response.
Three options:

1. **Send response**: `response.send(my_response).unwrap()`. The
   Originator receives it.
2. **Drop the sender**: silently don't respond. The Originator's pending
   request waits for other handlers.
3. **Send with `_lossy`**: `response.send_lossy(...)`. Don't care if the
   receiver is still listening.

The `process` is synchronous in the trait, but implementations can be
async (it's a `Future`).

## Appendix C — The matching commitment, in detail

Why does the response have `Commitment = <Request as Committable>::Commitment`?

When a handler responds with `Response`, the Originator needs to verify
the response goes with the original request. The commitment is the key:

```rust
fn on_response(handler: PublicKey, response: Response) {
    let commitment = response.commitment();
    let pending = self.pending.get_mut(&commitment)
        .expect("response for unknown commitment");
    pending.responses.insert(handler, response);

    // Check if we have enough responses
    if pending.responses.len() >= self.quorum {
        // ... commit ...
    }
}
```

The type constraint ensures at compile time: a response for request A
can't accidentally match pending request B.

**`Digestible` too**: `Digest = <Request as Digestible>::Digest`. Same
reason — for indexing, deduplication, keying.

## Appendix D — The `Monitor` interface, in depth

`collector/src/lib.rs:67-79`:

```rust
pub trait Monitor: Clone + Send + 'static {
    type PublicKey: PublicKey;
    type Response: Committable + Digestible + Codec;

    fn collected(&mut self, handler: Self::PublicKey,
                 response: Self::Response, count: usize);
}
```

Called **per response**. The `count` argument is the count of unique
handlers who have responded so far for this commitment.

`Monitor::collected` is invoked synchronously from the Originator's
mailbox processing. If your monitor does slow work, it blocks the
Originator's task — and new responses can't be processed. Keep it fast.

A typical Monitor implementation:

```rust
impl Monitor for MyMonitor {
    fn collected(&mut self, handler: PK, response: Response, count: usize) {
        metrics::RESPONSES_COLLECTED.inc();
        if count >= QUORUM {
            // ... assemble certificate ...
            metrics::QUORUM_REACHED.inc();
        }
    }
}
```

Counting metrics is the common use case. The Monitor can also decide to
**early-commit** if it knows the threshold before all responses arrive:

```rust
fn collected(&mut self, handler: PK, response: Response, count: usize) {
    if count >= QUORUM {
        // Already have quorum — but wait for any pending responses too,
        // since they might have different signatures.
        // (Or, in some protocols, early-commit on first quorum.)
    }
}
```

Most consensus implementations wait for all in-flight responses to be
counted (even after quorum) because additional responses might
contribute more aggregate information.

## Appendix E — The `cancel` mechanism

```rust
fn cancel(&mut self, commitment: <Request as Committable>::Commitment) -> Feedback;
```

Cancels a pending request by commitment. Drops the pending state. Any
future responses for this commitment are discarded.

Why cancel?
- **Quorum reached early**: don't wait for stragglers, save CPU.
- **Request no longer relevant**: timeout, user changed their mind.
- **Originator shutting down**: clean shutdown.

Important: `cancel` does **not** notify handlers. If a handler is
still processing, it'll eventually try to send a response, which gets
discarded silently.

## Appendix F — Common gotchas

### Confusing commitment and digest

`Committable::Commitment` and `Digestible::Digest` are often the same
type but semantically different:
- `Commitment` — the key for matching requests to responses.
- `Digest` — a content hash, for indexing.

You can have a request with `Commitment = u64` (round number) and
`Digest = Sha256Digest` (block hash). They serve different purposes.

### The `Digestible` bound for `Monitor`

```rust
type Response: Committable + Digestible + Codec;
```

Why does the Monitor's `Response` need `Digestible`? Because the
implementation may use the digest for indexing, deduplication, or
metrics. Forcing it at the trait level saves wrapper boilerplate.

### Same commitment, different requests

If two different `Request` types have the same `Commitment`, they'll
merge in the Originator. If they shouldn't merge (because they ask for
different things), use different `Commitment` types.

## Appendix G — The full message flow

```
Originator                              Handler 1           Handler 2            ...     Handler N
   |                                       |                  |                          |
   |-- Request { commitment: C } --------->|                  |                          |
   |                                       |                  |                          |
   |                                  (process)          (process)                  (process)
   |                                       |                  |                          |
   |<- Response { commitment: C } --------|                  |                          |
   |                                       |                  |                          |
   |                                       |                  |                          |
   |<- Response { commitment: C } -----------------------|                          |
   |                                       |                  |                          |
   |                                       |                  |                          |
   |<- Response { commitment: C } -------------------------------------------|          |
   |                                       |                  |                          |
   | (Monitor::collected called per response, count increments)              |          |
   |                                       |                  |                          |
   | (count == QUORUM → commit / assemble certificate / etc.)                |          |
   |                                       |                  |                          |
   | (cancel(commitment) — stop waiting)                                       |          |
```

The Originator's state machine:

```
1. send(req) → pending[C] = {responses: {}, ...}, broadcast
2. on_response(p, r) → pending[C].responses[p] = r
   → if pending[C].responses.len() >= QUORUM: commit
3. cancel(C) → pending.remove(C)
```

## Appendix H — Where Simplex uses this pattern

In Simplex's Voter, the **Batcher** is essentially a Collector (chapter 11):

```rust
// On incoming notarize vote:
fn on_vote(vote: Notarize) {
    let view = vote.view();
    let key = view;  // commitment == view number
    let pending = self.votes.entry(key).or_insert_with(Pending::default);
    pending.votes.insert(vote.signer, vote);

    if pending.votes.len() >= quorum {
        // Assemble Notarization
        let cert = assemble_certificate(pending.votes.values());
        self.emit(Notarization { ... });
        // Stop collecting more votes
        self.votes.remove(&key);
    }
}
```

Same pattern: pending requests keyed by commitment (view), accumulate
responses, fire when quorum is reached.

## Appendix I — The P2P integration

`collector/src/p2p/`. The P2P-backed implementation:

```rust
pub fn send(&mut self, recipients: Recipients<PK>, request: Request) -> Feedback {
    let encoded = Request::encode(&request);
    self.sender.send(recipients, encoded, false)
}
```

```rust
// On incoming response:
fn on_message(from: PK, message: Response) {
    let commitment = message.commitment();
    self.pending.entry(commitment).or_insert_with(Pending::default)
        .responses.insert(from, message);

    if self.pending[&commitment].responses.len() >= self.quorum {
        // ... assemble + report via Monitor ...
        self.cancel(commitment);
    }
}
```

Standard request/response pattern, with the matching-commitment check.

## Appendix J — Common patterns

### The "wait for all" pattern

In some protocols, you want all responses (even after quorum) for
aggregate verification:

```rust
fn on_response(&mut self, from: PK, response: Response) {
    let pending = &mut self.pending[&response.commitment()];
    pending.responses.insert(from, response);
    self.monitor.collected(from, response, pending.responses.len());

    // Don't commit yet — wait for ALL responses
    if pending.responses.len() == self.recipients.len() {
        self.commit(response.commitment());
    }
}
```

Useful when more responses = stronger aggregate.

### The "early commit" pattern

In other protocols, you commit on first quorum:

```rust
fn on_response(&mut self, from: PK, response: Response) {
    let commitment = response.commitment();
    let pending = &mut self.pending[&commitment];
    pending.responses.insert(from, response);
    self.monitor.collected(from, response, pending.responses.len());

    // Commit on first quorum
    if pending.responses.len() >= self.quorum && !pending.committed {
        pending.committed = true;
        self.commit(commitment);
        // Could cancel or keep collecting
    }
}
```

Useful when you want fastest progress.

## Where to look in the code (expanded)

- `collector/src/lib.rs:21-79` — the traits.
- `collector/src/p2p/` — the P2P-backed implementation.
- `collector/src/p2p/originator.rs` — the originator loop.
- `collector/src/p2p/handler.rs` — the handler loop.
- `consensus/src/simplex/actors/batcher/` — Simplex's use of the Collector
  pattern.

## If you only remember three things

1. **Collector = request with quorum semantics.** Send to N peers, commit when M respond.
2. **Monitor is the observability hook.** Called per-response with the running count.
3. **Committable + Digestible + Codec** — the type bound that ties everything together.

→ Next: **Chapter 11 — Simplex (deep dive)**. We've now covered the substrate.
Time to revisit consensus with everything in hand: runtime, codec, crypto,
p2p, storage, coding, resolver, broadcast, collector. The Simplex algorithm
in full.