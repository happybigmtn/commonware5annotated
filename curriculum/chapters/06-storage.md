# Chapter 06 — Storage: bytes on disk that survive a crash

> How Commonware persists data durably and recovers cleanly.

## The problem

Consensus messages are temporary — once the protocol decides on a block, the
in-flight messages are no longer needed. But the **block itself** is forever.
The signed certificates. The committed state. The application database.

Three things must be true:

1. **Durable.** After a power pull, the data is still there. The OS says "fsync
   succeeded" → the data will survive a crash. (And many systems lie about
   this, so we test it.)
2. **Recoverable.** When the process restarts, it can rebuild its in-memory
   state from disk. This means **replay** must be fast and **partial writes**
   must be handled correctly.
3. **Prunable.** Consensus runs forever; the disk doesn't. Old data must be
   safely deletable without breaking recovery.

Commonware splits this into two layers:

1. **`Storage` (abstract)** — opens blobs in named partitions.
2. **`Journal`** (and other higher-level types) — append-only logs, KV stores,
   authenticated databases built on top.

## The two-layer model

From `runtime/src/lib.rs:608-779` (chapter 02) and `storage/src/lib.rs`:

```
Storage        ← abstract trait (chapter 02)
  │
  ├─→ open_versioned(partition, name, version_range) → (Blob, size, version)
  ├─→ remove(partition, name) → ()
  └─→ scan(partition) → Vec<name>
       │
       ▼
Blob            ← a fixed-size byte range, opened by name
  │
  ├─→ read_at(offset, len) → bytes
  ├─→ write_at(offset, bytes) → ()
  ├─→ resize(len) → ()
  ├─→ sync() → ()                  ← durability barrier
  └─→ start_sync() → Handle<()>    ← non-blocking sync
       │
       ▼
Journal         ← append-only log on top of Blobs
  │
  ├─→ append(section, item) → offset
  ├─→ get(section, offset) → item
  ├─→ replay(buffer_size) → Stream<Item>
  ├─→ sync_all() → ()
  └─→ prune(section) → ()           ← delete all blobs < section
```

`Storage` is the abstract interface. It can be backed by:

- **Local filesystem** (tokio runtime)
- **In-memory HashMap** (deterministic runtime for tests)
- **Cloud storage** (S3, R2, GCS — via the `deployer` AWS support)
- **Faulty wrapper** that randomly fails reads/writes (for testing)

`Blob` is the lowest unit. One Blob = one (partition, name) tuple. A Blob is
just a byte range; you can read/write arbitrary offsets within it.

`Journal` is the **append-only log** that consensus and other primitives
actually use. It's the most important storage type.

## The journal — the workhorse

Open `storage/src/journal/mod.rs:1-15`:

> Journals provide append-only logging for persisting arbitrary data with fast
> replay, historical pruning, and rudimentary support for fetching individual
> items.

Two flavors:

| Flavor | Item size | Format |
|---|---|---|
| `Journal<Fixed>` (segmented/fixed.rs) | All items same size | `item_0 \| item_1 \| ... \| item_n` |
| `Journal<Variable>` (segmented/variable.rs) | Variable | `[length varint][item_0][length varint][item_1]...` |

Both organize data into **sections**. A section is a `u64` you assign. Each
section lives in its own Blob. This lets you:

- **Append** new items by writing to the current section.
- **Prune** old data by deleting Blobs for sections below a threshold.
- **Bound memory** by capping how many sections are kept open simultaneously.

### The on-disk format (variable)

From `variable.rs:9-20`:

```text
+---+---+---+---+---+---+---+---+
|       0 ~ 4       |    ...    |
+---+---+---+---+---+---+---+---+
| Size (varint u32) |   Data    |
+---+---+---+---+---+---+---+---+
```

Each item is `[varint length][data bytes]`. The length is a u32 varint
(capped at chapter 03's `usize ≤ u32` rule). Reading is "read length, read
that many bytes." Writing is "write length, write data."

### The on-disk format (fixed)

From `fixed.rs:4-12`:

```text
+--------+--------+--------+----------+
| item_0 | item_1 |   ...  | item_n-1 |
+--------+--------+--------+----------+
```

Items are packed back-to-back. Item `i` lives at offset `i * SIZE`. Trivial
to compute, trivial to read, trivial to write. No length prefix needed
because you know the size.

### Sync — when bytes hit the disk

From `variable.rs:31-32`:

> Data written to `Journal` may not be immediately persisted to `Storage`. It
> is up to the caller to determine when to force pending data to be written
> to `Storage` using the `sync` method. When calling `close`, all pending
> data is automatically synced.

This is critical. The OS buffers writes. Calling `write_at` doesn't mean the
bytes are on disk; they're in a page cache. Only `sync()` (which calls
`fsync`) makes them crash-durable.

When does the consensus engine sync? Look at chapter 01's Simplex spec,
line 313:

> Before sending a message, the `Journal` sync is invoked to prevent
> inadvertent Byzantine behavior on restart (especially in the case of
> unclean shutdown).

So Simplex syncs **before broadcasting a message**. That way, if the process
crashes 1 second later, the messages that other validators saw are also on
this node's disk. No "I voted for block B but the log doesn't show it" weirdness.

### Repair on crash — like SQLite and RocksDB

From `variable.rs:140-145`:

> Like [sqlite](https://github.com/sqlite/sqlite/blob/...) and [rocksdb](...),
> the first invalid data read will be considered the new end of the journal
> (and the underlying Blob will be truncated to the last valid item). Repair
> occurs during replay (not init) because any blob could have trailing bytes.

This is the **single most important property** of the journal. A crash mid-write
might leave trailing garbage in a Blob. The journal doesn't panic on
encountering it; it just **truncates** to the last valid item.

The pattern from `AGENTS.md` testing:

```rust
// Manually corrupt data
let (blob, size) = context.open(&cfg.partition, &1u64.to_be_bytes()).await.unwrap();
blob.write_at(size - 4, vec![0xFF; 4]).await.unwrap();
blob.sync().await.unwrap();

// Re-initialize and verify recovery
let journal = Journal::init(context, cfg).await.unwrap();
let stream = journal.replay(buffer_size).await.unwrap();
// Verify corrupted items are skipped/truncated
```

This is the property that makes "unclean shutdown" a non-event. You might
lose the last half-written item. You'll never see a corrupt journal that
refuses to start.

### Replay — rebuilding state from disk

When the process starts, you call `journal.replay(buffer_size)`. You get a
stream of all items in (section, offset) order. Your application walks the
stream and rebuilds its in-memory state.

The example from chapter 01's consensus config:

```rust
let cfg = simplex::Config {
    partition: "log".into(),     // journal partition name
    ...
    write_buffer: NZUsize!(1024 * 1024),
    replay_buffer: NZUsize!(1024 * 1024),
    ...
};
```

The `replay_buffer` and `write_buffer` sizes control how much data the
journal stages in memory before flushing. Tune for your workload.

### Page cache — cutting down on disk I/O

Look at the example from `variable.rs:62-65`:

```rust
let page_cache = CacheRef::from_pooler(&context, NZU16!(1024), NZUsize!(10));
```

`CacheRef` is a shared page cache. Pages are read into it; subsequent reads
of the same page hit memory. Across all journals in your app, you typically
share one cache (or a few, partitioned by access pattern).

This is why replay is fast even on slow disks — the hot pages stay cached.

## The other storage primitives

Open `storage/src/lib.rs:13-36`. The storage crate ships more than just
journals:

| Module | What it is | Stability |
|---|---|---|
| `journal` | Append-only log (covered above) | BETA |
| `archive` | Fixed-size value archive | BETA |
| `freezer` | Large immutable data archive | BETA |
| `index` | ULTRA-coded compressed index | BETA |
| `ordinal` | Ordinal key-value store | BETA |
| `rmap` | Rank map | BETA |
| `metadata` | Atomic metadata writes | BETA |
| `qmdb` | Quick Merkle Database (chapter 16) | ALPHA |
| `mmr` / `merkle::mmr` | Merkle Mountain Range (chapter 16) | ALPHA |
| `merkle::mmb` | Merkle Mountain Belt (chapter 16) | ALPHA |
| `bitmap` | Authenticated bitmap | ALPHA |
| `cache` | Caching helpers | ALPHA |
| `bmt` | Binary Merkle tree | ALPHA |
| `queue` | FIFO queue | ALPHA |

We'll dig into the **authenticated** ones (qmdb, mmr, mmb) in chapter 16.

## A worked example: Simplex's write-ahead log

The Simplex consensus engine uses a journal as a **write-ahead log (WAL)** for
crash recovery. The flow:

1. **Startup**: open the journal. `replay()` yields every message this node
   ever received or sent. The engine rebuilds its in-memory state from those.

2. **Receive message**: append the message to the journal (but don't sync yet).

3. **Process message**: update in-memory state.

4. **Send message**: append the outgoing message to the journal. **Sync the
   journal** (this is the durability barrier from chapter 01). Then broadcast.

5. **Crash**: if we crash anywhere in steps 2-3, on restart we'll replay the
   received-but-not-processed message and finish it. If we crash in step 4
   after sync but before broadcast, we'll re-broadcast on restart (the WAL
   records the intent). Either way, **no Byzantine behavior** is possible
   because the WAL is the source of truth.

This is the canonical WAL pattern, but applied to a consensus protocol where
"Byzantine behavior" means losing track of votes you sent.

## Putting it together: how a node recovers from crash

```
Boot
  │
  ▼
deterministic::Runner starts (in tests) or tokio::Runner starts (in prod)
  │
  ▼
Create runtime context, open Storage, scan partition
  │
  ▼
Initialize Simplex engine with journal at partition "log"
  │
  ▼
Engine.start(): journal.replay() yields all past messages
  │
  ▼
Rebuild in-memory state from replayed messages
  │
  ▼
Resume participation in consensus
```

If you `kill -9` the process mid-consensus, restart, the engine sees the
half-written message in the journal, truncates past it, rebuilds its state
to where it was, and rejoins the network. Other nodes see this as a brief
hiccup. No consensus fork. No double-vote.

## Where to look in the code

- `runtime/src/lib.rs:608-779` — `Storage` and `Blob` traits (chapter 02).
- `storage/src/journal/mod.rs:1-15` — the journal concept.
- `storage/src/journal/segmented/variable.rs:9-20` — variable-size format.
- `storage/src/journal/segmented/fixed.rs:4-12` — fixed-size format.
- `storage/src/journal/segmented/variable.rs:140-145` — the SQLite/RocksDB-style
  repair pattern.
- `consensus/src/simplex/engine.rs` — how Simplex uses the journal as WAL.
- `examples/log/src/main.rs:148-150, 213-220` — how to configure a journal in
  a real app.

## DDIA Chapter 5 — replication, leaders, followers, quorum

DDIA's Chapter 5 ("Replication") is the canonical reference for
distributed storage. Commonware's storage layer assumes you know these
concepts; let me lay them out.

### Why replicate

A single node can fail. To survive failures, you store the same data
on multiple nodes. The question: how do those nodes stay in sync?

Three main approaches:

1. **Single-leader replication.** One node (the leader) accepts writes.
   Followers replicate from the leader. If the leader fails, a follower
   is promoted.

2. **Multi-leader replication.** Multiple nodes accept writes. Each
   node replicates to the others. Used for multi-datacenter setups
   where one datacenter "owns" each write.

3. **Leaderless replication.** Any node accepts a write. Reads go to
   multiple nodes and reconcile (e.g., Dynamo-style quorum reads).

Commonware's storage layer is **single-node** by design: each validator
has its own journal, and replication is the job of the consensus
protocol (which finalizes blocks across all validators). So the storage
layer doesn't implement leader/follower logic; the **consensus protocol
is the replication layer**.

But the storage layer must be **durable on a single node**, which means
it must handle single-node failures (crashes, power loss) correctly.
That's where DDIA Chapter 5's concepts apply.

### Sync vs async replication

Within single-leader replication, there's a sub-question: when does a
write become "durable on the followers"?

- **Synchronous replication.** Leader waits for all followers to
  acknowledge before returning success to the client. Slow but safe.
- **Asynchronous replication.** Leader returns success as soon as it's
  written locally. Followers catch up later. Fast but risky.

Commonware's journal sync (`sync()` calling `fsync`) is the equivalent
of "synchronous replication to disk." When `sync()` returns, the data
is on disk. Until then, it's only in the page cache.

For consensus, the WAL pattern is: write to the journal, **sync**
the journal, then broadcast. This is synchronous replication to disk
plus the broadcast is asynchronous replication to peers. If the
broadcast fails, the local state is still correct (just stale).

### Quorum writes

In leaderless or multi-leader systems, a **quorum** is a write that
succeeds on a majority of nodes (typically `w > n/2` where `n` is the
total). Reading from `r` nodes where `r + w > n` guarantees that any
read sees the latest write.

Commonware's consensus protocol implements a form of quorum writes via
the certificate scheme (chapter 04): `2f + 1` votes on a block is the
"write quorum," and `f + 1` is the minimum to overlap with any
previous quorum.

Storage is a local concern; replication is a consensus concern. The
separation keeps each layer's complexity manageable.

### Replication and Commonware's design

Why isn't storage replicated within the `storage` crate? Because:

1. **The consensus protocol IS the replication.** Each validator stores
   its own copy of finalized blocks. The protocol ensures all validators
   agree on what those blocks are.

2. **Local failures are common.** Single-node storage must handle crash
   recovery, partial writes, and corruption. Replicated storage has
   the same problems plus inter-node consistency.

3. **Different fault models need different storage.** Some apps want
   local-only (single-validator setups), some want consensus-replicated
   (the typical BFT setup), some want geographic replication. The
   storage crate handles the local case; higher-level primitives handle
   the others.

## LSM trees vs B-trees — the on-disk data structure question

Within a single node, the **on-disk data structure** matters. Commonware
uses LSM-tree-inspired designs (log-structured journals) rather than
B-trees. Let me explain why.

### B-trees

B-trees are the standard on-disk data structure for databases
(PostgreSQL, MySQL/InnoDB, SQLite). Properties:

- **In-place updates.** Modifying a key updates the page in place.
- **Sorted.** Keys are stored in sorted order; range scans are fast.
- **Read-optimized.** One seek to find the page; one read.
- **Write-amplification.** A small update may modify multiple pages
  (the leaf, internal nodes, sibling pages during splits).

### LSM trees

LSM (Log-Structured Merge) trees are the alternative (LevelDB, RocksDB,
Cassandra). Properties:

- **Append-only writes.** All writes go to an in-memory memtable, then
  are flushed as a sorted run to disk. Old runs are merged in the
  background ("compaction").
- **Sorted within runs.** Each run is sorted; range scans work within
  a run.
- **Write-optimized.** Writes are sequential appends to memtable +
  sequential flush to disk. No in-place updates.
- **Read-amplification.** A read checks the memtable, then each sorted
  run from newest to oldest, until it finds the key.

### Why Commonware uses LSM-like designs

The journal (`storage/src/journal/`) is essentially an LSM-tree without
the explicit compaction step:

- **Append-only writes.** `journal.append` writes to a buffer, then
  flushes to disk.
- **Sequential I/O.** All writes are sequential appends. No seeks.
- **Compaction as pruning.** When you prune a section, you delete old
  blobs — equivalent to LSM compaction for the "delete" operation.

The reason: BFT consensus is **write-heavy**. Every notarized block
generates a journal write. With a B-tree, every write might require
random I/O (seek + update). With an LSM, every write is sequential.

### Write amplification

A concept from SSD/HDD literature: **write amplification** is the
ratio of actual disk writes to logical writes. For B-trees, it can be
10-50x (small writes cause many page rewrites). For LSM, it's typically
1-4x (no in-place rewrites, but compaction rewrites old data).

For SSDs, write amplification matters because SSDs have a finite write
endurance. For HDDs, it matters less (sequential writes are fast either
way). For both, lower amplification is better.

### Compaction

In an LSM, **compaction** is the background process that merges sorted
runs. Without compaction, reads would scan every run forever, getting
slower over time. Compaction trades write I/O for read I/O.

Commonware's journal doesn't have explicit compaction, but it has
**pruning** (which deletes old blobs entirely). This is "compaction
to zero" — the most aggressive form.

The trade-off: pruning loses historical data (you can't replay the
chain beyond the pruned section). For BFT consensus, that's fine (the
finalized state is what matters). For append-only logs that need to
preserve history, it's not.

