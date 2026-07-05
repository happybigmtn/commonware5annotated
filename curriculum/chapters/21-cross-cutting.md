# Chapter 21 — Cross-cutting patterns: the conventions, the runtime API, and the test patterns

> A new reviewer's reference card. The things that span every primitive — the
> acronyms you keep seeing, the runtime methods you keep needing, the test
> patterns you keep reaching for. The stuff that should have been in chapter 00
> but is too detailed for an orientation.

## Why this chapter exists

After reading chapters 00-20, a careful reviewer would notice that the same
handful of concepts show up over and over — `Context`, mailboxes, the
deterministic runtime, conformance, the simulated network, the Byzantine mocks
— but no single chapter pulls them together. This chapter is that pull-together.

If you read it first you'll be lost. If you read it last, you'll say "oh
that's what was meant by that." Read it when you start writing code.

## Glossary of acronyms

Every abbreviation the Commonware docs throw at you, in one place.

| Acronym | Expansion | What it means |
|---|---|---|
| **BFT** | Byzantine Fault Tolerance | Surviving participants that may behave arbitrarily (lie, equivocate, stay silent). The library's core assumption. |
| **DSMR** | Decoupled State Machine Replication | Consensus runs over **hashes**; application runs over **contents**. This is the philosophy chapter 00 names. |
| **PoS** | Proof of Stake | Validators are weighted by staked tokens. Most Commonware consensus dialects are leader-based with PoS-style stake weighting. |
| **BLS** | Boneh-Lynn-Shacham | A pairing-friendly signature scheme that **aggregates**: many signatures → one. BLS12-381 is the specific elliptic curve. |
| **ZK** | Zero-Knowledge | Prove a statement without revealing the witness. Used in Golden DKG and ZODA (Fiat-Shamir). |
| **DKG** | Distributed Key Generation | A group of parties jointly compute a shared key without any one party knowing it. Used for threshold cryptography. |
| **TLE** | Threshold Encryption | Encrypt a message such that any `t` of `n` parties can decrypt, but `t-1` cannot. `commonware-cryptography::bls12381::tle`. |
| **VRF** | Verifiable Random Function | Output is a random value plus a proof the value was computed correctly. Used in leader election. |
| **KZG** | Kate-Zaverucha-Goldberg | A polynomial commitment scheme. Used in some coding schemes (not directly in Commonware today, but referenced). |
| **Fiat-Shamir** | (named after authors) | Convert an interactive proof into a non-interactive one by replacing the verifier's random challenge with a hash of the transcript. |
| **Lagrange** | (interpolation) | Reconstruct a polynomial from a subset of points. The math behind threshold signature reconstruction. |
| **Feldman** / **Pedersen** | (verifiable secret sharing) | VSS schemes used as building blocks for DKG. |
| **MMR** | Merkle Mountain Range | An append-only Merkle structure. A "mountain range" of perfect Merkle trees. |
| **BMT** | Binary Merkle Tree | The classic binary Merkle tree (left-right, hash children). |
| **QMDB** | Quorum-Merkle Database | A mutable authenticated database where every committed operation has a Merkle proof. |
| **ADB** | Authenticated Database | Any DB where every read returns a Merkle proof. |
| **MMB** | Merkle Mountain Batch | Pyramid-bagged MMR for tiny current-state proofs. |
| **ZODA** | ZODA (the name itself, no expansion) | Reed-Solomon + Hadamard + Fiat-Shamir. The advanced coding scheme. |
| **RS** | Reed-Solomon | Classic erasure coding. ZODA is built on top. |
| **DPI** | Distributed Point-of-Inclusion | "Was this item included in the finalized chain?" |
| **FTL** | Faster-Than-Light | (joke, but sometimes used informally) Faster than naive protocols. |
| **Hash** | (cryptographic) | SHA-256 (sometimes BLAKE3) — collision-resistant, pre-image-resistant. |
| **Digest** | (output of a hash) | 32 bytes typically. `commonware-cryptography::Digest` trait. |
| **Hasher** | (stateful hash state) | `.update(data)` then `.finalize()` → `Digest`. |
| **Codec** | (encode/decode trait) | Deterministic binary serialization. |
| **WAL** | Write-Ahead Log | Append-only journal for crash recovery. |
| **OTP** | One-Time Pad / One-Time Password | (here: irrelevant; not used) |
| **FIFO** | First In First Out | Queue order. Mailboxes guarantee FIFO per sender. |
| **LIFO** | Last In First Out | Stack order. Not used. |
| **API** | Application Programming Interface | Standard. |
| **CRUD** | Create/Read/Update/Delete | Standard. |
| **DAG** | Directed Acyclic Graph | Used in some consensus variants (not Commonware Simplex — that's a chain). |
| **P2P** | Peer-to-Peer | Direct communication without a central server. |
| **WAN** | Wide Area Network | The internet. Opposite of LAN. |
| **LAN** | Local Area Network | A local network. |
| **TLS** | Transport Layer Security | (not used directly — Commonware uses its own authenticated handshake) |
| **QUIC** | (transport) | (not currently used by Commonware) |
| **NAT** | Network Address Translation | (handled by P2P bootstrapper) |
| **DOS** | Denial of Service | Defended by P2P rate limiting and block list. |
| **DDoS** | Distributed Denial of Service | Same defenses, plus authenticated handshakes. |
| **Otel** / **OTLP** | OpenTelemetry Protocol | The trace export format Commonware supports. |
| **MSRV** | Minimum Supported Rust Version | Commonware's MSRV is 1.91.1. |
| **SAT** / **SMT** | Satisfiability / Satisfiability Modulo Theories | (not used) |
| **NOOP** / **NOP** | No Operation | (pattern name) |
| **ASAP** | As Soon As Possible | (jargon) |
| **EOS** | End of Stream | (not a Commonware term) |

If you see an acronym I missed, grep `docs/blogs/` — most are defined in the
intro posts.

## The Runtime Context API — every method

The `Context` is your handle to the runtime. It's what you receive when
`Runner::start(|context| async move { ... })` runs. Every async code path in
Commonware takes a `Context` (or a clone of it) and uses it to do work.

`Context` is not one trait — it's a sum of traits implemented by the same type:
`Runner`, `Supervisor`, `Spawner`, `Clock`, `Metrics`, `Storage`, `Network`, etc.
The methods below come from those traits.

### From `Supervisor`

```rust
fn child(&self, label: &'static str) -> Self;
fn with_attribute(self, key: &'static str, value: impl Display) -> Self;
fn name(&self) -> Name;
```

- **`child(label)`** — create a child context. The child's metrics get the
  current label as prefix; the child gets a fresh supervision-tree node. Use
  static role names: `"engine"`, `"worker"`, `"resolver"`. **Dynamic values
  belong in `with_attribute`**, not in the label.
- **`with_attribute(key, value)`** — attach a key/value to this context. Used
  for dynamic dimensions like `"epoch"`, `"shard"`, `"peer_id"`. Becomes a
  Prometheus label.
- **`name()`** — return the full identity (`Name { label, attributes }`). Used
  for debugging, sometimes by the conformance framework.

A supervisor tree example from the docs:

```
context
  ├── child("orchestrator")
  │     └── with_attribute("epoch", "5")
  │           ├── counter: votes      -> orchestrator_votes{epoch="5"}
  │           ├── counter: proposals  -> orchestrator_proposals{epoch="5"}
  │           └── child("engine")
  │                 └── gauge: height -> orchestrator_engine_height{epoch="5"}
```

### From `Spawner`

```rust
fn shared(self, blocking: bool) -> Self;
fn dedicated(self) -> Self;
fn spawn<F, Fut, T>(self, f: F) -> Handle<T>;
fn stop(self, value: i32, timeout: Option<Duration>) -> impl Future<Output = Result<(), Error>>;
fn stopped(&self) -> signal::Signal;
```

- **`shared(blocking)`** — hint that the next spawned task may briefly block.
  The runtime can use this to route the task to a blocking-friendly thread pool
  so async tasks on the work-stealing executor are not starved. `blocking=false`
  is the default. Use for short blocking work (small file IO, DNS, etc.).
- **`dedicated()`** — pin the next spawned task to a dedicated thread. Use for
  long-lived, prioritized work (consensus engine, gossip loop).
- **`spawn(f)`** — start a supervised task. Consumes `self`. Returns a
  `Handle<T>` you can `await` or `abort()`. **All tasks are supervised**: when
  a parent finishes or is aborted, all descendants are aborted.
- **`stop(value, timeout)`** — request shutdown. Doesn't kill tasks; signals
  them via `Signal`. Waits for all `Signal` references to be dropped. The
  `value` is what `Signal::await` returns. Idempotent.
- **`stopped()`** — returns a `Signal` future. Resolves when `stop()` is called.
  Use this in your actor loop to break out cleanly.

The `Handle<T>` returned by `spawn` is what you await:

```rust
let handle = context.spawn(|ctx| async move {
    // actor logic
});
let result: Result<T, Error> = handle.await;
handle.abort();   // explicit abort
```

### From `Clock`

```rust
fn current(&self) -> SystemTime;
fn sleep(&self, duration: Duration) -> impl Future<Output = ()>;
fn sleep_until(&self, deadline: SystemTime) -> impl Future<Output = ()>;
fn timeout<F, T>(&self, duration: Duration, future: F) -> impl Future<Output = Result<T, Error>>;
```

- **`current()`** — return the runtime's virtual time.
- **`sleep(duration)`** — suspend for at least `duration`. In the deterministic
  runtime, time advances in `cycle` increments; if `duration < cycle`, the
  executor will skip to the next actionable sleep deadline.
- **`sleep_until(deadline)`** — like `sleep` but with an absolute time.
- **`timeout(duration, future)`** — race the future against `sleep(duration)`.
  Returns `Err(Error::Timeout)` if the deadline wins.

**Gotcha**: in the deterministic runtime, sleeping until `SystemTime::limit()`
will never resolve. Always wrap in `context.timeout(...)`.

### From `Metrics`

```rust
fn register<N, H, M>(&self, name: N, help: H, metric: M) -> Registered<M>;
fn encode(&self) -> String;
```

- **`register(name, help, metric)`** — register a metric. The name gets the
  current supervisor label as prefix; attributes become Prometheus labels.
  `name` must match `[a-zA-Z][a-zA-Z0-9_]*`. The returned `Registered<M>` must
  be retained for the metric to stay exposed.
- **`encode()`** — dump all metrics in Prometheus text format. Used in tests
  to assert metric values.

The supported metric types are `Counter`, `Gauge`, `CounterFamily<Label>`,
`GaugeFamily<Label>`, `Histogram`. Use `Family<LabelSet, Counter>` for
high-cardinality dimensions that would explode the time-series count.

**Gotcha**: if you re-register a metric with the same name+attributes+type,
you get the existing handle. If you re-register with a different type, it
panics. Names and attributes must stay bounded.

### From `Storage`

```rust
async fn open(&self, partition: &str, name: &[u8]) -> Result<(Blob, u64)>;
async fn open_versioned(&self, partition: &str, name: &[u8], versions: RangeInclusive<u16>)
    -> Result<(Blob, u64, u16)>;
async fn remove(&self, partition: &str, name: Option<&[u8]>) -> Result<()>;
async fn scan(&self, partition: &str) -> Result<Vec<Vec<u8>>>;
```

- **`open(partition, name)`** — open an existing blob or create a new one.
  Returns `(blob, current_size)`. Returns the blob at version
  `DEFAULT_BLOB_VERSION` (0).
- **`open_versioned(partition, name, versions)`** — open a blob whose version
  is in `versions`. Used for migrations. Returns `(blob, size, version)`.
- **`remove(partition, Some(name))`** — delete one blob. The blob's open
  handles remain readable until dropped; re-opening the name creates a fresh
  blob.
- **`remove(partition, None)`** — delete the entire partition.
- **`scan(partition)`** — list all blob names in the partition.

**Partition names** must be non-empty ASCII alphanumeric, `-`, or `_`. Other
characters fail with `Error::PartitionNameInvalid`.

**Blob names** are arbitrary bytes (typically a u64 or hash serialized big-endian).

### From `Blob` (the result of `open`)

```rust
async fn read_at(&self, offset: u64, len: usize) -> Result<IoBufsMut>;
async fn read_at_buf(&self, offset: u64, len: usize, bufs: IoBufsMut) -> Result<IoBufsMut>;
async fn write_at(&self, offset: u64, bufs: IoBufs) -> Result<()>;
async fn write_at_sync(&self, offset: u64, bufs: IoBufs) -> Result<()>;
async fn resize(&self, len: u64) -> Result<()>;
async fn sync(&self) -> Result<()>;
async fn start_sync(&self) -> Handle<()>;
```

- **`read_at(offset, len)`** — read exactly `len` bytes at `offset`. Returns
  `IoBufsMut` (zero-copy buffers). Caller-provided chunks preserved.
- **`write_at(offset, bufs)`** — write `bufs` at `offset`. NOT durable until
  `sync()`.
- **`write_at_sync(offset, bufs)`** — write AND durably persist in one call.
  Earlier unsynced writes are NOT made durable by this call.
- **`resize(len)`** — change the blob length. Extending fills with zeros;
  shrinking truncates.
- **`sync()`** — make all pending writes durable.
- **`start_sync()`** — request a sync; the returned `Handle<()>` resolves when
  durability is achieved.

**Critical contract**: drop-unsynced data may be discarded. Always `sync()`
before drop if you care about durability.

### From `Network` (the `Network` trait, not the simulated one)

```rust
async fn bind(&self, socket: SocketAddr) -> Result<Listener>;
async fn dial(&self, socket: SocketAddr) -> Result<(Sink, Stream)>;
```

- **`bind(addr)`** — bind a socket, return a `Listener`.
- **`dial(addr)`** — connect to a peer.

`Sink` and `Stream` are async byte streams. Commonware's P2P layer wraps
these; you usually don't call them directly. (The `tokio::Runtime` impl uses
real TCP; the `deterministic::Runtime` impl uses the deterministic mock.)

