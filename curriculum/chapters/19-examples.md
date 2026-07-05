# Chapter 19 — Examples: real apps, real composition

> Reading the `examples/` directory to see how primitives compose.

## Why examples matter

The cleanest way to understand a library is to read its examples. They
show the actual wiring: which traits get implemented, which primitives
get composed, what knobs get tuned. Reading the substrate chapters gives
you the mental models; reading the examples shows you the muscle memory.

Commonware ships seven examples. Each one focuses on a different
composition pattern.

## The seven examples

| Example | Subset of primitives | Use case |
|---|---|---|
| `log` | Simplex + P2P + Storage + Crypto | Minimal consensus app |
| `chat` | P2P + Crypto + Runtime | Encrypted group messaging |
| `bridge` | Simplex + Marshal + Crypto | Cross-chain consensus certificates |
| `sync` | Resolver + Broadcast + Storage | State sync from peers |
| `flood` | P2P + Broadcast | Gossip benchmarking |
| `estimator` | Simulated network | Mechanism performance simulation |
| `reshare` | BLS threshold + DKG | Threshold key resharing |

Plus the external examples (`alto`, `battleware`) that aren't in the
monorepo but show real production usage.

## `log` — minimal consensus app

The simplest example that uses Simplex. We touched it in chapter 01's
`main.rs` walkthrough; let me go a bit deeper.

**What it does:** validators agree on a sequence of 16-byte random
messages. Each view, the leader generates a random secret and broadcasts
its hash. All validators agree on the hash (consensus). The app tracks the
finalized sequence.

**Files** (`examples/log/src/`):
- `main.rs` — CLI parsing, network setup, engine start.
- `application/mod.rs` — your `Application` implementation.
- `application/actor.rs` — the mailbox-based actor pattern (chapter 15).
- `application/ingress.rs` — the message types.
- `application/reporter.rs` — finalization notifications.
- `gui.rs` — a Ratatui-based terminal UI showing consensus progress.

**The composition** (from `main.rs:148-231`):
1. Tokio runtime with a storage directory.
2. P2P `discovery` network (chapter 05).
3. Register three channels: votes (0), certificates (1), resolver (2).
4. Application actor: generates random secrets on propose, hashes on
   verify.
5. Simplex engine: ties it all together.

**What it teaches you:**
- The exact wiring for a working consensus app.
- How to set up P2P discovery with bootstrappers.
- How to use the actor pattern from chapter 15.
- How the `certify` hook defaults to "always true" when you don't need it.

## `chat` — encrypted P2P messaging

**What it does:** friends send each other encrypted messages. No
consensus. No storage. Just P2P + cryptography.

**Files** (`examples/chat/src/`):
- `main.rs` — CLI, network setup, message loop.
- `handler.rs` — receives and displays messages.
- `logger.rs` — TUI logger.

**What it teaches you:**
- How to use P2P without consensus.
- The simplest possible authenticated encryption flow.
- Friend discovery (chapter 05) without a blockchain.

## `bridge` — cross-chain consensus certificates

**What it does:** an L1 emits consensus certificates (BLS threshold sigs
on block hashes). A "bridge" chain finalizes those certificates and
exposes them to consumers (other chains, light clients).

**Files** (`examples/bridge/src/`):
- `main.rs` — orchestrator.
- `application/` — the bridge application.
- `bin/` — separate binaries for source chain, sink chain, relayer.
- `types.rs` — certificate types.

**What it teaches you:**
- Cross-chain interoperability via succinct certificates.
- BLS threshold sigs in production (chapter 04).
- How Marshal + Simplex combine to deliver certificates to applications.

## `sync` — state sync from peers

**What it does:** a "server" has a database (say, a key-value store). A
"client" connects, asks for the data behind key K, the server sends it
(with a Merkle proof), the client verifies and applies.

**Files** (`examples/sync/src/`):
- `main.rs` — server + client setup.
- `databases/` — the database type.
- `net/` — the network protocol.

**What it teaches you:**
- How to use Resolver (chapter 08) for state sync.
- How to use Broadcast (chapter 09) for serving.
- How to integrate authenticated data structures (chapter 16) into a
  sync protocol.

## `flood` — gossip benchmarking

**What it does:** spam random messages to all peers, measure bandwidth,
throughput, message delivery.

**What it teaches you:**
- How to use Broadcast (chapter 09) at scale.
- How to measure throughput in the simulated network (chapter 02).

## `estimator` — mechanism performance simulation

**What it does:** runs a full consensus instance in the simulated
network, varying latency / jitter / packet loss, measures finalization
latency, throughput, message count.