## Write-ahead logging — ARIES and the three passes

The journal's repair mechanism (`storage/src/journal/segmented/variable.rs:140-145`)
is essentially a mini-ARIES algorithm. ARIES (Algorithm for Recovery
and Isolation Exploiting Semantics) is the canonical WAL recovery
algorithm from IBM's 1992 paper. Let me walk through it.

### The three passes

When a database crashes with a WAL, on restart it runs three passes:

1. **Analysis pass.** Walk the WAL forward from the last checkpoint.
   Identify which transactions committed, which aborted, which were
   in-progress. Build a transaction table and a dirty-page table.

2. **Redo pass.** Walk the WAL forward again, replaying every committed
   transaction's actions. This restores the database to the
   pre-crash state.

3. **Undo pass.** Walk the WAL backward, undoing every uncommitted
   transaction's actions. This rolls back partial work.

The trick: by writing all changes to the WAL **before** applying them
to the database, the WAL is sufficient to recover. No data is "lost"
in the WAL sense; the question is just whether to redo or undo it.

### How Commonware's journal simplifies ARIES

A full ARIES implementation is ~10,000 lines. Commonware's journal is
much simpler because it makes stronger assumptions:

1. **Append-only.** No in-place updates. A crash mid-write leaves a
   partial item at the end. We don't need to distinguish "committed" from
   "uncommitted" — anything not yet fully written is truncated.

2. **No transactions.** Each append is atomic at the item level (via the
   varint length prefix). There's no notion of "transaction spanning
   multiple items."

3. **Truncation as undo.** The repair mechanism is "find the first
   invalid item, truncate there." That's the undo pass, simplified.

So the recovery flow is:

```
1. Open the journal (ReplayState)
2. Walk forward, decoding each item.
3. If decode succeeds, advance offset, record as valid.
4. If decode fails (EndOfBuffer, InvalidVarint, InvalidLength), stop.
5. Truncate the blob to the last valid offset.
6. Continue normal operation from the truncated position.
```

This is ARIES's analysis and undo passes merged into one forward walk.
The redo pass is trivially "do nothing — the journal is the WAL, and
we just replay it."

The result: a recovery algorithm that's correct (matches SQLite's
behavior) and simple (~100 lines).

### The "first invalid byte" principle

The key insight: a partial write leaves an invalid byte sequence at the
end. Either:

- The length prefix is malformed (truncated mid-varint).
- The length prefix is valid but points past the buffer (data not yet
  written).
- The data is corrupt (length looks valid but the bytes are garbage).

In all cases, the **first byte after the last valid item** is the
"new end of journal." Truncating there gives a consistent state.

This is what SQLite calls the "rollback journal" pattern: the WAL is
appended to, partial writes are detected on recovery, and truncation
fixes them. RocksDB does the same with its "log reader" pattern.

## Crash consistency — what `fsync` actually does

When you call `sync()` on a `Blob`, it eventually calls `fsync(2)` on
Linux (or `FlushFileBuffers` on Windows, or `fsync` on macOS via
`fcntl(F_FULLFSYNC)`). What does the OS do?

### The layers

```
fsync() in libc
  -> filesystem (ext4, XFS, ZFS, ...)
    -> block layer (page cache flush)
      -> disk driver (NVMe, SATA, virtio)
        -> physical storage (SSD, HDD, NVRAM)
```

Each layer has its own idea of "persistent."

- **`write()` syscall.** Bytes go into the page cache. They're NOT on
  disk yet. A power failure loses them.

- **`fdatasync()` syscall.** Bytes are flushed to disk. After this
  returns, the data is durable.

- **`fsync()` syscall.** Like `fdatasync`, but also flushes file
  metadata (mtime, size). Necessary if you've appended data and need
  the new size to be durable.

- **`sync_file_range()` syscall.** Flushes a range of the file. Useful
  for large files where you don't want to fsync the whole thing.

`fdatasync` is faster than `fsync` because it skips metadata. For
appends, you usually want `fsync` (so the new size is durable).

### Ordered vs journaled writes

When you write to two files in sequence (`write(file1)`, `write(file2)`,
`fsync(file1)`, `fsync(file2)`), is the order preserved on disk? Only
if the filesystem uses **journaled writes**.

- **ext4 with `data=ordered`** (the default): yes, ordering is preserved.
  The journal writes metadata before data; data is flushed in order.
- **ext4 with `data=writeback`**: NO. Data may be reordered; only metadata
  is ordered. Risky.
- **XFS**: yes, ordered.
- **ZFS**: yes, transactional (always consistent, never needs explicit
  fsync).
- **Btrfs**: yes, copy-on-write transactional.

For Commonware's journal, ordering matters: the metadata must reflect
the size **after** the data is written. If they're reordered, you can
have "metadata says file is 10 MB, but only 5 MB of data is actually
on disk." On restart, reading the "missing" 5 MB returns garbage or
zeros.

`fsync()` on each write guarantees ordering. Commonware's journal
relies on this.

### What happens on power loss

Without `fsync`:

1. `write()` returns. Data is in page cache.
2. Power fails.
3. On reboot, the page cache is empty. The disk has only what was
   written before. Data is lost.

With `fsync`:

1. `write()` returns. Data is in page cache.
2. `fsync()` is called. Kernel flushes page cache to disk, waits for
   completion.
3. `fsync()` returns.
4. Power fails.
5. On reboot, the data is on disk. Reading returns the bytes that were
   written.

This is the durability guarantee. It's what `journal.sync()` gives
you.

### The battery-backed write cache gotcha

Some disks have a **write cache** that sits between the kernel and
the platters/flash. The cache is volatile: a power failure loses its
contents. Drives with battery backup (BBWC) or supercapacitor backup
make the cache non-volatile; otherwise, `fsync()` may return before
the data is actually on the platters.

The fix: `Flush File Cache` command (ATA) or equivalent SCSI command.
Linux's `fsync()` issues this for SATA drives, but **only if the drive
supports it** (most do, but some VMs and RAID controllers don't pass
through).

If you're running Commonware in a VM, check that the virtualization
layer honors `fsync`. Some don't (e.g., older QEMU versions with
`cache=writeback` instead of `cache=strict`).

## SSD vs HDD — why storage choices matter

Commonware runs on both SSDs and HDDs, but the storage characteristics
differ enough that you should know what's under the hood.

### HDD (Hard Disk Drive)

- **Random read/write latency.** ~5-10 ms (seek + rotational latency).
- **Sequential read/write bandwidth.** ~100-200 MB/s.
- **No write amplification concern.** Writes are physically random;
  no flash erase-before-write.
- **Failure mode.** Mechanical wear (bearings, head crashes).

### SSD (Solid State Drive)

- **Random read latency.** ~50-100 microseconds.
- **Random write latency.** ~200-500 microseconds (asymmetric).
- **Sequential read/write bandwidth.** ~500 MB/s to 7 GB/s (NVMe).
- **Write amplification.** SSDs erase in 4-256 KB blocks; a 4 KB write
  requires reading the entire block, modifying, and writing back.
  Repeated small writes cause high amplification.
- **Write endurance.** SSDs wear out after ~1,000-100,000 write cycles
  per cell (depending on SLC/MLC/TLC/QLC).
- **TRIM.** When files are deleted, the OS sends TRIM to inform the SSD
  that blocks are free. The SSD can then erase them in advance, keeping
  write performance up.

### Why WAL is doubly important on SSD

On HDD: random writes are slow, so append-only journals are good.

On SSD: random writes are fast, so journals don't help as much. But:

- **Write amplification.** Small random writes on SSD amplify into large
  block writes. A WAL-style sequential append keeps writes large and
  sequential, which is good for SSD endurance.
- **Wear leveling.** Sequential writes are easier for the SSD's wear
  leveller to distribute evenly.

So the WAL pattern is good for both media. The reason differs: HDD
because of latency, SSD because of endurance.

### What Commonware assumes

Commonware's storage layer assumes:
- The OS's `fsync` is honored (battery-backed write cache or equivalent).
- The filesystem orders writes correctly (ext4 `data=ordered`, XFS, ZFS).
- The disk doesn't lie about durability (no write-only cache without
  flush).

If you're on a constrained platform (embedded device, network-attached
storage with weird caching), you might need to verify these. In tests,
the deterministic runtime uses an in-memory storage backend that
simulates durability without real I/O.

## Compression in detail — zstd and the trade-offs

`storage/src/journal/segmented/variable.rs:47-53` enables zstd compression.
Let me unpack when this is worth using.

### zstd basics

zstd (Zstandard) is a dictionary-based LZ compression algorithm from
Facebook. Properties:

- **Fast.** ~500 MB/s compression, ~1.5 GB/s decompression on a modern
  CPU. Faster than gzip, comparable to LZ4.