### The `Auditor` (only on `deterministic::Context`)

```rust
let state: String = context.auditor().state();
```

- **`state()`** — return the SHA-256 hash of the runtime's accumulated events
  as a hex string. Same seed → same hash. Use it to assert determinism:

  ```rust
  let state1 = run_test(seed);
  let state2 = run_test(seed);
  assert_eq!(state1, state2);
  ```

### The Deterministic `Config` (for `deterministic::Runner`)

```rust
deterministic::Config::default()
    .with_seed(42)                                 // change the seed
    .with_cycle(Duration::from_millis(1))          // how much time advances per loop iteration
    .with_start_time(UNIX_EPOCH)                   // virtual clock start
    .with_timeout(Some(Duration::from_secs(30)))   // panic if test runs longer
    .with_catch_panics(true)                       // catch panics (returns Error::Exited)
    .with_network_buffer_pool_config(cfg)          // tune buffer pool sizes
    .with_storage_buffer_pool_config(cfg)
    .with_storage_fault_config(faults)             // inject deterministic storage faults
```

`cycle` must be non-zero when `timeout` is set, and must be `>= SYSTEM_TIME_PRECISION`.
`start_time` must be `>= UNIX_EPOCH`. These are checked by `Config::assert()`.

### Checkpoint / Recovery

```rust
let mut checkpoint = None;
loop {
    let runner = if let Some(checkpoint) = checkpoint.take() {
        deterministic::Runner::from(checkpoint)
    } else {
        deterministic::Runner::timed(Duration::from_secs(30))
    };
    let (complete, next_checkpoint) = runner.start_and_recover(f);
    if complete { break; }
    checkpoint = Some(next_checkpoint);
}
```

This is the **recovery test pattern**: save the state after each iteration,
recreate the runner from the checkpoint, resume. If `complete` is false,
something panicked; we restart from the same state.

### The `external` feature

If your code talks to external processes (e.g., a sequencer with its own
runtime), build with `--features runtime/external` and use:

```rust
let future = my_external_call();
future.pace(&pacer, expected_latency).await;
```

`pace(latency, future)` constrains how fast `future` resolves. In the
deterministic runtime it ensures the test makes progress even when an external
process is stalled. In `tokio::Runtime` it's a no-op (production behavior is
unmodified).

**Only enable this if your code interacts with external processes** that have
their own runtime. Otherwise leave it off — it slows tests down.

## The `commonware_macros::select!` macro

`commonware-macros` provides a `select!` macro that is **cooperative-task-safe**
across both deterministic and Tokio runtimes. The AGENTS.md is explicit:

> Use `commonware_macros::select!` for concurrent operations.

The macro is used like:

```rust
use commonware_macros::select;

select! {
    msg = receiver.recv() => {
        // handle msg
    },
    _ = clock.sleep(duration) => {
        // handle timeout
    },
    signal = stopped_signal => {
        // handle shutdown
    },
}
```

What it does that the std `tokio::select!` doesn't:

1. **Automatically fusing** futures. You can use it in a loop without losing
   messages on subsequent iterations.
2. **Dropping the losing future correctly** under both runtimes. In a Tokio
   select, dropping a future mid-poll can leak state.
3. **Compatible with the deterministic runtime's sleep semantics**. Tokio's
   select with deterministic sleeps can miscount time.

The macro is exported by `runtime` (via `use commonware_macros::select;`).
**Always prefer it over `tokio::select!`** in protocol crates, per the runtime
isolation rule.

## The Mailbox API — Sender, Receiver, Feedback, Policy

Chapter 15 covers the design. Here are the actual types and methods.

### `Feedback` and `Unreliable<Feedback>`

From `actor/src/lib.rs:14-29`:

```rust
pub enum Feedback {
    Ok,        // accepted within capacity
    Backoff,   // accepted via overflow policy
    Closed,    // endpoint closed (receiver dropped)
}
impl Feedback {
    pub const fn accepted(self) -> bool {
        matches!(self, Self::Ok | Self::Backoff)
    }
}
```

`Unreliable<T>` is for endpoints that may reject work:

```rust
pub enum Unreliable<T> {
    Outcome(T),   // endpoint produced some outcome
    Rejected,     // endpoint rejected under backpressure
}
```

### `Sender<T: Policy>` and `UnreliableSender<T: UnreliablePolicy>`

```rust
pub fn enqueue(&self, message: T) -> Feedback;
```

Non-blocking. Returns immediately with `Ok` / `Backoff` / `Closed` (or
`Outcome(...)` / `Rejected` for unreliable).

### `Receiver<T: Policy>` and `UnreliableReceiver<T: UnreliablePolicy>`

```rust
pub async fn recv(&mut self) -> Option<T>;
pub fn try_recv(&mut self) -> Result<T, TryRecvError>;
```

`recv()` returns `None` after all senders are dropped and all buffered
messages drained. `try_recv()` is non-async.

### `mailbox::new(metrics, capacity) -> (Sender<T>, Receiver<T>)`

The standard constructor. `capacity` is the **bounded ready queue** size. When
that fills, the `Policy::handle` is invoked.

### The `Policy` trait

```rust
pub trait Policy: Sized {
    type Overflow: Overflow<Self>;
    fn handle(overflow: &mut Self::Overflow, message: Self);
}
```

You implement `Policy` for your message enum. The default `Overflow` is
`VecDeque<Self>` (FIFO queue). For example, `examples/log/src/application/ingress.rs`:

```rust
pub enum Message<D: Digest> {
    Propose { response: oneshot::Sender<D> },
    Verify { response: oneshot::Sender<bool> },
}

impl<D: Digest> Policy for Message<D> {
    type Overflow = VecDeque<Self>;

    fn handle(overflow: &mut VecDeque<Self>, message: Self) {
        overflow.push_back(message);
    }
}
```

The "reliable" mailbox always accepts overflow messages. The "unreliable"
mailbox lets your `Policy` reject work entirely under backpressure:

```rust
pub trait UnreliablePolicy: Sized {
    type Overflow: Overflow<Self>;
    fn handle(overflow: &mut Self::Overflow, message: Self) -> bool;  // false = reject
}
```

### Lifetime rules for `handle`

> Do not enqueue into the same mailbox from this method or from destructors
> triggered by editing `overflow`. This method runs while the mailbox holds its
> overflow lock, so same-mailbox re-entry can deadlock.

> This method should not unwind after mutating `overflow`. A panic, including
> one from a destructor triggered while editing `overflow`, can leave retained
> overflow data stranded in the mailbox.

### Drop semantics

- **Dropping all `Sender`s** disconnects the mailbox, but the `Receiver` keeps
  returning buffered messages until both ready and overflow are drained.
- **Dropping the `Receiver`** closes the mailbox. Subsequent `enqueue()` calls
  return `Feedback::Closed`.

## The Simulated Network API

The test network from `p2p/src/simulated/`. Every consensus test uses it.

### Creating the network

```rust
let (network, mut oracle) = Network::new(
    context.child("network"),
    Config {
        max_size: 1024 * 1024,         // max message size
        disconnect_on_block: true,     // disconnect when a peer is blocked
        tracked_peer_sets: NZUsize!(1), // how many peer sets to keep
    },
);
network.start();
```

### Registering channels per peer

```rust
let (vote_sender, vote_receiver) = oracle.register(pk, 0).await?;
let (certificate_sender, certificate_receiver) = oracle.register(pk, 1).await?;
```

Each peer has multiple logical channels indexed by `u32`. The same channel
index on different peers routes messages. In `examples/log`:

- `0` = votes
- `1` = certificates
- `2` = resolver

### Configuring links

```rust
oracle.add_link(pk1, pk2, Link {
    latency: Duration::from_millis(10),
    jitter: Duration::from_millis(3),
    success_rate: 0.95,   // 95% packet delivery
}).await?;
```

`Link` controls latency, jitter, and packet loss per direction.

### Dynamic network conditions

```rust
oracle.update_link(pk1, pk2, Link { ... }).await?;   // change at runtime
oracle.remove_link(pk1, pk2).await?;                // partition
```

### Blocking misbehaving peers

```rust
let blocker = oracle.control(validator.clone());
// ... in voter/batcher logic ...
blocker.block(offender_pk).await?;
```

After `block()`, all subsequent messages from `offender_pk` are dropped.
`oracle.blocked()` returns the set of blocked peer pairs.

### Sending a message

```rust
sender.send(Recipients::All, message_bytes, priority).await?;
```

`Recipients::All` broadcasts; you can also pass a `Vec<PublicKey>` for a subset.

## Byzantine test patterns

The Byzantine mocks live in `consensus/src/simplex/mocks/`. They replace
honest peers with adversarial ones. The test framework asserts that honest
nodes correctly detected and ignored the bad behavior.

### The four Byzantine mocks

1. **`Conflicter`** — sends two different proposals for the same view
   (equivocation).
2. **`Nuller`** — never participates. Silent.
3. **`Liar`** — sends invalid signatures.
4. **`Reorderer`** — sends messages out of order.

To use:

```rust
if idx == 0 {
    let cfg = mocks::conflicter::Config { /* ... */ };
    let engine = mocks::conflicter::Conflicter::new(context, cfg);
    engine.start(pending);
} else {
    let engine = Engine::new(context, cfg);
    engine.start(pending, recovered, resolver);
}
```

