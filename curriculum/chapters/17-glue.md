# Chapter 17 — Glue: default constructions that span primitives

> Sensible defaults when you want a full stack without assembling every piece.

## The problem

You want to build a stateful blockchain application on Commonware. You
need:
- A consensus engine (Simplex).
- A marshal to bridge certs and blocks.
- An authenticated database (QMDB).
- A way to manage speculative state across forks.
- A simulation harness for testing.

You could assemble all of this yourself. But it's tedious, and you might
miss subtleties (e.g., the `inline` vs `deferred` marshal choice from
chapter 12).

`commonware-glue` provides **default constructions** that span multiple
primitives. Use them when you're building a standard stateful
blockchain.

## The two modules

`glue/src/lib.rs:7-12`:

```rust
pub mod stateful;        // stateful app on top of QMDB
#[cfg(any(test, feature = "test-utils"))]
pub mod simulate;        // simulation harness
```

## `stateful` — speculative state on QMDB

`glue/src/stateful/mod.rs:1-17`:

> A stateful application built on consensus must maintain speculative state
> for every pending chain built on top of the finalized tip. This module
> provides the [`Application`] trait and a [`Stateful`] actor that automates
> that bookkeeping:
>
> 1. Before each `propose` or `verify`, the actor forks unmerkleized batches
>    from the parent block's pending state (or from committed database state
>    if the parent has been finalized).
> 2. The application executes against those batches and returns merkleized
>    results, which the actor stores as a new pending tip keyed by the
>    block's digest.
> 3. On finalization, the actor applies the winning tip's changesets to the
>    underlying databases and prunes pending entries from dead forks.

This is the **speculative execution** pattern. While consensus is racing
multiple branches (each view could finalize a different block), you
execute all of them. The "winning" branch's state gets applied to the
durable database.

The actor (`stateful/actor/`) handles:
- Forking batches per block.
- Storing pending tips in memory (no disk writes on the hot path).
- Applying the winning tip on finalization.
- Pruning dead forks.

The DB layer (`stateful/db/`):
- `Unmerkleized` trait — batches that haven't been hashed yet.
- `Merkleized` trait — hashed batches ready to commit.
- `ManagedDb` trait — the lifecycle of a database.
- `DatabaseSet` — group multiple databases into one unit.
- `db::p2p::standard` and `db::p2p::compact` — P2P resolvers for fetching
  sync data from peers.

## The `Application` trait

`glue/src/stateful/mod.rs:125-285`. Defines:

```rust
pub trait Application<E>: Clone + Send + 'static {
    type SigningScheme: Scheme;
    type Context: Epochable + Viewable + Send;
    type Block: CertifiableBlock<Context = Self::Context>;
    type Databases: DatabaseSet<E>;
    type InputProvider: Send;

    fn sync_targets(block: &Self::Block) -> ...;
    fn genesis(&mut self) -> impl Future<Output = Self::Block> + Send;
    fn propose(&mut self, context, ancestry, batches, input)
        -> impl Future<Output = Option<Proposed<Self, E>>> + Send;
    fn verify(&mut self, context, ancestry, batches)
        -> impl Future<Output = Option<Merkleized>> + Send;
    fn apply(&mut self, context, block, batches)
        -> impl Future<Output = Merkleized> + Send;
    fn finalized(&mut self, context, block, databases)
        -> impl Future<Output = ()> + Send { async {} }
}
```

You implement this trait. The actor handles all the state-management
bookkeeping. You just write the actual state-transition logic.

## Lazy recovery — restart without losing speculative work

`glue/src/stateful/mod.rs:70-80`:

> Pending state is kept entirely in memory to avoid disk writes on the
> consensus hot path. After a restart the map is empty, but the actor
> recovers lazily: when `propose` or `verify` encounters a parent whose
> state is missing, the actor walks back through the block DAG to the
> nearest known ancestor or the finalized tip, then replays forward via
> `Application::apply` to fill the gap.

So:
- Pending state lives in memory (fast).
- On crash, pending state is gone.
- On restart, the actor lazily replays from the last finalized tip when
  needed.

This is the right tradeoff: hot path stays fast; cold path (recovery)
takes a bit longer.

## `simulate` — the test harness

`glue/src/simulate/mod.rs:1-12`:

> Provides a configurable test framework that composes the core consensus
> stack (e.g. p2p, simplex, marshal, broadcast, application) with action
> injection, progress tracking, and property checking.

The components:
- `EngineDefinition` — trait for wiring up a validator's full stack.
- `Plan` — declarative test config with action injection.
- `Team` — manages running validators (start, crash, restart).
- `ProgressTracker` — monitors finalization progress.
- `action` — inject actions mid-test (kill validator, partition network, etc.).
- `property` — check properties after the test.
- `processed` / `reporter` — observe what happened.

This is what Alto and other production apps use for their end-to-end
testing. You can simulate 100 validators, run them for 10,000 simulated
seconds, inject 50 random crashes and partitions, verify consensus made
progress, all deterministically.

## When to use Glue

Use it when:
- You're building a stateful blockchain app.
- You want speculative execution across forks.
- You want a full simulation harness.

Don't use it when:
- You only need a specific primitive (just use that crate directly).
- Your app has custom state-management needs that Glue's pattern doesn't
  fit.

## Where to look in the code

- `glue/src/lib.rs` — the entry point.
- `glue/src/stateful/mod.rs:125-285` — the `Application` trait.
- `glue/src/stateful/actor/` — the `Stateful` actor (manages speculative state).
- `glue/src/stateful/db/` — the database lifecycle traits.
- `glue/src/simulate/` — the test harness.

## Speculative execution — where it lives in the literature

The "fork-and-merge" pattern in this chapter has many names:

- **Speculative execution** (software/runtime terminology).
- **Optimistic concurrency** (database terminology, DDIA chapter 7).
- **Fork-join model** (parallel computing, FJ framework, Lea 2000).
- **Multi-version concurrency control** (MVCC, database internals).

Database MVCC (Silberschatz, "Database System Concepts", chapter 14)
is the closest cousin. Postgres implements it as: every row carries
a `xmin` (creator transaction) and `xmax` (deleter transaction);
`SELECT` sees rows where `xmin < current_txid < xmax`; updates are
copy-on-write; commit/abort adjusts visibility.

Commonware's Stateful actor is **MVCC on a fork-tree of blocks**:

```
finalized root
      |
   block N (cert'd)
      |
   +--+---+----+-----+
   |  |   |    |     |
  forks at block N+1
   A  B   C    D     E
```

Each fork is a separate `Merkleized` result, kept in memory, keyed
by block digest. The "winning fork" is whatever block Simplex
finalizes; the loser forks get pruned.

DDIA chapter 7 ("Transactions") is the right background reading;
the "snapshot isolation" section is most directly applicable.
Commonware's design sits one level up from database transactions
because the unit of conflict is a block (containing many state
transitions) rather than a transaction (containing one).

## Where the actor model fits

The `Stateful` actor's pattern matches exactly the actor-model
discussion in chapter 15:

- **Mailbox**: messages from Marshal (`Propose`, `Verify`,
  `Finalize`) arrive via the actor's `Receiver`.
- **State**: pending forks keyed by block digest.
- **Bounded ready queue**: Marshal never overruns the actor
  during healthy operation; under simulated heavy partition, the
  mailbox backpressures the upstream actor.
- **Supervision**: the actor is a child of the runtime's
  supervisor; if Simplex's Marshal aborts, the Stateful actor
  aborts (and its in-memory pending maps are lost — to be
  lazily rebuilt on restart).

The application-defined `Application<E>` trait is **what makes this
an actor-shaped interface**: the trait's methods are the messages,
and the actor's struct holds the Application instance plus the
pending map.

## The full `Application` trait, dissected

Beyond the appendix's walkthrough, here is each method's **why**,
not just its signature.

### `genesis`

`fn genesis(&mut self) -> impl Future<Output = Self::Block> + Send`.

Called exactly once. Returns the **floor block** — the block at
height 0 from which everyone starts. The implementation must be
deterministic across all nodes: every validator's `genesis` should
produce the same bytes (so all nodes converge on the same root).
This is the **purpose of determinism in Commonware**: agreement on
the genesis block, agreement on the state at every subsequent
height.

### `propose`

`fn propose(&mut self, context, ancestry, batches, input) -> Option<Proposed<Self, E>>`.

