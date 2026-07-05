# Chapter 11 — Simplex in full: consensus with the substrate complete

> Re-walking the consensus algorithm now that we know every primitive it uses.

## What we now have

Eleven chapters of substrate behind us. Every primitive Simplex uses:

| Substrate | What it provides | Chapter |
|---|---|---|
| `Runtime` | Tokio or deterministic executor | 02 |
| `Auditor` | SHA-256 of every event for reproducibility | 02 |
| `Simulated Network` | realistic test net with bandwidth, partitions | 02, 05 |
| `Codec` | deterministic, safe wire format | 03 |
| `Cryptography` | Ed25519 / BLS12-381 / threshold signatures | 04 |
| `P2P` | authenticated, encrypted channels, rate-limited | 05 |
| `Storage` | blobs + journals with crash recovery | 06 |
| `Coding` (ZODA) | bandwidth-efficient block dissemination | 07 |
| `Resolver` | fetch data by hash | 08 |
| `Broadcast` | WAN data dissemination with caching | 09 |
| `Collector` (Batcher) | gather votes to form certificates | 10 |

Now let's see how they all compose inside Simplex.

## The trait surface — what YOU implement

Open `consensus/src/lib.rs:99-244`. Six traits you have to implement to plug your application in:

### `Automaton` (line 101)

```rust
pub trait Automaton: Clone + Send + 'static {
    type Context;
    type Digest: Digest;

    fn propose(&mut self, context: Self::Context)
        -> impl Future<Output = oneshot::Receiver<Self::Digest>> + Send;

    fn verify(&mut self, context: Self::Context, payload: Self::Digest)
        -> impl Future<Output = oneshot::Receiver<bool>> + Send;
}
```

**Propose**: when it's your turn to lead, build a payload. Return a digest
(via the oneshot channel) once you have one. If you can't (still syncing,
no transactions, etc.), drop the channel — that's treated as a missing
proposal, triggers `nullify`.

**Verify**: when someone else proposes, check if the payload is valid. Keep
the request **pending** while the verdict might change. Return `false` only
for permanently invalid payloads. (The doc warns specifically: don't return
`false` because of "missing dependencies" — just wait.)

### `CertifiableAutomaton` (line 153)

Adds one method:

```rust
pub trait CertifiableAutomaton: Automaton {
    fn certify(&mut self, _round: Round, _payload: Self::Digest)
        -> impl Future<Output = oneshot::Receiver<bool>> + Send {
        // default: always certify
    }
}
```

This is the chapter 07 hook. With erasure coding, you wait for shards to
arrive, reconstruct, verify hash matches, then return `true`. Default impl
always returns `true` (skip the wait).

### `Relay` (line 197)

```rust
pub trait Relay: Clone + Send + 'static {
    type Digest: Digest;
    type PublicKey: PublicKey;
    type Plan: Send;
    fn broadcast(&mut self, payload: Self::Digest, plan: Self::Plan) -> Feedback;
}
```

When the engine wants to ship a payload to peers, it calls this. Default
implementation: `broadcast::buffered::Engine` from chapter 09 (uses ZODA
under the hood).

### `Reporter` (line 216)

```rust
pub trait Reporter: Clone + Send + 'static {
    type Activity;
    fn report(&mut self, activity: Self::Activity) -> Feedback;
}
```

Observability hook. The engine reports "I voted for block X in view Y" or
"I saw a Byzantine fault from peer Z". Your implementation decides what to
do (log it, slash the peer, reward the voter, etc.).

### `Monitor` (line 236)

```rust
pub trait Monitor: Clone + Send + 'static {
    type Index;
    fn subscribe(&mut self) -> impl Future<Output = (Self::Index, mpsc::Receiver<Self::Index>)> + Send;
}
```

Subscribe to "the latest finalized index." Lets other parts of your system
know when consensus has progressed.

### `Block` and `CertifiableBlock` (lines 59-79)

```rust
pub trait Block: Heightable + Codec + Digestible + Send + Sync + 'static {
    fn parent(&self) -> Self::Digest;
}

pub trait CertifiableBlock: Block {
    type Context: Clone + Encode;
    fn context(&self) -> Self::Context;
}
```

Your block type. `parent()` chains blocks. `context()` provides consensus
metadata (proposer, view, epoch) that gets committed to the digest for
certification safety.

## The four-actor architecture

From `consensus/src/simplex/actors/mod.rs:1-5` and the ASCII diagram in
chapter 01:

```
        +------------+         Peers
        |            +-------> (votes, certificates)
        |  Batcher   |
        |            |<------- (votes, certificates from us)
        +-------+----+
                |
                v
+---------------+       +---------+
|               |<----->|         |
|  Application  |       |  Voter  |  <------> Peers
|               +------>|         |
+---------------+       +-+-------+
                          |
                          v
                     +----+----+
                     |         |
                     |Resolver |  <------> Peers
                     |         |
                     +---------+
```

### The `Batcher` (`actors/batcher/`)

Responsibilities:
1. Receive incoming `Notarize`, `Nullify`, `Finalize` messages from peers
   on the "vote" channel.
2. **Batch verify** at quorum — verify all signatures in a batch when you
   have enough to form a certificate (chapter 04). This is the laziness
   optimization: don't verify until you have to.
3. If a batch fails, **bisect** to find the bad message and `block!` the
   sender (chapter 05).
4. Forward assembled certificates to the Voter.

The batcher's internals are split into:
- `actor.rs` — the main loop
- `round.rs` — per-view state
- `verifier.rs` — the batch-verify and bisect logic
- `ingress.rs` — mailbox and message types

### The `Voter` (`actors/voter/`)

The brain of the engine. Six files:
- `actor.rs` — main loop, view transitions
- `round.rs` — per-view state machine
- `state.rs` — long-lived state (the journal from chapter 06)
- `slot.rs` — data structures for current/finalized slots
- `ingress.rs` — mailbox and message types

The voter does the chapter 01 dance: enter view, start timers, listen to
batcher, drive participation. When `2f+1` notarizations arrive, ask
application to certify, then broadcast `finalize`.

### The `Resolver` (`actors/resolver/`)

Wraps `commonware-resolver` (chapter 08) for the consensus-specific use
case. When the Voter needs a certificate from a prior view (to verify a
proposal's parent), it asks the resolver, which asks peers, which respond
with the missing certificate.

### The `Application`

Your code. The Voter calls `application.propose(...)` when you're the
leader, `application.verify(...)` when others propose, and
`application.certify(...)` after a notarization.

## A single view, end to end

Say it's view 5, you're not the leader. Here's what happens at your node,
with every substrate called out:

```
Voter enters view 5.
  │
  ▼
Deterministic RNG (chapter 02) selects leader = peer L.
  │
  ▼
Voter starts t_l = 2Δ, t_a = 3Δ via Clock::sleep (chapter 02).
  │
  ▼
Batcher receives Notarize(c, 5) from peer L over authenticated P2P (chapter 05).
  │
  ▼
Batcher stores it in the in-progress vote set.
  │
  ▼
...wait until quorum...
  │
  ▼
Batcher has 2f+1 Notarize(c, 5) from unique signers (chapter 04).
  │
  ▼
Batcher batch-verifies the signatures (chapter 04 BatchVerifier with
randomness to prevent rogue-key attacks).
  │
  ▼
If valid: Batcher forms Notarization(c, 5) certificate, forwards to Voter.
If invalid: bisect, block!(offending peer) (chapter 05 block macro).
  │
  ▼
Voter cancels t_a (chapter 02 Clock). Marks c as notarized.
  │
  ▼
Voter broadcasts Notarization(c, 5) to peers via P2P (chapter 05).
  │
  ▼
Voter calls Application::verify(c) via oneshot channel (lib.rs:141-145).
  │
  ▼
Application verifies. If using coding (chapter 07), Application waits for
shards via Resolver, reconstructs, checks hash, returns true.
  │
  ▼
If verify fails: Voter broadcasts Nullify(5).
If verify succeeds:
  │
  ▼
Voter calls Application::certify(round, c) via oneshot channel (lib.rs:179).
  │
  ▼
Application decides: do I have the data? is it permanently uncertifiable?
Returns true/false.
  │
  ▼
If certify=true: Voter broadcasts Finalize(c, 5) via P2P.
If certify=false: Voter broadcasts Nullify(5).
  │
  ▼
Voter enters view 6.
```

That's the whole view in one diagram. Every box on the side is a chapter
we've read.

## The write-ahead log — crash recovery

The Voter (`actors/voter/state.rs`) holds a journal (chapter 06) at
partition `"log"`. On every state-changing action, it writes to the journal
and (before broadcasting) syncs to disk.

The pattern:

```rust
// Hypothetical voter code
fn on_finalize(&mut self, cert: Finalization) {
    // 1. Write to WAL
    self.journal.append(self.partition, cert.encode()).await.unwrap();

    // 2. Sync before broadcasting
    self.journal.sync_all().await.unwrap();

    // 3. Broadcast to peers
    self.sender.send(Recipients::All, cert.encode(), false).await.unwrap();
}
```

Why this order? Because if we crash between step 1 and step 3, on restart
we'll replay the WAL and re-broadcast. If we broadcast first and crashed
before writing to WAL, on restart we'd see "I voted for block B" in other
peers' logs but no record of it ourselves — Byzantine behavior.

This is the pattern from chapter 06: the WAL is the source of truth.

## Recovery on restart

The full restart sequence:

```
Boot.
  │
  ▼
Open Storage, scan partition "log".
  │
  ▼
Initialize Voter with the journal. Replay (chapter 06) yields all past
messages received and sent.
  │
  ▼
Rebuild in-memory state from replayed messages.
  │
  ▼
Apply the same state-machine transitions that produced them.
  │
  ▼
Now the engine is at the same state it was before the crash.
  │
  ▼
Catch up on any consensus progress that happened during downtime (via
Resolver fetching missing certificates).
  │
  ▼
Resume participation.
```

If the journal is corrupted (e.g., we crashed mid-write), chapter 06's
auto-repair kicks in: truncate to last valid item, replay, recover.

## The test pattern

From `consensus/src/simplex/mod.rs:790-1020` (the `all_online` test), here's
how every chapter's substrate shows up in one test:

```rust
fn all_online<S, F, L>(mut fixture: F)
where
    S: Scheme<Sha256Digest, PublicKey = PublicKey>,
    F: FnMut(&mut deterministic::Context, &[u8], u32) -> Fixture<S>,
    L: Elector<S>,
{
    let n = 5;
    let executor = deterministic::Runner::timed(Duration::from_secs(300));   // CH 02
    executor.start(|mut context| async move {
        let Fixture { participants, schemes, .. } = fixture(&mut context, &namespace, n);

        let mut oracle =
            start_test_network_with_peers(context.child("network"), participants.clone(), true).await;  // CH 05

        let mut registrations = register_validators(&mut oracle, &participants).await;

        let link = Link { latency: Duration::from_millis(10), jitter: Duration::from_millis(1), success_rate: 1.0 };
        link_validators(&mut oracle, &participants, Action::Link(link), None).await;  // CH 05

        let elector = L::default();
        let relay = Arc::new(mocks::relay::Relay::new());    // CH 09 (broadcast)
        let mut reporters = Vec::new();
        let mut engine_handlers = Vec::new();

        for (idx, validator) in participants.iter().enumerate() {
            let reporter_config = mocks::reporter::Config { ... };   // CH 04 (scheme)
            let reporter = mocks::reporter::Reporter::new(...);
            reporters.push(reporter.clone());

            let (actor, application) = mocks::application::Application::new(...);  // CH 11 (app)
            actor.start();
            let cfg = config::Config {
                scheme: schemes[idx].clone(),                  // CH 04
                elector: elector.clone(),
                blocker: oracle.control(validator.clone()),     // CH 05
                automaton: application.clone(),
                relay: application.clone(),
                reporter: reporter.clone(),
                strategy: Sequential,                          // CH 07
                partition: validator.to_string(),
                mailbox_size: NZUsize!(1024),
                epoch: Epoch::new(333),
                floor: config::Floor::Genesis(...),
                leader_timeout: Duration::from_secs(1),
                ...
                page_cache: CacheRef::from_pooler(...),         // CH 06
                forwarding: ForwardingPolicy::Disabled,
            };
            let engine = Engine::new(context.child("engine"), cfg);

            let (pending, recovered, resolver) = registrations.remove(validator).unwrap();
            engine_handlers.push(engine.start(pending, recovered, resolver));
        }

        // ... wait for all engines to make progress ...

        for reporter in reporters.iter() {
            reporter.assert_no_faults();      // CH 11 (Byzantine detection)
            reporter.assert_no_invalid();     // CH 04 (signature verification)
            // Verify certificates for all views
            ...
        }

        let blocked = oracle.blocked().await.unwrap();   // CH 05
        assert!(blocked.is_empty());   // no peer got blocked → no Byzantine behavior
    });
}
```

Every line maps to a chapter. This is the integration test that exercises
the whole stack — runtime, simulator, codec, crypto, p2p, storage, marshal,
consensus.

## Byzantine scenarios the test catches

Byzantine mocks (`consensus/src/simplex/mocks/conflicter.rs`) replace a
normal peer with a misbehaving one:

```rust
if idx_scheme == 0 {
    // Create Byzantine actor instead of normal engine
    let cfg = mocks::conflicter::Config { ... };
    let engine = mocks::conflicter::Conflicter::new(context, cfg);
    engine.start(pending);
} else {
    let engine = Engine::new(context, cfg);  // honest
    engine.start(pending, recovered, resolver);
}
```

Common scenarios:
- **Conflicter** — sends two different proposals for the same view.
- **Nuller** — never participates.
- **Liar** — sends invalid signatures.
- **Reorderer** — sends messages out of order.

The test asserts `assert_no_faults()` and `assert_no_invalid()` to verify
the honest nodes correctly detected and ignored the Byzantine ones.

## The full picture

```
+----------------------------------------+
|  Your Application                      |  ← Automaton, CertifiableAutomaton
+----------------------------------------+
                ↕
+----------------------------------------+
|  Simplex::Engine                       |  ← Voter + Batcher + Resolver
|  - Voter: state machine, timers        |
|  - Batcher: vote collection, batch verify
|  - Resolver: fetch missing certificates
+----------------------------------------+
                ↕
+----------------------------------------+
|  substrate primitives                  |
|  - P2P (authenticated channels)        |  ← CH 05
|  - Storage (journal for WAL)           |  ← CH 06
|  - Coding (block dissemination)        |  ← CH 07
|  - Resolver (peer fetches)             |  ← CH 08
|  - Broadcast (caching engine)          |  ← CH 09
|  - Crypto (signatures, schemes)        |  ← CH 04
+----------------------------------------+
                ↕
+----------------------------------------+
|  Runtime                               |  ← CH 02
|  - Tokio (prod) or Deterministic (test)|
|  - Storage trait, Network trait        |
|  - Spawner, Clock, Supervisor          |
|  - Auditor (SHA-256 reproducibility)   |
+----------------------------------------+
```

That's the whole system. Twelve chapters to get here.

## Where to look in the code

- `consensus/src/lib.rs:99-244` — the trait surface.
- `consensus/src/simplex/actors/mod.rs:1-5` — the four actors.
- `consensus/src/simplex/actors/voter/actor.rs` — the Voter's main loop.
- `consensus/src/simplex/actors/batcher/verifier.rs` — batch verification.
- `consensus/src/simplex/actors/resolver/` — the resolver integration.
- `consensus/src/simplex/mod.rs:30-68` — the protocol spec.
- `consensus/src/simplex/mod.rs:790-1020` — the integration test.
- `examples/log/src/main.rs` — a tiny running app using all of this.

## The full Simplex algorithm in pseudocode

`consensus/src/simplex/mod.rs:29-118` is the authoritative spec.
Here's a tight pseudocode rendering that maps line-for-line to the
module's prose — useful for the formal-style reader who wants to
hold the whole algorithm in their head.

```
# Per-validator state.
Voter {
    view: View
    journal: WAL
    leader_timeout: Timer(t_l)
    activity_timeout: Timer(t_a)
    has_nullified_view: bool   # have I sent nullify(v)?
    has_finalized_view: bool   # have I sent finalize(c,v)?
}

# Entry point: enter view v.
fn enter_view(v):
    self.view = v
    leader = self.scheme.elector.determine_leader(v, ...)

    # Heartbeat timers. Δ = network round-trip.
    self.leader_timeout = now + 2 * Δ
    self.activity_timeout = now + 3 * Δ

    self.has_nullified_view = false
    self.has_finalized_view = false

    # Request missing certificates for ancestry. The Voter asks
    # Resolver to fetch any nullifications from v_floor to v
    # that haven't been seen.
    request_missing_nullifications(v_floor..v)

    # If we're the leader, propose.
    if self.public_key == leader:
        proposal = self.automaton.propose(...)
        broadcast(Notarize(proposal, v), self.public_key)

# Triggered by incoming notarize vote for view v.
fn on_notarize(peer, container, v):
    if v != self.view: return
    if not self.has_seen(peer): add to vote set

    if peer == self.elector.determine_leader(v) and count(notarizes for c) >= 1:
        cancel(leader_timeout)

    # Eligibility check: parent must be finalized, all views
    # between parent and v must have nullifications.
    if can_propose_ancestry(container):
        # Application verifies the payload.
        if await self.automaton.verify(container):
            broadcast(Notarize(container, v), self.public_key)
        else:
            broadcast(Nullify(v), self.public_key)

# Triggered by reaching quorum on notarize votes for c, v.
fn on_notarize_quorum(c, v):
    cancel(activity_timeout)
    mark_notarized(c, v)

    # Always broadcast the certificate, even if we haven't verified.
    broadcast(Notarization(c, v), self.public_key)

    # Now try to certify (wait for data delivery, etc.).
    if await self.automaton.certify(v, c):
        broadcast(Finalize(c, v), self.public_key)
    else:
        broadcast(Nullify(v), self.public_key)

# Triggered by reaching quorum on nullify votes for view v.
fn on_nullify_quorum(v):
    broadcast(Nullification(v), self.public_key)
    enter_view(v + 1)

# Triggered by reaching quorum on finalize votes for c, v.
fn on_finalize_quorum(c, v):
    mark_finalized(c, v)
    recursively finalize(c.parent)
    broadcast(Finalization(c, v), self.public_key)
    # NB: don't enter v+1 here — already advanced past it.

# Leader timeout: leader didn't propose in 2Δ.
fn on_leader_timeout():
    broadcast(Nullify(self.view), self.public_key)

# Activity timeout: 2f+1 votes haven't arrived in 3Δ.
fn on_activity_timeout():
    broadcast(Nullify(self.view), self.public_key)
```

The actual Commonware source is over 200 lines of Rust in
`consensus/src/simplex/actors/voter/`. The pseudocode strips it
down to the salient state transitions.

A few subtleties the pseudocode hides:

1. **Lazy verification.** `verify` and `certify` return `bool` in
   the pseudocode; in source they're `oneshot::Receiver<bool>`. The
   Voter keeps awaiting the channel until either the verdict
   arrives or the view transitions (in which case the verdict is
   dropped).
2. **Concurrent voting.** Multiple votes can race. The Voter uses
   `BTreeMap<PK, Vote>` for deduplication.
3. **WAL writes.** Every "broadcast" in the pseudocode implicitly
   means "write to WAL, sync, then broadcast." Crash recovery
   replays the WAL.

## PBFT phases — and where Simplex departs

Castro-Liskov's **Practical Byzantine Fault Tolerance (PBFT, 1999)**
established the byzantine-fault-tolerant consensus template. Many
follow-on protocols (Tendermint, HotStuff, Simplex, etc.) are
PBFT-family with simplifications. Understanding PBFT's structure
makes Simplex's choices visible.

**PBFT's three phases** (per round):

```
Pre-prepare  Leader → Replicas   { pre-prepare(view, digest) }
Prepare      Replica → Replicas  { prepare(view, digest, i) }
Commit       Replica → Replicas  { commit(view, digest, i) }
```

Each phase has a quorum-collection step:

- After pre-prepare: leader announces the next digest.
- After prepare: 2f+1 replicas agree the pre-prepare was valid.
- After commit: 2f+1 replicas agree to commit.

Total messages per round: `O(N²)` per phase (all-to-all), times 3
phases = `O(N²)` but with a 3x multiplier of actual sent bytes.
Three *round trips* per decision.

**Where Simplex departs:**

| Property | PBFT | Simplex |
|---|---|---|
| Phases | 3 (pre-prepare + prepare + commit) | 2 (propose + certify) |
| Leader election | Rotating, view number modulo N | VRF over view number, view * height |
| Certificate structure | Pre-prepare + prepare-set | Notarize vote → Notarization |
| View change | New view message + new-view collection | `nullify(v)` + next view |
| Finalization | After commit phase | After separate finalize vote |
| Messages per decision | O(N²) | O(N × 2) = O(N) |

The deeper shift: PBFT's `prepare` and `commit` are *implicit
votes on the proposal's validity*. Simplex collapses them into
`notarize(c, v)` — the certifies-when-canonical signal is encoded
in the vote's namespace and signature. The "finalize" vote (`f+1`
later) is a separate, weaker signal — "I've seen this notarization,
I'm willing to finalize."