### The `Reporter` mock

`mocks::reporter::Reporter` records every activity the consensus engine
reports. After the test, assert:

```rust
for reporter in reporters.iter() {
    reporter.assert_no_faults();   // no peer voted for two blocks at same height
    reporter.assert_no_invalid();  // no peer voted with invalid signature
}
```

### The `Supervisor` mock

`mocks::supervisor::Supervisor` is a higher-level observer:

```rust
let supervisor = mocks::supervisor::Supervisor::new(config);
let (mut latest, mut monitor) = supervisor.subscribe().await;

while latest < required_containers {
    latest = monitor.next().await.expect("event missing");
}

let faults = supervisor.faults.lock().unwrap();
assert!(faults.is_empty());   // no Byzantine faults occurred
```

### The `all_online` integration test

From `consensus/src/simplex/mod.rs:790-1020`. This is the canonical
multi-validator test:

```rust
fn all_online<S, F, L>(mut fixture: F)
where
    S: Scheme<Sha256Digest, PublicKey = PublicKey>,
    F: FnMut(&mut deterministic::Context, &[u8], u32) -> Fixture<S>,
    L: Elector<S>,
{
    let n = 5;
    let executor = deterministic::Runner::timed(Duration::from_secs(300));
    executor.start(|mut context| async move {
        let Fixture { participants, schemes, .. } = fixture(&mut context, &namespace, n);

        let mut oracle =
            start_test_network_with_peers(context.child("network"), participants.clone(), true).await;
        let mut registrations = register_validators(&mut oracle, &participants).await;

        let link = Link {
            latency: Duration::from_millis(10),
            jitter: Duration::from_millis(1),
            success_rate: 1.0,
        };
        link_validators(&mut oracle, &participants, Action::Link(link), None).await;

        let elector = L::default();
        let relay = Arc::new(mocks::relay::Relay::new());
        let mut reporters = Vec::new();
        let mut engine_handlers = Vec::new();

        for (idx, validator) in participants.iter().enumerate() {
            let reporter_config = mocks::reporter::Config { /* ... */ };
            let reporter = mocks::reporter::Reporter::new(/* ... */);
            reporters.push(reporter.clone());

            let (actor, application) = mocks::application::Application::new(/* ... */);
            actor.start();
            let cfg = config::Config {
                scheme: schemes[idx].clone(),
                elector: elector.clone(),
                blocker: oracle.control(validator.clone()),
                automaton: application.clone(),
                relay: application.clone(),
                reporter: reporter.clone(),
                strategy: Sequential,
                partition: validator.to_string(),
                mailbox_size: NZUsize!(1024),
                epoch: Epoch::new(333),
                floor: config::Floor::Genesis(/* ... */),
                leader_timeout: Duration::from_secs(1),
                // ... more ...
                page_cache: CacheRef::from_pooler(&context, NZU16!(16_384), NZUsize!(10_000)),
                forwarding: ForwardingPolicy::Disabled,
            };
            let engine = Engine::new(context.child("engine"), cfg);
            let (pending, recovered, resolver) = registrations.remove(validator).unwrap();
            engine_handlers.push(engine.start(pending, recovered, resolver));
        }

        // ... wait for all engines to make progress ...

        for reporter in reporters.iter() {
            reporter.assert_no_faults();
            reporter.assert_no_invalid();
        }

        let blocked = oracle.blocked().await.unwrap();
        assert!(blocked.is_empty());
    });
}
```

Every line maps to a chapter:

| Line | Chapter |
|---|---|
| `deterministic::Runner::timed(...)` | 02 — deterministic runtime |
| `context.child("network")` | 02 — supervisor tree |
| `start_test_network_with_peers(...)` | 05 — P2P / simulated network |
| `register_validators(...)` | 05 — registering P2P channels |
| `Link { latency, jitter, success_rate }` | 02 + 05 — simulated link |
| `mocks::relay::Relay::new()` | 09 — broadcast Relay trait |
| `mocks::reporter::Reporter::new(...)` | 11 — Reporter trait |
| `mocks::application::Application::new(...)` | 11 — Application (Automaton) |
| `Engine::new(context.child("engine"), cfg)` | 11 — Simplex::Engine |
| `page_cache: CacheRef::from_pooler(...)` | 06 — page cache for journal reads |
| `engine.start(pending, recovered, resolver)` | 11 — Engine::start |
| `oracle.blocked()` | 05 — peer blocking |
| `assert_no_faults()` / `assert_no_invalid()` | 11 — Byzantine detection |

### Recovery testing

```rust
let mut checkpoint = None;
loop {
    let runner = if let Some(checkpoint) = checkpoint.take() {
        deterministic::Runner::from(checkpoint)
    } else {
        deterministic::Runner::timed(Duration::from_secs(30))
    };

    let (complete, next_checkpoint) = runner.start_and_recover(f);

    if complete { break; }
    checkpoint = Some(next_checkpoint);
}
```

This pattern is used to test unclean shutdowns. After `start_and_recover`,
`complete` is true if the future finished normally; false if the runtime
crashed (timed out, panicked). The checkpoint is reusable.

## The `simulate` harness (from `glue`)

`glue/src/simulate/` provides an integration test framework for full stacks.

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

You implement this to wire up your app's stack. The simulator calls `build`
for each validator in the test.

### `Plan`

```rust
pub struct Plan {
    validators: usize,
    duration: Duration,
    actions: Vec<Action>,
    properties: Vec<Property>,
}
```

A `Plan` says: run N validators for T seconds, perform these actions
mid-test, check these properties at the end.

### Actions

Actions are mid-test mutations:

- `Action::Kill(validator)` — stop one validator
- `Action::Partition({ a, b })` — disconnect A from B
- `Action::Link(link)` — change a link's parameters
- `Action::Restart(validator)` — bring a killed validator back
- `Action::AddValidator(pk)` — rotate in a new validator
- `Action::RemoveValidator(pk)` — rotate out a validator

### Properties

Properties are post-test assertions:

- `Property::NoForks` — no validator finalized two blocks at the same height
- `Property::Liveness(target_height)` — every validator finalized at least
  `target_height` blocks
- `Property::Faults(fault_set)` — exactly `fault_set` were Byzantine

### `Team`

`Team` orchestrates the running validators:

```rust
let team = Team::start(plan, engine_def).await?;
// team handles kill / restart / partition internally
```

### `ProgressTracker`

Watches finalization progress:

```rust
let progress = team.progress();
while progress.current_height() < target {
    progress.wait_for_height(target).await;
}
```

## Namespace conventions

Per `AGENTS.md`:

> Namespaces (used for domain separation in transcripts, hashing, etc.) must
> follow the pattern: `_COMMONWARE_<CRATE>_<OPERATION>`

Examples:

- `_COMMONWARE_CODING_ZODA` — ZODA encoding in the coding crate
- `_COMMONWARE_STREAM_HANDSHAKE` — Handshake protocol in the stream crate
- `_COMMONWARE_CRYPTOGRAPHY_BLS12381_DKG` — BLS12-381 DKG in the cryptography crate

**Why namespaces?** A signature on message M with namespace N is only valid
for the protocol using namespace N. Different namespaces prevent cross-protocol
replay: a signature meant for "code execution" can't be re-used as a "vote."
The convention `_COMMONWARE_<CRATE>_<OPERATION>` ensures global uniqueness
across the workspace.

When you write a Commonware app, your namespace is up to you. Common practice:

```rust
const APP_NAMESPACE: &[u8] = b"_COMMONWARE_EXAMPLES_LOG";   // examples/log
const APP_NAMESPACE: &[u8] = b"_YOURAPP_CONSENSUS";          // your app
```

Then for sub-protocols, append an operation:

```rust
let consensus_ns = union(APP_NAMESPACE, b"_CONSENSUS");
let p2p_ns = union(APP_NAMESPACE, b"_P2P");
```

## The runtime isolation rule

Per `AGENTS.md`:

> **CRITICAL**: Protocol primitives should remain runtime-agnostic:
> - Do not introduce direct `tokio` usage in protocol crates unless the module
>   is explicitly runtime-specific, a benchmark, or a test.
> - Existing runtime-owning crates and paths such as `runtime`, `deployer`'s
>   AWS feature, `commonware-macros`, and `commonware-utils::sync` may use
>   `tokio` where that is already part of their contract.
> - Prefer `futures` for async operations in runtime-agnostic code.
> - Use capabilities exported by `runtime` traits for I/O operations.
> - This keeps primitives portable across different runtime implementations.

What this means concretely:

- **Never** `use tokio;` in a primitive crate (cryptography, consensus,
  storage, p2p, etc.).
- **Always** use the `Clock`, `Spawner`, `Storage`, `Network` traits from
  `commonware-runtime`.
- For async combinators, use `futures::select_biased!`, `futures::future::join`,
  etc. — not the `tokio::` versions.
- For `select!`, use `commonware_macros::select!`.

The exception: `runtime/` itself, `deployer/` (which boots an AWS node), and
test modules. Those may import `tokio`.

## Error handling patterns

Per `AGENTS.md`:

> Use `thiserror` for all error types:

```rust
#[derive(Error, Debug)]
pub enum Error {
    #[error("descriptive message: {0}")]
    VariantWithContext(String),

    #[error("validation failed: Context({0}), Message({1})")]
    ValidationError(&'static str, &'static str),

    #[error("wrapped: {0}")]
    Wrapped(#[from] OtherError),
}
```

> **Failures Are Fatal**: Errors returned by mutable methods (e.g. `put`,
> `delete`, `sync`) are treated as unrecoverable. The database may be in an
> inconsistent state after such an error. Callers must not use a database
> after a mutable method returns an error. Reviews need not comment the
> database being in an inconsistent state after such an error.

**This is critical**: if your storage `put` returns an error, the storage
is in an undefined state. **Don't retry. Don't continue. Crash.** The
expectation is that crashes are handled at the runtime level — abort the
task, restart from a checkpoint, replay the WAL.

Immutable methods (`get`, `read`) can return errors that the caller recovers
from.

The `runtime::Error` enum is exported for use by all primitives:

```rust
use commonware_runtime::Error as RuntimeError;
```

Variants include `Exited`, `Closed`, `Aborted`, `Timeout`, `BindFailed`,
`ConnectionFailed`, `WriteFailed`, `ReadFailed`, `SendFailed`, `RecvFailed`,
`ResolveFailed`, `PartitionMissing`, `BlobMissing`, `BlobCorrupt`, etc.

## Tracing across actor boundaries

Per `AGENTS.md`:

> `tracing`'s implicit context is task-local and does not survive a mailbox
> channel. When a request crosses an actor boundary, the message must carry
> its `Span` as a field (created at enqueue with the caller as parent) and
> the actor must re-enter it with `.instrument(span)` when processing.
> At dequeue, open a child span so queue wait and processing time are
> distinguishable.

What this means concretely:

1. **Don't** rely on `tracing::Span::current()` across a mailbox. The task
   that's processing a message is a different task from the one that enqueued
   it. The implicit context is gone.
2. **Do** include a `Span` field in your message enum:

   ```rust
   pub enum Message {
       Propose { response: oneshot::Sender<Digest>, span: Span },
       Verify { response: oneshot::Sender<bool>, span: Span },
   }
   ```

3. **Do** instrument the handler:

   ```rust
   async fn handle(&mut self, msg: Message) {
       match msg {
           Message::Propose { response, span } => {
               async move {
                   // work
               }.instrument(span).await;
           },
       }
   }
   ```

