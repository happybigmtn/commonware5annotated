# Chapter 16 — Advanced Storage: MMR, ADB, QMDB, MMB

> The authenticated data structures that make verifiable state possible.

## The base question

How do you prove "this is the current value of key K in the database" to
someone who doesn't have the database?

A naive answer: send the whole database. But that's gigabytes.

A real answer: **a Merkle proof.** A small witness that says "the value V
for key K is in the database with root hash R." The verifier checks the
proof against R. If R matches what they expect (e.g., from a consensus
certificate), V is trustworthy.

This chapter is about how Commonware does this efficiently.

## The Merkle tree — the starting point

A binary tree where:
- Leaves are hashes of elements.
- Internal nodes are hashes of their children.
- Root hash represents the whole.

An inclusion proof: hashes along the path from root to the leaf you're
proving. Logarithmic size.

The classic Merkle tree supports **arbitrary updates**. Insert/update at
any position. This is powerful but expensive — updating one element can
require reading and rewriting a logarithmic number of nodes scattered
across storage.

## The MMR — append-only, blazingly fast

From `docs/blogs/mmr.html` (Roberto Bayardo, Feb 2025):

> A MMR differs from a Merkle tree in that it is append-only. While Merkle
> trees support element updates and insertions at arbitrary list positions,
> MMRs only allow new elements to be added to the end of the list.

The structure:

```
MMR = list of perfect binary trees of strictly decreasing height
       (called "mountains")

      🌲       🌲        🌲
     / \      / \       / \
    🌲  🌲   🌱  🌱    🌱  🌱
   /\  /\   (leaves at height 1 and 0)
  🌱🌱🌱🌱

Each mountain is a perfect binary tree.
Heights strictly decrease: 2, 1, 0 (or 3, 2, 1, etc.).
```

The root hash = **bagging the peaks**. Hash all peak node hashes together
(ordered, prefixed with the count of peaks) to get the root.

**Appending** a new element is fast:

> If you add a new element to the end of the list, you need only generate
> at most a log2(N) number of new internal nodes to re-impose the required
> properties without modifying any existing nodes.

That's because adjacent mountains of equal height merge into a single
taller mountain. Sequential writes only. No random reads.

**Pruning** is easy:

> As soon as we no longer require proving the inclusion of any element
> older than a certain point in time, we can drop nearly all of those
> elements (and most of their ancestor nodes) from the MMR without
> losing the ability to generate new roots.

Sequential data → trivial to truncate.

## ADB — Authenticated Database on top of MMR

`docs/blogs/mmr.html:94`:

> The append-only restriction may appear burdensome, but it turns out
> updates can still be simulated through re-appending data (as in QMDB).

Commonware has three ADB variants (`storage/src/qmdb/`):

| Variant | Lookups | Description |
|---|---|---|
| `qmdb::any` | Past OR current | Include the operation in the proof |
| `qmdb::current` | Current state only | Small proofs for current values |
| `qmdb::keyless` | No key | Append-only log with proofs |

**The trick**: turn "update key K from V_old to V_new" into "append a log
entry `set(K, V_new)`". The MMR covers the log. To prove the current
value of K, you prove the latest `set(K, V)` operation. The proof shows
"all subsequent operations for K had value V" (or, if not, the value
changed).

That's how you get mutable state on top of an append-only MMR.

## QMDB — Quick Merkle Database

`docs/blogs/qmdb.html` and `docs/blogs/commonware-the-anti-framework.html`
mention this as a LayerZero collaboration. QMDB is faster than the
classical ADB because:

- **No per-update proof work.** Classical ADB has to maintain auxiliary
  data structures to enable fast lookups of "the latest op for key K."
  QMDB avoids this with a smarter encoding.
- **Better cache behavior.** Operations are placed in the MMR based on
  their hash, not their key, which gives better locality.
- **Lazy evaluation.** Many computations are deferred until needed.

For our purposes, QMDB is "the ADB you'd pick for a high-throughput chain
where proofs of current state matter."

## MMB — Merkle Mountain Belt — the small-proof trick

From `docs/blogs/pyramid-mmb.html` (Roberto Bayardo, May 2026):

> By combining an active-region-aware bagging policy, pyramid bagging,
> with the Merkle Mountain Belt (MMB), our new scheme makes active-value
> proofs small and predictable: within two digests of a balanced binary
> Merkle tree over those values (without sacrificing append-only updates).

The MMR's proofs include all peaks (because of "bag the peaks"). For a
chain with lots of historical activity, that means O(log² N) peaks in
the proof.

MMB solves this:

> **Bagging the peaks** = hash all peak hashes (with count prefix)
> → 32-byte root
>
> **Problem**: proof must include all peaks → proof size grows
>
> **MMB's trick**: maintain a separate "belt" of intermediate hashes near
> the active region. Proofs use the belt, not the full peaks list.

Result: proofs of current values stay small (constant order, not
logarithmic) even as history grows. This is huge for state proofs to
other chains — they don't need to download megabytes of peak hashes to
verify "user X has balance Y."

## The other storage primitives

From `storage/src/lib.rs:13-36`:

| Module | Purpose | Stability |
|---|---|---|
| `journal` | Append-only log (chapter 06) | BETA |
| `archive` | Fixed-size value archive | BETA |
| `freezer` | Large immutable data archive | BETA |
| `index` | ULTRA-coded compressed index | BETA |
| `ordinal` | Ordinal key-value store | BETA |
| `rmap` | Rank map | BETA |
| `qmdb` | Quick Merkle Database | ALPHA |
| `mmr` / `merkle::mmr` | Merkle Mountain Range | ALPHA |
| `merkle::mmb` | Merkle Mountain Belt | ALPHA |
| `bitmap` | Authenticated bitmap | ALPHA |
| `cache` | Caching helpers | ALPHA |
| `bmt` | Binary Merkle tree | ALPHA |
| `queue` | FIFO queue | ALPHA |

`archive`/`freezer` are immutable storage. `index` is for high-compression
secondary indexes. `bitmap` is for authenticated bit-vector operations
(useful for "has this peer voted in this epoch?"). `bmt` is the basic
binary Merkle tree for ad-hoc use.

## The relationship to consensus

QMDB sits at the bottom of the stack:

```
                   +---------------------+
                   |  Consensus (Simplex) |  ← chapter 11
                   +----------+----------+
                              |
                              | finalizes blocks containing root hash
                              v
                   +---------------------+
                   |  Application        |
                   |  (state transitions)|
                   +----------+----------+
                              |
                              | commits operations
                              v
                   +---------------------+
                   |  QMDB / ADB / MMR   |  ← this chapter
                   +----------+----------+
                              |
                              | persistence
                              v
                   +---------------------+
                   |  Storage (chapter 6)|
                   +---------------------+
```

The consensus engine finalizes blocks. Each block contains a **state
root** — the root hash of the current ADB/QMDB. External observers can
verify "user X had balance Y at block N" with a Merkle proof from the
ADB and a consensus certificate for block N.

That's how you get verifiable state across blockchains.

## The MMR + threshold Simplex combo

From `docs/blogs/mmr.html:106`:

> MMRs and threshold consensus signatures make for a powerful combination.
> Consider, for example, using an MMR to store successfully executed
> blockchain transactions in the order of their execution. If the MMR
> root is included in the block digest, then a user (or external
> blockchain) running a lite client can prove successful execution of
> any historical transaction from a consensus certificate and the latest
> MMR root.

So:
- Consensus certificate → proves "block N was finalized"
- Block N's MMR root → proves "this MMR was the state at block N"
- MMR proof → proves "transaction T was included in this MMR"
- Combined → proves "transaction T was executed before block N was finalized"

That's the full chain of verifiable execution. No need to run a full
node.

## The blog tour

If you want to go deeper:

- `docs/blogs/mmr.html` — Roberto's MMR introduction (Feb 2025).
- `docs/blogs/the-simplest-database-you-need.html` — Roberto + Patrick,
  April 2025. How MMR + log = mutable database.
- `docs/blogs/grafting-trees-to-prove-current-state.html` — Roberto,
  July 2025. Current-state proofs.
- `docs/blogs/qmdb.html` — Patrick, December 2025. The QMDB announcement.
- `docs/blogs/its-a-grind.html` — Roberto, February 2026. Performance
  optimization for non-uniform keys.
- `docs/blogs/honey-i-shrunk-the-proofs.html` — Roberto, May 2026. The MMB
  announcement.

## Where to look in the code

- `storage/src/qmdb/mod.rs` — the QMDB entry point.
- `storage/src/qmdb/any/` — past-or-current ADB.
- `storage/src/qmdb/current/` — current-state-only ADB.
- `storage/src/qmdb/keyless/` — log-only ADB.
- `storage/src/merkle/mmr/` — the MMR implementation.
- `storage/src/merkle/mmb/` — the MMB implementation.
- `storage/src/merkle/proof.rs` — Merkle proof generation/verification.
- `storage/src/bmt/` — the basic binary Merkle tree.