- **Dictionary mode.** A pre-trained dictionary on similar data can
  double the compression ratio for small items.
- **Streaming API.** Compress/decompress without loading the whole data
  into memory.
- **Levels 1-22.** Higher levels compress better but slower. Level 3 is
  the default; level 19 takes ~10x longer for ~10% better ratio.

### When compression helps for journals

For BFT consensus, the journal typically stores:

1. **Consensus messages** (votes, certificates). Small (~1 KB), highly
   redundant across views (most fields like block hash, view number
   repeat). Compression ratio: ~3-5x.

2. **Application payloads** (transactions). Highly variable; depends
   on transaction types. Often compressed already (if the application
   uses snappy or similar). Marginal additional savings.

3. **Block data**. Large, often already compressed (if it includes
   compressed state). Marginal savings.

### Trade-offs

| Aspect | Without compression | With compression |
|---|---|---|
| Disk I/O bandwidth | Full data size | Compressed size (often 2-4x smaller) |
| CPU usage | None | ~5-15% of one core |
| Latency | Direct write | Compression adds ~10-50 microseconds per KB |
| Memory | Just the buffer | zstd's internal state (~256 KB) |
| Recovery complexity | Trivial | Must remember to enable on every restart |

For most workloads, **enabling compression is a net win**. The CPU
cost is small compared to the disk I/O savings, especially for small
items where the overhead of the length prefix is significant relative
to the data.

The warning in the docs (`variable.rs:52-53`) — "if any data was
written with compression enabled, you must always set the compression
field" — is critical: data compressed with zstd cannot be read as
uncompressed data. The setting is part of the wire format.

### Dictionary compression

For very small items, zstd's default model doesn't have enough context
to compress well. The fix: a **dictionary** — a sample of similar data
that trains the model. Commonware doesn't ship dictionary mode for the
journal by default, but it's a knob if your items are tiny and similar.

## The variable-size journal frame format — the binary layout

The format in `storage/src/journal/segmented/variable.rs:9-20` is:

```
+---+---+---+---+---+---+---+---+
|       0 ~ 4       |    ...    |
+---+---+---+---+---+---+---+---+
| Size (varint u32) |   Data    |
+---+---+---+---+---+---+---+---+
```

But the actual binary layout has more nuance. Let me describe it
precisely.

### The frame

Each frame is:

```
[varint u32 length][length bytes of data]
```

The length is a varint u32 (1-5 bytes, capped per chapter 03's rule).
The data is the encoded `T: Encode` value.

For example, if `T = Certificate` and the certificate is 256 bytes,
the frame is:

```
[0x80 0x02][256 bytes of cert]
```

The `[0x80 0x02]` is varint for 256 (low 7 bits in first byte:
`0x00`; continuation bit set; second byte: `0x02`; no continuation;
result: `0x00 | (0x02 << 7) = 256`).

### The section

Frames are written sequentially into a section blob. The section blob
has no inherent structure beyond "frames concatenated":

```
section_5.blob:
+----------+----------+----------+----------+
| frame_0  | frame_1  | frame_2  | frame_3  |
| len=200  | len=180  | len=220  | len=160  |
| 200 B    | 180 B    | 220 B    | 160 B    |
+----------+----------+----------+----------+
```

The blob is named by the section number: section 5 is at
`{partition}/0000000000000005` in the storage namespace.

### The metadata header

The first 8 bytes of each section's blob name are the **big-endian
section number**. So:

- Section 0 → key `0x0000000000000000`
- Section 1 → key `0x0000000000000001`
- Section 5 → key `0x0000000000000005`
- Section 100 → key `0x0000000000000064`

This makes sections lexicographically sortable by their blob name,
which simplifies scans.

### Frame alignment

Frames are NOT aligned to any particular block size. A 200-byte frame
is followed immediately by the next frame's varint length prefix.
There's no padding, no alignment.

This is fine on SSDs (which handle arbitrary offsets efficiently) but
slightly suboptimal on HDDs (which prefer block-aligned access). For
BFT consensus where item sizes are typically 1-10 KB, the cost is
negligible.

### Checksums

Notably, the journal's frame format does NOT include checksums. The
reason: varint length + codec decode is enough validation. A corrupted
length prefix causes `InvalidVarint` on read; a corrupted data payload
causes `CodecError` on decode. Either way, the recovery mechanism
truncates at the first failure.

For higher-integrity requirements, you'd add a CRC32 or hash per
frame. Commonware doesn't, because the codec already provides strong
validation (the encoded data has internal structure that detects
corruption). This is a deliberate design choice; append it if your
threat model demands it.

## The repair mechanism — what truncation actually does

The repair is described in `variable.rs:140-145`:

> Like sqlite and rocksdb, the first invalid data read will be
> considered the new end of the journal (and the underlying Blob will
> be truncated to the last valid item). Repair occurs during replay
> (not init) because any blob could have trailing bytes.

Let me unpack this more carefully.

### The replay state machine

```
ReplayState {
    blob: Blob,
    offset: u64,           // current read offset
    valid_offset: u64,     // last successfully decoded offset
}

loop {
    match read_frame_at(&mut self.blob, self.offset) {
        Ok(frame) => {
            // Frame is valid; advance.
            self.valid_offset = self.offset;
            self.offset += frame.len_with_prefix();
            // Yield the frame to the caller.
            yield frame;
        }
        Err(Error::EndOfBuffer) => {
            // Truncated frame (data not yet written).
            break;
        }
        Err(Error::InvalidVarint(_)) |
        Err(Error::InvalidLength(_)) => {
            // Malformed frame (corruption or truncation).
            break;
        }
        Err(e) => return Err(e),  // Other error (I/O), propagate.
    }
}

// Truncate to the last valid offset.
self.blob.resize(self.valid_offset).await?;
```

The repair happens **only on `EndOfBuffer` or specific decode errors**.
Other errors (e.g., `Error::Wrapped(...)` from codec decode) propagate
as-is — the caller decides whether to truncate or retry.

### Why `resize()` rather than rewrite

Commonware uses `Blob::resize(new_len)` to truncate. This is a single
syscall (`ftruncate(2)` on POSIX). It's atomic and fast.

The alternative would be to rewrite the blob from scratch, copying only
the valid prefix. That's wasteful — `resize` just updates the file's
size metadata. No data is moved.

For SSDs, `resize` doesn't even unmap the trimmed blocks (that
requires `fallocate` with `FALLOC_FL_PUNCH_HOLE` or `FITRIM`). The
blocks are still allocated but the file is shorter. Most journaling
filesystems handle this gracefully.

### When repair is wrong (and what to do)

If the "invalid" byte is actually a sign of disk corruption (not just
a partial write), truncating silently loses data. Commonware's choice
is: **always trust the last valid item** and assume any invalid suffix
is a partial write.

For higher-integrity requirements, you could:

1. Log a warning on every truncation, with the size of the truncated
   portion.
2. Verify a hash of the truncated region against an expected hash
   (stored elsewhere).
3. Run a separate integrity scan periodically.

Commonware doesn't do these by default. The reasoning: in BFT, the
protocol is the source of truth, not the local journal. A truncated
journal means "the last few items we appended weren't durable," not
"the system is broken." The protocol will catch up via gossip.

## Page cache behavior — why Commonware's simulator doesn't need it

`storage/src/journal/segmented/variable.rs:62-65` accepts a `CacheRef`
— a shared page cache. In production, this is the OS page cache. In
the deterministic runtime (chapter 02), there's no real page cache;
the in-memory backend doesn't need one.

### Linux page cache basics

When you `read()` a file, the kernel checks the page cache first. If
the page is there (a "cache hit"), it returns immediately. If not
(a "cache miss"), the kernel reads from disk into a free page, then
returns.

`max_page_cache_size` caps how much memory the kernel will use. Beyond
that, pages are evicted (LRU).

When you `write()`, the kernel allocates a page in the cache, copies
your data, marks it dirty. Later, `fsync` flushes dirty pages to disk.

### Why Commonware exposes `CacheRef`

For production, `CacheRef` is a wrapper around the OS page cache. You
allocate a fixed-size pool of pages, share it across all journals in
your app. Reads from any journal can hit pages in the cache.

The benefit: hot pages (recently-written data, recently-read data)
stay in memory. Subsequent access is fast.

For large journals (100s of GB), you can't cache everything. The
cache becomes "the working set." As long as the working set fits,
performance is excellent.

### Why the simulator doesn't need a cache

The deterministic runtime uses an in-memory storage backend
(`runtime/src/deterministic.rs`). Reads are O(1) hashmap lookups;
writes are O(1) hashmap inserts. No real I/O. No page cache needed.

For tests, this is great: replay is instant, and recovery scenarios
can be tested by corrupting the in-memory state directly.

### `drop_caches` (for testing)

If you're benchmarking on a real system and want to test cold-cache
performance, Linux provides `/proc/sys/vm/drop_caches`:

```bash
# Drop page cache
echo 1 > /proc/sys/vm/drop_caches
# Drop page cache and slab
echo 3 > /proc/sys/vm/drop_caches
```

This forces subsequent reads to hit disk. Useful for testing that your
warm-cache assumptions don't mislead you.

## Rust pattern — synchronization for storage

The storage crate uses different patterns for different layers. Let me
walk through them.

### `Arc<RwLock<Index>>` for in-memory state

The journal maintains in-memory state (the `Index` struct: which
sections exist, their sizes, their offsets). This is shared across
tasks (the append task, the replay task, the prune task).

```rust
struct Journal {
    storage: E,                          // the abstract storage
    index: Arc<RwLock<Index>>,           // in-memory index, shared
    config: Config,
    metrics: Metrics,
}
```

`Arc<RwLock<Index>>` allows:
- Multiple concurrent readers (e.g., replay, get).
- Exclusive writer (e.g., append, prune).
- Cheap clone (just bumping the Arc refcount).

This is the standard Rust pattern for shared state with infrequent
writes.

### `parking_lot::RwLock` vs `std::sync::RwLock`

Commonware uses `parking_lot::RwLock` for several reasons:

1. **Smaller.** `parking_lot::RwLock` is ~10x smaller in memory than
   `std::sync::RwLock`.
2. **Faster.** On uncontended reads, it's a simple atomic load. The
   std version has more overhead.
3. **No poisoning.** If a thread panics while holding a `parking_lot`
   lock, the lock is released normally. The std version "poisons" the
   lock, requiring subsequent holders to handle the poison.
4. **Fair strategies.** `parking_lot` offers fair (FIFO) and
   unfair (default) locking. Std only offers unfair.

For a hot-path like journal replay, the speed difference matters.

### When `Mutex` not `RwLock`

A `Mutex` is better than `RwLock` when:
- The critical section is short (no benefit from concurrent reads).
- Writes are frequent (an `RwLock` reader blocks writers; on
  write-heavy workloads, `Mutex` may even be faster because of the
  simpler implementation).
- The protected state is not `Sync` (e.g., it contains a `RefCell`).

For the journal's index, reads happen on every `get`/`replay` (frequent),
writes happen on every `append`/`prune` (less frequent). `RwLock` is
the right choice.

For per-blob write buffers (where a writer appends frequently), `Mutex`
might be a better fit. The actual choice depends on benchmark
results.

### Atomics for counters

Commonware uses `AtomicU64` for metrics counters:

```rust
struct Metrics {
    appends_total: AtomicU64,
    bytes_appended: AtomicU64,
    truncations_total: AtomicU64,
    ...
}
```

Each counter is updated with `fetch_add(1, Ordering::Relaxed)`. Lock-
free. Cheap. Good for hot paths.

### The "spawn and ignore" pattern

Some storage operations are slow (`fsync`, large blob writes).
Commonware spawns them as separate tasks and returns a handle:

```rust
let handle: Handle<()> = blob.start_sync();
// ... do other work ...
handle.await;  // wait for sync to complete
```

The `start_sync` is non-blocking; the actual `fsync` happens in the
background. This lets the application do other work while the disk
catches up.

## The variable-size journal in detail

The main-body section "The variable-size journal frame format"
introduces the binary layout. This section goes deeper: the
frame format in detail, the section naming, the repair
mechanism, and the metrics.

### The frame format

A variable-size journal record is laid out on disk as:

```
+--------+--------+--------+-----+--------+
| len_0  | data_0 | len_1  | ... |padding |
+--------+--------+--------+-----+--------+
```

Where each `len_n` is a 4-byte big-endian length prefix, each
`data_n` is the actual record bytes, and the padding aligns the
section to a disk block boundary.

The length prefix is **canonical**: no "lazy zero-padding," no
short-form encoding. Every length prefix is exactly 4 bytes,
big-endian.