**What it teaches you:**
- How to use the simulated network (chapter 02) for serious benchmarks.
- How to wire up an instrumented consensus engine for measurement.
- How to interpret the results (the `docs/blogs/` files do this for
  Alto's 20% perf win from buffered signatures).

## `reshare` — threshold secret resharing

**What it does:** a committee holds shares of a threshold secret. A new
committee is formed (e.g., validator set rotation). Run the DKG resharing
protocol to give the new committee shares of the SAME secret (no key
generation, just redistribution).

**Files** (`examples/reshare/`):
- The orchestrator that runs the public/secret resharing protocol.

**What it teaches you:**
- Threshold DKG + resharing in production.
- Validator set rotation without losing the threshold key.
- BLS12-381 threshold sigs across epochs (chapter 04).

## Reading order for examples

If you're building a real app, here's the recommended reading order:

1. **`log`** — see the minimum wiring for a consensus app.
2. **`chat`** — see P2P without consensus.
3. **`bridge`** — see consensus + marshal + certificates.
4. **`sync`** — see resolver + broadcast.
5. **`estimator`** — see benchmarking patterns.
6. **`flood`** — see gossip at scale.
7. **`reshare`** — see threshold key lifecycle.

Then read the external `alto` source for the production-grade example.

## A walkthrough of `log/main.rs`

Let's trace the critical parts (lines 100-245 from chapter 01):

```rust
// 1. Set up the runtime
let runtime_cfg = tokio::Config::new().with_storage_directory(storage_directory);
let executor = tokio::Runner::new(runtime_cfg);

// 2. Set up P2P discovery
let p2p_cfg = discovery::Config::local(signer, namespace, listen_addr, ...);

// 3. Inside the executor:
executor.start(async |context| {
    // 4. Initialize the network
    let (mut network, mut oracle) = discovery::Network::new(context.child("network"), p2p_cfg);

    // 5. Provide authorized peers
    oracle.track(0, validators.clone());

    // 6. Register the consensus channels
    let (vote_sender, vote_receiver) = network.register(0, Quota::per_second(NZU32!(10)), 256);
    let (certificate_sender, certificate_receiver) = network.register(1, Quota::per_second(NZU32!(10)), 256);
    let (resolver_sender, resolver_receiver) = network.register(2, Quota::per_second(NZU32!(10)), 256);

    // 7. Initialize the application
    let application = application::Application::new(context.child("application"), ...);

    // 8. Initialize the consensus engine
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
        // ... timing and buffer config ...
    };
    let engine = simplex::Engine::new(context.child("engine"), cfg);

    // 9. Start everything
    application.start();
    network.start();
    engine.start(
        (vote_sender, vote_receiver),
        (certificate_sender, certificate_receiver),
        (resolver_sender, resolver_receiver),
    );

    // 10. Block on the GUI
    let gui = gui::Gui::new(context.child("gui"));
    gui.run().await;
});
```

That's the entire wiring. Every line maps to a chapter:

- Line 1: chapter 02 (Tokio runtime).
- Line 2: chapter 05 (P2P discovery).
- Line 4: chapter 02 (runtime context).
- Line 5: chapter 05 (Manager::track).
- Line 6: chapter 05 (channel registration with quotas).
- Line 7: chapter 11 (your Application).
- Line 8: chapter 11 (Simplex engine).
- Line 9: chapter 15 (actor pattern).
- Line 10: just blocks on user input.

Sixteen chapters of substrate compressed into ten lines of code.

## Alto — a real blockchain on Commonware

The single most important example is the one that isn't in the
monorepo: **[Alto](https://github.com/commonwarexyz/alto)**, the
Commonware team's reference blockchain. Alto is **what the library
looks like in production** — every constraint the docs hint at,
exercised for real.

### What Alto does

Alto is a minimal L1: users submit transactions, validators agree on
ordering via Simplex, the chain finalizes every ~200ms. There's no
smart contract VM — transactions are simple token transfers. The
deliberate minimalism lets Alto be the **canary**: every optimization
the team publishes has been benchmarked end-to-end against Alto, and
the numbers come from a real running network rather than `estimator`.

### Alto's architecture, briefly

```
┌──────────────────────────────────────────────────────────┐
│                Network ingress (gRPC/RPC)                │
└───────────────────────────┬──────────────────────────────┘
                            │ tx submission
                            ▼
┌──────────────────────────────────────────────────────────┐
│            Mempool (dedup, ordering, fee auction)         │
└───────────────────────────┬──────────────────────────────┘
                            │ ordered txs
                            ▼
┌──────────────────────────────────────────────────────────┐
│           Block proposer (leader per view)                │
└───────────────────────────┬──────────────────────────────┘
                            │ proposed block
                            ▼
┌──────────────────────────────────────────────────────────┐
│        Simplex consensus (Simplex + buffered sigs)        │
└───────────────────────────┬──────────────────────────────┘
                            │ finalized blocks
                            ▼
┌──────────────────────────────────────────────────────────┐
│          Marshal (block + certificate archive)           │
└───────────────────────────┬──────────────────────────────┘
                            │ finalized blocks
                            ▼
┌──────────────────────────────────────────────────────────┐
│     QMDB state (current account balances + proofs)       │
└──────────────────────────────────────────────────────────┘
```

### Performance numbers

Published by the team (May 2025, `docs/blogs/buffered-signatures.html`):

| Metric                  | Before buffered sigs | After | Improvement |
|-------------------------|----------------------|-------|-------------|
| Block time              | ~250 ms              | ~200 ms | 20%       |
| Finality latency        | ~375 ms              | ~300 ms | 20%       |
| CPU usage (per validator)| 100% baseline       | 35%   | 65%       |

The 65% CPU reduction is the headline result: **buffered BLS
signatures** amortize verification across votes in a 50ms window.
Alto is the production witness to the lab simulation in `estimator`.

### Why Alto matters for the curriculum

You cannot learn Commonware from the docs alone — the docs describe
mechanisms in isolation. Alto shows **how a working system composes**
all of them. When the chapter on Simplex says "the proposer generates
a block each view," Alto's `block_proposer.rs` shows the actual code
that does it. When the chapter on QMDB says "current-state proofs
are O(1)," Alto's `state.rs` proves it.

In the reading order below, treat Alto as the **last** example (after
the monorepo's seven) — it's where everything you learned clicks
into place.

## Battleware — VRF, timelock encryption, MMRs

[Battleware](https://github.com/commonwarexyz/battleware) (mentioned
in `docs/blogs/only-time-will-tell.html`) is an onchain game that
uses three Commonware primitives in creative combination:

1. **VRF** (verifiable random function, on the Ed25519 signature
   scheme) — decides which player gets to act each round. Provably
   fair randomness without a trusted oracle.
2. **Threshold timelock encryption** (`docs/blogs/only-time-will-tell.html`,
   BTE/Batch Pari) — encrypts a player's move so opponents can't see
   it until the round ends. Cryptographic anti-cheat.
3. **MMRs** (chapter 16) — commit a log of actions compactly, with
   small proofs for any prior round.

Battleware is short — fewer than 1k lines — and illustrates that
Commonware primitives compose beyond consensus. The lessons are
**don't invent new crypto** (use `cryptography::bls12381` for the
BTE part), and **keep your state in MMRs** when you need provable
history.

## `log` — line-by-line

Reading the source is faster than reading about it. `examples/log/src/`
has six files. Each plays a specific role.

### `main.rs` — the orchestrator

`main.rs` does what we covered earlier — parse CLI, set up runtime,
wire network, start application and engine, run GUI. **Twelve lines
of real logic** (lines 207-244 of `main.rs:231-244`):

```rust
application.start();        // start the actor loop
network.start();            // start P2P
engine.start(               // start Simplex
    (vote_sender, vote_receiver),
    (certificate_sender, certificate_receiver),
    (resolver_sender, resolver_receiver),
);
gui.run().await;            // block on TUI
```

The rest of the 250 lines is **idiomatic glue**: clap parsing,
config loading, log setup, error handling around the orchestrator.

### `application/mod.rs` — the trait

The application module exposes the `Application` type plus the
`Scheme` (BLS threshold signature scheme used for voting). The
shape:

```rust
pub mod actor;
pub mod ingress;
pub mod reporter;

pub use actor::Application;
pub use ingress::{Propose, Verify};
pub use reporter::Reporter;

pub fn genesis<D: Digest>() -> Floor<D> {
    // ... returns the genesis Floor (empty)
}

pub struct Scheme;
impl Scheme {
    pub fn signer(...) { /* ed25519 single-signer voting */ }
    pub fn threshold(...) { /* BLS threshold voting */ }
}
```

The `Scheme::threshold` variant is what unblocks production: BLS
threshold sigs instead of per-validator ed25519 votes. The example
defaults to `signer` for simplicity (no DKG needed), and you flip
to `threshold` for the real setup.

### `application/actor.rs` — the mailbox pattern

The application's actor implements the `Automaton` and `Relay` traits
(chapter 11). Three mailboxes drive its loop:

```rust
pub struct Application {
    context: Context,
    inner: Mutex<Inner>,
    propose_mailbox: Mailbox<ProposeRequest>,
    verify_mailbox: Mailbox<VerifyRequest>,
    reporter_mailbox: Mailbox<Report>,
}

struct Inner {
    scheme: Scheme,
    hasher: Sha256,
    public: Public,
    secrets: HashMap<Digest, [u8; 16]>,
}
```

The actor:

1. Listens on `propose_mailbox` for `ProposeRequest` — generates a
   random 16-byte secret, hashes it, stores the `(hash → secret)`
   pair, returns the hash to consensus via the `oneshot` channel.
2. Listens on `verify_mailbox` for `VerifyRequest` — looks up the
   hash, returns whether we have a matching secret (the
   "verifier-of-record" check from chapter 11).
3. Listens on `reporter_mailbox` for `Report` — emits metrics for
   finalization progress (view, height, age).

This is the **canonical Commonware actor** (chapter 15): mailbox-driven,
non-blocking, no locks held across `.await`.

### `application/ingress.rs` — message types

```rust
pub struct ProposeRequest {
    pub context: Context,
    pub responder: oneshot::Sender<Digest>,
}

pub struct VerifyRequest {
    pub context: Context,
    pub payload: Digest,
    pub responder: oneshot::Sender<bool>,
}
```

The `Context` here is the runtime context for instrumentation
(chapter 02). The `oneshot::Sender` is the return channel. Together
they form the **request/response over mailbox** pattern from
chapter 15: the caller doesn't block, but gets a future to await
when it needs the answer.

### `application/reporter.rs` — finalization notifications

```rust
pub struct Reporter { ... }

impl crate::Reporter for Reporter {
    async fn report(&mut self, activity: Activity) {
        match activity {
            Activity::Finalization { height, view, hash } => {
                tracing::info!(height, view, ?hash, "finalized");
                self.metrics.finalizations.inc();
            }
            Activity::Participation { height, view } => {
                self.metrics.participations.inc();
            }
            ...
        }
    }
}
```

`Reporter` is the third leg of the Simplex contract (chapter 11):
`Automaton` decides what to propose/verify, `Relay` decides what to
forward, `Reporter` is notified of consensus progress. The `Reporter`
is how your application learns about finalized blocks.

### `gui.rs` — the TUI

`gui.rs` is a Ratatui-based TUI showing live consensus state: view
number, height, peers connected, rate of finalization. Pure for the
example — but the pattern (separate visualization from consensus) is
how you ship a healthy production deployment: dashboards should not
be the only way to debug.

## `chat` — encrypted group messaging

`examples/chat/src/main.rs` (~200 lines) implements an end-to-end
encrypted group chat. There's no consensus — just P2P plus crypto.

### Architecture

```
User A:    ┌──────────┐                 User B:    ┌──────────┐
           │  chat    │                            │  chat    │
           │ client   │                            │ client   │
           └────┬─────┘                            └────┬─────┘
                │                                       │
                │  ciphertext via P2P                   │
                │  (encrypted by recipient pubkey)      │
                │                                       │
                ▼                                       ▼
           ┌──────────────────────────────────────────────┐
           │            P2P (chapter 05)                  │
           └──────────────────────────────────────────────┘
```

### Message flow

1. User A composes a message.
2. The client encrypts it with **User B's public key** (the P2P
   handshake already provided this — chapter 04).
3. The ciphertext is sent over P2P to User B.
4. User B's client decrypts with its private key and displays.

The encryption uses the `chacha20-poly1305` AEAD with a key derived
from the shared P2P session key. Each `chat` instance runs as a
peer — bootstrap to the same set of friends, and you have a group.

### What it teaches you

- P2P without consensus is enough for low-fanout messaging.
- The `discovery::Manager` from chapter 05 also serves as a friend
  directory.
- AEADs over the P2P session are **better than end-to-end** for
  peer-to-peer groups (you trust the running session, not the
  long-term key).

## `bridge` — cross-chain consensus certificates

`examples/bridge/` (~500 lines) ships three binaries:

1. **`source`** — runs Simplex consensus, emits BLS threshold
   signatures on its own block hashes.
2. **`relayer`** — listens for `Finalization` messages from the
   source, submits them to the sink.
3. **`sink`** — accepts BLS threshold signatures, includes them in
   its own consensus, exposes a verifyable cross-chain root.

### Why this matters

Real cross-chain messaging (Optimism, Wormhole, LayerZero) all
share the same skeleton: **a succinct proof that "X happened on
chain A" signed by the validators of A, validated cheaply on chain
B.** BLS threshold sigs are the optimal primitive for this — one
signature, one verification, ~100 bytes regardless of validator count.

### The cert types

```rust
// examples/bridge/src/types.rs
pub struct SourceCertificate {
    pub height: u64,
    pub root: Digest,           // the source chain's state root at height
    pub signature: BlsSignature, // threshold sig over (height, root)
}

pub struct SinkInclusion {
    pub certificate: SourceCertificate,
    pub sink_height: u64,       // the sink chain's height that included it
    pub sink_signature: BlsSignature,
}
```

The bridge is simply: relayer watches source `Finalization`s,
forwards to sink, sink includes them. The sink's consumers (other
chains, light clients) read `SinkInclusion` and verify the
`SourceCertificate` against the source's known BLS public key.

## `sync` — server-client state synchronization

`examples/sync/` (~400 lines) implements a server that holds a
key-value database and a client that fetches entries with proofs.

### The pieces

- **`databases/`** — the `Database` trait implementation. Backed by
  `storage::adb::current` (chapter 16) so every entry has a Merkle
  proof.
- **`net/`** — the wire protocol. Uses `Resolver` (chapter 08) for
  client-side: "I want the proof for key K." Uses `Broadcast`
  (chapter 09) for server-side: "I'm sending you entry K and its
  proof."
- **`main.rs`** — spawns either server or client based on CLI flags.

### The flow

```
client: generate_session_key()
        send "hello" with session public key
server: send "challenge" with nonce
client: send "auth" with sign(session_key, challenge)
server: verify, then accept data requests
client: request proof(K)
server: respond with (value, merkle_proof_to_root(K))
client: verify proof against known server root
```

This is essentially TLS handshake + authenticated data structures
+ the Resolver pattern from chapter 08. **The proof is what makes
this useful in adversarial contexts** — without proof verification,
a server could send you whatever bytes it wants.

## `flood` — gossip benchmarking

`examples/flood/src/main.rs` (~200 lines). Spams peers with random
messages and measures throughput.

### What gets measured

- **Bandwidth** (per-validator egress) — Prometheus counter.
- **Message delivery latency** — histogram, from broadcast send to
  receiver callback.