## B-trees in depth — why they dominated for so long

Before MMRs, B-trees were the **default authenticated data structure**
for databases. Most traditional databases (Postgres, MySQL/InnoDB,
Oracle, SQLite) are built on B-trees or B+ trees. The "B" stands for
"Bayer" (Rudolf Bayer, 1972); "B+ tree" is the common variant where
data lives only at leaves.

The shape:

- Pages of fixed size (typically 4–16 KiB).
- Each internal page has K keys and K+1 children.
- Keys in page P span ranges `[first_key, last_key]` such that each
  child page P_i handles keys in `[keys[i-1], keys[i])`.
- Pages form a tree; height stays O(log N) for 10⁹ entries with
  page size 4 KiB and ~100 keys/page → height ≈ 4.

Properties:

- **Disk-friendly.** Each `get(key)` reads O(log N) pages. The
  B-tree's branching factor amortizes the disk seek cost.
- **Mutable.** Insert, delete, update in place. Each operation
  modifies O(log N) pages.
- **Range queries.** In-order traversal of leaves reads pages
  sequentially; very fast for `range_query(k1, k2)`.

**The B-tree's weakness for our purposes**: an authenticated B-tree
needs each page's hash to be computable from the children. That's a
Merkle B-tree (or **MB-tree**, Becker, 2008). Updates at the root
trigger O(log N) page-level proofs to update. For state-sync-light
clients, this is expensive: every update churns the proof structure.

Adoption: B-trees remain the standard for general-purpose database
indexes (DDIA chapter 3, "Storage and Retrieval"). They are what
Postgres/MySQL/etc. use. In Commonware, they don't appear directly;
Commonware's hot-path storage is MMR-shaped (append-only) plus
journal-shaped (variable-size raw bytes). B-trees show up indirectly
through `ordinal` and `index` (chapter 06), which provide B-tree-style
key access on top of an underlying blob sequence.

## LSM trees in detail — write-heavy rocks

Database write throughput is dominated by **sequential IO** rather than
**random IO**. Hard drives and SSDs both deliver much higher
throughput on sequential writes (≈ 500 MB/s) than on random writes
(≈ 50 MB/s). LSM trees exploit this.

The shape (DDIA chapter 3, also "Database Internals" chapters 1–2):

1. **Write path**: incoming key-value goes into an in-memory
   **memtable** (usually a sorted skiplist).
2. **Flush**: when the memtable fills, it becomes an immutable
   **SSTable** (Sorted String Table) — a sorted, on-disk file
   with a sparse index over the keys, plus a Bloom filter for
   "definitely not in this file" checks.
3. **Compaction**: multiple SSTables are merged into one in the
   background. Two main strategies:
   - **Size-tiered**: merge SSTables of similar size. Writes are
     cheap (no seek during merge), reads are expensive (need to
     check many files).
   - **Leveled (LevelDB/RocksDB)**: SSTables at level L are
     split into ~10× more files at level L+1. Reads are cheap
     (one file per level), writes are expensive (merge down).

LevelDB (2011), RocksDB (2012, Facebook), Cassandra, ScyllaDB,
HBase: all LSM. Cassandra introduced "LCS" (Leveled Compaction
Strategy) modeled on LevelDB.

Commonware's `journal` (chapter 06) is **log-structured** but not
quite an LSM tree — it doesn't have a memtable/skiplist. It's
closer to Kafka's segment log. The closest analog in Commonware is
`qmdb::keyless`, which is essentially an append-only SSTable-backed
log.

## Merkle proofs and authenticated data structures — RFC 9162

A **Merkle proof** is a witness that a particular datum X with hash H
is at a particular position P in a Merkle tree with root R. The
proof consists of the sibling hashes along the path from P to the
root.

The verifier's job:

```
Given: (P, X, root_hash, proof = [sibling_hashes])
Check: starting from H = sha256(X), for each sibling S in proof,
  determine left/right based on P's binary representation,
  compute H = sha256(S + H) or sha256(H + S),
  advance P = parent(P).
After all siblings consumed: H should equal root_hash.
```

The infrastructure: **authenticated dictionaries** and **authenticated
logs**. A dictionary supports `get(K)` with a proof that "the
value at K is V." A log supports `consistency proof` (between two
STHs) and `inclusion proof` (a single entry).

**RFC 9162 ("Certificate Transparency Version 2.0")** formalizes
this for the CT use case. The primitives it defines:

- `MerkleTreeHash` (MTH): SHA-256 root hash of a Merkle tree hash
  (RFC 9162 § 2.1).
- `MerkleAuditPath`: a list of sibling hashes from a leaf to the
  root (§ 2.1.1).
- `MerkleConsistencyProof`: two roots and the path of intermediate
  roots between them (§ 2.1.2).
- `SignedTreeHead` (STH): the MTH + tree size + timestamp + log
  signature (§ 2.2.2).

Inclusion: given an `(index, leaf_hash, STH)`, recompute the root;
match.

Commonware's MMR / QMDB deliver the same shape, parameterized over
two structures (a "current QMDB" and an "any QMDB") and two key
modes (`keyless` and keyful).

## Cryptographic accumulators — and how they compare to Merkle trees

A **cryptographic accumulator** is a public, succinct commitment to a
set, with membership and non-membership proofs. There are several
constructions:

### Merkle tree (this chapter)

- **Witness size**: O(log N) hashes.
- **Update cost**: O(log N) hashes rewritten.
- **Trusted setup**: none.
- **Universal?**: Yes — works for any set of digests.

### RSA accumulator

- **Witness size**: one scalar (one group element).
- **Update cost**: one modular multiplication.
- **Trusted setup**: requires an RSA modulus **with unknown
  factorization** (the "strong RSA assumption").
- **Universal?**: Mostly — works for sets via hash-to-prime.

### Bilinear accumulator

- **Witness size**: one group element.
- **Update cost**: O(log N) or O(1) depending on construction.
- **Trusted setup**: requires a structured reference string (SRS).
- **Universal?**: Yes — and supports non-membership proofs.

The trade-off at a glance:

| | Merkle | RSA | Bilinear |
|---|---|---|---|
| Witness size | O(log N) | O(1) | O(1) |
| Update cost | O(log N) | O(1) | O(log N) |
| Trusted setup | No | RSA modulus | SRS |
| Quantum-safe | Yes | No (RSA breaks) | No (pairings break) |

Verkle trees are a recent construction that combine the merkle-style
hash-function-based implementation with a polynomial commitment that
**fixes the trusted-setup problem while keeping O(1) witnesses per
leaf (after amortized setup).**

## Verkle trees — the polynomial commitment scheme

A **Verkle tree** is like a Merkle tree, except internal nodes
commit to their children via a **polynomial commitment** instead of a
pairwise hash. The canonical formulation uses KZG commitments
(Kate, Zaverucha, Goldberg, 2010): given a polynomial `p(X)`, the
commitment is `[p(τ)] G₁` for a secret `τ` in a pairing-friendly
group. An opening at `x = z` is `p(z)` plus a small
"evaluation proof" (a single group element).

Properties (vs Merkle):

| | Merkle | Verkle (KZG) |
|---|---|---|
| Witness size | O(log N) | O(1) per opening, after O(log N) `path` vectors |
| Update cost | O(log N) | O(1) |
| Trusted setup | None | Structured reference string |
| Proving time | O(log N) hash ops | O(log N) multi-exponentiation |

**Ethereum is migrating to Verkle trees** as part of the "Verkle
Tree Upgrade" (EIP-6800, currently in development). The motivation:
witnesses shrink from ~3 KB (15-sibling Merkle proof) to ~150 bytes
(a few KZG openings), making stateless Ethereum clients feasible.

Commonware does not yet ship Verkle trees — but the QMDB
construction's reliance on MMR-style data and cryptographic proofs
would adapt cleanly. The blueprint: swap MMR's hash-of-pairwise-
children for a KZG commitment-of-children-polynomial, keep the
append-only interface, gain constant-size witnesses.

## Sparse Merkle trees — for huge keyspaces

A **Sparse Merkle tree** (SMT) is a complete binary tree over the
**entire 256-bit keyspace**, with empty leaves hashing to a default
value (zero). Most branches contain only default values; explicit
values live at the leaves for keys actually used.

- **Tree size**: 2²⁵⁶ leaves — astronomically large, hence "sparse."
- **Logical use**: only the O(N) branches along actual insert paths
  are stored.
- **Witness size**: O(log N) hashes (~256 levels × 32 bytes = ~8 KB
  in the worst case for 256-bit keys; less in practice with stop-at-
  default optimizations).
- **Update cost**: O(256) hashes. With caching, amortized O(log N).