This means the parser knows exactly where each record starts and
ends, with no ambiguity.

### The section file

A section file is a single file on disk containing a sequence of
records. Each section has:

- A `section_size` (the maximum size of the section, e.g., 256 MB).
- A `pending` flag (whether there are uncommitted records in the
  section).
- A `pruned` position (the offset of the first valid record,
  after pruning).
- A `repaired` flag (whether the section has been repaired after
  a crash).

The section file is named based on its position in the journal:

```
journal/
  ├── 0000000000-0000000255    # section 0, items 0-255
  ├── 0000000256-0000000511    # section 1, items 256-511
  └── 0000000512-0000000767    # section 2, items 512-767
```

The naming scheme encodes the range of items in the filename,
which makes it easy to find a specific item by section number.

### Why sections

Sections are used instead of a single growing file for two
reasons:

1. **Pruning.** When a section is fully pruned (all items are
   below the prune threshold), the entire file can be deleted.
   This is much faster than rewriting a single growing file.

2. **Repair.** On crash recovery, only the current (in-progress)
   section needs to be repaired. Earlier sections are known to be
   consistent.

For a journal that stores a billion events, having 256 MB
sections means ~4000 sections. Pruning the oldest section
(deleting the file) is fast; pruning 1 billion records from a
single file would be slow.

### The repair mechanism

When the journal is opened after a crash, it scans the current
section for the last valid record:

1. Start at the end of the section.
2. Read the last 4 bytes as a length prefix.
3. If the length prefix is consistent with the rest of the
   section, mark this as the end of valid data.
4. Otherwise, walk backward until you find a consistent prefix.
5. Truncate the section at the first inconsistent point.

The repair mechanism is robust to torn writes: even if a partial
write happened, the checksum (or just the consistent-length-prefix
heuristic) catches it.

### The metrics

The variable-size journal exposes several metrics:

- `items`: the number of items in the journal.
- `size`: the total size of the journal in bytes.
- `sections`: the number of section files.
- `pruned`: the total number of pruned items.
- `repaired`: the total number of repaired sections.

These metrics are useful for monitoring: a sudden increase in
`repaired` indicates a crash; a growing `pruned` indicates the
pruning protocol is working.

### When the variable-size journal is right

The variable-size journal is the right choice when:

- Records have variable sizes (the common case for consensus
  events).
- Records are append-only and rarely updated.
- Pruning is done in bulk (delete old sections entirely).

For fixed-size records (e.g., telemetry samples, heartbeats),
the fixed-size journal is more efficient.

### The repair on append

If a write to the journal fails partway (e.g., a crash), the
section file may have a torn record at the end. The next time
the journal is opened, the repair logic detects this and
truncates the section.

The repair is automatic; no manual intervention is needed. The
consensus protocol ensures that any record after the truncation
point is not committed, so no data is lost.


## The fixed-size journal in detail

The main-body section "The fixed-size journal" introduces the
fixed-size journal. This section goes deeper: when to use it,
the alignment, and the throughput trade-offs.

### When to use the fixed-size journal

The fixed-size journal is the right choice when:

- Records are all the same size (e.g., telemetry, heartbeats,
  fixed-size messages).
- Records need to be written and read with O(1) random access
  (no length prefix needed).
- Throughput is more important than space efficiency.

If you have 1000 validators each sending a 256-byte heartbeat
every second, the fixed-size journal is ideal: each heartbeat
is one 256-byte record, and the journal layout is trivial.

### The alignment

Fixed-size records must be aligned to the disk's sector size
(typically 512 bytes or 4 KB). Commonware's fixed-size journal
enforces this alignment:

- Records are padded to a multiple of the alignment.
- Sections are aligned to the alignment boundary.

The alignment is important because unaligned writes are either
silently rejected by the disk or split into multiple writes,
both of which hurt performance.

### The throughput trade-offs

The fixed-size journal has higher throughput than the variable-
size journal because:

- No length prefixes (no parsing overhead).
- Predictable write sizes (no allocation per record).
- Sequential writes (the disk can stream records without
  seeking).

The cost: records that are smaller than the alignment waste
space. For a 256-byte record with 4 KB alignment, you waste
3.75 KB per record (93% wasted). For a 100-byte record, you
waste 99% of the space.

If records are typically close to the alignment size, the fixed-
size journal is a clear win. If records vary widely, the
variable-size journal is better.

### The fixed-size section

A fixed-size section file contains a sequence of fixed-size
records. Each record is at the offset `record_index *
record_size`. There are no length prefixes.

The section file is padded to a multiple of the section size,
so the total file size is exactly `num_records * record_size`.

### The repair mechanism

Like the variable-size journal, the fixed-size journal repairs
on startup:

1. Scan the current section for the last fully-written record.
2. If a torn record is detected (unaligned or with bad
   checksum), truncate the section.

Because records are fixed-size and aligned, the repair is
simpler: a torn record is just one that wasn't fully written.

### When to use each

| Workload | Use fixed-size | Use variable-size |
|----------|----------------|-------------------|
| Heartbeats (small, uniform) | ✓ | |
| Telemetry samples | ✓ | |
| Consensus events (variable) | | ✓ |
| Block data (large, variable) | | ✓ |
| Small messages, high volume | ✓ | |

For Commonware's consensus use case, the variable-size journal
is the right choice because consensus events vary in size (some
are small proposals, others are large certificate sets).


## The Manager in detail

The main-body section "The Manager — managing blobs per section"
introduces the Manager. This section goes deeper: the pruning
logic, the snapshot logic, the metrics, and the integration with
the application.

### The pruning logic

The Manager tracks a "prune boundary" — the position in the
journal below which all data can be deleted. The pruning
protocol is:

1. The application computes the prune boundary based on
   application state (e.g., "all data below height 1000 is no
   longer needed").
2. The application tells the Manager to prune up to that
   position.
3. The Manager walks the sections and deletes any section that
   is entirely below the prune boundary.
4. The Manager reports the number of pruned bytes.

The Manager is conservative: it only deletes sections that are
fully below the boundary, never partial sections. This avoids
the complexity of rewriting partial sections.

### The snapshot logic

A **snapshot** is a point-in-time copy of the application's
state. The Manager doesn't create snapshots itself, but it
supports the application's snapshot logic:

1. The application creates a snapshot of its in-memory state.
2. The application tells the Manager to remember the snapshot's
   position in the journal.
3. The Manager refuses to prune past the snapshot position (so
   the data needed to recreate the snapshot is preserved).

This is the basis for incremental snapshots: the application
can take a snapshot every N events, and the journal only needs
to preserve data back to the oldest snapshot.

### The metrics

The Manager exposes several metrics:

- `pruned_bytes`: the total number of bytes pruned.
- `pruned_items`: the total number of items pruned.
- `oldest_retained`: the position of the oldest retained item.
- `oldest_snapshot`: the position of the oldest snapshot.

These metrics are useful for monitoring: a non-zero
`oldest_snapshot` indicates the snapshot protocol is working.

### The integration with the application

The Manager is integrated with the application via a callback
or a query:

```rust
let manager = Manager::new(journal.clone());
loop {
    let prune_position = compute_prune_position(&app_state);
    manager.prune(prune_position).await?;
    let snapshot_position = compute_snapshot_position(&app_state);
    manager.set_snapshot(snapshot_position).await?;
    sleep(Duration::from_secs(60)).await;
}
```

The application decides when to prune and snapshot; the Manager
does the actual work.

### The cost of pruning

Pruning a section involves deleting the file, which is fast.
The cost is amortized over many events: a 256 MB section
containing ~100,000 events is deleted in one shot, so the
per-event pruning cost is tiny.

For very large journals (TB+), pruning may take longer, and the
application may want to throttle the pruning to avoid I/O
spikes.

### Putting it together

The Manager is the application-facing API for managing the
journal's lifecycle. It supports pruning (deleting old data)
and snapshotting (preserving data needed for recovery). The
integration is straightforward: the application decides when
to prune and snapshot, and the Manager does the actual work.

If your application needs more complex storage management
(e.g., tiered storage, archival), you'd build it on top of
the Manager. The Manager provides the basics; more complex
policies are application-specific.


## The integration with the simulator

Commonware's deterministic runtime (chapter 02) uses a simulated
disk that doesn't have a real page cache. This section explains
how the storage layer integrates with the simulator.

### The simulator's disk model

The simulator's "disk" is an in-memory byte buffer with the
following properties:

- **Deterministic.** All operations are reproducible.
- **Synchronous.** Each I/O completes immediately.
- **Crashable.** The simulator can inject crashes at any point.

The simulator's disk is *not* a real disk. It doesn't model
seek times, rotational latency, or SSD write amplification. It
just stores bytes and returns them.

### The page cache in the simulator

The simulator doesn't have a page cache. All reads go through the
disk model, which returns the bytes immediately. There's no
caching to drop.

This means:

- Reads are O(1) (just a memory copy).
- Writes are O(1) (just a memory copy).
- There's no concept of "warm" vs "cold" cache.

For testing, this is fine: the test is concerned with logic,
not performance. The page cache would add variability without
adding value.

### The crash model in the simulator

The simulator can inject crashes at any point during a test:

- Before a write completes.
- After some bytes are written but before others.
- During a `fsync()` (although in the simulator, `fsync()` is a
  no-op).

The journal's repair logic is exercised by these crashes: the
simulator can verify that the journal correctly truncates at
the first invalid byte, regardless of when the crash happens.

### Why this matters

The deterministic simulator enables property-based testing of
the storage layer: you can run thousands of test cases, each
with random operations and random crash points, and verify
that the journal always recovers to a consistent state.

Without the simulator, you'd have to manually inject crashes on
real hardware, which is slow and non-reproducible. With the
simulator, the test is fast and deterministic.

### Putting it together

The integration with the simulator is what makes Commonware's
storage layer testable. The journal's logic is exercised by
thousands of simulated scenarios, ensuring that the repair
logic works in all cases.

For production, you use the real filesystem with the real disk.
For testing, you use the simulator with the simulated disk.
The journal's behavior is identical in both cases (modulo the
performance characteristics).
## Replication in depth (DDIA Chapter 5)

The main-body section "DDIA Chapter 5 — replication, leaders,
followers, quorum" introduces replication. This section goes
deeper: the trade-offs between sync and async replication, the
three replication log types (statement-based, row-based, WAL-
based), leaderless replication with quorums, and how Commonware
fits into the replication taxonomy.

### Why replicate

Replication serves three purposes:

1. **Availability.** If one node fails, others can continue
   serving requests. Without replication, a single node failure
   takes the system down.
2. **Latency.** Geographic replication puts data closer to
   users. A user in Tokyo gets faster responses from a Tokyo
   replica than from a US replica.
3. **Throughput.** Multiple replicas can serve reads in parallel,
   increasing the system's total read capacity.

The trade-off: replication adds complexity. Keeping multiple
replicas in sync is the central challenge.

### Leader-follower replication

In **leader-follower** (also called primary-backup or master-
slave), one node is the leader and the others are followers:

- All writes go to the leader.
- The leader replicates writes to followers.
- Reads can go to either the leader or a follower (depending on
  consistency requirements).

This is the simplest replication model. Most traditional
databases (Postgres, MySQL with replication, MongoDB) use this.

The challenge: how does the leader replicate writes to
followers? Three options:

1. **Synchronous.** The leader waits for the follower to confirm
   before acknowledging the write to the client. Strong
   consistency, but slow (latency = max round-trip to any
   follower).

2. **Asynchronous.** The leader acknowledges immediately and
   replicates in the background. Fast, but the follower may be
   behind (a "replica lag").

3. **Semi-synchronous.** The leader waits for some (not all)
   followers. A compromise between consistency and latency.

Commonware's storage is asynchronous: writes are durable on the
leader before being acknowledged, and replication to followers
happens in the background. This is appropriate for BFT consensus
where each replica is itself a leader for its own consensus
instance.

### Statement-based replication

In **statement-based replication**, the leader sends the SQL
statements (or equivalent operations) to followers:

```
INSERT INTO users (name, email) VALUES ('alice', 'alice@example.com');
```

Followers execute the same statements and arrive at the same
state.

The problem: non-deterministic statements (e.g., `NOW()`,
`RAND()`, triggers) can produce different results on different
followers. This was the approach used by MySQL before row-based
replication, and it caused countless data drift bugs.

### Row-based replication

In **row-based replication**, the leader sends the row changes:

```
Table: users
Row: id=42
Before: {name: 'Alice', email: 'alice@old.com'}
After:  {name: 'Alice', email: 'alice@new.com'}
```