4. **Use `#[tracing::instrument]` with `skip_all`** on functions that perform
   work. Always `skip_all` and opt fields in explicitly; never capture
   parameters implicitly.

   ```rust
   #[tracing::instrument(name = "voter.on_certificate", level = "info", skip_all, fields(view = view))]
   async fn on_certificate(&mut self, view: View, cert: Certificate) { ... }
   ```

5. **Span names** are dot-separated paths (`voter.on_certificate`), not
   `::`. The operation segment matches the function name. A span name must
   identify the crate and the operation, not the call site — `verify` is
   useless; `qmdb.verify` is not.

6. **Don't hold an `entered()` guard across `.await`.** Use `.instrument()`
   on the future instead.

## Conformance — adding to your type

If you implement `Encode`, you can add conformance tests. This is the
official recipe from `AGENTS.md`:

### Step 1: Add `Arbitrary` Implementation

```rust
#[cfg(feature = "arbitrary")]
impl arbitrary::Arbitrary<'_> for MyType {
    fn arbitrary(u: &mut arbitrary::Unstructured<'_>) -> arbitrary::Result<Self> {
        // construct MyType from random bytes
    }
}
```

### Step 2: Add Conformance Test Module

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[cfg(test)]
    mod conformance {
        use commonware_codec::conformance::CodecConformance;

        commonware_conformance::conformance_tests! {
            CodecConformance<MyType>,
            CodecConformance<MyType2> => 1024,  // custom case count
        }
    }
}
```

### Step 3: Generate Fixtures

```bash
just test-conformance                       # add new types, fail if existing hash changed
just regenerate-conformance -p my-crate     # acknowledge an intentional change
```

The conformance tests generate deterministic random values, encode them, hash
all the bytes together with SHA-256, and compare to a stored value in
`conformance.toml`. If you change a wire format intentionally, you regenerate
the fixture.

**WARNING**: `regenerate-conformance` is effectively a manual approval of a
breaking change. Use only when you have intentionally changed the format and
verified the change is correct.

## Concrete types — block, certificate, context

### `Finalization<S, D>`

```rust
pub struct Finalization<S: Scheme, D: Digest> {
    pub proposal: Proposal<D>,
    pub certificate: SchemeCertificate<S, D>,
}
```

`Proposal` carries the payload digest and view. `SchemeCertificate<S, D>` is
scheme-specific — for Ed25519 it's a `Vec<S::Signature>`; for BLS12-381 it's
an aggregated signature.

### `Notarization<S, D>`

Same shape as `Finalization` but with `Notarize` votes instead of `Finalize`.

### `Nullification<S, D>`

```rust
pub struct Nullification<S: Scheme, D: Digest> {
    pub view: View,
    pub certificate: SchemeCertificate<S, D>,   // over Nullify votes
}
```

### `Proposal<D>`

```rust
pub struct Proposal<D: Digest> {
    pub view: View,
    pub payload: D,
    // ... parent if CertifiableBlock is in use ...
}
```

### `SchemeCertificate<S, D>`

For Ed25519: `Vec<(S::PublicKey, S::Signature)>`.
For BLS12-381: `(S::Signature, Vec<...>)` — the aggregated signature plus the
signer list.

### `Round`

```rust
pub struct Round {
    pub epoch: Epoch,
    pub view: View,
}
```

### `Context<D, P>`

```rust
pub struct Context<D: Digest, P: PublicKey> {
    pub round: Round,
    pub leader: P,        // who proposed (or is supposed to propose)
    pub parent: Option<D>, // parent digest if available
}
```

### `Elector` and `Config` (chapter 11, full)

```rust
pub trait Config<S: Scheme>: Clone + Default + Send + 'static {
    type Elector: Elector<S>;
    fn build(self, participants: &Set<S::PublicKey>) -> Self::Elector;
}

pub trait Elector<S: Scheme>: Clone + Send + 'static {
    fn elect(&self, round: Round, certificate: Option<&S::Certificate>) -> Participant;
}
```

Built-in electors:

- `RoundRobin` — `(epoch + view) % num_participants`. Optionally shuffled by seed.
- `Random` — randomness derived from a BLS threshold VRF. Falls back to
  round-robin for view 1 (no certificate available).

### `ForwardingPolicy`

From `consensus/src/simplex/config.rs:22-42`:

```rust
pub enum ForwardingPolicy {
    Disabled,       // no proactive forwarding
    SilentVoters,   // forward to peers that didn't vote for the proposal
    SilentLeader,   // forward to the leader of the next view if they didn't vote
}
```

Forwarding is a **liveness aid** when bandwidth allows. With `Disabled`, peers
fetch blocks lazily via the resolver. With `SilentVoters`, the engine
broadcasts the block to peers that missed it after we locally certify. With
`SilentLeader`, the engine sends only to the next view's leader.

For low-bandwidth deployments, `Disabled` is fine. For low-latency
deployments, `SilentVoters` minimizes the chance the next leader is missing
the block.

### `Reporter::Activity`

For Simplex, `Activity` is:

```rust
pub enum Activity<D: Digest, P: PublicKey> {
    VoteNotarization(View, D),
    VoteNullification(View),
    VoteFinalization(View, D),
    Notarization(View, D),
    Nullification(View),
    Finalization(View, D),
    Fault(Fault<D, P>),
    MissingBlock(D),
    InvalidSignature(P),
}
```

Implementations log, slash, or reward based on activity type.

### `Strategy`

From `commonware_parallel`:

```rust
pub trait Strategy {
    type Item: Send;
    fn execute<F, R>(self, work: F) -> Result<Vec<R>, Error>
    where F: FnMut(Self::Item) -> R;
}
```

Built-ins:

- `Sequential` — runs in the current task. No parallelism.
- `Rayon` — uses a Rayon thread pool. Created via
  `Runtime::create_strategy(concurrency)`.

Simplex's `config.strategy` is used for parallel batch verification and
signature aggregation. `Sequential` is the safe default; `Rayon` is faster
when many cores are available.

## The Page Cache and Buffer Pool

### `CacheRef::from_pooler(context, page_size, cache_size)`

```rust
let page_cache = CacheRef::from_pooler(&context, NZU16!(16_384), NZUsize!(10_000));
```

The page cache lets journals amortize disk reads. Pages are 16 KB
(`page_size`); the cache holds `cache_size` of them. LRU eviction.

Use it in storage-heavy paths (journal reads, QMDB current-state reads).
Skip it for one-shot reads.

### `BufferPool`

The runtime maintains buffer pools for I/O. Two pools:

- **network buffer pool** — for `Sink::send` and `Stream::recv`. Configured
  for typical network message sizes.
- **storage buffer pool** — for `Blob::read_at` / `write_at`. Configured for
  disk block sizes.

`BufferPoolThreadCache` is a per-thread cache that reduces contention.
Disabled by default in the deterministic runtime (MIRI is slow with it on).

Tune via `with_network_buffer_pool_config` / `with_storage_buffer_pool_config`.

## The `finalize` (chapter 17) flow

When `Marshal::Finalization` arrives:

1. The `Application` trait's `finalized()` hook is called (default no-op).
2. `Stateful::handle_finalize(block, cert)`:
   - Find pending state for this block.
   - `database.merkleize(...)` — apply to disk.
   - `database.finalize(...)` — update sync targets.
   - `prune_dead_forks(block, cert)` — drop losers.
   - `marshal_ack_tx.send(block.height)` — acknowledge to Marshal.

**Critical**: if `merkleize` or `finalize` returns an error, the storage is
unrecoverable. Abort the actor; don't continue.

## Putting it all together — a minimal Simplex app

Putting it all together, here's the structure of `examples/log/src/main.rs`
(maps to the actual file at `examples/log/src/main.rs:50-246`):

```rust
use commonware_consensus::simplex::{self, elector::RoundRobin, types::{Epoch, ViewDelta}};
use commonware_cryptography::{ed25519, Sha256, Signer as _};
use commonware_p2p::{authenticated::discovery, Manager as _};
use commonware_parallel::Sequential;
use commonware_runtime::{buffer::paged::CacheRef, tokio, Quota, Runner, Supervisor as _};
use commonware_utils::{ordered::Set, union, NZUsize, TryCollect, NZU16, NZU32};

const APPLICATION_NAMESPACE: &[u8] = b"_COMMONWARE_EXAMPLES_LOG";

fn main() {
    // 1. Parse CLI
    let matches = parse_cli();
    let (key, port) = parse_identity(matches);
    let signer = ed25519::PrivateKey::from_seed(key);
    let validators: Set<_> = parse_validators(matches);
    let bootstrappers = parse_bootstrappers(matches);
    let storage_directory = matches.get_one::<String>("storage-dir").unwrap();

    // 2. Tokio runtime
    let runtime_cfg = tokio::Config::new().with_storage_directory(storage_directory);
    let executor = tokio::Runner::new(runtime_cfg);

    executor.start(async |context| {
        // 3. P2P network
        let p2p_cfg = discovery::Config::local(/* ... */);
        let (mut network, mut oracle) = discovery::Network::new(context.child("network"), p2p_cfg);
        oracle.track(0, validators.clone());

        // 4. P2P channels
        let (vote_sender, vote_receiver) = network.register(0, Quota::per_second(NZU32!(10)), 256);
        let (certificate_sender, certificate_receiver) = network.register(1, Quota::per_second(NZU32!(10)), 256);
        let (resolver_sender, resolver_receiver) = network.register(2, Quota::per_second(NZU32!(10)), 256);

        // 5. Application
        let namespace = union(APPLICATION_NAMESPACE, b"_CONSENSUS");
        let scheme = application::Scheme::signer(&namespace, validators.clone(), signer.clone()).unwrap();
        let (application, scheme, reporter, mailbox) = application::Application::new(
            context.child("application"),
            application::Config { hasher: Sha256::default(), scheme, mailbox_size: NZUsize!(1024) },
        );

        // 6. Simplex config
        let cfg = simplex::Config {
            scheme,
            elector: RoundRobin::<Sha256>::default(),
            blocker: oracle,
            automaton: mailbox.clone(),
            relay: mailbox.clone(),
            reporter: reporter.clone(),
            partition: String::from("log"),
            mailbox_size: NZUsize!(1024),
            epoch: Epoch::zero(),
            floor: simplex::Floor::Genesis(application::genesis::<Sha256>()),
            replay_buffer: NZUsize!(1024 * 1024),
            write_buffer: NZUsize!(1024 * 1024),
            leader_timeout: Duration::from_secs(1),
            certification_timeout: Duration::from_secs(2),
            timeout_retry: Duration::from_secs(10),
            fetch_timeout: Duration::from_secs(1),
            activity_timeout: ViewDelta::new(10),
            skip_timeout: ViewDelta::new(5),
            fetch_concurrent: NZUsize!(32),
            page_cache: CacheRef::from_pooler(&context, NZU16!(16_384), NZUsize!(10_000)),
            strategy: Sequential,
            forwarding: simplex::ForwardingPolicy::Disabled,
        };
        let engine = simplex::Engine::new(context.child("engine"), cfg);

        // 7. Start everything
        application.start();
        network.start();
        engine.start((vote_sender, vote_receiver), (certificate_sender, certificate_receiver), (resolver_sender, resolver_receiver));

        // 8. Block on GUI
        let gui = gui::Gui::new(context.child("gui"));
        gui.run().await;
    });
}
```

That's the entire app. ~150 lines. Every line maps to a chapter. Twenty
chapters of primitives, and you can build a real Byzantine fault-tolerant
replicated log in a day.

## Common gotchas — the list

These are the mistakes every new Commonware user makes. Memorize them.

### 1. Using `tokio::select!` in a protocol crate

The deterministic runtime doesn't support it. Use `commonware_macros::select!`.

### 2. Capturing `Context` by reference instead of consuming

`Spawner::spawn` consumes `self`. `Context::child` consumes `self`. Patterns:

```rust
// CORRECT
let child = context.child("worker");
let handle = child.spawn(|ctx| async move { ... });