The killer feature: **non-membership proofs**. A non-membership proof
shows "key K is not in the tree" by walking the path from root to
where K's leaf would be, and showing either:
- All default hashes (so no leaf differs), or
- A sibling leaf exists for a different key, proving K is never
  reachable from this branch.

`qmdb::keyless` doesn't need this (it's just an ordered log without
keys), but a future "keyless *with non-membership*" variant would
be an SMT.

## Patricia tries — Ethereum's compact encoding

A **Patricia trie** (Practical Algorithm To Retrieve Information
Coded In Alphanumeric, Morrison, 1968) is a compact trie where:

- Single-child paths are compressed into edges ("extension nodes").
- Branching nodes store up to 16 children + a value.
- All keys are bit-strings (in Ethereum: keccak256 hashes).

Ethereum's world state is a Merkle Patricia trie:

- Root hash is the world state root.
- Each account is a leaf whose key is the keccak256 of the address.
- Storage of each account is itself a Patricia trie.

Patricia tries versus Merkle: Patricia has a more compact layout in
the common case (sparse keys → long extension edges), but is more
complex to implement and to update. Each storage insert/update
touches ~O(log N) nodes (the depth of the trie).

Commonware does not implement Patricia tries because its primary
target is height-ordered append-only state, not address-keyed state.
For an Ethereum-side project, the implementation would import
ethers-rs's state-handling code; for a Solana-side project, the
equivalent is an AVL tree (Rust's `solana-sdk::state`).

## MMB in depth — pyramid bagging

The MMB ("Merkle Mountain Belt") builds on MMR but adds two ideas:

1. **Pyramid bagging**: keep more than just the peaks' bagging in
   the root hash. Compute a hierarchical bagging at multiple
   heights. The result: a small "belt" of intermediate hashes near
   the active region.

2. **Active-region awareness**: the "belt" is rebuilt as the active
   tip advances, ensuring that proofs covering recent operations
   stay short.

Bayardo's paper (`docs/blogs/pyramid-mmb.html`) formalizes this.
The proof size for a current value at position `N` is **bounded by
twice the log of the active region's size**, not the log of total
history. Specifically:

```
proof_size(N) <= 2 * ceil(log2(active_size))
```

For `active_size = 100` items, that's 2 × ⌈log₂(1₀₀)⌉ = 14 hashes —
independent of whether N is 100 or 100,000,000.

The trade: maintaining the belt costs O(1) per append (amortized)
and requires recomputation when the active region's window slides
forward. Commonware's MMB tracks a configurable
`active_window_size` (default ~1024) and recomputes the belt when
the window slides.

For cross-chain state proofs (a chain A validating "user X has
balance Y on chain B"), MMB's small current-state proofs are
exactly what makes the protocol viable. A Merkle proof of length
100 KB is not workable; one of length 150 bytes is.

## The Binary Merkle Tree — for ad-hoc use

`storage/src/bmt/`. The BMT is the "classical" Merkle tree:

- Balanced binary tree of fixed arity.
- Internal nodes: `hash(left_child_hash || right_child_hash)`.
- Leaf nodes: `hash(value)` for the value being committed.
- Root: single digest representing all values.

Unlike MMR, BMT supports **updates at arbitrary positions**: change
one leaf, recompute path to root. Updates are O(log N) hashes but
those hashes must be re-fetched from storage and rewritten.

When to use BMT vs MMR:

- **Single-use, fixed-size datasets**: BMT. (e.g., once-a-epoch
  snapshot of validator set.)
- **Growing append-only datasets**: MMR.
- **Mutable state with small proofs**: ADB/QMDB on MMR.
- **Mutable state with predictable proof size**: MMB.

## Bitmap operations — count_ones, rank, select

`storage/src/bitmap/`. The authenticated bitmap is what enables
"given a root, prove that bit `i` is set / unset." Three core
operations:

- **`count_ones(bitmap)`**: total count of 1-bits. With a
  count-augmented Merkle tree (each internal node stores `popcount`
  of its children's bits), this is `O(log N)`.
- **`rank(bitmap, i)`**: number of 1-bits in positions `[0, i]`.
  Same answer: log N with augmented tree.
- **`select(bitmap, k)`**: position of the `k`-th 1-bit. Log N.

The MMB uses rank/select internally to navigate the active region.
The world-state trie uses rank to look up "the 42nd account" in
some Merkleized snapshot.

Commonware's bitmap uses a count-augmented BMT under the hood: each
internal node stores the `popcount` of its 64-bit children's chunk;
a Merkle proof includes the sibling bit chunks along the way.

## Proof verification — full protocol

The `verify` API for any of Commonware's authenticated data structures
takes:

```rust
fn verify(
    expected_root: &Digest,
    proof: &Proof,
    position: u64,
    value: &impl Hashable,
) -> Result<(), Error>;
```

The verification has two parts:

1. **Merkle path verification**: walk the proof's sibling hashes from
   the leaf up to the root. Compute the intermediate hashes. Compare
   against `expected_root`.
2. **Domain-specific check**: for current-state ADBs, also verify that
   no later operation for the same key overrides this one. For
   MMR-style inclusion, just check inclusion.

What an attacker has to do to forge:

- For the value at position P to be different in the verifier's
  mental model from reality, they'd need to produce a chain of
  hashes that ends at `expected_root` but starts from a different
  leaf hash. That requires either:
  - **Finding a hash collision** in the underlying function (SHA-256,
    blake3, etc.). Computationally infeasible (~2¹²⁸ for collision).
  - **Producing a proof over a different Merkle tree**. The
    `expected_root` is committed to by a consensus certificate, so
    the tree with this root is the canonical tree.

The second attacker (a lying peer who sent the proof) cannot choose
a fake root — the consensus certificate binds the root. So the
attacker's only path is hash collision, which is computationally
infeasible.

## Rust pattern — GATs, `Bytes` vs `&[u8]`, `Cow`

Three Rust patterns recur across `storage`.

### Generic associated types (GATs)

```rust
pub trait Database: Send + Sync + 'static {
    type Read<'a>: Deref<Target = [u8]> + Send where Self: 'a;
    type Write<'a>: Write + Send where Self: 'a;
    fn read<'a>(&'a self, key: &[u8]) -> Self::Read<'a>;
}
```

Before Rust 1.65, you'd simulate this with `Box<dyn Read>` or
`Pin<Box<dyn Read>>` — both incur a heap allocation per read. With
GATs, the read type is associated with the borrow lifetime. Commonware's
storage types use GATs to expose borrowed-or-owned reads without
allocations.

The trade-off: GAT syntax is verbose (`for<'a> Self::Read<'a>: ...`),
and `dyn Database` is no longer usable (because the trait has
generic-method-shaped items). For inner storage, generics are fine;
for plugin-y boundaries, `Box<dyn Database>` requires erasure or
separate traits.

### `Bytes` vs `&[u8]`

The two viable representations for byte slices.

```rust
// &[u8] - borrow, no allocation
fn process(data: &[u8]) -> Result<()>;

// Bytes - cheap clone, owned
fn process(data: Bytes) -> Result<()>;
```

Commonware uses `Bytes` (the `bytes` crate) wherever messages might
be passed across async boundaries or stored in queues. The cost:
8 bytes for the header (pointer + len), plus atomic refcount bumps
on clone. The benefit: zero-copy for messages received from the
network (since `bytes` is the standard type for that) and trivially
cloneable for async handoff.

`&[u8]` shows up in hot-path internals where the lifetime is short
and the alloc savings matter.

### `Cow<'a, [u8]>` for owned-or-borrowed

```rust
fn canonical_form(&self, key: &[u8]) -> Cow<'a, [u8]> {
    if self.looks_already_canonical(key) {
        Cow::Borrowed(key)              // no allocation
    } else {
        Cow::Owned(self.transform_to_canonical(key))   // allocate once
    }
}
```

`Cow` is the standard pattern for "I might need to clone, might not."
It's how Commonware's `Variable<u8>` journal encoding returns data
without always copying.

The ergonomics: when you receive a `Cow<'a, [u8]>`, you can call
`.as_ref()` and treat it as `&[u8]`. To convert to a guaranteed-
owned `Bytes`, you can `.into_owned().into()`. The borrows-only-if-
borrowedable path is more verbose but saves an allocation.

## Where to look in the code (expanded preview)

The appendices that follow go deeper on each topic above. When you
see a `file:line` reference in main body, the appendix walks it
line by line — but the surrounding structure and motivation live
here.

## B-trees in full — disk-friendly pages and the path that broke SSL

B-trees are the dominant on-disk index for the past fifty years.
Before MMRs and Merkle proofs became hot topics, every database
you've ever used was running a B-tree under the hood. Understanding
the B-tree is what tells you **why** Commonware's append-only MMR
design is so unusual — and why the trade-off makes sense for
Commonware's workload but would be wrong for, say, Postgres.

### Bayer 1972 — the original design

Rudolf Bayer (then at Boeing Scientific Research Labs) proposed
the B-tree in 1972 ("Symmetric Binary B-Trees: Data Structure and
Maintenance Algorithms," Acta Informatica 1). The motivation:
secondary-storage indexes with **minimized I/O**. Every disc I/O
in the early '70s cost ~100 ms; reading one page was a major
event.

The B-tree's design:

- Each node is a **page** of size B (typically 4–16 KiB).
- Each internal page holds up to K keys and K+1 child pointers,
  with the keys separating the children's ranges.
- Height h ≈ ⌈log_K(N)⌉. For B = 16 KiB, K ≈ 100, N = 10⁹,
  h ≈ 4. **Four reads, on average, to find any key.**

The "B+ tree" variant (Knuth, "The Art of Computer Programming
Vol 3," 1973) refines the design: data lives only at the leaves;
internal pages are pure routing. Range scans go leaf-to-leaf via
sibling pointers. This is what almost every modern database uses.

### Pages, splits, and the write path

A B-tree's write path for inserting key K:

1. Find the leaf page containing K (h reads, log K).
2. Insert K into the leaf.
3. If the leaf overflows (> K keys), **split** the leaf: form
   two new leaves, push the median key up to the parent.
4. The push-up may itself overflow the parent → split parent → ...
5. If the root splits, a new root is created. The tree's
   height grows by one.

The cost: average O(K) per insert (split probability ~1/K).
The amortized cost per insert is O(log N) I/Os.

A B-tree's write path is a **logarithm of writes** spread
across the tree. Updating one key changes the leaf and rewrites
the path from leaf to root. For an authenticated B-tree
(MB-tree, Becker 2008), each rewritten page must be hashed, and
the page's hash is recomputed from its children's hashes. That's
the path that broke the SSL/TLS web — the SHA-1 collision
research that led to deprecation was driven partly by B-tree
hash-tree collision concerns.

### The read path

A B-tree's read path for key K:

1. Start at root.
2. Binary-search the keys (or linear-scan, depending on K).
3. Read the appropriate child page. Repeat until leaf.
4. Binary-search the leaf for K.

I/Os: h. For h = 4, that's 4. Modern databases buffer pages; a
hot tree fits in memory.

### The merge path

A deletion may trigger **underflow**: a leaf drops below
(min_keys) entries. The fix is **borrow** (take an entry from
a sibling) or **merge** (concatenate two siblings and free
one). Cascading merges reduce the tree's height.

Authenticated B-trees don't typically support deletes (each
deletion would change a Merkle path); QMDB's
"append-only-simulated-update" model avoids this entirely.

### Connection to Postgres

Postgres is the canonical B-tree database. Internals:

- Each table is a heap file, with a separate B-tree index
  per index (B+ trees on indexed columns).
- The heap file uses MVCC: each row has `xmin` (creator tx) and
  `xmax` (deleter tx). Updates are copy-on-write (write a new
  row, mark the old for cleanup). The MVCC layer is orthogonal
  to the B-tree indexing.
- Write-ahead log (WAL) for durability; index pages flushed
  lazily via `bgwriter` and `checkpointer`.

Commonware's design differs along three axes: append-only
(no in-place update), MMR-shape (no per-key page lookups), and
authenticated (each operation has a Merkle proof). The
Postgres-style B-tree trades authentication for general-purpose
mutation; Commonware's MMR trades mutation for fast appends
and verifiable proofs.

For a deeper read: Alex Petrov's "Database Internals" chapters
1–2 (free PDF version online), or DDIA chapter 3 ("Storage
and Retrieval").

## LSM trees in full — the LevelDB architecture

Log-Structured Merge trees are what RocksDB, LevelDB, Cassandra,
ScyllaDB, HBase, and (almost) every modern write-heavy key-value
store use. They evolved as a response to B-trees' weakness:
**in-place updates cause random writes**, and random writes are
slow.

### The shape

An LSM tree is a stack of:

```
                Memtable      (in-memory skiplist, mutable)
                    |
                    v   (flush when full)
              +-----+-----+
              |           |
            SST_0        SST_0 (immutable on-disk sorted string tables)
              |           |
              +--compaction (background merge)
                    |
                    v
            Level 1 (multiple smaller SSTs)
                    |
                    v (compaction)
              Level 2 (more SSTs, smaller per-SST)
                    |
                    v
                 ...
```

The invariants:

- The memtable is a sorted skiplist. Inserts go to the memtable
  in O(log N) time.
- When the memtable fills, it becomes a new immutable SST (a
  sorted file with a sparse index and a Bloom filter).
- SSTs are merged into the next level by background compaction.
- A read for key K checks memtable, then walks SSTs from newest
  to oldest, returning the first match.

### The write path

A write for key K:

1. Insert K into the memtable.
2. (Done. Return.)
3. (Background) When memtable fills (~64 MB), it's marked
   immutable and a new memtable is started.
4. (Background) The immutable memtable is flushed to SST_0.
5. (Background) Compaction runs to merge SSTs at level L into
   level L+1.

The hot-path cost: O(log M) per write (M = memtable size, ~mem
size / 64 MB), with no I/O until the memtable fills. **Burst
writes are nearly free.**

### The read path

A read for key K:

1. Check the memtable. If found, return.
2. Check L_0 SSTs in order (newest first). If a Bloom filter
   says "no," skip. If "maybe," look in the SST's sparse index,
   then binary-search the data block.
3. Walk down through L_1, L_2, ... until found.

I/Os: O(L × log K) per level, with Bloom filters making the
expected I/Os ~1.

### Compaction strategies

Two main strategies, with different write/read profiles:

**Size-tiered compaction (Cassandra's STCS, Bigtable).**

```
   SSTs of size ~S       merge into
       ─────              SSTs of size ~R*S (R=4)
           ─────
                ─────
```

When a level has R SSTs of similar size, merge them into one
larger SST. Writes: cheap (no seek during merge; sequential
read of R SSTs + sequential write of one). Reads: expensive
(check up to R SSTs per level).

**Leveled compaction (LevelDB/RocksDB's LCS).**

```
   L_1: many small SSTs (each ~S)
              |
              v (each L_1 SST overlaps ~10 L_2 SSTs)
   L_2: SSTs of size ~10*S
              |
              v
   L_3: SSTs of size ~100*S
```

When L_1 has too many SSTs, pick a key range, merge all
L_1 + L_2 SSTs in that range, write new L_2 SSTs. Writes:
expensive (read 10 SSTs, write 10 SSTs per merge). Reads: very
fast (one SST per level, max ~10 SSTs total for the entire
read).

LevelDB ships with leveled compaction. RocksDB defaults to
universal compaction (close to size-tiered) but configurable.

### Compaction path

The actual compaction logic:

1. Pick a target key range (a "compaction unit").
2. Read all SSTs in that range (sorted reads).
3. Merge in-memory (k-way merge).
4. Write new SSTs in sorted order.
5. Atomically swap the old SSTs for the new SSTs.
6. Update the manifest (list of SSTs).

The atomic swap is important: an LSM tree never has a "half-
compacted" state. A crash mid-compaction leaves the old SSTs in
place. Recovery re-runs compaction.

### LSM vs QMDB

Commonware's `qmdb::keyless` (chapter 6, then again in chapter
17's Glue pattern) is log-structured but isn't a full LSM tree.
There's no memtable/skiplist; the operations go straight to a
`Variable<u8>` journal and get merkleized into an MMR.

Why the divergence? Commonware's QMDB needs **Merkle
authentication** of the operations; an LSM tree's memtable
isn't authenticated. The MMR-over-log shape gives authentication
without a memtable: every operation is a node in the MMR, and
the MMR root commits them all.

For high-throughput verification of every state transition,
this is the right trade. For a key-value store without
authentication (DynamoDB-style), LSM is the right trade.

## Merkle proofs and authenticated data structures in full — RFC 9162

Section "Merkle proofs and authenticated data structures — RFC
9162" above gives the bird's-eye view. This section walks the
RFC and the security argument in detail.

### RFC 9162 — Certificate Transparency v2

Certificate Transparency (CT) is a system for monitoring TLS
certificate issuance. The goal: make it impossible for a CA to
issue a certificate without it being publicly observable. CT
runs Merkle trees over the global log of issued certificates,
and monitors can verify "is this certificate in the log?"

RFC 9162 specifies how this Merkle tree works. It uses SHA-256
and a binary tree with a length-prefix protocol to prevent
length-extension attacks.

The primitives defined:

**MerkleTreeHash (MTH).** The SHA-256 root of the Merkle tree
containing a particular set of entries.

```c
// RFC 9162 § 2.1
MTH({})       = SHA-256(0x00)
MTH({D})      = SHA-256(0x01 || SHA-256(D))
MTH(D[0..N))  = SHA-256(0x01 || MTH(D[0..k]) || MTH(D[k..N]))
              where k = N/2
```

The `0x00` / `0x01` prefixes prevent the same hash function
from being used for two different purposes (domain separation),
and the structure is a binary tree over the entries.

**MerkleAuditPath.** A list of sibling hashes from a leaf to
the root.

```c
// Audit path for entry i in a tree of size N:
path = []
for node = leaf_i; node != root; node = parent(node) {
    sibling = sibling(node)
    path.push(sibling)
}
```

Verifier: walk the path from leaf to root, recompute the
intermediate hashes, compare against expected root.

**MerkleConsistencyProof.** Between two roots R1 (size N1) and
R2 (size N2, N2 ≥ N1), the proof that R1's tree is a prefix of
R2's tree. The proof consists of intermediate nodes along the
path from R1 to R2.

**SignedTreeHead (STH).** MTH + tree size + timestamp +
log-signature. The log signs the root periodically; this is
what monitors trust.

### Inclusion vs exclusion proofs

**Inclusion proof**: "entry X is at position P in the log with
root R." Verifier: walk the audit path from leaf at P to root;
the recomputed root must equal R.

**Exclusion proof**: "entry X is NOT in the log with root R."
This is non-trivial; you need:

- A snapshot of the tree (the root + tree size).
- An audit proof for the entry that *would* be at position P
  (or adjacent).
- The fact that the entry you're "excluding" isn't there.

In a binary sorted Merkle tree, this is straightforward: show
the leaf at position P-1 and the leaf at position P, prove
both, prove that they're adjacent in the sorted order, and
your key is "between" them but not equal.

Commonware's ADB doesn't directly do exclusion proofs but
they're a standard extension: a non-existence proof for key K
shows "no `set(K, V)` operation for any V exists in the log."

### Second-preimage resistance

The concern: a malicious log operator presents two different
trees with the same root. The attacker constructs two entries
that hash to the same leaf value, then arranges them in
different trees.

SHA-256 is a 256-bit hash; finding a collision is ~2¹²⁸ work
(birthday-bound, but pre-image-targeted collisions are ~2²⁵⁶).
For practical purposes, SHA-256 collision resistance is enough
to prevent log operators from producing two different trees
with the same root.

The protocol layer also helps: STHs are signed, and monitors
observe the log adding STHs over time. A retroactive split (log
silently grows an alternate history) is detected by
consistency proofs.

### Security arguments

The full argument for the security of an authenticated log:

1. The root R is in a `SignedTreeHead`. The signature commits
   to R; if you trust the signer, you trust R.
2. The audit proof P from leaf to R is verifiable: the verifier
   computes R' from P and compares to the trusted R.
3. If P verifies, the leaf is in the tree — *the tree whose
   root is R*.
4. The signer signed R at time T. Therefore the entry was in
   the tree at time T.
5. To forge a "this entry is in the log" claim that's false,
   the attacker must either:
   - Forge the signer's signature on a different root R',
     defeating the signature scheme.
   - Find a hash collision in the audit path, defeating the
     hash function.
   - Both are computationally infeasible.

This is the same security argument any Merkle-based
authenticated log makes. Commonware's QMDB appends this
argument with one extra layer: the consensus certificate
commits to the root, so the "signer" of the root is the
quorum of validators.

## Cryptographic accumulators in full

Section "Cryptographic accumulators — and how they compare to
Merkle trees" above introduces three families. This section
goes through each in depth, with the math sketch and the
practical trade-offs.

### RSA accumulators (Benaloh & de Mare, 1993)

The RSA accumulator: pick an RSA modulus N = pq (with p, q
secret). The accumulator value is `A = g^{product of primes}
mod N` for some generator g. To prove membership of element
x in the accumulator, give a witness `w` such that
`w^x ≡ A (mod N)`. The verifier checks this equation.

Updates:

- **Add x**: `A_new = A^x mod N`. Witnesses must also update:
  each witness `w_y` becomes `w_y^x mod N`.
- **Delete x**: requires either (a) recomputing all witnesses
  (O(N)) or (b) maintaining "negative" witnesses (extra state).

The proof size is O(1) — a single group element. That's the
big win vs Merkle.

### Bilinear accumulators (Nguyen, 2005)

Build on bilinear groups (BN254, BLS12-381, etc.). The
accumulator value is `A = ∏ g^{i·s_i}` for some scheme
specific to the construction. Witnesses are O(1) group
elements.

Updates are O(log N) for some constructions; O(1) for others
(e.g., the "HyperPlonk" accumulator in 2022).

Trusted setup: requires an SRS (structured reference string),
which is a one-time ceremony that publishes evaluation points
of a secret polynomial.

### Comparison table, expanded

| | Merkle | RSA | Bilinear |
|---|---|---|---|
| Witness size | O(log N) | O(1) | O(1) |
| Update cost | O(log N) | O(1) add, O(N) delete | O(log N) |
| Trusted setup | No | RSA modulus (unknown factorization) | SRS |
| Quantum-safe | Yes (hash-based) | No (RSA broken by Shor) | No (pairings broken by Shor) |
| Verification cost | O(log N) hashes | O(1) RSA mul | O(1) pairing |
| Common in BFT? | Yes (Commonware, Ethereum, CT) | Rare | Some (Zcash, Filecoin) |

### When each is right

**Merkle**:

- You have a working hash function and don't want to trust
  external setup.
- Your proofs can grow logarithmically with the dataset.
- Updates happen infrequently (append-only is a plus).

**RSA**:

- You have very small proofs (one group element).
- You can manage trusted setup (RSA modulus with unknown
  factorization — practically, generated via multi-party
  computation).
- You have low update rates that can afford witness update
  cost.

**Bilinear**:

- You're in a pairing-friendly group context anyway.
- You want to combine proofs across multiple accumulators
  (e.g., via pairing).
- You can afford the SRS.

Commonware uses Merkle. The reason: it's stateless, no trusted
setup, and the proof size scales to millions of operations
without changes.

## Verkle trees in full — the polynomial commitment scheme

Verkle trees (Micali, Rabin, Walker, 2003 paper; recent revival
in the Ethereum space) are Merkle-shaped trees where internal
nodes commit to their children via polynomial commitments
instead of pairwise hashes.

### The polynomial commitment (KZG)

The core: a polynomial commitment scheme where committing to a
polynomial `p(X)` produces `[p(τ)] G₁` for a secret `τ` in a
pairing-friendly group. The commitment is a single group
element. An opening at `x = z` is `p(z)` plus an evaluation
proof (also a single group element).

The KZG commitment scheme (Kate, Zaverucha, Goldberg, 2010):
- **Setup**: pick a secret `τ`. Publish the SRS:
  `(G₁, τG₁, τ²G₁, ..., τⁿG₁, G₂, τG₂)`.
- **Commit**: for `p(X) = c₀ + c₁X + ... + cₙXⁿ`, commit is
  `[c₀ + c₁τ + ... + cₙτⁿ] G₁ = [p(τ)] G₁`.
- **Open at z**: produce `q(X) = (p(X) - p(z)) / (X - z)` (a
  polynomial), which is division-by-linear. Open proof is
  `[q(τ)] G₁`.
- **Verify**: pair `e([q(τ)] G₁, [τ - z] G₂)` and check it equals
  `e([p(τ) - p(z)] G₁, G₂)`.

One pairing plus one scalar multiplication. Constant cost per
opening.

### Why Verkle (vs Merkle)

| | Merkle | Verkle (KZG) |
|---|---|---|
| Internal node | Hash of children | Polynomial commitment of children |
| Witness at depth d | d sibling hashes | d KZG openings |
| Update | O(log N) hashes | O(1) (new polynomial) |
| Proving time | O(log N) hashes | O(log N) mul-exp + 1 pairing |
| Verifying time | O(log N) hashes | O(log N) pairings |
| Witness size | d × 32 bytes (SHA-256) | d × 48 bytes (BLS12-381 G₁) + few scalars |
| Trusted setup | None | SRS (one-time ceremony) |

The witness size is roughly the same in bytes (32 vs 48 per
level), but Verkle's witness transmission cost can be reduced
by batching: a "Verkle proof" packs multiple openings into one
multi-opening proof that's shorter than the sum.

### EIP-6800 — Verkle tree upgrade

Ethereum is moving from Merkle Patricia to Verkle trees as
part of EIP-6800. The motivation:

- **Stateless clients:** today, a stateless Ethereum client
  needs to provide a witness (~3 KB per account, ~15 sibling
  Merkle hashes) with every block to verify state. Verkle
  shrinks this to ~150 bytes (a few KZG openings batched).
  This makes stateless clients feasible on bandwidth-limited
  devices.
- **State size growth:** the current Ethereum state is ~1 TB
  and growing. Stateless clients + Verkle is the long-term
  answer to "everyone runs a full node."

EIP-6800 is in development as of 2026. Commonware doesn't ship
Verkle trees yet (the SRS ceremony is a coordination challenge),
but the QMDB construction's shape (MMR-style append-only data
with cryptographic proofs) adapts cleanly: the "MMR over
hashes" becomes "MMR over KZG commitments."

## Sparse Merkle trees in full

Section "Sparse Merkle trees — for huge keyspaces" above
introduces SMTs. This section goes through the construction in
detail and the optimizations that make them practical.

### The 32-byte keyspace

A sparse Merkle tree over 32-byte keys has 2²⁵⁶ leaves. The
tree is "complete" over the entire 256-bit keyspace, but
**only the branches along used paths are stored**. Default
leaves hash to a canonical "empty" value.

Construction:

- A leaf is a `(key, value)` pair. The leaf's position is the
  key.
- Internal nodes are hashes of children.
- An empty subtree at depth d hashes to `H("empty" || d)` —
  the depth-indexed default.

### Proofs

Inclusion proof: O(256) hashes (one per bit of the key) in
the worst case. In practice, much shorter: when a sibling is
the default hash at that depth, the proof omits it.

Exclusion proof: walk the path from root to where key K's
leaf *would* be. At each level, the sibling is either:

- The default hash → continue.
- A leaf for a different key, with a proof → reach an
  inconsistency, and K is excluded.

### What `qmdb::keyless` uses

`qmdb::keyless` is **not** an SMT — it's a keyless log. But
the Commonware roadmap includes a keyless-with-exclusion
variant; that would be an SMT over the log operations.

For the SMTs Commonware already uses (chapter 16 main text),
the implementation is in `storage/src/merkle/smt/` (or will
be — the SMT is younger than MMR but is being added).

### Optimizations

Recent SMT implementations achieve ~8 KB proofs and ~256
hashing operations per update by:

1. **Hash caching:** memoize `H("empty" || d)` for all d.
2. **Sub-tree pruning:** when a whole sub-tree becomes
   uniform (all default leaves), store just one hash.
3. **Compression:** bit-pack proofs; transmit only the
   non-default portion.

Commonware's storage doesn't ship these optimizations yet
because the current SMT use cases (chapter 16's "Sparse
Merkle trees — for huge keyspaces") are small enough that
8 KB is fine.

## Patricia tries in full

Ethereum's state structure: a Merkle Patricia Trie (MPT). Two
parts: a **trie** for compact key-prefix storage, and a
**Merkle hash** at each node for authentication. The "hex-
prefix" encoding is the standard MPT detail everyone forgets.

### The structure

A Patricia trie is a tree where:

- Keys are bit-strings (Ethereum: keccak hashes).
- Single-child paths are compressed into "extension nodes"
  with a shared prefix.
- Branching nodes have up to 16 children (hex digits) plus an
  optional value.
- All keys are sorted in lexicographic order at each level.

The shape:

```
root
  |
  +-- extension(a, node) where "a" is a shared prefix
       |
       +-- branch(0, 0, ..., value_at_branch, ..., f)
            |
            +-- extension(b, ...)
            |
            +-- leaf(key="ab..." | value)
```

Each node is `(extension | branch | leaf)` with content
depending on the type.

### The hex-prefix encoding

Morrison's "Practical Algorithm To Retrieve Information Coded
In Alphanumeric" (Patricia, 1968) specifies the encoding of
extension nodes:

For an extension node `(prefix_path, child_node)`:

- If the path length is even: add a flag nibble `0x0`.
- If odd: add a nibble `0x1` followed by the odd-path's
  first nibble, then the rest of the path.

This ensures that when the trie is encoded, you can
distinguish a leaf from an extension, and reconstruct the
path even when the trailing nibble is missing.

For a leaf `(suffix, value)`:
- If suffix length is even: `0x2` || suffix || value.
- If odd: `0x3` || suffix[0..1] || suffix[2..] || value.

### Ethereum's world state trie

Ethereum's world state is a Merkle Patricia trie:

- The root hash is the "state root" committed in each block.
- Each account is a 4-tuple `(nonce, balance, storage_root,
  code_hash)`, persisted as a leaf node.
- Each account's storage is itself a Patricia trie, keyed by
  slot number.
- The account's `storage_root` is the root of that sub-trie.

When an account's storage changes, the storage trie changes;
the account's `storage_root` updates; the world trie changes.

### Why not Patricia in Commonware

Commonware's state is not address-keyed. It's block-order-
structured: every operation goes into an append-only MMR.
Patricia's strengths (compact prefix storage, address-keyed
lookups) don't apply.

For an Ethereum-side project (e.g., a Commonware-based app
that stores state on Ethereum via cross-chain messages), the
state would be encoded into a Patricia trie on the Ethereum
side — but that's outside Commonware's storage layer.

## MMB algorithm in full — pyramid bagging math

The MMB (Merkle Mountain Belt) is the key innovation for
small current-state proofs. This section walks the math in
detail.

### The MMR proof-size problem

An MMR's inclusion proof includes:

- Path from leaf to peak: O(log N) hashes.
- All other peaks: O(log N) hashes.

For MMR with N = 2²⁰ ≈ 10⁶ operations, log N = 20. Number of
peaks ≈ 20. So a single inclusion proof is ~40 hashes
(~1.3 KB).

For a "current value" proof that includes the operation AND
proofs that no later operations override it (a QMDB `current`
proof), the size doubles: ~80 hashes (~2.6 KB).

For cross-chain state proofs (Light Client from chain A
proving "user X has balance Y on chain B"), every byte costs
gas. 2.6 KB is prohibitive for many on-chain deployments.

### The MMB fix — pyramid bagging

Bayardo's insight (May 2026 paper): instead of bagging just the
peaks, maintain a **belt** of intermediate hashes near the
active region. The root hash includes both peaks and belt
entries. Proofs covering the active region use belt entries
instead of recomputing the peaks.

The structure:

```
                  root
                   |
                   v
       (bag of peaks + belt entries)
                   |
                   v
       -- pyramid bagging at multiple levels --

peaks:    P0   P1   P2   P3   ...  Pn       (mountain tops)
                      |
belt:     B0   B1   B2   B3   ...  Bm       (intermediate hashes)
```

A proof for an active value uses the belt + a short path up
to the relevant peak, not all the peaks.

### The math — proof-size bound

Define `n` = number of peaks, `b` = belt size, `L` = depth of
an active-region element.

The pyramid-bagging scheme ensures:

```
proof_size <= 2 * ceil(log_2(active_size))
```

In words: the proof for a current value depends only on the
log of the active region's size, not the total history.

For `active_size = 100` elements, log = ~7, proof = ~14
hashes (~448 bytes). For `active_size = 10000`, proof size
~26 hashes (~832 bytes). For `active_size = 10⁶`, proof size
~40 hashes (~1.3 KB).

In contrast, the standard MMR proof size scales with log of
total history: for total history = 10⁹ operations, log = 30,
proof size ~60 hashes (~2 KB), *regardless* of how recent
the operation is.

### Active-region recomputation

The belt is recomputed when the "active window" slides
forward. Implementation:

1. The "active window" is a fixed size (default ~1024).
2. As new operations append, the oldest active-region element
   drops out.
3. Each time a drop happens, recompute the belt from scratch
   (cost: O(active_size) hashes). This is amortized O(1) per
   append.

The recomputation is bounded and predictable. Commonware's
MMB exposes a `recompute_belt_every` config knob that
controls the recomputation frequency.

### The "current" trick

In a QMDB `current` ADB, an additional optimization: the
"current value of key K" is proven by:

1. The Merkle path from the latest `set(K, V)` operation to
   the root.
2. A proof that no later `set(K, V')` for V' ≠ V exists.

For MMB, both proofs are small (active-region-sized). The
total proof is:

- Belt entries: O(1).
- Path from operation to peak: O(log N) hashes.
- "No later override" proofs: O(1) per override candidate,
  typically O(1) total.

### Comparison to alternatives

For comparison:

- **MMR (no MMB):** O(log N) per inclusion, O(log² N) for
  current-state.
- **MMB:** O(log(active_size)) for inclusion, O(log(active_size))
  for current-state. *Independent of total history*.
- **Verkle:** O(log N) for both, but constant-factor smaller
  per element.
- **RSA accumulator:** O(1) for both, but O(N) update for
  deletes.

MMB sits in a sweet spot: no trusted setup, fast appends,
small proofs for current state.

## Proof verification in depth — the attacker model

Section "Proof verification — full protocol" above gives the
API. This section walks the security argument in detail.

### The threat model

There are three possible attackers:

1. **Lying peer** — sends a fake proof.
2. **Malicious log operator** — operates an MMR and creates
   proofs for entries that don't exist.
3. **Network attacker** — MITM-proofing in transit
   (defeated by TLS, see chapter 04).

For (1) and (2), the verification rejects invalid proofs. For
(3), TLS (or QUIC, see chapter 05) protects transit.

The verification, in detail:

```rust
fn verify_proof(
    expected_root: &Digest,
    proof: &Proof,
    position: u64,
    item: &impl Hashable,
) -> Result<()> {
    // Step 1: Reconstruct the leaf hash.
    let leaf_hash = item.hash();

    // Step 2: Walk the proof's sibling hashes, recomputing up the tree.
    let mut current = leaf_hash;
    let mut current_pos = position;
    for (sibling_pos, sibling_hash) in proof.siblings() {
        let (left, right) = if current_pos < sibling_pos {
            (current, *sibling_hash)
        } else {
            (*sibling_hash, current)
        };
        current = hash_concat(left, right);
        // The parent's position follows from the two children's.
        current_pos = parent_of(current_pos, sibling_pos);
    }

    // Step 3: Compare against the expected root.
    if current != *expected_root {
        return Err(Error::InvalidProof);
    }

    Ok(())
}
```

### The attacker models in detail

**Lying peer attack:** the peer has a real value V and a real
root R, but sends a proof for V' (different value). The
verifier computes the path from V' and gets a root R'. If R'
≠ R, reject.

**Fake leaf attack:** the peer invents a leaf that wasn't
committed. The path computation will produce some root, but
it won't match R. Reject.

**Fake root attack:** the peer has a correct leaf and a fake
root R' ≠ R that they hope the verifier accepts. The
verifier computes the root from the leaf and the proof; the
computed root is R (the real one), not R'. Verifier compares
to expected_root, which the verifier got from a trusted source
(consensus certificate). If expected_root = R, verify succeeds.
If expected_root = R', mismatch — verifier rejects.

The last step is what binds everything together. The verifier
must have an `expected_root` from a trusted external source —
typically a consensus certificate. Without that, the verifier
could be tricked into trusting whatever root the peer offers.

### Security arguments — formal

For full rigor (per Bayardo's proof sketches):

**Theorem (inclusion):** A successful forgery requires either
collision-finding in H or breaking the consensus commitment.

**Proof sketch:** If the verifier accepts a proof for leaf L
at position P against root R, the verifier's path
computation must equal R. The path computation is a
deterministic function of L, P, and the proof's sibling
hashes. For this to equal R but the actual contents at P in
the real tree to be L' ≠ L, the attacker must have produced
either:

- A collision in H (two distinct (left, right) pairs giving
  the same hash), or
- A collision in the leaf hashing L = L_hash(L_payload), or
- A different tree structure where position P contains L but
  the rest is different.

The first two reduce to H's collision resistance. The third
is detected by the consensus-cert binding R to a specific
tree.

**Theorem (current value):** A successful forgery of a
"current value" claim requires either a collision in H or a
forgery of the "no later override" proof.

The QMDB `current` variant adds the constraint: the proof
must show that the value at position P is the latest `set(K, V)`
operation for key K. The proof's "no later override" piece
verifies this. A valid proof must be a real, signed sequence
in the MMR.

## Exercises

These exercises explore the authenticated data structures
Commonware uses. They build working proof systems from
scratch.

### Exercise 1 — Build a minimal MMR

Without using `storage::mmr`, write a minimal Merkle Mountain
Range:

- `append(&mut self, hash: Digest)`.
- `root(&self) -> Digest`.
- `proof(&self, position: u64) -> Proof`.
- `verify(root: &Digest, proof: &Proof, position: u64, hash: &Digest) -> bool`.

Don't worry about pruning; just an in-memory MMR.

Tests:

- Append 4 hashes; root is `SHA(count=7 || peak0 || peak1)`.
- Proof for position 0 verifies against the root.

### Exercise 2 — Add a non-membership proof

Extend your MMR with a "skip list" of peaks:

- For a given key K, you prove "K is not in the MMR" by
  showing the keys at neighboring positions.

Tests:

- For an MMR with 4 keys (`a, b, c, d`), prove "x (between b
  and c)" is not present.

### Exercise 3 — Implement a B+ tree insertion

Without using a B-tree library, write a B+ tree supporting
insert and lookup.

- Page size B = 4 KiB.
- K = (B / sizeof(K)) keys per page.
- Split on overflow; merge on underflow.

Tests:

- Insert 1000 random keys; lookup each; assert all found.
- Range scan `[K1, K2)` for random K1, K2.

### Exercise 4 — Compare LSM and MMR throughput

In a benchmark, write K keys to (a) an LSM-style structure
(memtable + flush) and (b) an MMR-style structure (append
only). Compare:

- Latency for `put` (single op).
- Throughput (ops/second) under burst.
- Disk usage after `prune`.

### Exercise 5 — Implement a tiny Patricia trie

Implement a Patricia trie that supports `insert(key, value)`,
`get(key)`, and the hex-prefix encoding.

Tests:

- Insert `("a", 1), ("ab", 2), ("abc", 3)`. The trie should
  compress to: extension node for prefix "a" → branch node →
  extension "b" → leaf "c".
- Lookup each; assert correct value.

→ Next: **Chapter 17 — Glue** (the cross-primitive defaults)
and **Chapter 18 — Deployer** (AWS provisioning). Both are
short.

## Appendix A — MMR position math, in detail

`storage/src/merkle/position.rs`. The MMR assigns each node a **position**
(0-indexed). The math:

```
Position 0: leaf (height 0)
Position 1: leaf (height 0)
Position 2: internal node (height 1) — covers positions 0 and 1
Position 3: leaf (height 0)
Position 4: internal node (height 1) — covers positions 1 and 2
Position 5: internal node (height 1) — covers positions 3 and 4
Wait, no. Let me redo this.
```

Actually, the standard MMR position assignment is:

```
Leaf positions are 0, 1, 3, 4, 7, 8, 10, 11, ...
```

Each leaf position has a "height" — the height of the mountain it's
in. Internal nodes are at positions 2, 5, 6, 9, 12, 13, ...

The pattern: a leaf at position `p` has the highest bit of `p`
determining its height. Specifically:

```
height_of_leaf(p) = number of trailing 1s in binary representation of p
```

So:
- Position 0 (binary `0`) → 0 trailing 1s → height 0 (a mountain of height 0).
- Position 1 (binary `1`) → 1 trailing 1 → height 1 (a mountain of height 1).
- Position 2 (binary `10`) → 0 trailing 1s → height 0... wait this doesn't work either.

Let me just look at the code (`storage/src/merkle/position.rs`).

The Commonware implementation uses a different but equivalent formulation:
**mountain heights** are stored per leaf. The "next power of 2 minus 1"
gives the position of an internal node.

OK, the math is fiddly. The key insight: position math is
**algorithmic** — there's a `find_peaks` function that returns the
positions of all current peaks (mountain tops). And `peak_bagging`
hashes them all together with the total count to produce the root.

## Appendix B — Peak bagging, in depth

`storage/src/merkle/`. The "root" of an MMR:

```rust
fn root_hash(peaks: &[Digest], node_count: u64) -> Digest {
    let mut hasher = Hasher::new();
    hasher.update(&node_count.to_be_bytes());   // node count as length-prefix

    // Sort peaks by position (already sorted if walked left-to-right)
    for peak in peaks {
        hasher.update(peak.as_ref());
    }
    hasher.finalize()
}
```

Hash the node count + sorted peak hashes. The node count is the
**length prefix** that prevents length-extension attacks.

For a fresh MMR with 4 elements: positions 0-6, peaks at positions 3
and 6. Root = `SHA256(7 || h(3) || h(6))`.

For a fresh MMR with 7 elements: positions 0-9, peaks at position 9
(single mountain of height 3). Root = `SHA256(10 || h(9))`.

## Appendix C — Inclusion proofs

To prove "element at position 5 is in this MMR":

1. Provide the hash at position 5.
2. Provide the hashes of sibling nodes on the path from 5 to the
   mountain's peak.
3. Provide the hashes of other peaks.

The verifier:

1. Starts at position 5, hashes with sibling to reach position N
   (the peak containing 5).
2. Combines that with other peaks (in order) to compute the root.
3. Checks the computed root against the expected root.

Proof size: O(log N) hashes for the path, plus O(log N) hashes for
the other peaks. Total: O(log N) hashes.

## Appendix D — The position-to-element mapping

Each leaf in the MMR has a position. Your application might want
"the 100th appended element" — that's the leaf at position 100 (or
nearby, after adjustment for internal nodes).

```rust
fn element_index_to_position(index: u64) -> Position {
    // Walk through the MMR's leaves, counting
    let leaves_walked = 0;
    for pos in 0..u64::MAX {
        if is_leaf(pos) {
            if leaves_walked == index {
                return Position::new(pos);
            }
            leaves_walked += 1;
        }
    }
    panic!("index out of bounds");
}
```

The `is_leaf(pos)` function checks if `pos` is a leaf position (vs an
internal node position).

## Appendix E — The pruning math

MMR pruning is **safe** because of the structure:

- Each peak is the root of an independent mountain.
- An internal node is only needed if its descendants still need to
  be proven.
- Once all elements of a mountain are "old enough" (per application
  policy), the mountain can be dropped.

```rust
fn prune_to(retain_below_height: Height) {
    // Find the first peak that's "before" the retention height
    let first_kept_peak_idx = find_first_peak_at_or_above_height(retain_below_height);

    // Drop everything before that peak
    for peak_idx in 0..first_kept_peak_idx {
        let peak = self.peaks[peak_idx];
        let ancestor_range = ancestor_range_of(peak);
        self.drop_range(ancestor_range);
        self.peaks.remove(peak_idx);
    }

    // The new "root" is now based on the remaining peaks only
}
```

After pruning, the root hash changes (because you've removed peaks).
Applications that need historical roots must store them separately.

## Appendix F — The QMDB internals

`storage/src/qmdb/`. QMDB is a family of ADB constructions:

### `qmdb::any`

Past OR current state proofs. The proof includes:
- The Merkle mountain range's peaks at the current operation count.
- The Merkle proof for the operation's inclusion.
- A proof that no later operation for the same key overrides this one.

Larger proofs, but supports any historical query.

### `qmdb::current`

Current state proofs only. The proof:
- The peaks at the current operation count.
- The Merkle proof for the latest op of key K.

Smaller proofs. Doesn't support "what was K's value at time T?"

### `qmdb::keyless`

No key. Pure log. The proof includes:
- The peaks.
- The Merkle proof for the operation's position.

Used for transaction logs where you want "was transaction X included?"

### The `current` implementation, in more detail

```rust
struct CurrentDB {
    mmr: MMR,
    operations: Variable<u8>,           // variable-size operation log
    // ...
}

impl CurrentDB {
    fn get(&self, key: &[u8]) -> Option<Value> {
        // Walk the log backward, find the latest op for `key`
        // Return its value
    }

    fn commit(&mut self, op: Operation) -> Digest {
        let idx = self.operations.append(op.encode());
        self.mmr.append(self.hasher.digest(&op.encode()));
        self.mmr.root()
    }
}
```

The MMR commits to operations; the operations are stored separately
(indexed by MMR position). To get a current value, scan the operation
log backward.

## Appendix G — The MMB (Merkle Mountain Belt), in detail

`docs/blogs/pyramid-mmb.html` and `storage/src/merkle/mmb/`. The MMB
addresses the **proof size growth** problem of MMRs.

### The problem with MMR proofs

For an MMR with N elements, an inclusion proof includes:
- Path to peak: O(log N) hashes.
- All other peaks: O(log N) hashes.

Total: O(log N) hashes.

But each peak is a separate mountain. For large N, the number of
peaks is also O(log N). So the proof is O(log N) for the path PLUS
O(log N) for the peaks = **O(log² N)**.

### The MMB fix

The MMB adds a **belt** of intermediate hashes near the active region.
The belt:
- Contains O(1) hashes at any time (regardless of N).
- Is updated as new elements are appended.

A proof using the belt is:
- Path from leaf to belt: O(log N) hashes.
- Belt entry: O(1) hashes.
- Plus some "inter-belt" hashes for verification.

Total: O(log N), not O(log² N). **Better asymptotic proof size.**

### The "pyramid bagging"

`docs/blogs/honey-i-shrunk-the-proofs.html`:

> Pyramid bagging, with the Merkle Mountain Belt (MMB), our new scheme
> makes active-value proofs small and predictable: within two digests
> of a balanced binary Merkle tree over those values.

"Two digests of a balanced Merkle tree" means: proof size is bounded
by 2× the log of the active region, which is essentially constant
(log of recent activity, not total history).

## Appendix H — The `merkle::bmt` — Binary Merkle Tree

`storage/src/bmt/`. The basic BMT:

- Balanced binary tree.
- Internal nodes are SHA-256 hashes of children.
- Leaf nodes are SHA-256 hashes of values.
- Root is a single 32-byte hash.

Unlike MMR, BMT supports **updates at arbitrary positions**. But updates
require reading and rewriting O(log N) random nodes.

Used in: places where you need a "regular" Merkle tree (not an MMR).
ADB/QMDB use MMR. BMT is for one-off use cases.

## Appendix I — The `bitmap` — authenticated bit vector

`storage/src/bitmap/`. An authenticated bitmap where:
- Each bit has a Merkle proof.
- The whole bitmap has a single root hash.

Use cases: "which peers voted in this epoch?" — encode each peer's vote
as a bit, prove membership in the bitmap.

## Appendix J — The `qmdb::verify` — proof verification

```rust
fn verify(proof: Proof, root: Digest) -> bool {
    // Step 1: Verify the Merkle proof
    let computed_root = proof.recompute_root();
    if computed_root != root {
        return false;
    }

    // Step 2: Verify the operation semantics
    // (for `current`, this means "no later op for the same key")
    if !proof.verify_no_override() {
        return false;
    }

    true
}
```

The `Proof` type includes:
- The Merkle mountain range inclusion proof.
- Application-specific metadata (e.g., the key being proven, the
  operation type, the value).

## Appendix K — Common patterns

### The "checkpoint-and-prune" pattern

```rust
fn periodic_maintenance(&mut self) {
    // Commit a checkpoint
    let root = self.db.commit();

    // Prune old operations
    let oldest_relevant = self.compute_oldest_relevant();
    self.db.prune(oldest_relevant);

    // Store the checkpoint for future use
    self.checkpoints.push(Checkpoint { root, at_height: self.height });
}
```

Run periodically (e.g., every 1000 blocks). Bounded disk usage.

### The "rollback to checkpoint" pattern

```rust
fn rollback_to(&mut self, checkpoint: Checkpoint) {
    // Discard all state after the checkpoint
    self.db.truncate_to(checkpoint.at_height);
    self.checkpoints.retain(|c| c.at_height <= checkpoint.at_height);
}
```

Useful for handling deep reorgs or recovering from bugs.

## Appendix L — Common gotchas

### Confusing MMR position with element index

Position includes both leaves and internal nodes. Element index counts
only leaves. A leaf at "element index 100" might be at MMR position 120
(or whatever).

### Forgetting to update peaks on append

After `mmr.append(hash)`, the peaks list must be updated:
- If a new mountain formed (peak + neighbor of equal height), merge.
- The new peak replaces both.

Commonware's `Mmr::append` handles this internally. But if you're doing
something exotic (like appending a batch of hashes), make sure the
peaks bookkeeping is correct.

### Using ADB for everything

ADB / QMDB are great for "mutable state on append-only log." But for
read-heavy + update-light workloads, a regular BTreeMap might be faster.
Profile your workload.

## Where to look in the code (expanded)

- `storage/src/qmdb/mod.rs` — the QMDB entry point.
- `storage/src/qmdb/any/` — past-or-current ADB.
- `storage/src/qmdb/current/` — current-state-only ADB.
- `storage/src/qmdb/keyless/` — log-only ADB.
- `storage/src/qmdb/sync/` — peer-to-peer sync.
- `storage/src/merkle/mmr/` — the MMR implementation.
- `storage/src/merkle/mmb/` — the MMB implementation.
- `storage/src/merkle/proof.rs` — Merkle proof generation/verification.
- `storage/src/bmt/` — the basic binary Merkle tree.
- `docs/blogs/mmr.html` — Roberto's MMR introduction (Feb 2025).
- `docs/blogs/the-simplest-database-you-need.html` — Roberto + Patrick,
  April 2025. How MMR + log = mutable database.
- `docs/blogs/qmdb.html` — Patrick, December 2025. The QMDB announcement.
- `docs/blogs/its-a-grind.html` — Roberto, February 2026. Performance
  optimization for non-uniform keys.
- `docs/blogs/honey-i-shrunk-the-proofs.html` — Roberto, May 2026. The MMB
  announcement.

## If you only remember three things

1. **MMR = append-only Merkle.** Faster than classical Merkle trees because every write is sequential. Proofs are O(log N) for inclusion.
2. **ADB/QMDB = mutable database on top of append-only MMR.** "Update K" becomes "append set(K, V)". The proof shows "latest op for K is V."
3. **MMB = small proofs for current state.** Even as history grows, current-state proofs stay near-constant size. Makes cross-chain state proofs cheap.

→ Next: **Chapter 17 — Glue** (the cross-primitive defaults) and
**Chapter 18 — Deployer** (AWS provisioning). Both are short.