Followers apply the row changes. This avoids the non-determinism
of statement-based replication, but generates more data per
write.

### WAL-based replication

In **WAL-based replication**, the leader sends the raw write-
ahead log entries:

```
LSN 1000: BEGIN transaction
LSN 1001: INSERT INTO users ...
LSN 1002: COMMIT
```

Followers apply the WAL entries. This is the most efficient
format because the WAL is already structured for replay; followers
just execute the WAL.

Postgres uses WAL-based replication: the leader ships WAL
segments to followers, who replay them. This is why Postgres
replication is fast and reliable.

Commonware's journal is a WAL. Replication in Commonware
would ship WAL segments from the leader to followers, which
replay them. (Commonware doesn't have built-in replication yet,
but the journal format is well-suited to it.)

### Leaderless replication

In **leaderless** replication, any node can accept writes.
There is no leader. Examples: Amazon Dynamo, Cassandra, Riak.

The advantage: no single point of failure, and writes can be
served from any node.

The challenge: how do you keep replicas consistent without a
leader? The answer is **quorums**:

- For a write to be acknowledged, it must be persisted on W
  replicas (W = the write quorum).
- For a read to succeed, it must read from R replicas and
  reconcile the responses (R = the read quorum).

If W + R > N (where N is the total number of replicas), then
every read sees at least one replica with the latest write. This
guarantees **strong consistency**.

The trade-off: W + R > N means you can't write to all replicas
without reading from at least one of them, which adds latency.

### Commonware's replication

Commonware does not currently provide built-in replication in
the storage crate. The user is expected to run multiple nodes
and have them gossip the journal contents to each other (or
have a higher-level protocol handle replication).

For BFT consensus, each validator is itself a "replica" in the
consensus sense. The consensus protocol (Simplex, Marshal, etc.)
provides the replication: each validator runs the protocol and
arrives at the same ordered log of events.

So Commonware's replication is *application-level* (via the
consensus protocol), not *storage-level* (via the journal).
This is the right level for BFT: the consensus protocol
guarantees that all honest validators have the same events in
the same order, and each validator persists its own events
independently.

### The CAP theorem

The **CAP theorem** (Eric Brewer, 2000) says: in a distributed
system, you can have at most two of:

- **Consistency** (all nodes see the same data at the same
  time)
- **Availability** (every request receives a response)
- **Partition tolerance** (the system continues to operate
  despite network partitions)

Since partitions are inevitable in distributed systems, you
choose between consistency and availability during a partition.

Commonware chooses **consistency over availability** during
partitions: if the network is partitioned, the consensus
protocol halts progress rather than risk divergent logs. This
is the right choice for BFT consensus.

### Conflict resolution in leaderless replication

When two clients write to different replicas simultaneously, the
replicas have conflicting values. Leaderless databases resolve
conflicts using:

1. **Last-write-wins (LWW).** Use the timestamp to pick the
   winning write. Easy but loses data if clocks are skewed.
2. **Vector clocks.** Track causality between writes. More
   accurate but more complex.
3. **Application-level reconciliation.** The application handles
   conflicts (e.g., shopping cart union). Most flexible.

Commonware doesn't need conflict resolution because each event
is uniquely identified and the consensus protocol orders events.
There is no "two writes to the same key" scenario in BFT
consensus; the protocol orders all writes into a single
sequence.

### Putting it together

Replication is fundamental to distributed systems. Commonware
relies on application-level replication via the consensus
protocol, not storage-level replication via the journal. This
is the right choice for BFT: the consensus protocol provides
the strong consistency that BFT requires, and each validator
persists its events independently.

If you need storage-level replication (e.g., for snapshot
distribution), you can layer gossip or a custom protocol on top
of Commonware's journal.


## LSM trees in depth

The main-body section "LSM trees vs B-trees — the on-disk data
structure question" introduces LSM trees. This section goes
deeper: LevelDB's architecture, the write/read/compaction paths,
the size-tiered vs leveled compaction strategies, and why LSM
trees dominate for write-heavy workloads.

### LevelDB's architecture

LevelDB (designed by Jeff Dean and Sanjay Ghemawat at Google in
2011) is the canonical LSM tree implementation. Its architecture:

1. **Memtable.** An in-memory sorted data structure (typically
   a skip list). Writes go to the memtable.

2. **Immutable memtable.** When the memtable fills up, it
   becomes immutable, and a new memtable is created. Writes go
   to the new memtable.

3. **SSTables.** The immutable memtable is flushed to disk as a
   Sorted String Table (SSTable). Each SSTable is a sorted file
   on disk.

4. **Levels.** SSTables are organized into levels (typically 7
   in LevelDB). Level 0 is special: SSTables in level 0 may
   have overlapping key ranges. Levels 1+ are non-overlapping.

5. **Compaction.** Periodically, SSTables in adjacent levels
   are merged into larger SSTables in the next level. This is
   the "compaction" step.

The result is a write-optimized data structure: writes go to the
in-memory memtable, which is eventually flushed to disk as
SSTables. Reads check the memtable and all relevant SSTables.

### The write path

A write in LevelDB:

1. Append to the write-ahead log (WAL) for crash recovery.
2. Insert the key-value pair into the memtable.
3. Return success.

The write is durable (thanks to the WAL) and fast (the memtable
insertion is in-memory, O(log n) for a skip list). No disk I/O
on the write path, except the WAL append.

When the memtable fills up (typically 4 MB), it becomes
immutable, and a new memtable is started. The immutable memtable
is flushed to disk as a Level 0 SSTable in the background.

### The read path

A read in LevelDB:

1. Check the memtable.
2. If not found, check the immutable memtable (if any).
3. If not found, check Level 0 SSTables (may need to check
   multiple if keys overlap).
4. If not found, check each subsequent level (one SSTable per
   level, due to non-overlapping ranges).
5. If not found in any level, return "not found."

The number of disk reads is O(levels), which is typically 4-7
for a default LevelDB. Each level read is a binary search in a
sorted SSTable plus a possible disk I/O.

To speed up reads, LevelDB maintains **bloom filters** for each
SSTable. A bloom filter is a probabilistic data structure that
can tell you "definitely not in this SSTable" or "possibly in
this SSTable" with no false negatives and a tunable false
positive rate.

### The compaction path

Compaction is the background process that merges SSTables. There
are two main strategies:

**Size-tiered compaction:** SSTables of similar size are merged
into larger SSTables. This minimizes write amplification but
increases space amplification (multiple copies of the same key
may exist in different SSTables).

**Leveled compaction:** SSTables in level L are merged into
SSTables in level L+1, with the constraint that level L+1 SSTables
have non-overlapping key ranges. This minimizes space
amplification but increases write amplification.

LevelDB uses **leveled compaction**. RocksDB (the successor to
LevelDB) supports both strategies and lets you choose.

Commonware's journal is a simpler LSM-like structure: it
appends records to a section file, and the section file is
occasionally rotated. There's no real compaction because
Commonware's use case (an append-only log of consensus events)
doesn't need it.

### Write amplification

**Write amplification** is the ratio of bytes written to disk
vs bytes the user wrote. For LSM trees, this is typically 10x
or higher because of compaction: a single user write may be
rewritten multiple times as SSTables are merged.

For Commonware, write amplification is much lower (close to
1x) because there's no compaction. The journal just appends.

### Read amplification

**Read amplification** is the number of disk reads required to
satisfy a single user read. For LSM trees with leveled
compaction, this is O(levels) = O(log N) where N is the number
of records.

For Commonware's journal, read amplification is O(1) for a
single record: you know the section and the offset, so one read
suffices.

### Space amplification

**Space amplification** is the ratio of disk space used to the
"logical" size of the data. For LSM trees with size-tiered
compaction, this can be 2x or more (because of overlapping
SSTables). For leveled compaction, it's close to 1x.

For Commonware's journal, space amplification is close to 1x
(the section files exactly contain the records).

### Bloom filters

A **bloom filter** is a probabilistic data structure that can
tell you whether an element is in a set:

- No false negatives: if the filter says "no", the element is
  definitely not in the set.
- Possible false positives: if the filter says "yes", the
  element may or may not be in the set.

For a 1% false positive rate, a bloom filter uses about 10 bits
per element. This is much smaller than storing the elements
themselves.

Commonware's journal doesn't use bloom filters because the
record lookup is direct: you know the section and offset, so
you don't need to "find" the record in the section.

### Skip lists

The memtable in LevelDB is typically implemented as a **skip
list**: a probabilistic data structure that supports O(log n)
insertion, deletion, and lookup. Skip lists are similar to
balanced binary trees but simpler to implement.

A skip list consists of multiple levels of linked lists. The
bottom level contains all elements in sorted order. Higher
levels contain a subset of elements, with each higher level
having roughly half as many elements. Lookup proceeds by
starting at the top level and dropping down to lower levels
when an element would be exceeded.

### Why LSM trees dominate for write-heavy workloads

LSM trees have several properties that make them ideal for
write-heavy workloads:

1. **Sequential writes.** All writes go to the memtable (in-
   memory) and then to a new SSTable (sequential disk I/O).
   No random disk I/O on the write path.

2. **High write throughput.** The in-memory memtable can absorb
   millions of writes per second. Disk I/O is amortized over
   many writes via the SSTable flush.

3. **Predictable latency.** Write latency is dominated by the
   WAL append (a single sequential I/O), which is very fast.

The trade-off: reads are slower than B-trees because of the
multiple SSTable lookups. So LSM trees are best for workloads
that are write-heavy or write-and-read (but not read-heavy).

For Commonware's journal, which is essentially an append-only
log of consensus events, the LSM tree model is a perfect fit.

### Putting it together

LSM trees are the workhorse of modern write-heavy databases.
LevelDB, RocksDB, Cassandra, ScyllaDB, and many others use
variants of the LSM tree. Commonware's journal is a simpler
LSM-like structure: append to a section, occasionally rotate
the section, no compaction.

If you're building a key-value store on top of Commonware's
storage, an LSM tree would be a natural choice. For an
append-only log of events (the Commonware use case), the
journal is sufficient.


## B-trees in depth

The main-body section "B-trees" introduces B-trees. This
section goes deeper: the B+ tree invariant, the page split
algorithm, the read path with binary search, and why B-trees
dominate for read-heavy workloads.

### The B-tree structure

A **B-tree** is a self-balancing tree data structure designed
for disk-based storage. It generalizes the binary search tree
to allow each node to have many children.

A B-tree of order `m`:

- Each node has at most `m` children.
- Each internal node (except the root) has at least ⌈m/2⌉
  children.
- The root has at least 2 children (unless it's a leaf).
- All leaves are at the same depth.
- A node with `k` children contains `k - 1` keys.

For disk-based storage, `m` is typically chosen so each node
fills a disk page (e.g., 16 KB). With 64-bit keys and 128-byte
values, `m ≈ 100` for a 16 KB page.

### The B+ tree variant

A **B+ tree** is a variant used in databases:

- All values are stored in the leaves (not in internal nodes).
- Internal nodes only store keys and child pointers.
- Leaves are linked in a linked list, supporting range scans.

The advantage: internal nodes can have higher fanout (more
children per node) because they don't store values. This makes
the tree shallower and reads faster.

Most databases (Postgres, MySQL InnoDB, Oracle, SQLite) use B+
trees.

### The write path: page split

When a leaf fills up, B-tree writes trigger a **page split**:

1. The over-full leaf is divided into two leaves, each roughly
   half-full.
2. A new key (the "median" between the two halves) is inserted
   into the parent.
3. If the parent is also over-full, the split propagates up.
4. If the split propagates to the root, the root splits, and a
   new root is created. The tree grows by one level.

Page splits are expensive: they require writing the two new
leaves (typically on different disk pages), updating the parent,
and possibly cascading up the tree.

For B+ trees with a high write rate, page splits can dominate
write latency. Some databases mitigate this with **fill factor**
(leaving some space free in each page) to delay splits.

### The read path: binary search

A B+ tree read is straightforward:

1. Start at the root.
2. Binary search the keys in the current node to find the child
   pointer that contains the target.
3. Repeat for the child.
4. At the leaf, binary search the keys to find the value (or
   determine absence).

For a B+ tree with `m ≈ 100` and 1 billion records, the tree is
about 4 levels deep. A read takes 4 disk I/Os (one per level)
plus a final read of the leaf value.

This is much better than LSM trees, which take 4-7 disk I/Os
plus possible additional I/Os for bloom filter lookups and
multiple SSTable checks.

### Why B-trees dominate for read-heavy workloads

B-trees have several properties that make them ideal for read-
heavy workloads:

1. **Predictable read latency.** Every read takes O(log_m N)
   I/Os, where m is the page size. No variance based on data
   distribution.

2. **Efficient range scans.** The leaf-level linked list makes
   range scans fast: scan the first leaf, follow the linked
   list, scan each leaf in order.

3. **Strong consistency without compaction.** B-trees don't need
   background compaction (unlike LSM trees). Each write
   immediately updates the tree in place.

The trade-off: writes are slower because of page splits and
random disk I/O. So B-trees are best for workloads that are
read-heavy or read-and-write (but not write-heavy).

For a write-heavy workload like Commonware's journal (append-
only events), B-trees are the wrong choice. LSM trees (or just
an append-only log) are better.