Called when this node is the **leader of the next view**. The
ancestry stream supplies parent blocks backward to genesis (or to
the already-finalized block, whichever's closer). The `batches` are
forked from the parent's state.

The implementation:

1. Pulls transactions from `input` (mempool, oracle feed).
2. Executes them against the forked state.
3. Returns the proposed block + the resulting merkleized state.

If the leader can't fill a block (e.g., empty mempool), it returns
`None`. The Simplex engine treats `None` as "skip this view" and
the next leader takes over.

The `propose` future can **block** waiting on `input` — that's why
it's a future, not a sync function.

### `verify`

`fn verify(&mut self, context, ancestry, batches) -> Option<Merkleized>`.

Called when this node is **receiving a proposal** from another
leader. The same ancestry stream is supplied. The batches are
forked from the proposer's claimed parent.

The implementation:

1. Verifies that the batches are valid (signatures, sequence
   numbers, state transitions).
2. Executes them against the forked state to build the merkleized
   result (this is the speculative-execution path).
3. Returns `Some(merkleized)` if accepted (state matches what the
   proposer claimed) or `None` if permanently invalid.

Critically: **verification produces the same merkleized result as
propose**. This is the property that makes the protocol safe —
two honest nodes verifying the same block produce the same root
hash, which they can prove to each other in the certificate.

### `apply`

`fn apply(&mut self, context, block, batches) -> Merkleized`.

Called during **lazy recovery** when the actor needs to reconstruct
state for a parent block that's missing from the `pending` map.
The block is known-good (was previously notarized); the application
just executes it.

The constraint: `apply` must produce the **same merkleized result
as `verify` did for the same block**. Otherwise the lazy-recovery
state diverges from the originally-verified state, and the database
roots will not match.

### `finalized`

`fn finalized(&mut self, context, block, databases)`.

Hook for "this block is now durably applied." Default is no-op.
Override for: emitting metrics, updating external systems
(triggering a bridge, sending a notification, etc.).

### `sync_targets`

`fn sync_targets(&block) -> <Self::Databases as DatabaseSet<E>>::SyncTargets`.

Returns the per-database sync target. For each database, the sync
target is "what range of operations is this database responsible
for keeping" — typically the latest committed operation count or
some operation-count checkpoint.

The wrapper uses this during state sync to decide which peer to
ask for which operation range. See the `SyncPlan` appendix below.

## `DatabaseSet` — what each database does

The `DatabaseSet` is **N application-managed databases** wrapped
in a single trait. A typical set:

```rust
struct MyDatabases {
    current_state: qmdb::current::Db,           // mutable map (key -> value)
    transaction_log: qmdb::keyless::Db,        // append-only log of operations
}

impl DatabaseSet<Engine> for MyDatabases {
    type Unmerkleized = (current_state::Fork, log::Fork);
    type Merkleized = (current_state::Merkleized, log::Merkleized);
    type SyncTargets = (current_state::SyncTarget, log::SyncTarget);
    type Digest = (current_state::Digest, log::Digest);

    fn databases(&self) -> &Self { self }

    fn fork(&self, parents: Self::Unmerkleized) -> Self {
        // For each child database, fork from the parent's snapshot
        Self {
            current_state: self.current_state.fork(parents.0),
            transaction_log: self.transaction_log.fork(parents.1),
        }
    }
    // ... merkleize, finalize, sync ...
}
```

The fork-and-merge dance:

| Phase | All databases |
|---|---|
| **fork** | Take snapshot; produce mutable, unmerkleized children |
| **execute** | Application logic modifies the children |
| **merkleize** | Hash and commit; produce `Merkleized` |
| **finalize** | Apply winner's merkleized to underlying storage |

The `SyncTarget` per database is the metadata a syncing client
needs: "how far has this database progressed." For a QMDB current
state, that's the **last committed operation index**. For a
keyless log, it's the **last persisted segment**.

## Fork-join parallelism in the spec

"Fork unmerkleized batches" in the docstring refers to the
**per-block fork** the actor performs. The **join** is implicit in
the merkleization step: once the application returns the
`Merkleized` result, the actor combines it with other state.

This is **single-block parallelism** within a block, not
multi-block. The simulator (chapter 21's harness) can fork
**multiple blocks in parallel** when running pre-execution against
many candidates — that's where speculative execution scales
across cores.

The Commonware runtime's `ctx.shared(true).spawn(...)` (chapter 02)
allows the application to spawn parallel work in `verify` /
`propose`. A typical pattern:

```rust
async fn verify(&mut self, ctx, ancestry, batches) -> Option<...> {
    // Spawn 4 verifications in parallel for the 4 branches of ancestry
    let mut handles = vec![];
    for parent in ancestry_branch {
        let handle = ctx.shared(true).spawn(async move {
            verify_branch(parent).await
        });
        handles.push(handle);
    }
    // Join: collect results
    let results = futures::future::join_all(handles).await;
    // ...
}
```

This is **real fork-join parallelism** within the actor model —
the actor is one task, but its application can spawn sub-tasks
under the same supervisor.

## DDIA transaction models — and what Stateful provides

Compare Commonware's `Stateful` to DDIA's transaction levels:

| DDIA isolation level | Description | Commonware equivalent |
|---|---|---|
| **Read Committed** | Sees only committed data | n/a (no committable partial state) |
| **Snapshot Isolation** | Reads a consistent snapshot | `propose`/`verify` execute against the parent's snapshot |
| **Serializable** | Equivalent to some serial execution | Yes — the chain enforces linearizability by Simplex's safety |

The **highest** level is "Serializable Snapshot Isolation (SSI)" in
databases: even snapshot-isolated transactions can produce anomalies
that require abort-and-retry. Commonware's Stateful does not have
this complexity because the **Simplex consensus layer** enforces a
single total order — there is no chance of cross-block anomalies.
The "serializability" is protocol-level, not database-level.

The corollary: if your `Application` is non-deterministic (returns
different merkleized roots from the same input on different runs),
the protocol will fork. Stateful assumes determinism. See the
gotchas appendix.

## Optimistic concurrency control — when to rollback

The `verify` method is the **optimistic** path. The application
guesses that the proposed batches are valid, executes them, and
returns the merkleized result. If verification fails, the result
is dropped on the floor and the actor leaves it out of `pending`.

There's no "rollback" because there's nothing to roll back to —
the speculative execution was on a copy (the fork). If verify
fails, the fork is simply not added to `pending`. The committed
state (whatever's in the underlying QMDB) is unchanged.

This is **optimistic concurrency control** in the database sense:
"try the operation; if it would conflict, abort." Here, "conflict"
means "the merkleized state doesn't match what verify expects," and
"abort" means "drop the fork from pending."

The actual *commit* happens later — on finalization. By then the
fork has been built, executed, accepted by 2f+1 validators, and
becomes part of the consensus. Now `merkleize` + `finalize` apply
it to durable state.

## The `Stateful` actor's loop, in full

The actor's `run` future processes five message types:

```rust
async fn run(mut self) {
    loop {
        select! {
            // 1. Marshal: wants us to propose a block
            req = self.propose_receiver.recv() => {
                self.handle_propose(req).await;
            }
            // 2. Marshal: wants us to verify a peer's proposal
            req = self.verify_receiver.recv() => {
                self.handle_verify(req).await;
            }
            // 3. Marshal: a block is being finalized
            notif = self.finalize_receiver.recv() => {
                self.handle_finalize(notif).await;
            }
            // 4. Sync coordinator: state-sync update
            sync = self.sync_receiver.recv() => {
                self.handle_sync(sync).await;
            }
            // 5. Someone dropped shutdown; exit cleanly
            _ = self.stop_signal => break,
        }
    }
}
```

The order of these isn't accidental: `finalize` is third so that a
proposal-verify cycle can happen quickly without starvation.
`propose` and `verify` both require speculative execution that
might miss state; `handle_propose` / `handle_verify` may trigger
lazy recovery (next section). Finalization is the only "definitely
moves forward" event.

## The fork-and-merge pattern — code path

When `propose` arrives from Marshal:

```rust
async fn handle_propose(&mut self, req: Propose) {
    // 1. Resolve the parent state. If pending map has it, great.
    //    If not, lazily replay from finalized tip.
    let parent_state = self.get_or_replay_state(&req.parent).await;

    // 2. Fork — produce unmerkleized batches
    let batches = self.database.fork(parent_state);

    // 3. Run the application logic
    let proposed = self.application.propose(req.context, req.ancestry, batches, &mut req.input).await;

    // 4. If proposed, cache the merkleized state under block digest
    if let Some(p) = proposed {
        self.pending.insert(p.block.digest(), p.merkleized);

        // 5. Tell Marshal we're done proposing
        let _ = req.response_tx.send(p.block);
    }
}
```

When `verify` arrives:

```rust
async fn handle_verify(&mut self, req: Verify) {
    let parent_state = self.get_or_replay_state(&req.parent).await;
    let batches = self.database.fork(parent_state);

    let merkleized = self.application.verify(req.context, req.ancestry, batches).await;

    if let Some(m) = merkleized {
        self.pending.insert(req.block.digest(), m);
        let _ = req.response_tx.send(Some(m));   // tell marshal which root we computed
    } else {
        let _ = req.response_tx.send(None);
    }
}
```

When `finalize` arrives:

```rust
async fn handle_finalize(&mut self, notif: Finalize) {
    // 1. Find the pending state for this block
    let merkleized = self.pending.remove(&notif.block.digest())
        .expect("pending state must exist for finalized block");

    // 2. Apply to underlying databases
    self.database.merkleize(merkleized).await?;
    self.database.finalize(self.application.sync_targets(&notif.block)).await?;

    // 3. Tell application
    self.application.finalized(notif.context, &notif.block, &self.database).await;

    // 4. Prune dead forks (blocks whose parent cert is not this block)
    self.prune_dead_forks(&notif.block);

    // 5. Ack to marshal
    let _ = self.marshal_ack_tx.send(notif.block.height);
}
```

The **commit point** is the `database.merkleize` call. After this,
the block's effects are durable in the QMDB/whatever-database-the-
application-uses.

## The finalization flow — full sequence

A block moves through the actor as follows (once we observe its
finalization):

1. **Marshal** observes a notarization + nullification-free Simplex
   finalization for block H.
2. **Marshal** sends `Finalize { block, certificate }` to Stateful.
3. **Stateful** looks up `block.digest()` in `pending` map.
   - **Hit**: real merkleized state is right there; proceed.
   - **Miss**: bug or crash recovery — replay from finalized tip.
4. **Stateful** calls `database.merkleize(state)`. This commits
   the operations to the underlying storage.
5. **Stateful** calls `database.finalize(sync_targets)`. This
   updates the sync coordinator's targets.
6. **Stateful** calls `application.finalized(...)` (optional hook).
7. **Stateful** prunes pending forks that lost.
8. **Stateful** acks to Marshal.

The rough timing, on a typical finalization:

- Step 1: sim time = block finalized by Simplex consensus.
- Steps 2–3: sub-millisecond.
- Steps 4–5: bounded by storage latency, typically microseconds.
- Steps 6–8: application-defined; usually zero or short.

## Lazy recovery, in detail

The actor's `get_or_replay_state` is the heart of recovery:

```rust
async fn get_or_replay_state(&mut self, target: Block) -> DatabaseSet::Merkleized {
    // 1. Check in-memory cache
    if let Some(state) = self.pending.get(&target.digest()) {
        return state.clone();
    }

    // 2. Walk backward through the block DAG to a known ancestor
    let mut current = target.clone();
    let mut missing = vec![];
    loop {
        if let Some(known) = self.find_known_state(&current) {
            break;
        }
        missing.push_front(current);
        current = current.parent();
    }

    // 3. Replay forward
    let mut state = self.find_known_state(&current).unwrap();
    for block in missing {
        let batches = self.database.fork(state);
        state = self.application.apply(...);
        self.pending.insert(block.digest(), state.clone());
    }

    state
}
```

The **cost**: O(depth_of_replay) sequential calls to `apply`. For a
block 1000 deep from the finalized tip, this is 1000 application
calls.

The **mitigations**:

- The application implements `apply` quickly (no IO during apply;
  just compute).
- Pruning keeps `pending` small under steady state.
- Recovery happens *only after a crash* or a node that fell far
  behind — not on the hot path.

The `ReplayForward` recovery is **read-once, write-many**: it costs
the actor for one replay, then the resulting `pending` entries
serve future lookups.

## The `SyncPlan` — startup coordination

`glue/src/stateful/actor/sync_plan.rs`. The SyncPlan is the
**startup dance** between Marshal and Stateful:

```rust
pub struct SyncPlan {
    metadata: Metadata,
    floor: Option<Finalization>,
    marshal_start: MarshalStart,
    stateful_start: StatefulStart,
}

pub enum MarshalStart {
    SyncFromPeers,
    SyncFromFloor(Finalization),   // we have local state up to this floor
    NoSyncNeeded,
}

pub enum StatefulStart {
    StartEmpty,
    ContinueFromFloor(Finalization),
}
```

When the validator restarts:

1. **Open metadata**: read atomic state from disk (last known
   finalized block).
2. **Determine sync need**: if our stored height ≥ configured
   `floor`, we don't need state sync.
3. **If sync needed**: Marshal becomes the **syncer**
   (`SyncFromPeers`), pulling blocks from peers. Until sync
   completes, Stateful is **paused** (`StartEmpty`).
4. **If sync not needed**: Marshal starts normal finalization
   consumption. Stateful starts with the on-disk state
   (`ContinueFromFloor`).

The `SyncPlan` also configures the **sync target**: which
operations do we need, where do we start fetching from, what's the
op-count threshold for "we're caught up."

## The state sync flow

When state sync is needed:

```rust
async fn state_sync(&mut self, floor: Finalization) {
    let syncer = StateSyncCoordinator::new(self.database.clone());

    while let Some((block, cert)) = self.finalized_receiver.recv().await {
        if !syncer.is_complete() {
            syncer.observe_block(block, cert);
            if syncer.accepted(block) {
                self.ack_marshal(block.height).await?;
                continue;
            }
            // Not ours; ignore
        } else {
            // Syncer is done; process as normal finalize
            self.handle_finalize(block, cert).await;
        }
    }
}
```

The StateSyncCoordinator uses the `db::p2p::standard` or
`db::p2p::compact` resolver from chapter 08 to fetch operations
from peers. It walks the block from `floor` forward, asking peers
for each batch of operations, verifying against the block's
Merkle root, and applying to the database as it goes. When the
local database's target matches `syncer.target`, sync completes.

The price: while syncing, the validator is **not** participating
in consensus (it's just absorbing blocks). Once caught up, it
enters normal mode.

## The `simulate` harness — every type

`glue/src/simulate/`. The simulator composes a full Commonware stack
as a single testable unit.

### `EngineDefinition`

```rust
pub trait EngineDefinition {
    type Engine: Debug + Send + 'static;
    type Context: Debug + Send + 'static;
    type Validator: Debug + Send + Sync + 'static;
    type PublicKey: PublicKey;

    async fn build(
        &mut self,
        context: Self::Context,
        validator: Self::Validator,
        peers: Vec<Self::PublicKey>,
    ) -> Result<Self::Engine>;
}
```

Implement this to wire your full app (Simplex + Marshal + Glue
Stateful actor + QMDB + P2P) into a single `Engine`. The simulator
calls `build(context, validator, peers)` for each validator at test
startup.

### `Plan`

```rust
pub struct Plan {
    validators: usize,
    duration: Duration,
    actions: Vec<Action>,
    properties: Vec<Property>,
}
```

Declarative test config:

- `validators`: how many to instantiate.
- `duration`: how long to run.
- `actions`: injected mid-test events.
- `properties`: post-test assertions.

### `Action`

```rust
pub enum Action {
    Kill(PublicKey),                       // validator dies
    Partition { left: Vec<PublicKey>, right: Vec<PublicKey> },
    UpdateLink { from: PublicKey, to: PublicKey, link: Link },
    Restart(PublicKey),
    AddValidator(PublicKey),
    RemoveValidator(PublicKey),
}
```

Each Action has a **timestamp at which it fires** (from the
deterministic runtime). The `Plan` schedules them ahead of time;
the simulator executes them in order.

### `Property`

```rust
pub enum Property {
    NoForks,                       // no two validators ever disagreed on history
    Liveness,                      // finalization kept happening through the test
    Faults(Vec<Fault>),            // expected byzantine faults occurred
}
```

After the simulated duration, the runner checks each property. If
any fails, the test fails with a structured error indicating which
property and which validator/key.

### `Team`

```rust
pub struct Team {
    handles: Vec<(Engine, JoinHandle)>,
    network: SimulatedNetwork,
}
```

The orchestrator. Owns the simulated P2P network and the spawned
validator tasks. Provides `add_validator`, `remove_validator`,
`partition`, `inject_latency` as runtime methods (in addition to
the declarative `Plan` actions).

### `ProgressTracker`

```rust
pub struct ProgressTracker {
    target: Height,
    received: BTreeMap<Height, BTreeSet<PublicKey>>,
}
```

A subscriber to the consensus `Reporter` (chapter 11). Every
finalization event increments `received[H].len()`. Test logic
typically:

```
while progress.received.len() < target_height {
    progress.wait_one().await;
}
```

### Final testing flow

```rust
#[test]
fn test_stateful_progress_through_partition() {
    let plan = Plan::new()
        .validators(4)
        .duration(Duration::from_secs(60))
        .action(5, Action::Partition { left: vec![0,1], right: vec![2,3] })
        .action(10, Action::HealPartition)
        .property(Property::NoForks)
        .property(Property::Liveness { finalizations: 50 });

    let simulator = Simulator::new(plan);
    simulator.run().unwrap();
}
```

The simulator starts the deterministic runtime, builds each
validator via `EngineDefinition`, runs the duration, fires actions
at scheduled times, then checks properties. If everything matches,
the test passes.

## Rust pattern — generics over Engine, `impl Trait` returns

Three Rust patterns worth flagging in the Glue chapter.

### Generics over Engine definition

```rust
pub trait EngineDefinition {
    type Engine: Send + 'static;
    // ...
}
```

The simulator is generic over the engine type — it doesn't know
what app is being tested; it just knows there's an `Engine: Send`
and a `Context: Send`. This is the **dependency-inversion** pattern:
the test harness depends on an abstraction, not a concrete type.
Apps implement the abstraction; the harness works with any of them.

The trade-off: the simulator's source code never sees the
application's specific state-transition logic. To test
application-specific properties (e.g., "user X's balance at height
100 is Y"), the application must add its own custom `Property`
variant.

### `impl Trait` in return position

The Application trait uses `impl Future<Output = Self::Block> + Send`
as the return type of `genesis`, `propose`, etc. This avoids having
to name a concrete future type (some Pin<Box<dyn Future>>-style
boxed future) and lets the compiler fully type-check.

The catches:

- Every async trait method that returns `impl Future` produces a
  different concrete type per implementation. Boxing the future is
  the way to erase the type, but that costs an allocation.
- Commonware's async traits use this pattern heavily; in
  practice all call sites await the future directly without boxing.

### `where` clauses vs associated types

```rust
// Associated types only — what the trait wants
trait DatabaseSet<E: Rng + Spawner + Metrics + Clock> {
    type Unmerkleized: Send + 'static;
    type Merkleized: Send + 'static;
    type SyncTargets: Send + Sync + 'static;
}

// Where clauses — additional bounds on associated types
trait Application<E>: Clone + Send + 'static
where
    Self::Context: Epochable + Viewable + Send,
    Self::Block: CertifiableBlock<Context = Self::Context>,
{
    // ...
}
```

Associated types for the **shape** (the return types); where
clauses for the **bounds** (the constraints on those return types).
The two together form the trait's full contract. Implementing the
trait requires satisfying both.

Commonware leans toward associated types over generic methods
because they produce cleaner code: `Self::Merkleized` is clearer
than `M<Self>` where `M` is a generic parameter.

## Where to look in the code (expanded preview)

The appendices that follow go deeper on each topic above. When you
see a `file:line` reference in main body, the appendix walks it
line by line — but the surrounding structure and motivation live
here.

## Speculative execution literature in full

The "fork-and-merge" pattern in this chapter has many names.
This section walks through the academic literature on each
naming — and how they relate to Commonware's design.

### MVCC — Multi-Version Concurrency Control

Database MVCC lets many transactions read and write
concurrently without locking. Each row carries `xmin` (creator
txid) and `xmax` (deleter txid); reads see rows visible to the
read snapshot; updates are copy-on-write.

Postgres's MVCC implementation:

- Every row has `xmin` and `xmax`.
- `SELECT` reads rows where `xmin ≤ snapshot.xmax < xmax` (the
  snapshot's upper bound is < xmax means the deleter tx hasn't
  committed).
- `UPDATE` writes a new row with the same primary key, new
  `xmin`, and an `xmax` on the old row.
- Vacuum / `autovacuum` cleans up dead rows.

The result: readers don't block writers, writers don't block
readers, but two writers contending on the same row still
need locks.

Commonware's Stateful actor is **MVCC on a fork-tree of
blocks.** Each fork is a separate version of the state; the
"winner" is the one Simplex finalizes. The losers get pruned.
This is structurally similar to database MVCC, with the unit
of versioning being a block (containing many transactions) vs
a transaction (containing a single atomic change).

### Fork-join parallelism

The fork-join model (Lea, "A Java Fork/Join Framework," 2000)
is a parallel computing pattern where:

- A "fork" splits a task into sub-tasks that run in parallel.
- A "join" waits for sub-tasks and combines their results.

`fork::JoinHandle`, the parallel computing library, provides
this for Rust. `tokio::task::JoinSet` provides it for async
tasks.

Commonware's actor model is **fork-join within an actor**: the
actor's `propose` / `verify` method can spawn sub-tasks and
await them all. The actor itself is single-threaded (one actor
task, one message at a time); sub-tasks run on Tokio's
multi-threaded runtime.

### SSI — Serializable Snapshot Isolation

Postgres's "Serializable Snapshot Isolation" (SSI, Fekete et
al. 2008 paper) is a refinement of snapshot isolation that
detects serialization conflicts and aborts the conflict-edge
transactions.

SSI watches for **rw-antidependencies**: transaction A reads a
row, transaction B updates that row, B commits. If B's update
makes A's read invalid (a "dangerous structure"), one of them
must abort.

Commonware's design **doesn't need SSI**: the consensus layer
(Simplex) enforces a single total order. There can't be a
cross-block anomaly because every block is finalized one at a
time, in a known order. The "serializability" is at the
protocol level, not the database level.

This is the difference between **database transactions** and
**block transactions**. A database transaction is one atomic
change; a block transaction is many. The "block" is the
unit, and the unit is single-total-ordered by Simplex.

### How they compare

| Approach | Atomic unit | Conflict resolution | Commonware analog |
|---|---|---|---|
| **MVCC** | Row | Vacuum / abort-on-isolation-violation | Stateful (block-level) |
| **Fork-join** | Sub-task | JoinAll / cancel-on-first-error | Per-actor parallelism |
| **SSI** | Transaction | Detect rw-antidependency, abort | Not needed (Simplex) |
| **Speculative execution** (Tomasulo) | Instruction | Rollback on mispredict | `verify` (predict) + drop-on-mismatch |

The "speculative execution" naming in hardware (Tomasulo's
algorithm, 1967) is the closest analogue: the CPU executes
branches speculatively; on mispredict, it rolls back. Commonware
"speculatively executes" pending blocks; on `verify` failure,
the fork is dropped.

### DDIA chapter 7 in detail

Martin Kleppmann's "Designing Data-Intensive Applications"
chapter 7 ("Transactions") is the canonical reference for the
taxonomy above. The key chapter sections:

- **Weak isolation levels:** read committed, snapshot
  isolation, why they don't prevent all anomalies.
- **Serializable isolation:** the strong guarantee, how
  databases achieve it (SSI, two-phase locking, strict
  serializable).
- **What's a transaction:** the ACID properties, why atomicity
  and isolation matter for correctness.

The Commonware analog: **each block is a transaction**. The
"isolation" is enforced by Simplex. The "atomicity" is
enforced by the `merkleize` + `finalize` step (the entire
fork's effects are applied atomically when the block
finalizes).

## The actor model in Glue context

How Commonware's `Stateful` actor uses the actor model — the
full deep dive.

### The mailbox + state split

`Stateful` is an actor with:

- **Mailbox**: messages from Marshal
  (`Propose { parent, ancestry, input, response_tx }`,
  `Verify { parent, ancestry, batches, response_tx }`,
  `Finalize { block, certificate, ack_tx }`).
- **State**: a `BTreeMap<Digest, Merkleized>` of pending
  forks keyed by block digest; a `BTreeMap<Height,
  (Digest, Merkleized)>` of finalized state keyed by height.
- **Database**: an instance of `DatabaseSet<E>` (chapter 17's
  "DatabaseSet" main body).

The mailbox is the **input channel** (Marshal sends). The
state is **private to the actor**. The database is the
**durable side-effect** (it commits to disk when `merkleize`
is called).

### Trade-offs

Pros:

- **Single owner of state.** The pending forks map can't
  race with another task; only the actor's task modifies it.
- **Bounded mailbox.** Marshal never overruns the actor during
  healthy operation; under simulated heavy partition, the
  mailbox backpressures the upstream actor (chapter 15).
- **Mandatory supervision.** The actor is a child of the
  runtime's supervisor; if Simplex's Marshal aborts, the
  Stateful actor aborts.

Cons:

- **Single-threaded state.** The pending forks are managed by
  one task; no parallel writes. For most apps, this is fine
  (the actual state execution happens inside
  `propose`/`verify` futures, which the actor awaits one at a
  time).
- **Restart loses state.** On crash, the pending forks are
  gone. Lazy recovery is the workaround.
- **Mailbox latency.** Every Marshal-Send involves a
  context-switch into the actor's task. For Stateful, this is
  acceptable (Marshal-Sends are infrequent: one per view).

### Why not "DatabaseSet directly"?

You could imagine handling propose/verify without an actor —
just call `db.fork()` and run the application logic inline.
But Marshal's propose/verify semantics require:

- **Sequential processing per parent**: a `verify` for block B
  with parent A must complete before `verify` for B' with
  parent A' (so pending forks don't race).
- **Ordering between propose/verify/finalize**: Marshal's
  flow is "propose → notarize → finalization"; the actor's
  loop maintains this order.
- **Bounded pending forks**: pruning dead forks on finalize.

The actor's `select!` loop does this cleanly. An inline
implementation would need a queue anyway.

## Database connection pooling patterns

The `DatabaseSet<E>` is an abstraction over one or more
databases. This section walks through what pooling and
lifecycle look like in practice.

### Lifecycle

A `DatabaseSet::Fork` is created during `fork` and dropped
when the application finishes. Lifetime:

```
DatabaseSet (durable, on disk)
  |
  | fork(...)
  v
DatabaseSet::Fork (in-memory, mutable)
  |
  | merkleize()
  v
DatabaseSet::Merkleized (hashed, ready to commit)
  |
  | finalize()
  v
DatabaseSet (durable, applied; old fork dropped)
```

Each `Fork` is independent. Multiple Forks can be alive at
once (for speculative execution across parent blocks); each
holds its own in-memory state.

### Pooling

Commonware doesn't pool its forks — they're created and
destroyed freely. Fork creation is cheap (allocating a small
struct; the underlying journal doesn't open new files).

What **is** pooled:

- **Buffer pools** in the underlying journal
  (`journal::variable::Config { buffer_pool, ... }`): the
  `journal` reuses a fixed-size pool of `Bytes` buffers for
  reads and writes. This avoids allocation churn.
- **Read connections** to underlying storage (if the database
  uses a remote store).

### Resource management

The application holds:

- The `DatabaseSet<E>` (one instance per validator).
- The `Mailbox<StatefulMessage>` (the actor's send-side).
- The `JoinHandle` for the actor (so the runtime can wait on it).

When the actor terminates:

- The `Mailbox` clones it held are dropped (so senders see
  `Feedback::Closed`).
- The forks are dropped (in-memory state freed).
- The `DatabaseSet` is dropped (close any open files; flush
  journal buffers).

Commonware uses `Drop` for cleanup. A custom `Drop` impl isn't
needed because the inner types manage their own resources
(`journal::Variable` flushes on drop; `Database` flushes on
drop).

## The full Application trait in depth

Section "The full `Application` trait, dissected" above gives
each method's *why*. This section goes method by method,
through the failure modes and recovery.

### `genesis` — failure and recovery

```rust
fn genesis(&mut self) -> impl Future<Output = Self::Block> + Send;
```

**Failure modes**:

- Implementation returns an error: the actor treats this as a
  fatal start-up failure; the validator doesn't start.
- Implementation panics: same; runtime abort propagates to
  root.
- Implementation returns inconsistent bytes across two runs:
  consensus forks. The Commonware runtime's determinism (chapter
  02) catches this if you're testing under it; otherwise, the
  network fragments at genesis.

**Determinism**: every validator must produce the same bytes
from `genesis()`. This is why the function takes `&mut self`
(deterministic transform of state, even if the state is read-
only) and returns `impl Future + Send` (the future may be
async, but the awaiting happens at a deterministic time).

**Common implementation**: hardcoded block bytes. The genesis
block is constant for the deployment.

### `propose` — failure and recovery

```rust
fn propose(
    &mut self,
    context: (E, Self::Context),
    ancestry: impl Stream<Item = Self::Block> + Send,
    batches: <Self::Databases as DatabaseSet<E>>::Unmerkleized,
    input: &mut Self::InputProvider,
) -> impl Future<Output = Option<Proposed<Self, E>>> + Send;
```

**Failure modes**:

- **Can't fill a block** (empty mempool, e.g.): return
  `None`. The Simplex engine skips this view; the next leader
  takes over.
- **Implementation panics**: actor state is preserved; the
  block isn't proposed; the view skips. The next view may
  succeed.
- **Bug in the application**: invalid block structure. The
  actor logs; the view is wasted.
- **Block production takes too long**: Simplex's view timer
  expires; the actor is asked to participate with a different
  block (or skip). The pending fork is dropped.

**Recovery**: `propose` is best-effort. A failed proposal
means the validator doesn't propose *this* view but is still
alive to vote on others. The pending state for the proposed
block is kept in the actor's `pending` map briefly (in case
the validator becomes a verifier for the same block on a
later view), then pruned.

### `verify` — failure and recovery

```rust
fn verify(
    &mut self,
    context: (E, Self::Context),
    ancestry: impl Stream<Item = Self::Block> + Send,
    batches: <Self::Databases as DatabaseSet<E>>::Unmerkleized,
) -> impl Future<Output = Option<<Self::Databases as DatabaseSet<E>>::Merkleized>> + Send;
```

**Failure modes**:

- **Block is invalid** (bad signature, malformed, double-spend
  the actor can detect): return `None`. The actor doesn't add
  the fork to `pending`. Simplex treats this as "vote against
  the proposal."
- **Block is valid but state transition is broken**: return
  `Some(merkleized)`. The fork is added to `pending`; the
  actor votes for the block. On finalization, the bug becomes
  a chain split — assuming the chain is BFT, 2f+1 honest
  validators reject the bad block.
- **Implementation panics**: actor state preserved; the
  pending map doesn't grow; the validator doesn't vote this
  view. (Honest validators don't lie, but they might just
  fail to vote.)

**Recovery**: `verify` is best-effort. A failed verification
means the validator doesn't vote *this* block on *this* view.
The actor survives.

### `apply` — failure and recovery

```rust
fn apply(
    &mut self,
    context: (E, Self::Context),
    block: &Self::Block,
    batches: <Self::Databases as DatabaseSet<E>>::Unmerkleized,
) -> impl Future<Output = <Self::Databases as DatabaseSet<E>>::Merkleized> + Send;
```

`apply` runs during lazy recovery. The block is known-good
(previously notarized and finalized); the application just
re-executes.

**Failure modes**:

- **Implementation panics**: actor aborts; on restart, the
  crash recovery process is idempotent.
- **Implementation returns state that diverges from what
  `verify` accepted**: this is a determinism bug. The chain
  forks. Commonware's determinism checker should catch this
  in tests, but production deployments are vulnerable.

**Recovery**: by construction, `apply` is a deterministic
replay of a previously-executed block. The bug must be in
either the implementation's logic (and the same bug manifests
during `verify`, leading to a chain split) or in the replay
mechanism.

### `finalized` — failure and recovery

```rust
fn finalized(
    &mut self,
    _context: (E, Self::Context),
    _block: &Self::Block,
    _databases: &Self::Databases,
) -> impl Future<Output = ()> + Send;
```

**Failure modes**:

- **Hook panics**: actor aborts (the default no-op never
  panics). Production hooks should be careful — a panic here
  is post-finalization, which means the block was already
  applied.

**Recovery**: by construction, the block is finalized and
applied; the hook is best-effort. Most apps use this for
metrics or notifications, which shouldn't be critical-path.

### `sync_targets` — failure and recovery

```rust
fn sync_targets(block: &Self::Block) -> <Self::Databases as DatabaseSet<E>>::SyncTargets;
```

**Failure modes**:

- **Implementation panics**: actor aborts; sync coordinator
  doesn't see the targets; sync continues from the next
  finalized block.

**Recovery**: trivial; the next finalized block updates the
sync targets.

## DatabaseSet in depth — typical implementations

Section "`DatabaseSet` — what each database does" above
introduces the trait. This section walks through three
typical implementations.

### Simple application: pure current state

A typical app with one database:

```rust
struct MyDatabases {
    state: qmdb::current::Db,
}

impl DatabaseSet<Engine> for MyDatabases {
    type Unmerkleized = qmdb::current::Fork;
    type Merkleized = qmdb::current::Merkleized;
    type SyncTargets = qmdb::current::SyncTarget;
    type Digest = qmdb::current::Digest;

    fn databases(&self) -> &Self { self }

    fn fork(&self, parents: Self::Unmerkleized) -> Self {
        // parents is actually the database itself for
        // single-database sets; the trait works generically.
        // (The trait's design has parents as `Self::Unmerkleized`
        // for the multi-db case; for single-db, it's the
        // database clone.)
        Self { state: parents }
    }
    // ... merkleize, finalize, sync ...
}
```

For single-database sets, `Unmerkleized = Fork`, and `fork()`
takes the parent fork to produce a child fork.

### Multi-database: state + history log

A more realistic deployment: a mutable state database (for
current accounts) + a keyless log (for historical
transactions).

```rust
struct MyDatabases {
    state: qmdb::current::Db,
    log: qmdb::keyless::Db,
}

impl DatabaseSet<Engine> for MyDatabases {
    type Unmerkleized = (
        qmdb::current::Fork,
        qmdb::keyless::Fork,
    );
    type Merkleized = (
        qmdb::current::Merkleized,
        qmdb::keyless::Merkleized,
    );
    type SyncTargets = (
        qmdb::current::SyncTarget,
        qmdb::keyless::SyncTarget,
    );
    type Digest = (
        qmdb::current::Digest,
        qmdb::keyless::Digest,
    );

    fn databases(&self) -> &Self { self }

    fn fork(&self, parents: Self::Unmerkleized) -> Self {
        Self {
            state: parents.0,
            log: parents.1,
        }
    }
    // ... etc ...
}
```

Each database handles its own logic; the wrapper coordinates
their lifecycles.

### Aggregated databases with shared MMR

For multiple databases that share a single MMR (chapter 16's
QMDB family supports a "shared MMR" variant), you get all the
operations aggregated into one root:

```rust
struct Aggregated {
    inner: qmdb::any::Db,    // contains both state and log under one MMR
}
```

This is the most efficient layout: one MMR, one root, one
proof per query. The trade: less flexibility in individual
databases' lifecycles (they all sync together).

### The fork/merkleize/finalize table

The full lifecycle per database:

| Phase | Per-database action |
|---|---|
| **fork** | Take a snapshot of the parent; produce a mutable fork |
| **merkleize** | Hash the fork's changes; produce a `Merkleized` value ready to commit |
| **finalize** | Apply the merkleized changes to durable storage; update the sync target |

For each phase:

**fork**: the database's `fork() -> Fork` method creates a
copy-on-write view of the current state. Reads see the parent
state; writes go to the new fork.

**merkleize**: the database hashes the fork's changes and
produces a `Merkleized` (a digest + the data needed to apply
the changes). The digest will be the operation's commitment.

**finalize**: the database commits the merkleized changes to
durable storage. After this, the underlying DB has advanced to
the new state.

### Per-database sync targets

Each database exposes a `SyncTarget` — a "where am I" handle
for the syncer:

- `qmdb::current::SyncTarget { operation_count, root }` —
  "I've committed up to operation N with root R."
- `qmdb::keyless::SyncTarget { last_segment }` —
  "I've persisted up to segment S."

The syncer (chapter 17's SyncPlan section) reads each
database's sync target to figure out what data to fetch from
peers.

## Fork-join parallelism in depth

Section "Fork-join parallelism in the spec" above introduces
the concept. This section goes through what
"fork unmerkleized batches" means precisely and the
parallel-processing model.

### What "fork unmerkleized batches" means

In the docstring (`glue/src/stateful/mod.rs:1-17`):

> Before each `propose` or `verify`, the actor forks
> unmerkleized batches from the parent block's pending state
> (or from committed database state if the parent has been
> finalized).

The full reading:

1. **The actor receives a `propose` or `verify` request.**
2. **It looks up the parent block's state** in the `pending`
   map. If hit: that fork's state is the parent. If miss: lazy
   recovery walks backward through the block DAG.
3. **It calls `database.fork(parent_state)`** to produce an
   `Unmerkleized` snapshot. This snapshot is mutable: the
   application can read it and write to it.
4. **It hands the snapshot to `propose` or `verify`.** The
   application executes against the snapshot, modifying it
   in place.
5. **The application returns a `Merkleized` result** (or
   `Option<Merkleized>` for verify). The state is now hashed
   and ready to commit.

### The parallel processing model

Inside `propose` / `verify`, the application can spawn
sub-tasks via `ctx.shared(true).spawn(...)`. This is **fork-
join within the actor**:

```rust
async fn verify(
    &mut self,
    ctx: (E, Context),
    ancestry: impl Stream<Item = Block> + Send,
    mut batches: Databases::Unmerkleized,
) -> Option<Databases::Merkleized> {
    // Sequential sanity checks
    if !self.validate(&batches) { return None; }

    // Parallel speculative execution against many parents
    let mut handles = vec![];
    for parent in self.collect_parents(&ancestry) {
        let my_batches = batches.fork();
        let my_parent = parent;
        let handle = ctx.0.shared(true).spawn(async move {
            (my_batches, my_parent.execute(my_batches).await)
        });
        handles.push(handle);
    }

    // Join
    let results = futures::future::join_all(handles).await;

    // Pick the consistent result
    results.into_iter().find_map(|(b, r)| r)
}
```

This pattern is **rare in practice** (most Simplex apps don't
have multiple parent candidates inside one block), but it's
how fork-join parallelism fits the actor model.

### Determinism requirements

The critical invariant: the application must be **deterministic**
across all validators. If validator A's `verify` returns a
different `Merkleized` than validator B's for the same block,
the BFT consensus fails.

Determinism rules:

- **No wall clock.** Use the runtime's `Clock::now()` (chapter
  02); that's deterministic.
- **No uninitialized memory.** Rust initializes all types by
  default.
- **No system randomness.** Use `Rng` (deterministic-runtime
  gives you a seeded one for tests).
- **No network calls.** The `verify` is local compute. State
  sync is async and separate.
- **No thread-specific behavior.** Tokio's work-stealing can
  produce different scheduling orders; ensure your code is
  order-independent.

Commonware's determinism is not just "we hope" — it's enforced
by the deterministic runtime (chapter 02). Tests run against
the deterministic runtime; production runs against the real
runtime, but the application code is verified to produce the
same output as the deterministic one.

## DDIA transaction models in full

Section "DDIA transaction models — and what Stateful
provides" above gives the high-level comparison. This section
walks through each isolation level and its Commonware analog
in depth.

### Read Committed

The weakest isolation level. A transaction sees only committed
data; no dirty reads. But non-repeatable reads and phantoms are
allowed.

Postgres's default. Most databases default to this.

**Commonware analog**: n/a. There's no "committable partial
state" in Commonware — every block is its own transactional
unit (either final and applied, or non-existent).

### Snapshot Isolation

A transaction sees a consistent snapshot of the database at the
start of the transaction. Reads and writes within the
transaction are isolated from concurrent transactions; only on
commit does the transaction's effect become visible (subject to
first-committer-wins on row-level conflicts).

**Commonware analog**: `propose` and `verify` execute against
the parent block's state. The state is consistent at that
height; the application's fork is a private copy.

The "snapshot isolation" guarantee is automatic: every fork
sees only the parent's state, not concurrent forks.

### Serializable

The strongest traditional isolation. The transaction's behavior
is equivalent to some serial execution of the transactions;
no anomalies.

**Commonware analog**: Simplex enforces total order across
blocks. There's no cross-block anomaly because the consensus
layer guarantees the linearization. The "serializable"
guarantee is at the protocol level, not the database level.

### SSI — Serializable Snapshot Isolation

A refinement that detects serialization failures in snapshot
isolation and aborts the offending transaction. Reduces the
number of false positives.

**Commonware analog**: n/a. SSI's anomaly-detection work isn't
needed because the consensus layer prevents the anomalies.

### What Stateful provides

`Stateful` gives you:

- **Snapshot isolation per fork** — each fork sees the parent
  state, not other forks.
- **Serializable across blocks** — Simplex's total order
  guarantees linearizability.
- **Crash recovery via replay** — pending state is replayable
  from `apply`.

You do **not** get:

- **Cross-fork conflict detection** — forks run independently;
  if two validators have different forks racing, the Simplex
  layer picks one and the other gets pruned.
- **Read-write isolation across forks** — each fork is
  isolated; you can't query other forks.
- **Manual rollback** — you can drop a fork from the pending
  map (e.g., on `verify` failure), but the underlying DB state
  doesn't roll back.

### Pragmatic implications

If your application needs cross-account atomicity (e.g.,
"debit account A and credit account B, or roll both back"),
**Commonware's Stateful doesn't give you transactional
guarantees on this**. You must handle it at the application
level (e.g., "debit succeeds; if credit fails, debit is
recorded as a 'pending reversal' that gets resolved in the
next block").

This is a BFT system, not a distributed database. The trade
is different. Cross-block atomicity is achievable via
multi-block transactions (e.g., "debit in block N, credit in
block N+1 with a tombstone for rollback"), but they're a
design choice, not a default.

## Optimistic concurrency control in depth

Section "Optimistic concurrency control — when to rollback"
above gives the surface. This section walks through the
strategy in detail.

### The optimistic pattern

Optimistic concurrency control (Kung & Robinson, 1981) is a
strategy:

1. **Read** the data (no locks).
2. **Modify** the data locally.
3. **Validate** that no one else's conflicting change has been
  committed.
4. **Commit** if validation passes; otherwise, abort and
  retry.

This is exactly what `verify` does in Commonware:

1. **Read** the parent state (the fork).
2. **Modify** the state via the application's state-transition
   logic.
3. **Validate** that the resulting Merkle root matches what
   the proposer claimed (this is the certificate's check, not
   the `verify`'s).
4. **Commit** by adding to the pending map.

There's no "rollback" in the database sense because the
modification was on a copy (the fork). If validation fails, the
fork is just discarded.

### When to rollback

The rollback cases:

**Verify fails.** The fork doesn't produce the expected root.
Discard.

**Block doesn't finalize.** The fork sits in the pending map
until a sibling is finalized and this one is pruned.

**Block finalizes but state is inconsistent.** This is a
determinism bug in the application. The protocol can't help
here — the bug must be fixed and the chain restarted from a
known-good state.

### Conflict detection

The conflict detection is at the consensus level: two forks
with the same parent can't both be finalized; Simplex's BFT
guarantee ensures only one block at each height is canonical.

In Commonware's model, "conflict" == "fork A and fork B claim
the same parent." Simplex handles this via not-arbitrating-
simultaneously; only one fork gets the quorum.

### Why optimistic

A pessimistic approach ("lock the accounts you're
modifying") doesn't compose with BFT — locking requires a
leader, and the leader is exactly what BFT avoids.

Optimistic concurrency without database rollback is the
right model: speculate, validate later, drop on mismatch. The
"validation later" is block-level consensus; the "drop on
mismatch" is fork pruning.

## The Stateful actor loop in full

Section "The `Stateful` actor's loop, in full" above gave the
surface `select!`. This section walks through every message
type, every state transition, and the cleanup path.

### The full actor struct

```rust
struct Stateful<E: Rng + Spawner + Metrics + Clock, A: Application<E>> {
    application: A,
    pending: BTreeMap<Digest, Merkleized>,
    finalized: BTreeMap<Height, Merkleized>,
    database: A::Databases,
    mailbox: Mailbox<Message>,
    supervisor: Supervisor,
    // Configuration
    config: Config,
    // For lazy recovery:
    ancestry_provider: Arc<dyn AncestryProvider<A::Block>>,
    // For sync:
    sync_coordinator: StateSyncCoordinator<A::Databases>,
}
```

The actor holds:

- The **application** (the trait impl).
- The **pending map** (forks keyed by block digest).
- The **finalized map** (finalized state keyed by height).
- The **database** (the durable storage).
- The **mailbox** (for receiving messages).
- The **supervisor** (for spawning children if needed).
- **Ancestry provider** (used during lazy recovery — gives
  block → parent mapping).
- **Sync coordinator** (used during state sync).

### The message types

```rust
enum Message {
    Propose(ProposeRequest),
    Verify(VerifyRequest),
    Finalize(FinalizeRequest),
    Sync(SyncUpdate),
    StopSignal,
}

struct ProposeRequest {
    context: (E, Context),
    ancestry: BoxedStream<A::Block>,
    response_tx: oneshot::Sender<Option<Proposed<A, E>>>,
}

struct VerifyRequest {
    context: (E, Context),
    ancestry: BoxedStream<A::Block>,
    batches: A::Databases::Unmerkleized,
    response_tx: oneshot::Sender<Option<A::Databases::Merkleized>>,
}

struct FinalizeRequest {
    context: (E, Context),
    block: A::Block,
    certificate: Finalization,
    ack_tx: oneshot::Sender<()>,
}

struct SyncUpdate {
    new_target: A::Databases::SyncTargets,
    ack_tx: oneshot::Sender<()>,
}
```

### State transitions

The actor's state evolves as messages arrive:

```
              +-----------------------+
              |   initial (no state)  |
              +----------+------------+
                         |
                         | first Propose or Verify
                         v
              +-----------------------+
              |   active (handling)   |
              +----------+------------+
                         |
                         | Finalize
                         v
              +-----------------------+
              |   finalized+          |
              |   (some pending drop) |
              +-----------------------+
```

### The cleanup path

When the actor stops (stop signal, parent abort, panic):

1. Drop pending forks (in-memory data freed).
2. Flush database (writes pending operations to disk).
3. Drop the supervisor handle (siblings abort if not already).
4. Return from the run loop.

The database's flush-on-drop is the only I/O on shutdown.
Other cleanup is in-memory.

## Fork-and-merge pattern in depth

Section "The fork-and-merge pattern — code path" above gives
the propose-handler code. This section walks through the full
propose/verify/finalize code paths and the data structures
involved.

### Propose — full data flow

```rust
async fn handle_propose(
    &mut self,
    req: ProposeRequest,
    ctx: &mut Context,
) {
    // Step 1: Resolve the parent state.
    let parent_digest = req.ancestry.parent_digest();
    let parent_state = self.get_or_replay_state(parent_digest).await;

    // Step 2: Fork from parent.
    let batches = self.database.fork(parent_state);

    // Step 3: Run the application logic.
    let proposed = self.application.propose(
        req.context,
        req.ancestry,
        batches,
        &mut self.input_provider,
    ).await;

    // Step 4: If proposed, cache the merkleized state.
    if let Some(p) = proposed {
        self.pending.insert(p.block.digest(), p.merkleized.clone());

        // Step 5: Notify Marshal.
        let _ = req.response_tx.send(Some(p));
    } else {
        // Step 5: Notify Marshal of skip.
        let _ = req.response_tx.send(None);
    }

    // Metrics
    self.metrics.propose_count.inc();
}
```

The data structures involved:

- `req.ancestry: BoxedStream<A::Block>` — the chain backward
  from the parent.
- `batches: A::Databases::Unmerkleized` — the forked state.
- `proposed: Option<Proposed<A, E>>` — the resulting block +
  merkleized state (or `None`).
- `self.pending: BTreeMap<Digest, Merkleized>` — the cached
  pending forks.

### Verify — full data flow

```rust
async fn handle_verify(
    &mut self,
    req: VerifyRequest,
    _ctx: &mut Context,
) {
    let parent_digest = req.ancestry.parent_digest();
    let parent_state = self.get_or_replay_state(parent_digest).await;

    let batches = self.database.fork(parent_state);

    let merkleized = self.application.verify(
        req.context,
        req.ancestry,
        batches,
    ).await;

    if let Some(m) = merkleized {
        self.pending.insert(req.block.digest(), m.clone());
        let _ = req.response_tx.send(Some(m));
    } else {
        let _ = req.response_tx.send(None);
    }
}
```

### Finalize — full data flow

```rust
async fn handle_finalize(
    &mut self,
    req: FinalizeRequest,
    _ctx: &mut Context,
) {
    // Step 1: Get pending state.
    let merkleized = self.pending.remove(&req.block.digest())
        .expect("pending state must exist for finalized block");

    // Step 2: Merkleize (commit).
    self.database.merkleize(merkleized).await?;

    // Step 3: Finalize (apply to underlying storage).
    let sync_targets = self.application.sync_targets(&req.block);
    self.database.finalize(sync_targets).await?;

    // Step 4: Notify application.
    self.application.finalized(
        req.context,
        &req.block,
        &self.database,
    ).await;

    // Step 5: Prune dead forks.
    self.prune_dead_forks(&req.block, &req.certificate);

    // Step 6: Ack to Marshal.
    let _ = req.ack_tx.send(());
}
```

### The data flow

A single block's lifecycle in the actor:

```
Propose/Verify   Proposed     Finalize       Application
    |              |              |              |
    v              v              v              v
+-----------+  +----------+  +----------+  +----------+
| Fork      |->| Merkle-  |->| Commit   |->| Hook     |
| execution |  | ized     |  | + sync   |  | (opt.)   |
+-----------+  +----------+  +----------+  +----------+
       \         |                |              /
        \________|________________|_____________/
                  v
            BTreeMap<Digest, Merkleized>
              (pending + finalized)
```

## Finalization flow in depth

Section "The finalization flow — full sequence" above gives
the high-level. This section walks through every step,
including error paths and what triggers what.

### The full sequence diagram

```
Time ─────────────────────────────────────────────────────>
     Marshal               Stateful              Database
       |                     |                      |
       | Finalize{block=X}   |                      |
       +-------------------->|                      |
       |                     |                      |
       |                     | 1. Lookup pending[X] |
       |                     |    (or replay)       |
       |                     |                      |
       |                     | 2. merkleize(M) ---->|
       |                     |                      |
       |                     | 3. finalize(target) >|
       |                     |                      |
       |                     | 4. application       |
       |                     |    .finalized()      |
       |                     |                      |
       |                     | 5. prune_dead_forks  |
       |                     |                      |
       | ack_tx <------------+                      |
       |                     |                      |
```

### Every step

Step 1: Lookup or replay.

The actor looks up `block.digest()` in `self.pending`. If
found: this is the merkleized state. If not: lazy recovery —
walk the DAG backward to a known ancestor, replay forward.

Step 2: `merkleize`.

The actor calls `self.database.merkleize(merkleized)`. This is
the **commit point**. The merkleized state is committed to the
underlying QMDB (or whatever `DatabaseSet` is being used). The
QMDB returns a digest, which is the operation's commitment for
the consensus certificate.

Step 3: `finalize`.

The actor calls `self.database.finalize(sync_targets)`. This
updates the database's "I'm committed up to here" record. For
QMDB `current`, this updates the operation count; for `keyless`,
the segment count.

Step 4: `application.finalized()`.

Hook. Default no-op; production overrides for metrics,
notifications, etc.

Step 5: `prune_dead_forks`.

The actor walks the pending map, removing entries whose blocks
are not descendants of the freshly-finalized block. This
keeps the in-memory state bounded.

Step 6: Ack to Marshal.

Send `()` on the ack oneshot.

### What triggers what

The finalization flow is triggered by Simplex's `Finalize`
event (chapter 11). The Marshal actor translates this into a
`Finalize` message to the Stateful actor. The Stateful
actor's `handle_finalize` runs the commit protocol.

### Error paths

If step 2 (merkleize) fails (e.g., database disk error), the
actor:

- Logs the error.
- Sends an error on the ack oneshot (Marshal knows not to
  expect success).
- Continues running (the error is recoverable — next
  finalization can succeed).

If step 3 (finalize) fails (e.g., sync target update error),
similar.

If step 4 (application hook) panics, the actor aborts. This
is intentional: a panicking hook is a bug.

If step 5 (prune) fails (e.g., out of memory), the actor
aborts. Same.

If step 6 (ack) fails (the receiver dropped), no big deal —
Marshal wasn't listening anyway.

## Lazy recovery in depth

Section "Lazy recovery, in detail" above gives the
`get_or_replay_state` code. This section walks through the
cost, mitigations, and when it triggers.

### The full code path

```rust
async fn get_or_replay_state(
    &mut self,
    target: Block,
) -> Result<Databases::Merkleized, Error> {
    // Step 1: Check pending map.
    if let Some(state) = self.pending.get(&target.digest()) {
        return Ok(state.clone());
    }

    // Step 2: Check finalized map.
    if let Some(state) = self.finalized.get(&target.height()) {
        return Ok(state.clone());
    }

    // Step 3: Walk backward through the DAG.
    let mut current = target;
    let mut blocks_to_replay = VecDeque::new();
    loop {
        if self.has_known_state(&current) {
            break;
        }
        let parent = self.ancestry_provider.parent(&current).await?;
        blocks_to_replay.push_front(current);
        current = parent;
    }

    // Step 4: Replay forward.
    let mut state = self.get_known_state(&current)
        .expect("must have known state for the ancestor");
    for block in blocks_to_replay {
        let batches = self.database.fork(state);
        state = self.application.apply(
            self.context.clone(),
            &block,
            batches,
        ).await;
        self.pending.insert(block.digest(), state.clone());
    }

    Ok(state)
}
```

### The cost

The cost of lazy recovery:

- One `parent()` call per missing block (network/disk I/O for
  the block's metadata).
- One `apply()` call per missing block (the application's
  state transition).
- For B blocks of replay, the cost is O(B).

For B = 100 (a node that fell 100 blocks behind), the
recovery is 100 `apply()` calls. For an application with
10 ms per `apply`, that's 1 second of recovery time. The actor
is blocked for that second — Marshal sees backpressure.

### Mitigations

**Mitigation 1: fast apply**.

Make `apply` fast. Best practices:

- No I/O during apply. The batch is fetched once during
  finalization; subsequent applies use the cached batch.
- No state-dependent async (wall clock, network, etc.) during
  apply.
- Pre-allocate output data structures.

**Mitigation 2: smaller recovery granularity**.

Walk the DAG in chunks: replay 10 blocks at a time, yield to
the runtime, continue. This avoids blocking the actor for the
full recovery.

**Mitigation 3: state sync instead of apply**.

For a node that's far behind, state sync (chapter 17's "The
state sync flow" main body) is more efficient than replay.
State sync fetches the database's actual content via the P2P
resolver; the node doesn't have to execute every block.

The decision threshold: typically, if `B > threshold` (say,
100 blocks), use state sync; otherwise, use `apply`-based
replay.

### When it triggers

Lazy recovery triggers on:

1. **Cold start** — the node has no in-memory pending state.
   First Propose/Verify receives from Marshal after a restart.
2. **Catch-up after crash** — node restarts; pending map is
   empty; new messages need state for parent blocks that no
   longer have a pending fork.
3. **Far-behind node** — node was offline; pending map's
   ancestor is far behind the latest finalized block.
4. **Bug** — pending map was pruned incorrectly; next
   Propose/Verify needs to replay.

### The "read-once, write-many" property

After a lazy recovery, the replayed forks are inserted into
`pending`. Future lookups for those forks are O(1). The cost
of recovery is amortized: 1 pay-once, N lookups after.

This is the property that makes "lazy" recovery efficient. A
single replay pays for itself via many subsequent lookups.

## SyncPlan in depth

Section "The `SyncPlan` — startup coordination" above
introduces the structure. This section walks through it in
detail, including every configuration option.

### The full structure

```rust
pub struct SyncPlan {
    metadata: Metadata,
    floor: Option<Finalization>,
    marshal_start: MarshalStart,
    stateful_start: StatefulStart,
}

pub enum MarshalStart {
    SyncFromPeers,                    // we need state, marshal pulls it
    SyncFromFloor(Finalization),      // we have local state up to here
    NoSyncNeeded,                     // we are caught up
}

pub enum StatefulStart {
    StartEmpty,                       // no in-memory pending
    ContinueFromFloor(Finalization),  // resume from on-disk state
}
```

### Configuration options

The `SyncPlan` is computed from:

- **`metadata`**: the atomic state on disk (chapter 18's
  metadata format). Contains the last known finalized block.
- **`floor`**: the configured starting floor (the height from
  which the deployment expects to start).
- **`config`**: the validator's config (threshold,
  participants, etc.).

### The startup flow

```rust
async fn startup(
    config: Config,
    metadata: Metadata,
    floor: Option<Finalization>,
) -> Result<SyncPlan> {
    // Step 1: Read atomic state.
    let stored_height = metadata.last_finalized_height();

    // Step 2: Determine whether we need state sync.
    let needs_sync = match &floor {
        None => true,                       // no floor = no state = sync
        Some(f) if stored_height < f.height => true,  // behind floor
        Some(_) => false,                   // caught up
    };

    // Step 3: Configure Marshal.
    let marshal_start = if needs_sync {
        MarshalStart::SyncFromPeers
    } else if let Some(f) = &floor {
        MarshalStart::SyncFromFloor(f.clone())
    } else {
        MarshalStart::NoSyncNeeded
    };

    // Step 4: Configure Stateful.
    let stateful_start = if needs_sync {
        StatefulStart::StartEmpty
    } else {
        StatefulStart::ContinueFromFloor(
            metadata.last_finalized()
                .expect("metadata has a finalization")
        )
    };

    Ok(SyncPlan { metadata, floor, marshal_start, stateful_start })
}
```

### The variations

Four startup scenarios:

1. **First-time startup, configured floor.**
   `floor = Some(F)`, `stored_height = 0`. Needs sync;
   Marshal pulls from peers; Stateful starts empty.

2. **First-time startup, no floor.**
   `floor = None`, no `metadata`. Doesn't really make sense;
   the deployment must define a floor.

3. **Restart at or above floor.**
   `stored_height ≥ floor.height`. No sync needed; Marshal
   resumes; Stateful continues from disk.

4. **Restart below floor** (e.g., manual rollback).
   `stored_height < floor.height`. Needs sync; same as case 1.

### The flow during sync

When `MarshalStart::SyncFromPeers`:

1. Marshal becomes the **state syncer**. It pulls blocks from
   peers (chapter 17's "State sync flow" main body).
2. Stateful is told to start empty (`StatefulStart::StartEmpty`).
3. As Marshal receives blocks, it forwards them to Stateful
   as `Finalize` messages (so Stateful can build its pending
   map).
4. When the local database's target matches the syncer's
   target, sync completes. Marshal returns to normal mode.
5. Stateful's pending map is now populated; subsequent
   Proposes / Verifies work normally.

### Edge cases

**Floor moved backward (deployment rollback).** The new floor
is below the current stored height; we'd need to roll back
state. The `SyncPlan::should_state_sync()` returns true, and
the actor treats stored state as suspect. Recovery involves
state sync from peers up to the new floor.

**Floor moved forward (deployment upgrade).** New floor is
above stored height; we have state up to current, and we want
state further. Marshal resumes; Stateful continues. State sync
isn't needed (we're already past the new floor's base).

**No metadata on disk (truly first run).** `floor` is the
starting point; Marshal syncs from peers; Stateful builds
pending from scratch.

## State sync flow in depth

Section "The state sync flow" above gives the surface. This
section walks through when it runs, the protocol, the
trade-offs vs block sync, and the security considerations.

### When it runs

State sync runs:

1. **First-time startup.** No local state; need to fetch.
2. **Manual recovery.** Operator decides "I don't trust local
   state; rebuild from peers."
3. **Floor-rollback.** New deployment floor is below stored
   height; old state is suspect.
4. **After a significant outage.** Node was offline for hours;
   replay of every block is too slow; fetch the database state
   directly.

### The protocol

The state sync protocol:

1. **Initialize**: the actor creates a `StateSyncCoordinator`
   with the configured floor and its empty local database.
2. **Subscribe to finalizations**: the coordinator listens
   for finalized blocks.
3. **For each finalized block**:
   - **Determine which batches we need.** If the block's
     batch range is beyond our local target, we need them.
   - **Fetch the batches** via the `db::p2p::standard` or
     `db::p2p::compact` resolver.
   - **Verify** the batches against the block's Merkle root
     (chapter 16).
   - **Apply** to the local database.
   - **Update** the syncer's target.
4. **Sync complete when target reached.** The coordinator
   signals the actor to switch to normal mode.

### Block sync vs state sync

Two ways for a node to catch up:

**Block sync** — process every block from the floor up.
Execute every transaction, build the database locally.

- **Cost**: O(B × T) where B is blocks behind, T is per-block
  cost.
- **When**: small B (say < 100).

**State sync** — fetch the database's contents from peers
directly.

- **Cost**: O(S) where S is the database size.
- **When**: large B (say > 100).

Block sync is bounded by the application's per-block cost;
state sync is bounded by the network bandwidth.

### Trade-offs

**Block sync**:

- Pros: every block is verified individually; you participate
  in consensus during sync (you can vote, etc.).
- Cons: slow for large B; bounded by your CPU.

**State sync**:

- Pros: fast for large B; you only verify Merkle roots, not
  every transaction.
- Cons: you don't participate in consensus during sync;
  you're a "syncing" node.

For Commonware, the choice depends on:

- B = 10: block sync is fine. Use the existing propose/verify
  path; sync completes as side-effect.
- B = 1000: state sync. Use the dedicated `db::p2p` resolver.
- B = 10000: state sync is mandatory; block sync would
  take too long.

### Security

State sync fetches data from peers; the data must be
verified. The verifier checks:

- **Merkle proofs** for each batch against the block's
  Merkle root (chapter 16).
- **Certificate validity** for each block's finalization
  (chapter 11, Simplex).
- **Byzantine detection** if a peer returns inconsistent
  data — switch to a different peer.

Commonware's `db::p2p::standard` resolver (chapter 08)
handles the network protocol; the verification logic is part
of the database trait.

## `simulate` harness in full

Section "The `simulate` harness — every type" above gives the
surface. This section walks through every type in detail,
including how they fit together.

### The full type hierarchy

```
       Plan
       |
       +-- validators: usize
       +-- duration: Duration
       +-- actions: Vec<Action>
       +-- properties: Vec<Property>
       |
       v
   Simulator (created from Plan)
       |
       +-- Team (orchestrator)
       |    |
       |    +-- validators: Vec<Engine>
       |    +-- network: SimulatedNetwork
       |    +-- progress: ProgressTracker
       |
       +-- Engines (built via EngineDefinition)
       |    |
       |    +-- pub trait EngineDefinition
       |         |
       |         +-- type Engine
       |         +-- type Context
       |         +-- type Validator
       |         +-- type PublicKey
       |         +-- fn build() -> Engine
       |
       v
   (Test runs actions at scheduled times)
       |
       v
   (Properties checked at end)
```

### `EngineDefinition`

```rust
pub trait EngineDefinition: Send + Sync + 'static {
    type Engine: Debug + Send + 'static;
    type Context: Debug + Send + 'static;
    type Validator: Debug + Send + Sync + 'static;
    type PublicKey: PublicKey;

    async fn build(
        &mut self,
        context: Self::Context,
        validator: Self::Validator,
        peers: Vec<Self::PublicKey>,
    ) -> Result<Self::Engine>;
}
```

The `EngineDefinition` trait wires the full app into a single
`Engine`. Implementations:

```rust
struct MyApp {
    config: MyConfig,
}

impl EngineDefinition for MyApp {
    type Engine = MyEngine;
    type Context = MyContext;
    type Validator = MyValidatorConfig;
    type PublicKey = MyPublicKey;

    async fn build(
        &mut self,
        context: Self::Context,
        validator: Self::Validator,
        peers: Vec<Self::PublicKey>,
    ) -> Result<Self::Engine> {
        // Build the full stack: P2P + Simplex + Marshal +
        // Stateful actor + QMDB + etc.
        MyEngine::start(self.config.clone(), context, validator, peers).await
    }
}
```

The simulator calls `build()` once per validator during
startup.

### `Plan`

```rust
pub struct Plan {
    pub validators: usize,
    pub duration: Duration,
    pub actions: Vec<ScheduledAction>,
    pub properties: Vec<Property>,
}

pub struct ScheduledAction {
    pub time: Time,
    pub action: Action,
}
```

A `Plan` is the declarative test config.

Builder methods:

```rust
let plan = Plan::new()
    .validators(4)
    .duration(Duration::from_secs(60))
    .action(Time::from_secs(5),
            Action::Partition {
                left: vec![pkey0, pkey1],
                right: vec![pkey2, pkey3],
            })
    .action(Time::from_secs(10), Action::HealPartition)
    .property(Property::NoForks)
    .property(Property::Liveness { finalizations: 50 })
    .build();
```

### `Action`

```rust
pub enum Action {
    Kill(PublicKey),
    Partition { left: Vec<PublicKey>, right: Vec<PublicKey> },
    UpdateLink { from: PublicKey, to: PublicKey, link: Link },
    Restart(PublicKey),
    AddValidator(PublicKey),
    RemoveValidator(PublicKey),
    InjectLatency(PublicKey, PublicKey, Duration),  // from, to, latency
}
```

Each action has a timestamp at which it fires. The simulator's
event loop checks: "is it time for the next action?" and if
so, executes it.

`Kill` and `Restart` operate on the validator's `JoinHandle`.
`Partition` modifies the simulated network.
`InjectLatency` affects message delivery between two specific
peers.

### `Property`

```rust
pub enum Property {
    NoForks,                       // no two validators ever disagreed on history
    Liveness,                      // finalization kept happening through the test
    Faults(Vec<Fault>),            // expected byzantine faults occurred
    Custom(Box<dyn CustomProperty>),  // user-defined
}

pub trait CustomProperty: Send + Sync {
    fn check(&self, team: &Team) -> Result<()>;
}
```

Properties are checked after the simulated duration. Failed
properties produce structured errors with details (which
validators, which heights).

### `Team`

```rust
pub struct Team {
    engines: Vec<Engine>,
    handles: Vec<JoinHandle>,
    network: SimulatedNetwork,
    progress: ProgressTracker,
    // ...
}

impl Team {
    pub fn validators(&self) -> &[Engine];
    pub fn network(&mut self) -> &mut SimulatedNetwork;
    pub fn progress(&self) -> &ProgressTracker;
    pub async fn shutdown(self) -> Result<()>;
}
```

The `Team` owns the running validators and the simulated
network. The simulator uses `Team` to:

- Drive each engine with the current time.
- Fire scheduled actions.
- Observe finalization progress.

### `ProgressTracker`

```rust
pub struct ProgressTracker {
    target: Height,
    received: BTreeMap<Height, BTreeSet<PublicKey>>,
    reporter: Receiver<ConsensusEvent>,
}

impl ProgressTracker {
    pub async fn wait_for_height(&self, h: Height) -> &BTreeSet<PublicKey>;
    pub fn total_finalized(&self) -> Height;
}
```

The `ProgressTracker` subscribes to a consensus `Reporter`
(chapter 11). Every finalization event adds to `received`. A
test can wait for a specific height to be finalized, or assert
finalization kept happening throughout the test.

### Final usage

```rust
#[test]
fn test_through_partition() {
    let plan = Plan::new()
        .validators(4)
        .duration(Duration::from_secs(60))
        .action(t(5), Action::Partition { left: vec![0, 1], right: vec![2, 3] })
        .action(t(15), Action::HealPartition)
        .property(Property::Liveness { finalizations: 50 })
        .property(Property::NoForks)
        .build();

    let mut simulator = Simulator::new(plan, MyApp::new(CommonwareConfig::default()));
    simulator.run().unwrap();
}
```

The simulator:

1. Creates the deterministic runtime (chapter 02).
2. Calls `MyApp::build()` for each of the 4 validators.
3. Wires them into the simulated P2P network.
4. Runs for 60s of simulated time, firing actions at t=5 and
   t=15.
5. At the end, checks: did 50+ blocks finalize? did any two
   validators disagree on history?
6. If both pass, returns `Ok(())`. Otherwise, returns an error.

## Exercises

These exercises build a Commonware-style stateful app from
scratch. They simulate the fork-and-merge pattern in
isolation.

### Exercise 1 — Implement `DatabaseSet` for a `Vec<u8>` map

Without using `qmdb`, write a `DatabaseSet` for an in-memory
key-value map.

```rust
struct InMemoryDB { map: HashMap<Vec<u8>, Vec<u8>> }

impl DatabaseSet<MockEngine> for InMemoryDB {
    type Unmerkleized = InMemoryFork;
    type Merkleized = InMemoryMerkleized;
    type SyncTargets = ();
    type Digest = Digest;

    // ... fork / merkleize / finalize ...
}
```

The fork is a copy-on-write view of the map. Merkleize hashes
the map's contents (use a deterministic hash). Finalize
applies the merkleized state to the durable map.

Tests:

- Insert `(b"alice", b"100")` into a fork.
- Merkleize and finalize; assert the map has the entry.

### Exercise 2 — Add fork-and-merge

Write an actor that handles `propose`/`verify`/`finalize`
messages:

- `propose`: fork from parent, insert `(b"key", b"value")`,
  merkleize, return `(block, merkleized)`.
- `verify`: fork from parent, verify the proposed batch is
  valid, merkleize, return `merkleized`.
- `finalize`: apply the merkleized to the database, prune
  dead forks.

Tests:

- Send a propose; verify the actor returns a block.
- Send a verify with a valid batch; verify returns
  `Some(m)`.
- Send a finalize for the proposed block; database has the
  entry.

### Exercise 3 — Implement lazy recovery

Extend Exercise 2's actor with `get_or_replay_state`: if the
parent isn't in the pending map, replay from a known ancestor.

Tests:

- Pre-populate the pending map with block 1.
- Send a propose for block 3 (child of 2); actor should
  replay block 2 from the pending map.

### Exercise 4 — Build a simulated network

Write a `SimulatedNetwork` that:

- Holds a map `peers → mailbox sender`.
- `send(from, to, msg)`: routes the message to the right
  receiver; introduces latency from a configurable profile.
- `partition(left, right)`: blocks messages between the
  groups.

Tests:

- 4 validators; `send` returns each peer their messages.
- After `partition({0,1}, {2,3})`, no message from
  validator 0 reaches validator 2.

### Exercise 5 — Implement the simulate harness

Without using `glue::simulate`, write a harness that:

- Takes a `Plan`.
- Spawns the actors from Exercise 2 on the simulated network.
- Runs the test for the configured `duration`.
- At the end, checks the configured `properties`.

Tests:

- 4 validators; `duration = 1 sec`; assert all 4 reach height
  5 with no forks.

→ Next: **Chapter 18 — Deployer**. The "ship it to AWS" tooling.

## Appendix A — The `Application` trait, fully dissected

`glue/src/stateful/mod.rs:125-285`. The trait has six methods; let me walk
through each.

### `sync_targets`

```rust
fn sync_targets(block: &Self::Block) -> <Self::Databases as DatabaseSet<E>>::SyncTargets;
```

Given a finalized block, extract the **per-database sync targets** for
state sync. Each database in the set has its own sync target (e.g.,
the latest operation index, the latest committed root).

The wrapper uses this during state sync to update the syncing
coordinator.

### `genesis`

```rust
fn genesis(&mut self) -> impl Future<Output = Self::Block> + Send;
```

Build the **genesis block** for the first epoch. Called once at startup.
The returned block becomes the floor of consensus.

### `propose`

```rust
fn propose(
    &mut self,
    context: (E, Self::Context),
    ancestry: impl Stream<Item = Self::Block> + Send,
    batches: <Self::Databases as DatabaseSet<E>>::Unmerkleized,
    input: &mut Self::InputProvider,
) -> impl Future<Output = Option<Proposed<Self, E>>> + Send;
```

Build a new block on top of the provided parent ancestry.

Parameters:
- `context`: the runtime + consensus context (view, height, etc.).
- `ancestry`: stream of parent blocks (the chain backward).
- `batches`: per-database unmerkleized batches (you execute against these).
- `input`: your input source (mempool, oracle feed, etc.).

Returns `Proposed { block, merkleized }` or `None` (if you can't propose).

The `batches` parameter is **forked from the parent**. Each database
gives you a `DatabaseSet::Unmerkleized` view — a snapshot that hasn't
been hashed yet. You execute transactions, getting a `DatabaseSet::Merkleized`
view (hashed and ready to commit).

### `verify`

```rust
fn verify(
    &mut self,
    context: (E, Self::Context),
    ancestry: impl Stream<Item = Self::Block> + Send,
    batches: <Self::Databases as DatabaseSet<E>>::Unmerkleized,
) -> impl Future<Output = Option<<Self::Databases as DatabaseSet<E>>::Merkleized>> + Send;
```

Verify a block from a peer. Same structure as `propose`, but:
- No `input` (you didn't initiate this block).
- Returns `Some(merkleized)` if valid, `None` if permanently invalid.

If you can't decide yet (e.g., missing data), keep the future pending.

### `apply`

```rust
fn apply(
    &mut self,
    context: (E, Self::Context),
    block: &Self::Block,
    batches: <Self::Databases as DatabaseSet<E>>::Unmerkleized,
) -> impl Future<Output = <Self::Databases as DatabaseSet<E>>::Merkleized> + Send;
```

Apply a **previously certified** block (during lazy recovery). The block
is known-good (was previously notarized), so you execute unconditionally.

The returned merkleized state must match what `verify` accepted for the
same block. The wrapper can't re-check block-specific commitments
generically.

### `finalized`

```rust
fn finalized(
    &mut self,
    _context: (E, Self::Context),
    _block: &Self::Block,
    _databases: &Self::Databases,
) -> impl Future<Output = ()> + Send {
    async {}
}
```

Hook for "this block is finalized and applied to disk." Default impl is
no-op. Override for post-finalization work (metrics, observability,
external sync, etc.).

## Appendix B — The `DatabaseSet` trait

`glue/src/stateful/db/`. A `DatabaseSet` is a group of one or more databases
managed together:

```rust
pub trait DatabaseSet<E: Rng + Spawner + Metrics + Clock>: Clone + Send + Sync + 'static {
    type Unmerkleized: Send + 'static;
    type Merkleized: Send + 'static;
    type SyncTargets: Send + Sync + 'static;
    type Digest: Digest;

    fn databases(&self) -> &Self;

    fn fork(&self, unmerkleized: Self::Unmerkleized) -> Self;
    fn merkleize(&mut self, merkleized: Self::Merkleized) -> Result<()>;
    fn finalize(&mut self, sync_targets: Self::SyncTargets) -> Result<()>;

    // Sync operations
    async fn sync(&mut self, ...) -> Result<()>;
    // ...
}
```

A typical `DatabaseSet` has:
- A `current` QMDB (the active state).
- An `immutable` archive (finalized, immutable operations).

When you fork: take the current state + apply unmerkleized changes → new
working copy. When you merkleize: hash the changes → ready to commit.
When you finalize: apply to disk + update sync targets.

## Appendix C — The `Stateful` actor's loop

`glue/src/stateful/actor/`. The actor handles:

```rust
struct Stateful<E, A: Application<E>> {
    application: A,
    pending: BTreeMap<Digest, MerkleizedState>,  // in-memory speculative state
    finalized: BTreeMap<Height, (Digest, MerkleizedState)>,
    database: A::Databases,
    // ...
}

async fn run(mut self) {
    loop {
        select! {
            // Marshal: propose request
            req = self.propose_receiver.recv() => {
                self.handle_propose(req).await;
            }

            // Marshal: verify request
            req = self.verify_receiver.recv() => {
                self.handle_verify(req).await;
            }

            // Marshal: finalize notification
            notif = self.finalize_receiver.recv() => {
                self.handle_finalize(notif).await;
            }

            // Sync: state sync update
            sync = self.sync_receiver.recv() => {
                self.handle_sync(sync).await;
            }

            _ = self.stop_signal => break,
        }
    }
}
```

The actor owns the `Databases` (the on-disk state) and `pending` (the
in-memory speculative state).

## Appendix D — The fork-and-merge pattern

When you `propose`, you execute against a **forked** state:

```rust
async fn handle_propose(&mut self, context: Context, ancestry: Stream, input: Input) {
    // Get the batches (fork from parent)
    let parent_state = self.get_or_replay(ancestry).await;
    let batches = self.database.fork(parent_state);

    // Run propose (the application's logic)
    let proposed = self.application.propose(context, ancestry, batches, input).await;

    if let Some(p) = proposed {
        // Cache the merkleized state under the block digest
        self.pending.insert(p.block.digest(), p.merkleized);
        // ... notify Marshal that propose is done ...
    }
}
```

Multiple forks in memory. Each fork is keyed by its block's digest.

## Appendix E — The finalization flow

When Marshal says "block X is finalized":

```rust
async fn handle_finalize(&mut self, block: Block, finalization: Finalization) {
    // 1. Find the pending state for this block
    let merkleized = self.pending.remove(&block.digest())
        .expect("pending state must exist for finalized block");

    // 2. Apply to the underlying databases
    self.database.merkleize(merkleized).await?;
    self.database.finalize(sync_targets).await?;

    // 3. Notify application (for post-finalization work)
    self.application.finalized(context, &block, &self.database).await;

    // 4. Prune dead forks
    self.prune_dead_forks(&block, finalization).await;

    // 5. Acknowledge to Marshal
    self.marshal_ack_tx.send(block.height).await?;
}
```

This is the **commit point**. After this, the block's effects are
durably applied.

## Appendix F — The lazy recovery

If the node restarts, pending state is gone. On receiving a new
propose/verify request, we walk the block DAG backward to find a known
ancestor, then replay forward:

```rust
async fn replay_forward(&mut self, target_block: Block) -> Result<()> {
    // Walk backward from target_block until we hit a block we've applied
    let mut blocks_to_replay = VecDeque::new();
    let mut current = target_block;
    loop {
        if self.finalized.contains_key(&current.parent_height()) {
            break;  // Found a known ancestor
        }
        blocks_to_replay.push_front(current);
        current = self.get_parent(current).await?;
    }

    // Replay forward
    for block in blocks_to_replay {
        let batches = self.database.fork(self.get_state_at(block.parent_height())?);
        let merkleized = self.application.apply(context, &block, batches).await;
        self.pending.insert(block.digest(), merkleized);
    }
}
```

This is expensive but rare. Happens only after a crash or when the node
falls far behind.

## Appendix G — The SyncPlan

`glue/src/stateful/actor/sync_plan.rs`. The `SyncPlan` coordinates
startup between Marshal and Stateful:

```rust
pub struct SyncPlan {
    metadata: Metadata,           // atomic metadata handle
    floor: Option<Finalization>,  // configured starting floor
    marshal_start: ...,           // what Marshal should do at startup
    stateful_start: ...,          // what Stateful should do at startup
}
```

When the node starts:

1. Open the metadata (atomic state).
2. Decide: do we need state sync? (Compare our stored complete height
   with the configured floor.)
3. Configure Marshal and Stateful accordingly.

`SyncPlan::should_state_sync()` returns true if we need to sync from
peers (typically because we're starting fresh or from a stale height).

## Appendix H — The state sync flow

When state sync is needed:

```rust
async fn state_sync(&mut self, floor: Finalization) {
    // Initialize sync coordinator with the floor
    let syncer = StateSyncCoordinator::new(self.database.clone());

    // For each finalized block we observe, check if the syncer wants it
    while let Some((block, cert)) = self.finalized_receiver.recv().await {
        if !syncer.is_complete() {
            syncer.observe_block(block, cert);
            if syncer.accepted(block) {
                self.ack_marshal(block.height).await?;
                continue;
            }
        }

        // Syncer is done — process normally
        self.handle_finalize(block, cert).await;
    }
}
```

State sync fetches ops from peers via the `db::p2p::standard` or
`db::p2p::compact` resolver (chapter 08).

## Appendix I — The `simulate` harness, in detail

`glue/src/simulate/`. The `EngineDefinition` trait:

```rust
pub trait EngineDefinition {
    type Engine: Debug + Send + 'static;
    type Context: Debug + Send + 'static;
    type Validator: Debug + Send + Sync + 'static;
    type PublicKey: PublicKey;

    async fn build(
        &mut self,
        context: Self::Context,
        validator: Self::Validator,
        peers: Vec<Self::PublicKey>,
    ) -> Result<Self::Engine>;
}
```

Implement this to wire up your app's stack. The simulator calls `build`
for each validator in the test.

The `Plan`:

```rust
pub struct Plan {
    validators: usize,
    duration: Duration,
    actions: Vec<Action>,
    properties: Vec<Property>,
}
```

`Action`s are injected mid-test (kill validator, partition network,
inject latency, etc.). `Property`s are assertions checked at the end.

The `Team` orchestrates the running validators:

```rust
pub struct Team {
    handles: Vec<(Engine, JoinHandle)>,
    network: SimulatedNetwork,
    // ...
}
```

The `ProgressTracker` watches for finalization progress:

```rust
pub struct ProgressTracker {
    target: Height,
    received: BTreeMap<Height, Set<PublicKey>>,  // who finalized what height
}
```

When `received.len() == target`, the test knows all validators are
caught up.

## Appendix J — Test patterns

```rust
fn test_stateful_finalize() {
    // Setup: 4 validators, each with a Stateful app
    // Build and notarize 10 blocks
    // Verify: each validator applies all 10 blocks in order
    // Verify: each validator's database has the final state
    // Verify: each validator's finalized map has all 10 entries
}

fn test_stateful_fork() {
    // Setup: 4 validators
    // Build fork A: blocks at heights 1-5a
    // Build fork B: blocks at heights 1-5b (diverges from height 3)
    // Verify: each validator maintains both forks in pending state
    // Finalize fork A
    // Verify: validators that finalized A have applied 1-5a
    // Verify: fork B is pruned (no longer in pending)
}

fn test_stateful_crash_recovery() {
    // Setup: 4 validators
    // Build blocks 1-10, all finalized
    // Validator A crashes
    // Validator A restarts
    // Validator A sees a new block being proposed at height 11
    // Verify: A replays 1-10 from disk (fast — no fork needed)
    // Verify: A's database matches the other validators
}
```

## Appendix K — Common gotchas

### Not implementing `apply` correctly

`apply` runs during lazy recovery. If your `apply` returns different
state than `verify` accepted for the same block, you'll have inconsistent
state across validators.

The wrapper checks `propose`/`verify` against `apply` results via the
sync_targets comparison, but the application is responsible for
determinism.

### Forgetting to handle non-deterministic inputs

If your `verify` produces different results on different machines (e.g.,
because of uninitialized memory), consensus will fork. Make sure all
state transitions are deterministic.

### Not pruning dead forks

If you keep all forks in memory forever, you'll OOM. The wrapper prunes
dead forks automatically — but only after finalization. Don't hold
references to dead fork state.

## Where to look in the code (expanded)

- `glue/src/lib.rs` — the entry point.
- `glue/src/stateful/mod.rs:125-285` — the `Application` trait.
- `glue/src/stateful/actor/` — the `Stateful` actor.
- `glue/src/stateful/db/` — the database lifecycle traits.
- `glue/src/stateful/db/mod.rs` — `DatabaseSet` trait.
- `glue/src/stateful/db/p2p.rs` — P2P-backed state sync.
- `glue/src/stateful/probe/` — state probing.
- `glue/src/simulate/` — the test harness.
- `glue/src/simulate/engine.rs` — the engine definition.
- `glue/src/simulate/team.rs` — the team orchestration.

## If you only remember three things

1. **`Stateful` actor = speculative execution on QMDB.** Multiple forks in memory; winning fork gets applied to disk on finalization.
2. **Lazy recovery.** Pending state lives in memory; replay from finalized tip on restart.
3. **`simulate` is the integration test harness.** Composable validators + action injection + property checking.

→ Next: **Chapter 18 — Deployer**. The "ship it to AWS" tooling.