Three round trips for Simplex finality, two for optimistic
PBFT — and Simplex's two round trips are *better* in adverse
conditions because PBFT has to handle view changes within the
three-phase logic, while Simplex handles them via `nullify` which
is a clean separate path.

For deployments (Alto chain, see `examples/alto`), the *practical*
advantage of Simplex is the linear message complexity per view:
O(N) messages rather than O(N²). For N=100, that's a 100x reduction
in network bandwidth during consensus.

**The 2-hop / 3-hop latency.** Simplex's "wicked fast 2 network
hops" claim (from `consensus/src/simplex/mod.rs:9`):

```
hop 1: leader proposes notarization  →  peers receive
hop 2: peers notarize                   →  certificate at 2f+1
```

That's 2 hops for a `notarization` (speculative finality).
Then 1 more hop for `finalize(c, v)` votes. So 3 hops total for
full finalization. PBFT's 3 hops *also* give you finalization —
the trade-off is message count, not hop count.

## Tendermint vs Simplex vs HotStuff vs PBFT

Here's a tidy comparison table for the four major BFT consensus
families. Simplex is closest to Tendermint in structure; PBFT is
the canonical baseline; HotStuff is the modern "linear" family.

| Property | PBFT | Tendermint | HotStuff | **Simplex** |
|---|---|---|---|---|
| Year | 1999 | 2014 | 2018 | 2023 |
| Phases per decision | 3 | 2 (propose + precommit) | 3 (Chained) | 2 + finalization |
| Messages per decision (steady state) | O(N²) | O(N²) | O(N) | **O(N)** |
| Hops per decision | 3 | 3 | 3 (or 1 with chaining) | **2 (optimistic) / 3 (full)** |
| View-change complexity | O(N³) | O(N²) | O(N) | **O(N)** (with `nullify`) |
| View-change message | `new-view` | `Round2` step | `NewView` | `Nullify` vote + view-step |
| Leader rotation | `v mod N` | `v mod N` (rotating) | Chained (pipelined) | **VRF over (view, parent)** |
| Finalization timing | Commit phase | Precommit → Commit | 3-chain | **3 votes** (notarize+finalize) |
| Linearizability? | Yes | Yes | Yes | Yes |
| Optimistic case latency | 2-3 hops | 2-3 hops | 1-3 hops (chained) | **2-3 hops** |
| Worst-case latency | O(N) views × N messages | O(N) views | O(N) views | O(N) views (with skip-leader) |
| Signature scheme | Threshold or multisig | Same | Threshold BLS | **Pluggable (Ed25519, BLS, secp256r1)** |
| View-change certificate | `new-view` quorum | `Round2` quorum | `HighQC` quorum | **`Nullification` quorum** |
| Fork rule (`Safety`) | Single committed | Single committed | Single committed (3-chain) | **Single notarized-and-uncertified-excluded** (`forced inclusion`) |
| Common use | Hyperledger Fabric | Cosmos Hub | Flow, Aptos | **Alto (Commonware chain)** |

A few columns you might not have seen:

- **Linearizability?** All four give you linearizability of finalized
  blocks (a stronger consistency than serializability for the
  consensus output).
- **Forced inclusion** (column shown for Simplex). A notarization
  cert cannot be skipped by future proposals if no nullification
  cert exists for it. Tendermint and PBFT lack this property —
  their nullification depends on round timeouts and can be
  preemptively by later views.
- **View-change complexity.** PBFT's view change (`new-view`) is
  O(N³) because of `new-view` + `prepare` + `commit` *within the
  view-change protocol*. Tendermint's Round2 is O(N²). HotStuff
  reduced this to O(N) by chaining view changes through QC
  references. Simplex's nullify-as-vote is also O(N).

The "messages per decision" row is the operational fact: each
view's communication is a function of N. PBFT's quadratic
cubic-cost view-change made it impractical for `N` in the
hundreds. Simplex and HotStuff scaled because of their linear
behavior.

A final caveat: HotStuff's `O(N)` view-change assumes a leader
proposes the next view (pipeline). Simplex's nullify-then-next-view
is `O(N)` because `nullify` is a flat broadcast, not a chain of
phases.

## BLS threshold scheme with VRF

The `simplex::scheme::bls12381_threshold` module
(`consensus/src/simplex/scheme/bls12381_threshold/`) is the
threshold-sig flavor of Simplex. It pairs BLS12-381 threshold
signatures (chapter 04) with a **verifiable random function (VRF)**
for leader election.

Two parts to unpack:

**BLS threshold signatures.** A `(t, n)` threshold scheme lets any
`t` of `n` participants jointly produce a signature, but no smaller
group can. The signature is the *same* regardless of which `t`
signers contributed (the Lagrange coefficients fold them into one
combined signature). Verification takes one pairing op.

For consensus, `t = 2f+1` and `n = 3f+1` gives the
classic threshold: any `2f+1` of `3f+1` validators produce a valid
certificate, but `f + 1` cannot.

The advantages over multisig:

- **One signature to verify, not N.** Pairing op (`G2 × G1 → GT`).
- **One signature to transmit.** O(1) bytes regardless of `n`.
- **Aggregation is deterministic and order-independent.** No rogue-key
  attacks if you mix in a randomness source on aggregation.

`is_batchable()` returns `true` for BLS threshold; Simplex's Batcher
uses `BatchVerifier::add` to accumulate votes and then a single
`BatchVerifier::verify(rng, strategy)` to check all of them at once.

**VRF for leader election.** A VRF turns `(secret_key, input)` into
a deterministic but unpredictable `(output, proof)`. Anyone with
`public_key` and `proof` can verify `(input, output, proof)` and
confirm the output was produced by that key.

Why a VRF?

- **Deterministic** — given the same `(sk, view)`, the leader is
  the same on every node. No coordination needed.
- **Unpredictable** — without the secret key, the output is a
  random-looking 32 bytes. An attacker can't grind for "I'd like
  to be leader in view 17."
- **Verifiable** — every node can check that the leader was chosen
  by the VRF, not by hand.

The simplex scheme uses the VRF like this (`simplex/scheme/bls12381_threshold/vrf.rs`):

```
leader_for(view, parent_digest):
    # Concatenate the view number and the parent block's digest.
    input = H(view || parent)
    # Each validator runs the VRF and checks if it qualifies.
    (output, proof) = vrf.prove(my_sk, input)
    # Leader = the validator whose output is smallest (or whose output is
    # < some threshold). Provides probabilistic rotation.
    if output < threshold:
        return (my_pk, output, proof)
    else:
        return None
```

The full leader election iterates: each validator computes its VRF,
and only those whose output is below a threshold propose. For most
views, exactly one validator qualifies. For some views, multiple do
(probabilistic multi-leader; the others nullify).

The `Elector` trait abstracts this (`simplex::elector`). Each
scheme has its own: BLS threshold uses VRF; ed25519 and others use
round-robin or hash-based.

## The "View" abstraction in Simplex

A **view** is Simplex's unit of consensus progress. Each view:

1. Has a single leader (chosen by the Elector).
2. Produces at most one notarized block.
3. Either notarizes a block, nullifies (timeout), or both fail.

The view's lifecycle:

```
enter_view(v)
  → attempt to notarize c chosen by leader of v
  → either notarization reaches 2f+1 (success)
  → or nullification reaches 2f+1 (move on)
  → or both timers expire and we nullify locally
```

**View vs round.** Different protocols use these words differently.
In Paxos and Raft, a "round" is a (ballot-number, proposer-id)
pair — a contiguous space where proposers compete. In Simplex, a
"view" is tied to a single leader; round = view. Different numbers
mean different leaders.

In PBFT, "view" and "view-change" are similar (single leader per
view, view-change on leader failure). Simplex matches this usage.

A critical clarity: **view numbers are monotonically increasing
within an epoch**. If you see view 17's notarization, view 18 must
build on view 17's block (or its empty post-nullification state).
Time advances forward in view numbers, never backward.

The skip-leader mechanism (chapter 11 main text + Appendix K)
introduces *gap* views — you can advance from view 5 directly to
view 9 if views 6, 7, 8 had nullifications. The empty slots
aren't lost; they're recorded as nullifications.

**Views across multiple heights.** Simplex's view number is *per
chain*, not per height. The same `v` can refer to the leader of a
proposed block at height `h_v`. Different blocks at different
heights can have the same view number *only if* they propose
sequentially — block at `h` in view `v` is followed by block at
`h+1` in view `v+1` (or `v+k` for some `k` if there's been a
sequence of nullifications).

The spec's *Forced Inclusion* property (chapter 11 main text +
`consensus/src/simplex/mod.rs:121-138`) leverages this:

> A notarization in view `v` cannot be skipped by a future proposal
> unless a nullification cert for `v` exists. Because honest
> validators never broadcast `nullify(v)` for a notarized view,
> no Byzantine subset of size `f` can produce a nullification cert
> either. So the notarization is "in" the chain forever.

## Linearizability vs serializability

Simplex's consensus output is **linearizable**, not just serializable.
The distinction matters for systems that build on top of consensus.

**Serializability.** A history of transactions is serializable if
it produces the same result as some serial order of those
transactions. The serial order need not be the same as real time —
you could reorder to maximize throughput.

**Linearizability.** A history is linearizable if it produces the
same result as some order where each operation's "linearization
point" lies between its invocation and its response. In other
words, the order *respects real time*.

For Simplex's output: a finalized block B at view V has a
linearization point at the moment the `finalization(c, V)` votes
reached `2f+1`. Any block finalized "after" B (in view V' > V) has
a linearization point after B's. Real-world time and consensus time
agree.

Why linearizability (the stronger property) for consensus:

- A blockchain's transactions happen at specific wall-clock times.
  Linearizability gives those times real meaning.
- Cross-chain bridges (chapter 12 `examples/bridge`) need
  linearizability — a header finalized at time T must precede a
  withdrawal initiated at time T+1.
- Composing protocols: if protocol A produces linearizable output
  and protocol B depends on it, B's view of A is also
  linearizable.

What Simplex does *not* linearize: the *notarization* phase. A
block can be notarized (speculative finality) and then fail
certification, producing a nullification. The notarization was
incorrect as a "commit" — only `finalization` linearizes.

The block-stability properties Simplex *does* provide:

1. **Finalization = linearization.** Once `finalization(c, V)` is
   observed, the chain contains `c` at its position.
2. **Forced inclusion = monotonicity.** If `c` is notarized in `V`
   without timeout, every future proposal has `c` in its ancestry.
3. **Within-epoch consistency.** All honest validators agree on the
   chain within an epoch.

## Bracha's reliable broadcast — and how Simplex simplifies it

Bracha's **reliable broadcast (1994)** is the original
Byzantine-fault-tolerant broadcast protocol. It's worth knowing
because it's the foundation of all "send a value to many peers"
constructions in BFT land.

Bracha's three phases:

```
ECHO:   i → all:    "ECHO(m, source)"
READY:  i → all:    "READY(m, source)"     (once 2f+1 ECHO received)
→ DELIVER m           (once 2f+1 READY received)
```

Each phase is a quorum. The protocol:

1. Sender broadcasts a `SEND(m)`. Replicas enter ECHO phase.
2. A replica that has 2f+1 ECHO votes for `m` switches to READY.
3. A replica that has 2f+1 READY votes delivers `m`.

If the sender is honest, ECHO phase succeeds in one round, READY in
the next, DELIVER after the second. Total: 2 round trips (3 in
Simplex notation because it includes the broadcast).

If the sender is Byzantine, they might split ECHOs across different
ms. But to deliver, you need 2f+1 ECHO + 2f+1 READY. A Byzantine
sender can't deliver *two different values* — because that would
require 2f+1 ECHO for each, which sum to 4f+2, exceeding `n = 3f+1`.

**Where Simplex simplifies.** The consensus broadcast is reliable
because *quorum intersection* makes it so — once a notarization
forms, it's a certificate that's stable across all honest
participants. Simplex doesn't need Bracha's two rounds because:

- The leader's proposal is implicitly endorsed by the 2f+1 notarization votes.
- The notarization cert *is* the "DELIVER" step — its existence
  proves a quorum agreed.
- The finalize phase is a separate opt-in (you only finalize if
  you have the data).

The simplification lets Simplex skip Bracha's ECHO and READY
phases entirely, saving two round trips in the optimistic path.

For the protocol's own block broadcast (sending the *bytes* of
the block to all peers), Bracha-style reliability is still
relevant — but Commonware's `broadcast::buffered::Engine` (chapter
09) handles that via gossip + reference counting, not via
Bracha-style quorum.