### Postgres's B+ tree implementation

Postgres uses B+ trees for its indexes (and heap files for
table data). The relevant code is in `backend/access/nbtree/`.

The Postgres B+ tree:

- Has a page size of 8 KB by default.
- Uses a write-ahead log for crash recovery.
- Supports concurrent reads and writes via multi-version
  concurrency control (MVCC).

Postgres's B+ tree is highly optimized: it has specialized
"fast path" code for common access patterns, and it carefully
manages page splits to minimize write amplification.

### The connection to Postgres

Commonware's journal is to B+ trees what an append-only log is
to an in-memory data structure: a sequential, write-optimized
storage layer.

If you wanted to build a Postgres-like database on top of
Commonware's storage, you'd:

1. Use the journal for the WAL.
2. Use a custom storage layer for the table heap (B+ tree or
   heap file).
3. Use a custom storage layer for indexes (B+ trees).

Commonware provides the foundation (the journal); the database
layer (tables, indexes, queries) is built on top.

### B-tree concurrency

B-trees support concurrent reads via shared locks on pages.
Concurrent writes are trickier: a page split can affect multiple
pages, and care must be taken to avoid corrupting the tree.

Common B-tree concurrency techniques:

1. **Latch coupling.** Acquire a latch on the child before
   releasing the latch on the parent. This ensures the tree
   remains consistent.

2. **Lock coupling (or crab locking).** Hold a lock on the
   current page and the next page during traversal.

3. **Optimistic concurrency.** Assume no conflicts, validate at
   the end.

Postgres uses a combination of latch coupling and optimistic
concurrency for B+ tree access. The implementation is in
`backend/access/nbtree/nbtxlog.c` for the WAL logging.

### Why Commonware doesn't use B-trees for the journal

Commonware's journal is an append-only log of events. The
access pattern is:

- Append a new event (sequential write).
- Read events sequentially during replay (sequential read).
- Read a specific event by section and offset (random read,
  but rare).

For this access pattern, an append-only file is much simpler
and faster than a B+ tree. B+ trees are designed for point
lookups and range scans on large datasets; the journal's data
set is small (a single event at a time) and the access pattern
is sequential.

If you needed a B+ tree-like structure on top of Commonware's
storage, you could build it: use the journal for the WAL and
write pages to the blob store. But for the journal itself, an
append-only log is the right choice.

### Putting it together

B-trees are the workhorse of read-heavy databases. Postgres,
MySQL, Oracle, and SQLite all use them. Commonware's journal
is an append-only log instead, because the access pattern
doesn't need B-tree indexing.

If you're building a key-value store or a database on top of
Commonware's storage, B-trees are the natural choice for the
index layer. Commonware provides the storage foundation; the
index layer is application-specific.


## Write-ahead logging in depth

The main-body section "Write-ahead logging — ARIES and the three
passes" introduces WAL and ARIES. This section goes deeper: the
ARIES algorithm in full detail, the LSN, the dirty page table,
the transaction table, and why Commonware's journal is a
simplified ARIES.

### The ARIES algorithm

**ARIES (Algorithm for Recovery and Isolation Exploiting
Semantics)** was developed at IBM in the early 1990s and is
the basis for most modern database recovery. It has three
phases:

1. **Analysis.** Starting from the last checkpoint, scan the
   WAL forward to determine:
   - Which transactions were active at the time of the crash?
   - Which pages were dirty (modified in memory but not on
     disk)?

2. **Redo.** Starting from the earliest dirty page, replay the
   WAL to bring all pages up to date as of the crash.

3. **Undo.** For each transaction that was active at the crash,
   undo its changes (by writing compensating log records).

The key insight: redo restores the database to the exact state
at the crash (including uncommitted transactions), then undo
rolls back the uncommitted transactions. This gives the
**steal/no-force** property: pages can be flushed to disk before
transactions commit, and transactions don't have to force all
their changes to disk on commit.

### The Log Sequence Number (LSN)

Every WAL record has a **Log Sequence Number (LSN)** — a
monotonically increasing identifier. The LSN serves three
purposes:

1. **Ordering.** Records with smaller LSNs were written before
   records with larger LSNs. This gives a total order of
   operations.

2. **Position.** The LSN tells you where in the WAL the record
   lives. You can read a record by its LSN.

3. **Page tracking.** Each page has an LSN (`pageLSN`) that
   records the LSN of the last WAL record that affected the
   page. During redo, if a WAL record's LSN ≤ pageLSN, the
   record's change is already on disk.

In Commonware's journal, the equivalent of the LSN is the
section number and the offset within the section.

### The dirty page table

The **dirty page table** is an in-memory structure that tracks
which pages are dirty (modified in memory but not on disk). For
each dirty page, the table records:

- The page's identifier.
- The LSN of the first WAL record that made the page dirty
  (the "recLSN").

During the analysis phase, ARIES reconstructs the dirty page
table by scanning the WAL from the last checkpoint. During
the redo phase, ARIES starts from the smallest recLSN in the
table, ensuring all dirty pages are brought up to date.

In Commonware's journal, there is no dirty page table: the
journal is the WAL, and the "pages" are sections. A section is
"dirty" if it has uncommitted records (i.e., records after the
last commit point).

### The transaction table

The **transaction table** tracks active transactions: their ID,
their state (active, committed, aborted), and their last LSN.
During analysis, ARIES reconstructs the transaction table from
the WAL. During undo, ARIES uses the table to determine which
transactions need to be rolled back.

In Commonware's journal, transactions are tracked at the
application level. The journal just records the events; the
consensus protocol decides which events are committed.

### The redo pass

The redo pass replays the WAL from the earliest dirty page (or
the last checkpoint) forward. For each WAL record:

1. If the record's LSN ≤ pageLSN, the change is already on disk;
   skip.
2. Otherwise, apply the change to the page and update pageLSN.

This brings all dirty pages up to date as of the crash.

For Commonware, the redo pass is the "replay" function: the
journal is replayed from the last commit point forward,
rebuilding the in-memory state.

### The undo pass

The undo pass rolls back uncommitted transactions. For each
transaction that was active at the crash, ARIES follows the
chain of log records backward, writing compensating log records
(CLRs) to record the undo operations.

In Commonware, the consensus protocol ensures that no
uncommitted transactions exist at the time of a crash: a
block is only durable after the protocol commits it. So
there's no undo pass; only redo.

### The "first invalid byte" principle

Commonware's journal uses a **first invalid byte** pointer to
mark where valid data ends. After a crash, the journal is
truncated at the first invalid byte (the last successfully
written record).

This is simpler than ARIES: instead of tracking dirty pages and
uncommitted transactions, Commonware just records "data is valid
up to here, anything beyond is suspect." The reasoning: in a BFT
consensus protocol, all data is committed before being written
to the journal (the consensus protocol ensures this), so there's
no need to roll back uncommitted transactions.

### The compensation log record (CLR)

In ARIES, a CLR is a special log record that records an undo
operation. CLRs are themselves WAL records, so they are
replayable. This is how ARIES handles nested undo operations:
each undo creates a CLR, and the CLRs can be replayed during
recovery.

Commonware's journal doesn't need CLRs because there's no
undo. Every record in the journal is part of a committed
sequence.

### The check-point

Periodically, ARIES writes a **checkpoint** record to the WAL
that includes:

- The current dirty page table.
- The current transaction table.

After a crash, recovery starts from the most recent checkpoint,
not from the beginning of the WAL. This limits the recovery
time.

Commonware's journal doesn't have checkpoints; recovery always
replays from the beginning. For small journals (a few GB), this
is fine. For large journals (TB+), checkpoints would be
needed.

### Putting it together

ARIES is the gold standard for database recovery. Commonware's
journal is a simplified version: no dirty page tracking, no
transaction tracking, no undo. The reason: the consensus
protocol provides the "no uncommitted transactions" guarantee,
so ARIES's complexity is unnecessary.