// WRONG (can't move out of borrowed content)
let handle = context.child("worker").spawn(|ctx| async move { ... });
// (actually this works because child consumes — but if you try to use context after,
// it fails to compile. Just consume it.)
```

### 3. Forgetting `sync()` before drop

Drop-unsynced blob data may be discarded. Always `sync()` before dropping a
blob or journal that has pending writes.

### 4. Returning `false` from `verify` for temporary conditions

`verify` is single-shot. Once you return `false`, consensus treats the
proposal as permanently invalid. For "I'm still waiting for the data" or
"time skew," keep the future pending.

### 5. Re-proposing in the same view

`propose` is called once per view where you're leader. Don't re-propose
within the same view. That's Byzantine behavior.

### 6. Wrong namespace

Signatures are verified against a specific namespace. Wrong namespace =
verification fails. Use the protocol's namespace constant.

### 7. Mutating storage after a storage error

If `put` returns an error, the database is in an undefined state. Abort the
actor; don't continue. The error is unrecoverable.

### 8. Tracing across actors without carrying the span

`tracing::Span::current()` doesn't cross mailbox boundaries. Carry the span
as a field on your message type and re-enter it with `.instrument()`.

### 9. Span name with `::`

Span names use `.` not `::`. `voter.on_certificate` not `voter::on_certificate`.

### 10. Re-registering a metric with a different type

`register(name, ..., counter)` then `register(name, ..., gauge)` → panic.
Re-register with the same type → existing handle.

### 11. Sleeping past `SystemTime::limit()`

`context.sleep_until(SystemTime::limit())` will never resolve in the
deterministic runtime. Always wrap in `context.timeout(...)`.

### 12. Spawning without a child label

`context.spawn(...)` works but creates a metric named after the runtime's
default. Always use `context.child("name").spawn(...)` for clarity.

### 13. Using `test_rng()` when you need reproducibility

`test_rng()` is non-deterministic across runs. For reproducible tests, use
`test_rng_seeded(seed)` or pass a seeded `StdRng`.

### 14. Non-deterministic operations in `verify` or `apply`

`verify` and `apply` (chapter 17) must be deterministic. If two honest nodes
compute different state for the same block, consensus forks. The auditor
catches this: `context.auditor().state()` will differ between runs.

### 15. Holding an `entered()` guard across `.await`

`let _guard = span.enter(); ... await ...` will silently lose the span.
Use `.instrument(span)` on the future instead.

## Reference — where to look

| Concept | File | Lines |
|---|---|---|
| `Context` trait surface | `runtime/src/lib.rs` | 152-500 |
| `Spawner::shared/dedicated` | `runtime/src/lib.rs` | 262-281 |
| `Clock::timeout` | `runtime/src/lib.rs` | 484-499 |
| `Storage::open_versioned` | `runtime/src/lib.rs` | 662-667 |
| `Blob::write_at_sync` | `runtime/src/lib.rs` | 759-763 |
| `Auditor::state` | `runtime/src/deterministic.rs` | 144-183 |
| `Config::with_*` | `runtime/src/deterministic.rs` | 249-304 |
| `Checkpoint` | `runtime/src/deterministic.rs` | 442-463 |
| `Runner::start_and_recover` | `runtime/src/deterministic.rs` | 518-570 |
| `Network::new` (simulated) | `p2p/src/simulated/network.rs` | 194 |
| `Link` struct | `p2p/src/simulated/ingress.rs` | 134 |
| `register` channels | `p2p/src/simulated/ingress.rs` | 428 |
| `blocker.block` | `p2p/src/simulated/ingress.rs` | (see test patterns) |
| `Mailbox::Policy` | `actor/src/mailbox.rs` | 92-112 |
| `Sender::enqueue` | `actor/src/mailbox.rs` | 274-280 |
| `Receiver::recv` | `actor/src/mailbox.rs` | 310-326 |
| `mailbox::new` | `actor/src/mailbox.rs` | 359-367 |
| `Automaton` trait | `consensus/src/lib.rs` | 99-146 |
| `CertifiableAutomaton` | `consensus/src/lib.rs` | 153-191 |
| `Relay` trait | `consensus/src/lib.rs` | 197-213 |
| `Reporter` trait | `consensus/src/lib.rs` | 215-227 |
| `Monitor` trait | `consensus/src/lib.rs` | 229-244 |
| `Elector` trait | `consensus/src/simplex/elector.rs` | 78-87 |
| `ForwardingPolicy` | `consensus/src/simplex/config.rs` | 22-42 |
| `Floor` enum | `consensus/src/simplex/config.rs` | 46-51 |
| `Application` trait (Glue) | `glue/src/stateful/mod.rs` | 125-285 |
| `DatabaseSet` trait | `glue/src/stateful/db/mod.rs` | (chapter 17) |
| `simulate::EngineDefinition` | `glue/src/simulate/` | (chapter 17) |
| `examples/log/src/main.rs` | `examples/log/src/main.rs` | 50-246 |
| `examples/log/src/application/ingress.rs` | mailbox policy example | 1-92 |
| `examples/log/src/application/actor.rs` | application actor | 1-73 |
| AGENTS.md test patterns | `AGENTS.md` | (whole file) |

## Comprehensive Rust patterns reference

Commonware is a study in advanced Rust. This section catalogs every
pattern you need to be fluent in to read and extend the codebase.

### `PhantomData<T>` — the type-level witness

`PhantomData<T>` is a zero-sized marker that says "this type acts as
if it owns a `T`" without actually storing one. Commonware uses it
extensively for type-level parameters that don't affect runtime
representation:

```rust
// A digest typed by its hash algorithm
pub struct Digest<H: Hasher> {
    bytes: [u8; 32],
    _hasher: PhantomData<H>,
}

// Two different digest types, even though they're both 32 bytes
type Sha256Digest = Digest<Sha256>;
type Blake3Digest = Digest<Blake3>;

fn process<H: Hasher>(d: Digest<H>) { /* ... */ }
```

The `PhantomData<H>` makes `Digest<Sha256>` and `Digest<Blake3>`
**different types** at the type level, so you can't accidentally mix
them. It costs zero bytes at runtime.

### Higher-Ranked Trait Bounds (HRTB)

HRTBs let you say "for any lifetime, this trait is implemented":

```rust
fn with_callback<F>(f: F)
where
    F: for<'a> Fn(&'a Context) -> BoxFuture<'a, ()>,
{ /* ... */ }
```

The `for<'a>` says "this works for any lifetime." Commonware uses
HRTBs in the `Spawner` and `Clock` traits to express that the future
returned by a method can borrow from the context for any lifetime.

### Generic Associated Types (GATs)

GATs let you have type parameters on associated types:

```rust
pub trait Stream {
    type Item<'a> where Self: 'a;
    fn next<'a>(&'a mut self) -> Self::Item<'a>;
}
```

Commonware uses GATs in the `Automaton` and `Relay` traits to express
that the future returned depends on the lifetime of `&mut self`:

```rust
pub trait Automaton<D: Digest, P: PublicKey> {
    fn propose(&mut self, ctx: Context) -> impl Future<Output = D> + Send;
    fn verify(&mut self, ctx: Context, payload: D) -> impl Future<Output = bool> + Send;
}
```

The `impl Future<Output = D>` is a GAT in spirit (it's an opaque
future type tied to `&mut self`).

### Const generics — type-level numbers

Const generics let you parameterize types by **values**, not just
types:

```rust
pub struct FixedArray<T, const N: usize> {
    data: [T; N],
}

let arr: FixedArray<u32, 16> = FixedArray { data: [0; 16] };
```

Commonware uses const generics for fixed-size cryptographic
structures:

```rust
pub struct Signature<S: Scheme, const LEN: usize> {
    bytes: [u8; LEN],
    _scheme: PhantomData<S>,
}

type Ed25519Sig = Signature<Ed25519, 64>;
type BlsSig = Signature<Bls12381, 96>;
```

The length is part of the type, so you can't mix up signatures of
different lengths.

### Newtype pattern — type safety without overhead

The newtype pattern wraps a primitive in a struct for type safety:

```rust
pub struct ValidatorIndex(u64);
pub struct BlockHeight(u64);

// Can't accidentally compare them
let h: BlockHeight = 100;
let i: ValidatorIndex = 100;
assert!(h == i);  // type error: mismatched types
```

Commonware uses newtypes throughout: `View(u64)`, `Epoch(u64)`,
`BlockHeight(u64)`, etc. The wrapping adds zero runtime cost.

### RAII guards — resource management via Drop

RAII (Resource Acquisition Is Initialization) ties resource
management to scope:

```rust
pub struct MetricsGuard { /* ... */ }

impl Drop for MetricsGuard {
    fn drop(&mut self) {
        // automatically unregister metrics, release handles, etc.
    }
}

fn do_something() -> MetricsGuard {
    let guard = MetricsGuard::new();
    // ... do work ...
    guard   // dropped at end of scope
}
```

Commonware uses RAII for:

- **Journal write transactions**: a `JournalTransaction` automatically
  aborts if not committed.
- **Span guards**: `tracing::Span::enter()` returns a guard that
  exits the span on drop.
- **Lock guards**: `Mutex::lock()` returns a guard that unlocks on
  drop.

### Type state pattern — encoding states in types

The type state pattern encodes state machines in the type system:

```rust
pub struct Connection<State> { /* ... */ }
pub struct Disconnected;
pub struct Connecting;
pub struct Connected;

impl Connection<Disconnected> {
    pub fn connect(self) -> Connection<Connecting> { /* ... */ }
}

impl Connection<Connecting> {
    pub fn handshake(self) -> Result<Connection<Connected>, Error> { /* ... */ }
}

impl Connection<Connected> {
    pub fn send(&self, data: &[u8]) { /* ... */ }
    pub fn recv(&self) -> Vec<u8> { /* ... */ }
}
```

You can't call `send` on a `Disconnected` connection — the type
system enforces it.

Commonware uses type state in the `Engine` lifecycle: an `Engine`
starts in `Engine::new(...)` (constructed but not started), then
`engine.start(...)` returns an `EngineHandle` (running). You can't
call `start` twice.

### Builder pattern — flexible construction

The builder pattern constructs complex types via method chaining:

```rust
let cfg = simplex::Config::default()
    .with_scheme(scheme)
    .with_elector(elector)
    .with_partition("log".to_string())
    .with_mailbox_size(NZUsize!(1024))
    .with_leader_timeout(Duration::from_secs(1))
    .build();
```

Commonware uses builders for `Config` types throughout. The builder
returns `Self` by value, so methods can be chained.

### Strategy pattern — pluggable behavior

The strategy pattern selects behavior at runtime:

```rust
pub trait Strategy {
    type Item: Send;
    fn execute<F, R>(self, work: F) -> Result<Vec<R>, Error>
    where F: FnMut(Self::Item) -> R;
}

pub enum Sequential;
pub struct Rayon(usize);

impl Strategy for Sequential { /* sequential execution */ }
impl Strategy for Rayon { /* parallel execution */ }
```

Commonware's `Strategy` trait lets Simplex choose between sequential
and parallel (Rayon) execution for batch verification. The choice
is made at config time.

### Actor pattern — concurrency via messaging