## The recovery protocol in detail

Crash recovery for Simplex is the chapter 06 WAL pattern applied
to consensus:

**What's persisted.** On every state-changing action (vote,
certificate, finalize), the Voter writes to its journal *before*
broadcasting:

```rust
self.journal.append(partition, encoded_message).await?;
self.journal.sync_all().await?;     // fsync before broadcast
self.vote_sender.send(Recipients::All, encoded, false).await?;
```

What this captures:

- All votes we've sent (in case we need to re-send after a crash).
- All certificates we've verified (`Notarization`, `Finalization`).
- The view number we were on when we crashed.

What's *not* persisted:

- In-flight `oneshot::Receiver<bool>` futures (from
  `Automaton::verify` etc.) — these get re-invoked on restart.
- Peer mailbox contents — received messages not yet processed.
- Network timers — the deterministic runtime resumes them.

**What gets recovered on restart** (`consensus/src/simplex/actors/voter/state.rs`):

1. Open the journal, scan partition "log".
2. Replay (chapter 06) yields all past items.
3. For each item, decode and reapply to in-memory state:

```rust
match decode(item) {
    Notarize(vote)   => recovered_votes.push(vote),
    Nullify(vote)    => recovered_nullifies.push(vote),
    Finalize(vote)   => recovered_finalizes.push(vote),
    Notarization(c)  => recovered_notarizations.push(c),
    // ...
}
```

4. Rebuild the in-memory structures (`votes_by_view`,
   `notarizations_by_height`, `finalized`).
5. Detect the current view from recovered state.
6. Resume participation.

**What happens to in-flight requests.**

- `Automaton::verify` futures: dropped on crash. Commonware's
  convention is for the application to be idempotent — re-issuing
  `verify` with the same `(context, payload)` is safe.
- Resolver fetches: dropped on crash. The Resolver retries on
  restart (chapter 08).
- Mailbox contents: lost. Re-incoming messages will be processed
  again — duplicate detection via signatures handles this.

**The rebroadcast guarantee.** Suppose we crashed between "voted
`Finalize(c, v)`" and "broadcast my vote". On restart:

1. Replay WAL: we have `Finalize(c, v)` in the log.
2. Reapply: vote seen as if we just voted.
3. Network layer sends the vote to peers (re-broadcast).

Peers who already saw our vote see a duplicate and drop it
(handshake includes a uniqueness check). Peers who hadn't seen it
now have it. Either way, consensus progresses.

**The Byzantine safety argument.** Without the WAL-then-broadcast
order, you'd be in trouble:

- Vote received, process, broadcast, crash → WAL might be partial.
- On restart, replay sees no vote → don't broadcast.
- Peers saw our vote → they have a vote they attribute to us but
  we don't have it. Contradicts ourselves.

With the WAL-then-broadcast order, we're guaranteed: if the
broadcast left our process, the WAL has a record of it. Replay
re-broadcasts; we don't contradict ourselves.

## Performance analysis — bottlenecks and throughput

A 100-validator consensus cluster running Simplex hits these
bottlenecks at the steady state:

**Per-decision message count:** O(N) — one notarize vote per
validator (from leader's proposal) and one Notarization cert for
finalization. For N=100, that's 100 messages per view.

**Per-decision bandwidth:**

```
Send side:
   leader:  proposal (kB) → 99 peers        = 99k × proposal_size
   peer:    notarize vote (32+sig) × 99 peers = O(N)
   peer:    finalize vote × N votes          = O(N)

Receive side:
   leader:  1 vote per peer (N-1 votes)
   peer:    1 proposal + 1 cert per view     = O(1)

Total:    O(N) per decision in steady state
```

**Per-view latency:**

```
optimistic: t_l ≈ 2Δ = 200ms (one round to propose, one to notarize)
finalization: +1 hop = 3Δ = 300ms
```

For a 300ms finalization, that's ~3.3 decisions/sec. For
throughput > 3 decisions/sec, you need to *batch* — which
Simplex does by including many transactions in one block.

**Throughput ceiling.** A 1 MB block at 3.3 decisions/sec gives
~3.3 MB/s of consensus bandwidth. With coding (chapter 07) the
per-peer sender bandwidth is `O(N) × message_size / N` — so this
scales with `block_size / round_trip`. For 10 MB blocks at
3 decisions/sec, you get 30 MB/s, but the leader's egress must
support >10 MB per ~30ms which is 333 Mbps uplink — tight.

**Common bottlenecks in production:**

1. **Block size** → limited by leader's egress.
2. **Round trip** → limited by geo distribution.
3. **Signature verification** → BLS threshold is fast but not
   free; Ed25519 multisig is fast; secp256r1 is slow.
4. **WAL fsync** → every vote-casting does an fsync. With NVMe,
   ~100μs per fsync. With HDD, 10ms — your consensus rate drops
   to < 100/s.
5. **CPU for hash** → SHA-256 of block → 1 GB/s on a modern CPU,
   fine for 10 MB blocks but tight at 100 MB.

The Commonware authors, in their *carnot-bound* blog post,
explore the *theoretical* peak: how fast can a BFT consensus go
on commodity hardware? The answer depends mostly on your WAL
throughput and your network latency floor.

## Rust patterns in Simplex

Three patterns specific to Simplex's source:

**`BTreeMap` vs `HashMap` in voter state.** The Voter uses
`BTreeMap<View, Notarize>` and similar for several reasons:

1. **WAL replay order.** The WAL is appended-to in view order.
   During recovery, we reapply items, and `BTreeMap` keeps things
   sorted — useful when checking "is this view before/after the
   floor?"
2. **Determinism for tests.** `BTreeMap` iteration is sorted;
   tests asserting on the recovered state's hash (via
   `Auditor::state`) need this.
3. **Worst-case O(log N)** for the operation count is fine —
   voter state is at most a few hundred entries per validator.

Where `HashMap` *does* appear: in the resolver's `inflight` map
(`resolver/src/p2p/inflight.rs`). Resolver fetches aren't
versioned the same way voter votes are, and lookups are the hot
path.

**`Arc<RwLock<...>>` for the journal.** The Voter owns the
journal across async tasks (sometimes the voter forks a
background flusher). Wrapping in `Arc<RwLock<Journal>>` lets
multiple tasks coordinate:

```rust
self.journal_handle: Arc<RwLock<Journal<E>>>,
// Acquire for read (concurrent):
let journal = self.journal_handle.read().await;
// Acquire for write (mutating appends):
let mut journal = self.journal_handle.write().await;
journal.append(...).await?;
```

For the Voter itself, the pattern is "main loop owns writes,
background tasks own reads." `RwLock` is the right primitive —
reads are common, writes are rare (one per vote).

For higher-throughput paths, `parking_lot::RwLock` is faster but
the async `tokio::sync::RwLock` is what integrates with `.await`.

**`Pin` in async voting loops.** Some of Simplex's internal
futures are self-referential (a future holds a reference to its
own state across awaits). `Pin` is the Rust mechanism that
prevents you from moving such a future after `.await`:

```rust
let mut future = Box::pin(async move {
    let v = state.elector.determine_leader(view).await;
    state.on_leader(v).await;
});
future.as_mut().poll(...);  // pinned, can't be moved
```

