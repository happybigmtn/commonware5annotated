# Chapter 00 — Orientation: what Commonware is, and why

> The mental map for the whole curriculum. If you read one chapter, read this one.
> Then come back and re-read it after chapter 11.

## The territory in one paragraph

**Commonware** (by commonwarexyz) is a Rust library of distributed systems
**primitives** — small, composable building blocks for building blockchain /
consensus applications that survive **adversarial** (Byzantine) environments.
17 primitives, 50+ dialects, 1000+ benchmarks, ~93% test coverage,
dual-licensed Apache-2 / MIT.

## The 30,000 ft view — Feynman style

If you don't know what Commonware is for, the 17 crates look like alphabet soup.

**Imagine you and 3 friends are sitting in different coffee shops and you want to keep
a shared notebook.** Every time anyone writes a line, everyone else needs to see it.
And the catch is: **one of your friends might be evil** — they might lie about what
they wrote, they might pretend they never got your line, they might try to rewrite
history.

That's it. That's the whole problem of distributed consensus. The whole library.
Everything else is details.

**Now here's what makes Commonware weird:** Most projects say "here's a framework,
fill in these 12 traits and we'll run a blockchain for you." Commonware says
"no — go pick the pieces you want." It's the **anti-framework**.

Why? Because in their blog "Outgrowing the Virtual Machine," they point out that
every time a real builder wants to do something interesting — like have a game that
orders transactions in a custom way to deter cheating — they end up **fighting the
framework**. So Commonware gives you the Lego blocks instead.

### Why this approach wins

The DDIA (Designing Data-Intensive Applications, Martin Kleppmann) describes
three properties a distributed system needs:

1. **Reliability** — the system continues to work correctly, even when
   individual components fail.
2. **Scalability** — the system can handle increased load without breaking.
3. **Maintainability** — the system can be evolved by many engineers over
   time.

Commonware picks a stance on each:

| Property | Commonware's stance |
|---|---|
| **Reliability** | Build for the worst case: arbitrary (Byzantine) failures. If you handle a malicious peer, an honest-but-broken peer is easy. |
| **Scalability** | The runtime, the codec, the storage are all O(n) in the size of what they handle. The consensus is O(n²) in messages per view — but the size of the view is bounded, and pipelining lets you do thousands of views per second. |
| **Maintainability** | Small primitives, one job each. Wire formats are conformance-tested so changes don't silently break deployed nodes. Stability levels let you commit to "this API is frozen." |

The "anti-framework" stance is the maintainability bet: instead of one big
project that everyone fights over, you get small pieces you can swap.

## The single most important idea

> **Consensus should run over opaque "blobs" (just a hash). Not over a prescribed
> > block format. You give me the hash of whatever you want, and I'll get everyone
> > to agree on the hash. What the hash points to is your problem.**

That's the **Decoupled State Machine Replication (DSMR)** idea. Consensus is dumb.
It's a hash-agreement machine. Application is smart. It does whatever it wants.

### Where DSMR came from

The traditional blockchain framework (Bitcoin, Ethereum, Tendermint Core,
Cosmos SDK) conflates three things:

1. **Consensus** — what's the agreed order of operations?
2. **Execution** — what does each operation do to state?
3. **Data** — where do we store the results?

Bitcoin Core does all three in one C++ process. Ethereum's Geth does the same.
Tendermint Core splits consensus (Go) from execution (ABCI app, any language).

Commonware pushes this further: the consensus crate doesn't even know what a
"block" is. It only knows about "payloads" (hashes). The application figures
out what the payloads point to. The Marshal crate (chapter 12) is the bridge
that translates "certificates the consensus produces" into "blocks the
application actually executes."

This is a real architectural simplification. Let me show you.

### Bitcoin's mental model (conflated)

```
Block = {
    header: { version, prev_block, merkle_root, timestamp, bits, nonce },
    transactions: [Tx, Tx, Tx, ...]
}
Merkle root = hash of all transactions.
```

When a miner finds a block, they:
1. Construct the header.
2. Mine (find a nonce that makes the hash small enough).
3. Broadcast the block.

Every other node:
1. Receives the block.
2. Verifies the PoW (very expensive).
3. Validates every transaction (signature, double-spend, script execution).
4. Updates their UTXO set.

Notice: **the consensus (PoW) and the validation (script + UTXO) are
intertwined.** The block format *is* the application.

### Commonware's mental model (decoupled)

```
Consensus Input: Digest = [u8; 32]            // just a hash
Consensus Output: Notarization { view, payload: Digest, certificate }
Application: I have a digest. What's behind it? My problem.
Marshal: I take a Notarization, fetch the bytes behind the digest,
          verify they hash to it, and hand them to the application.
```

When a validator is leader for view 5:
1. Application produces a payload (whatever it wants — a list of EVM
   transactions, a Solana-like block, a Move module, a game state update).
2. Hash it to get a `Digest`.
3. Send the digest into consensus.

Other validators:
1. Receive the digest.
2. Run consensus on the digest (this is cheap — it's just a hash).
3. Once notarized, Marshal fetches the actual bytes behind the digest
   (lazily, in parallel, via ZODA erasure coding — chapter 07).
4. Application verifies the bytes (whatever that means for your app).

**The validator voted on a hash, not on bytes.** That means the
**consensus-critical path never has to parse your application format.**
Hot path stays small and fast. Cold path (fetching + verification) can be
heavy and slow without affecting consensus throughput.

This is the architectural insight that motivates the entire library.

## The second most important idea — the testability trick

> All protocols are written against an abstract Runtime trait. In production, the
> > trait is implemented by Tokio. In tests, it's implemented by a Deterministic
> > Runtime that controls time, randomness, and the network. Same code, two
> > completely different execution models.

That means they can run thousands of adversarial scenarios (Byzantine nodes,
network partitions, crashes) with one seed and **get the same answer every time**.
Reproducible distributed systems testing. That's huge — usually distributed
systems testing is just "run it and pray."

### Why this matters

DDIA chapter 1 makes a sharp distinction between **faults** (one machine
breaks) and **failures** (the system as a whole stops doing what it should).
Distributed systems testing usually fails because:

1. **Non-determinism**: two runs of the same test do different things. You
   see a bug once, can never reproduce it, can't write a regression test.
2. **Slow tests**: you can't run 10,000 adversarial scenarios if each one
   takes 30 seconds of wall clock.
3. **Environmental flakiness**: real networks have real packet loss, real
   latency jitter, real DNS timeouts. Your test passes locally and fails in CI.

The deterministic runtime solves all three:
1. **Same seed → same execution**: a SHA-256 of the entire event history.
2. **Time is virtual**: 10,000 simulated seconds run in milliseconds of wall clock.
3. **Network is simulated**: no flakiness.

This isn't novel — TLA+ (Leslie Lamport's model checker) does similar things
for protocol design. What Commonware adds is that **the test code is the
production code**: same trait, two implementations.