The actor pattern encapsulates state and behavior in actors that
communicate via messages:

```rust
pub struct Actor {
    state: Mutex<State>,
    inbox: Mailbox<Message>,
}

impl Actor {
    async fn run(mut self, mut ctx: Context) {
        loop {
            select! {
                msg = self.inbox.recv() => self.handle(msg).await,
                _ = ctx.stopped() => break,
            }
        }
    }

    async fn handle(&mut self, msg: Message) {
        match msg {
            Message::Propose { response } => {
                let result = self.state.propose();
                response.send(result);
            },
            // ...
        }
    }
}
```

Commonware uses the actor pattern pervasively. The mailbox provides
backpressure; the message-passing prevents data races.

## Advanced error handling

Commonware's error handling is opinionated. This section captures
the patterns in detail.

### `thiserror` vs `anyhow` — when to use which

The rule (from AGENTS.md):

- **Library code** uses `thiserror` — typed errors with named variants.
- **Application code** uses `anyhow::Result` — opaque errors with
  context.

```rust
// Library: typed
#[derive(Debug, thiserror::Error)]
pub enum ConsensusError {
    #[error("invalid proposal: {0}")]
    InvalidProposal(String),
    #[error("notarization failed")]
    NotarizationFailed,
    #[error("journal error")]
    Journal(#[from] storage::journal::Error),
}

// Application: opaque
async fn main() -> anyhow::Result<()> {
    let cfg = parse_config().context("loading config")?;
    run(cfg).await.context("running app")?;
    Ok(())
}
```

Why this split? Library callers want to **match** on error variants
to decide what to do. Application code is at the top of the call
stack and just needs to display a useful error message.

### The unrecoverable pattern — `Failures Are Fatal`

For mutable storage operations, errors are **fatal**. The storage
may be in an inconsistent state. Don't retry; abort:

```rust
let result = database.put(key, value);
match result {
    Ok(_) => continue(),
    Err(e) => {
        // Storage is in an undefined state. Abort.
        tracing::error!(error = %e, "storage failure");
        return Err(StorageError::Inconsistent);
    }
}
```

This is the "Failures Are Fatal" pattern from AGENTS.md. The
runtime handles the abort; the application must not continue using
the storage after a failure.

For immutable operations (`get`, `read`), errors are recoverable.
The caller can try again, fall back, etc.

### Context wrapping — adding information as errors propagate

Use `anyhow::Context` to add context at each step:

```rust
fn load_config() -> anyhow::Result<Config> {
    let bytes = std::fs::read("config.toml")
        .context("reading config.toml")?;
    let cfg: Config = toml::from_slice(&bytes)
        .context("parsing config.toml")?;
    Ok(cfg)
}
```

The error message becomes: `"parsing config.toml: missing field
`participants`"`. The chain of context tells you exactly where the
failure happened.

### Error propagation via `?`

The `?` operator propagates errors up the call stack:

```rust
async fn setup() -> anyhow::Result<()> {
    let cfg = load_config()?;       // propagates load errors
    let runtime = build_runtime(cfg.clone())?;   // propagates build errors
    start_consensus(runtime, cfg).await?;        // propagates consensus errors
    Ok(())
}
```

The `?` automatically converts error types via `From`. So
`load_config`'s `anyhow::Error` propagates without explicit
conversion.

### Logging at boundaries — errors at the edges

Log errors at boundaries (network, FFI, public API), not deep in
the call stack:

```rust
// Deep in the stack: just return Err
fn process_block(&mut self, block: &Block) -> Result<(), Error> {
    self.apply(block)?;
    self.commit()?;
    Ok(())
}

// At the API boundary: log and convert
pub async fn handle_request(&mut self, req: Request) -> Response {
    match self.process_block(&req.block) {
        Ok(_) => Response::ok(),
        Err(e) => {
            tracing::error!(error = %e, "failed to process block");
            Response::error("internal error")
        }
    }
}
```

Logging at every level creates noise. Logging only at boundaries
gives you a clean trail of where errors propagated to.

### Common error types in Commonware

- `commonware_runtime::Error` — runtime errors (network, storage,
  spawn, etc.).
- `commonware_storage::Error` — storage errors.
- `commonware_consensus::Error` — consensus errors.
- `commonware_cryptography::Error` — cryptographic errors.
- `commonware_codec::Error` — encoding/decoding errors.

Each is a `thiserror` enum with specific variants. Library code
should use the most specific one.

## Advanced testing patterns

Commonware's testing strategy is multi-layered. This section covers
each layer.

### Property-based testing — `proptest`

Property-based testing generates random inputs and checks invariants:

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn bls_aggregate_uniqueness(
        sigs in any::<Vec<BlsSignature>>(),
    ) {
        let agg = aggregate(&sigs);
        prop_assert!(verify_aggregate(&agg, &[], &public_keys(&sigs)));

        // Negative: removing any sig breaks verification
        for i in 0..sigs.len() {
            let reduced: Vec<_> = sigs.iter().enumerate()
                .filter(|(j, _)| *j != i)
                .map(|(_, s)| s.clone())
                .collect();
            let reduced_agg = aggregate(&reduced);
            prop_assert!(!verify_aggregate(&reduced_agg, &[], &public_keys(&sigs)));
        }
    }
}
```

The property: "the aggregate signature is valid if and only if all
input signatures are included." `proptest` tries to find inputs that
violate this; if it does, you have a bug.

### Fuzzing — `cargo fuzz`

Fuzzing generates malformed inputs to find crashes:

```rust
// fuzz/fuzz_targets/codec_decode.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    let _ = commonware_codec::decode::<MyType>(data);
});
```

Commonware uses `cargo fuzz` for the codec and cryptography crates.
Each fuzz target takes raw bytes and tries to decode; any panic or
undefined behavior is a bug.

### Model-based testing — the simulate harness

Model-based testing runs a **model** of the system alongside the
real system and checks they agree:

```rust
let model = Model::new();
let real = System::new();
let actions = vec![
    Action::SendMessage(from, to, msg),
    Action::CrashValidator(pk),
    Action::PartitionValidators(a, b),
];
for action in actions {
    model.apply(&action);
    real.apply(&action).await;
    assert_eq!(model.state(), real.state());
}
```

Commonware's `glue::simulate` provides this. It's particularly
useful for testing crash recovery, view changes, and Byzantine
behaviors.

### Conformance testing — wire format stability

Conformance tests ensure the wire format doesn't change accidentally:

```rust
commonware_conformance::conformance_tests! {
    CodecConformance<MyType>,
}
```

The test encodes a deterministic set of values, hashes the bytes,
and compares against a stored hash in `conformance.toml`. If the
hash changes, the wire format changed — that's a breaking change
that requires explicit approval.

### Integration test pyramid

Commonware's test pyramid:

```
        /\
       /  \       E2E tests (Alto runs in CI)
      /────\      Few; expensive; slow
     /      \
    /────────\    Integration tests (simulate harness)
   /          \   Some; medium cost; medium speed
  /────────────\
 /              \  Unit tests (every primitive)
/________________\ Many; cheap; fast
```

For each new feature:

1. **Unit tests** for the function itself.
2. **Integration tests** for the primitive's behavior under
   realistic conditions.
3. **E2E tests** in the simulate harness for full-system behavior.

### Load testing — saturating the system

Load testing pushes the system past its design limits:

```rust
let config = LoadConfig {
    target_tps: 100_000,         // 100k TPS
    duration: Duration::from_secs(60),
    validator_count: 100,
    latency: Duration::from_millis(50),
};
let results = run_load_test(config).await?;
assert!(results.p99_latency < Duration::from_millis(500));
assert!(results.throughput > 50_000);
```

Commonware's `flood` example (chapter 19) is a load test for
gossip. The `estimator` example is a load test for consensus.

### Chaos testing — injecting failures

Chaos testing randomly injects failures:

```rust
let chaos = ChaosConfig {
    crash_probability: 0.01,     // 1% chance per second
    partition_probability: 0.001,
    latency_jitter: Duration::from_millis(50),
};
run_with_chaos(chaos, async {
    run_normal_consensus().await
}).await?;
```

The system should remain **safe** (no forks) under chaos. Liveness
may degrade, but safety must hold.

## Performance profiling

Optimizing Commonware code requires the right tools.

### `perf` — the Linux profiler

`perf` is the canonical Linux profiler:

```bash
# Sample at 1kHz for 60 seconds
perf record -F 1000 -p $(pidof mychain) -g -- sleep 60

# Generate a flame graph
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

The flame graph shows CPU time per function. Wide blocks are hot
spots.

### `cargo flamegraph` — easier profiling

`cargo flamegraph` wraps `perf` with Cargo integration:

```bash
cargo flamegraph --bin mychain -- --config /etc/mychain.toml
```

It generates `flamegraph.svg` in the current directory.

### `samply` — cross-platform sampling profiler

`samply` works on macOS, Linux, and Windows. The deployer's `aws
profile` command uses it:

```bash
samply record -- ./target/release/mychain --config /etc/mychain.toml
```

Output: a profile in `samply`'s format + a flame graph.

### CPU vs memory vs I/O profiling

Different profilers for different resources:

| Resource | Tool          | Output                        |
|----------|---------------|-------------------------------|
| CPU      | `perf`, `samply` | Flame graph, hot spots      |
| Memory   | `heaptrack`, `valgrind --tool=massif` | Allocation profile |
| I/O      | `iotop`, `bcc` tools | I/O operations             |
| Network  | `tcpdump`, `wireshark` | Packet capture         |
| Locks    | `perf lock`, `mutrace` | Lock contention       |

For BFT systems, CPU and network are usually the bottlenecks.
Memory is rarely the issue (Commonware is memory-efficient).

### Cache-line alignment — `repr(align(64))`

Modern CPUs have 64-byte cache lines. If two variables share a
cache line, writes to one invalidate the other — **false sharing**.
The fix: align hot variables to cache lines.

```rust
#[repr(align(64))]
struct AlignedCounter {
    value: AtomicU64,
}
```

This forces the counter to its own cache line. Commonware uses
this for hot counters (votes per view, blocks per height).

### False sharing — the silent killer

False sharing looks like this:

```rust
// BAD: counters share a cache line
struct Hot {
    counter_a: AtomicU64,
    counter_b: AtomicU64,
}

// Two threads write to counter_a and counter_b respectively.
// Every write to a invalidates b's cache line → b's CPU has to
// re-fetch. Massive slowdown under contention.
```

The fix:

```rust
// GOOD: counters on separate cache lines
#[repr(align(64))]
struct AlignedCounter(AtomicU64);