You'll see `Box::pin` and `Pin::as_mut` in the actor loop's
inner machinery (`consensus/src/simplex/actors/voter/actor.rs`).
The reason: the actor's main loop recursively awaits sub-futures
(verify, certify, fetch); these futures borrow state from the
actor; the actor is pinned in a single place (the actor's task),
so the futures stay valid across awaits.

If you see `!Unpin` in a Simplex signature, that's what's going
on. Common cases:

- `Pin<Box<dyn Future<Output = ()> + Send>>` for spawning.
- `impl Future<Output = T> + Send + Unpin` for types that don't
  need to be pinned.
- `Pin<&mut Self>` in the actor's main run loop.

---

## Deepening 1 — The full Simplex algorithm as a state machine

The pseudocode sketch in main body is a useful one-page summary. Here
is the algorithm rendered as a *state machine*: every state the Voter
can be in, every transition out of that state, and every message that
triggers each transition. This is the rendering you want when reading
source or implementing it from scratch.

### The states

A Voter in view `v` is always in one of six logical states. Some are
composed (a Voter can be "in `ProposingReceived` and `NotarizeQuorum`"
at the same time — once notarize quorum forms, the certificate sits
in-state even while the certify verdict is awaited).

```
P_Idle            No view yet, or just transitioned out
P_WaitingLeader   Timer L running; no proposal yet
P_HasProposal     Leader's proposal received, app being asked to verify
P_NotarizedQuorum 2f+1 notarize votes collected; Notarization cert exists
P_WaitingCertify  Notarization in state, App's certify verdict pending
P_Finalized       Finalize quorum reached for THIS view
P_Nullified       2f+1 nullify votes reached OR local nullify emitted
```

Each state has one or more **exits**: events that move the Voter out
into a successor state.

### The state machine, as a transition table

For each state and each event, exactly one transition. The full table:

| From state | Event | Condition | To state | Side effect |
|---|---|---|---|---|
| any | `enter_view(v)` | always | P_WaitingLeader | start timers L,A; reset has_* |
| P_WaitingLeader | `notarize(c, v)` from peer | v == self.view | (in same state) | record vote; cancel L if leader |
| P_WaitingLeader | timer L fires | now >= timer_L | P_Nullified (local) | self.nullify(v) |
| P_HasProposal | `verify` returns true | app says OK | P_NotarizedQuorum | self.notarize(c, v) |
| P_HasProposal | `verify` returns false | app says bad | P_Nullified | self.nullify(v) |
| P_NotarizedQuorum | quorum notarize | votes >= 2f+1 | (same) | assemble Notarization; broadcast |
| P_NotarizedQuorum | `certify` returns true | app says data OK | P_Finalized | self.finalize(c, v) |
| P_NotarizedQuorum | `certify` returns false | app says no data | P_Nullified | self.nullify(v) |
| P_NotarizedQuorum | timer A fires | now >= timer_A | P_Nullified | self.nullify(v) |
| P_Finalized | quorum finalize | votes >= 2f+1 | (same) | broadcast Finalization; mark finalized |
| any | null_quorum for v | nullify >= 2f+1 | P_Nullified | broadcast Nullification |
| P_Nullified | `enter_view(v+1)` | always | P_WaitingLeader | reset |

The `enter_view(v+1)` from any terminal state in view `v` is the **only
way progress happens**. There is no "skip" or "stall."

### The message handlers, in detail

Every message handler in `consensus/src/simplex/actors/voter/actor.rs`
maps to one state transition.

**`on_proposal(c, v)`** — fired when a `notarize(c, v)` from the
leader of view `v` arrives.

```rust
fn on_proposal(&mut self, c: Digest, v: View) {
    if v != self.view { return; }                  // stale, drop
    if self.leader(c.signer) != c.signer { return; } // not leader, drop

    // Cancel leader timer (we heard from the leader at least)
    self.cancel_timer_l();

    // Eligibility check: parent finalized, ancestry gaps filled
    if !self.can_propose_ancestry(c) {
        // Could be that we're behind; kick off a resolver fetch
        self.resolver.fetch(c.parent).await;
        return;       // we'll reprocess when resolver delivers
    }

    // Ask the application to verify
    let verdict_rx = self.app.verify(c).await;     // oneshot::Receiver<bool>
    self.verdicts.insert(v, verdict_rx);           // remembered across awaits
}

fn on_verify_verdict(&mut self, v: View, ok: bool) {
    if v != self.view { return; }
    if ok {
        // Vote to notarize
        self.notarize(c, v).await;     // WAL write, sync, broadcast
    } else {
        // Vote to nullify
        self.nullify(v).await;
    }
}
```

**`on_notarize_vote(peer, c, v)`** — fired when *another* voter
casts a notarize vote.

```rust
fn on_notarize_vote(&mut self, peer: PK, c: Digest, v: View) {
    if v != self.view { return; }
    self.votes_notarize[v][c].insert((peer, sig));
    if self.votes_notarize[v][c].len() == QUORUM {
        // Quorum formed — assemble Notarization
        self.assemble_notarization(c, v).await;
    }
}
```

**`on_nullify_vote(peer, v)`** — fired on a nullify vote from a peer.

```rust
fn on_nullify_vote(&mut self, peer: PK, v: View) {
    if v != self.view { return; }
    self.votes_nullify[v].insert((peer, sig));
    if self.votes_nullify[v].len() == QUORUM {
        self.assemble_nullification(v).await;
        self.enter_view(v + 1).await;
    }
}
```

**`on_finalize_vote(peer, c, v)`** — fired when a peer votes to
finalize a notarized chunk.

```rust
fn on_finalize_vote(&mut self, peer: PK, c: Digest, v: View) {
    if v != self.view { return; }
    self.votes_finalize[v][c].insert((peer, sig));
    if self.votes_finalize[v][c].len() == QUORUM {
        self.assemble_finalization(c, v).await;
        // NB: enter_view(v+1) already happened implicitly when the
        // notarization was assembled (after app.certify returned true).
        // Do not advance again here — that would skip a view.
    }
}
```

**`on_leader_timeout()` and `on_activity_timeout()`** — timer
handlers.

```rust
fn on_leader_timeout(&mut self) {
    if self.view_entered_at + 2*DELTA <= self.clock.now() {
        self.nullify(self.view).await;
    }
}

fn on_activity_timeout(&mut self) {
    if self.view_entered_at + 3*DELTA <= self.clock.now() {
        self.nullify(self.view).await;
    }
}
```

These timeouts are independent — `L` fires at 2Δ (leader didn't
propose), `A` fires at 3Δ (2f+1 votes haven't arrived).

### The cryptographic notarization assembly

When `2f+1` notarize votes accumulate, the Voter (or the Batcher that
already counted them) builds the certificate:

```rust
async fn assemble_notarization(&mut self, c: Digest, v: View) {
    let votes = self.votes_notarize[v][c].clone();  // 2f+1 sigs
    let cert = Notarization { view: v, payload: c, signatures: votes };
    // Persist before broadcast (WAL pattern from chapter 06)
    self.journal.append(encode(&cert)).await?;
    self.journal.sync().await?;
    self.sender.send(Recipients::All, encode(&cert), false).await?;
    self.mark_notarized(c, v);
}
```

The interesting detail: the Batcher (not the Voter) does this
aggregation in some configurations — see chapter 10. The Voter can be
either an active assembler (Batcher-free mode) or a consumer of
Batcher-assembled certificates (`consensus/src/simplex/actors/voter/actor.rs`).

### Views and their data: a "view object"

In Commonware source, every view has an associated `round` struct
holding the votes for that view. The struct looks roughly like:

```rust
struct Round {
    view: View,
    leader: PK,
    timer_l: Instant,
    timer_a: Instant,
    votes_notarize: BTreeMap<Digest, BTreeSet<(PK, Sig)>>,
    votes_nullify: BTreeSet<(PK, Sig)>,
    votes_finalize: BTreeMap<Digest, BTreeSet<(PK, Sig)>>,
    notarization: Option<Notarization>,
    nullification: Option<Nullification>,
    finalization: Option<Finalization>,
}
```

When `enter_view(v+1)` is called, the existing round `v` is kept in
memory briefly (in case peers replay votes for `v`) and eventually
pruned (after activity timeout or finalized state is durable on disk).

### The view's lifecycle (the "small" state machine inside each view)

```
   enter_view(v)
        |
        v
   register_timer_l(t_l = now + 2 DELTA)
   register_timer_a(t_a = now + 3 DELTA)
        |
        v
   +---------------------+
   |   wait for events   | <----+
   +----+----+----+-------+      |
        |    |    |              |
        v    v    v              |
  proposal  nullify  timer  ...  |
        |    |    |              |
        v    v    v              |
  respond  enter(v+1)  respond   |
        |                     ---+
        v
   wait for quorum
        |
        v
   assemble cert, broadcast
        |
        v
   enter(v+1)
```

A view's lifetime is bounded by **either** a successful notarization
**or** a successful nullification. The first one wins; the second is
ignored. So if 2f+1 nullify votes *and* 2f+1 notarize votes both
arrive, the protocol takes whichever forms first.

### Why the state machine matters for Byzantine reasoning

Each transition is a "checkpoint" where a Byzantine peer can be
detected. The state machine treats *every* event as a vote that needs
a peer to authenticate. A peer that double-votes (sends notarize for
two different `c` in the same `v`) is detected the moment they send
the second — schema validation rejects it, batch-verify fails on it,
the `Oracle.control().block()` is called.

Reading the source as a state machine (rather than as a sea of `if`s)
makes this clear. Look for `consensus/src/simplex/actors/voter/actor.rs`
and `consensus/src/simplex/actors/batcher/round.rs` for the
implementations.

---

## Deepening 2 — PBFT in full: pre-prepare, prepare, commit

The earlier "PBFT phases and where Simplex departs" section is a
useful comparison; this section reconstructs PBFT end-to-end with
Castro-Liskov's actual algorithm, the famous view-change protocol,
and the worked example.

### The setting

`n = 3f + 1`. The leader is `v mod n`. The system is partially
synchronous (DLS-style, 1988). Three core properties:

1. **Safety**: no two honest nodes commit different values for the
   same sequence number.
2. **Liveness**: every client's request is eventually committed.
3. **Byzantine tolerance**: up to `f` peers can be arbitrarily faulty.

### The three phases (extended)

```
Pre-prepare: Leader -> all  : pre-prepare(view v, sequence n, digest d)
Prepare:     Replica -> all : prepare(view v, sequence n, digest d, i)
Commit:      Replica -> all : commit(view v, sequence n, digest d, i)
```

Three round trips. Why three?

- **Pre-prepare**: leader announces the proposal. Replicas learn `d`
  is the leader's claim for slot `n` of view `v`.
- **Prepare**: replicas exchange what they saw in pre-prepare. Once
  2f+1 prepare votes on `(v, n, d)` accumulate, the replica is
  **prepared** — i.e., it has proof that a quorum agrees that the
  leader proposed `d` for slot `n` of view `v`.
- **Commit**: replicas announce readiness to commit. Once 2f+1 commit
  votes accumulate, the replica **commits** `d` at slot `n` of view
  `v`.

The **prepare** and **commit** votes are themselves signatures over
`(view, sequence, digest)`. Both phases are quorum phases. The
three-phase structure (rather than two) is what gives PBFT its
**view-change** property: even if the leader equivocates within view
`v`, the prepare quorum guarantees all honest replicas agree, and the
commit quorum is irrevocable.

### A worked example: n=4, f=1

Four replicas: `R1` (leader), `R2`, `R3`, `R4`. One Byzantine: `R4`.
View `v = 7`, sequence `n = 100`. Leader proposes `d = H("hello")`.

**Round 1: pre-prepare.**

```
R1 -> R2: pre-prepare(7, 100, H("hello"))
R1 -> R3: pre-prepare(7, 100, H("hello"))
R1 -> R4: pre-prepare(7, 100, H("hello"))
R1 -> (self): pre-prepare seen
```

`R4` is Byzantine. It might delay, drop, or send a *different* digest.
Assume `R4` is silent for now.

**Round 2: prepare.**

```
R1 -> all: prepare(7, 100, H("hello"), R1)
R2 -> all: prepare(7, 100, H("hello"), R2)
R3 -> all: prepare(7, 100, H("hello"), R3)
```

(R4 might or might not send — it's Byzantine.)

Each replica counts prepare votes on `(7, 100, H("hello"))`. After
2 = 2f+1 prepares, the replica is **prepared**. Specifically:
R1 sees {self, R2, R3} = 3 prepares. R2 sees {R1, R2, self} = 3. R3
sees {R1, R2, R3} (self) = 3. All three honest replicas are prepared.

The Byzantine R4 might be stuck with fewer prepares. But `f+1 = 2`
honest replicas is the minimum threshold, and we have 3.

**Round 3: commit.**

```
R1 -> all: commit(7, 100, H("hello"), R1)
R2 -> all: commit(7, 100, H("hello"), R2)
R3 -> all: commit(7, 100, H("hello"), R3)
```

Each replica counts commit votes on `(7, 100, H("hello"))`. After 2
commits, the replica **commits** the value. After 2 = 2f+1, three
honest replicas commit.

**At-least-once delivery.** The application sees the commit on each
honest replica. Idempotency must be handled by the application
(chapter 12 covers this for Marshal).

### View changes — PBFT's most expensive part

If the leader is faulty (silent or equivocating), replicas trigger a
**view change** to rotate to a new leader. View-change in PBFT is
famous for being O(n^3) in message complexity. Why?

A view change has **three sub-steps**:

1. **`new-view`**: replicas stop accepting the current leader's
   proposals. They send `new-view(v+1)` to the new leader with a
   `prepare-certificate` (the strongest prepare quorum they have).
2. **`prepare` again**: the new leader collects `new-view`s,
   re-proposes the **highest prepared digest** (or `NONE` if no
   consensus), and replicas run a fresh `prepare` round.
3. **`commit`**: replicas exchange commit votes for the new leader's
   proposal.

Total per view change: `O(n)` for new-view, `O(n^2)` for prepare
(each replica-to-replica), `O(n^2)` for commit. So `O(n^3)` if you sum
all message-byte complexity. The O(n^2) per phase * 3 phases * n
replicas * signature size is what made PBFT impractical beyond a
few dozen replicas.

### Where Simplex departs — the point of it all

PBFT has **two distinct safety arguments** that have to interlock:

1. **Stable view**: within view `v`, all honest replicas agree on what
   they prepared.
2. **View-change safety**: when rotating from `v` to `v+1`, the new
   leader must propose a value consistent with the strongest
   prepare-cert replicas have, or risk forcing a fork.

Simplex unifies these. A Simplex `nullify(v)` is *both* an end-of-view
signal and a view-change trigger in one vote. The `notarize(v+1)` of
the new view is its own proof — there's no "new-view" carry-over
needed because new views reference the parent block's certificate in
their proposal, which encodes the chain state cleanly.

The result: Simplex's view-change is **O(n)** (just the nullify
quorum), not O(n^3). Same safety, roughly n^2x lower bandwidth.

### Why PBFT's O(n^2) per phase was actually used

Two reasons PBFT in production uses O(n^2) per phase:

1. **All-to-all is robust.** With every replica sending every other
   replica, any message-loss pattern still produces a quorum.
2. **Threshold is locally computable.** Each replica can count
   "how many `prepare(v, n, d)` sigs do I have?" without coordinating
   with anyone else.

Coded BFT (using erasure codes) can compress the all-to-all phase
into "broadcast + ack" (O(n) messages per view), but at the cost of
more complex recovery. Commonware's Simplex uses BLS threshold
signatures to collapse the all-to-all: one aggregate signature per
quorum check (chapter 04). Same end result, roughly n times less
bandwidth.

---

## Deepening 3 — Tendermint in full

Buchman's 2016 master's thesis ("The Tendermint Specification") and
the follow-on Buchman-Kwon-Milosevic (2018, "The latest gossip on
BFT consensus") describe Tendermint as a PBFT derivative with two
key simplifications: (1) weight-based leader rotation (round-robin
modulo validator count, no VRF randomness), and (2) locked-value
preservation across rounds.

### Round structure with locking

Tendermint's consensus runs in **rounds**. Each round has a single
leader (`round_number mod validator_set_size`). The round proceeds:

```
Round R:         Leader = R mod n
  1. Propose:    leader -> all  : proposal(round R, height H, value V)
  2. Prevote:    i -> all       : prevote(round R, height H, digest V_or_nil)
  3. Precommit:  i -> all       : precommit(round R, height H, digest V_or_nil)
  4. Round increment if neither quorum reaches 2f+1
```

The **polka** is "2f+1 prevotes on the same (round, value)." When
a replica observes a polka, it enters the precommit phase. When
2f+1 precommits on the same value, the value is **committed**.

### Locked value

A replica `i` that has observed a polka for `(R, V)` is said to be
**locked on V in round R**. If round R times out and the round changes
to R+1, the replica's new proposal (or vote) must reference V — it
cannot vote for a competing value at the new height without first
unlocking via a `Polka` of `nil` or a new proposal of V.

The locking rule prevents the system from forking across rounds: if
honest replica A locked on V at round R, and the network forces a
round change, then at round R+1 honest A can only vote V (or
contribute a `nil` to skip the round entirely). This is what gives
Tendermint **single-value commit** under asynchrony.

### The famous "two rounds of locks" failure mode

What if:

- Replica A locks on V at round R1.
- Replica B doesn't see the polka, but sees a precommit quorum for
  a different value V' at round R2.
- The chain has two locked candidates.

Tendermint's resolution: at round R3 (or higher), the locked replica A
*and* the unlocked replica B converge because **polka on V propagates
forward**. If the network eventually delivers the round R1 polka to
replica B, replica B unlocks (per the locking rule it's now bound to
V). If the network never delivers it, replica B continues voting V'
unilaterally. But replica A's votes for V at R2 (when it sees a
proposal for V') outvote B's votes for V' — so V wins.

This is similar to Simplex's **Forced Inclusion**: a notarization in
view v is permanent unless nullified. Tendermint's "locked value
propagates forward" is the same safety invariant with different
machinery.

### Why Tendermint's locking is *implicit*

A subtle but important detail: in Tendermint, locks are kept in
**memory** (the replica's `lockedValue` and `lockedRound` fields).
There is no "lock certificate" broadcast — the lock is private state
of each replica, derived from the polkas it has seen.

Simplex's analogous state is `notarizations_by_view` — the
Notarization certificate itself is the broadcast proof. Tendermint's
memory-bound locks are cheaper to *broadcast* but more expensive to
*verify* on view-change (the new leader must collect locked-value
proofs from replicas).

### Round-change in Tendermint

If a round times out (no proposal, no polka), replicas start a new
round via the same pre-vote / pre-commit flow with `nil`. After 2f+1
precommits on `nil`, the round changes.

This is **O(n) messages per round change** — much cheaper than
PBFT's O(n^3) view-change. Tendermint's simpler round-change is what
made it practical for cosmos-grade deployments (roughly 150 validators).

### Tendermint vs Simplex — the quick diff

| Property | Tendermint | Simplex |
|---|---|---|
| Leader rotation | `R mod n` | VRF over `(view, parent)` |
| Lock implementation | Per-replica memory | Per-view certificate (broadcast) |
| Round-change | O(n) | O(n) |
| Finality message | precommit poll | separate finalize vote |
| Fork rule | locked value persists | notarization can't be skipped |
| Optimistic case | 2 hops (propose + prevote) | 2 hops (propose + notarize) |
| Finality confirmation | precommit quorum | finalize quorum |

Tendermint uses an **accountable** safety rule: locked replicas are
detectable as Byzantine if they violate the rule. Simplex uses a
**stateless** rule: any replica can verify a notarization cert and
deduce the canonical chain. The trade-off is more broadcast
bandwidth (notarizations are bigger than individual votes) for
simpler reasoning.

---

## Deepening 4 — HotStuff in full

Yin, Malkhi, Reiter, Gueta, Abraham ("HotStuff: BFT Consensus with
Linearity and Responsiveness", 2019) introduced the modern
"linear-time" BFT that has become the dominant consensus algorithm in
production systems. AptosBFT, Jolteon, and various Ethereum L2
solutions are all HotStuff derivatives.

### The pipelining insight

HotStuff's key idea is **pipelined BFT**: rather than running the
3-phase protocol independently per decision, you pipeline decisions so
each new decision's "phase 1" is the prior decision's "phase 3."

The chain becomes:

```
Block 1:  <-prepare(round 1)->  <-precommit(round 1)->  <-commit(round 1)->  <-decide(round 1)->
                     --- p i p e l i n e ---
Block 2:                   <-prepare(round 2)->  <-precommit(round 2)->  <-commit(round 2)->
                           --- p i p e l i n e ---
Block 3:                                     <-prepare(round 3)->  <-precommit(round 3)->
```

Each block waits for its **prepare**, **precommit**, **commit**, and
**decide** phases, but those phases are shared with later blocks. A
**3-chain** (3 consecutive certified blocks) is the commit rule. A
**4-chain** is the prepare rule (which justifies the 3-chain commitment).

### Phases re-derived

Even though HotStuff pipelines, the **logic** is the same 3-phase
structure as PBFT, with one innovation: a *quorum certificate* (QC)
is a single BLS aggregate signature, so each phase is `O(n)` messages
not `O(n^2)`.

```
Prepare:     leader -> all : prepare(round R, height H, QC', digest D)
Precommit:   i -> all      : precommit if 2f+1 prepares on same D
Commit:      i -> all      : commit if 2f+1 precommits on same D
Decide:      i commits D locally if 2f+1 commits on same D
```

Each phase has a quorum certificate (BLS aggregate). The leader's
proposal carries the **highest QC** it has seen; replicas accept
the proposal as the basis for their prepare vote.

### The 3-chain commit rule

A replica commits a block B if there exist B's QC plus two more
consecutive QCs forming a **3-chain**:

```
   B (round R)         <-- QC at R
   B' (round R+1)      <-- QC at R+1, formed on B's QC
   B'' (round R+2)     <-- QC at R+2, formed on B''s QC
```

Three consecutive QCs in the chain are *proof* that the chain's
"view" has stabilized. The third QC votes for the second, the
second votes for the first, and the first is the committed block.

The proof intuition: a Byzantine leader cannot produce a 3-chain
unless an honest supermajority has signed the chain — because each
QC requires 2f+1 votes, and the only way to have 3 QCs is to have
the 2f+1 honest supermajority voting across 3 rounds.

### Responsiveness

HotStuff is **responsive**: in good network conditions, the protocol
proceeds at the speed of the actual network rather than waiting for
a worst-case DELTA. Specifically:

- The leader waits for prepare votes to form a QC; the QC forms as
  fast as 2f+1 votes arrive.
- The next phase (precommit) starts immediately after the QC, not
  after a fixed timer.

This is what gives HotStuff sub-second finality in production
(compared to PBFT's "wait 2DELTA before starting prepare"). Simplex
inherits this property (each phase is triggered by quorum arrival,
not by timeout).

### Linear communication complexity

The big line item:

| Phase | Messages | Bytes (n=100, BLS agg) |
|---|---|---|
| Prepare | O(n) | ~100 x leader-proposal |
| Precommit | O(n) | ~100 x vote |
| Commit | O(n) | ~100 x vote |
| Decide | local | 0 |
| **Round total** | **O(n)** | **~3 x n x vote_bytes** |

Compare to PBFT's `O(n^2)` per phase = `O(n^2)` per decision =
~3 x n^2 x vote_bytes. At n=100: 30,000 message equivalents for
PBFT, 300 for HotStuff. **Two orders of magnitude difference** at
production scale.

### View change in HotStuff

HotStuff's view change is a **single prepare phase**: the new leader
proposes a block that references the highest QC the network has
seen. Replicas vote to prepare if the proposal extends the chain
they're working on; otherwise they vote `nil`. After 2f+1 precommits,
the round is locked at the new QC.

Cost: `O(n)` for the new prepare round. The new leader's proposal
carries a "high QC" — the highest QC it has seen — and replicas
vote to switch. Worst case: ~3 view changes (linear backoff) before
the new leader proposes something safe.

This is why HotStuff's view-change is also linear, and why HotStuff's
total bandwidth at worst is `O(n)` per decision + `O(n)` per view
change = `O(n)` per effective round.

### Simplex vs HotStuff

The two are very close structurally. The differences:

1. **Election**: Simplex uses VRF randomness; HotStuff uses
   round-robin (or weighted round-robin in PoS).
2. **Justification**: Simplex's notarization certificate is the
   prepare equivalent; Simplex's finalize is the precommit
   equivalent. Both are BLS aggregates.
3. **Finality**: Simplex has a 3rd phase (finalize), HotStuff uses
   a 3-chain pipelining trick.
4. **Optimization**: Simplex's `nullify(v)` replaces HotStuff's
   vote-`nil` for view-changes. Simpler reasoning, no chaining.

Practically: HotStuff and Simplex produce equivalent properties
(linear messages, partial-synchrony safety, post-GST liveness,
single-value commit) with similar worst-case complexity. Simplex's
cleaner message vocabulary makes it easier to reason about the
state-machine.

---

## Deepening 5 — The full BFT comparison table

Below is the canonical 15-row comparison across PBFT, Tendermint,
HotStuff, and Simplex. Each row is a property the system has or
doesn't, with notes on why.

| # | Property | PBFT | Tendermint | HotStuff | Simplex |
|---|---|---|---|---|---|
| 1 | Phases per decision | 3 | 2+polling | 3 (pipelined) | 2 + finalize |
| 2 | Messages per decision | O(n^2) | O(n^2) | O(n) | O(n) |
| 3 | Hops per decision | 3 | 3 | 1-3 | 2-3 |
| 4 | View-change messages | O(n^3) | O(n) | O(n) | O(n) |
| 5 | View-change round trips | 3 | 1 | 1 | 1 |
| 6 | View-change cert | new-view QC | polka-of-nil | new high-QC | nullification cert |
| 7 | Leader rotation | mod n | mod n | mod n | VRF (view, parent) |
| 8 | Fork rule | single commit | locked value | 3-chain | single notarization |
| 9 | Signature scheme | multisig or BLS | multisig or BLS | BLS threshold | pluggable |
| 10 | Verification cost | O(n) per cert | O(n) per cert | O(1) per cert | O(1) per cert (BLS) |
| 11 | Linearizability? | yes | yes | yes | yes |
| 12 | Optimistic latency | 2-3 DELTA | 2-3 DELTA | 1 DELTA | 2 DELTA |
| 13 | Worst-case latency | O(n) views | O(n) views | O(n) views | O(n) views |
| 14 | Beacon chain length | 0 | 1 (lock cert) | 3 | 0 |
| 15 | Implementation complexity | medium | medium | high (chained) | low |

**Notes on each row:**

1. Phases: Simplex's "2 + finalize" means the notarization phase
   (2 hops for propose+notarize-quorum) plus one finalization phase
   for the certify verdict. PBFT has three all-to-all phases.
   Tendermint's "polling" is its second prevote/precommit cycle.
   HotStuff's "3 (pipelined)" is a pipeline, not 3 round-trips.

4. View-change messages: PBFT's O(n^3) is the famous problem (new-view
   + prepare + commit). Tendermint's O(n) is via polka-of-nil
   round-change. HotStuff's O(n) is via new-high-QC. Simplex's O(n)
   is the nullify vote.

7. Leader rotation: Simplex is the only one with VRF-based
   election, which provides *provable* randomness (the leader can't
   be predicted far in advance).

10. Verification cost: BLS threshold makes the cert check O(1) via
    one pairing. Multisig is O(n) — you verify each signature.
    Simplex's scheme is pluggable; the table assumes BLS threshold.

12. Optimistic latency: 1 DELTA for HotStuff because pipelining cuts
    the steady-state decision cost. PBFT's 2-3 is its all-to-all
    phase cost. Simplex's "2 DELTA" is two round trips (propose +
    notarize quorum).

15. Implementation complexity: Simplex's clean separation of
    notarize/finalize as distinct votes is simpler to reason about
    than HotStuff's chaining. Tendermint's locking is implicit (in
    memory); Simplex's is explicit (in certificates). PBFT's
    interlocking prepare/commit protocols are widely considered
    the most error-prone.

### Picking the right protocol for your deployment

| Need | Best choice |
|---|---|
| <= 30 validators, predictable membership | PBFT or Tendermint (mature) |
| 30-200 validators, partial-synchrony | Tendermint or Simplex |
| 100-1000 validators, need linear comms | HotStuff, Simplex, or Narwhal/Bullshark |
| Pre-shared sequencer, fast finality | Ordered Broadcast (chapter 14) |
| Validator rotation every epoch | Simplex (epoch-aware signatures) |

Simplex is the Commonware-native choice because of (a) BLS threshold
for O(1) cert verification, (b) pluggable scheme for migration
flexibility, (c) clean state-machine that's testable, and (d) the
forced-inclusion safety invariant that maps cleanly onto Ethereum-style
ledger semantics.

---

## Deepening 6 — BLS threshold scheme with VRF in depth

The earlier section introduced BLS threshold and VRF. This section
goes deeper into the math, the security model, and the implementation
choices.

### The BLS signature scheme, foundational

**Setup.** Three groups from BLS12-381:

- `G1`: an additive group of points on the BLS12-381 G1 curve.
- `G2`: an additive group of points on the G2 curve.
- `GT`: a multiplicative group; specifically the *target group*.

A bilinear map (pairing) `e: G1 x G2 -> GT` with the property:

```
e([a]P, [b]Q) = e(P, Q)^(a*b) = e([b]P, [a]Q)
```

where `[a]P` is scalar-multiplication of point `P` by `a`. The pairing
is computed via "Miller's loop" and a final exponentiation.

**Key generation.** `sk` is a random scalar. `pk = [sk] g2`, where
`g2` is the G2 generator.

**Sign.** Given message `m`:

1. Hash `m` into a point on G1: `H(m) in G1`. (Hash-to-curve, RFC 9380.)
2. Signature: `sigma = [sk] H(m) in G1`.

**Verify.** Compute `e(sigma, g2)` and `e(H(m), pk)`. The check is:

```
e(sigma, g2) = e(H(m), pk)
```

The lhs is `e([sk] H(m), g2) = e(H(m), g2)^(sk)`. The rhs is
`e(H(m), [sk] g2) = e(H(m), g2)^(sk)`. Both sides agree, so an
honest verifier accepts. A forger without `sk` cannot compute a
matching `(H(m), sigma)` pair without solving discrete log — believed
hard.

### Threshold signatures

A `(t, n)`-threshold scheme distributes the signing capability.
Commonware uses Pedersen DKG (or similar) so that:

- Each of `n` participants gets a secret share `sk_i`.
- A subset `S subset {1..n}`, `|S| >= t`, can run a distributed signing
  protocol to produce `sigma = [group_sk] H(m)` without ever assembling
  the group secret.

**Lagrange interpolation** is the workhorse. Given `t` shares
`(x_i, y_i)`, the constant term `a_0` of the degree-`t-1` polynomial
that passes through them is:

```
a_0 = sum_{i in S} y_i * lambda_i(0), where lambda_i(0) = prod_{j in S, j != i} x_j / (x_j - x_i)
```

Each `lambda_i(0)` is a public coefficient (depends only on the
participant indices). The protocol:

1. Each participant `i in S` produces a partial signature
   `sigma_i = [sk_i] H(m)`.
2. The combiner (any single party, possibly the leader) computes
   `sigma = sum lambda_i(0) * sigma_i`. (Scalar multiplication of each
   partial sigma by its Lagrange coefficient, then sum.)
3. `sigma` is a valid BLS signature under `pk_group = sum [sk_i] g2`.

The crucial property: **the resulting `sigma` looks identical regardless
of which `S` set was chosen.** An observer cannot distinguish "S1
signs the message" from "S2 signs the message" — the Lagrange
coefficients fold the choice away.

### Commonware's BLS threshold module

`consensus/src/simplex/scheme/bls12381_threshold/` has:

- `bls12381_threshold/mod.rs` — the `Scheme` implementation.
- `bls12381_threshold/vrf.rs` — the VRF wiring.
- `bls12381_threshold/certificate.rs` — the certificate type.

The `Scheme::assemble_certificate` implementation:

```rust
fn assemble_certificate<'a, I>(sigs: I) -> Self::Certificate
where I: Iterator<Item = &'a Self::Signature>
{
    // Aggregate as curve points (additive group)
    let mut agg = G1::identity();
    for sig in sigs {
        agg += sig;
    }
    agg.into()
}
```

That's it. BLS aggregation is just point addition.

### The security proof (sketched)

The BLS scheme's security rests on the *co-CDH hardness assumption*:
given `g1, [a]g1, g2, [b]g2`, computing `[ab]g1` is hard. (Some
variants use *co-Diffie-Hellman* instead — equivalent up to
non-degeneracy of the pairing.)

For BLS threshold specifically:

- The Pedersen DKG produces shares that reconstruct `group_sk` only
  with `t` shares. Fewer than `t` shares yield nothing.
- An adversary controlling up to `f = t - 1` participants cannot
  forge a signature without solving co-CDH.
- The aggregate is verified against `pk_group = sum [sk_i] g2`, which
  is also the BLS pubkey of the group. So a single pairing check
  suffices.

Threshold BLS is *non-attributable*: you can't tell which `t` of the
`n` participants signed. This is by design — Lagrange interpolation
destroys that information. For attribution, use `bls12381_multisig`
(which is the same scheme with no threshold and a bitfield of which
indices contributed).

### The VRF, in depth

A **verifiable random function (VRF)** is a public-key analog of a
hash function: it produces an output that's a deterministic function
of `(sk, input)`, but unpredictable without `sk`. Verification is by
the corresponding `pk`:

**VRF produce.** Given `sk` and `input`:

1. Hash `input` to a G2 point: `H_vrf(input) in G2`.
2. Compute `output_point = [sk] H_vrf(input) in G2`.
3. Compute `proof = [sk] H_proof(input) in G2` (with a different
   hash function for the proof).
4. Return `(output_point, proof)`.
5. Convert `output_point` to a 32-byte `output` via some encoding
   (typically `output_point.to_bytes()` truncated).

**VRF verify.** Given `pk`, `input`, `output`, `proof`:

1. Recompute `H_vrf(input)` and `H_proof(input)`.
2. Check `e(proof, g2) == e(H_proof(input), pk)`.
3. Check `output_point == output.to_curve_point()`.
4. If both pass, accept.

The pairing on step 2 enforces that the proof was made by the same
key that produced the output (because `e([sk] H_proof(input), g2) ==
e(H_proof(input), [sk] g2)`).

### Why VRF + Simplex for leader election

In Simplex's election (`consensus/src/simplex/scheme/bls12381_threshold/vrf.rs`):

```rust
fn leader_for(view: View, parent: Digest) -> Option<(PK, VrfOutput)> {
    let input = H(view_le_bytes(view) || parent.as_ref());
    let (output, proof) = vrf.prove(my_sk, input);
    if output.as_bytes() < THRESHOLD {
        Some((my_pk, output))
    } else {
        None
    }
}
```

Each validator computes their VRF; those whose output is below the
threshold "qualify" as leader candidates. The threshold is set so
that on average, one validator per view qualifies (e.g., `1/n` of the
range).

The `Elector::determine_leader(view, parent)` (in
`consensus/src/simplex/elector.rs`) returns a `Leader` — the
candidate with the *smallest* VRF output, or a deterministic selector
(tied on `pk`).

**Why this beats `view mod n`:**

- **Randomness.** Honest validators can't predict the leader far in
  advance. An attacker who knows the protocol can't pre-position a
  malicious leader except by controlling `sk`s.
- **Sampled distribution.** With `THRESHOLD = range/n`, most views
  have one candidate, some have zero or two. Zero is fine (everyone
  nullifies). Two is rare; whoever gets the smaller output "wins"
  arbitrarily.
- **Parent-digest binding.** Two views with the same number but
  different parents have *different* elections. A Byzantine leader
  in view 17 with parent X can't re-run the election with parent X'
  if X' would have produced a different leader.

The last point is critical: it prevents a Byzantine leader from
"forking" by sitting on a parent until they get the leader slot.

---

## Deepening 7 — The View abstraction in depth

The earlier "View abstraction in Simplex" section is a useful
introduction. This section grounds the abstraction in formal
distributed-systems language and explains why Simplex's specific
construction maps cleanly onto Ethereum-style ledger semantics.

### What is a "view" formally?

A view `v` is a **monotonically increasing time index** associated with
a specific (consensus instance, attempt). In Simplex, each view:

1. Names a single leader (deterministically computed from the view).
2. Names a single digest that all honest replicas either notarize or
   nullify.
3. Has a bounded lifetime (less than or equal to 3 DELTA of network time) under synchrony.
4. Yields a single piece of consensus output if a notarization forms:
   `Notarization(c, v)`.

The view number is independent of the *block height* in the ledger.
A ledger block at height `h` has a parent at height `h-1`. The view
at which `h` was proposed is some `v_h >= v_{h-1}`. Many heights can
correspond to one view if the view had multiple proposals
(rare; happens only if `Application::propose` returns multiple
digests).

### The "view" vs "round" vs "slot" vocabulary

Different BFT papers use different names for the same concept. The
table:

| Term | Used in | Meaning |
|---|---|---|
| View | PBFT, Simplex, HotStuff | A leader-rocks attempt to commit |
| Round | Tendermint, Raft | Same as view (leader chosen per round) |
| Ballot | Paxos | (ballot_number, proposer_id) pair |
| Slot | Blockchain | A position in the ledger (monotonic per chain) |
| Height | Ethereum | Same as slot |
| Sequence number | PBFT | Same as slot |

In Commonware, **view** and **round** are used interchangeably in
code (`consensus/src/simplex/types.rs` has both `View` and `Round` as
type aliases). **Slot** refers to the ledger position, **view** is the
attempt at committing the slot's next block.

### Gap semantics

Simplex allows views to be **skipped** under asynchrony:

```
view:    1   2   3   4   5   6   7   8
status:  N   N   N   -   -   N   N   N
         ^   ^   ^              ^   ^   ^
         |   |   |              |   |   |
        notarized in v1-3,       (network stalled,
        nullification in v4-5;   now recovering)
        views 6-8 each succeeded
```

A validator that was offline during views 4-5 sees:

- When it joins, it requests the missing nullification certificates
  from a peer (via Resolver).
- It then knows "v1=block A, v2=block B, v3=block C, v4=null,
  v5=null, v6=block D."
- It can verify that all its own votes are consistent with these.

The **gap-free property** is preserved: every view is *accounted for*
(either notarization or nullification), but the views are not
necessarily contiguous in time.

### Forced Inclusion, formally

The Simplex specification's forced-inclusion lemma (cited from
`consensus/src/simplex/mod.rs:121-138`):

> If a notarization for `(c, v)` exists in the network and the
> prevailing network condition is synchronous, then for all `v' > v`,
> every honest replica's proposal at view `v'` either extends `c` or
> extends one of `c`'s descendants.

The proof: by quorum intersection, at least one honest replica
emitted the notarization for `(c, v)`. That honest replica emits a
nullification or notarization for *every* view `v' > v` (because
it's never silent for two consecutive views). The honest replica's
*certification* of any of those views builds on a chain that
includes `c` (because to build a chain you reference the parent,
and `c`'s finalization is the canonical tip after view `v`).

Thus no future proposal can skip `c`, regardless of which view that
proposal appears in. By induction, `c` is in every future chain.

This is what makes Simplex "Ethereum-equivalent" for ledger
semantics: a notarized block is *unforkable*. The only way a
notarization can be excluded from history is if a nullification is
formed for it, and that's impossible in synchronous conditions.

### Why "view" rather than "slot" in consensus messages

Simplex's messages carry `View`, not `Slot`. Why?

- A view is a *consensus attempt* (a leader-rocks).
- A slot is a *ledger position*.

These are different things: a slot might be *attempted* in multiple
views if the earlier attempts failed (nullified). Saying "block at
slot 5 was proposed in view 17" is the right fact; saying "block
at slot 5 was proposed in view 5" confuses consensus with ledger.

Ethereum's beacon-chain model has the same distinction: each
*slot* has an associated *epoch* and *committee*. The "slot" is the
ledger position; the "epoch" is the consensus-attempt grouping.

For Simplex, the model is: a view contains a single proposal, and
once notarized, that proposal's *block* is associated with the view
permanently. The slot for that block is the one that contains the
view's parent (the block chain's height increments by 1 per
notarization).

### Cross-references

The View abstraction shows up in:

- Chapter 12: Marshal uses views to match notarization certs to blocks.
- Chapter 13: Aggregation uses views for tip gossiping (the
  `(view, height)` tuple).
- Chapter 14: Ordered Broadcast has a different sense of "view" —
  the sequencer rotation period (epoch).

---

## Deepening 8 — Linearizability vs serializability

The earlier section on linearizability vs serializability covers the
high-level distinction. This section gives the formal definitions and
shows what Simplex provides.

### Serializability (the weaker property)

A history `H` of operations on an object is **serializable** if
there exists a serial history `H'` such that:

1. `H` and `H'` contain the same operations.
2. For any two operations `op1` and `op2` on the same object, if
   `op1` precedes `op2` in `H`, then `op1` precedes `op2` in `H'`.
3. The two histories produce the same results on the object.

The serial order need not match the *real-time* order. `op1` could
have happened at wall-clock `t1`, and `op2` at `t2 > t1`, but in
`H'` they appear in reverse. As long as the operations don't
interleave on the same object, this is fine.

**Example.** Two bank transfers, no shared accounts:

```
H:  Transfer(alice -> bob, 100) invoked at t1
    Transfer(carol -> dave, 50)  invoked at t2

H': Transfer(alice -> bob, 100)
     Transfer(carol -> dave, 50)
```

Serializable. The order is `H'`(even if it matched real time, that's
incidental).

### Linearizability (the stronger property)

A history `H` is **linearizable** if there's a serial history `H'`
such that:

1. `H` and `H'` contain the same operations.
2. For any two operations `op1` and `op2` on the same object, if
   `op1` completes before `op2` is invoked (real-time order), then
   `op1` precedes `op2` in `H'`.
3. The two histories produce the same results.

The key addition: the linearization point of each operation (where
it "happens" in the serial order) must lie between its invocation
and response times.

**Example.** Reading same key concurrently:

```
H:  read(key) invoked at t1, returns 5     [client A]
    write(key, 7) invoked at t2, returns    [client B]
    read(key) invoked at t3, returns 7

H': write(key, 7) [happens at "linearization point" between t2 and t3]
     read(key) [returns 7]
     read(key) [returns 5]
```

The first read returns the old value, the second the new value. The
linearization order has `write` between the two reads, which is
consistent with real-time (write completed before t3).

### Simplex's linearizability

Simplex provides **linearizability** of finalized blocks, with the
linearization point being the moment the Finalization certificate
reaches `2f+1`. Specifically:

> A finalized block B at view V has linearization point `t_f` =
> the moment when 2f+1 Finalize(B, V) votes had been received from
> unique validators.

After `t_f`, every honest node's view of "the chain" includes B at
its position. A block finalized at time `t_f` and a state transition
applied at time `t_f + 1` respect real-time order: the state
transition observes the block.

### Why linearizability (not just serializability) for consensus

Three reasons:

1. **Cross-chain bridges.** A bridge from chain A to chain B needs
   to say "withdrawal at time T on chain B corresponds to deposit at
   time `T - DELTA` on chain A." Linearizability ensures the deposit at
   time `T - DELTA` is finalized before any state change referencing it
   on chain B.
2. **Composability.** If protocol A produces linearizable output and
   protocol B consumes it, B's view of A is also linearizable.
   Serializability wouldn't necessarily have this property — A's
   operations might not respect the real-time order B assumes.
3. **Real-world timestamps.** A user submits a transaction at time
   `t1`. They expect it to be ordered after other transactions at
   `t < t1`. Linearizability captures this.

### What Simplex does NOT linearize

**Notarization is not linearizable.** A notarized block may fail
certification later. The block isn't yet "committed" — its
commit-status is a 2nd-phase thing. So during the certify window,
the block is "speculatively final."

This is why Simplex has a *separate* finalize vote. Without the
finalize, the system is "almost linearizable but not quite" — a
Byzantine leader could trick some honest replicas into voting for an
un-certifiable block, and the network would grind on a view timeout.

The finalize phase is what kills that attack: if certify fails, the
view is nullified.

### The relationship to other consistency models

The consistency-model hierarchy (weaker -> stronger):

```
Eventual consistency (Dynamo, Cassandra single-region)
    -> Causal consistency (COPS, lazy replication)
    -> Read-your-writes (the client's own writes are visible)
    -> Sequential consistency (PRAM, total order across nodes)
    -> Serializability (some serial order)
    -> Linearizability (serial order respects real time)
    -> Strict serializability (linearizability + real-time write ordering)
```

Simplex provides the **second-strongest**: linearizability. Strict
serializability (the strongest) requires additional guarantees about
real-time ordering of *writes* that linearizability allows to be
vague. Commonware applications add strict serialization only when
required (e.g., a financial application that needs timestamp
ordering, not just linearization).

### Where the formal definitions live in source

`consensus/src/simplex/mod.rs:121-138` has the formal
`Notarization`, `Nullification`, `Finalization` definitions.
The `Safety` and `Liveness` lemmas in the appendix exercise the
above reasoning. The integration test `all_online` and the
`conflicter` mocks verify the properties empirically.

---

## Deepening 9 — Bracha's reliable broadcast in full

The earlier "Bracha's reliable broadcast" section gives an overview.
This section gives the full algorithm with message counts and the
key insight.

### Bracha's protocol (1984/1987 versions)

Three phases, each is a quorum phase:

```
SEND:   sender -> all  : SEND(m, sender)     # broadcast the message
ECHO:   i -> all       : ECHO(m, sender)     # sign "I saw m"
READY:  i -> all       : READY(m, sender)    # sign "I'm ready to deliver"
DELIVER: local        : -> m                 # 2f+1 READY received, deliver
```

Specifically:

```
Sender S:
  - broadcast SEND(m, S) to all
  - if S is honest: peers will get the message and proceed

Replica i:
  - upon receiving SEND(m, S) [if direct from S, not gossip]:
      send ECHO(m, S) to all
  - upon receiving 2f+1 ECHO(m, S):
      send READY(m, S) to all
  - upon receiving 2f+1 READY(m, S) [including f+1 of which from senders distinct from m's sender]:
      deliver m
  - upon receiving 2f+1 READY(m, S) [or equivalent]:
      if already-sent READY, ignore; if not, send
```

The "f+1 of which from senders distinct from m's sender" line is the
**fork prevention**: if a Byzantine sender broadcasts two different
values m1 and m2, the 2f+1 READY threshold for each requires 4f+2
signatures, exceeding the n = 3f+1 total. So `DELIVER` for m1 and
`DELIVER` for m2 can't both fire.

### Message complexity

Each step has a quorum:

- **ECHO**: O(n^2) total — each replica broadcasts once, so n x n
  messages.
- **READY**: O(n^2) total — same shape, n x n.
- **DELIVER**: local event.

So `O(n^2)` total per broadcast, over two round trips.

If the sender is honest, ECHO and READY can sometimes be combined:
the first round is SEND + ECHO, second round is "2f+1 ECHO received,
broadcast READY." Total: 2 round trips, O(n^2) messages.

### The "f+1 distinct senders" detail

Why is the f+1 distinct sender rule necessary? Consider:

- n = 4 (f = 1).
- Sender S = R4 (Byzantine).
- S broadcasts two different values m1 and m2 (one to R1, one to R3).

R2 sees m1 (sent to it by S), R1 sees m1 (same), R3 sees m2 (sent
to it). R4 sees m1.

To deliver m1, we need 2f+1 = 3 READY votes for m1. Possible: R1, R2,
R4 (including f+1 of which from senders distinct from S = R4 which is
also Byzantine; so distinct senders of READY for m1 are R1, R2, *not
yet* R4 itself if R4 includes its own READY).

Wait, the f+1 rule is that the 2f+1 READY includes at least f+1
senders that are *not* the original sender. This prevents the
Byzantine sender R4 from padding the count by signing its own m1
or m2's READY.

Without the f+1 rule: R4 signs READY for both m1 and m2. R1 sees
2 READY's for m1 (R1, R4) = 2 < 2f+1 = 3. R2 sees same. So R1, R2
don't deliver m1 yet. But R1 also sees READY(m2) from R3; that's 2.
Same for R2.

With the f+1 rule: if R1, R2 both see READY(m1), they need a third
sentinel from "not R4". Without f+1, R1 won't broadcast READY until
it has 2f+1 (which it can't get). Hmm, let me re-read.

Actually the original Bracha rule is somewhat subtle. The condition
is f+1 *distinct* senders for the READY votes *after accounting for
self*. So for honest R1 to deliver, R1 needs 2f+1 READY votes for
m1 with at least f+1 of them coming from peers other than R4 (the
sender of m). This prevents R4 itself from being one of the f+1.

If f = 1, then R1 needs 2 READY votes for m1 from senders other than
R4. That's 2 out of {R1, R2, R3} = 3 peers. So at least 2 of {R1,
R2, R3} must sign READY(m1). Similarly for m2.

If R4 broadcasts both m1 (to R1, R2) and m2 (to R3), then R1, R2
have m1 (each signs READY(m1) after seeing each other's ECHO), and
R3 has m2 (signs READY(m2)). To deliver m1, need R1, R2 + 1 = 3
votes for m1. But R3 hasn't seen m1, so it can't sign READY(m1).
R1, R2 have only 2 READY for m1, can't deliver.

To deliver m2, similarly can't deliver. So both deliveries are
blocked. The f+1 rule prevents partial delivery.

This is the **equivocation argument** that Bracha's protocol
provides: a Byzantine sender can confuse some replicas, but cannot
force a delivery split.

### Where Simplex uses (and doesn't use) Bracha

Simplex **does not** use Bracha for consensus messages (the votes
and certificates are propagated via standard gossip + batcher).

Simplex **does** use Bracha-like reliability for *data dissemination*
in the block layer (chapter 07 ZODA coding). Specifically:

- The leader encodes the block into shards and sends each peer a
  strong shard.
- Peers accumulate shards until they have `k` of them.
- The reconstruction is the "deliver."

This is similar to Bracha in spirit (DA implies availability), but
not exactly Bracha because there's no `ECHO/READY/DELIVER` cast — the
gossip layer handles reliable delivery via reference counting
(chapter 09).

### The Bracha-to-quorum-intersection simplification

A general insight: in any BFT protocol using Bracha, the second and
third phases can be combined into a single quorum phase if the system
already has quorum-intersection guarantees.

Simplex exploits this: the *notarization* quorum IS the "deliver"
phase. Once 2f+1 notarize votes exist for `(c, v)`, the value c is
"delivered" for view v — there's no need for a separate READY
phase.

The catch: this only works for views that *eventually form a
notarization*. For views that fail (network asynchrony, leader
equivocation), Simplex uses the `nullify(v)` protocol — which is
itself a simple quorum phase, not Bracha-style.

So Simplex "embeds" Bracha's mechanism for the happy path and
substitutes a simpler mechanism for the failure path. The cost is
that failures take a view-timeout (3DELTA); the benefit is two
fewer round trips for the happy path.

### The full ECHO/READY/DELIVER in code

```rust
async fn on_send(&mut self, m: Digest, sender: PK) {
    // First step: echo what we got from the sender
    self.echoes.entry(m).or_default().insert(self.pk.clone());
    self.broadcast(BrachaMsg::Echo(m, sender)).await;
}

async fn on_echo(&mut self, m: Digest, sender: PK) {
    self.echoes.entry(m).or_default().insert(sender);
    // f+1 echoes means: at least one honest validator saw the message
    if self.echoes[&m].len() == 2 * self.f + 1
       && !self.ready_sent.contains(&m)
    {
        self.ready_sent.insert(m);
        self.broadcast(BrachaMsg::Ready(m, self.original_sender(m))).await;
        self.check_deliver(m).await;
    }
}

async fn on_ready(&mut self, m: Digest, sender: PK) {
    self.readies.entry(m).or_default().insert(sender);
    if self.readies[&m].len() == 2 * self.f + 1
       && !self.ready_sent.contains(&m)
    {
        self.ready_sent.insert(m);
        self.broadcast(BrachaMsg::Ready(m, self.original_sender(m))).await;
        self.check_deliver(m).await;
    }
}

async fn check_deliver(&mut self, m: Digest) {
    // f+1 distinct senders + total 2f+1 ready
    let distinct = self.readies[&m].len();
    let non_sender = self.readies[&m].iter()
        .filter(|p| *p != &self.original_sender(m))
        .count();
    if distinct >= 2 * self.f + 1 && non_sender >= self.f + 1 && !self.delivered.contains(&m) {
        self.delivered.insert(m);
        self.deliver(m).await;
    }
}
```

This is the canonical Bracha implementation pattern. Commonware's
own ordered broadcast (chapter 14) uses a similar pattern but with
*linked* chunks instead of independent broadcasts.

---

## Deepening 10 — The recovery protocol in depth

The earlier "Recovery protocol" section explains the WAL pattern.
This section formalizes what state is recovered vs discarded, the
boot sequence, and the safety argument.

### What is persisted (durable state)

After every state-changing action, the Voter (`actors/voter/state.rs`)
writes the following to its journal:

| Type | When written | What's in the entry |
|---|---|---|
| `Notarize(vote)` | Voter casts a notarize vote | (view, digest, public_key, signature) |
| `Nullify(vote)` | Voter casts a nullify vote | (view, public_key, signature) |
| `Finalize(vote)` | Voter casts a finalize vote | (view, digest, public_key, signature) |
| `Notarization(cert)` | Voter assembles a notarization | (view, payload, signatures) |
| `Nullification(cert)` | Voter assembles a nullification | (view, signatures) |
| `Finalization(cert)` | Voter assembles a finalization | (view, payload, signatures) |

The **order is critical**: write *first*, then broadcast. This is the
"no contradictory state" property.

### What is NOT persisted

State that lives only in memory:

- In-flight `oneshot::Receiver<bool>` from `Automaton::verify` etc.
  These are dropped on crash. The application's `verify` must be
  idempotent — re-issuing the same request is safe.
- Peer mailbox contents — messages received but not yet processed.
  These get re-received from peers after restart.
- Network timers (`Clock::sleep_until`). The deterministic runtime
  resumes them via the WAL's timestamps.
- In-progress `BTreeMap` view votes — these are reconstructible from
  the WAL on restart, but the runtime may opt to start with empty
  maps and rely on peer replays for correctness.
- The Batcher and Resolver states — same as the Voter; reconstructible
  on restart.

The reason: WAL-then-broadcast ensures the *critical path* of the
protocol's state is recoverable. Non-critical in-progress state is
discarded but recomputable from network.

### The recovery sequence on restart

```
Boot a node:

1. Open the journal (chapter 06). The journal is in the storage partition
   named by the engine config.
2. Replay the journal. For each item decoded:
    - If Notarize(vote): add to recovered votes_by_view.
    - If Nullify(vote): add to recovered nullify votes_by_view.
    - If Finalize(vote): add to recovered finalize votes_by_view.
    - If Notarization(cert): add to recovered notarizations_by_height.
    - If Nullification(cert): add to recovered nullifications_by_view.
    - If Finalization(cert): add to recovered finalizations_by_height.
3. Rebuild the durable in-memory state:
    - For each height h: set finalized_at_h to the recovered finalization
      (if any).
    - For each height h: set notarizations_at_h to recovered set.
    - For each view v: set votes_by_view to recovered set.
4. Reconstruct the current view: the highest view number with a
   notarization or nullification, plus 1 (the next view to attempt).
5. Initialize round state for the recovered view.
6. Initialize transient state:
    - voted_in_current_view = false
    - finalized_recently = empty
    - awaiting_verdicts = empty (since the oneshot were dropped)
7. Begin voting on the recovered view's progress.

Optionally:
8. Ask peers for any newer certificates via Resolver. This bridges any
   time gap during downtime.
```

### What happens to in-flight Application calls

The most subtle part. Suppose the Voter is awaiting
`Automaton::verify(...)` for some `(c, v)` when the node crashes.

- The future is dropped. The application's `verify` may be partially
  complete or not started.
- On restart, the Voter's `awaiting_verdicts` is empty (the
  future's identity is forgotten).
- A vote for `(c, v)` may never have been sent (it was conditional on
  `verify` returning true).
- A peer's notarization for `(c, v)` may have arrived before our
  crash; on restart we'll see it from the network replay.
- Our final state on restart: we're at the recovered view with no
  pending verdicts, ready to vote fresh.

The application **must be idempotent** in `verify` so that re-issuing
the same call is safe. If the application has side effects in `verify`
(non-idempotent), the protocol can never be safely restarted.

### The rebroadcast guarantee

The critical safety argument: if a message was broadcast (left our
node), then the WAL had it recorded first. So on restart:

- WAL item exists -> re-decode and re-broadcast.
- WAL item doesn't exist -> message was never broadcast, no re-emit.

This eliminates the "the network thinks I voted for X, but I don't
have a record of it" failure mode. Even if the network drops our
broadcast (adversarial peer refused to relay), we re-emit on restart.

The cost: a small amount of duplicate traffic on restart. The peer's
de-duplication (via the signed message's unique identifier) filters
those out.

### Byzantine safety argument

Suppose we want to prove: a Byzantine Voter cannot vote two different
values for the same `(view, slot)` pair, even across crashes.

The argument:

1. On the first crash, the WAL has some items. Each item encodes a
   view, slot, and a single signed vote.
2. On restart, we replay items in order. The Voter's vote set is
   `BTreeMap<(View, Slot), BTreeMap<PK, Signature>>`. If we already
   have a vote from the local validator for `(v, s)`, we don't vote
   again.
3. The Voter's `vote_on(view, slot)` is "add to map, write to WAL,
   broadcast." If we add to the map, we write to WAL, then we
   broadcast. So the broadcast only happens if the WAL has it.
4. If we crash between map-add and WAL-write, the WAL is empty for
   this vote. On restart, we don't re-vote (no WAL item).
5. If we crash between WAL-write and broadcast, on restart we replay
   the WAL item but the vote is already in the map, so we don't
   re-broadcast. Wait, that's a bug — we WOULD re-broadcast.

Let me re-think. The protocol order is:

```
vote:
  1. Add to in-memory map
  2. Append to WAL
  3. Sync WAL
  4. Broadcast
```

If we crash between step 2 and 4, on restart the WAL has the vote.
The map is now empty. We replay the WAL item, add it back to the
map. Now we have to decide: broadcast or not?

The safe choice: **always re-broadcast on restart if the WAL says we
voted**. The peer's dedup logic catches duplicates. So:

```
on_replay_wal_item(item):
  self.vote_map.insert(item.view, item.slot, item);
  if not yet broadcast in this epoch:
    broadcast(item)
    mark_as_broadcast
```

This handles the partial-write case. The danger case (a Byzantine
Voter who re-broadcasts even when not needed to confuse peers) is
prevented because the `mark_as_broadcast` flag persists only in
memory — we'd need a separate "log of broadcasts" in the WAL to be
fully Byzantine-safe. Commonware's Voter uses a single bit per item:

```rust
struct JournalItem {
    item: Vote,
    broadcast: bool,    // did we send it yet?
}
```

`broadcast = false` after a partial-failure crash; the WAL replay
sets it to true after the broadcast completes.

### Recovery on crash during certify

If the Voter is awaiting `Automaton::certify(round, payload)` and
crashes:

- The certify future is dropped. The application's `certify` callback
  may be partial or not started.
- On restart, the WAL has the Notarization certificate (we wrote it
  before the certify call).
- The Voter resumes voting from the recovered state. It does NOT
  re-call certify.
- The recovered Notarization serves as the "evidence" of progress;
  the Voter continues to broadcast the Finalize vote (or Nullify if
  certify fails on a fresh restart that has the data).

Why this works: the **certify call is one-shot**. Its only purpose is
to gate the Finalize vote. Once a Notarization is in the WAL, we have
the proof of "2f+1 voted to notarize". The Finalize vote depends on
the application's certify verdict. On restart, we **trust the
recovered state** for the Notarization, and we **re-issue** the
certify call to check if Finalize should be cast.

---

## Deepening 11 — Performance analysis in depth

The earlier "Performance analysis" gives headline numbers. This
section walks through the cost model layer by layer, then derives
the **carnot bound** (theoretical maximum throughput) for a Simplex
deployment.

### The cost model, per view

In one view, with N = 100 honest replicas:

| Phase | Sender | Messages | Bytes (each) | Total bytes |
|---|---|---|---|---|
| Pre-attach timer | n participants | n heartbeats (optional) | 32 | ~3 KB |
| Leader proposes | 1 -> n-1 | n-1 notarize broadcasts | 8 KB | ~800 KB |
| Notarize votes | n-1 -> n | n(n-1) broadcasts | 128 | ~1.2 MB |
| Finalize votes | n-1 -> n | n(n-1) broadcasts | 128 | ~1.2 MB |
| Finalize cert propagation | 1 -> n-1 | n-1 broadcasts | 1 KB | ~100 KB |
| **Per view total** | | | | ~3.5 MB |

For a network round-trip DELTA = 50 ms (e.g., US-east to US-west):

- Pre-quorum wait: 1 DELTA = 50 ms
- Notarize quorum formation: 1 DELTA = 50 ms
- Total per view in steady state: 100 ms = 10 views/sec

Throughput per second: 10 views × ~8 KB per block proposal = 80 KB/s
of *consensus* bandwidth per validator. Note: the *block's* contents
are separate; those are broadcast via the Resolver.

### The bottleneck analysis

What limits throughput in practice?

**Round-trip latency.** DELTA ~ 50 ms means a per-view latency of
100 ms minimum (one round to propose, one to notarize, plus the
finalize hop). This is the **headline bottleneck** in geo-distributed
consensus.

**Signature verification.** BLS threshold: 1 pairing per cert = ~50us.
Ed25519 multisig: 100 sigs × 50us each = 5 ms. Multisig is 100x
slower on the verify path.

**WAL fsync.** NVMe drives: ~100us/fsync. HDD: ~10ms/fsync. Each
vote triggers a WAL fsync. NVMe can sustain 10,000 fsyncs/sec; HDD
can sustain 100. This is the **disk-side bottleneck**.

**CPU for hashing.** SHA-256: ~1 GB/s on a modern CPU. For block
proposals of 1 MB, hashing takes 1ms. For 100 MB blocks: 100 ms
(almost a full round trip).

**Network bandwidth.** A 1 Gbps uplink can push ~125 MB/s. For 10
views/sec with 1 MB block proposals, the validator egress is 10 MB/s,
fine. For 100 MB blocks, it's 1 GB/s, saturating the uplink.

### The carnot bound

The "carnot bound" (from the Commonware blog post on the topic) is the
theoretical maximum throughput of a Simplex deployment given its
hardware constraints. The bound:

```
Throughput_max = min(
    1 / (view_latency),
    disk_fs / (votes_per_view),
    network_egress / (block_size),
    cpu_cycles / (verify_per_view)
)
```

For N=100, NVMe disks, 1 Gbps uplink, modern CPU:

```
1 / view_latency  = 10 views/sec (DELTA=50ms)
disk_fs / votes   = 10,000 / 3 = 3,333 views/sec (way above)
network_egress    = 125 MB/s / block_size
cpu / verify      = 100,000 views/sec (way above)
```

The bottleneck is **view latency**, not the others. So a Simplex
deployment's actual throughput is the round-trip-limited number.

For a single-region (DELTA=2ms) deployment: 500 views/sec → 500 blocks/sec
× 1 MB blocks = 500 MB/s. The disk and CPU still aren't the bottleneck;
the network starts to be.

For a WAN deployment: 10 blocks/sec is realistic. That's the
production target for an Alto-style chain.

### The optimization matrix

Where Commonware spends optimization effort (high to low impact):

| Optimization | Where | Effect |
|---|---|---|
| BLS threshold signatures | All consensus messages | O(1) cert verification |
| Erasure coding | Large blocks | O(N/k) leader egress instead of O(N) |
| Coded broadcast | Mid-size blocks | Effective bandwidth x 3-5 |
| Batch verification | Vote collection | One pairing for all votes |
| WAL append-only | All state | Crash recovery + fast appends |
| Deterministic runtime | Tests | Reproducibility + fast simulation |
| View-timeout tuning | view transition | Lower timeout = faster failure recovery |

### Throughput x consensus vs throughput x application

Important distinction: **the throughput of the consensus layer** is
the view-rate (or notarization rate). **The throughput of the
application layer** is the rate at which transactions are processed.
For most BFT chains, application throughput >> consensus throughput
*per block*, because each block can hold many transactions.

For a chain with 1 KB transactions and 1 MB blocks: 1000 tx/block
× 10 blocks/sec = 10,000 tx/sec. That's the application-level
throughput. The consensus throughput is 10 blocks/sec, but each
block carries 1000x more transactions than its "consensus overhead."

The carnot bound for the *application* is therefore:

```
app_throughput = blocks_per_sec * tx_per_block
                = (1 / view_latency) * (block_size / tx_size)
```

For Alto's typical config: 10 blocks/sec × 1000 tx/block = 10,000 tx/sec.

### Per-view latency breakdown: the actual time pieces

In a single view, the latency has pieces:

```
time 0:        leader begins proposing
time 0..DELTA:  proposer's notarize broadcast travels to peers
time DELTA:     peers receive proposer's notarize, begin voting
time DELTA..2*DELTA: vote broadcasts travel
time 2*DELTA:   quorum forms at voter i (peer's notarization cert)
time 2*DELTA..3*DELTA: finalize votes + cert formation
time 3*DELTA:   finalization completed (block finalized)
```

So per-view latency is `3 * DELTA` in steady state. For DELTA = 50ms,
that's 150ms per view, giving ~6-7 views/sec as the steady-state
maximum. Going below 150ms requires either (a) fewer phases (Simplex
is already optimal here), or (b) lower latency (faster network), or
(c) pipelined views (which adds complexity).

### Geographic distribution and its cost

WAN latency varies widely:

| Hop | Typical DELTA | Notes |
|---|---|---|
| Same datacenter | 0.5-2 ms | LAN |
| Same region (US-east) | 5-10 ms | few AZs |
| Cross-region (US-east to US-west) | 50-80 ms | common |
| Cross-continent (US to EU) | 80-150 ms | fibers |
| Geo-distributed (US + Asia + EU) | 150-300 ms | extreme |

A Simplex deployment scales linearly with geographic spread: at
DELTA = 200ms, you get ~3 blocks/sec. At DELTA = 10ms, you get
~60 blocks/sec. Both are well within the disk and CPU budgets.

### The single-leader bottleneck and the skip rule

A pathological case: a Byzantine leader who never proposes. Every
view times out at 3 DELTA, so 2*DELTA + 3*DELTA = 5*DELTA per
attempted view. With DELTA = 50ms, that's 250ms per *attempted*
view. At 10 attempted views/sec, the chain might appear stuck at 1
block per 5 sec.

The skip-leader mechanism partially addresses this: if a validator
in view `v` sees its leader is silent, it can preemptively nullify
without waiting for the 3 DELTA timer. The next view has a fresh
leader (different VRF output).

Skipping multiple consecutive views (if leaders keep failing) keeps
the chain moving forward. In the worst case (no leader ever
proposes), the chain makes ~10 nullifications/sec rather than
getting stuck. Eventually a leader proposes, the chain catches up.

---

## Exercises — Simplex

A set of exercises that test the deeper understanding from this
chapter. Try them on paper, then implement in `examples/log` if you
want a code-level check.

### Exercise 11.1 — Trace a Simplex view manually

For a 5-validator cluster (n=5, f=1), with views `v=4` and `v=5`:

1. Manually set up the cluster with one Byzantine validator (noisy
   peer). Use BLS multisig (4 signers per cert).
2. Walk through what happens if the leader of v=4 (call them `L_4`)
   never proposes, and the leader of v=5 (`L_5`) proposes a valid
   block `B5`.
3. List, in order, every message sent, including votes and
   certificates. Note who sends what to whom.
4. Show that the system reaches a finalization on `B5` at v=5.
5. Bonus: in v=6, have `L_4` suddenly send a notarize for a
   competing block `B6'`. What happens?

### Exercise 11.2 — Prove forced-inclusion on a worked example

Suppose you observe a notarization for `B` at view 7. Then, at view 8,
the new leader proposes a different block `B'` that *doesn't extend
B's chain*.

1. Use the quorum-intersection argument from chapter 01 to show this
   can't happen if the network is synchronous.
2. Concretely: how many honest validators voted for `B`? At least
   how many would have to vote for `B'`? What does quorum intersection
   say about the overlap?
3. Re-state the proof for the general case (any `v < v'`).

### Exercise 11.3 — Implement a `nullify` test

Write a test that verifies: when all 4 honest validators nullify
view v, the cluster moves to view v+1 within `view_timeout`.

Sketch:

```rust
#[test]
fn test_nullify_advances_view() {
    let runner = deterministic::Runner::default();
    runner.start(|ctx| async move {
        let mut oracle = build_oracle(ctx, 5);
        let (engines, supervisors) = build_simplex_stack(ctx, 5);
        let initial_views: Vec<_> = supervisors.iter()
            .map(|s| s.current_view())
            .collect();
        // Wait for nullifications to propagate.
        ctx.sleep(Duration::from_secs(2)).await;
        let new_views: Vec<_> = supervisors.iter()
            .map(|s| s.current_view())
            .collect();
        for (initial, new) in initial_views.iter().zip(new_views.iter()) {
            assert!(new > initial, "view did not advance");
        }
    });
}
```

Make the test stricter: ensure that within `2 * leader_timeout`, all
4 honest validators are at view `initial_view + 1` (or higher).

### Exercise 11.4 — Compare BLS threshold vs multisig bandwidth

For n=100 validators finalizing 100 blocks:

1. With BLS threshold: each quorum cert is 1 curve point (~96 bytes).
   Total bytes for certs: 100 certs × 96 bytes = ~10 KB.
2. With Ed25519 multisig: each quorum cert has 100 individual
   signatures × 64 bytes each = 6,400 bytes. Total: ~640 KB.

How does the bandwidth compare? Which is more scalable? What changes
when n = 1,000?

### Exercise 11.5 — The "block-by-block state machine" view

Consider the chain from a block-level perspective (chapter 12
Marshal):

1. For each block B finalized at view V, what certificates are in
   the WAL?
2. What view numbers does a node record for that block?
3. Why does Marshal need to track *both* the block and the certificate
   — can't one be derived from the other?
4. Sketch the data structure Marshal uses to answer "give me block
   h's certificate" or "give me block at digest X."

### Exercise 11.6 — Implement BLS threshold aggregation

In `consensus/src/simplex/scheme/bls12381_threshold/`, find the
`assemble_certificate` function. Implement the aggregation:

```rust
fn assemble_certificate<I>(sigs: I) -> Self::Certificate
where I: IntoIterator<Item = Self::Signature>
{
    // Sum all the signatures
    let agg = sigs.into_iter().fold(Signature::identity(), |a, s| a + s);
    // Note: this requires `Signature` to implement `Add` (curve addition)
    Certificate { aggregate: agg }
}
```

Verify: 1) the result is a valid BLS signature under the group's
pubkey, 2) any subset of 2f+1 signatures produces the same
aggregate (due to commutativity).

### Exercise 11.7 — Write the PBFT-handshake simulation

Implement a small simulation of PBFT with n=4, f=1:

```python
def pbft_round(leader, replicas, value):
    # Pre-prepare
    leader.send_all(("pre-prepare", value))
    for r in replicas:
        r.recv_from_leader(("pre-prepare", value))
        r.send_all(("prepare", value))
    # Prepare
    for r in replicas:
        votes = r.recv_all(("prepare", value))
        if len(votes) >= 2*f + 1:
            r.set_prepared(value)
            r.send_all(("commit", value))
    # Commit
    for r in replicas:
        votes = r.recv_all(("commit", value))
        if len(votes) >= 2*f + 1:
            r.commit(value)
            return value
    return None
```

Run it with one Byzantine replica (drops all messages). Verify that
the 3 honest replicas still commit.

---


## Appendix A — The Voter's state machine, in detail

`consensus/src/simplex/actors/voter/`. Six files. Let me trace what
each does.

### `state.rs` — the durable state

The `Voter` holds a `Journal` (chapter 06) for the WAL. On every
state-changing action:

```rust
fn on_finalization(&mut self, cert: Finalization) -> Result<()> {
    // Step 1: write to WAL
    self.journal.append(self.partition, cert.encode()).await?;

    // Step 2: sync to disk
    self.journal.sync_all().await?;

    // Step 3: only then broadcast
    self.vote_sender.send(Recipients::All, cert.encode(), false).await?;
}
```

The order is critical: WAL write → sync → broadcast. If we crash between
sync and broadcast, on restart we'll replay the WAL and re-broadcast.
No Byzantine behavior.

### `slot.rs` — per-height state

Each "slot" (height in the chain) has:
- `finalized: Option<Finalization>` — the finalization at this height (if
  any).
- `notarizations: BTreeMap<View, Notarization>` — notarizations seen at
  various views for this height.
- `nullifications: BTreeSet<View>` — nullifications seen.
- `blocks: HashMap<Digest, Block>` — verified blocks at this height.

The Voter tracks slot state for **current_height** and a window of
recent heights (for backfill).

### `round.rs` — per-view state

Each view (within a slot) has:
- `leader: PublicKey` — who should propose.
- `timer_l: Option<Instant>` — leader timeout.
- `timer_a: Option<Instant>` — activity timeout.
- `votes_notarize: BTreeMap<PublicKey, Notarize>` — incoming notarize votes.
- `votes_nullify: BTreeMap<PublicKey, Nullify>` — incoming nullify votes.
- `votes_finalize: BTreeMap<PublicKey, Finalize>` — incoming finalize votes.
- `notarization: Option<Notarization>` — assembled notarization (if reached quorum).
- `nullification: Option<Nullification>` — assembled nullification.
- `finalization: Option<Finalization>` — assembled finalization.

Per-view state is **transient** — created when entering a view,
discarded when leaving (or kept briefly for backfill).

### `actor.rs` — the main loop

The Voter's main event loop:

```rust
async fn run(mut self) {
    loop {
        select! {
            // Outgoing from this node
            msg = self.vote_sender.recv() => {
                self.on_self_vote(msg).await;
            }

            // Incoming from the Batcher (certificates)
            cert = self.cert_receiver.recv() => {
                self.on_certificate(cert).await;
            }

            // Timer l (leader timeout)
            _ = self.clock.sleep_until(self.timer_l) => {
                self.on_leader_timeout().await;
            }

            // Timer a (activity timeout)
            _ = self.clock.sleep_until(self.timer_a) => {
                self.on_activity_timeout().await;
            }

            // View transition triggered externally
            cmd = self.command_receiver.recv() => {
                self.on_command(cmd).await;
            }

            // Shutdown
            _ = self.stop_signal => {
                break;
            }
        }
    }
}
```

`commonware-macros::select!` polls all branches; whichever is ready
first runs its handler. Cooperative multitasking.

## Appendix B — The Batcher, in detail

`consensus/src/simplex/actors/batcher/`. Four files.

### `actor.rs` — the Batcher main loop

The Batcher:
1. Receives incoming votes from the "vote" P2P channel.
2. Groups them by view.
3. When 2f+1 votes accumulate for the same `(view, payload)`:
   - Batch-verify the signatures (chapter 04 BatchVerifier).
   - If all valid: form certificate, forward to Voter.
   - If invalid: bisect to find the bad vote, block! the sender (chapter 05).
4. Periodically prunes old views from memory.

### `round.rs` — per-view vote tracking

```rust
struct RoundState {
    notarizes: BTreeMap<Digest, BTreeSet<(PublicKey, Signature)>>,
    nullifies: BTreeMap<PublicKey, Signature>,
    finalizes: BTreeMap<Digest, BTreeSet<(PublicKey, Signature)>>,
}
```

`notarizes` groups votes by payload — same `view`, different payloads
→ separate sets. Only one can reach quorum.

`nullifies` is per-signer (each signer nullifies once per view).

`finalizes` groups votes by payload (similar to notarizes).

### `verifier.rs` — batch verification

The batch verifier uses `BatchVerifier::add` to accumulate votes, then
`verify(rng, strategy)` to batch-verify all at once.

```rust
fn try_finalize(&mut self, view: View, payload: Digest, votes: BTreeSet<(PK, Sig)>) {
    let mut verifier = S::BatchVerifier::new();
    for (pk, sig) in &votes {
        let msg = encode_finalize(view, payload);
        verifier.add(namespace, &msg, pk, sig);
    }
    if !verifier.verify(&mut self.rng, &self.strategy) {
        // Bisect to find the bad vote
        // (See below)
        return;
    }
    // All valid — form certificate
    let cert = Finalization { view, payload, signatures: votes };
    self.voter_sender.send(cert).await;
}
```

Batch verification is **much faster** than N individual verifications,
especially for BLS where pairings can be batched.

### `bisect` — finding the bad vote

When batch verification fails:

```rust
fn bisect(&self, votes: &BTreeSet<(PK, Sig)>) -> Option<(PK, Sig)> {
    if votes.len() == 1 {
        return votes.iter().next().cloned();
    }

    // Split in half
    let mid = votes.len() / 2;
    let (left, right) = votes.split_at(mid);

    // Check left half
    if !verify_batch(left) {
        return self.bisect(left);
    }
    // Check right half
    if !verify_batch(right) {
        return self.bisect(right);
    }

    None  // Both halves verify, but the batch didn't?! Bug in our code.
}
```

Logarithmic search through votes to find the culprit. Bisecting through
1000 votes takes ~10 iterations.

## Appendix C — The Resolver actor

`consensus/src/simplex/actors/resolver/`. The Resolver is a thin wrapper
around `commonware-resolver` (chapter 08):

```rust
pub struct ResolverActor<R: Resolver> {
    resolver: R,
    pending: HashMap<Key, oneshot::Sender<Block>>,
}

impl ResolverActor {
    fn fetch(&mut self, key: Key) -> oneshot::Receiver<Block> {
        let (tx, rx) = oneshot::channel();
        self.pending.insert(key, tx);
        self.resolver.fetch(key);
        rx
    }

    fn on_delivery(&mut self, delivery: Delivery<Key, Subscriber>, value: Block) {
        if let Some(tx) = self.pending.remove(&delivery.key) {
            let _ = tx.send(value);
        }
    }
}
```

The Voter calls `resolver.fetch(cert.proposal.payload)` when it needs
the block contents behind a notarization.

## Appendix D — The Application trait, in detail

`consensus/src/lib.rs:101-191`. The `Automaton` trait:

```rust
pub trait Automaton: Clone + Send + 'static {
    type Context;
    type Digest: Digest;

    fn propose(&mut self, context: Self::Context)
        -> impl Future<Output = oneshot::Receiver<Self::Digest>> + Send;

    fn verify(&mut self, context: Self::Context, payload: Self::Digest)
        -> impl Future<Output = oneshot::Receiver<bool>> + Send;
}
```

### `propose`

Called when this node is the leader for a view. Returns a channel
(oneshot) that will resolve to the proposed digest.

Two ways to respond:
- **Send a digest**: `tx.send(my_digest)`. The Voter uses this digest as
  the proposal payload.
- **Drop the channel**: signal that you can't propose (still syncing,
  no transactions, etc.). The Voter nullifies the view.

Important: the doc warns explicitly (line 117-121):

> Returning a payload from `propose` commits the local proposer to
> verifying the same `(context, payload)`.

So once you propose, you're locked in. If you later can't verify
your own proposal, you've created a Byzantine condition.

### `verify`

Called when this node sees a proposal from another leader. Returns a
channel that will resolve to `true` (valid) or `false` (invalid).

Three ways to respond:
- **Send `true`**: valid, vote to notarize.
- **Send `false`**: permanently invalid, nullify.
- **Keep the channel pending**: not ready to decide yet (e.g., waiting
  for missing data). The Voter waits.

The doc warns (line 130-140):

> Once the returned channel resolves or closes, consensus treats
> verification as concluded and will not retry the same request.
> Implementations should therefore keep the request pending while the
> verdict may still change.

So `verify` is **not** a synchronous function. You can wait as long as
you need, but once you decide, you're done.

### `certify`

`CertifiableAutomaton::certify` (chapter 11 main text). Called after
notarization to ask "should I vote to finalize?"

Same pattern: oneshot channel returning bool, can keep pending.

### Verdict semantics

For `verify` and `certify`, the verdict is "permanent" once delivered:
- Returning `false` means "this is permanently invalid."
- Closing the channel means "I can never decide" (shutdown).
- Keep pending if the verdict might change.

Returning `false` (or closing) for a temporary condition is a bug — the
protocol will halt.

## Appendix E — The four-actor coordination

The Voter, Batcher, Resolver, and Application communicate via **mailboxes**
(chapter 15). The Voter is the orchestrator:

```
┌────────────┐
│   Voter    │
└─┬───┬───┬──┘
  │   │   │
  │   │   └──→ Application (propose/verify/certify calls)
  │   └──────→ Resolver (fetch block contents)
  └──────────→ Batcher (assembled certificates)
```

Mailboxes:

| From | To | Message | Purpose |
|---|---|---|---|
| P2P (vote channel) | Batcher | `Notarize / Nullify / Finalize` votes | Collect signatures |
| Batcher | Voter | `Notarization / Nullification / Finalization` certificates | Notify voter of progress |
| P2P (cert channel) | Voter | Incoming certificates from other nodes | Notify voter of network consensus |
| Voter | P2P | Outgoing votes and certificates | Broadcast to peers |
| Voter | Resolver | Fetch requests | Get missing blocks |
| Resolver | Voter | Block data deliveries | Provide fetched blocks |
| Voter | Application | `propose(context)`, `verify(context, payload)`, `certify(round, payload)` | Drive the protocol |

The Voter never blocks on Application calls — they're all oneshot
channels. If Application takes 5 seconds to decide `verify`, the Voter
keeps processing other events.

## Appendix F — The Engine.start() lifecycle

When you call `engine.start(pending, recovered, resolver)`:

```rust
pub fn start(self, pending: Pending, recovered: Recovered, resolver: Resolver)
    -> (MailboxHandle, ...)
{
    let (voter_tx, voter_rx) = mailbox();
    let (batcher_tx, batcher_rx) = mailbox();

    // Spawn voter
    let voter_handle = self.context.child("voter").spawn(move |ctx| async move {
        let mut voter = Voter::new(ctx, voter_rx, batcher_tx, resolver);
        voter.run().await
    });

    // Spawn batcher
    let batcher_handle = self.context.child("batcher").spawn(move |ctx| async move {
        let mut batcher = Batcher::new(ctx, batcher_rx, voter_tx);
        batcher.run().await
    });

    (pending.handle(), recovered.handle())
}
```

Where `pending`, `recovered`, `resolver` are the P2P channel handles
(chapter 05) for votes, certificates, and resolver messages
respectively.

The Engine wires up:
- The three P2P channels.
- The supervisor tree (chapter 15): voter and batcher are children of
  the engine's context.
- The mailbox connections between them.
- The shutdown coordination.

## Appendix G — The four-actor message types

From `consensus/src/simplex/actors/voter/ingress.rs`:

```rust
pub enum Message {
    Notarize(Notarize),
    Nullify(Nullify),
    Finalize(Finalize),
    Notarization(Notarization),
    Nullification(Nullification),
    Finalization(Finalization),
    Proposal(Proposal),       // leader's proposed payload
    // ... internal commands ...
}
```

`Notarize / Nullify / Finalize` are **votes** (from one validator).
`Notarization / Nullification / Finalization` are **certificates**
(quorum-aggregated). `Proposal` is the leader's block (carrying the
payload).

The Voter differentiates votes from certificates by sender (votes from
self, certificates from Batcher or other nodes).

## Appendix H — The state recovery on restart

`consensus/src/simplex/actors/voter/state.rs`. On restart:

```rust
fn recover(journal: Journal) -> Result<RecoveredState> {
    let mut recovered = RecoveredState::default();
    let mut stream = journal.replay(buffer_size);

    while let Some((section, offset, item)) = stream.next().await? {
        match decode(&item)? {
            Message::Notarize(v) => recovered.notarizes.push(v),
            Message::Nullify(v) => recovered.nullifies.push(v),
            Message::Finalize(v) => recovered.finalizes.push(v),
            Message::Notarization(c) => recovered.notarizations.push(c),
            // ...
        }
    }

    // Rebuild in-memory state from recovered messages
    recovered.rebuild_state();
    Ok(recovered)
}
```

After replay, the Voter has all votes/certificates it had before the
crash. It then continues from where it was.

If a finalization was in the WAL but never broadcast, the Voter will
re-broadcast it on startup. If a vote was received but not processed,
the Voter processes it now.

## Appendix I — The relation to the WAL pattern

The WAL pattern from chapter 06 is critical here. Without it:

- Vote received, processed, broadcast → done.
- Vote received, processed, NOT broadcast yet → on crash, lost.
- Vote received, processed, broadcast, crashed before WAL → on restart,
  others saw your vote but you don't have it. Could re-vote
  (contradicting yourself if vote was conditional).

With the WAL:

- Vote received → append to WAL, sync, process, broadcast.
- Crash anywhere → on restart, replay WAL, re-broadcast if needed.

The WAL makes the Voter **crash-safe AND Byzantine-safe** simultaneously.

## Appendix J — Test patterns

From `consensus/src/simplex/mod.rs` (the `all_online` test, etc.):

### Setting up the test

```rust
fn setup_test() -> TestHarness {
    let executor = deterministic::Runner::timed(Duration::from_secs(300));
    let (network, oracle) = Network::new_with_peers(context, peers, true);
    // ... register channels, link peers, create fixtures ...
}
```

### Asserting Byzantine safety

```rust
for reporter in reporters.iter() {
    reporter.assert_no_faults();    // No peer voted for two blocks at the same height
    reporter.assert_no_invalid();   // No peer voted with an invalid signature
}
```

### Asserting progress

```rust
let required_containers = View::new(100);
while latest < required_containers {
    latest = monitor.recv().await.expect("event missing");
}
```

### Asserting no forks

```rust
for view in View::range(View::new(1), latest_complete) {
    let payloads = notarizes.get(&view).unwrap();
    assert!(payloads.len() <= 1, "fork at view {view}");
}
```

### Asserting determinism

```rust
let state1 = slow_and_lossy_links::<MinPk>(seed);
let state2 = slow_and_lossy_links::<MinPk>(seed);
assert_eq!(state1, state2);
```

## Appendix K — Performance knobs

The Simplex config exposes:

| Knob | Effect |
|---|---|
| `leader_timeout` | How long to wait for leader's proposal. Shorter = faster recovery from bad leader, but more false timeouts. |
| `certification_timeout` | How long to wait for certification. With erasure coding, this is your "data delivery deadline." |
| `timeout_retry` | How long until re-broadcasting votes (to overcome dropped messages). |
| `fetch_timeout` | How long until retrying a fetch from peers. |
| `activity_timeout` | How long to track past views (for backfill). |
| `skip_timeout` | How many views a leader can skip before being skipped. |
| `fetch_concurrent` | Max concurrent fetch operations. |
| `mailbox_size` | Bounded ready queue size. |
| `replay_buffer` | Size of the WAL replay buffer. |
| `write_buffer` | Size of the WAL write buffer. |
| `page_cache` | The page cache for journals. |

Tune for your workload. Common defaults work for typical block times
(~200-500ms).

## Appendix L — Common gotchas

### The `verify` returning `false` for temporary conditions

```rust
// BAD — returning false because data isn't here yet
async fn verify(&mut self, ...) -> Receiver<bool> {
    let (tx, rx) = oneshot::channel();
    if !self.has_block(payload) {
        tx.send(false);  // BUG — block might arrive later
    }
    rx
}

// GOOD — keep pending until the verdict is stable
async fn verify(&mut self, ...) -> Receiver<bool> {
    let (tx, rx) = oneshot::channel();
    self.fetch_block(payload).await;
    let result = self.check_block(payload);
    tx.send(result);
    rx
}
```

### Re-proposing in `propose`

`propose` is called once per view where you're leader. If you propose
once and the view fails, the next view's leader proposes. Don't
re-propose in the same view — that's Byzantine behavior.

### Submitting votes with the wrong namespace

```rust
// BAD — using the wrong namespace
let sig = signer.sign(b"MY_NAMESPACE", msg);

// GOOD — using the protocol's expected namespace
let sig = signer.sign(SIMLEX_NAMESPACE, msg);
```

The Voter (via the scheme) uses a specific namespace. Wrong namespace =
signature won't verify. Use the constant.

## Where to look in the code (expanded)

- `consensus/src/lib.rs:99-244` — the trait surface.
- `consensus/src/simplex/actors/mod.rs:1-5` — the four actors.
- `consensus/src/simplex/actors/voter/` — voter state machine.
- `consensus/src/simplex/actors/batcher/` — vote collection + verification.
- `consensus/src/simplex/actors/resolver/` — block fetching.
- `consensus/src/simplex/mod.rs:30-119` — the protocol spec.
- `consensus/src/simplex/mod.rs:790-1020` — the integration test.
- `consensus/src/simplex/scheme/` — the four supported signature schemes.
- `consensus/src/simplex/elector.rs` — leader election.
- `examples/log/src/main.rs` — a tiny running app using all of this.

## If you only remember three things

1. **Six traits you implement: `Automaton`, `CertifiableAutomaton`, `Relay`, `Reporter`, `Monitor`, `Block`.** Everything else is provided.
2. **Four actors doing the work: `Batcher` (collects votes), `Voter` (drives views), `Resolver` (fetches certs), `Application` (your code).** Non-blocking between them via channels.
3. **Every primitive we studied is a tool in this stack.** Runtime for async, codec for the wire, crypto for identity, P2P for delivery, storage for the WAL, coding for blocks, resolver + broadcast for data movement.

## Cross-reference

For concrete types (`Finalization`, `Notarization`, `Round`, `Context`),
the full `Elector` trait, the `ForwardingPolicy` enum, the
`Reporter::Activity` enum, and the full Simplex Config struct, see
**Chapter 21 — Cross-cutting patterns** (`chapters/21-cross-cutting.md`).

→ Next: **Chapter 12 — Marshal** (the layer that turns Simplex certificates
into downloaded, verified, finalized blocks) and the rest of the consensus
crate.