If you needed full ARIES-style recovery (e.g., for a general-
purpose database on top of Commonware's storage), you'd build
it. Commonware's journal is the foundation; the recovery layer
is application-specific.


## Crash consistency in depth

The main-body section "Crash consistency — what fsync actually
does" introduces the topic. This section goes deeper: the
layers between `fsync` and the disk, the ordered vs journaled
write distinction, write barriers, and the BBWC gotcha.

### The layers

When you call `fsync(fd)` on a Linux file descriptor, the bytes
travel through several layers before hitting the disk:

1. **Your application.** Calls `write()` to put bytes in the
   page cache, then `fsync()` to flush them.

2. **Page cache.** The kernel's in-memory cache of file pages.
   `write()` puts bytes here; `fsync()` triggers flushing.

3. **Block layer.** The kernel's block device layer, which
   schedules I/O requests to the underlying device.

4. **Device driver.** The driver's queue, which submits I/O
   requests to the device.

5. **Disk controller.** Schedules the actual I/O on the disk.

6. **Physical disk.** The medium itself (HDD platter, SSD flash).

`fsync()` ensures that the bytes have made it through all these
layers and are persistent on the physical disk. But the
"persistent" guarantee depends on the hardware: an HDD's
platter, an SSD's flash, or a disk controller's cache.

### Ordered vs journaled writes

Filesystems like ext4 support two modes for metadata writes:

**Ordered mode:** File data is written to disk before the
associated metadata (e.g., the inode update that increases the
file size). This ensures that after a crash, the file's
contents are consistent with its size.

**Journaled mode:** Both file data and metadata are written to
the journal before being applied to the filesystem. This is
slower but provides stronger consistency.

Commonware's journal is structured to avoid filesystem-level
consistency issues:

- Writes are aligned to disk blocks (typically 4 KB).
- The journal uses checksums to detect torn writes (a write that
  spans multiple blocks and is partially written).
- The repair logic truncates at the first invalid byte, so
  partial writes are handled gracefully.

### Write barriers

A **write barrier** is an instruction to the disk controller
(or the device) that says "all I/O submitted before this
barrier must be completed before any I/O submitted after." This
ensures ordering across I/O requests.

Without barriers, the disk controller may reorder I/O requests
for performance. This is fine if the application doesn't care
about ordering, but catastrophic if it does (e.g., writing a
data block before its corresponding metadata block).

Commonware's journal uses write barriers via `fsync()` to
ensure that records are written in order. Without this, a crash
could leave the journal in an inconsistent state.

### The BBWC gotcha

A **Battery-Backed Write Cache (BBWC)** is a battery-backed
RAM on the disk controller that caches writes. The battery
ensures the cache survives a power loss.

The gotcha: if the disk controller reports a write as "complete"
when it's only in the BBWC (not yet on the physical disk), then
a power loss can lose the write — *even though the application
called `fsync()`*.

To prevent this, you need to either:

1. **Disable the BBWC.** Some controllers have a "write-through"
   mode that bypasses the cache.
2. **Use `FLUSH CACHE` commands.** SCSI and SATA have explicit
   commands to flush the BBWC.

Commonware's journal assumes `fsync()` is sufficient, which is
true for most modern disks. For high-reliability deployments,
you might want to verify that the BBWC is in write-through mode
or use direct I/O (`O_DIRECT`) to bypass the cache entirely.

### Filesystem differences

Different filesystems have different consistency properties:

**ext4:** The default Linux filesystem. Uses journaling for
metadata; supports ordered and journaled modes. Generally
reliable.

**XFS:** Another common Linux filesystem. Uses journaling for
metadata; generally reliable. Has better scalability for large
files than ext4.

**ZFS:** A copy-on-write filesystem with built-in checksumming.
Provides the strongest consistency guarantees but is more
complex.

**Btrfs:** Similar to ZFS, with copy-on-write and checksumming.

Commonware's journal works on all these filesystems, but the
performance and reliability characteristics differ. For
high-reliability deployments, ZFS or Btrfs are preferred because
of their built-in checksumming and copy-on-write semantics.

### Direct I/O (O_DIRECT)

**O_DIRECT** is a Linux flag that bypasses the page cache. Reads
and writes go directly to the disk.

The advantage: predictable latency (no page cache thrashing).
The disadvantage: the application must manage its own caching,
and the page cache isn't used for subsequent reads.

Commonware doesn't use O_DIRECT by default; it relies on the
page cache for reads. The page cache provides excellent read
performance for repeated access patterns.

### The fsync cost

`fsync()` is expensive: it forces a flush of the dirty pages
to the disk, which can take milliseconds (HDD) to tens of
microseconds (SSD). For a write-heavy workload, `fsync()` can
be a bottleneck.

Commonware mitigates this with **batched sync**: writes from
multiple records are batched into a single `fsync()`. This
amortizes the `fsync()` cost over many writes.

### The atomicity of sector writes

Modern disks guarantee that sector-sized writes (typically 512
bytes or 4 KB) are atomic: a write either completes or doesn't,
with no torn state.

Commonware's journal is designed around this: each record is
aligned to the sector size, so a partial write is detected by
a checksum mismatch, not by a torn state.

### Putting it together

Crash consistency is a deep topic with many subtleties.
Commonware's journal is designed to handle the common cases
(aligned writes, checksums, repair on startup) but assumes a
modern filesystem with reasonable behavior. For high-
reliability deployments, you should verify the filesystem's
behavior on power loss and consider ZFS or Btrfs for the
strongest guarantees.


## Compression in depth

The main-body section "Compression in detail — zstd and the
trade-offs" introduces zstd and the trade-offs. This section
goes deeper: zstd's internal algorithm, dictionary compression,
and when each compression algorithm is right.

### zstd internals

**zstd** (developed by Yann Collet at Facebook, 2015) is a
modern compression algorithm with a focus on speed. Its
architecture:

1. **Huffman coding on literals.** The input bytes are first
   encoded with a Huffman code based on byte frequency.

2. **Dictionary matching.** Repeated sequences are replaced with
   back-references (offset, length) to a dictionary.

3. **Huffman coding on sequences.** The sequence of literals
   and back-references is encoded with another Huffman code.

4. **Entropy coding.** The final Huffman-coded stream is
   entropy-coded with FSE (Finite State Entropy), a fast
   ANS-based entropy coder.

The combination of dictionary matching and entropy coding gives
zstd compression ratios close to zlib (gzip) but at much
higher speeds (~5x faster compression, ~2x faster
decompression).

### Dictionary compression

**Dictionary compression** is a zstd feature that uses a
precomputed dictionary to improve compression for small inputs
(or inputs with repeated patterns). The idea:

1. Train a dictionary on a sample of representative data.
2. Both encoder and decoder load the dictionary.
3. The encoder uses the dictionary for back-references; the
   decoder uses it to resolve those references.

For data with repeated patterns (e.g., JSON with similar keys,
protocol buffers with the same fields), dictionary compression
can improve the compression ratio by 2-5x.

Commonware's journal doesn't use dictionary compression by
default because the data is varied and small. For specialized
use cases (e.g., compressing a specific type of consensus
event), dictionary compression could be a win.

### LZ4

**LZ4** is another modern compression algorithm, designed for
maximum speed. Its compression ratio is worse than zstd (about
10-20% lower), but it's 2-3x faster to compress and decompress.

LZ4 is used in cases where compression speed is critical:
- Real-time data streams.
- High-throughput message buses.
- Network protocols with low latency requirements.

Commonware's journal doesn't use LZ4 by default, but it could
be configured for latency-sensitive workloads.

### Snappy

**Snappy** (developed by Google in 2011) is the predecessor to
LZ4 and zstd. Its compression ratio is similar to LZ4, and its
speed is between LZ4 and zstd.

Snappy is used in Hadoop, Cassandra, and many other systems.
Commonware doesn't use Snappy directly, but the patterns are
similar.

### When to use each

| Algorithm | Compression ratio | Speed         | When to use |
|-----------|-------------------|---------------|-------------|
| zstd      | High              | Medium        | Default choice |
| LZ4       | Medium            | Very fast     | Latency-critical |
| Snappy    | Medium            | Fast          | Compatibility with existing systems |
| gzip/zlib | High              | Slow          | Maximum compression, slow speed OK |

For Commonware's journal, zstd is the default because the
compression ratio is high and the speed is sufficient. For
latency-sensitive workloads, LZ4 might be preferred.

### The cost of compression in the hot path

Compression in the write path adds CPU overhead: each record
must be compressed before being written. For a high-throughput
journal, this can be a significant fraction of the CPU time.

Commonware mitigates this by:

1. **Optional compression.** Compression is opt-in; if you don't
   need it, you can disable it.
2. **Lower compression levels.** zstd levels 1-3 are very fast;
   only levels 19-22 are slow.
3. **Batched compression.** Multiple records are compressed
   together, amortizing the per-record overhead.

For most workloads, compression is a net win: the disk I/O
saved is larger than the CPU cost.

### The decompression cost in replay

During replay (e.g., after a crash), records must be
decompressed. This is faster than compression (typically 2x),
so the replay cost is dominated by disk I/O, not decompression.

Commonware's replay function handles decompression transparently:
the application sees decompressed records regardless of the
underlying compression.

### Putting it together

Compression is a useful tool for reducing disk usage and
improving I/O throughput, at the cost of CPU. Commonware's
journal supports compression via zstd, with the ability to
configure the compression level. For most workloads, the
default (no compression) is fine; for space-constrained or
I/O-bound workloads, compression is a win.


## Page cache in depth

The main-body section "Page cache behavior" introduces the topic.
This section goes deeper: Linux page cache internals, when to
bypass (O_DIRECT), and the `drop_caches` debugging tool.

### Linux page cache basics

The Linux page cache is a kernel-managed cache of file pages.
When you `read()` a file, the kernel first checks the page
cache; if the data is there (a "cache hit"), it returns
immediately. If not (a "cache miss"), the kernel reads from the
disk and adds the data to the cache.

When you `write()` a file, the kernel writes to the page cache
(marking the page "dirty") and returns immediately. The actual
disk write happens later, either when the page is evicted
("write-back") or when the application calls `fsync()`.

The page cache is transparent to applications: you don't need
to do anything special to benefit from it.

### The benefits

The page cache provides several benefits:

1. **Read acceleration.** Repeated reads of the same data are
   served from memory, which is orders of magnitude faster than
   disk.

2. **Write coalescing.** Multiple small writes can be combined
   into a single large disk write, improving throughput.

3. **Prefetching.** The kernel can prefetch data ahead of the
   application's read pattern, hiding disk latency.

4. **Lazy fsync.** `fsync()` only needs to flush the dirty
   pages; if the application has already-written data is
   already on disk, `fsync()` is a no-op for those pages.

### The size

The page cache size is dynamic: it grows as applications read
and write, and shrinks when memory is needed for other
purposes. You can see the current size with:

```
$ cat /proc/meminfo | grep -i cache
Cached:          12345678 kB
```

For a server with 64 GB of RAM, the page cache might use 30-50
GB at steady state.

### The eviction policy

The Linux kernel uses a variant of LRU (Least Recently Used) to
evict pages from the page cache. Pages that haven't been used
recently are evicted first.

The eviction is done in "page reclaim" threads that run in the
background. Under memory pressure, the reclaim threads are
scheduled more aggressively.

### When to bypass

There are cases where bypassing the page cache is desirable:

1. **Self-caching applications.** If your application has its
   own cache (e.g., an LSM tree's memtable), the page cache is
   redundant. Bypassing it saves memory.

2. **Predictable latency.** The page cache can introduce
   unpredictable latency (a cache miss is much slower than a
   hit). For applications that need consistent latency,
   bypassing the page cache may be preferable.

3. **Direct I/O to user-space buffers.** Some applications
   (e.g., databases with their own buffer pool) prefer to
   manage their own memory.

For Commonware's journal, the page cache is desirable: reads
of recently-written records are common (e.g., during replay),
and the page cache provides them at memory speed.

### The `O_DIRECT` flag

Linux's `O_DIRECT` flag opens a file with direct I/O, bypassing
the page cache. Reads and writes go directly to the disk.

The catch: with `O_DIRECT`, the application must:

- Align reads and writes to the disk's sector size (typically
  512 bytes or 4 KB).
- Ensure buffer addresses are aligned to the sector size
  (some systems require page alignment).

Misaligned direct I/O fails with `EINVAL`. So `O_DIRECT` is
tricky to use correctly.

Commonware's journal doesn't use `O_DIRECT` by default; the
page cache is preferred. If you need predictable latency, you
can configure `O_DIRECT` for the journal's section files.

### The `drop_caches` tool

For debugging cache-related issues, Linux provides
`/proc/sys/vm/drop_caches`. You can drop the page cache with:

```
# echo 1 > /proc/sys/vm/drop_caches   # drop page cache
# echo 2 > /proc/sys/vm/drop_caches   # drop slab cache
# echo 3 > /proc/sys/vm/drop_caches   # drop both
```

This is useful for benchmarking: drop the cache, run the
benchmark, and observe the cold-cache performance.

Commonware's `CacheRef` type (mentioned in the main body)
exposes this functionality in a controlled way: you can drop
the page cache from within a test, ensuring that subsequent
reads are cold.

### The deterministic runtime

Commonware's deterministic runtime (chapter 02) doesn't have a
page cache: all reads are deterministic and synchronous.
There's no I/O scheduler, no prefetching, no write coalescing.

This is by design: deterministic I/O is essential for
reproducible testing. The runtime simulates the disk, and
all "reads" return data from the simulated disk immediately.

For production, you use the real filesystem with the real page
cache. For testing, you use the deterministic runtime with the
simulated disk.

### Putting it together

The Linux page cache is a transparent, high-performance cache
that benefits most applications. Commonware's journal relies on
it for read acceleration. The deterministic runtime bypasses
it for testing, ensuring that all I/O is reproducible.

If you need predictable latency, you can use `O_DIRECT` to
bypass the page cache. But this is rarely needed for
Commonware's use case (a consensus log), where the page
cache's variable latency is acceptable.


## Exercises

These exercises build hands-on familiarity with the storage
layer. They are ordered roughly by increasing depth.

### Exercise 1: Read a section file

Open one of Commonware's section files with a hex editor. Identify
the length prefixes, the data records, and the padding. Decode
a few records manually.

### Exercise 2: Simulate a crash

Use Commonware's `Storage` API to write some records. Then
kill the process abruptly (kill -9). Restart and observe the
journal repair in action.

### Exercise 3: Compare fixed vs variable journals

Write the same data to a fixed-size journal and a variable-size
journal. Measure the disk usage and throughput for each.

### Exercise 4: Test compression

Configure the variable journal with compression enabled. Write
the same data with and without compression. Measure the
resulting file sizes and the throughput difference.

### Exercise 5: Drop the page cache

Write a workload that reads records repeatedly. Measure the
throughput. Then drop the page cache and measure the throughput
again. Compare the results.
## Appendix A — The variable-size journal, in detail

`storage/src/journal/segmented/variable.rs:1-160`. The on-disk format:

```
Section 0 blob:
  +-------+-------+-------+-----+-------+-------+-----+-------+
  | len_0 | data_0| len_1| data_1| ... |len_n-1|data_n-1| (padding)
  +-------+-------+-------+-----+-------+-------+-----+-------+
  where len_i is a varint u32 (1-5 bytes), data_i is len_i bytes

Section 1 blob:
  +-------+-------+-------+-----+...
  | len_0 | data_0| len_1| data_1| ...
  +-------+-------+-------+-----+...
```

Each section is a **separate Blob** in the storage partition. Blobs are
addressed by `section.to_be_bytes()` (so section 0 → key `0x0000000000000000`,
section 1 → `0x0000000000000001`, etc.).

### The `frame.rs` — parsing layer

`storage/src/journal/frame.rs`. The `Frame` struct wraps an item with
its position metadata:

```rust
pub struct FrameInfo {
    pub offset: u64,        // start offset in the section blob
    pub len: u32,           // length of the data
    pub valid: bool,        // was the frame valid on read?
}
```

The `encode_frame_into` function writes a frame:

```rust
pub fn encode_frame_into(buf: &mut BytesMut, data: &[u8]) {
    let len = data.len() as u32;
    UInt(len).write(buf);     // varint length prefix
    buf.put_slice(data);      // raw data
}
```

The `decode_item` function reads a frame:

```rust
pub fn decode_item(buf: &[u8], offset: u64) -> Result<(FrameInfo, &[u8]), Error> {
    let len = decode_length_prefix(buf, offset)?;
    let data_offset = offset + len.encode_size() as u64;
    if data_offset + len as u64 > buf.len() as u64 {
        return Err(Error::EndOfBuffer);  // truncated
    }
    let info = FrameInfo {
        offset,
        len,
        valid: true,
    };
    Ok((info, &buf[data_offset as usize..(data_offset + len as u64) as usize]))
}
```

If the length prefix is malformed (too large for buffer), or the data
extends past the buffer end, decoding fails.

### The repair mechanism

`variable.rs:140-145` (mentioned in chapter 06 main text):

> Like sqlite and rocksdb, the first invalid data read will be
> considered the new end of the journal (and the underlying Blob will be
> truncated to the last valid item). Repair occurs during replay (not
> init) because any blob could have trailing bytes.

The implementation (`variable.rs:3930+` in `ReplayState`):

```rust
loop {
    match read_frame_at(&mut self.blob, self.offset) {
        Ok(frame) => {
            // Decode succeeded, advance offset
            self.valid_offset = self.offset;
            self.offset += frame.len_with_prefix();
            // Emit the frame
        }
        Err(Error::EndOfBuffer) => {
            // Truncated frame — end of valid data
            break;
        }
        Err(Error::InvalidVarint(_)) | Err(Error::InvalidLength(_)) => {
            // Corrupt frame — end of valid data
            break;
        }
        Err(e) => return Err(e),  // Other error, propagate
    }
}
// Truncate to last valid offset
self.blob.resize(self.valid_offset).await?;
```

If the last item was partially written (e.g., only the length prefix
made it to disk), the journal truncates past it. The next append starts
at the truncated position.

### The compression option

`variable.rs:47-53`:

> `Journal` supports optional compression using `zstd`. This can be
> enabled by setting the `compression` field in the `Config` struct to
> a valid `zstd` compression level.

When compression is enabled, each item is compressed before being
written. The decoded item is uncompressed lazily.

**Warning from the docs** (`variable.rs:52-53`):

> This setting can be changed between initializations of `Journal`,
> however, it must remain populated if any data was written with
> compression enabled.

If you compress some items and not others, the journal will fail to
replay. Set `compression` consistently across restarts.

## Appendix B — The fixed-size journal

`storage/src/journal/segmented/fixed.rs`. Items are fixed-size:

```
Section N blob:
  +--------+--------+--------+----------+
  | item_0 | item_1 |   ...  | item_k-1 |
  +--------+--------+--------+----------+
```

Item `i` is at offset `i * CHUNK_SIZE`. Reading is trivial:
`blob.read_at(i * CHUNK_SIZE, CHUNK_SIZE)`.

The trade-off: simpler format, faster random access, but you have to know
the item size ahead of time.

Used by `storage/src/index/` and similar — when items are fixed-size
structs (e.g., hashes, fixed records), fixed journal is faster.

### Size constraints

`fixed.rs:47-57`:

```rust
pub struct Config {
    pub partition: String,
    pub page_cache: CacheRef,
    pub write_buffer: NonZeroUsize,
}
```

`CHUNK_SIZE` comes from the generic type parameter `A: CodecFixed`. For
example, `Journal<Context, [u8; 32]>` has `CHUNK_SIZE = 32`.

If you try to use a non-fixed-size type, the compiler rejects it.

## Appendix C — The `Manager` — managing blobs per section

`storage/src/journal/segmented/manager.rs`. The Manager tracks which
section blobs exist and handles blob lifecycle.

Key responsibilities:

1. **Lazy blob creation** — when you append to section N, the Manager
   creates the blob for N if it doesn't exist.
2. **Blob caching** — recently-used blobs are kept open (per the OS
   file descriptor limit).
3. **Eviction** — when too many blobs are open, the Manager evicts the
   least-recently-used.
4. **Pruning** — when you call `prune(min_section)`, the Manager removes
   blobs for sections below `min_section`.

The Manager is generic over the append factory:

```rust
pub struct Manager<E: Storage + Metrics, F: AppendFactory> { ... }
```

`AppendFactory` is a trait that produces `Writer` instances (a wrapper
around `Blob` with write buffering).

## Appendix D — `oversized.rs` — items too big for a section

`storage/src/journal/segmented/oversized.rs`. Some items don't fit in a
single section's blob (e.g., a 100 MB block in a 16 MB-sectioned blob).

The `Oversized` struct stores such items in **separate blobs** keyed by
their position:

```rust
pub struct Oversized {
    // Stored in: {partition}_oversized/{section_be}/{offset_be}
}
```

So if you append a 100 MB item at section 5, offset 1000, the actual
data goes to `{partition}_oversized/0000000000000005/00000000000003e8`.
The main section's blob gets a small **pointer** entry pointing to it.

On replay, the journal reads the pointer, then reads the oversized blob,
then returns the reconstructed item. The consumer doesn't see the
complexity — it just sees the item.

## Appendix E — The `metadata` crate

`storage/src/metadata/`. Atomic metadata writes.

A metadata entry is `(key, value)` where both are arbitrary byte strings.
Metadata is stored in a dedicated blob (one per partition). All writes
are atomic — either the new value is fully written or the old value
stays.

```rust
let mut meta = Metadata::new(context, "partition").await?;
meta.put(b"latest_height", &100u64.to_be_bytes()).await?;
meta.sync().await?;
```

Used by Marshal (chapter 12) to track "what's the latest finalized
block?" durably and atomically.

## Appendix F — The `archive` crate

`storage/src/archive/`. Fixed-size value archive. Like a key-value store
where values are all the same size.

```rust
pub struct Archive<E: Storage, V: CodecFixed> {
    storage: E,
    partition: String,
    next_index: u64,
    // ...
}
```

- `Archive::put(value) -> u64` — append a value, return its index.
- `Archive::get(index) -> Option<V>` — fetch by index.
- `Archive::sync() -> ()` — durably persist.

Used for: finalized certificates (chapter 12), notarization logs, etc.
When values are fixed-size, Archive is faster than a general KV store.

### Pruning

`Archive::prune(below_index)` — delete all values with index <
`below_index`. The Archive keeps the metadata (next_index, etc.) but
frees the disk space.

## Appendix G — The `freezer` crate

`storage/src/freezer/`. Large immutable data archive.

Used for: blocks (chapters 11, 12), transaction batches, snapshots.

```rust
let mut freezer = Freezer::new(context, "blocks").await?;
freezer.put(block_data).await?;
freezer.sync().await?;

// Retrieve
let data = freezer.get(index).await?;
```

Unlike `Archive`, values can be variable-size. The internal layout
handles large items by splitting across multiple underlying blobs (like
the oversized mechanism for journals).

## Appendix H — The `ordinal` crate

`storage/src/ordinal/`. Ordinal key-value store.

The "ordinal" part means: keys have a total ordering, and operations can
be efficiently range-scanned.

```rust
let mut ord = Ordinal::new(context, "store").await?;
ord.put(b"key1", value1).await?;
ord.put(b"key2", value2).await?;

let mut iter = ord.into_iter();
while let Some((key, value)) = iter.next().await? { ... }
```

Internally, uses a log-structured merge tree (LSM-tree) variant. Writes
are appended; reads scan sorted runs; compaction merges runs in the
background.

Used for: secondary indexes, range queries, anything needing sorted
access.

## Appendix I — The `index` crate

`storage/src/index/`. ULTRA-coded compressed index.

ULTRA is a compression scheme for sorted integer sequences. If you have
millions of monotonically increasing values (like block heights or
sequence numbers), ULTRA stores them in far less space than naive
storage.

```rust
let mut index = Index::new(context, "heights").await?;
index.put(1).await?;
index.put(5).await?;
index.put(100).await?;
// Compact storage: ~16 bits per element vs 64 bits for a u64.
```

Used by authenticated databases (chapter 16) to store operation
positions cheaply.

## Appendix J — The `rmap` crate

`storage/src/rmap/`. Rank map.

Maps each ordinal position to its **rank** (the count of elements ≤ that
position). Used in authenticated databases to do efficient
"key-at-position" lookups.

## Appendix K — The `translator` trait

`storage/src/translator.rs`. Translates domain keys to internal keys.

The default is identity translation (key `K` → key `K`). Apps can plug
in a custom translator for keys that don't naturally fit in the storage
key space:

```rust
pub trait Translator: Send + Sync + 'static {
    type Key: Array;
    fn prefix(&self, key: &[u8]) -> Option<Vec<u8>>;  // none means "no match"
    fn extract(&self, key: &[u8]) -> Option<Vec<u8>>;
}
```

- `prefix` — given an internal key, return the original domain key's
  prefix (used for queries).
- `extract` — given a domain key, compute the internal key's prefix
  (used for inserts).

Used by ADB/QMDB (chapter 16) to map app-level keys (e.g., variable-length
strings) to fixed-size storage keys.

## Appendix L — Storage testing patterns

`AGENTS.md` documents the standard test patterns. The key ones:

### Pattern: simulate partial writes

```rust
// Write valid data
journal.append(1, valid_data).await?;

// Simulate a partial write
let (blob, size) = context.open(&partition, &1u64.to_be_bytes()).await?;
blob.resize(size - 1).await?;  // truncate last byte
blob.sync().await?;

// Verify recovery truncates to last valid item
let journal = Journal::init(context, cfg).await?;
assert_eq!(journal.size().await?, expected_truncated_size);
```

### Pattern: simulate corruption

```rust
// Manually corrupt data
let (blob, size) = context.open(&partition, &1u64.to_be_bytes()).await?;
blob.write_at(size - 4, vec![0xFF; 4]).await?;  // corrupt last 4 bytes
blob.sync().await?;

// Verify journal recovers
let journal = Journal::init(context, cfg).await?;
let stream = journal.replay(buffer_size).await?;
// Corrupted items are skipped/truncated
```

### Pattern: checkpoint-based crash recovery

```rust
let mut checkpoint = None;
loop {
    let runner = if let Some(c) = checkpoint.take() {
        deterministic::Runner::from(c)
    } else {
        deterministic::Runner::timed(Duration::from_secs(30))
    };
    let (complete, next) = runner.start_and_recover(f);
    if complete { break; }
    checkpoint = Some(next);
}
```

`start_and_recover` runs the closure, then returns the executor's state
as a `Checkpoint`. The next iteration resumes from that checkpoint.

This is **how you test unclean shutdown**. The journal sees whatever
state made it to disk, then recovers.

## Appendix M — Common pitfalls

### Forgetting to sync before broadcasting

From chapter 06 main text, this is the canonical WAL pattern. Forgetting
`sync()` before broadcasting means your broadcast might arrive at peers
before your own disk has the message. On crash + restart, you'd be
inconsistent with your peers.

### Setting the wrong section for pruning

When you prune, you choose a minimum section. If you prune too
aggressively (e.g., before the journal's replay has caught up to that
section), you lose data.

Rule of thumb: prune only after finalization of that section is
complete.

### Using `compression: Some(level)` inconsistently

Covered in Appendix A. If you enable compression once, you can't disable
it (the data won't decode). Make compression a stable decision for the
lifetime of the journal.

### Mixing `Fixed` and `Variable` on the same partition

Both journals use the same partition name; they have different
on-disk formats. Don't try to instantiate both on the same partition —
the data is incompatible.

## Where to look in the code (expanded)

- `storage/src/journal/segmented/variable.rs` — the main journal.
- `storage/src/journal/segmented/fixed.rs` — fixed-size variant.
- `storage/src/journal/segmented/oversized.rs` — large-item handling.
- `storage/src/journal/segmented/manager.rs` — section blob management.
- `storage/src/journal/frame.rs` — frame encoding/decoding.
- `storage/src/metadata/` — atomic metadata.
- `storage/src/archive/` — fixed-size archive.
- `storage/src/freezer/` — large immutable archive.
- `storage/src/ordinal/` — ordinal KV.
- `storage/src/index/` — ULTRA-coded index.
- `storage/src/rmap/` — rank map.
- `storage/src/translator.rs` — the translator trait.
- `consensus/src/simplex/engine.rs` — how Simplex uses the journal as WAL.

## If you only remember three things

1. **Two layers: `Storage` (abstract) → `Blob` (byte range) → `Journal` (log).** Tests use in-memory storage; production uses filesystem or cloud.
2. **Sync is a durability barrier.** Writes go to OS page cache; `sync()` forces them to disk. Consensus engines sync before broadcasting.
3. **The journal auto-repairs on crash.** Like SQLite WAL and RocksDB: the first invalid byte is the new end. You never see a "corrupt journal, can't start" failure.

→ Next: **Chapter 07 — Coding (erasure coding + ZODA)**. We've got a way to
store and ship data. But what if the data is a 10 MB block and shipping the
whole thing is too slow? That's what erasure coding fixes.