struct Hot {
    counter_a: AlignedCounter,
    counter_b: AlignedCounter,
}
```

For high-contention counters (votes, blocks), this is a 5-10x
speedup.

## Production deployment checklist

A pre-flight checklist for production.

### Pre-deployment

- [ ] **Configuration validated**: all required fields populated,
      instance types available, regions support the instance types.
- [ ] **Quotas raised**: EC2 instance quotas per region, IAM role
      quotas, S3 bucket quotas (100 by default).
- [ ] **SSH keys configured**: key pair created in target region,
      public key registered with the deployer.
- [ ] **VPC and subnets created**: explicit VPC (not default),
      subnets in each AZ, route tables configured.
- [ ] **Security groups staged**: rules defined, references to
      subnets and CIDRs validated.
- [ ] **IAM roles and policies**: deployer role has broad permissions,
      validator role has least-privilege.
- [ ] **S3 bucket naming**: name is globally unique, doesn't collide
      with other deployments.
- [ ] **Logging configured**: CloudWatch log groups created,
      retention policies set.
- [ ] **Metrics configured**: Prometheus scrape config updated,
      dashboards created.
- [ ] **Alerting configured**: SNS topics, PagerDuty integration.
- [ ] **Disaster recovery plan**: documented runbook, tested
      recently.
- [ ] **Capacity planning**: instance sizes match expected load.
- [ ] **Network bandwidth**: validators have ≥ 100 Mbps links.
- [ ] **Testnet verification**: same binary + config deployed to
      testnet, smoke tests passed.

### Post-deployment

- [ ] **Instance health**: all instances passing status checks
      within 5 minutes.
- [ ] **Finalization progress**: blocks being produced at expected
      rate.
- [ ] **CPU usage**: < 70% baseline; < 90% peak.
- [ ] **Memory usage**: < 80% baseline; < 95% peak.
- [ ] **Disk usage**: < 70%; EBS volume sizing adequate.
- [ ] **Network throughput**: < 50% of available bandwidth.
- [ ] **Log volume**: expected rate (info-level, ~50 MB/day/validator).
- [ ] **Error rate**: < 0.1% (by finalization count).
- [ ] **Cross-region latency**: < 100ms RTT for majority of peers.
- [ ] **Alert system**: alarms firing as expected, PagerDuty
      integration works.

### Alerting rules

The deployer emits CloudWatch alarms for:

| Metric                          | Threshold         | Severity |
|---------------------------------|-------------------|----------|
| `finalization_lag > 5 blocks`   | Any 5-min window  | Page     |
| `view_changes_per_minute > 10`  | Any 5-min window  | Page     |
| `cpu_usage > 80%`               | 5-min sustained   | Page     |
| `disk_usage > 85%`              | Any check         | Page     |
| `peers_connected < 2f+1`        | Any check         | Page     |
| `error_log_rate > 1/min`        | Any 5-min window  | Page     |
| `finalization_latency_p99 > 2s` | Any 5-min window  | Page     |
| `instance_status_failed`        | Any check         | Auto-heal |
| `memory_usage > 90%`            | 5-min sustained   | Page     |

### Rollback plan

Every deploy should have a documented rollback:

```markdown
# Rollback procedure

## When to roll back
- Finalization stops for > 5 minutes after deploy.
- Error rate > 1% sustained for > 10 minutes.
- P95 latency > 2x pre-deploy baseline.
- Any unrecoverable error in logs.

## How to roll back
1. Identify the previous binary version: `aws s3 ls s3://deployment/binaries/`
2. Update user-data script to point at previous version.
3. Restart instances: `deployer aws restart --config mychain.yaml`
4. Wait for instances to come up (~3 minutes).
5. Verify finalization resumes.
6. Investigate the root cause.

## Post-rollback
1. File a post-mortem issue.
2. Identify the bug or regression.
3. Add a test case.
4. Re-deploy after the fix is verified.
```

### Disaster recovery

The full DR plan covers:

1. **Single instance failure**: terminate + relaunch (~3 min).
2. **AZ failure**: validators in other AZs continue; deployer
   replaces AZ (~10 min).
3. **Region failure**: validators in other regions continue;
   deployer spawns DR region (~30 min).
4. **Corruption recovery**: restore from S3 checkpoint (~1 hr).
5. **Bug in production**: roll back to previous binary; debug
   in testnet (~30 min).

Test each scenario quarterly.

## Advanced gotchas — the deeper list

Beyond the chapter's earlier gotchas, here are 30+ more common
pitfalls.

### 31. Blocking the executor with sync I/O

The async runtime expects non-blocking I/O. Blocking I/O (sync file
reads, sync HTTP calls) stalls the executor:

```rust
// BAD: blocks the executor
let contents = std::fs::read_to_string("config.toml")?;

// GOOD: spawn blocking
let contents = tokio::fs::read_to_string("config.toml").await?;
```

For CPU-bound work, use `spawn_blocking`:

```rust
let result = tokio::task::spawn_blocking(move || {
    expensive_cpu_work()
}).await?;
```

### 32. Forgetting to clone the Sender before passing

`Mailbox::Sender` is **not** `Clone` by default (only when `T: Clone`).
Passing it by value moves it; you can't use the original after:

```rust
let (tx, rx) = mailbox::new(...);
let tx2 = tx.clone();      // explicit clone
process(tx);                // moves tx
process(tx2);               // uses the clone
```

If you forget to clone, the second `process` call fails to compile.

### 33. Spawning from a borrowed Context

`Spawner::spawn` consumes `self`. You can't spawn from `&Context`:

```rust
// WRONG
fn start(&self, ctx: &Context) {
    ctx.spawn(|c| async move {});   // can't move out of borrow
}

// RIGHT
fn start(self, ctx: Context) {
    ctx.spawn(|c| async move {});   // moves ctx
}
```

### 34. Using `unwrap` in production paths

`unwrap` panics. In production, prefer:

```rust
let value = option?;                  // propagate Option::None
let result = fallible?;                // propagate Result::Err
let value = option.unwrap_or(default); // provide fallback
```

`unwrap` is OK in tests and examples. In library code, propagate
errors.

### 35. `expect`ing on user input

`expect("invalid config")` panics with a fixed message. For user
input, return `Result` with context:

```rust
// BAD
let key = input.parse().expect("invalid key");

// GOOD
let key = input.parse().context("parsing user key")?;
```

### 36. Hardcoding timeouts

Hardcoded timeouts (`Duration::from_secs(1)`) are inflexible:

```rust
// BAD
let cfg = Config { timeout: Duration::from_secs(1) };

// GOOD
let cfg = Config {
    timeout: Duration::from_millis(self.config.timeout_ms),
};
```

Make timeouts configurable.

### 37. Logging sensitive data

`tracing::info!(private_key = ?key, "loaded")` leaks the key:

```rust
// BAD
tracing::info!(secret = ?my_secret, "loaded");

// GOOD
tracing::info!("loaded secret");
```

Never log secrets, private keys, or PII.

### 38. Using `Arc<Mutex<T>>` everywhere

`Arc<Mutex<T>>` is a code smell. Prefer:

1. **Channels**: actors communicate via messages.
2. **Lock-free data structures**: `RwLock`, atomics.
3. **Message passing**: the actor pattern.

`Arc<Mutex<T>>` is the answer when you need shared mutable state.
But usually, you don't — actors are simpler.

### 39. Not setting `RUST_LOG` in tests

Tests without `RUST_LOG=...` produce no logs. Set up a subscriber:

```rust
#[test]
fn my_test() {
    let _ = tracing_subscriber::fmt()
        .with_test_writer()
        .with_env_filter("debug")
        .try_init();
    // ...
}
```

### 40. Comparing floats with `==`

Float equality is fragile:

```rust
// BAD
assert_eq!(latency.as_secs_f64(), 0.050);

// GOOD
assert!((latency.as_secs_f64() - 0.050).abs() < 0.001);
```

### 41. Off-by-one in view calculations

Views are 1-indexed in some protocols, 0-indexed in others:

```rust
// WRONG (off-by-one)
for view in 0..max_view { /* misses view max_view */ }

// RIGHT
for view in 1..=max_view { /* includes view max_view */ }
```

### 42. Not handling the `None` case from `recv()`

`Mailbox::Receiver::recv()` returns `None` when all senders are
dropped. Always handle:

```rust
match receiver.recv().await {
    Some(msg) => process(msg),
    None => break,        // channel closed
}
```

### 43. Dropping a `Journal` mid-write

If a `Journal` is dropped mid-write, data may be lost. Always
`sync()` before drop:

```rust
// At end of scope
journal.sync().await?;
drop(journal);
```

### 44. Using `RwLock` where `Mutex` is correct

`RwLock` is **slower** than `Mutex` for write-heavy workloads:

- `Mutex`: 1 lock holder at a time, writers wait.
- `RwLock`: multiple readers OR one writer. Overhead from tracking.

For BFT code (consensus is write-heavy), `Mutex` is usually faster.

### 45. Forgetting to spawn the actor

A constructed actor that never has `start()` called does nothing:

```rust
let actor = MyActor::new(ctx);
actor.start();    // forget this, and the actor sits idle
```

### 46. Using `HashMap` with non-deterministic iteration order

`HashMap` iteration order is non-deterministic across runs. For
deterministic behavior, use `BTreeMap` or `IndexMap`:

```rust
// BAD: order varies between runs
for (k, v) in hashmap.iter() { /* ... */ }

// GOOD: deterministic order
for (k, v) in btreemap.iter() { /* ... */ }
```

For consensus, determinism is required.

### 47. Trusting `unwrap()` on arithmetic

`u64::MAX + 1` panics in debug, wraps in release:

```rust
// BAD
let n = counter + 1;       // panics on overflow in debug

// GOOD
let n = counter.checked_add(1).ok_or(Overflow)?;
```

### 48. Forgetting to bump CHANGELOG

Bump CHANGELOG on every PR. AGENTS.md requires it.

### 49. Not running conformance tests

Conformance tests catch wire-format drift. Run before merging:

```bash
just test-conformance
```

### 50. Mixing deterministic and non-deterministic randomness

The deterministic runtime provides reproducible randomness:

```rust
// Deterministic (good for tests)
let mut rng = context.gen();    // seeded RNG

// Non-deterministic (bad for tests)
let mut rng = thread_rng();
```

For tests, always use `context.gen()`. Production can use either.

### 51. Embedding git commit hashes as strings

Use a build script to embed the version:

```rust
// In build.rs
let git_hash = Command::new("git")
    .args(["rev-parse", "HEAD"])
    .output()?.stdout;
println!("cargo:rustc-env=GIT_HASH={}", git_hash);

// In code
const VERSION: &str = env!("GIT_HASH");
```

### 52. Not testing under load

Unit tests with 4 validators pass. Production has 100. Run
integration tests at scale:

```rust
let config = estimator::Config {
    n_validators: 100,
    ..Default::default()
};
estimator::run(config).await?;
```

### 53. Mixing up epoch and view

`Epoch` and `View` are different. Epoch is the validator set
generation; View is the consensus round within an epoch:

```rust
// BAD
let round = current_view + 1;       // mixing concepts

// GOOD
let next_view = current_view.next();
let epoch_advance = current_view.is_epoch_boundary();
```

### 54. Not validating the chain tip

When joining a network, verify the chain tip:

```rust
let tip = peer.get_chain_tip().await?;
verify_finalization(&tip, &known_validator_set)?;
```

Trusting a chain tip without verification is a fork.

### 55. Forgetting `await` on futures

`future` and `await future` are different:

```rust
// BAD: doesn't wait
fn async_fn() -> impl Future<Output = ()> {
    async { /* ... */ }
}