## The third idea — stability as a first-class concept

> Each primitive is tagged ALPHA → EPSILON. Stability is transitive: a BETA
> > primitive can only depend on BETA or higher. You can compile your app with
> > `--cfg commonware_stability_BETA` and the compiler will refuse to let you
> > depend on an ALPHA primitive.

It's like Semver, but for cryptographic protocols, where a "breaking change" might
mean everyone's nodes get slashed.

### The full stability ladder

| Level | Index | Meaning |
|---|---|---|
| **ALPHA** | 0 | Breaking changes expected. No migration path provided. |
| **BETA** | 1 | Wire and storage formats stable. Breaking changes include a migration path. |
| **GAMMA** | 2 | API stable. Extensively tested and fuzzed. |
| **DELTA** | 3 | Battle-tested. Bug bounty eligible. |
| **EPSILON** | 4 | Feature-frozen. Only bug fixes and performance improvements accepted. |

### What this buys you

When you build on Commonware, you pick a **target stability**. If you say "I
only want to depend on BETA+ primitives," the compile-time flags hide every
ALPHA primitive from your build. You literally cannot accidentally use an
unstable API.

Compare to a typical Rust library where SemVer promises exist but aren't
enforced by the compiler. You might `cargo update` and suddenly your binary
breaks because some transitive dep bumped from 0.3 to 0.4. Commonware says:
"I promise this primitive's public API is frozen at GAMMA. If I break it,
my CI fails before I can merge the breaking change."

This is the kind of guarantee you want when you're deploying code that
holds a billion dollars of user funds.

## The crate map

Here is every crate in the workspace, what it actually does, and when you'd
reach for it.

### Core runtime and codec

| Crate | Stability | What it does | When to use |
|---|---|---|---|
| `runtime` | BETA (tokio), ALPHA (deterministic) | Abstract async scheduler. | Always. You can't write Commonware code without it. |
| `codec` | (core) | Deterministic binary serialization. | Whenever you need to put something on the wire or to disk. |

### Identity and consensus

| Crate | Stability | What it does | When to use |
|---|---|---|---|
| `cryptography` | varied | Keys, signatures, hashing, BLS12-381, transcripts, certificates. | Always — for identity, signatures, content hashing. |
| `consensus` | varied per dialect | Order opaque messages under BFT assumptions. | When you need Byzantine agreement. |
| `marshal` | varies | Bridge between consensus certificates and application blocks. | When you want to "do Simplex but with real blocks." |

### Network and data

| Crate | Stability | What it does | When to use |
|---|---|---|---|
| `p2p` | BETA | Authenticated, encrypted peer-to-peer transport with rate limiting. | When you need to send messages between processes. |
| `resolver` | varies | Fetch data identified by a fixed-length key. | When consensus commits to a hash and you need to retrieve the bytes. |
| `broadcast` | varies | Disseminate data across many peers with caching. | For gossip protocols, block dissemination. |
| `collector` | varies | Gather responses from a quorum with monitoring. | When you need to assemble certificates (built into consensus). |
| `coding` | ALPHA | Erasure coding — Reed-Solomon, ZODA. | When bandwidth matters and you can spend CPU on encoding. |

### Persistence

| Crate | Stability | What it does | When to use |
|---|---|---|---|
| `storage` | varied | Pluggable persistent storage — blobs, journals, authenticated databases. | When you need to remember anything across restarts. |
| `stream` | varies | Encrypted message streams over arbitrary transport. | For custom protocols over TCP, QUIC, in-memory channels. |

### Coordination

| Crate | Stability | What it does | When to use |
|---|---|---|---|
| `actor` | BETA | Concurrent components via bounded mailboxes. | When you have a long-running component that processes messages. |
| `glue` | varies | Default cross-primitive constructions. | When you want a standard stateful app on top of QMDB. |
| `deployer` | ALPHA | Cloud infra provisioning (AWS multi-region). | When you're ready to deploy. |

### Utilities

| Crate | Stability | What it does | When to use |
|---|---|---|---|
| `formatting` | ALPHA | Encode/parse data (hex, base64). | Trivially. |
| `invariants` | ALPHA | Mini-fuzz framework. | When writing property tests. |
| `macros` | ALPHA | Procedural macros (select!, stability scope). | Mostly via attributes; rarely used directly. |
| `math` | ALPHA | Math types (rationals, fields). | When you need fixed-point or field arithmetic. |
| `mcp` | ALPHA | MCP server for LLMs. | When you're using Claude/Cursor/etc. |
| `parallel` | ALPHA | Parallel fold strategies. | When you have embarrassingly parallel work. |
| `pipeline` | varies | Mechanisms under development. | When you want to try the bleeding edge. |
| `utils` | ALPHA | Common helpers (NZUsize, ordering, hashing). | Trivially. |

### Auxiliary crates

| Crate | Stability | What it does |
|---|---|---|
| `conformance` | (test-only) | Conformance testing for wire and storage formats. |
| `examples/*` | varies | Working examples — `log`, `chat`, `bridge`, `sync`, `flood`, `estimator`, `reshare`. |

## What this library is NOT

Commonware is sometimes confused with other things. Let me be precise.

### It's not a blockchain

Commonware doesn't ship a blockchain. It ships the primitives you'd use to
build one (or any other replicated state machine). Alto
(https://github.com/commonwarexyz/alto) is a working blockchain built on
top, and it's external to this repo.

### It's not a smart contract platform

Commonware doesn't have an EVM, WASM runtime, or Move interpreter. It
consensus-orders opaque hashes. You bring your own execution.

### It's not a validator client

Commonware doesn't ship a binary you run to validate some specific chain.
It ships a library you write your own binary from. Compare to:
- **`lighthouse`** — Ethereum consensus client, validates Ethereum.
- **`solana-validator`** — Solana validator, validates Solana.
- **`commonware-log`** — a toy validator that just commits to hashes.

If you want to validate Bitcoin, Ethereum, or Solana, use those clients.
If you want to build a new BFT system, use Commonware.

### It's not a consensus algorithm

Commonware ships Simplex, Aggregation, and OrderedBroadcast (three consensus
algorithms). It also ships primitives you'd use to implement more. But
it's not "the Simplex library" or "the BFT library" — it's "the library of
stuff you'd need to write any of those."

### It's not a database

Commonware's storage crate has authenticated data structures (MMR, QMDB,
MMB) but no SQL, no B-tree, no LSM. For a general-purpose database, use
something else. Use Commonware when you need authenticated storage (where
every read returns a Merkle proof).