- **Duplicate ratio** — how often the same message arrives twice
  (Broadcast is best-effort; dedup is the receiver's job).

### The harness

```rust
let cfg = flood::Config {
    message_size: 1024,        // bytes per message
    target_throughput: 1000,   // messages per second
    duration: Duration::from_secs(60),
};
let results = flood::run(cfg).await?;
println!("delivered: {} msgs in {:?}", results.delivered, results.elapsed);
println!("bandwidth:  {:.2} MB/s", results.bandwidth_mbps());
println!("duplicate ratio: {:.2}%", results.dup_ratio * 100.0);
```

### What it teaches you

- Broadcast (chapter 09) is **rate-limited** at the channel level
  (`Quota::per_second`). Tune this to your bandwidth budget.
- Duplicate suppression matters: every flood test in `estimator` runs
  with dedup enabled (Bloom filter on hashes) to measure "useful"
  bandwidth.

## `estimator` — mechanism performance simulation

`examples/estimator/src/main.rs` (~300 lines). Runs a full Simplex
instance in a simulated network and measures the outcome.

### What it does

```rust
let cfg = estimator::Config {
    n_validators: 100,
    latency: Duration::from_millis(50),
    jitter: Duration::from_millis(10),
    packet_loss: 0.001,                       // 0.1%
    byzantine_fraction: 0.0,                  // honest for control run
    block_payload_size: 1024,                 // bytes
    view_timeout: Duration::from_millis(1000),
    duration: Duration::from_secs(60),
    sample_interval: Duration::from_millis(100),
};
let results = estimator::run(cfg).await?;
```

### The metrics

- **Finalization latency** (p50, p99, max) — wall-clock from proposal
  to finalization.
- **Throughput** — blocks per second.
- **View changes per epoch** — how often the consensus had to skip
  a leader.
- **Network messages** — total count broken down by type.

### Why this is the unit test for new mechanisms

When the team designed buffered sigs, they ran `estimator` with and
without, over thousands of network conditions (latencies 10-500ms,
jitter 0-100ms, loss 0-10%, validator count 4-200). The Alto team
saw the simulated 20% block time improvement **before** shipping any
binary changes.

## `reshare` — threshold key resharing

`examples/reshare/src/main.rs` (~300 lines). Demonstrates DKG
resharing: an old committee holds shares of secret S, a new committee
is formed, the new committee ends up with shares of the **same** S
(no regeneration).

### The setup

```rust
// Old committee holds (s_1, ..., s_n) where any t can recover S
let old_committee = (0..n_old).map(|i| share_i).collect();
// New committee will hold (s'_1, ..., s'_n')
let new_committee_members = pick_new_set(...);

// Run resharing
let new_shares = reshare(old_committee, new_committee_members).await?;
// new_shares[i] is the share for member i of the new committee
```

### Why this matters

DKG key generation (chapter 04) is expensive — a multisignature
ceremony. **Resharing avoids it.** As long as at least `t+1` of the
old committee are honest, you can rotate the committee without
regenerating the key. Alto uses this to onboard new validators
without downtime (the network keeps producing blocks while the
resharing happens).

### The flow

1. Old committee members each ` Feldman-commit` their share
   (BLS pubkey of the share, chapter 04).
2. Each new member requests a share from each old member.
3. Old members verify the new member is "in" (against a Schnorr
   proof) and respond with a fresh share of `S`.
4. New member aggregates, gets their new share.
5. Once all new members have ≥ `t` shares, the new committee is
   active.

## Patterns the examples teach

Reading all seven, patterns emerge. They're not enforced — they're
the natural shape of "doing Commonware well."

### Composition over inheritance

There is **no class hierarchy** in Commonware. The examples compose
**behaviors** by trait impls:

```rust
impl Automaton for Application { ... }
impl Relay for Application { ... }
impl Reporter for Reporter { ... }
```

Each trait represents one capability the consensus engine needs.
Your application satisfies them by being the runtime + state, not by
subclassing anything. This is **the central design choice** of
Commonware: primitives are types, behaviors are traits, composition
plugs them together.

### The "channels + quotas" pattern

Every P2P channel in every example has rate limiting:

```rust
let (sender, receiver) = network.register(
    channel_id,               // unique per channel
    Quota::per_second(NZU32!(10)),  // 10 msgs/sec
    256,                      // backpressure buffer
);
```

The `10 msgs/sec` is the per-peer rate. The `256` is the receiver's
mailbox size. **These are non-defaults you must think about.**
Too loose: garbage data DoSes the consensus. Too tight: legitimate
messages get dropped under burst.

### Non-blocking everything

Look at every `.await` in the examples. There are **no locks held
across `.await`** and **no blocking I/O**. The actor pattern (chapter
15) ensures this: actors message each other, never call each other.

This is a hard rule for Commonware code. The deterministic runtime
gets it for free (no real I/O); production code gets it by being
disciplined.

### Graceful defaults, explicit overrides

`Config::local` for P2P in `log`. `Scheme::signer` for voting in
`log`. `Ordered` strategy for Simplex by default. Every primitive has
a "happy path" config and a "tune every knob" config.

The examples take the happy path. Production code begins with the
happy path and only deviates when measurement demands it.

### Trivial instrumentation

Look at every file. There's a `tracing::info!` or `tracing::debug!`
at every meaningful boundary. The motivation isn't debug-print
hygiene — it's so you can later turn on `RUST_LOG=debug` and see
exactly what the consensus was doing in a problematic moment.

## Rust patterns — how the examples are written

Three patterns to internalize when reading or extending.

### `clap` for CLI parsing

Every example uses clap's builder API:

```rust
let matches = Command::new("commonware-log")
    .about("generate secret logs and agree on their hash")
    .arg(Arg::new("bootstrappers").long("bootstrappers")...)
    .arg(Arg::new("me").long("me").required(true))
    .arg(Arg::new("participants").long("participants").required(true)...)
    .arg(Arg::new("storage-dir").long("storage-dir").required(true))
    .get_matches();
```

The pattern is `Command::new(name)` + `Arg::new("flag").long("flag")`
+ `.get_matches()`. Commonware wraps this in a thin helper internally
(see `p2p::config::parse_key` for parsing `key@addr` style values).
For your own apps, **clone this idiom** — flag-based CLIs are easier
to script than positional.

### `tracing` for structured logging

Every event is structured:

```rust
tracing::info!(key = ?signer.public_key(), "loaded signer");
tracing::debug!(view = view.0, height = h, "voted");
tracing::warn!(peer = %peer, "rate limit exceeded");
tracing::error!(err = %e, "instance launch failed");
```

Fields are typed (`?` for Debug, `%` for Display, plain for primitives).
The output goes through a `tracing-subscriber` that's typically
configured for JSON in production (shippable to Loki, Datadog, etc.)
and pretty-colored in development.

**The AGENTS.md rule is sharp here**: spans are work-units, not
runtime context; carry them across actor boundaries; record only
at root on errors. The examples follow this implicitly and so should
your code.

### `thiserror` + `anyhow` for errors

Two patterns:

```rust
// Library code (in primitives) — explicit, typed
#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("invalid proposal: {0}")]
    InvalidProposal(String),
    #[error("journal error")]
    Journal(#[from] storage::journal::Error),
    #[error("consensus not ready")]
    NotReady,
}

// Application code (in examples) — opaque, ergonomic
async fn main() -> anyhow::Result<()> {
    let cfg = parse_config().context("loading config")?;
    run(cfg).await.context("running app")?;
    Ok(())
}
```

**Library code** uses `thiserror` because callers want to match on
the error variants. **Application code** uses `anyhow::Result`
because the caller is `?`-chaining all the way up and presenting a
nice message. Mixing them is the wrong move — `anyhow` loses type
info across the API boundary.

## Alto in full — production architecture

The Alto codebase is the canonical reference for how Commonware primitives
compose in production. The previous section gave the headline numbers; here
is the deep dive.

### Alto's design choices

Alto's design is **deliberately minimal**. A token-transfer chain, no smart
contracts, no EVM, no WASM. The minimalism has a purpose: it lets Alto be the
**canary** for every optimization the team publishes. When the buffered-
signatures blog claimed 20% block time reduction, Alto ran the optimization
end-to-end and reported real numbers. When MMB claims constant-size proofs,
Alto integrated it and the team saw the disk savings on a real chain.

The design choices that follow from "minimal canary":

1. **No virtual machine.** Transaction validation is a static type check
   (signature scheme, format, balance). No general computation.
2. **Single BLS scheme.** All validators use BLS12-381 with the same
   threshold configuration. No per-app signature choice.
3. **Fixed validator set for now.** Validator rotation is a roadmap item,
   not a v1 feature.
4. **No bridges (yet).** The `bridge` example shows the pattern; Alto will
   integrate it later.

### The data flow in detail

When a user submits a transaction, the flow is:

```
User                    Alto Node                   Other Validators
 │                          │                              │
 │ POST /tx (signed)        │                              │
 ├─────────────────────────>│                              │
 │                          │ validate signature           │
 │                          │ check nonce                  │
 │                          │ insert into mempool          │
 │                          │ broadcast tx to peers       │
 │                          ├─────────────────────────────>│
 │                          │                              │
 │                          │ (mempool sync)               │
 │                          │<─────────────────────────────┤
 │                          │                              │
 │                          │ leader of view v selects txs │
 │                          │ builds proposed block        │
 │                          │ broadcasts proposal          │
 │                          ├─────────────────────────────>│
 │                          │                              │
 │                          │ (proposal propagation)       │
 │                          │<─────────────────────────────┤
 │                          │                              │
 │                          │ validators vote notarize     │
 │                          ├─────────────────────────────>│
 │                          │                              │
 │                          │ buffered sigs aggregate      │
 │                          │ BLS aggregate over ~50ms     │
 │                          │ emit aggregate sig           │
 │                          ├─────────────────────────────>│
 │                          │                              │
 │                          │ 2f+1 aggregate sigs          │
 │                          │ => block is notarized        │
 │                          │                              │
 │                          │ finalize at view v+1         │
 │                          │ apply block to QMDB state    │
 │                          │ write notarization to journal│
 │                          │                              │
 │ 202 Accepted             │                              │
 │<─────────────────────────┤                              │
```

Each step has a **failure mode** the design handles:

- **Invalid signature**: rejected before mempool insertion.
- **Mempool desync**: validators gossip all mempool txs; eventually
  consistent.
- **Leader Byzantine**: skip certificate (chapter 11) advances to next view.
- **Block dissemination failure**: resolver (chapter 08) fetches from peers
  on demand.
- **Byzantine vote**: signature verification rejects it.

### What makes Alto fast

The 200ms block time is the result of several stacked optimizations:

1. **Buffered BLS signatures** (chapter 20): aggregate ~5 votes per 50ms
   window. Saves 60% of CPU on signature work.
2. **Coded block dissemination** (chapter 07): validators gossip shards
   of the block via ZODA. Leader egress is `O(block)`, not `O(n × block)`.
3. **Small block size**: 256 KB max. Allows dissemination in a single
   RTT to most peers.
4. **High-bandwidth validators**: minimum 100 Mbps network link. Real
   validators have gigabit.
5. **Skip certificates** (Simplex, chapter 11): Byzantine leaders don't
   stall the chain. The chain advances to the next view in one extra RTT.
6. **In-memory hot state** (QMDB, chapter 16): current state is in memory
   with a 32 MB heap; disk is for the log only.
7. **Pre-built proposals**: the leader pre-fetches txs from mempool
   during the previous view's finalization, so proposal generation is
   O(1) when the leader's turn arrives.

The cumulative effect: 200ms blocks with `n = 100` validators, ~50ms
RTT network, and 35% CPU per validator.

### The source walkthrough

Alto's source is structured as:

```
alto/
├── application/      # the consensus "Application" trait impl
│   ├── actor.rs
│   ├── ingress.rs
│   ├── reporter.rs
│   └── scheme.rs
├── block/            # the Block type and validation
│   ├── block.rs
│   ├── commitment.rs
│   └── verifier.rs
├── consensus/        # Simplex + Marshal glue
│   ├── marshal.rs
│   └── finalize.rs
├── mempool/          # transaction ordering + dedup
│   ├── mempool.rs
│   └── priority.rs
├── network/          # P2P configuration
│   └── p2p.rs
├── state/            # QMDB-backed current state
│   ├── state.rs
│   └── apply.rs
├── storage/          # journal configuration
│   └── journal.rs
├── api/              # user-facing RPC endpoints
│   ├── server.rs
│   └── handlers.rs
├── bin/              # binary entry points
│   ├── node.rs
│   └── admin.rs
└── main.rs
```

The key file for understanding Alto is
`alto/src/consensus/finalize.rs`. This is the `Reporter` implementation
that decides what to do when consensus reports a finalization. It's
~150 lines and contains the entire logic of "block is finalized, apply
it to state, write notarization to journal, ack to marshal, emit
metrics."

```rust
impl Reporter for AltoReporter {
    async fn report(&mut self, activity: Activity) {
        match activity {
            Activity::Finalization { height, view, hash } => {
                let block = self.marshal.get_block(hash).await?;
                self.state.apply(&block).await?;
                self.journal.append(finalization_cert(&activity)).await?;
                self.marshal.ack(height).await?;
                self.metrics.finalizations.inc();
            },
            Activity::Fault(fault) => {
                self.slashing.on_fault(fault).await?;
            },
            // ...
        }
    }
}
```

That's the entire flow in 15 lines. The complexity lives in `state.apply`,
`journal.append`, and `marshal.get_block` — and each is a thin wrapper
around a Commonware primitive.

### Performance numbers from production

The team publishes performance numbers from Alto on the benchmarks page.
For the standard 100-validator configuration, 50ms RTT, 0.1% packet loss:

| Metric                          | Value     |
|---------------------------------|-----------|
| Block time                      | 200 ms    |
| Finality latency                | 300 ms    |
| Throughput (single-shard)       | 5,000 TPS |
| CPU usage per validator         | 35%       |
| Memory (validator process)      | 256 MB    |
| Disk (journal, 1B transactions) | 80 GB     |
| Network egress (per validator)  | 5 MB/s    |

The numbers assume reasonable hardware (c6i.2xlarge) and a healthy network.
Under stress (cross-region, packet loss), throughput drops but finality
latency stays sub-second.

## Battleware in full — VRF, TLE, MMRs

Battleware is the second external example, and it's the most
**creative** use of Commonware primitives. The previous section
introduced it; here's the deep dive.

### What Battleware is

Battleware is an on-chain turn-based game. Players take turns
performing actions; each action is committed to a chain so players can
verify the game history. The interesting part: **all player moves are
hidden until the round ends**, so no player can react to another's
move.

This requires three cryptographic primitives in combination:

1. **VRF** for fair leader election (who acts each round).
2. **Threshold Timelock Encryption** for hiding moves until reveal.
3. **MMRs** for compact commitment to move history.

### The architecture

```
Game Round R:
1. VRF elects player P to act this round
2. Player P computes their move M
3. Player P encrypts M under (round_R, threshold_pubkey)
4. Player P submits encrypted M to the chain
5. Chain validators commit encrypted M to MMR (history grows)
6. Round R ends when timer expires
7. Threshold committee emits decryption key for (round_R)
8. Player M decrypts; move is revealed
9. Game state transitions based on M
10. Round R+1 begins
```

The VRF ensures **provable fairness**: the leader of each round is
deterministically derived from a public seed and verifiable. No
trusted oracle.

The TLE ensures **hidden information**: moves can't be observed until
the round ends. No MEV-style front-running.

The MMR ensures **compact history**: every prior move has a small
inclusion proof. Players can verify any past game state.

### VRF — verifiable random function

A VRF takes a secret key `sk` and an input `x`, and outputs:

- A random value `y = VRF(sk, x)`
- A proof `π` that `y` is the correct output

Anyone with the public key `pk` can verify: `Verify(pk, x, y, π) = true`.

Battleware uses an **Ed25519-based VRF** (chapter 04). The leader
election is:

```rust
let seed = previous_block_hash;          // deterministic
let vrf_input = (round_number, seed);
let (vrf_output, vrf_proof) = vrf_eval(player_sk, vrf_input);
let leader_index = vrf_output[0] as usize % num_players;
// emit vrf_proof as part of the block; validators verify
```

The proof is included in the block. Any verifier can check that the
correct leader was elected. No player can manipulate who leads.

### Threshold Timelock Encryption (TLE)

TLE is a cryptographic primitive that lets a committee encrypt a
message such that:

- It cannot be decrypted until a **time** (round number) is reached.
- Once the time is reached, **any `t+1` of `n`** committee members
  can decrypt.

Commonware's `cryptography::bls12381::tle` implements this via:

- **Identity-Based Encryption (IBE)**: identity is `(round_number)`.
  The encryption key is derived from the identity.
- **Threshold key extraction**: the master secret key is split into
  `n` shares via Shamir's secret sharing. Each committee member has
  one share.
- **Time-lock via VDF** (optional): adds a verifiable delay so
  decryption takes ~round_duration even with all shares.

The encryption:

```rust
use commonware_cryptography::bls12381::tle;

let encrypted = tle::encrypt(
    &threshold_public_key,    // master pubkey
    round_number,             // identity (the "time")
    &move_bytes,              // plaintext
)?;
```

The decryption (after round ends):

```rust
let shares: Vec<(usize, tle::Share)> = committee_members
    .iter()
    .enumerate()
    .map(|(i, member)| (i, member.decrypt_share(round_number, &encrypted)))
    .collect();

let plaintext = tle::combine_shares(&threshold_public_key, round_number, &encrypted, &shares)?;
```

This is the **BTE construction** from chapter 20. Battleware is the
first game to use it in production.

### MMR — compact game history

Battleware uses `storage::mmr::Mmr` (chapter 16) to commit to all
prior moves:

```rust
let mut mmr = Mmr::new();
for round in 0..current_round {
    let move_commitment = hash(move_history[round].commitment);
    mmr.add(&move_commitment).await?;
}
// At any point, prove "round R had move X":
let proof = mmr.proof(round_index).await?;
let root = mmr.root();
```

The MMR proof is `O(log N)` where `N` is the number of rounds. For a
game with 1000 rounds, the proof is ~10 hashes (~320 bytes).

### What Battleware teaches

The lessons from Battleware:

1. **Don't invent crypto.** Use Commonware's primitives. The TLE
   implementation is ~500 lines of carefully audited code; re-implementing
   it is a recipe for vulnerabilities.
2. **Compose primitives creatively.** Battleware uses VRF + TLE + MMR
   in a way no single primitive was designed for. The composability
   of Commonware's design is the point.
3. **Keep state in MMRs.** Any state that needs to be verifiable by
   multiple parties (game history, audit trails, voting records)
   belongs in an MMR.
4. **Threshold crypto needs DKG.** The TLE threshold key must be
   generated via DKG (chapter 04). Battleware uses Pedersen DKG at
   game start.

## `log` example in full — line by line

The previous section gave the architectural overview. Here is the
ground-level walkthrough.

### The file layout

```
examples/log/
├── Cargo.toml
├── src/
│   ├── main.rs                # the orchestrator (~250 lines)
│   ├── application/
│   │   ├── mod.rs             # the Application type + Scheme (~50 lines)
│   │   ├── actor.rs           # the actor loop (~70 lines)
│   │   ├── ingress.rs         # message types + mailbox policy (~90 lines)
│   │   └── reporter.rs        # finalization notifications (~40 lines)
│   └── gui.rs                 # the Ratatui TUI (~150 lines)
└── README.md
```

Six files. ~650 lines total. Each plays a specific role.

### `main.rs` — the orchestrator, dissected

`main.rs` has three distinct phases:

**Phase 1: CLI parsing and identity setup** (lines 47-150).

```rust
let matches = Command::new("commonware-log")
    .about("generate secret logs and agree on their hash")
    .arg(Arg::new("bootstrappers").long("bootstrappers")
        .num_args(1..).value_delimiter(','))
    .arg(Arg::new("me").long("me").required(true))
    .arg(Arg::new("participants").long("participants").required(true)
        .num_args(1..).value_delimiter(','))
    .arg(Arg::new("storage-dir").long("storage-dir").required(true))
    .get_matches();

let me = matches.get_one::<String>("me").unwrap();
let parts: Vec<&str> = me.split('@').collect();
let key: u64 = parts[0].parse().expect("invalid key");
let port: u16 = parts[1].parse().expect("invalid port");

let signer = ed25519::PrivateKey::from_seed(key);
```

The `--me key@port` syntax is the convention throughout Commonware.
The key is an integer (for the example) or a hex string (production).
The signer is constructed from the seed.

```rust
let participants: Vec<u64> = matches
    .get_many::<String>("participants")
    .unwrap()
    .map(|s| s.parse().expect("invalid participant"))
    .collect();
let validators: Set<_> = participants
    .iter()
    .map(|peer| ed25519::PrivateKey::from_seed(*peer).public_key())
    .try_collect()
    .expect("duplicate keys");
```

The `Set<_>` is `commonware_utils::ordered::Set`, which enforces
deduplication and ordering invariants. The `try_collect` fails if any
two participants have the same key.

**Phase 2: Runtime + network + application** (lines 150-220).

```rust
let storage_directory = matches.get_one::<String>("storage-dir").unwrap();
let runtime_cfg = tokio::Config::new().with_storage_directory(storage_directory);
let executor = tokio::Runner::new(runtime_cfg);

executor.start(async |context| async move {
    // ... build network, application, engine ...
});
```

The `tokio::Runner` is the production runtime (chapter 02). For tests,
`deterministic::Runner` is used instead.

```rust
let p2p_cfg = discovery::Config::local(
    signer.clone(),
    &union(APPLICATION_NAMESPACE, b"_P2P"),
    SocketAddr::new(IpAddr::V4(Ipv4Addr::LOCALHOST), port),
    SocketAddr::new(IpAddr::V4(Ipv4Addr::LOCALHOST), port),
    bootstrapper_identities.clone(),
    1024 * 1024,
);
let (mut network, mut oracle) = discovery::Network::new(context.child("network"), p2p_cfg);
oracle.track(0, validators.clone());
```

`Config::local` is the convenience constructor for local-only
deployments. Production uses `Config::production` which adds TLS, larger
message sizes, and stricter rate limits.

```rust
let (vote_sender, vote_receiver) = network.register(0, Quota::per_second(NZU32!(10)), 256);
let (certificate_sender, certificate_receiver) = network.register(1, Quota::per_second(NZU32!(10)), 256);
let (resolver_sender, resolver_receiver) = network.register(2, Quota::per_second(NZU32!(10)), 256);
```

Three P2P channels, each rate-limited to 10 msgs/sec with a 256-message
mailbox buffer. The channel IDs are:
- `0` = votes (Simplex's notarize/nullify/finalize messages)
- `1` = certificates (Simplex's notarization/nullification/finalization)
- `2` = resolver (block fetch requests)

**Phase 3: Engine + GUI** (lines 220-250).

```rust
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
```

Every field is a knob with a default that works. The page cache
(16KB pages, 10000 cached) is sized for a typical workload. The
`Sequential` strategy runs in the current task; production uses
`Rayon` for parallel batch verification.

```rust
application.start();
network.start();
engine.start(
    (vote_sender, vote_receiver),
    (certificate_sender, certificate_receiver),
    (resolver_sender, resolver_receiver),
);
let gui = gui::Gui::new(context.child("gui"));
gui.run().await;
```

Four lines of orchestration. The runtime continues in the background
after `gui.run().await` blocks.

### `application/mod.rs` — the trait surface

`mod.rs` exposes the public API:

```rust
pub mod actor;
pub mod ingress;
pub mod reporter;

pub use actor::Application;
pub use ingress::{Propose, Verify};
pub use reporter::Reporter;
pub use application::genesis::genesis;

pub struct Scheme;

impl Scheme {
    pub fn signer(...) -> Result<SchemeType, Error> { ... }
    pub fn threshold(...) -> Result<SchemeType, Error> { ... }
}
```

The `Scheme` enum is the **voting scheme**. Two variants:

- `signer` — single-signer Ed25519 voting. Used in the example for
  simplicity. Each validator signs their own vote.
- `threshold` — BLS threshold signature voting. Used in production.
  Validators contribute shares; any `2f+1` shares reconstruct a valid
  vote.

The choice is made at construction time; the rest of the application
doesn't care which scheme is in use.

### `application/actor.rs` — the actor

`actor.rs` is ~70 lines and contains the entire application logic:

```rust
pub struct Application {
    context: Context,
    inner: Mutex<Inner>,
    propose_mailbox: Mailbox<Message<D>>,
    verify_mailbox: Mailbox<Message<D>>,
    reporter_mailbox: Mailbox<Message<D>>,
}

struct Inner {
    scheme: Scheme,
    hasher: Sha256,
    secrets: HashMap<Digest, [u8; 16]>,
}

impl Application {
    pub fn new(context: Context, cfg: Config) -> (Self, SchemeType, Reporter, Mailbox<Message<D>>) {
        // ... build application, return tuple
    }

    pub fn start(self) -> Handle<()> {
        self.context.spawn(|ctx| self.run(ctx))
    }

    async fn run(self, mut context: Context) {
        loop {
            select! {
                msg = self.propose_mailbox.recv() => {
                    if let Some(msg) = msg {
                        self.handle_propose(msg).await;
                    } else { break; }
                },
                msg = self.verify_mailbox.recv() => { ... },
                _ = context.stopped() => break,
            }
        }
    }

    async fn handle_propose(&mut self, req: ProposeRequest) {
        let mut secret = [0u8; 16];
        OsRng.fill_bytes(&mut secret);
        let digest = Sha256::hash(&secret);
        self.inner.lock().await.secrets.insert(digest, secret);
        req.responder.send(digest).unwrap();
    }

    async fn handle_verify(&mut self, req: VerifyRequest) {
        let inner = self.inner.lock().await;
        let result = if let Some(secret) = inner.secrets.get(&req.payload) {
            Sha256::hash(secret) == req.payload
        } else {
            true   // we don't have it; trust consensus
        };
        req.responder.send(result).unwrap();
    }
}
```

The `Mutex` is `tokio::sync::Mutex` — async-aware. **No locks are
held across `.await`** except the brief `lock().await` calls.

The actor implements `Automaton` and `Relay` via separate
`Mailbox` types. Each mailbox has its own FIFO queue; messages don't
block each other.

### `application/ingress.rs` — the message types

```rust
pub enum Message<D: Digest> {
    Propose {
        context: Context,
        response: oneshot::Sender<D>,
    },
    Verify {
        context: Context,
        payload: D,
        response: oneshot::Sender<bool>,
    },
}

impl<D: Digest> Policy for Message<D> {
    type Overflow = VecDeque<Self>;

    fn handle(overflow: &mut VecDeque<Self>, message: Self) {
        overflow.push_back(message);
    }
}
```

The `Policy` trait decides what to do when the mailbox is full.
Default: FIFO overflow queue. The example uses the default.

The message types carry:
- `Context` — runtime context for the handler.
- `oneshot::Sender` — the response channel.

The consensus engine calls `mailbox.send(...)` with one of these
messages. The actor picks it up in its loop, processes it, and sends
the response.

### `application/reporter.rs` — finalization notifications

```rust
pub struct Reporter {
    metrics: Arc<Metrics>,
}

impl crate::Reporter for Reporter {
    async fn report(&mut self, activity: Activity) {
        match activity {
            Activity::Finalization { height, view, hash } => {
                tracing::info!(
                    height, view, hash = ?hash,
                    "block finalized",
                );
                self.metrics.finalizations.inc();
            },
            Activity::Notarization { height, view, hash } => {
                self.metrics.notarizations.inc();
            },
            Activity::Nullification { view } => {
                self.metrics.nullifications.inc();
            },
            Activity::Fault(fault) => {
                tracing::warn!(?fault, "consensus fault detected");
                self.metrics.faults.inc();
            },
            _ => {},
        }
    }
}
```

The `Reporter` is the **third leg** of the consensus contract:

- `Automaton` decides what to propose/verify (chapter 11).
- `Relay` decides what to forward (chapter 09).
- `Reporter` is notified of consensus progress.

Without a `Reporter`, the application is **blind** to finalization. It
would propose and verify blocks but never know which ones became
final. The `Reporter` is the bridge from "consensus events" to
"application state."

### `gui.rs` — the TUI

`gui.rs` is ~150 lines of Ratatui. It shows:

```
┌─ Commonware log ──────────────────────────────────────────────┐
│ Validator 3 / 4     |  View 142     |  Height 12345           │
│ Finalized: 12344    |  Last block: 0xabc123...               │
│ Peers: 3 connected  |  Messages: 12,345 sent, 11,234 rcvd    │
├──────────────────────────────────────────────────────────────┤
│ Recent activity:                                              │
│   22:26:00.123  INFO  voted notarize view=142                 │
│   22:26:00.456  INFO  voted finalize view=141                 │
│   22:26:00.789  WARN  view change triggered, leader=1 stale   │
└──────────────────────────────────────────────────────────────┘
```

The TUI is **pure for the example**. The pattern — separate
visualization from consensus — is how you ship a healthy production
deployment. Dashboards should never be the only way to debug.

## `chat` example in full — encrypted group messaging

The chat example shows P2P without consensus. Here's the deep dive.

### The architecture

```
User A (chat client)                    User B (chat client)
    │                                       │
    │ session_key_A↔B (from P2P handshake)  │
    │<─────────────────────────────────────>│
    │                                       │
    │ msg_B = AEAD_encrypt(session_key, plaintext) │
    │<──────────────────────────────────────│
    │                                       │
    │ msg_B decrypts → display plaintext    │
```

Each chat client is a Commonware P2P peer. The P2P handshake
(chapter 05) establishes a **session key** between every pair of
peers. The session key is used for AEAD encryption.

### The AEAD session-key flow

The P2P handshake (chapter 05) does:

1. **Key exchange**: both sides contribute ephemeral keys (X25519 or
   similar). Each side derives a shared secret.
2. **Authentication**: each side signs a transcript of the key
   exchange. Signatures verify identities.
3. **Session key derivation**: the shared secret + signatures are
   fed to a KDF (HKDF-SHA256), producing a session key.

The session key is **symmetric** — both sides have it. The handshake
is asymmetric but the resulting session is symmetric.

Once the session key exists, all messages between the two peers are
AEAD-encrypted:

```rust
// sending
let ciphertext = ChaCha20Poly1305::new(session_key)
    .encrypt(nonce, plaintext.as_ref())?;

// receiving
let plaintext = ChaCha20Poly1305::new(session_key)
    .decrypt(nonce, ciphertext.as_ref())?;
```

The nonce is a counter (each message gets a fresh nonce) plus a
random prefix (to prevent nonce reuse across sessions).

### The encrypted group messaging pattern

For a group chat, you need encryption for `n > 2` peers. Two patterns:

**Sender keys** (used in Signal, WhatsApp): each user has a long-term
key pair and rotates a "sender key" per group. The sender key is
distributed to all group members via pairwise AEAD. All messages are
encrypted with the current sender key.

**Pairwise sessions** (used in chat): each pair has its own session
key. To send to the group, encrypt once per recipient.

The chat example uses **pairwise sessions** for simplicity:

```rust
async fn broadcast(&self, plaintext: &[u8]) -> Result<()> {
    let ciphertext = self.aead_encrypt(plaintext)?;
    for peer in &self.peers {
        self.network.send(peer, &ciphertext).await?;
    }
    Ok(())
}
```

For a group of `n` peers, this is `n` AEAD encryptions and `n` network
sends. For small groups (≤ 20), this is fine. For larger groups,
sender keys scale better.

### What `chat` teaches

- P2P without consensus is enough for low-fanout messaging.
- The `discovery::Manager` from chapter 05 also serves as a friend
  directory.
- AEADs over the P2P session are **better than end-to-end** for
  peer-to-peer groups: you trust the running session, not the
  long-term key.

## `bridge` example in full — cross-chain consensus certificates

The bridge example shows cross-chain communication via consensus
certificates. Here's the deep dive.

### The three binaries

The bridge example ships three binaries in one crate:

1. **`source`** — runs Simplex consensus, emits BLS threshold signatures
   on its own block hashes.
2. **`relayer`** — listens for `Finalization` messages from source,
   forwards them to sink.
3. **`sink`** — accepts BLS threshold signatures, includes them in its
   own consensus, exposes a verifyable cross-chain root.

```
Source Chain: Simplex → emits Finalization(B, σ_BLS)
                       ↓
Relayer:    forwards (B, σ_BLS) to sink
                       ↓
Sink Chain: includes (B, σ_BLS) in its own block
            emits Finalization(B, σ_BLS, sink_height)
                       ↓
Light Client: verifies σ_BLS against source's known BLS pubkey
              verifies inclusion in sink via sink's state proof
```

### The cert types

The bridge example defines:

```rust
// SourceCertificate: the source's BLS-signed proof of (height, root)
pub struct SourceCertificate {
    pub height: u64,
    pub root: Digest,           // state root at this height
    pub signature: BlsSignature,
}

// SinkInclusion: the sink's record of including a SourceCertificate
pub struct SinkInclusion {
    pub certificate: SourceCertificate,
    pub sink_height: u64,       // the sink height that included it
    pub sink_signature: BlsSignature,
}
```

The `SourceCertificate` is ~100 bytes regardless of validator count
(thanks to BLS aggregation). The `SinkInclusion` adds another ~100
bytes.

### The verification

A light client (a chain or an off-chain verifier) verifies:

```rust
// Verify source BLS signature
let source_pk = known_source_public_key();
let msg = (height, root).encode()?;
let valid = BlsScheme::verify(&source_pk, &msg, &signature);
assert!(valid, "invalid source signature");

// Verify inclusion in sink
let sink_root = known_sink_state_root();
let inclusion_proof = sink_inclusion_proof(sink_height, &source_cert)?;
assert!(verify_proof(&sink_root, &source_cert, &inclusion_proof));
```

The whole verification is O(1) — one BLS verify, one Merkle proof
verify. ~5ms on a modern CPU.

### Why this matters

Real cross-chain messaging (Optimism, Wormhole, LayerZero) all share
the same skeleton: a succinct proof that "X happened on chain A"
signed by the validators of A, validated cheaply on chain B. BLS
threshold sigs are the optimal primitive for this — one signature,
one verification, ~100 bytes regardless of validator count.

## `sync` example in full — server-client state synchronization

The sync example shows authenticated data fetching. Here's the deep dive.

### The server-client handshake

The server holds a key-value database. Each entry has a Merkle proof
back to the database root. The client wants to fetch entries and
verify them.

```rust
// Client → Server: handshake
let session_key = generate_ephemeral_key();
let hello = Hello { session_pubkey: session_key.public() };
send(hello);

// Server → Client: challenge
let nonce = random_nonce();
let challenge = Challenge { nonce, server_pubkey };
send(challenge);

// Client → Server: auth
let signature = sign(sk, &challenge);
let auth = Auth { signature };
send(auth);

// Server: verify signature against known client pubkey
// (in production, the client would have a known identity)
// For the example, the handshake is just protocol setup.
```

The handshake establishes an encrypted, authenticated channel. From
this point, all messages are encrypted with the session key and
authenticated with HMAC.

### The Resolver + Broadcast composition

The client uses `Resolver` (chapter 08) to request entries:

```rust
let resolver = Resolver::new(network, my_session_key);
let proof = resolver.request(key).await?;
```

The server uses `Broadcast` (chapter 09) to respond:

```rust
let broadcast = Broadcast::new(network, my_database);
broadcast.publish(entry_key, (value, merkle_proof)).await?;
```

The resolver sends a request; the broadcast publishes the response.
The composition is **request/response** over a PubSub substrate.

### The state synchronization protocol

The full flow:

```
client → "hello"
server → "challenge"
client → "auth"
server → "ok"

(client now authenticated)

client → "give me entry K"
server → (value, merkle_proof(K), server_signature)
client → verifies merkle_proof against known server_root
client → verifies server_signature

(proof chain: entry K's value is authentic, signed by server)
```

The proof is what makes this useful in adversarial contexts. Without
proof verification, a server could send you whatever bytes it wants.

The client also verifies the **server's signature on the response**,
which prevents replay attacks (each response has a unique nonce
derived from the request).

## `flood` example in full — gossip benchmarking

The flood example measures gossip performance. Here's the deep dive.

### The harness

```rust
let cfg = flood::Config {
    message_size: 1024,
    target_throughput: 1000,
    duration: Duration::from_secs(60),
    participant_count: 100,
};
let results = flood::run(cfg).await?;
```

`flood::run` spawns `participant_count` virtual nodes, each spamming
messages via Broadcast (chapter 09). It measures:

- **Bandwidth** (per-validator egress) — Prometheus counter.
- **Message delivery latency** — histogram, from broadcast send to
  receiver callback.
- **Duplicate ratio** — how often the same message arrives twice
  (Broadcast is best-effort; dedup is the receiver's job).
- **CPU usage** — sampled at 100Hz, reported as average and peak.

### The throughput measurement

Throughput is measured two ways:

1. **Send rate**: messages sent per second, averaged over the run.
2. **Receive rate**: messages received per second, averaged over the
   run.

The difference between send rate and receive rate tells you about
**packet loss** in the gossip layer. In a healthy network, the two
should be within a few percent.

### The test scenarios

Common scenarios the flood example supports:

```rust
// 1. Uniform latency, no loss
let cfg = flood::Config {
    latency: Duration::from_millis(50),
    jitter: Duration::from_millis(0),
    packet_loss: 0.0,
    ..default
};

// 2. WAN-like (high latency, some jitter)
let cfg = flood::Config {
    latency: Duration::from_millis(150),
    jitter: Duration::from_millis(30),
    packet_loss: 0.01,
    ..default
};

// 3. Adversarial (extreme loss)
let cfg = flood::Config {
    latency: Duration::from_millis(200),
    jitter: Duration::from_millis(100),
    packet_loss: 0.10,
    ..default
};
```

For each scenario, the flood example reports whether the broadcast
layer keeps up. The headline number is **useful bandwidth** (bytes
delivered, divided by total bytes sent) — a measure of gossip
efficiency.

### What `flood` teaches

- Broadcast is **rate-limited** at the channel level
  (`Quota::per_second`). Tune this to your bandwidth budget.
- Duplicate suppression matters: every flood test in `estimator` runs
  with dedup enabled (Bloom filter on hashes) to measure "useful"
  bandwidth.
- The broadcast layer is **best-effort**: a Byzantine peer can drop
  messages. The application must use consensus (chapter 11) or
  another mechanism for guaranteed delivery.

## `estimator` example in full — mechanism performance simulation

The estimator example runs a full Simplex instance in a simulated
network and measures the outcome. Here's the deep dive.

### The setup

```rust
let cfg = estimator::Config {
    n_validators: 100,
    latency: Duration::from_millis(50),
    jitter: Duration::from_millis(10),
    packet_loss: 0.001,
    byzantine_fraction: 0.0,
    block_payload_size: 1024,
    view_timeout: Duration::from_millis(1000),
    duration: Duration::from_secs(60),
    sample_interval: Duration::from_millis(100),
    // optional: which Byzantine mock to use
    byzantine_strategy: None,   // or Some(ByzantineStrategy::Conflicter)
};
let results = estimator::run(cfg).await?;
```

The config controls:

- **Network**: latency, jitter, packet loss.
- **Validators**: count, fraction that's Byzantine, which Byzantine
  strategy.
- **Workload**: block payload size, view timeout.
- **Run length**: duration, sampling interval.

### The metrics

The estimator records:

- **Finalization latency** (p50, p99, max) — wall-clock from proposal
  to finalization.
- **Throughput** — blocks per second.
- **View changes per epoch** — how often consensus had to skip a
  leader.
- **Network messages** — total count broken down by type (votes,
  certificates, resolver requests).
- **CPU usage** — sampled per validator, reported as mean and peak.

### The output

```rust
struct EstimatorResults {
    pub finalization_latency_p50: Duration,
    pub finalization_latency_p99: Duration,
    pub finalization_latency_max: Duration,
    pub throughput_bps: f64,
    pub view_changes_per_epoch: f64,
    pub messages_total: u64,
    pub messages_per_type: HashMap<String, u64>,
    pub cpu_usage_mean: f64,
    pub cpu_usage_peak: f64,
}
```

### The network conditions sweep

The estimator's killer feature: it sweeps network conditions
automatically. A typical `estimator --sweep` runs:

```
Latency:    10ms, 50ms, 100ms, 200ms, 500ms
Jitter:     0ms, 10ms, 50ms
Packet loss: 0%, 0.1%, 1%, 5%
Validators: 4, 10, 50, 100
Byzantine:  0%, 10%, 20%, 33%
```

That's 5 × 3 × 4 × 4 × 4 = 960 configurations. Each runs for 60
seconds. Total: 16 hours of simulation time (which the deterministic
runtime compresses to ~10 minutes of wall-clock).

The output is a CSV table:

```
validators,latency_ms,jitter_ms,loss,byzantine,p50_ms,p99_ms,bps,cpu_mean
4,10,0,0.0,0.0,80,95,200,0.1
4,10,0,0.0,0.33,90,120,180,0.15
...
```

### Why this is the unit test for new mechanisms

When the team designed buffered sigs, they ran `estimator --sweep`
with and without, over thousands of network conditions. The Alto team
saw the simulated 20% block time improvement **before** shipping any
binary changes.

The estimator is the **lab** for new mechanisms. The pattern is:

1. Implement a new mechanism in Commonware.
2. Add a flag to `estimator` to enable/disable it.
3. Run `--sweep` for both configurations.
4. Compare CSVs. The new mechanism wins or loses on the numbers.

## `reshare` example in full — threshold key resharing

The reshare example demonstrates DKG resharing. Here's the deep dive.

### The setup

A committee of `n_old` validators holds shares of secret `S`. We want
to rotate to a new committee of `n_new` validators who will also hold
shares of `S` — without re-running DKG.

```rust
// Old committee holds shares (s_1, ..., s_n_old) of secret S
let old_committee = (0..n_old).map(|i| share_i).collect::<Vec<_>>();

// New committee will hold shares (s'_1, ..., s'_n_new) of secret S
let new_committee_members = pick_new_set(n_new)?;

// Run resharing
let new_shares = reshare(old_committee, new_committee_members).await?;
// new_shares[i] is the share for member i of the new committee
```

### The protocol

Each old committee member runs:

```
1. Feldman-commit the share s_i:
   commit_i = (g · s_i, g · s_i·α, g · s_i·α², ...)  // G1 elements

2. Each new committee member j requests a share:
   request_ij = PKE.Encrypt(pk_j, s_i)   // encrypted to j's public key

3. Old member i verifies j is "in" via a Schnorr proof:
   schnorr_ij = prove_knowledge(s_i, pk_j)

4. Old member i sends encrypted share + proof to new member j:
   (enc_ij, schnorr_ij, commit_i)

5. New member j decrypts, verifies Schnorr proof against commit_i:
   s_i_j = decrypt(enc_ij)
   verify_schnorr(schnorr_ij, commit_i, pk_j)

6. New member j sums all received shares:
   s'_j = sum_i s_i_j

7. Once j has ≥ t+1 shares, j's share of S is verified by:
   Verify(joint_Feldman_commitment, s'_j, j)
```

### The security model

The security holds as long as:

- At least `t+1` old committee members are honest.
- At least `t+1` new committee members are honest.
- The encryption scheme is IND-CCA secure.
- The Schnorr proofs are zero-knowledge.

This is the **Pedersen DKG with resharing** construction, adapted to
Commonware's BLS12-381 primitives.

### Where it shines

Validator set rotation (chapter 20) requires DKG every time the set
changes. Synchronous DKG fails if the network is choppy. Golden DKG
(chapter 20) is the asynchronous variant; resharing is the
**incremental** variant — only the delta is communicated.

Alto uses resharing to onboard new validators without downtime:

1. Network produces blocks via current committee.
2. New validator joins, runs reshare with current committee.
3. New validator receives share of S.
4. Network continues producing blocks.
5. After 2f+1 new committee members have shares, the new committee
   is "active" — old committee can be deprecated.

No DKG ceremony. No consensus halt. The network keeps going.

## Patterns the examples teach

The earlier section listed five patterns. Here are deeper explanations
and code references.

### Composition over inheritance

There is **no class hierarchy** in Commonware. The examples compose
**behaviors** by trait impls:

```rust
impl Automaton<D, P> for Application { ... }
impl Relay<D, P> for Application { ... }
impl Reporter for Reporter { ... }
```

Each trait represents one capability the consensus engine needs.
Your application satisfies them by being the runtime + state, not by
subclassing anything.

This is **the central design choice** of Commonware. Primitives are
types, behaviors are traits, composition plugs them together.

### The "channels + quotas" pattern

Every P2P channel in every example has rate limiting:

```rust
let (sender, receiver) = network.register(
    channel_id,                          // unique per channel
    Quota::per_second(NZU32!(10)),       // 10 msgs/sec per peer
    256,                                 // mailbox buffer size
);
```

The `10 msgs/sec` is the per-peer rate. The `256` is the receiver's
mailbox size. These are non-defaults you must think about.

Too loose: garbage data DoSes the consensus.
Too tight: legitimate messages get dropped under burst.

### Non-blocking everything

Look at every `.await` in the examples. There are **no locks held
across `.await`** and **no blocking I/O**. The actor pattern (chapter
15) ensures this: actors message each other, never call each other.

This is a hard rule for Commonware code. The deterministic runtime
gets it for free (no real I/O); production code gets it by being
disciplined.

```rust
// WRONG: lock held across .await
let mut guard = self.state.lock().await;
self.network.send(msg).await;          // .await while holding lock
guard.update(msg);                      // potential deadlock

// RIGHT: lock released before .await
let update = {
    let mut guard = self.state.lock().await;
    guard.compute_update(msg)
}; // lock released here
self.network.send(&update).await;       // .await without lock
```

### Graceful defaults, explicit overrides

`Config::local` for P2P in `log`. `Scheme::signer` for voting in
`log`. `Ordered` strategy for Simplex by default. Every primitive has
a "happy path" config and a "tune every knob" config.

The examples take the happy path. Production code begins with the
happy path and only deviates when measurement demands it.

### Trivial instrumentation

Look at every file. There's a `tracing::info!` or `tracing::debug!`
at every meaningful boundary. The motivation isn't debug-print
hygiene — it's so you can later turn on `RUST_LOG=debug` and see
exactly what the consensus was doing in a problematic moment.

```rust
tracing::info!(key = ?signer.public_key(), "loaded signer");
tracing::debug!(view = view.0, height = h, "voted notarize");
tracing::warn!(peer = %peer, "rate limit exceeded");
tracing::error!(err = %e, "instance launch failed");
```

The fields are typed (`?` for Debug, `%` for Display, plain for
primitives). The output is JSON in production, pretty in development.

### The mailboxes-as-contracts pattern

Every actor has mailboxes for its inputs. The mailbox types are the
**contract**: anyone who wants to talk to the actor knows the type
and can `send()` to it. The actor's internal state is private.

```rust
pub struct Application {
    propose_mailbox: Mailbox<Message<D>>,    // public: send to me
    verify_mailbox: Mailbox<Message<D>>,     // public: send to me
    // private:
    inner: Mutex<Inner>,
}
```

This is **encapsulation** without `pub`/`private` keywords on every
field. The mailboxes are the only public API; the inner state is
accessed only through the actor loop.

## Exercises — practice

Five hands-on exercises to deepen your understanding.

### Exercise 1 — Run `log` locally with 4 nodes

```bash
# In four terminals, run:
cargo run --release --bin commonware-log -- \
    --me 1@3001 --participants 1,2,3,4 \
    --bootstrappers 2@127.0.0.1:3002,3@127.0.0.1:3003,4@127.0.0.1:3004 \
    --storage-dir /tmp/log-1

cargo run --release --bin commonware-log -- \
    --me 2@3002 --participants 1,2,3,4 \
    --bootstrappers 1@127.0.0.1:3001,3@127.0.0.1:3003,4@127.0.0.1:3004 \
    --storage-dir /tmp/log-2

# ... etc for nodes 3 and 4
```

Watch the TUI on each. After 30 seconds, you should see all four
nodes showing the same height and view. Verify the messages
finalize.

### Exercise 2 — Replace `Scheme::signer` with `Scheme::threshold`

In `examples/log/src/main.rs`, swap:

```rust
let scheme = application::Scheme::signer(...)?;
```

with:

```rust
let scheme = application::Scheme::threshold(...)?;
```

Run `cargo run` and observe. Note any compilation errors — the
threshold variant needs different arguments (it doesn't take a
single `signer` but a set of public keys).

### Exercise 3 — Add a metric to the Reporter

In `examples/log/src/application/reporter.rs`, add a counter for
"participations" (every time the validator participates in a view):

```rust
let participations = context.register(
    "log_participations_total",
    "times this validator participated in a view",
    metrics::Counter::new(),
);

// In the report() method:
Activity::Participation { height, view } => {
    participations.inc();
}
```

Restart and verify the metric appears in the metrics output.

### Exercise 4 — Measure `flood` bandwidth

Run the flood example with various message sizes:

```bash
cargo run --release --bin commonware-flood -- --message-size 1024 --duration 30
cargo run --release --bin commonware-flood -- --message-size 65536 --duration 30
cargo run --release --bin commonware-flood -- --message-size 1048576 --duration 30
```

Record the per-validator egress for each. Plot message size vs
bandwidth. Is the relationship linear?

### Exercise 5 — Read `alto/src/consensus/finalize.rs`

Clone https://github.com/commonwarexyz/alto. Read
`alto/src/consensus/finalize.rs`. Identify:

1. The `Reporter::report` implementation.
2. The state machine for handling `Activity::Finalization`.
3. The error handling at each step.
4. The metrics emitted.
5. The interaction with `marshal`, `state`, and `journal`.

Compare with `examples/log/src/application/reporter.rs`. What's
similar? What's different?

## Where to look in the code

- `examples/log/` — minimal consensus app.
- `examples/chat/` — encrypted P2P messaging.
- `examples/bridge/` — cross-chain certificates.
- `examples/sync/` — state sync.
- `examples/flood/` — gossip benchmark.
- `examples/estimator/` — mechanism performance.
- `examples/reshare/` — threshold resharing.
- `https://github.com/commonwarexyz/alto` — production blockchain.
- `https://github.com/commonwarexyz/battleware` — onchain game.

## Appendix A — `log`: a complete code walkthrough

`examples/log/src/main.rs` (the orchestrator) is short. Let me walk
through it section by section.

### Lines 47-90: CLI parsing

```rust
let matches = Command::new("commonware-log")
    .about("generate secret logs and agree on their hash")
    .arg(Arg::new("bootstrappers").long("bootstrappers").required(false)...)
    .arg(Arg::new("me").long("me").required(true))
    .arg(Arg::new("participants").long("participants").required(true)...)
    .arg(Arg::new("storage-dir").long("storage-dir").required(true))
    .get_matches();
```

Standard `clap` parsing. Four flags:
- `--bootstrappers`: comma-separated `key@addr` pairs (the bootstrap nodes).
- `--me`: `key@port` (this node's identity and listening port).
- `--participants`: comma-separated keys (all validators in the network).
- `--storage-dir`: where to persist the journal.

### Lines 93-105: Identity setup

```rust
let me = matches.get_one::<String>("me").expect(...);
let parts = me.split('@').collect::<Vec<&str>>();
let key = parts[0].parse::<u64>().expect(...);
let signer = ed25519::PrivateKey::from_seed(key);   // ed25519 from seed
```

**Key derivation from integer**: the seed is an integer `u64`. The
ed25519 key is derived deterministically. This is **insecure** (real
deployments would use a hardware wallet or KMS) but it makes the
example runnable without a key management layer.

The `tracing::info!(key = ?signer.public_key(), "loaded signer")` line
prints the public key so you can verify which "me" you are.

### Lines 109-126: Participant setup

```rust
let participants = matches.get_many::<u64>("participants")...
    .cloned().collect::<Vec<_>>();
let validators: Set<_> = participants
    .iter()
    .map(|peer| {
        let verifier = ed25519::PrivateKey::from_seed(*peer).public_key();
        tracing::info!(key = ?verifier, "registered authorized key");
        verifier
    })
    .try_collect()
    .expect("public keys are unique");
```

Build the set of validator public keys. Same insecure derivation: peer
`N` has the same public key no matter which node runs it. In
production, each node would have its own keypair.

The `try_collect` ensures no duplicate keys.

### Lines 128-141: Bootstrapper parsing

```rust
let bootstrappers = matches.get_many::<String>("bootstrappers");
let mut bootstrapper_identities = Vec::new();
if let Some(bootstrappers) = bootstrappers {
    for bootstrapper in bootstrappers {
        let parts = bootstrapper.split('@').collect::<Vec<&str>>();
        let bootstrapper_key = parts[0].parse::<u64>().expect(...);
        let verifier = ed25519::PrivateKey::from_seed(bootstrapper_key).public_key();
        let bootstrapper_address =
            SocketAddr::from_str(parts[1]).expect("...");
        bootstrapper_identities.push((verifier, bootstrapper_address.into()));
    }
}
```

Parse bootstrappers into `(public_key, ingress_address)` pairs. The
`Ingress` type (chapter 05) wraps the socket address.

### Lines 144-150: Runtime + storage

```rust
let storage_directory = matches.get_one::<String>("storage-dir").expect(...);
let runtime_cfg = tokio::Config::new().with_storage_directory(storage_directory);
let executor = tokio::Runner::new(runtime_cfg);
```

Tokio runtime with the storage directory for the journal (chapter 06).

### Lines 153-160: P2P config

```rust
let p2p_cfg = discovery::Config::local(
    signer.clone(),
    &union(APPLICATION_NAMESPACE, b"_P2P"),    // namespaced handshake
    SocketAddr::new(IpAddr::V4(Ipv4Addr::LOCALHOST), port),
    SocketAddr::new(IpAddr::V4(Ipv4Addr::LOCALHOST), port),    // same for listen + dialable
    bootstrapper_identities.clone(),
    1024 * 1024,    // 1 MB max message size
);
```

P2P discovery configuration. `Config::local` is a convenience for
local-only deployments (no DNS, no private IPs).

The `union(APPLICATION_NAMESPACE, b"_P2P")` namespaces the P2P handshake
(chapter 04). Each app should use a unique namespace.

### Lines 163-191: Inside the executor

```rust
executor.start(async |context| {
    // Initialize the network
    let (mut network, mut oracle) = discovery::Network::new(context.child("network"), p2p_cfg);

    // Provide authorized peers
    oracle.track(0, validators.clone());

    // Register consensus channels
    let (vote_sender, vote_receiver) = network.register(0, Quota::per_second(NZU32!(10)), 256);
    let (certificate_sender, certificate_receiver) = network.register(1, Quota::per_second(NZU32!(10)), 256);
    let (resolver_sender, resolver_receiver) = network.register(2, Quota::per_second(NZU32!(10)), 256);
```

Inside the runtime context (chapter 02):
1. Initialize the network.
2. Tell the oracle which peers to allow (`track`).
3. Register three channels for the three Simplex channel types:
   - 0 = votes (chapter 11 batcher input)
   - 1 = certificates (chapter 11 voter input)
   - 2 = resolver (chapter 11 voter → resolver)

### Lines 193-204: Application setup

```rust
let namespace = union(APPLICATION_NAMESPACE, b"_CONSENSUS");
let scheme = application::Scheme::signer(&namespace, validators.clone(), signer.clone())
    .expect("private key must be in participants");
let (application, scheme, reporter, mailbox) = application::Application::new(
    context.child("application"),
    application::Config {
        hasher: Sha256::default(),
        scheme,
        mailbox_size: NZUsize!(1024),
    },
);
```

The application:
- Uses `Sha256` for hashing.
- Uses ed25519 signatures for voting.
- Has a mailbox of 1024 messages.

`Scheme::signer` creates an ed25519 scheme that knows about the
validator set.

### Lines 207-230: Simplex engine

```rust
let cfg = simplex::Config {
    scheme,
    elector: RoundRobin::<Sha256>::default(),    // round-robin leader election
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
```

All the knobs (chapter 11 Appendix K). The page cache (16KB pages,
10000 cached) is sized for the typical workload.

### Lines 234-244: Start everything

```rust
application.start();
network.start();
engine.start(
    (vote_sender, vote_receiver),
    (certificate_sender, certificate_receiver),
    (resolver_sender, resolver_receiver),
);

let gui = gui::Gui::new(context.child("gui"));
gui.run().await;
```

1. Start the application actor (chapter 15).
2. Start the network (chapter 05).
3. Start the Simplex engine (chapter 11) with the three P2P channels.
4. Start the GUI (Ratatui-based TUI showing consensus progress).

The function blocks on `gui.run().await`. The runtime continues in the
background.

## Appendix B — The application's `propose` and `verify`

`examples/log/src/application/actor.rs`. The application logic:

```rust
fn propose(&mut self, context: Context) -> oneshot::Receiver<Digest> {
    let (tx, rx) = oneshot::channel();
    self.propose_mailbox.send(ProposeRequest { context, responder: tx });
    rx
}

fn verify(&mut self, context: Context, payload: Digest) -> oneshot::Receiver<bool> {
    let (tx, rx) = oneshot::channel();
    self.verify_mailbox.send(VerifyRequest { context, payload, responder: tx });
    rx
}
```

Both methods are **non-blocking**: they send a request to the application's
mailbox and return the oneshot receiver immediately. The actual work
happens in the actor's loop.

```rust
async fn handle_propose(&mut self, req: ProposeRequest) {
    // Generate a random 16-byte secret
    let mut secret = [0u8; 16];
    OsRng.fill_bytes(&mut secret);

    // Hash the secret (the consensus sees the hash, not the secret itself)
    let digest = Sha256::hash(&secret);

    // Store the (secret, digest) pair for later verification
    self.secrets.insert(digest, secret);

    // Notify consensus
    req.responder.send(digest).unwrap();
}
```

The propose logic:
1. Generate random secret.
2. Hash it.
3. Store the (hash → secret) mapping locally.
4. Return the hash to consensus.

The **secret is never broadcast**. Only its hash. Validators agree on
the hash; the secret stays with whoever proposed.

```rust
async fn handle_verify(&mut self, req: VerifyRequest) {
    // Verify by hashing the locally-stored secret and comparing
    if let Some(secret) = self.secrets.get(&req.payload) {
        let computed = Sha256::hash(secret);
        req.responder.send(computed == req.payload).unwrap();
    } else {
        // We don't have this secret (we're not the proposer)
        // Trust the consensus layer to handle it
        req.responder.send(true).unwrap();
    }
}
```

The verify logic: if we have the secret, recompute and compare. If we
don't (we're not the proposer), trust the hash.

## Appendix C — The `chat` example

`examples/chat/src/main.rs`. Simpler than `log`:

```rust
let executor = tokio::Runner::default();
// ... parse args, set up P2P discovery ...
executor.start(|context| {
    // ... create P2P, handler, logger ...
});
```

The chat app:
- **No consensus.** Direct P2P messaging.
- **No storage.** Messages are ephemeral.
- **Encrypted messages** (via the P2P handshake).

Use case: encrypted group chat. The "consensus" is implicit — humans
agree on who's in the group.

## Appendix D — The `bridge` example

`examples/bridge/src/`. Three binaries in one crate:

1. **Source chain**: emits consensus certificates (BLS threshold sigs
   on block hashes).
2. **Sink chain**: accepts certificates, includes them in its own
   consensus.
3. **Relayer**: bridges source certificates to sink.

The flow:
```
Source chain: Simplex consensus → emits Finalization with threshold sig
                                                          ↓
Relayer: picks up Finalizations, forwards to sink
                                                          ↓
Sink chain: receives Finalizations, includes in own Simplex consensus
```

Cross-chain interoperability via succinct certificates.

## Appendix E — The `sync` example

`examples/sync/src/`. Server-client state sync:

```rust
// Server: maintains a database, serves keys on demand
let server_db = ...;
network.register(0, server_db);  // channel 0 = data requests

// Client: connects, fetches keys, verifies
let client = Client::new(network);
client.fetch(key);
```

The client verifies proofs against the server's root (chapter 16). The
`db::p2p::standard` resolver (chapter 08) handles the wire protocol.

## Appendix F — The `flood` example

`examples/flood/src/`. A gossip benchmark:

```rust
for i in 0..N {
    let msg = random_message();
    network.broadcast(Recipients::All, msg, false);
    sleep(Duration::from_millis(10));
}
```

Measures:
- Bandwidth usage (per validator egress).
- Message delivery latency.
- Throughput (messages per second).

Use this to size your network — if `flood` saturates the link at
1000 msgs/sec, real consensus (with re-broadcasts) might too.

## Appendix G — The `estimator` example

`examples/estimator/src/`. Mechanism performance simulation:

```rust
let config = estimator::Config {
    n_validators: 100,
    latency: Duration::from_millis(150),
    jitter: Duration::from_millis(30),
    packet_loss: 0.01,
    duration: Duration::from_secs(60),
};
let results = estimator::run(config);
println!("Finalization latency: {:?}", results.avg_finalization_latency);
println!("Throughput: {} blocks/sec", results.throughput);
```

Vary the parameters (latency, jitter, loss, validator count) to see how
Simplex behaves under different network conditions. The results drive
Alto's performance tuning (chapter 20's buffered-signatures blog).

## Appendix H — The `reshare` example

`examples/reshare/`. Threshold secret resharing:

```rust
// Old committee: holds shares of secret S
let old_committee = ...;

// New committee: will hold shares of secret S (same secret, new shares)
let new_committee = ...;

// Run the resharing protocol
reshare(old_committee, new_committee).await?;
```

After resharing, the **same** threshold key is shared among the new
committee. Used for validator set rotation without re-running DKG.

## Appendix I — External examples

### `alto`

`https://github.com/commonwarexyz/alto`. Production blockchain using
Commonware:

- Simplex consensus.
- Marshal for block delivery.
- P2P for networking.
- Storage for the WAL.
- QMDB for state.
- Buffered signatures for performance (chapter 20).

~200ms block times, ~300ms finality (chapter 20's buffered signatures
blog).

### `battleware`

Onchain game secured by VRF, Timelock Encryption, and MMRs. Uses
Commonware primitives in creative ways.

## Appendix J — Reading order for examples

If you're building a real app, read in this order:

1. **`log`** (~250 lines) — see the minimum wiring.
2. **`chat`** (~200 lines) — see P2P without consensus.
3. **`sync`** (~400 lines) — see resolver + broadcast.
4. **`bridge`** (~500 lines) — see consensus + marshal.
5. **`reshare`** (~300 lines) — see threshold key lifecycle.
6. **`estimator`** (~300 lines) — see benchmarking patterns.
7. **`flood`** (~200 lines) — see gossip at scale.

Then read the external `alto` source (~5000 lines) for production-grade.

## Appendix K — Common patterns across examples

### The "parse, validate, panic" pattern

```rust
let key = parts[0].parse::<u64>().expect("Key not well-formed");
```

Examples use `.expect(...)` for parsing failures. Production would use
proper error handling with `anyhow::Result`.

### The "trace at boundaries" pattern

```rust
tracing::info!(key = ?signer.public_key(), "loaded signer");
```

Every major setup step emits a tracing event. Makes debugging easy.

### The "graceful defaults, explicit overrides" pattern

```rust
let p2p_cfg = Config::local(signer, namespace, addr, addr, bootstrappers, 1024 * 1024);
```

`Config::local` sets sensible defaults. Production would use
`Config::production` (which adds TLS, larger message sizes, etc.) and
override only the specific fields.

### The "channels with quotas" pattern

```rust
network.register(channel_id, Quota::per_second(NZU32!(10)), 256);
```

Every channel has a rate limit + mailbox size. Prevents unbounded growth.

## Where to look in the code (expanded)

- `examples/log/` — minimal consensus app (~250 lines).
- `examples/chat/` — encrypted P2P messaging (~200 lines).
- `examples/bridge/` — cross-chain certificates (~500 lines).
- `examples/sync/` — state sync (~400 lines).
- `examples/flood/` — gossip benchmark (~200 lines).
- `examples/estimator/` — mechanism performance (~300 lines).
- `examples/reshare/` — threshold resharing (~300 lines).

## If you only remember three things

1. **`log` is the minimal consensus app.** Read it first; everything else is `log` minus something or plus something.
2. **Each example is `log` minus something or plus something.** `chat` = `log` - Simplex. `bridge` = `log` + Marshal. `sync` = Resolver + Broadcast.
3. **The ten-line wiring is everything.** All 20 chapters of substrate, ten lines of main.rs.

→ Next: **Chapter 20 — Frontier**. The research blog tour. What's at the
edge of Commonware's capabilities today.