// GOOD: waits
async fn async_fn() {
    let _ = some_future().await;
}
```

## Cross-references index — where each concept lives

This index helps you find where a concept is discussed in depth.

### Concepts in chapters 00-21

| Concept                       | Primary chapter | Secondary chapters |
|-------------------------------|-----------------|--------------------|
| BFT consensus                 | 01              | 11, 18, 20         |
| Byzantine faults              | 01              | 11, 19             |
| Codahedron (coders)           | 00              | -                  |
| Codec (encode/decode)         | 03              | 21                 |
| Commitment schemes            | 04              | 16                 |
| Conformance testing           | 03              | 21                 |
| Consensus engine              | 11              | 20, 18             |
| Context (runtime)             | 02              | 21                 |
| DKG                           | 04              | 19, 20             |
| Deterministic runtime         | 02              | 21                 |
| Ed25519                       | 04              | -                  |
| Encryption (AEAD)             | 04              | 19                 |
| FEC (forward error correction)| 07              | 20                 |
| Finalization                  | 11              | 18, 19             |
| Hashing                       | 04              | -                  |
| Journal (WAL)                 | 06              | 16, 21             |
| Mailbox                       | 15              | 21                 |
| Marshal (block delivery)      | 12              | 19                 |
| Mempool                       | 19              | -                  |
| MMR                           | 16              | 20                 |
| Network (P2P)                 | 05              | 18                 |
| Ordered Broadcast            | 09              | 20                 |
| Page cache                    | 06              | 21                 |
| Private key management        | 04              | 18                 |
| Proposer                      | 11              | 19                 |
| Resolver                      | 08              | 19                 |
| Reporter                      | 11              | 19, 21             |
| Reshare                       | 19              | 20                 |
| Simplex                       | 11              | 20                 |
| Span (tracing)                | 21              | -                  |
| Storage                       | 06              | 16, 21             |
| Threshold signatures          | 04              | 11, 20             |
| Timeouts                      | 11              | 21                 |
| Validator rotation            | 20              | 19                 |
| View changes                  | 11              | 21                 |
| VRF                           | 04              | 19                 |
| WAL                           | 06              | 16, 21             |
| ZK proofs                     | 04              | 20                 |

### Concepts by chapter

- **Chapter 00** — orientation, design philosophy, abstractions.
- **Chapter 01** — Byzantine fault tolerance, the model.
- **Chapter 02** — runtime abstraction, deterministic execution.
- **Chapter 03** — codec, wire formats, conformance.
- **Chapter 04** — cryptography (signatures, hash, DKG, ZK).
- **Chapter 05** — P2P networking, discovery.
- **Chapter 06** — storage, journals, page cache.
- **Chapter 07** — coding (erasure, ZODA).
- **Chapter 08** — resolver (data fetching).
- **Chapter 09** — broadcast (data dissemination).
- **Chapter 10** — collector (data aggregation).
- **Chapter 11** — Simplex consensus.
- **Chapter 12** — Marshal (block delivery + state sync).
- **Chapter 13** — aggregation (signature aggregation).
- **Chapter 14** — ordered broadcast.
- **Chapter 15** — actor pattern.
- **Chapter 16** — storage advanced (ADB, MMR, MMB, QMDB).
- **Chapter 17** — glue (stateful application, simulate).
- **Chapter 18** — deployer (AWS deployment).
- **Chapter 19** — examples (log, chat, bridge, sync, etc.).
- **Chapter 20** — frontier (research, papers).
- **Chapter 21** — cross-cutting patterns.

### The full picture

If you read every chapter, you have a complete picture of Commonware:

- **Why it exists** (chapter 00).
- **What it does** (chapters 01-17).
- **How you ship it** (chapters 18-19).
- **Where it's going** (chapter 20).
- **How to internalize it** (chapter 21).

The curriculum is self-contained. There is no external dependency
beyond the source code in the monorepo.

## Advanced concurrency patterns

Commonware's runtime is built on top of `tokio`, but uses an
abstraction layer so the same code runs on the deterministic
runtime (chapter 02). This section catalogs the concurrency
patterns that work across both.

### `select!` — racing multiple futures

`commonware_macros::select!` is a `tokio::select!` replacement that
works on both runtimes:

```rust
use commonware_macros::select;

select! {
    msg = receiver.recv() => handle_message(msg).await,
    _ = clock.sleep(duration) => handle_timeout().await,
    _ = stopped_signal => break,
}
```

The macro **automatically fuses** futures (you can use it in a loop
without losing messages on subsequent iterations). Tokio's select
doesn't do this; you need `FusedFuture` explicitly.

### `join!` — running futures concurrently

`futures::future::join` runs multiple futures concurrently:

```rust
use futures::future::join;

let (a, b, c) = join!(
    fetch_block(height),
    fetch_certificates(height),
    fetch_validator_set(),
);
```

All three run in parallel; you wait for all of them. Use this when
you need results from multiple sources.

### `try_join!` — failing fast

`futures::future::try_join` is like `join` but returns early if any
future errors:

```rust
use futures::future::try_join;

let (a, b) = try_join!(
    fetch_a(),
    fetch_b(),
)?;
// If either errors, we get the error and the other is dropped.
```

Use this when you need all-or-nothing semantics.

### Channels — `oneshot`, `mpsc`, `broadcast`

Three channel types cover most cases:

```rust
// oneshot: single value, single receiver
let (tx, rx) = oneshot::channel();
tx.send(value);
let value = rx.await?;

// mpsc: many values, single receiver (mailbox)
let (tx, rx) = mpsc::channel(100);
tx.send(value).await?;
while let Some(msg) = rx.recv().await { /* ... */ }

// broadcast: many values, many receivers (P2P)
let (tx, _rx1) = broadcast::channel(100);
let _rx2 = tx.subscribe();
tx.send(value);
```

`oneshot` is for request/response. `mpsc` is for actor mailboxes.
`broadcast` is for fan-out messaging.

### Cancellation — `select!` + `stop()`

Long-running tasks should be cancellable. Use `select!` with
`stopped_signal`:

```rust
async fn long_running_task(mut ctx: Context) {
    loop {
        select! {
            _ = ctx.stopped() => break,
            msg = inbox.recv() => handle(msg).await,
        }
    }
}
```

When `ctx.stop()` is called, the task breaks out cleanly. The
runtime handles aborting descendants.

### `tokio::spawn` vs `Context::spawn`

The two are similar but not identical:

| Aspect          | `tokio::spawn`         | `Context::spawn`        |
|-----------------|------------------------|-------------------------|
| Runtime         | `tokio` only           | Any `Spawner` impl      |
| Determinism     | Non-deterministic      | Deterministic if test   |
| Cancellation    | Via `JoinHandle::abort` | Via `Context::stop`     |
| Span propagation | task-local             | `Context`-aware         |

Use `Context::spawn` in protocol crates. Use `tokio::spawn` only in
runtime-specific code.

### Shared state — `Arc<Mutex<T>>` vs `Arc<RwLock<T>>` vs channels

Three options for shared state:

```rust
// Mutex: simple, but blocks readers while writer holds
let state = Arc::new(Mutex::new(my_state));

// RwLock: multiple readers OR one writer
let state = Arc::new(RwLock::new(my_state));

// Channel: actors own state, communicate via messages
let (tx, rx) = mpsc::channel(100);
// state lives in the actor; tx clones go to other actors
```

For Commonware, **channels are usually the right choice**. Mutex is
a code smell; RwLock is even more so (it's slower than Mutex for
write-heavy workloads).

### Bounded channels — backpressure

Always use bounded channels. Unbounded channels are a memory leak
waiting to happen:

```rust
let (tx, rx) = mpsc::channel(100);    // bounded to 100 messages
```

When the buffer is full, `tx.send(...).await` blocks until the
receiver catches up. This is backpressure.

### Timeouts on individual operations

For per-operation timeouts:

```rust
use commonware_runtime::Clock;

let result = clock.timeout(Duration::from_secs(5), fetch_block(height)).await;
match result {
    Ok(Ok(block)) => /* success */,
    Ok(Err(e)) => /* fetch error */,
    Err(_) => /* timeout */,
}
```

Always set timeouts on network operations. Without them, a slow
peer can stall consensus indefinitely.

## Style guide — the AGENTS.md rules

Every Commonware contributor follows these rules. The complete list
is in `AGENTS.md`; this section captures the most important.

### Use `thiserror` for library errors

Library code defines typed errors:

```rust
#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("invalid input: {0}")]
    InvalidInput(String),
    #[error("storage failed")]
    Storage(#[from] storage::Error),
}
```

### Use `anyhow` for application errors

Application code uses opaque errors:

```rust
async fn main() -> anyhow::Result<()> {
    let cfg = parse()?;
    run(cfg).await?;
    Ok(())
}
```

### Use `tracing` for logging

Structured logging with `tracing`:

```rust
tracing::info!(view = v.0, "voted notarize");
tracing::warn!(peer = %pk, "rate limit exceeded");
tracing::error!(error = %e, "consensus failed");
```

### Span names use `.` not `::`

Span names are dot-separated paths:

```rust
#[tracing::instrument(name = "consensus.voter.on_certificate")]
async fn on_certificate(...) { /* ... */ }
```

### `select!` from `commonware_macros`, not `tokio`

```rust
use commonware_macros::select;

select! { /* ... */ }
```

### Span propagation across actors

Carry span in message:

```rust
pub enum Message {
    Propose { span: tracing::Span, response: oneshot::Sender<Digest> },
}
```

### Failures are fatal for mutable storage

```rust
match storage.put(key, value) {
    Ok(_) => continue(),
    Err(e) => return Err(StorageError::Fatal),
}
```

### Use `NZUsize!` for non-zero sizes

```rust
let mailbox_size = NZUsize!(1024);   // not 1024usize
```

The non-zero wrapper enforces the constraint at the type level.

### Use `CacheRef::from_pooler` for page caches

```rust
let cache = CacheRef::from_pooler(&context, NZU16!(16_384), NZUsize!(10_000));
```

### Avoid `tokio` in protocol crates

```rust
// BAD
use tokio::sync::Mutex;

// GOOD
use commonware_runtime::Mutex;   // or futures::lock::Mutex
```

### Add conformance tests for new types

```rust
commonware_conformance::conformance_tests! {
    CodecConformance<MyType>,
}
```

### Update CHANGELOG on every PR

The CHANGELOG tracks user-facing changes. Bump it on every PR.

### Bump VERSION on releases

VERSION follows semver. Bump it for each release; the build script
embeds it.

## Quick reference — the one-page cheat sheet

A printable summary of the most important rules.

```rust
// 1. Every async function takes Context
async fn my_function(ctx: Context) -> Result<()> {
    // ...
}

// 2. Mailbox-based actors
let (tx, rx) = mailbox::new(metrics, capacity);
ctx.spawn(|c| actor.run(c, rx));

// 3. P2P channels with quotas
let (tx, rx) = network.register(id, Quota::per_second(NZU32!(10)), 256);

// 4. Typed errors
#[derive(Error, Debug)]
pub enum Error {
    #[error("...")]
    Variant,
}

// 5. Namespaces
const NAMESPACE: &[u8] = b"_COMMONWARE_<CRATE>_<OP>";

// 6. Domain separation
let msg = union(NAMESPACE, &payload);

// 7. Select with the macro
select! { msg = rx.recv() => ..., _ = clock.sleep(d) => ... }

// 8. Spans
#[instrument(name = "my.op", skip_all)]
async fn op(&self) { /* ... */ }

// 9. Storage with sync before drop
journal.sync().await?;
drop(journal);

// 10. Conformance tests
commonware_conformance::conformance_tests! {
    CodecConformance<MyType>,
}
```

## Final notes

This chapter collects the conventions that span every primitive.
Internalizing them takes time; reading the source is the fastest
way. When in doubt, look at `examples/log` — it's the minimal
end-to-end example and shows every pattern in one place.

The next step is **build something**. The exercises in earlier
chapters give specific tasks; the easiest starting point is
"modify `examples/log` to add a new metric." That single change
touches: the Reporter trait, the metrics API, the CLI args, the
runtime context. You'll have read half the library by the time it
works.

## If you only remember three things

1. **`Context` is the runtime.** Every async function takes one. Every
   operation (spawn, sleep, store, send, log, metric) goes through it.
2. **Mailboxes are how actors talk.** Bounded ready queue + overflow policy.
   Always clone `Sender`, never share by reference.
3. **Test with the deterministic runtime.** Same seed, same hash. If you find
   non-determinism, the auditor catches it.

→ **You are done.** Twenty chapters + this reference card = full curriculum.