## Why Rust?

This is a Rust library, not a C++/Go/TypeScript library. Why?

### Memory safety

Distributed systems code runs 24/7 with billions of dollars of value at
stake. A use-after-free in Bitcoin Core or a data race in Geth is a
multi-million-dollar bug. Rust's ownership and borrow checker eliminates
these at compile time.

### Zero-cost abstractions

You can write high-level trait-based code (e.g., `Automaton`,
`CertifiableAutomaton`) without paying a runtime cost. The compiler
monomorphizes generics into specialized machine code. No vtable lookups
for hot paths.

### Async/await ecosystem

Tokio is the most mature async runtime in any language. Commonware
piggybacks on Tokio for production, which gives you:
- io_uring for high-performance Linux storage and network.
- Work-stealing scheduler.
- Battle-tested timers.
- Cancellation safety.

### Strong type system for protocols

Consensus protocols have lots of invariants: "this signature was over
this exact message" or "this vote is for view 5." Rust's type system
makes these errors at compile time, not at runtime.

### Cargo and crates.io

Adding a new dependency is `cargo add foo`. Building is `cargo build
--release`. Testing is `cargo test`. The whole Commonware workspace
builds with one command and has reproducible builds.

### The trade-offs

Rust has a learning curve. If you've never written Rust, you'll spend
the first few chapters fighting the borrow checker. That's normal. The
other choice is to use Go (which Commonware's competitor Cosmos SDK does)
and accept data races, GC pauses, and weaker type guarantees.

## The deepest reading list

The actual deep dives live in `docs/blogs/`. The most foundational:

- `introducing-commonware.html` — DSMR thesis
- `commonware-the-anti-framework.html` — why no monolithic stack
- `commonware-runtime.html` — the abstract deterministic runtime
- `commonware-cryptography.html` — seeds, links, views
- `threshold-simplex.html` — the consensus algorithm
- `mmr.html`, `adb-current.html`, `adb-any.html`, `qmdb.html`, `pyramid-mmb.html`
  — the data authentication family
- `carnot-bound.html`, `constantinople.html`, `minimmit.html`, `bte.html`,
  `golden.html`, `phone-a-friend.html`, `batch-pari.html` — the research frontier

## External references to read alongside

If you want to go deeper on the underlying theory, these are the books
the next chapters draw from:

- **Designing Data-Intensive Applications** by Martin Kleppmann (DDIA) —
  the canonical reference for distributed systems thinking. We cite it
  throughout.
- **Introduction to Modern Cryptography** by Katz & Lindell — the standard
  crypto textbook. Used for chapter 04.
- **A Graduate Course in Applied Cryptography** by Boneh & Shoup — the
  more advanced cryptography text. Free online. Used for BLS12-381,
  threshold crypto, ZK proofs.
- **Computer Networking: A Top-Down Approach** by Kurose & Ross — the
  standard networking textbook. Used for chapter 05.
- **Database Internals** by Alex Petrov — distributed database design.
  Used for chapter 06 and 16.
- **Programming Rust** by Blandy, Orendorff, and Tindall — for the Rust
  patterns referenced throughout.
- **Rust for Rustaceans** by Jon Gjengset — for the advanced Rust
  patterns (Pin, async, lifetimes).

## A distributed systems primer — what we're really building

Before we go further into Commonware specifically, it's worth stepping back
and laying out the problem space. The whole library is a response to a
particular corner of a particular field. If you already know this stuff,
skim. If you're coming from a non-distributed-systems background — web
backend, embedded, game dev, ML — this section is your on-ramp.

### What a distributed system is

A distributed system is a set of **independent computers** (nodes, processes,
hosts, replicas, validators, peers — names vary) that **appear to the user as
a single coherent system**. The classic definition, from Leslie Lamport:
"a system in which the failure of a computer you didn't even know existed
can render your own computer unusable."

Why do we build distributed systems? Three reasons, in declining order of
how often they're the *real* reason:

1. **Scale** — one machine's CPU, RAM, disk, or network bandwidth isn't
   enough. Search engines (Google), social networks (Meta), payment
   processors (Visa), blockchains — all are distributed because no single
   machine can hold all the data or serve all the requests.
2. **Reliability** — one machine fails; the system keeps working. DDIA
   chapter 1 calls this "reliability." A replicated database that survives
   2 disk failures is more reliable than a single-disk database, even if
   it's a bit slower per request.
3. **Geography** — the users are in Tokyo and Berlin; you want low latency
   for both. You put a server in each region. (CDN, multi-region DB, edge
   compute.)

The fourth reason — and the one that makes Commonware specifically hard —
is **adversarial environments**: blockchains, multi-party computation,
permissioned consortia, decentralized finance. Here, the other nodes
aren't just *potentially broken*; they might be *actively malicious*.

### The four canonical problems

Every distributed systems textbook (Tanenbaum & van Steen, DDIA, Cachin
et al.'s *Introduction to Reliable and Secure Distributed Systems*) boils
the problem space down to four sub-problems. They appear, in different
guises, throughout this curriculum.

**Replication.** Multiple copies of the same data, on different machines,
for redundancy and throughput. The question: how do you keep them
consistent? (See DDIA chapter 5.) Replicated state machines (Schneider
1990) formalize this: each replica runs the same deterministic state
machine on the same inputs in the same order, so they all reach the same
state. *Consensus* (below) is how you agree on the order of inputs.

**Consensus / agreement.** Multiple nodes must agree on a value (or a
sequence of values). Paxos, Raft, PBFT, Tendermint, HotStuff, Simplex —
all consensus protocols. Crash-fault-tolerant (CFT) consensus tolerates
crashes; Byzantine-fault-tolerant (BFT) consensus tolerates lies. The
*outcome* of consensus is a *certificate*: a quorum of attestations that
proves the agreement happened.

**Fault tolerance.** The system must continue working despite component
failures. Two sub-vocabularies matter. **Crash faults** — a node stops
responding. Easier to handle; if you don't hear from a node for a
timeout, you can route around it. **Byzantine faults** — a node actively
lies, equivocates (sends different messages to different peers), or
colludes with other Byzantine nodes. Much harder; you need
cryptographic signatures and quorum overlap to detect and isolate them.

**Time and ordering.** Without a shared clock, how do two nodes know that
event A happened before event B? Lamport (1978) defined "happens-before"
as a partial order on events based on message-passing. Vector clocks
(Fidge/Mattern 1988) refined this into a per-process clock that can
detect concurrency. Logical clocks underpin every consensus protocol.
Commonware's `cryptography::transcript` (chapter 04) is essentially a
hash-based logical clock.

There's a fifth, less-canonical but increasingly important problem:
**privacy** (multi-party computation, zero-knowledge proofs). Commonware
touches this lightly (chapter 13 aggregation, chapter 20 frontier
research).

### Why distributed systems are hard — the four horsemen

DDIA chapter 8 lists the reasons. We can collapse them into four.

**1. No shared clock.** Each node has its own clock, and they drift.
NTP can synchronize clocks to within milliseconds, but that's not enough
for "transaction T happened at 12:34:56.000123456" assertions. Worse:
two nodes can see events in different orders even if they happen
simultaneously in real time. Lamport's insight: don't rely on physical
time; use logical time defined by message dependencies.

**2. No shared memory.** Nodes communicate by sending messages over a
network. The network is slow (microseconds-to-milliseconds), unreliable
(packets drop), and reorders messages. You can't read another node's
memory; you have to ask, and wait, and hope.

**3. Partial failures.** Some nodes can fail while others keep working.
You don't know which are which until you try to talk to them. And the
failure detection itself is unreliable — the network might be slow, not
the node. This is the deepest source of bugs: you can't tell the
difference between "Bob is dead" and "Bob's message is delayed."

**4. Concurrency without total order.** Two events at different nodes are
*concurrent* if neither happened-before the other. There's no global
ordering. Consensus protocols impose a total order, but at the cost of
messages, time, and liveness.

### Network vs node failures — a taxonomy

Werner Vogels (Amazon's CTO, ex-Distributed Systems textbook author)
distinguishes:

| Failure type | What happens | Detection |
|---|---|---|
| **Crash failure** | Node stops executing. | Timeout. |
| **Omission failure** | Node fails to send/receive specific messages. | Timeout or out-of-sequence. |
| **Network partition** | Group A and group B can't communicate, but each group's intra-group network works. | Both groups see each other as crashed. |
| **Byzantine failure** | Node sends arbitrary (potentially adversarial) messages. | Cryptographic verification + quorum overlap. |
| **Timing failure** | Node's clock drifts; timestamps wrong. | Out-of-band comparison (e.g., GPS, atomic clock). |
| **Response failure** | Node responds, but the response is wrong (e.g., bug, attack). | Quorum comparison + invariants. |

Commonware's P2P layer (`p2p/src/`) handles crash, omission, partition,
and Byzantine. Timing and response failures (beyond what crypto catches)
are out of scope — they're application-layer concerns.

### Real-world failure case studies — why this isn't theoretical

Three incidents worth remembering, because each one maps to a different
distributed-systems pathology Commonware tries to defend against.

**AWS S3 outage, February 28, 2017.** A single engineer mistyped a
command during a debugging session and took down a large chunk of S3
servers. Took down much of the US internet (Slack, Trello, Quora,
Medium). The post-mortem famously noted "this was a good reminder of
how much we depend on S3." Lesson: even a system designed for
five-nines availability fails when *humans* make mistakes. Commonware's
deterministic runtime and conformance tests reduce the chance of
operator-induced disaster, but they don't eliminate it.

**Cloudflare CPU exhaustion, July 2019.** A malformed HTTP request
triggered a regex backtracking bug in a Cloudflare edge module. The
CPU spike cascaded into global packet drops; Cloudflare's edge routers
started dropping legitimate traffic. Lesson: one bug in one module can
take out a global infrastructure. Commonware's `runtime` abstraction
makes it possible to *test* for these pathologies in CI, before they
ship. The `loom` and `deterministic` infrastructure lets you explore
thousands of timing/interleaving combinations.

**Ethereum DAO hack, June 2016.** A reentrancy bug in a smart contract
let an attacker drain 3.6 million ETH. Ethereum hard-forked to recover
the funds, splitting the chain into ETH and ETC. Lesson: distributed
*agreement* (consensus) doesn't fix *correctness* (the application).
Commonware's `certify` hook and the `Automaton`/`CertifiableAutomaton`
traits (chapter 11) are how you separate consensus (what everyone
agrees on) from validation (what everyone agrees is *valid*). The
DAO had no equivalent separation.

A fourth, more recent: **Solana network outage, September 2021.** A
bot tried to mint 400,000 NFTs at once; the network couldn't process
the transaction load; validators ran out of memory; the network went
offline for 17 hours. Lesson: distributed systems have to defend
against *legitimate but excessive* load, not just adversarial load.
Commonware's bandwidth model (`p2p/src/simulated/mod.rs:12-57`) lets
you test "what happens if 10MB blocks hit a 10KB/s link?"

The common thread: every distributed-systems failure has a *theory*
behind it that was published decades before the incident. FLP impossibility
(1985) explains why liveness is hard. CAP (2000) explains why
partition-tolerance forces a consistency/availability trade-off. The
Byzantine Generals paper (1982) explains why 1/3 of nodes can lie. None
of this is hand-wavy — the papers give the precise theorems.

## A brief history of distributed consensus

The algorithms Commonware ships have parents going back 50 years.
Here's the lineage, with citations.

### 1978 — Lamport: "Time, Clocks, and the Ordering of Events"

Leslie Lamport's paper introduces **happens-before** (`→`) and **logical
clocks**. Two events at the same node are totally ordered by local time.
An event at node A sending to node B is `A.send → B.recv`. The
*causal order* is the transitive closure of these relations. Two events
that aren't causally related are *concurrent* — neither happens-before
the other. The paper shows how to assign integer timestamps that
respect this order: on each event, take max(local time, max incoming
message timestamps) + 1. This is the foundation of every consensus
protocol since, including the "view number" in Simplex.

### 1982 — Lamport, Shostak, Pease: "The Byzantine Generals Problem"

The defining paper for adversarial fault tolerance. Defines the
problem: a group of generals, some loyal, some traitors, must agree on
a common plan via pairwise messages. Proves the bound: with `n` generals
and `f` traitors, agreement is possible iff `n ≥ 3f + 1`. (We walk
through the full derivation in chapter 01.) Lamport's choice of
"Byzantine" was deliberate — he was inspired by the complexity of
Byzantine politics, where conspiracies and counter-conspiracies
abound. The paper introduces the **oral messages (OM)** model and the
**signed messages (SM)** model, with different fault thresholds.

### 1985 — Fischer, Lynch, Paterson: "Impossibility of Distributed
Consensus with One Faulty Process"

The **FLP impossibility**. In a purely asynchronous network (no
bounds on message delay), with even *one* crash fault, no deterministic
consensus protocol can guarantee termination. Proof sketch: there's
always a *bivalent* configuration (one where the outcome is still
undetermined) that can be driven to either decision by adversarial
message delays. So termination requires synchrony assumptions.

The escape hatches: (a) randomization (Ben-Or 1983), (b) partial
synchrony (Dwork, Lynch, Stockmeyer 1988), (c) synchronous networks
(strong, unrealistic assumptions). Commonware's Simplex uses (b).

### 1988 — Dwork, Lynch, Stockmeyer: "Consensus in the Presence of
Partial Synchrony"

The **DLS paper**. Formalizes three system models:

- **Asynchronous**: no bounds on message delay. FLP applies; no
  deterministic consensus possible.
- **Partially synchronous**: there's a *Global Stabilization Time*
  (GST) after which all messages arrive within `Δ`, but `Δ` is
  unknown.
- **Synchronous**: messages always arrive within a known `Δ`.

The paper proves: any consensus protocol can be live in only one of
these models. Safe protocols (no forks) can be live under partial
synchrony — but not asynchronous. The famous "DLS upper bound" is
*n ≥ 2f + 1* for partially synchronous consensus — Simplex matches
this exactly.

### 1989 — Lamport: "The Part-Time Parliament" (Paxos)

Lamport's Paxos paper, written in 1989 but published in 1998 because
the reviewers thought it was a joke about a fictional parliament. Paxos
is *crash-fault-tolerant* consensus, not Byzantine. It introduced the
*prepare / accept / commit* pattern (chapter 11 cross-references this)
and the idea that progress depends on having a stable leader. Lamport
later wrote "Paxos Made Simple" (2001) — the original paper is famously
opaque. Commonware doesn't ship Paxos, but the patterns show up in
many BFT protocols (PBFT, HotStuff, Tendermint, Simplex all borrow
the leader-based structure).

### 1999 — Castro, Liskov: "Practical Byzantine Fault Tolerance"

**PBFT**. The first *practical* BFT protocol — the Byzantine Generals
solution was theoretical but O(n^f+1) messages. PBFT brings it down to
O(n²). Three phases (pre-prepare, prepare, commit), each with O(n²)
message complexity. View changes are expensive. PBFT powers systems
like IBM's Hyperledger Fabric (older versions) and is the historical
baseline for BFT performance.

### 2008 — Nakamoto: "Bitcoin: A Peer-to-Peer Electronic Cash System"

Satoshi's Bitcoin paper. Introduces *Nakamoto consensus*: probabilistic
finality via Proof-of-Work mining, longest-chain rule. This is a
fundamentally different model from PBFT-style BFT — no fixed validator
set, no cryptographic finality, but probabilistic. Commonware doesn't
ship Nakamoto consensus (it ships Simplex-style BFT), but Bitcoin's
design is what made BFT-relevant systems economically interesting.

### 2014 — Kogias, Bugnion: "Enhancing Bitcoin Security and Performance
with Strong Consistency via Collective Signing" (ByzCoin)

First popular BFT consensus layered on top of a blockchain — combines
PBFT-style agreement with Bitcoin-style PoW membership. ByzCoin used
*BLS threshold signatures* to achieve O(n) message complexity per view
(vs O(n²) for PBFT). This is the ancestor of HotStuff and Simplex.

### 2016 — Buchman: "Tendermint: Byzantine Fault Tolerance in the Age of
Blockchains"

Buchman's PhD thesis. Tendermint is a PBFT descendant with two key
improvements: (a) it uses rounds with explicit *lock* and *unlock*
phases (Polder protocol, by Pease et al. 1980 as part of the original
Byzantine work), and (b) it integrates with a blockchain app via ABCI
(Application Blockchain Interface). Cosmos SDK is built on Tendermint.
Simplex differs from Tendermint in important ways — Tendermint
requires *polka* (2f+1 of any vote) plus *polka+1* for locking, while
Simplex uses notarize/nullify as separate message types and avoids
round locking entirely.

### 2019 — Yin, Malkhi, Reiter, Gueta, Abraham: "HotStuff: BFT Consensus
with Linearity and Responsiveness"

HotStuff introduced the *chained* BFT structure: blocks form a chain
across views, and each view's certificate is also the next view's
*prepare*. This reduces the message complexity to O(n) per view and
O(log n) views per decision. Facebook's Libra (now Diem) used
HotStuff; it's also the basis for many modern protocols (Jolteon,
Pala, Chained HotStuff).

### 2023 — Das, Xiang, Ren: "Simplex Consensus: A Simple and Efficient
BFT Consensus with Safety and Liveness"

The paper behind Commonware's `consensus::simplex` crate. Simplex
simplifies HotStuff further: notarization at 2 hops, finalization at
3, with O(n) message complexity per view via BLS threshold signatures
and O(1) views under partial synchrony. The paper is ~14 pages and
quite readable; Commonware's `consensus/src/simplex/mod.rs:1-119`
documents the deviations Commonware made from the paper.

### Why each step mattered

A sketch of the dependencies:

- Lamport 1978 → provides the causal order that consensus needs.
- Byzantine Generals 1982 → provides the threat model and the 3f+1
  bound.
- FLP 1985 → proves we need synchrony assumptions.
- DLS 1988 → formalizes the synchrony spectrum and gives the 2f+1
  bound.
- PBFT 1999 → first *practical* implementation of the Byzantine
  bound.
- Tendermint 2016 → integrates locking + a blockchain app interface.
- HotStuff 2019 → linear message complexity per view.
- Simplex 2023 → simplest realization of linear + responsive + 3f+1.

Commonware's Simplex is the latest step in a 45-year chain of
refinement. The chapters that follow explain every piece of it.

## Reading order and dependencies

The 22 chapters form a directed graph, not a list. You don't have to
read them in order; the question is which order fits *your* goal.

### The dependency graph

```
              ┌────────────────────────────────────────┐
              │                                        │
              ▼                                        │
        [00 Orientation] ───────────────────┐          │
              │                             │          │
   ┌──────────┼──────────┐                  │          │
   ▼          ▼          ▼                  ▼          │
[02 Runtime][03 Codec][01 Byzantine]   [04 Crypto]   [05 P2P]
              │          │                  │          │
              │          │                  ▼          │
              │          │              [04 Crypto]    │
              │          │                  │          │
              ▼          ▼                  ▼          ▼
              [06 Storage] ─────────►[07 Coding]──────┘
              │          │                  │
              ▼          ▼                  │
        [08 Resolver][09 Broadcast][10 Collector]
              │          │                  │
              └──────────┼──────────────────┘
                         │
                         ▼
                  [11 Simplex]
                         │
              ┌──────────┼─────────────┐
              ▼          ▼             ▼
      [12 Marshal] [13 Aggregation] [14 Ordered Broadcast]
              │          │             │
              └──────────┼─────────────┘
                         │
                         ▼
              ┌──────────┼─────────────┐
              ▼          ▼             ▼
         [15 Actor] [16 Storage Adv] [17 Glue]
              │          │             │
              └──────────┼─────────────┘
                         │
                         ▼
              ┌──────────┼─────────────┐
              ▼          ▼             ▼
       [18 Deployer] [19 Examples]  [21 Cross-cutting]
                         │
                         ▼
                  [20 Frontier]
```

Lines are "depends on." Arrows go from chapter that depends to chapter
depended on.

### Three reading paths

Different goals, different paths.

**Implementer path** — you want to ship a working system on top of
Commonware. Focus on the *how* more than the *why*.

1. **00 Orientation** — the mental model.
2. **02 Runtime** — the abstract runtime (you'll use it everywhere).
3. **03 Codec** — how to put things on the wire.
4. **05 P2P** — the network layer.
5. **11 Simplex** — the consensus protocol.
6. **12 Marshal** — the bridge to your application.
7. **15 Actor** — concurrent components.
8. **17 Glue** — the standard composition.
9. **18 Deployer** — putting it in production.
10. **19 Examples** — running code you can copy.

You can skim 01 (Byzantine theory) on a first pass; come back to it
after 11 once you have context. Skim 04 (Cryptography) and 06
(Storage); the trait surfaces are what matter for implementation.

**Researcher path** — you want to understand the algorithmic
contributions and the open problems.

1. **00 Orientation**.
2. **01 Byzantine** — the theory.
3. **04 Cryptography** — the BLS threshold signatures and transcripts.
4. **06 Storage** — the authenticated data structures (MMR, QMDB, MMB).
5. **07 Coding** — erasure coding and ZODA.
6. **11 Simplex** — the consensus algorithm in depth.
7. **13 Aggregation** — BLS aggregation for bandwidth.
8. **14 Ordered Broadcast** — the partial-order primitive.
9. **20 Frontier** — the research directions.
10. **21 Cross-cutting** — patterns across primitives.

Skim 18 (Deployer) — it's production ops, not research.

**Operator path** — you want to run a Commonware system in production.

1. **00 Orientation**.
2. **02 Runtime** — the runtime choices (Tokio vs deterministic, io_uring,
   blocking pool sizing).
3. **03 Codec** — wire formats and conformance.
4. **05 P2P** — network topology, rate limits, encryption.
5. **06 Storage** — on-disk formats, journal compaction.
6. **11 Simplex** — how consensus behaves under failure.
7. **18 Deployer** — multi-region deployment.
8. **19 Examples** — `sync`, `estimator`, `reshare` for ops.
9. **21 Cross-cutting** — telemetry, metrics, tracing.

Skim 01 (theory) and 04 (cryptography math) — focus on the operational
implications, not the proofs.

### Cross-reference rules

When you see "see chapter XX," it means: chapter XX explains this
concept at length. The full cross-reference index is in chapter 21
(cross-cutting patterns).

## Glossary of acronyms

Every acronym that appears anywhere in this curriculum, with a one-line
definition and a chapter cross-reference.

### Distributed systems / consensus

- **BFT** — Byzantine Fault Tolerance. Tolerates adversarial nodes.
  Chapter 01.
- **CFT** — Crash Fault Tolerance. Tolerates only crashes. Paxos, Raft.
  Chapter 01.
- **FLP** — Fischer-Lynch-Paterson. The 1985 impossibility result.
  Chapter 01.
- **DLS** — Dwork-Lynch-Stockmeyer. The 1988 partial synchrony paper.
  Chapter 01.
- **CAP** — Consistency, Availability, Partition tolerance. The trade-off
  theorem. Chapter 01.
- **DSMR** — Decoupled State Machine Replication. The consensus-over-hashes
  idea. Chapter 00.
- **PBFT** — Practical Byzantine Fault Tolerance. Chapter 01, 11.
- **PoW** — Proof of Work. Bitcoin-style consensus. Chapter 01, 20.
- **PoS** — Proof of Stake. Modern blockchain consensus.
- **Nakamoto** — Nakamoto consensus. Probabilistic-finality PoW. Chapter 01.
- **HotStuff** — Yin et al. 2019. Linear BFT. Chapter 01, 11.
- **Tendermint** — Buchman 2016. Rounds + locking BFT. Chapter 01.
- **Simplex** — Das-Xiang-Ren 2023. Commonware's BFT. Chapter 01, 11.
- **ABCI** — Application Blockchain Interface. Tendermint's app interface.
- **DS** — Distributed System (general). Chapter 00.

### Consensus objects

- **BLS** — Boneh-Lynn-Shacham. Pairing-based threshold signatures.
  Chapter 04, 11, 13.
- **VRF** — Verifiable Random Function. Used for leader election.
  Chapter 11.
- **DKG** — Distributed Key Generation. Distributing key shares. Chapter 04.
- **KZG** — Kate-Zaverucha-Goldberg. Polynomial commitment scheme.
  Chapter 07, 20.
- **ZK** — Zero Knowledge. Proofs that don't reveal inputs. Chapter 20.
- **ZODA** — Zero-Oblivious Data Availability. Erasure-coding scheme.
  Chapter 07.
- **ZK-SNARK** / **ZK-STARK** — succinct / scalable transparent ZK proofs.
- **MMR** — Merkle Mountain Range. Append-only authenticated structure.
  Chapter 06.
- **ADB** — Authenticated Database. Commonware's authenticated KV store.
  Chapter 06.
- **QMDB** — Quorum-Merkle Database. Commonware's authenticated DB. Chapter 06.
- **MMB** — Merkle Mountain Bitmap. Authenticated bitmap for variable-bitrate
  data. Chapter 06.
- **BMT** — Binary Merkle Tree. Used in some primitives.
- **TLE** — Two-Level Encoding. ADB's encoding scheme.

### Commonware primitives

- **RS** — Reed-Solomon. Erasure coding. Chapter 07.
- **MPI** — Merkle Proof Inclusion.
- **CPI** — Commonware P2P Interface.
- **DPI** — Deterministic P2P Interface.
- **ASF** — Application Submission Filter.
- **AS** — (variable — application-specific).
- **ACID** — Atomicity, Consistency, Isolation, Durability. DB transactions.
- **BASE** — Basically Available, Soft state, Eventually consistent. AP side
  of CAP.

### Networking

- **IP** — Internet Protocol (layer 3).
- **TCP** — Transmission Control Protocol (layer 4, reliable byte stream).
  Chapter 05.
- **UDP** — User Datagram Protocol (layer 4, unreliable datagrams).
  Chapter 05.
- **QUIC** — Quick UDP Internet Connections. Modern reliable-transport-over-UDP.
  Chapter 05.
- **TLS** — Transport Layer Security. Encrypted TCP. Chapter 05.
- **OTP** — One-Time Pad. Cryptography.
- **DTLS** — Datagram TLS. Encrypted UDP. Chapter 05.
- **MTU** — Maximum Transmission Unit. Largest packet size.
- **QoS** — Quality of Service. Network priority.

### Commonware / Rust / runtime

- **API** — Application Programming Interface.
- **AST** — Abstract Syntax Tree.
- **ABI** — Application Binary Interface.
- **CPU** — Central Processing Unit.
- **GC** — Garbage Collection.
- **JIT** — Just-In-Time (compilation).
- **MIR** — Rust's Mid-level Intermediate Representation.
- **LLVM** — Low-Level Virtual Machine. Compiler backend Rust uses.
- **RFC** — Request for Comments (Rust language proposals).
- **CI** — Continuous Integration.
- **CD** — Continuous Deployment.
- **SaaS** — Software as a Service.

### Acronyms *not* used here but you'll see elsewhere

- **MEV** — Maximal Extractable Value. Blockchain ordering games.
- **DeFi** — Decentralized Finance. The application domain Commonware
  often targets.
- **NFT** — Non-Fungible Token.
- **L2** — Layer 2 (blockchain scaling).
- **EVM** — Ethereum Virtual Machine.

## Common pitfalls — what trips up newcomers

Reading distributed systems papers or Commonware code, these are the
mistakes that bite people new to the field.

**1. Assuming time is shared.** If you have `SystemTime::now()` on two
different nodes, you get two different times. They might be off by
milliseconds (NTP drift) or seconds (clock skew). Never use wall-clock
time for ordering events. Use view numbers, sequence numbers, logical
timestamps (Lamport clocks), or rounds.

**2. Confusing safety with liveness.** "Safety" means nothing bad ever
happens (no forks, no double-spends). "Liveness" means something good
eventually happens (blocks finalize, transactions confirm). A protocol
can be safe-but-not-live (you never make progress but never fork) or
live-but-not-safe (you confirm blocks but they might contradict each
other). Most BFT protocols guarantee the first always and the second
under synchrony. (Chapter 01 main text.)

**3. Ignoring message loss.** In every real network, messages drop. If
your protocol doesn't handle a missing message, it will deadlock or
fork when one does. The discipline: every protocol has timers and
recovery paths. Simplex has `t_l` and `t_a` timers; missing `notarize`
triggers `nullify`.

**4. Counting replicas instead of quorum size.** "I have 5 replicas, so
I tolerate 2 crashes." Wrong. You tolerate `floor((n-1)/2)` crashes for
CFT (with `2f+1` quorum). For BFT, you tolerate `floor((n-1)/3)`. Always
state the quorum, not just the count.

**5. Trusting your "Byzantine" node's silence.** "I haven't heard from
the bad node, so it's not doing anything." Wrong. Silence is not a
proof of honesty. A Byzantine node might be preparing a coordinated
attack (signing conflicting messages, withholding evidence). Always
require *active* evidence (signatures, certificates) before trusting a
node's claim.

**6. Treating the network as instantaneous.** "My leader proposed at
t=0; I should see it at t=0+ε." Wrong. Even on a LAN, network round-trip
is 100µs-1ms. Across the internet, 10-100ms. Across continents, 100ms+.
You *must* assume `Δ > 0` for any protocol that aims to be correct.

**7. Equating partial synchrony with synchrony.** "Partial synchrony
means messages arrive within `Δ`." Wrong. Partial synchrony means
messages *eventually* arrive within `Δ`, after some unknown time GST
(Global Stabilization Time). Before GST, messages can be arbitrarily
delayed. Commonware's Simplex is correct in both regimes — but the
behavior differs (slow progress before GST, fast after).

**8. Forgetting that "consensus" doesn't mean "the right answer."**
Consensus only means "everyone agrees." It doesn't mean "everyone
agrees on the truth." If your application logic is buggy, consensus
just propagates the bug faster. Commonware's `certify` hook (chapter 11)
is how you separate consensus (agreement) from validation (correctness).

**9. Confusing certificate construction with consensus.** A certificate
is the *output* of consensus — a set of signatures proving agreement
happened. Constructing it is a local operation (collect signatures,
verify them, batch into a single proof). Don't conflate the two phases.

**10. Using ed25519 where BLS threshold is needed.** ed25519 is great for
identity but terrible for aggregation. If you need a single signature
representing "2f+1 of these public keys signed," use BLS threshold.
Commonware's `bls12381_threshold` scheme is exactly this. (Chapter 04
and 11.)

**11. Reading the Byzantine Generals paper and assuming the proof
generalizes.** It doesn't. The `3f+1` bound is for *oral messages* and
*Byzantine agreement*. Synchronous BFT, asynchronous BFT, and
authenticated BFT have different bounds. (Chapter 01.)

**12. Treating determinism as a feature of the algorithm.** It's a
feature of the *test environment*. Commonware's `deterministic`
runtime gives you deterministic tests; Commonware's *production* code
runs on Tokio and is non-deterministic (by necessity, since it talks
to real networks with real time).

## Exercises

These check understanding before you move on. Solutions are in the
chapter footnotes / blog post follow-ups; try them closed-book first.

**Exercise 0.1 — The `3f+1` formula.** Without looking, write down the
formula `n ≥ 3f + 1` and explain in 3 sentences why it holds. Hint:
start with "if honest nodes must outvote..."

**Exercise 0.2 — Trace a 4-validator Simplex run.** With n=4, f=1,
write out the messages exchanged for a single successful view: who
proposes, who votes, when the notarization forms, when the finalization
forms. Then redo it with the leader being the Byzantine node.

**Exercise 0.3 — Map the failure case studies.** For each of AWS S3
2017, Cloudflare 2019, Ethereum DAO 2016, identify which distributed-
systems pathology it maps to. Hint: one is "operator error" (no
consensus failure), one is "application bug" (no consensus failure),
one is "load-induced" (related to liveness).

**Exercise 0.4 — Read the Lamport 1978 paper.** Get "Time, Clocks, and
the Ordering of Events" from a library or online. Skim it. Identify
where Lamport introduces the `→` operator and the logical-clock
construction. Notice that the paper is 14 pages and changes distributed
systems forever.

**Exercise 0.5 — Build the dependency graph for your use case.** Pick
one of the three reading paths (implementer, researcher, operator) and
write down *your* reading order, with one sentence per chapter on why
it comes before the next.

**Exercise 0.6 — Translate one Commonware acronym.** Pick any acronym
from the glossary (MMR, QMDB, MMB, ADB, BLS, ZODA) and write a
two-sentence explanation. If you can't, that's the chapter you should
read next.

## Appendix A — Build & test infrastructure in depth

The repo is not just source code; it's a working production toolchain. Let
me unpack the machinery.

### The `justfile` — every command you'd ever run

`justfile` (~250 lines) is the entry point for all development workflows.
The most important recipes:

```bash
just lint                       # Format + lint + stability check
just test -p commonware-cryptography        # Test one crate
just test --workspace           # Test everything
just test --features commonware-runtime/iouring-storage  # Feature-gated
just fuzz <crate>/fuzz          # 60s of fuzzing per target
just test-conformance           # Verify wire formats haven't drifted
just regenerate-conformance     # Acknowledge an intentional wire format change
just udeps                      # Find unused dependencies
just pre-pr                     # The full pre-PR checklist
just doc                        # Build docs for BETA+ only
just miri <module>::            # Run MIRI for memory safety verification
```

`just test` is a thin wrapper over `cargo nextest run` (nextest is the
modern test runner that runs tests in parallel processes, not threads).
That's why the test suite is so fast despite 1000+ benchmarks.

### The CI matrix

From `AGENTS.md`:

> - **OS**: Ubuntu, Windows, macOS
> - **Features**: Standard, io_uring storage, io_uring network (Linux only)
> - **Toolchain**: Stable (default), Nightly (formatting/fuzzing)

Three GitHub Actions workflows run on every PR:

- **Fast** — every push. Quick tests, lint.
- **Slow** — main + PR. Full test matrix including slow tests and feature
  combos. Cancellable.
- **Coverage** — main + PR. `cargo llvm-cov` report.

### The `Cargo.toml` workspace — every dependency matters

`Cargo.toml:89-207` lists ~80 direct dependencies. A few worth knowing:

- **`blst`** — Supranational's BLS12-381 implementation. Fast, audited.
- **`blake3`** — the hash function Commonware uses for transcripts. Faster
  than SHA-256, with a tree mode for parallelism.
- **`chacha20poly1305`** — symmetric encryption for stream sessions.
- **`curve25519-dalek`** — X25519 for the P2P handshake.
- **`ed25519-consensus` / `ed25519-zebra`** — two ed25519 implementations,
  chosen for performance.
- **`futures` / `futures-util`** — async primitives (chapter 02).
- **`io-uring`** — Linux io_uring bindings for high-perf storage/network
  on Linux.
- **`libfuzzer-sys`** — fuzzing harness (libFuzzer).
- **`loom`** — permutation testing for concurrent code (chapter 02).
- **`mio` / `tokio`** — async runtimes.
- **`prometheus-client`** — metrics.
- **`rand` / `rand_chacha` / `rand_core`** — RNG.
- **`rayon`** — data parallelism (the `commonware_parallel::Rayon`
  strategy).
- **`sha2`** — SHA-256.
- **`thiserror`** — error types.
- **`tracing` / `tracing-opentelemetry` / `tracing-subscriber`** —
  structured logging with OTLP export.
- **`x25519-dalek`** — X25519 for key exchange.
- **`zeroize`** — secure memory zeroing for key material.
- **`zstd`** — compression (used in the journal).

Each of these has a reason. Removing one would lose a feature or slow
something down significantly.

### The stability mechanism — in code

We saw the `ALPHA → EPSILON` levels in the README. Here's how they work in
practice.

**The macro** (`commonware-macros`):

```rust
commonware_macros::stability_scope!(ALPHA {
    pub mod my_module;          // everything in here is ALPHA
});

commonware_macros::stability_scope!(BETA, cfg(not(target_arch = "wasm32")) {
    pub mod tokio;              // BETA + only on non-WASM
});

#[commonware_macros::stability(ALPHA)]
pub fn unstable_function() { }  // individual item
```

These expand to `#[cfg(not(commonware_stability_RESERVED))]` (and
analogous for the higher levels).

**Compilation-time enforcement**:

```bash
# Your app depends only on BETA+ primitives
RUSTFLAGS="--cfg commonware_stability_BETA" cargo build -p my-app
```

If `my-app` accidentally uses an ALPHA primitive, the cfg flags hide it
from the compiler. Build fails. **You cannot accidentally use unstable
primitives in a stable build.**

This is the trick from chapter 00's "third idea" made real.

### The conformance workflow — keeping wire formats honest

From `AGENTS.md`:

```bash
# Run conformance tests
just test-conformance

# Regenerate conformance hashes after an intentional change
just regenerate-conformance
```

The flow:

1. Every primitive that implements `Encode` (chapter 03) gets a
   `conformance_tests!` invocation.
2. Tests generate deterministic values via `arbitrary`, encode them,
   hash all the bytes together with SHA-256.
3. The hash is compared to a stored value in `conformance.toml`.
4. Mismatch = wire format changed = test fails.
5. Intentional change = `just regenerate-conformance` to update the
   stored hash.

This is what protects deployed nodes from silently breaking when a new
release changes a wire format. Critical for BFT protocols where one
mismatched node can fork the chain.

### The MCP server — Commonware for LLMs

`README.md:96-105` mentions the MCP server:

> Make your LLM more effective by connecting to the Commonware Library
> MCP server. Learn more here.

The MCP server (`mcp/`) exposes Commonware's documentation, primitives,
and code patterns to LLMs via the Model Context Protocol. So if you use
Claude Code or Cursor, you can connect `https://mcp.commonware.xyz` and
the LLM gets first-class knowledge of Commonware.

That's meta — Commonware helps LLMs understand Commonware.

### The `pipeline` directory — mechanisms under development

`README.md:57`:

> **pipeline**: Mechanisms under development.

What's in here changes release to release. It's the staging area for
primitives that aren't ready for the main crates yet. Things graduate
from `pipeline` to a proper crate once they hit BETA stability.

### What I missed the first time

In the original chapter 00, I listed the crates and the big ideas but
skipped:
- The build/test machinery (`justfile`, `nextest`, fuzzing).
- The exact stability-macro mechanism.
- The conformance workflow in detail.
- The MCP integration.
- The pipeline directory.
- The dependency story (`blst` vs other BLS libs, `blake3` vs `sha2`, etc.).
- The CI matrix.

These matter because they explain **why Commonware looks the way it does.**
The CI matrix, the stability mechanism, the conformance tests — they're
all enforcement of "code that ships must be code that works."

## If you only remember three things

1. **Commonware is Lego, not a kitchen.** Pick the primitives you want. Skip the ones you don't.
2. **Consensus runs over hashes.** What the hashes point to is your problem.
3. **The runtime is abstract.** That's why everything can be tested deterministically.

→ Next: **Chapter 01 — Byzantine Fault Tolerance**. Open `chapters/01-byzantine.md`.