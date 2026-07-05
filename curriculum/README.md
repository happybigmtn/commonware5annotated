# Commonware Annotated

> **A fully self-contained learning guide to distributed systems, told through the lens of the Commonware library.**

This is the annotated companion to [Commonware](https://commonware.xyz) — a Rust
library of distributed-systems primitives for building Byzantine fault-tolerant
applications. We read every primitive in the library, walk through every
consensus algorithm, follow every data structure from disk to wire, and explain
every concept from first principles with references to canonical sources.

## Who this is for

You are:

- A Rust developer who wants to learn distributed systems through a real,
  production-grade codebase rather than a textbook.
- A distributed-systems engineer who wants to see how a modern library
  implements the theory you already know.
- A researcher who wants a deep, opinionated tour of the Commonware design
  choices before you read the source.
- A student who has heard of "Byzantine consensus" and wants to understand it
  deeply enough to actually build something.

You are **not** required to know Rust, consensus algorithms, or BFT in
advance. We teach what we use, and we link to canonical references (DDIA,
Boneh & Shoup, Kurose & Ross, RFCs, the original papers) when depth matters.

## How to read this book

There are **six phases**, **twenty-two chapters**, and **~55,000 lines** of
content. You don't have to read it linearly.

### Three reading paths

**Path A — Implementer.** You want to write Commonware code.

```
00 → 01 → 02 → 03 → 04 → 05 → 06 → 11 → 12 → 15 → 17 → 19 → 21
```

You'll learn enough to read `examples/log` and start building your own app.
About 16 hours of reading.

**Path B — Researcher.** You want to understand the frontier.

```
00 → 01 → 02 → 04 → 07 → 11 → 13 → 14 → 17 → 20
```

You'll learn the Simplex paper, ZODA, Minimmit, BTE, Constantinople, the
Carnot bound. About 14 hours.

**Path C — Operator.** You want to deploy Commonware.

```
00 → 02 → 05 → 06 → 11 → 12 → 17 → 18 → 21
```

You'll learn P2P, storage, Marshal, Glue, the Deployer, observability.
About 12 hours.

If you're reading for the first time and have time, **read linearly** — the
chapters build on each other.

### Time investment

| Chapter size | Time to read carefully | Time to skim |
|---|---|---|
| Small (~1,200 lines) | 1 hour | 15 min |
| Medium (~2,000 lines) | 2 hours | 30 min |
| Large (~3,000 lines) | 3 hours | 45 min |
| Very large (~3,500 lines) | 4 hours | 60 min |

Full linear read: ~50 hours.

### What's in each chapter

Every chapter has:

1. **Main body** — Feynman-style walkthrough. Read this first.
2. **Exercises** — hands-on tasks at the end. Do these to internalize.
3. **Appendices** — deeper technical content (math, internals, edge cases).
   Read when you want to build something, debug something, or understand
   the underlying theory.

The appendices reference the same `file:line` locations as the main text, so
you can read both side-by-side with the source open.

### Chapter 21 is special

Chapter 21 — Cross-cutting patterns is **not part of the linear reading order**.
It's the reference card: glossary of acronyms, the full `Context` API, the
mailbox API, the simulated network API, Byzantine test patterns, namespace
conventions, error handling rules, tracing rules, common gotchas, Rust
patterns reference. Skip to it when you start writing code.

## Conventions used throughout

- **`file:line`** references let you jump straight into the source.
- **Blockquotes with 💡** are the "this is the one insight" callouts.
- **Analogies** are deliberately informal. If one doesn't click for you,
  find another.
- Each chapter ends with a short **"If you only remember three things"** summary.
- **Common patterns** sections show how the primitives compose in real code.
- **Common gotchas** sections capture pitfalls the docs sometimes gloss over.
- **Exercises** at the end are graded easy → impossible.

## Prerequisites

We don't assume much. We do assume:

- Comfort reading Rust. The Async Rust book
  (https://rust-lang.github.io/async-book/) is a great primer.
- Some familiarity with TCP/IP, public-key cryptography, and B-trees at the
  "Wikipedia level." We deepen every concept we use.
- Willingness to look at code. We cite `file:line` references — read them.

If you don't have these, read the chapters in order. The Distributed Systems
Primer in chapter 00 covers what a Rust developer needs to know about
distributed systems; the Rust primer in chapter 02 covers what a distributed
systems developer needs to know about Rust.

## Reference sources we draw from

The curriculum integrates material from:

- **DDIA — Designing Data-Intensive Applications** by Martin Kleppmann
  (chapters 1, 5, 7, 9 most heavily)
- **Boneh & Shoup — A Graduate Course in Applied Cryptography**
  (chapters on number theory, ECC, pairings, threshold crypto, ZK)
- **Katz & Lindell — Introduction to Modern Cryptography**
  (modern crypto textbook reference)
- **Kurose & Ross — Computer Networking** (OSI model, TCP/UDP, congestion)
- **Petrov — Database Internals** (LSM trees, B-trees, WAL, ARIES)
- **van Steen & Tanenbaum — Distributed Systems** (DHT, gossip, quorum)
- **Hewitt & Agha — Actor model** (1973, 1986)
- **Hoare — CSP** (1978)
- **Simplex paper** — Das-Xiang-Ren 2023 (eprint.iacr.org/2023/463)
- **PBFT paper** — Castro-Liskov 1999
- **Tendermint paper** — Buchman 2016
- **HotStuff paper** — Yin et al. 2019
- **Bracha's broadcast** — 1987
- **DLS impossibility** — Dwork-Lynch-Stockmeyer 1988
- **FLP impossibility** — Fischer-Lynch-Paterson 1985
- **Byzantine Generals** — Lamport-Shostak-Pease 1982
- **Malkhi-Reiter Byzantine quorum** — 1998
- **Kademlia** — Maymounkov-Mazières 2002
- **RFC 9162** — Certificate Transparency v2.0
- **RFC 9000** — QUIC
- **RFC 9221** — HTTP/3 datagrams
- **RFC 1700** — Network byte order
- **EIP-6800** — Verkle tree migration
- **The Commonware docs** — https://commonware.xyz/docs/blogs/

## The curriculum at a glance

### Phase 1 — Foundation

| # | Chapter | Lines | One-liner |
|---|---|---|---|
| 00 | Orientation | 1,235 | What Commonware is, DSMR, anti-framework thesis, distributed systems primer. |
| 01 | Byzantine Fault Tolerance | 1,269 | `n ≥ 3f + 1`, FLP, DLS, the Byzantine Generals paper, BFT comparison. |

### Phase 2 — Substrate

| # | Chapter | Lines | One-liner |
|---|---|---|---|
| 02 | Runtime | 2,087 | Abstract async + the deterministic testing trick + Rust for distributed systems. |
| 03 | Codec | 2,647 | How bytes become typed messages + schema evolution + format comparison. |
| 04 | Cryptography | 3,267 | Number theory, ECC, BLS12-381, threshold sigs, DKG, quantum resistance. |
| 05 | P2P | 2,728 | TCP/IP, X25519+ChaCha20-Poly1305, NAT traversal, gossip. |
| 06 | Storage | 3,156 | LSM, B-trees, WAL, ARIES, crash consistency, compression. |

### Phase 3 — Building blocks

| # | Chapter | Lines | One-liner |
|---|---|---|---|
| 07 | Coding | 2,906 | Information theory, Reed-Solomon, Hadamard, ZODA, network coding. |
| 08 | Resolver | 1,870 | Content-addressable storage, DHTs, BitTorrent, IPFS, CDN. |
| 09 | Broadcast | 1,851 | Epidemic algorithms, gossip, bandwidth modeling, HTTP/3. |
| 10 | Collector | 2,065 | Quorum systems, monitor pattern, cancel semantics. |

### Phase 4 — Consensus

| # | Chapter | Lines | One-liner |
|---|---|---|---|
| 11 | Simplex | 3,641 | The consensus algorithm itself + comparison to PBFT, Tendermint, HotStuff. |
| 12 | Marshal | 2,284 | Bridge between consensus certificates and application blocks. |
| 13 | Aggregation | 2,138 | BLS aggregation, certificate transparency, cross-chain bridges. |
| 14 | Ordered Broadcast | 2,127 | Total order broadcast theory + DAG consensus comparison. |

### Phase 5 — Composition

| # | Chapter | Lines | One-liner |
|---|---|---|---|
| 15 | Actor | 2,922 | Actor model, CSP comparison, Erlang/OTP, lock-free structures. |
| 16 | Storage Advanced | 2,034 | B-trees, LSM, Merkle, Verkle, MMB, Patricia tries. |
| 17 | Glue | 3,194 | Default stateful app on QMDB + simulate harness. |

### Phase 6 — Operations & research frontier

| # | Chapter | Lines | One-liner |
|---|---|---|---|
| 18 | Deployer | 2,946 | AWS Regions/AZs/VPC/EC2/S3/IAM, CI/CD, disaster recovery. |
| 19 | Examples | 2,771 | Alto, Battleware, every `examples/*` walked through. |
| 20 | Frontier | 3,185 | Carnot Bound, Minimmit, MMB, BTE, Constantinople, Golden. |
| 21 | Cross-cutting | 2,924 | Reference card. Acronyms, Context API, mailboxes, test patterns, Rust patterns. |

**Total: 55,247 lines, ~50 hours of careful reading.**

## Where to start the Commonware source

If you're reading this book alongside the Commonware source, the most
useful starting point is the workspace root at
`https://github.com/commonwarexyz/monorepo`. The relevant directories:

| Source path | Curriculum chapter |
|---|---|
| `runtime/` | 02 |
| `codec/` | 03 |
| `cryptography/src/bls12381/` | 04 |
| `p2p/src/authenticated/` | 05 |
| `p2p/src/simulated/` | 05, 21 |
| `storage/src/journal/` | 06 |
| `coding/src/` | 07 |
| `resolver/src/` | 08 |
| `broadcast/src/buffered/` | 09 |
| `collector/src/` | 10 |
| `consensus/src/simplex/` | 11 |
| `consensus/src/marshal/` | 12 |
| `consensus/src/aggregation/` | 13 |
| `consensus/src/ordered_broadcast/` | 14 |
| `actor/src/mailbox.rs` | 15 |
| `storage/src/merkle/`, `storage/src/qmdb/`, `storage/src/mmb/` | 16 |
| `glue/src/stateful/`, `glue/src/simulate/` | 17 |
| `deployer/src/` | 18 |
| `examples/log/`, `examples/chat/`, etc. | 19 |
| `docs/blogs/*.html` | 20 |

## Reading progress

See `progress.md` for the full per-round history.

## A note on style

This book was written by reading the Commonware source. Every chapter is
backed by `file:line` references into that source. If you spot an error
or an inaccuracy, please open an issue at
`https://github.com/happybigmtn/commonware5annotated`.

---

→ Start with **Chapter 00 — Orientation** (`chapters/00-orientation.md`).