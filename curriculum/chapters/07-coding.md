# Chapter 07 — Coding: erasure coding, and the ZODA trick

> How to ship a 10 MB block without bottlenecking on the leader.

## The problem

Say you have a 100-validator network. The leader in view 5 wants to propose a
10 MB block. The naive approach: leader sends the full block to all 99 other
validators.

```
Leader egress: 99 × 10 MB = 990 MB outbound
```

Even on a 1 Gbps link (125 MB/s), that's ~8 seconds of full-bandwidth
shipping. During which, the leader is doing nothing else, and the validators
are waiting. Consensus halts.

Worse: most of those 990 MB are redundant. Every validator ends up with the
same block. We've burned bandwidth to copy the same bytes 99 times.

## The trick — erasure coding

**Split the data into `n` shards. Any `k` of them can reconstruct the original.**

```
Leader has 10 MB block B
  │
  ▼  encode into n shards
  │
  ├── shard 0 (1 MB)
  ├── shard 1 (1 MB)
  ├── ...
  └── shard n-1 (1 MB)
  │
  ▼
Send shard i to validator i. Total leader egress: 10 MB.
  │
  ▼
Validators exchange shards among themselves.
  │
  ▼
Once each validator has any k shards, reconstruct B.
```

Each validator still has 10 MB total data to deal with, but the leader's
egress dropped from 990 MB to 10 MB. **100x reduction.**

Now each validator's egress is `(k-1) × 1 MB` (sending their shard to the
other `k-1` validators they need shards from). If you do this in parallel
with a clever scheme, total bandwidth across the network is `n × 10 MB`
(the same as the data), distributed evenly. **Optimal.**

This is the theory. The practice is more subtle.

## The `Scheme` trait

Open `coding/src/lib.rs:160-218`:

```rust
pub trait Scheme: Debug + Clone + Send + Sync + 'static {
    type Commitment: Digest;
    type Shard: Clone + Debug + Eq + Codec<Cfg = CodecConfig> + Send + Sync + 'static;
    type CheckedShard: Clone + Send + Sync;
    type Error: std::fmt::Debug + Send;

    fn encode(
        config: &Config,
        data: impl Buf,
        strategy: &impl Strategy,
    ) -> Result<(Self::Commitment, Vec<Self::Shard>), Self::Error>;

    fn check(
        config: &Config,
        commitment: &Self::Commitment,
        index: u16,
        shard: &Self::Shard,
    ) -> Result<Self::CheckedShard, Self::Error>;

    fn decode<'a>(
        config: &Config,
        commitment: &Self::Commitment,
        shards: impl Iterator<Item = &'a Self::CheckedShard>,
        strategy: &impl Strategy,
    ) -> Result<Vec<u8>, Self::Error>;
}
```

The three operations:

1. **`encode(data)`** — split `data` into `n` shards + a `Commitment` (a hash
   that uniquely identifies the encoding).

2. **`check(commitment, index, shard)`** — verify a single shard is valid
   against the commitment. Returns a `CheckedShard` that's safe to feed to
   `decode`.

3. **`decode(commitment, checked_shards)`** — reconstruct the original data
   from any `k` checked shards.

The `Strategy` parameter is `commonware_parallel::Strategy` — choose
`Sequential` for single-threaded or `Rayon` for parallel encoding/decoding.

The `Config` (lines 26-44):

```rust
pub struct Config {
    pub minimum_shards: NonZeroU16,    // need this many to reconstruct (k)
    pub extra_shards: NonZeroU16,      // extra shards = n - k
}
```

So `n = minimum_shards + extra_shards`. Any `k = minimum_shards` shards
suffice.

## The "Check Agreement" bug

Now the subtle part. The trait's docs (`coding/src/lib.rs:131-150`) make this
explicit:

> **Check Agreement**: `Scheme::check` should agree across honest parties,
> even for malicious encoders.
>
> It should not be possible for parties A and B to both call `check`
> successfully on their own shards, but then have either of them fail when
> calling `check` on the other's shard.

In plain English: if my shard passes `check`, and your shard passes `check`,
then if I check **your** shard, that should also pass. And if we both decode
from different subsets of shards, we should get the **same data**.

**Why does this matter for consensus?**

If a malicious leader proposes a block and gives random garbage shards to
different validators, the validators can't tell — until they try to
reconstruct and get different bytes from different shard subsets. By then
it's too late; they've already voted to finalize the block hash.

Plain **Reed-Solomon** has this bug. If the encoder is malicious, different
decoders can get different results.

The fix: **ZODA** (Zero-Overhead Data Availability).

## ZODA — the novel solution

`coding/src/zoda/mod.rs:1-15`:

> What makes ZODA interesting is that upon receiving and checking one shard,
> you become convinced that there exists an original piece of data that will
> be reconstructable given enough shards. This fails in the case of, e.g.,
> plain Reed-Solomon coding.
>
> Ultimately, this stems from the fact that you can't know if your shard
> comes from a valid encoding of the data until you have enough shards to
> reconstruct the data. With ZODA, you know that the shard comes from a
> valid encoding as soon as you've checked it.

ZODA is built from Reed-Solomon + Hadamard checksums, structured so that
**checking a shard proves it belongs to a valid encoding** — no waiting
until you have k of them.

The high-level structure (from `zoda/mod.rs:77-115`):

1. **Arrange the data as a matrix** `X` of shape `n*S × c` (rows = shard
   groups, columns = data chunks).

2. **Reed-Solomon encode** to get `X'` of shape `pad((n+k)*S) × c`.

3. **Merkle-commit** the rows of `X'` to get a vector commitment `V`.

4. **Hash `V` + data size** → `Com` (the public commitment).

5. **Derive randomness** from `Com` (Fiat-Shamir) to generate a
   Hadamard-check matrix `H` of shape `c × S'`.

6. **Compute `Z = X @ H`** — these are the checksum columns.

7. **Shard `i`** contains: data size, vector commitment `V`, the slice
   `Z[i*S..(i+1)*S]`, and **the rows of `X'` for shard `i`** with
   inclusion proofs in `V`.

To check a shard:

1. Verify `Com = hash(V, size)`.
2. Re-derive `H` from `Com`.
3. Verify each row of the shard is in `V` (Merkle proof).
4. Verify the shard's data `X'[rows] @ H = Z[rows]` (the Hadamard check).

If all four pass, you **know** the shard is from a valid encoding. No need
for k shards to verify.

This is what `coding/src/lib.rs:479-484` flags:

```rust
/// A marker trait indicating that `Scheme::check` or `PhasedScheme::check`
/// proves validity of the encoding.
///
/// This means that upon a successful call to `check`, the shard results from
/// a valid encoding of the data, and thus, if other participants also call
/// check, then the data is guaranteed to be reconstructable.
pub trait ValidatingScheme {}
```

ZODA implements `ValidatingScheme`. Plain Reed-Solomon does not.

## The `PhasedScheme` — strong vs weak shards

For gossip-based block dissemination, you actually want **two kinds of
shards**:

- **Strong shard** — the original shard from the encoder. It has all the
  verification material (vector commitment, checksum data).
- **Weak shard** — a stripped-down shard that you forward to peers. It just
  has the rows and inclusion proofs. Smaller, faster to gossip, but only
  verifiable against your **checking data** (which you derived from your
  strong shard).

That's the `PhasedScheme` trait (`coding/src/lib.rs:287-382`):

```rust
pub trait PhasedScheme {
    type StrongShard: ...;
    type WeakShard: ...;
    type CheckingData: ...;

    fn encode(...) -> Result<(Commitment, Vec<StrongShard>), Error>;

    fn weaken(
        config, commitment, index, strong_shard,
    ) -> Result<(CheckingData, CheckedShard, WeakShard), Error>;

    fn check(
        config, commitment, checking_data, index, weak_shard,
    ) -> Result<CheckedShard, Error>;

    fn decode(config, commitment, checking_data, checked_shards, ...) -> Result<Vec<u8>, Error>;
}
```

The flow:

1. **Leader** encodes and sends **strong shards** to each validator.
2. Each validator calls `weaken` on their strong shard → gets:
   - `CheckingData` — used to validate weak shards they receive from peers.
   - `CheckedShard` — their verified contribution to reconstruction.
   - `WeakShard` — a smaller shard to forward to peers.
3. Validators gossip **weak shards** to each other.
4. Each validator `check`s received weak shards against their `CheckingData`.
5. Once you have `k` checked shards, `decode`.

This is the pattern from `docs/blogs/commonware-broadcast.html` — efficient
gossip with full verification, no bandwidth waste.

## Why this matters for chapter 01's `certify` hook

Now we can connect chapter 01's design to chapter 07's machinery.

In Simplex, after a block is **notarized** (quorum of votes), the engine
asks the application to **certify** it before finalizing. With erasure
coding, the application's certification logic becomes:

```rust
async fn certify(&self, block_hash: Hash) -> bool {
    // Have I received enough shards to reconstruct the block?
    let checked_shards: Vec<_> = self.shards_for(block_hash).iter()
        .filter_map(|s| Zoda::check(&self.cfg, &self.commitments[block_hash], s.index, &s.data).ok())
        .collect();

    if checked_shards.len() < self.cfg.minimum_shards {
        return false;  // can't reconstruct yet
    }

    // Try to reconstruct and verify
    let Ok(data) = Zoda::decode(&self.cfg, &self.commitments[block_hash],
                                &self.checking_data, checked_shards.iter(),
                                &self.strategy) else {
        return false;  // decode failed
    };

    // Did the hash match? Did the data pass app-level validation?
    hash(&data) == block_hash && self.app_verify(&data)
}
```

If `certify` returns `true`, the application has the data, has verified the
hash, and has done whatever app-specific checks (state transition
validation, signature checks, etc.). The voter broadcasts `finalize`.

If `certify` returns `false` (no shards yet, or verification failed), the
voter broadcasts `nullify` — skip this view, try again next leader.

This is the **decoupling in action**: consensus agrees on the hash. The
application handles the data. They sync up at the `certify` step.

## The Carnot Bound — the theoretical ceiling

`docs/blogs/carnot-bound.html` (March 2026) — Commonware published a paper
establishing a tight lower bound on coding efficiency for protocols with
fast finality. The result:

> The Carnot Bound establishes a tight lower bound on coding efficiency for
> protocols with fast finality, and shows that an additional round of voting
> breaks the barrier.

This is the theoretical limit on how efficient erasure coding can be for
block dissemination in BFT consensus. ZODA approaches the bound.

## Putting it together

```
                Leader's view 5
                propose block B
                       │
                       ▼
        ┌──────────────────────────────┐
        │  ZODA::encode(B) →           │
        │    commitment, n strong shards│
        └──────────────┬───────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   validator 1    validator 2    validator N
        │              │              │
        │  weaken()    │  weaken()    │  weaken()
        │              │              │
        ▼              ▼              ▼
   weak shards    weak shards    weak shards
        │              │              │
        └──────────────┴──────────────┘
                       │
                       ▼
              Gossip weak shards
                       │
                       ▼
        ┌──────────────┴──────────────┐
        ▼              ▼              ▼
   check weak      check weak      check weak
   against         against         against
   checking data   checking data   checking data
        │              │              │
        └──────────────┬──────────────┘
                       ▼
            Once k checked shards:
                       │
                       ▼
              ZODA::decode() → B
                       │
                       ▼
              certify(B) → hash matches → ready to finalize
```

## Where to look in the code

- `coding/src/lib.rs:160-218` — the `Scheme` trait.
- `coding/src/lib.rs:131-150` — the Check Agreement property.
- `coding/src/lib.rs:479-484` — `ValidatingScheme` marker.
- `coding/src/lib.rs:287-382` — the `PhasedScheme` trait (strong/weak shards).
- `coding/src/reed_solomon.rs` — the Reed-Solomon implementation.
- `coding/src/zoda/mod.rs` — the ZODA implementation (and the paper link).
- `coding/src/zoda/topology.rs` — the size calculations.
- `consensus/src/simplex/mod.rs:80-100` — the `CertifiableAutomaton` hook.
- `docs/blogs/carnot-bound.html` — the theoretical bound.

## Information theory primer — Shannon entropy and the ceiling

Erasure coding is a child of information theory. Two facts underpin
everything that follows.

### Shannon entropy

For a random variable `X` taking values in some alphabet, the **entropy**
`H(X)` measures the average information content per symbol:

```
H(X) = -Σ p(x) log_2 p(x)        (in bits)
```

A uniform random byte has entropy 8 bits (you're equally surprised by
every value). A biased coin (always heads) has entropy 0 bits.

The point: the entropy is the **minimum** number of bits needed to
encode the variable, on average. You can't compress below the entropy.

For erasure coding, the analogue: the **rate** `R = k/n` (message
symbols / encoded symbols) is at most 1. You can't get `R > 1` because
you can't encode more information than you started with.

### Why random codes are optimal asymptotically

Shannon's channel coding theorem (1948): for a channel with capacity
`C`, there exist codes of rate `R < C` that achieve arbitrarily low
error probability as block length grows.

The proof is **non-constructive**: it shows random codes achieve the
bound, but doesn't tell you how to find them. Reed-Solomon, BCH,
LDPC, Turbo codes, and Polar codes are all **constructive** families
that approach the bound for specific channels.

For erasure coding on a network with packet loss, the "channel" is
binary: either you get the packet or you don't. The capacity is
`1 - p_loss`. A code with rate `R < 1 - p_loss` can recover with high
probability.

In BFT block dissemination: we want recovery from any `k` of `n`
shards, so the code is MDS (Maximum Distance Separable) and the rate
is exactly `k/n`. There's no randomness in the channel; you choose
the erasures you can tolerate.

### The two key bounds for erasure coding

1. **Singleton bound.** Distance `d <= n - k + 1`. A code with `d = n - k + 1`
   is MDS and optimal. Reed-Solomon is MDS; so is ZODA's underlying code.

2. **Plotkin bound.** For codes with small alphabets, rate-distance
   trade-offs are constrained. Doesn't matter much for large-field
   codes (like RS over `F_{2^64}`).

These say "you can't do better than RS for the metrics you care about."
ZODA doesn't improve the rate; it improves the **checking semantics**.

## Reed-Solomon in depth — Vandermonde, Lagrange, Berlekamp-Massey

Reed-Solomon (RS) is the workhorse of erasure coding. Commonware
implements it in `coding/src/reed_solomon.rs`. Let me unpack the math
more deeply than the appendix does.

### The Vandermonde construction

Pick `k` distinct elements of `F_q`: `α_0, α_1, ..., α_{k-1}`. Form
the **Vandermonde matrix**:

```
V = [ 1        1        ...  1       ]
    [ α_0      α_1      ...  α_{k-1} ]
    [ α_0^2    α_1^2    ...  α_{k-1}^2 ]
    [ ...      ...      ...  ...     ]
    [ α_0^{k-1} α_1^{k-1} ... α_{k-1}^{k-1} ]
```

For an RS code with `n` symbols (where `n <= q`), you pick `n`
evaluation points `α_0, ..., α_{n-1}`. The encoded message is the
matrix-vector product `V * m`, where `m` is the message as a column
vector.

The Vandermonde matrix is invertible when the `α_i` are distinct (which
they are by construction). This gives a one-to-one mapping between
messages and encodings.

### The encoding via polynomial evaluation

Equivalently: view the message `m = (m_0, m_1, ..., m_{k-1})` as
coefficients of a polynomial:

```
f(x) = m_0 + m_1*x + m_2*x^2 + ... + m_{k-1}*x^{k-1}
```

The encoded symbols are `f(α_0), f(α_1), ..., f(α_{n-1})`.

Why this is equivalent to the Vandermonde multiplication: the matrix
product `V * m` is exactly `f(α_i)` for each `i`. The two formulations
are different ways of seeing the same linear operation.

### Decoding via Lagrange interpolation

Given `k` encoded symbols `(α_i, y_i)`, recover `f(x)` (and hence `m`):

```
f(x) = Σ y_i * L_i(x)

L_i(x) = Π_{j ≠ i} (x - α_j) / (α_i - α_j)
```

The Lagrange basis polynomials `L_i(x)` are 1 at `α_i` and 0 at every
other `α_j`. So `f(α_i) = y_i` by construction.

Once you have `f(x)`, read off the coefficients — they are the
original message `m`.

### Berlekamp-Massey decoding (for error correction)

Lagrange interpolation handles **erasures** (you know which symbols are
missing). For **errors** (symbols are corrupted but you don't know
which), you need error correction.

Berlekamp-Massey (and the related Berlekamp-Welch algorithm) find the
shortest LFSR that explains the received sequence, then use it to find
the error locator polynomial. With the error locations known, the
correct symbol values can be recovered.

For BFT block dissemination, we usually assume **erasures** (shards
either arrive or they don't; an arriving shard is assumed correct after
`check`). Errors (corrupted shards) are detected by the check and
treated as missing. So Lagrange interpolation is what we use.

### Why RS is MDS

MDS = Maximum Distance Separable. A code is MDS if `d = n - k + 1`
(the Singleton bound is tight).

For RS, the minimum distance is `n - k + 1`. Proof sketch: any
polynomial of degree `k - 1` is uniquely determined by `k` points.
Two distinct polynomials can agree on at most `k - 1` points. So they
can differ in at most `n - (k-1) = n - k + 1` positions. The minimum
weight of a nonzero codeword is `n - k + 1`.

This means: RS can correct `n - k` erasures (any subset of size `k`
reconstructs) OR `floor((n-k)/2)` errors (using the error correction
algorithm).

### Field choice in Commonware

Commonware uses **Goldilocks**, the prime field `F_p` with `p =
2^64 - 2^32 + 1 = 0xFFFFFFFF00000001`. Properties:

- **64-bit prime.** Arithmetic is fast on 64-bit CPUs (just multiply
  and reduce).
- **NTT-friendly.** The field has a primitive root of unity of order
  `2^32`, so the Number Theoretic Transform (NTT) — the fast
  polynomial-multiplication algorithm — works efficiently.
- **Carried at the math layer.** `commonware-math/src/fields/goldilocks.rs`
  implements the field; `coding/` uses it.

The 64-bit elements mean a 1 MB block is 131,072 field elements. With
`k = 67` (a typical value), each shard is `131072 / 67 ≈ 1957`
elements.

## The Check Agreement bug — formal definition

The `Scheme::check` trait method's contract (`coding/src/lib.rs:131-150`)
is subtle. Let me state it formally.

### The property

For any `Scheme` implementation, for any message `data`, the
`encode(data)` produces `(commitment, shards)` where `shards[i]` is
shard `i`. Honest parties call `check` and `decode` on these shards.

The **Check Agreement** property says:

```
For any honest party A and honest party B:
    if A holds shard i with check(pass)
       and B holds shard j with check(pass)
    then A's decode(shards_i_subset) = B's decode(shards_j_subset)
    AND A's check(j's shard) = pass
```

In plain English: if both shards pass `check`, then they're
**mutually consistent** — they came from the same encoding. Decoding
from either subset yields the same data.

### Why plain RS violates it

Consider a malicious encoder. Alice sends Bob shard 5 (random bytes).
Alice sends Charlie shard 7 (different random bytes). Both Bob and
Charlie call `check` on their own shards:

- `check(shard_5)` against Alice's identity: passes (any single shard
  trivially "checks" against itself).
- `check(shard_7)` against Alice's identity: passes.

But `check(shard_5)` from Charlie's perspective **fails** — there's
no polynomial consistent with both shard 5 and shard 7 that Charlie
knows about.

Or: Bob and Charlie try to reconstruct. Bob uses shards `{1, 5, ...}`
and gets polynomial `f_B(x)`. Charlie uses shards `{1, 7, ...}` and
gets polynomial `f_C(x) ≠ f_B(x)`. Decoding yields different
messages.

### Why this breaks consensus

In Simplex's `certify` hook (chapter 07 main text), each validator
certifies by reconstructing from the shards they have. If different
validators reconstruct to **different** messages (because the encoder
gave them different "valid" shards), then:

1. Validator X signs "block hash matches reconstructed data."
2. Validator Y signs "block hash matches reconstructed data" — but
   the data is different.

Both signatures are valid (each validator's reconstruction matches the
block hash **as they computed it**). The block hash is the same. But
the underlying data is different. Consensus has finalized on a hash
that maps to different bytes depending on who you ask.

This is catastrophic. It means a malicious encoder can split the
network's view of finalized blocks, even though every individual
validator sees a "valid" certificate.

### Why ZODA fixes it

ZODA's check is **not** "this shard is consistent with itself." It's
"this shard is consistent with a specific encoding (identified by the
commitment)." The Hadamard check ties every row of every shard to a
deterministic matrix, and that matrix is committed via the Merkle
vector commitment.

To forge a "valid" shard that doesn't match the commitment, an
attacker would need to find a polynomial consistent with both the
Hadamard check and a different matrix — a 126-bit-hard problem.

So `check(pass)` in ZODA means "this shard came from THE encoding
identified by `com`." Not "from some encoding." That's the property
Check Agreement requires.

## Hadamard codes and transforms — the matrix H_2^k

ZODA uses **Hadamard codes** as a checksum layer. The Hadamard code
is a Reed-Muller code of order 1 over the binary alphabet, and it has
a beautiful structure: the matrix `H_2^k` is a `2^k × 2^k` matrix of
`+1` and `-1` entries with the property `H * H^T = n * I`.

### The matrix `H_2^k`

```
H_1 = [1  1]                  (1x1 trivially)
     [1 -1]

H_2 = [1  1  1  1]
     [1 -1  1 -1]
     [1  1 -1 -1]
     [1 -1 -1  1]

H_4 = [H_2  H_2]    (Kronecker product)
     [H_2 -H_2]
```

The pattern: `H_{2^k} = H_2 ⊗ H_{2^{k-1}}`, where `⊗` is the Kronecker
product.

Properties:
- **Entries are ±1** (or 0 and 1, depending on convention).
- **Self-inverse** (up to scaling): `H * H = n * I`.
- **Orthogonal rows:** distinct rows have inner product 0.

### Hadamard transform

Computing `y = H * x` is the **Hadamard transform** of `x`. The naive
algorithm is `O(n^2)`; the **fast Hadamard transform** is `O(n log n)`
using the recursive structure.

For `n = 2^k`, the fast transform is exactly the FFT structure with
twiddle factors all equal to 1. It's faster than FFT because there's
no complex multiplication, just adds and subtracts.

### Why ZODA uses it

In ZODA, the Hadamard check is `Z = X @ H`, where `X` is the data
matrix and `H` is a Hadamard matrix. The result `Z` is a "checksum"
matrix that's easy to verify:

- If `X` is a valid RS-encoded matrix, `Z` is determined by `X` and `H`.
- If `X'` is the decoded data, `Z == X' @ H` (with high probability
  only if `X` was valid).

This is the property the `check` method verifies. For each row of the
shard, compute the Hadamard check; verify it matches the committed
checksum row.

### Over GF(p) vs binary

The Hadamard matrix has ±1 entries, which is binary-friendly. But
ZODA works over `F_p` (Goldilocks, a 64-bit prime field). So the
"±1" becomes "+1 and p-1" in `F_p`. The algebraic structure is the
same.

The "fast Hadamard transform" in `F_p` is also `O(n log n)` with
the same recursion. It's used during encoding to compute `Z` quickly.

## Fiat-Shamir transform — interactive to non-interactive proofs

ZODA's commitment-to-randomness step uses the **Fiat-Shamir transform**.
Let me explain what this is.

### The setting

You have an **interactive proof** between a prover and a verifier. The
prover sends commitments, the verifier sends random challenges, the
prover sends responses. Three rounds: commit, challenge, respond.

The soundness: if the prover can convince the verifier, they must
know the secret. Soundness error is `1/q` where `q` is the challenge
space size.

### The trick

What if the verifier's challenges are **deterministic** instead of
random? Specifically: the verifier sends `challenge = H(commitment ||
public_input)`, where `H` is a cryptographic hash.

This makes the proof **non-interactive**: the prover computes the
challenge themselves by hashing their own commitment. The verifier
checks the same hash on the receiving end.

Soundness: if the prover can produce a valid transcript without
knowing the secret, they must have found a hash collision — i.e.,
broken `H`. As long as `H` is collision-resistant, the non-interactive
proof has the same soundness as the interactive one.

### Why ZODA uses it

In ZODA, the encoder needs to derive a random Hadamard matrix `H` from
the public commitment `Com`. Without Fiat-Shamir, the encoder would
have to send `H` in the clear, and the verifier would have to trust
it was actually random. With Fiat-Shamir, `H = H_func(Com)` is
**determined by the commitment**. An encoder who picks a "bad" `H`
is committing to it via the public `Com`.

In effect: Fiat-Shamir turns "I chose this random matrix" into "I
committed to this random matrix before I knew my data." That's the
soundness.

### Implementation in Commonware

The Fiat-Shamir step is in the `zoda/mod.rs` and `topology.rs` files.
The `Com` is hashed (via SHA-3 or BLAKE3) and the output is expanded
into field elements that form the rows of `H`. The hash function's
collision resistance is the security foundation.

The 126-bit security level in `topology.rs:7` comes from the field size
(64 bits per element) and the number of challenge samples. Together
they give `2^126` soundness.

## The ZODA paper — Dziembowski et al.

ZODA is published as `eprint.iacr.org/2025/034` (cited in
`coding/src/zoda/mod.rs`). The authors are Stefan Dziembowski and
collaborators. The construction is called "Zero-Overhead Data
Availability" and is intended to provide data availability sampling
(DAS) without the overhead of standard approaches.

### The setup

A block producer has `B` bytes of data. They want to commit to `B`
(via a small commitment) and produce `n` shards such that:
1. Any `k` shards can reconstruct `B`.
2. Checking a single shard proves it came from a valid encoding of `B`
   (no "check agreement" bug).

The "zero overhead" claim: the commitment is the same size as a
standard Merkle root (32 bytes for BLAKE3). The shards are slightly
larger than RS shards (because of the Hadamard checksum columns), but
the total encoding overhead is comparable.

### The security theorem

The paper proves:

> For any PPT adversary that produces `(Com, V, H)` such that at least
> one of `n` shards passes the verifier's check, the data can be
> reconstructed with all but `2^(-126)` probability.

(The exact statement is in the paper; this is the gist.)

The theorem assumes:
- The adversary is computationally bounded (PPT = polynomial-time
  Turing machine).
- The hash function is collision-resistant.
- The Hadamard transform over `F_p` is a good randomness extractor.

Under these assumptions, the construction is sound.

### The `ValidatingScheme` marker

Commonware's `ValidatingScheme` trait (`coding/src/lib.rs:479-484`)
marks schemes that satisfy the property:

```rust
pub trait ValidatingScheme {}

impl ValidatingScheme for Zoda {}
```

Plain `ReedSolomon` does NOT implement this trait. That's the type-
level distinction between "can be validated shard-by-shard" (ZODA)
and "needs all k shards to verify" (RS).

## Network coding — the butterfly and the field context

Erasure coding is a special case of **network coding**, where
intermediate nodes in a network are allowed to compute on packets,
not just forward them. ZODA is somewhere between erasure coding
and network coding; let me explain the lineage.

### The butterfly network

A classic example. Source S wants to send `a` and `b` to both
receivers R1 and R2. The bottleneck is the link from layer 2 to
layer 3 (one packet at a time).

```
        a
        |
        v
        X -----> R1 (receives a and a+b, computes b = (a+b) - a)
       / \
      /   \
     X     X -- a+b --> R2 (receives b and a+b, computes a = (a+b) - b)
    / \   / \
   a   b X   X
       ...
```

The intermediate node computes `a+b` and sends it on the bottleneck
link. R1 already has `a`, subtracts, gets `b`. R2 already has `b`,
subtracts, gets `a`.

This is **impossible with plain routing** (the bottleneck can't carry
both `a` and `b`). Network coding achieves the multicast capacity.

### Random linear network coding

In a general network with many sources and sinks, random linear codes
work: every intermediate node picks random coefficients over a field,
and emits `Σ c_i * packet_i`. Receivers collect enough linear
combinations to invert.

For BFT block dissemination, the network topology is simpler (we have
a star: leader broadcasts, validators exchange). Erasure coding
specializes to this case. But the lineage is the same.

### Why broadcast is harder than point-to-point

Point-to-point communication needs `k` correct packets for a `k`-packet
message. That's it.

Broadcast needs `k` correct packets **seen by every recipient**. If
different recipients see different subsets, they reconstruct
different messages. The check agreement property addresses exactly
this issue.

In the network-coding world, this is the **byzantine broadcast**
problem. ZODA's contribution is making byzantine broadcast efficient
enough to use in BFT consensus.

## Phased scheme semantics — strong vs weak shards in depth

The `PhasedScheme` trait (`coding/src/lib.rs:287-382`) is what makes
gossip-based dissemination work. Let me explain the semantics in
depth.

### Why two shard types

The leader produces **strong shards** that contain everything needed
to verify the encoding. These are large (~2x the original data with
checksums).

When validators gossip shards to each other, they want to forward the
**minimum** needed for a peer to contribute to reconstruction. They
don't want to forward the full strong shard — that's redundant (every
strong shard has the same `Com` and `V`).

So a peer **weakens** their strong shard into a weak shard:
- Remove `Com` and `V` (the receiver has these).
- Remove the full checksum slice (the receiver has their own).
- Keep just the rows + Merkle proofs.

The weak shard is much smaller and can be verified against the
receiver's own `CheckingData` (derived from their strong shard).

### The `weaken` operation

```rust
fn weaken(
    config: &Config,
    commitment: &Self::Commitment,
    index: u16,
    strong_shard: &Self::StrongShard,
) -> Result<(CheckingData, CheckedShard, WeakShard), Error>;
```

Inputs:
- The encoding parameters.
- The public commitment.
- This peer's shard index.
- This peer's strong shard.

Outputs:
- `CheckingData` — the local state needed to verify weak shards from
  peers. Includes the row commitment and (typically) some derived
  randomness.
- `CheckedShard` — this peer's verified contribution to reconstruction.
- `WeakShard` — what to forward.

The `CheckingData` is shared across all weak shards this peer
receives. It's "the peer's view of the encoding."

### The `check` operation

```rust
fn check(
    config: &Config,
    commitment: &Self::Commitment,
    checking_data: &Self::CheckingData,
    index: u16,
    weak_shard: &Self::WeakShard,
) -> Result<CheckedShard, Error>;
```

Inputs:
- The encoding parameters.
- The public commitment.
- The local `CheckingData`.
- The peer's index and weak shard.

Output: the peer's checked shard (or an error).

The check verifies:
1. The weak shard's rows match the rows committed in `V`.
2. The Hadamard check passes for the rows.
3. The weak shard matches the expected index.

If all pass, the peer contributes their rows to the reconstruction.

### The security model

The phased scheme assumes:

- **The encoder is honest** (or detected via check failure).
- **The leader's strong shard is the source of truth.**
- **Peers can independently verify weak shards** using their own
  CheckingData.

The model is "trust the leader's first shard, then trust no one." Any
peer can independently verify any other peer's weak shard using only
the public commitment and their own CheckingData.

### When the encoder is malicious

If the leader's strong shard doesn't check (i.e., `Zoda::check`
fails), the validator **doesn't sign**. This is the failure mode:
the protocol halts because no one can certify the block.

The recovery: the leader is replaced (nullify), and a new leader
proposes the same block (or a different one). If the malicious
leader is the root cause, eventually they're slashed.

For BFT with `2f + 1` honest validators, this works: the malicious
leader can't fool everyone.

## Strategy types in detail — Sequential, Parallel, Threshold

The `Strategy` enum from `commonware-parallel` (chapter 07 Appendix D)
gives three execution modes for encoding/decoding. Let me compare them
in detail.

### Sequential

Single-threaded. The encoder loops over shards:

```rust
let shards: Vec<Shard> = data
    .chunks(rows_per_shard)
    .enumerate()
    .map(|(i, chunk)| encode_shard(config, chunk, i))
    .collect::<Result<_, _>>()?;
```

Properties:
- **Deterministic.** Same input, same output, same timing.
- **Slow.** One core; no parallelism.
- **Low memory.** No thread pool overhead.

For tests, this is the right choice — the test expects a single,
deterministic execution.

### Parallel (Rayon)

Multi-threaded via rayon. Each shard encoded on a separate thread:

```rust
let shards: Vec<Shard> = data
    .chunks(rows_per_shard)
    .enumerate()
    .par_bridge()  // rayon: parallel iterator
    .map(|(i, chunk)| encode_shard(config, chunk, i))
    .collect::<Result<_, _>>()?;
```

Properties:
- **Fast.** Uses all available cores.
- **Deterministic *if* rayon is configured for it.** Commonware uses
  a `RayonPool` that's set up for deterministic work distribution.
- **Higher memory.** Each thread has its own stack and local data.

For production encoding/decoding of large blocks, this is the right
choice.

### Threshold

A hybrid: encode sequentially, then decode in parallel once `k` shards
are available. Or: a more sophisticated scheme that partitions the
work.

Commonware doesn't ship a `Threshold` strategy by name; it's a
conceptual pattern. You can build it by checking shard availability
and dispatching to parallel decode when ready.

### When to use each

| Workload | Strategy |
|---|---|
| Test code with small blocks | Sequential |
| Production encoding 10 MB+ blocks | Parallel |
| Production decoding under load | Parallel (with thread pool sizing) |
| Real-time / latency-critical | Sequential (predictable timing) |
| Memory-constrained | Sequential (lower peak memory) |

The `Strategy` is passed as `&impl Strategy` to `Scheme::encode` and
`Scheme::decode`, so the caller chooses per call site.

## Rust pattern — `const generics` and `typenum`

The `coding` crate uses Rust's type-level arithmetic for compile-time
size calculations. Let me explain the patterns.

### `const generics` for compile-time array sizes

Since Rust 1.51, you can write:

```rust
fn encode<const N: usize>(data: [u8; N]) -> [u8; N * 2] { ... }
```

The `N` is a compile-time constant. The compiler can specialize the
function for each `N`. For ZODA's row-size calculations, you might
have:

```rust
struct Shard<const ROWS: usize, const COLS: usize> {
    rows: [[F; COLS]; ROWS],
}
```

This guarantees at compile time that all rows have the same length.
No runtime bounds checks. No allocation.

### `typenum` for type-level arithmetic

`const generics` give you constants but not arithmetic on them. For
"the size of shard i depends on k and n," you need to compute at
compile time.

`typenum` provides **type-level numbers** and arithmetic:

```rust
use typenum::{U2, U3, U6, Prod, Add, Sub};

type Small = U2;
type Large = U3;
type Combined = Prod<Small, Large>;  // = U6
```

`Prod<A, B>` is the type-level product. The compiler resolves this
to a concrete number at compile time.

You can write:

```rust
fn combine<A: typenum::Unsigned, B: typenum::Unsigned>(
) -> [u8; <Prod<A, B> as typenum::Unsigned>::USIZE] {
    ...
}
```

The `USIZE` associated constant extracts the numeric value.

### Trade-offs

`const generics` vs `typenum`:
- `const generics` is built-in, simpler, more limited.
- `typenum` is a crate, more expressive, less ergonomic.

For Commonware, the choice depends on the complexity of the
arithmetic. Simple size calculations use `const generics`. More
complex (nested product types) might use `typenum`.

### Other patterns in `coding/`

A few more worth flagging:

- **Trait objects vs generics.** `Scheme` is generic (`T: Scheme`),
  not boxed (`Box<dyn Scheme>`). This is static dispatch, which is
  faster but requires monomorphization at every call site.

- **`PhantomData` for type-level state.** Some types carry type-level
  parameters (e.g., `Scheme<Config>`) where the parameter doesn't
  appear in fields. `PhantomData` keeps the type system happy.

- **Builder pattern for configuration.** The `Config` struct uses
  derive builders for ergonomic construction:

  ```rust
  let cfg = Config::builder()
      .minimum_shards(NZU16!(67))
      .extra_shards(NZU16!(33))
      .build();
  ```

- **Error types with `thiserror`.** Consistent with the rest of
  Commonware. `Error` is an enum with descriptive variants.

### Compile-time checks for encoding correctness

ZODA has many size invariants that should hold. With `const generics`,
these become compile-time checks:

```rust
fn encode<const DATA_BYTES: usize, const K: usize, const N: usize>(
    data: [u8; DATA_BYTES],
) -> Result<[Shard; N], Error>
where
    // Compile-time check: N >= K, K >= 1
    [(); N - K]: ,
    [(); K - 1]: ,
{
    ...
}
```

The `[(); N - K]: ` syntax asserts that the difference is non-negative
at compile time. The compiler rejects code where `N < K`. You can't
even typecheck a configuration that violates the threshold.

This is the kind of pattern that makes Rust's type system a powerful
tool for protocol implementations: you encode the protocol's
invariants in types, and the compiler enforces them.

## The Vandermonde matrix, formally — the deep construction

The Reed-Solomon "in depth" section above showed the polynomial-evaluation
view. Let's go deeper into the **matrix-algebra view**, because it explains
properties that the polynomial view alone doesn't surface: why a single
field element matters, why `n ≤ q` is a hard cap, and why some encoders
ship a **systematic** form (data + parity) while others ship an
**unsystematic** form (only encoded symbols).

### The Vandermonde matrix in full

Pick `n` distinct field elements `α_0, α_1, ..., α_{n-1} ∈ F_q` with
`n ≤ q` and `n > k - 1`. The **Vandermonde matrix** is:

```
        | 1          1          ...  1            |
        | α_0        α_1        ...  α_{n-1}      |
V   =   | α_0^2      α_1^2      ...  α_{n-1}^2    |
        | ...        ...        ...  ...          |
        | α_0^{k-1}  α_1^{k-1}  ...  α_{n-1}^{k-1} |
```

It's a `k × n` matrix: `k` rows (one per message symbol, or one per
polynomial coefficient), `n` columns (one per encoded symbol).

Three properties make this matrix special:

1. **Any `k × k` submatrix is invertible.** Take any `k` columns. They
   form a `k × k` Vandermonde; if the `α_i` are distinct, the determinant
   is `Π_{i < j} (α_j - α_i) ≠ 0` in `F_q`. So every `k`-column subset
   is full rank.

2. **The encoding is `c = V · m`.** With `m ∈ F_q^k` the message as a
   column vector, `c ∈ F_q^n` is the codeword. The whole encoding
   operation is one matrix-vector multiply.

3. **Decoding is "pick any `k` equations, invert."** Given
   `c_{i_1}, ..., c_{i_k}` (any `k` of the `n` coordinates), pick the
   corresponding columns of `V`, invert to get `V_sub^{-1}`, multiply
   by the received codeword: `m = V_sub^{-1} · c_sub`.

That's it. No polynomial evaluation needed (although that's the same
operation, viewed differently).

### The systematic form

A common variant is the **systematic form**: the first `k` symbols of
the codeword are the message itself. To construct it, multiply by an
invertible `k × k` matrix that swaps `m` into the first `k` positions:

```
c_sys = (I_k | P) · m
```

where `I_k` is the identity and `P` is a `k × (n-k)` parity matrix. The
parity matrix is computed from the original Vandermonde:

```
P = (V_left)^{-1} · V_right
```

where `V_left` is the first `k` columns of `V` and `V_right` is the
last `n-k` columns. The result `P` is the **parity-check matrix** for
the systematic encoding.

For BFT block dissemination, **systematic** is sometimes preferable:
the leader can ship the block data as-is to one set of peers and the
parity to another, and `k` of the data symbols plus the parity already
reconstruct everything. Commonware's ZODA does not use the systematic
form because every shard needs the same Merkle-tree + Hadamard
overhead anyway — there's no bandwidth win.

### Why `n ≤ q`

The Vandermonde matrix requires `n` distinct field elements. If `q` is
the field size, we have at most `q` choices. So `n ≤ q` is hard.

Commonware uses Goldilocks (`q ≈ 2^64`), which is plenty — `n` of a
million is still feasible. But for tiny fields (like `GF(2^8)`, the
classic choice for QR codes and small files), `n ≤ 255`. If your
deployment needs more shards, you need a bigger field.

A common trick: **extend the field**. Use `GF(2^{16})` (n ≤ 65535) for
storage systems, or arithmetic codes for arbitrary `n` (e.g., Zoda's
NTT-friendly construction, where `n` can be any power of 2 up to
`2^32` in Goldilocks).

### Why RS is MDS — the proof in full

**Theorem.** For RS with parameters `(n, k)` over `F_q`, the minimum
distance is `d = n - k + 1`.

**Proof.** Let `c, c'` be two distinct codewords. Both come from
polynomials `f, f'` of degree ≤ `k - 1`. The difference `g = f - f'`
is a polynomial of degree ≤ `k - 1` with at most `k - 1` roots (a
non-zero polynomial over a field has at most `deg(g)` roots).

The Hamming distance between `c` and `c'` equals the number of
positions where they differ:

```
d(c, c') = |{i : f(α_i) ≠ f'(α_i)}|
        = |{i : g(α_i) ≠ 0}|
        ≥ n - (k - 1)
        = n - k + 1
```

The lower bound is tight (achieved when `g` has exactly `k - 1` roots,
all at distinct `α_i`). So `d_min = n - k + 1`, achieving the **Singleton
bound**. ∎

This is the deepest reason RS is the gold standard: it achieves the
Singleton bound with equality. No other `[n, k, d]` code can do better
for the same `n, k`.

## Lagrange interpolation, with code

The "in depth" section sketched Lagrange interpolation. Here's the
algorithm in Rust, with a concrete `GF(2^8)` implementation. The goal
is to show you exactly what the math becomes when you implement it.

### GF(2^8) arithmetic

`GF(2^8)` is the field with 256 elements, polynomial arithmetic mod
an irreducible polynomial like `x^8 + x^4 + x^3 + x + 1` (0x11B, the
AES choice). Addition is XOR. Multiplication is polynomial
multiplication mod 0x11B.

```rust
/// An element of GF(2^8) with the AES irreducible polynomial.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
struct Gf2p8(u8);

impl Gf2p8 {
    const IRRED: u16 = 0x11B;

    fn mul(self, other: Self) -> Self {
        let mut a = self.0 as u16;
        let mut b = other.0 as u16;
        let mut r = 0u16;
        for _ in 0..8 {
            if b & 1 != 0 { r ^= a; }
            let carry = a & 0x80;
            a <<= 1;
            if carry != 0 { a ^= Self::IRRED & 0xFF; }
            b >>= 1;
        }
        Self(r as u8)
    }

    fn inv(self) -> Self {
        // Fermat's little theorem: a^(q-2) = a^(-1) in GF(q)
        let mut r = Self(1);
        for _ in 0..254 {  // q - 2 = 254
            r = r.mul(self);
        }
        r
    }
}

impl std::ops::Add for Gf2p8 {
    type Output = Self;
    fn add(self, other: Self) -> Self { Self(self.0 ^ other.0) }
}

impl std::ops::Sub for Gf2p8 {
    type Output = Self;
    fn sub(self, other: Self) -> Self { Self(self.0 ^ other.0) }
    // char 2: -x = +x
}

impl std::ops::Mul for Gf2p8 {
    type Output = Self;
    fn mul(self, other: Self) -> Self { Self::mul(self, other) }
}
```

The `inv` uses Fermat's little theorem: in a finite field, `a^(q-1) = 1`
for non-zero `a`, so `a^(q-2) = a^(-1)`. The cost: 254 multiplications.
Faster methods (extended Euclidean) exist but are messier to code.

### Lagrange interpolation in GF(2^8)

```rust
/// Recover the polynomial coefficients from k samples.
/// `points` is `[(α_i, f(α_i))]` for k distinct α_i.
fn lagrange_interpolate(
    points: &[(Gf2p8, Gf2p8)],
) -> Vec<Gf2p8> {
    let k = points.len();
    debug_assert!(k >= 1);

    // Recover f(x) evaluated at 0, 1, ..., k-1.
    // (We evaluate at consecutive integers for simplicity.)
    let xs: Vec<Gf2p8> = (0..k).map(|i| Gf2p8(i as u8)).collect();

    let mut coeffs = vec![Gf2p8(0); k];
    for i in 0..k {
        // Compute the Lagrange basis L_i(x) evaluated at x = xs[j] for all j.
        // L_i(x) = Π_{m ≠ i} (x - α_m) / (α_i - α_m)
        let (alpha_i, y_i) = points[i];
        let mut basis_at = [Gf2p8(0); 16];  // assume k ≤ 16
        for j in 0..k {
            let mut num = Gf2p8(1);
            let mut den = Gf2p8(1);
            for m in 0..k {
                if m == i { continue; }
                let (alpha_m, _) = points[m];
                num = num * (xs[j] - alpha_m);
                den = den * (alpha_i - alpha_m);
            }
            basis_at[j] = num * den.inv();
        }

        for j in 0..k {
            coeffs[j] = coeffs[j] + y_i * basis_at[j];
        }
    }
    coeffs
}
```

The cost is `O(k^2)` field multiplications for each basis polynomial
evaluation, and there are `k` of them: total `O(k^3)` field operations.
For `k = 67` (Commonware's typical value), that's `300,763`
multiplications. At ~10 ns each (with Goldilocks), that's 3 ms. Fast
enough.

### The NTT-based alternative

`O(k^3)` is fine for `k ≤ 100`. For larger `k`, use the **Number
Theoretic Transform (NTT)** — the finite-field analog of the FFT:

- **Forward NTT** on the message `m`: `O(k log k)`.
- **Element-wise multiply** by Vandermonde factors: `O(k)`.
- **Inverse NTT**: `O(k log k)`.
- Total: `O(k log k)`.

This is what ZODA uses in `coding/src/reed_solomon.rs`. The catch: NTT
requires `q` to have a primitive root of unity of order `n`. Goldilocks
has `2^32`-th roots of unity, so `n` can be any power of two up to
`2^32`. More than enough.

The trade-off: NTT code is harder to read. Lagrange is `O(k^3)` but
30 lines of math; NTT is `O(k log k)` but 300 lines of careful
index-bookkeeping. For `k ≤ 100`, Lagrange wins on code clarity.

### Why Commonware uses NTT for ZODA anyway

Three reasons:

1. **The total data is large.** With a 10 MB block and `n = 100`, each
   shard is ~100 KB. The polynomial degree `k - 1 ≈ 100`. So `k` is
   small, but the field elements per shard is large (`131,072` 64-bit
   field elements per MB). You can't naively do `O(N × k^3)` per shard.

2. **Encoding is per-column, not per-row.** The matrix `X` is
   `n*S × c`. Each *column* is an independent polynomial of degree
   `n - 1` (after Reed-Solomon extension). So you do `c` NTTs of size
   `n + k` each. Total work: `O(c × (n+k) × log(n+k))`.

3. **Test reproducibility.** The NTT over Goldilocks is deterministic
   for a given input; that's exactly what the deterministic executor
   (chapter 02) needs for replay tests.

## Berlekamp-Massey error correction, in detail

Lagrange handles **erasures**: you know which symbols are missing. For
**errors** (corrupted symbols, but you don't know which), you need
error correction. The classic algorithm is **Berlekamp-Massey**.

### Why BFT usually doesn't need this

In Commonware's BFT model, the encoding layer produces shards that are
either valid (the bytes pass `Scheme::check`) or invalid (the bytes
fail `check`). Invalid shards are treated as erasures, not errors.

The `Scheme::check` step on ZODA is a polynomial-time test against
the commitment. A corrupted shard fails the test and is dropped. The
decoder sees a smaller set of "valid shards" and reconstructs.

So BFT consensus does erasure decoding (Lagrange), not error decoding
(Berlekamp-Massey).

But the BCH codes in chapter 03 and the cryptographic accumulators
in chapter 04 *do* use error correction. And you should know the
algorithm if you ever design a network with adversarial corruption
that's not detected at the encoding layer.

### The LFSR view

The key observation: a polynomial of degree `d` corresponds to a
linear feedback shift register (LFSR) of length `d` that produces the
sequence `f(α_0), f(α_1), f(α_2), ...` as its output stream.

A **noisy** received sequence comes from an LFSR plus occasional
errors. If we can find the shortest LFSR that produces the
(error-free) prefix of the received sequence, we have the polynomial,
and from there the original message.

### Berlekamp-Massey in 30 lines

```python
def berlekamp_massey(s):
    """Find the shortest LFSR that produces s[:n]."""
    n = len(s)
    C = [0] * (n + 1)  # connection polynomial
    B = [0] * (n + 1)  # previous C
    C[0] = 1
    B[0] = 1
    L = 0              # current LFSR length
    m = 1              # shift counter
    b = 1              # previous discrepancy

    for n_idx in range(n):
        # Compute discrepancy
        d = s[n_idx]
        for i in range(1, L + 1):
            d ^= C[i] & s[n_idx - i]

        if d == 0:
            m += 1
        elif 2 * L <= n_idx:
            T = C[:]
            # C(x) <- C(x) - (d/b) * x^m * B(x)
            for i in range(n + 1 - m):
                C[i + m] ^= (d * inv(b)) & B[i]
            L = n_idx + 1 - L
            B = T
            b = d
            m = 1
        else:
            for i in range(n + 1 - m):
                C[i + m] ^= (d * inv(b)) & B[i]
            m += 1

    return C[:L + 1]
```

The output is the connection polynomial `C(x)` of length `L`. The
roots of `C(x)` in the field are the error locations. (Over `GF(q)`,
this means computing the roots explicitly — typically done by
Chien search.)

### Berlekamp-Welch

For RS decoding with errors *and* erasures, **Berlekamp-Welch** is the
practical algorithm. It solves a system of equations for the polynomial
`f(x)` and the error locator `e(x)`. Cost: `O(k^3)` field operations.

Commonware doesn't ship Berlekamp-Welch because ZODA's check step
converts errors to erasures. But if you ever write a system that needs
error-correcting codes without a pre-shard check, this is the algorithm
to reach for.

## The Check Agreement bug — three worked attacks

The "Check Agreement" section above stated the property formally. Let
me show you **three concrete attack scenarios** on a protocol that
naively uses plain Reed-Solomon without ZODA's per-shard check.

### Setup: a toy BFT protocol

```
1. Leader produces block B (10 MB).
2. Leader encodes B with RS(k=67, n=100).
3. Leader sends shard i to validator i.
4. Each validator calls check(shard_i) — passes (a single shard
   trivially passes its own identity check).
5. Validators gossip shards.
6. Once a validator has 67 checked shards, it reconstructs B.
7. Each validator votes on hash(B).
```

The attacker's goal: cause different validators to reconstruct
**different** bytes, even though all of them voted `notarize` on the
**same** hash.

### Attack 1 — the "shuffle" attack

The malicious leader knows 67 honest validators will each receive one
distinct shard. The leader does:

```
Send validator 1: shard 0
Send validator 2: shard 0  (duplicate, not shard 1!)
Send validator 3: shard 1
Send validator 4: shard 1
...
```

Wait, but each shard has a distinct identity. So the duplicates are
detectable — validator 2 sees "this is shard 0, not shard 1." Hmm.

OK, the leader instead sends **valid-looking** but **arbitrary** bytes
to half the validators, and the **actual** encoded shards to the other
half. From the leader's perspective:

```
For i in 0..50:
    send validator i: random bytes (not from any real RS encoding)
For i in 50..100:
    send validator i: actual shard (i - 50)  (or some mapping)
```

Validator 1 (in the random-bytes group) calls `check(shard_0)`:
passes (because plain RS's check on a single shard is "this shard is
self-consistent," which is always true).

Validator 51 (in the real-shard group) calls `check(shard_0)`: passes
(because shard 0 is a real encoded shard).

Now both validators have "passed" their check, but they have
**different shards**. The shards aren't actually from the same
encoding, but neither validator can tell.

When validators 1 and 51 reconstruct:
- Validator 1 picks 67 random shards → decoder runs Lagrange → some
  polynomial f_1 → some message m_1 → hash(m_1).
- Validator 51 picks 67 real shards → decoder runs Lagrange → the
  actual polynomial f_actual → the actual message m_actual → hash.

Both hashes are deterministic from their own reconstruction. Both
validators sign `notarize(hash, view)`. As long as the hash is the
same... wait, but the hash is `hash(B)`, where B is the original
block. And `hash(B)` is committed to by the leader's proposal, not
recomputed by the validators.

So validators sign `notarize(hash(B), view)` regardless of what they
reconstruct. The hash is `B`'s hash, even if their reconstruction is
garbage. They never check `hash(m_actual) == hash(B)`.

**Wait, that's the bug.** The honest validators trust that the leader
encoded the real B. They sign based on the hash commitment. The
attack is silent: the malicious leader gives garbage shards to most
validators, real shards to a few, and the few who have real shards
won't realize the others have garbage.

In Simplex's `certify` hook (chapter 01), the validator reconstructs
and **checks** `hash(reconstructed_data) == hash(B)`. That's the
defense. Without that check, the attack is invisible.

But what if half the validators succeed and half fail the certify
check?

- The successful ones vote `finalize`.
- The failed ones vote `nullify` (per the certify contract).
- Consensus halts: no `2f+1` finalization, no `2f+1` nullification.

That's the **better** outcome (consensus halts, attack fails). The
worse outcome is: a buggy protocol that **trusts the shard check on
its own** without reconstructing and verifying. That protocol gets
attacked silently.

### Attack 2 — the equivocation attack

The malicious leader proposes block B with hash H. After receiving
votes, the leader releases **two different decodings** of the same
shard set: one decodes to message m_1, another to m_2. (This is
mathematically impossible for *honest* RS — Lagrange is deterministic
— but a buggy decoder can be coerced.)

For honest validators, this attack only works if the decoder has a
non-deterministic bug. The defense: use a deterministic decoder
(Commonware does this; chapter 02's deterministic executor enforces
it).

For malicious validators, this attack is just Byzantine behavior and
is handled by the standard `2f+1` honest majority assumption.

### Attack 3 — the withholding attack

A subset of the validators colludes. They receive real shards but
refuse to forward them to others. The remaining honest validators
reconstruct from garbage shards (because the colluders are the only
ones with real shards, and they're withholding).

Result: a minority of validators reconstruct correctly, a majority
fail. The leader's proposal gets `2f+1` nullifications, not
finalizations. The view transitions.

The attack **fails to split the network's view** but **does delay
consensus by one view transition**. The recovery: skip-timeout (chapter
11), or just wait for view `v+1` to start fresh.

### Why ZODA defeats all three

ZODA's `check` is per-shard and **commits to the encoding**. To forge
a shard that passes ZODA's check but isn't from the real encoding,
an attacker needs to find a polynomial `f` such that:

1. `f`'s Merkle root is the committed `V`.
2. `f` passes the Hadamard check against the committed `Z`.

This requires either:
- Finding a hash collision in BLAKE3 (computationally infeasible), or
- Finding a polynomial that satisfies the check constraints without
  being derived from the committed data (also computationally
  infeasible, by the soundness theorem in `eprint.iacr.org/2025/034`).

So ZODA's `check` gives a **per-shard, deterministic** proof of
encoding validity. Plain RS's `check` gives only "this shard is
self-consistent," which is trivial to satisfy with random bytes.

## Fast Walsh-Hadamard Transform, with code

The Hadamard matrix `H` is `H · H^T = n · I`. Computing `y = H · x`
directly is `O(n^2)`. The **Fast Walsh-Hadamard Transform (FWHT)**
is `O(n log n)` and in-place. ZODA uses the FWHT to compute
`Z = X @ H` quickly.

### The recursion

The Kronecker product structure of `H_{2^k}` gives a divide-and-
conquer algorithm:

```
FWHT(x):
    n = len(x)
    if n == 1: return x
    even = [x[0], x[2], x[4], ...]
    odd  = [x[1], x[3], x[5], ...]
    even = FWHT(even)
    odd  = FWHT(odd)
    for i in 0..n/2:
        x[i]       = even[i] + odd[i]
        x[i + n/2] = even[i] - odd[i]
    return x
```

Each recursion halves the problem. `T(n) = 2 · T(n/2) + O(n) = O(n log n)`.

### Rust implementation

```rust
fn fwht<F: Field>(data: &mut [F]) {
    let n = data.len();
    if n <= 1 { return; }
    debug_assert!(n.is_power_of_two());

    let mut len = 1;
    while len < n {
        for start in (0..n).step_by(2 * len) {
            for i in 0..len {
                let u = data[start + i];
                let v = data[start + len + i];
                data[start + i] = u + v;
                data[start + len + i] = u - v;
            }
        }
        len *= 2;
    }
}
```

This is the **iterative** form — easier to read than the recursive
version, no auxiliary buffer.

For ZODA's `Z = X @ H`:
- `X` is `n*S × c` (a matrix of field elements).
- `H` is `c × S'` (a Hadamard matrix).
- Each row of `X` is an independent vector; FWHT it to get the
  corresponding row of `Z`.
- Total: `n*S` FWHTs, each `O(c log c)`, so `O(n*S*c*log c)` work.

For a 10 MB block with `n = 67, c = 2^16, S = 2^9`: that's
`67 × 2^9 × 2^16 × 16 = 2^36` operations. About 100 billion. At 10
ns per field op, ~1000 seconds. **Yikes**, that's a lot.

Wait, that's not right. Let me redo it.

Actually `S` is the samples per shard (rows of X' per shard). `c` is
columns of X. Each row of X' is `c` field elements. Each FWHT operates
on a row of length `c`. There are `n*S` rows total.

`n*S = 67 * 2^9 = 34304`. Each FWHT is `O(c log c) = O(2^16 * 16)`.

Total ops: `34304 × 2^16 × 16 ≈ 3.6 × 10^{10}`. At 10 ns per op, 360
seconds. Still slow. But ZODA optimizes by:
- Doing the FWHT **once**, not per-shard.
- Caching the result.
- Parallelizing across rows.

In practice, with a 16-core machine and parallelism: `360 / 16 ≈ 22
seconds` per encode. For a 10 MB block, that's about 0.5 MB/s. Slow
for production but acceptable for a 1-second-per-block consensus.

For higher throughput, you'd reduce `c` (fewer columns, larger
shards) or use a smaller field (binary Hadamard) or hardware
acceleration (SIMD or GPU).

### In-place vs out-of-place

The code above is in-place: `O(1)` auxiliary memory. The recursive
form allocates at every level. The iterative form is better for cache
locality.

### Connection to FFT

The FFT has the same recursive structure but with twiddle factors
(roots of unity). The FWHT has all twiddle factors = 1 (or = -1 for
the second half). So the FWHT is **simpler** than FFT — no complex
multiplication, just additions and subtractions.

For `n = 2^k`, the FWHT and the radix-2 FFT are essentially the same
algorithm with different constants. The FWHT is faster because each
"twiddle multiplication" is a no-op (multiply by 1) or a negation
(multiply by -1).

### When you'd use FWHT outside ZODA

- **Signal processing.** Walsh functions are a complete basis for
  binary signals. Audio codecs (MP3, AAC) use them implicitly via DCT.
- **Cryptography.** The original GGM construction (Goldreich-Goldwasser-
  Halevi) uses FWHT-style trees for pseudorandom functions.
- **Compressed sensing.** Some sparse-signal-recovery algorithms use
  FWHT for sub-linear-time matrix multiplication.
- **Coding theory.** MacWilliams identities use FWHT to compute weight
  distributions of code duals.

## Fountain codes: LT and Raptor

We've covered Reed-Solomon (block codes) and ZODA (RS + checksums).
What about **fountain codes** — codes that can produce an unbounded
number of encoded symbols?

### The motivation

For one-to-many distribution (like BitTorrent or CDN), the sender
doesn't know in advance how many receivers there are or how lossy
their connections are. RS requires you to commit to `n` (the total
number of encoded symbols) at encoding time.

A **fountain code** produces an **unbounded stream** of encoded
symbols. The receiver stops once it has enough to reconstruct. The
sender doesn't know (or care) when the receiver is "done."

This is **rateless** — the rate `k/n` approaches zero as `n → ∞`.

### Luby Transform (LT) codes, 1998

Michael Luby's insight: a sparse graph + belief-propagation decoding
gives near-linear-time fountain codes.

**Encoding (LT)**:
1. Pick a degree `d` from the **soliton distribution**.
2. Pick `d` source symbols uniformly at random.
3. XOR them (over binary) or sum them (over a finite field).
4. Output the encoded symbol.
5. Repeat until the receiver signals "done."

**Soliton distribution**:
```
ρ(1) = 1/k
ρ(d) = 1 / (d × (d - 1))   for d = 2, 3, ..., k
```

The expected degree is roughly `H(k) ≈ ln(k)` — sparse! Most encoded
symbols are XORs of ~ln(k) source symbols.

**Robust soliton distribution**: a tweak that adds extra degree-1
symbols to handle the "tail" of the decoding process. Luby's paper
gives the exact formula.

**Decoding**: belief propagation over the bipartite graph
(source symbols ↔ encoded symbols).

```
1. While there's an encoded symbol of degree 1:
       Reveal its source symbol.
       For every other encoded symbol containing it:
           Remove the source symbol (XOR out).
2. If all source symbols revealed: success.
3. Else: failure (need more encoded symbols).
```

For the robust soliton distribution, decoding succeeds with high
probability when the receiver has `k + O(√k × ln²(k/δ))` encoded
symbols. The overhead is `O(√k × ln²(k/δ))`, which is small for
large `k`.

### Raptor codes

LT codes have one weakness: the overhead is `O(√k × ln²(k))`, which
matters for small `k`. **Raptor codes** (Shokrollahi 2006) add a
**pre-code** (typically an LDPC or RS code) on the source symbols
before the LT step.

```
Source symbols (k)
    │
    ▼
Pre-code to k' intermediate symbols (k' > k, e.g., k' = k × 1.02)
    │
    ▼
LT-code the k' symbols
    │
    ▼
Encoded symbols (unbounded stream)
```

The pre-code gives the receiver "slack": even if LT decoding doesn't
fully succeed, the pre-code's error correction can fill in the gaps.

Raptor codes are **linear-time** in `k` (encoding and decoding both
take `O(k)` work), with **constant overhead**. They're used in 3GPP
multimedia broadcast (MBMS) and 5G.

### Why BFT consensus doesn't use them

For BFT block dissemination:

- **The sender knows the receiver count.** 100 validators. RS with
  `n = 100` is exactly right.
- **The sender knows the loss rate.** It's bounded (chapter 05 P2P
  uses reliable streams for control plane). RS with `k = 67` tolerates
  33% loss.
- **The receiver wants deterministic completion.** No "stop when you
  have enough" — they want `k` shards and done.
- **CPU overhead of LT decoding is similar to RS decoding** for
  typical `k`. The "rateless" benefit doesn't pay off.

ZODA's approach is closer to RS-with-checksums than to fountain codes:
fixed `n`, bounded `k`, deterministic decoding.

### Where fountain codes are the right choice

- **CDN distribution.** The sender doesn't know how many receivers
  there are.
- **Multicast video.** Receivers join and leave at arbitrary times.
- **Satellite / radio broadcast.** Bandwidth is the bottleneck; you
  want every byte to count, and you can't retransmit.
- **Disk archival.** Backblaze's Vaults uses a fountain-code-like
  construction for cold storage.

### Connection to ZODA

ZODA's "any `k` of `n`" is closer to RS than to fountain codes. But
you could imagine a hybrid:

1. Use ZODA within an epoch (fixed `n`).
2. When the peer set rotates (chapter 01), use a fountain-code-style
   "download from anyone who has it" for new peers.

Commonware doesn't ship this, but it's a reasonable extension.

## ZODA's security proof, sketched

The "ZODA paper" section above stated the security theorem. Let me
sketch why it's true.

### The theorem (informal)

> For any PPT adversary producing `(Com, V, H)` with at least one
> valid shard, the data can be reconstructed with all but `2^(-126)`
> probability.

### The reduction

Suppose an adversary forges a valid shard (one that passes all four
check steps) without having actually encoded the committed data.

The check steps are:
1. `Com == hash(V, data_bytes)` — collision resistance of BLAKE3.
2. `H = H_func(Com)` — Fiat-Shamir randomness.
3. Each row of the shard is in `V` (Merkle proof).
4. `X'[S_i] @ H == Z[S_i]` (Hadamard check).

If the adversary can pass all four without encoding the data, they
must have:

- Found a way to compute `H` from `Com` (which is allowed — it's
  Fiat-Shamir).
- Forged a Merkle proof of a row that isn't actually in `V`
  (requires breaking BLAKE3's collision resistance).
- Or: produced a polynomial `X'` whose Merkle root is `V` but whose
  Hadamard check `X' @ H = Z'` doesn't match the committed `Z`.

The third case: the adversary chooses `X'` first, computes
`V = Merkle(X')`, then `Com = hash(V, data_bytes)`, then `H` from
Fiat-Shamir, then `Z = X' @ H`. This is a valid encoding! So this
case is fine — the adversary actually did encode some data, just not
the data the protocol was supposed to encode.

The dangerous case: the adversary wants to claim `Com` for some
specific data `D`, but actually encode different data `D'`. Then
`Com = hash(Merkle(X(D)), data_bytes)` is fixed by the protocol, and
the adversary needs to produce `X'` whose Merkle root equals
`Merkle(X(D))`. That's a hash collision.

**Conclusion**: forging a valid shard requires breaking BLAKE3's
collision resistance. BLAKE3 has 128-bit collision resistance (the
underlying permutation is 256-bit, but collision resistance is
half-of-due-to-birthday). For 126-bit security, ZODA's parameters
ensure the adversary has only `2^(-126)` probability of success per
attempt.

### Parameter selection

The 126-bit level comes from:

- **Field size**: 64 bits per element.
- **Number of challenge samples**: 2 (the Hadamard matrix has 2
  columns of checksum data per shard, sampled via Fiat-Shamir).
- **Total soundness**: `2^64 × 2^62 = 2^126` (roughly).

The "roughly" is because the security reduction is more nuanced than
the product. The exact analysis is in the Dziembowski et al. paper.

### Why 126 bits and not 128

A 64-bit field gives at most 64 bits of "challenge hardness" per
sample. To reach 128 bits, you'd need either:

- A 128-bit field (Goldilocks is 64-bit; going to 128-bit is a
  different polynomial and slower arithmetic).
- 2 challenge samples (which ZODA does, giving ~126 bits).
- A different commitment scheme (e.g., hash-based Merkle with 256-bit
  hashes, but BLAKE3 already does this — the 126-bit comes from the
  field, not the hash).

The 126-bit level is **plenty** for BFT. The Bitcoin network's
collision-resistance target is ~80 bits (SHA-256's effective collision
resistance). 126 bits is generous.

## Random Linear Network Coding, deeply

The "network coding" section above mentioned random linear network
coding (RLNC) in passing. Let me give it more depth.

### The setup

A directed acyclic graph with `V` nodes. The source has `k` packets.
Each intermediate node receives packets, computes random linear
combinations, and forwards. Each sink collects the received packets
and inverts the system to recover the source.

### The encoding vectors

Every packet carries a **coding vector**: a list of coefficients
`(c_1, c_2, ..., c_k)` indicating which source symbols are combined.
A naive node keeps `c_1 = c_2 = ... = c_k = 0` (a "generation" vector)
and modifies it as it processes packets:

```
Send packet:  (coding_vector, payload)
              where payload = Σ c_i × source_packet_i
```

When a node forwards, it updates the coding vector:
- If node X sends packet P to node Y, and Y re-encodes with random
  vector `r = (r_1, ..., r_k)`:
- New coding vector: `r · P.coding_vector` (matrix-vector multiply)
- New payload: `Σ r_j × P.payload_j` (where P is a set of received packets)

The sinks invert a `k × k` matrix formed by the coding vectors of
their `k` received packets.

### The field-size question

For RLNC to work with high probability, the field `F_q` must be large
enough that random linear combinations are independent. The
probability of two random vectors being linearly dependent over
`F_q` is roughly `1/q`.

For `q = 256` (`GF(2^8)`), the dependency probability is `1/256 ≈ 0.4%`.
Over 100 packets, the chance of at least one dependent pair is ~33%.

For `q = 65536` (`GF(2^{16})`), it's `1/65536 ≈ 0.0015%`. Over 100
packets, the chance is ~0.15%.

For Commonware's Goldilocks (`q ≈ 2^64`), it's `≈ 2^{-64}` — effectively
zero.

### RLNC vs erasure coding for BFT

For BFT block dissemination:

- **Erasure coding** (RS, ZODA) pre-commits to `n` and `k`. The sender
  encodes once; the receivers know when they're done.
- **RLNC** generates on the fly. Each forwarder re-encodes.

RLNC is better for **multi-hop networks** where the topology isn't
known in advance (mesh networks, ad-hoc wireless). For BFT, where
the topology is a known peer set, erasure coding is simpler and
more efficient.

The exception: **gossip-based broadcast** (chapter 09), where peers
re-broadcast messages to newcomers. This is conceptually similar to
RLNC (each forwarder "re-encodes" the message by sending it), but
without the linear-combination cleverness.

### The cost of RLNC

The CPU cost per forwarder:
- Encoding: `O(k)` field multiplications (one matrix-vector).
- Decoding at sinks: `O(k³)` for inverting the system (or `O(k²·⁵)`
  with Strassen), or `O(k² × log k)` with the more efficient
  algorithms.

For `k = 67`: ~300,000 ops per decode. At 10 ns per op (Goldilocks),
about 3 ms. Acceptable.

Bandwidth cost: **each forwarder adds overhead equal to the coding
vector size** — `k` field elements per packet. For `k = 67` and
64-bit field, that's 536 bytes per packet of overhead. Small but
non-zero.

For ZODA, the per-shard overhead is larger (Merkle proof + Hadamard
checksum data) but the topology is simpler (no multi-hop).

## Exercises

These are designed to build intuition for the concepts above. Most
require only Rust's standard library; some reference Commonware types
for context.

### Exercise 1 — GF(256) arithmetic from scratch

Implement `Gf2p8` (an element of GF(2^8) with the AES polynomial
`0x11B`) and verify the field axioms:

```
∀ a, b, c: a + (b + c) == (a + b) + c
∀ a, b: a + b == b + a
∀ a: a + 0 == a
∀ a: a + a == 0
∀ a, b, c: a × (b × c) == (a × b) × c
∀ a, b: a × b == b × a
∀ a ≠ 0: a × a^(-1) == 1
```

**Hint**: define `add`, `sub`, `mul`, `inv` for your element type.
Use Fermat's little theorem for inversion.

### Exercise 2 — Vandermonde encoder for `k = 4, n = 6`

Implement a Reed-Solomon encoder over GF(256) for the parameters
`k = 4, n = 6`:

1. Pick 6 distinct field elements `α_0, ..., α_5`.
2. Construct the Vandermonde matrix `V` (4 × 6).
3. For a sample 4-element message `m`, compute `c = V · m`.
4. Pick any 4 of the 6 encoded symbols and decode (Lagrange
   interpolation).
5. Verify the decoded message matches `m`.

**Stretch goal**: do the same in Goldilocks with `k = 67, n = 100`.
Use the `commonware-math::fields::goldilocks` crate.

### Exercise 3 — The two-shard attack on plain RS

Implement the "shuffle" attack from the section above:

1. Encode a 1 KB message `B` with plain RS(`k = 4, n = 6`).
2. Implement a "honest" `check(shard)` that returns `true` for any
   single shard (this is the bug — plain RS has no better check).
3. Have a "malicious encoder" send `shard_0` to validator 1 and
   `shard_0` again (duplicate!) to validator 2. Both call `check` and
   get `true`.
4. Have validator 1 and validator 2 try to reconstruct. Both succeed,
   but they reconstructed the **same** data (because RS is
   deterministic). The attack fails because the encoding was
   deterministic.
5. Now have the malicious encoder send **valid-looking but different**
   shards to validators 1 and 2 (e.g., random field elements of the
   right size). Both pass `check`. Both try to reconstruct with
   shards from each other + their own. **They get different
   reconstructions**. Show this concretely.

**Stretch goal**: implement ZODA's check and verify that the same
attack fails.

### Exercise 4 — FWHT on a sample vector

Implement the FWHT (in any language) and verify:

1. `fwht([1, 1, 1, 1]) == [4, 0, 0, 0]` (the constant vector maps to
   one non-zero coefficient).
2. `fwht([1, -1, 1, -1]) == [0, 0, 4, 0]`.
3. `fwht(fwht(x)) == n × x` (FWHT is its own inverse, up to scaling).
4. Compute `fwht` for an 8-element vector and verify the
   `H · H^T = n · I` property.

### Exercise 5 — Luby Transform encoder with robust soliton

Implement the LT encoder (Luby 2002):

1. Implement the robust soliton distribution for `k = 100, δ = 0.05, c = 0.1`.
2. Sample degrees from the distribution.
3. Encode: pick `d` random source symbols, XOR them (binary case).
4. Generate 200 encoded symbols.
5. Decode with belief propagation. Verify decoding succeeds with ~150
   symbols on average.

**Stretch goal**: implement over a finite field (not just binary) and
try `k = 1000`.

### Exercise 6 — ZODA parameter selection

Read `coding/src/zoda/topology.rs` and answer:

1. What is `SECURITY_BITS`? Why that value and not 128?
2. What does `data_cols` represent? How is it chosen?
3. What is the formula for `required_samples`? Walk through what each
   variable means.
4. For a 1 MB block with `min_shards = 67, extra_shards = 33`, what
   are the values of `data_rows`, `samples`, `encoded_rows`?

## Cauchy Reed-Solomon and the systematic-form tradeoff

The standard RS construction uses Vandermonde matrices, which can have
numerical stability issues for certain field elements. **Cauchy
matrices** offer an alternative that's strictly better-conditioned and
supports efficient software implementations.

### The Cauchy matrix

Let `X = {x_0, ..., x_{k-1}}` and `Y = {y_0, ..., y_{n-1}}` be two
**disjoint** subsets of `F_q`. The Cauchy matrix is:

```
C_{ij} = 1 / (x_i - y_j)
```

The Cauchy matrix is always invertible (any square submatrix is
invertible, like Vandermonde). Its determinant has a closed form:

```
det(C) = Π_{i < j} (x_j - x_i)(y_i - y_j) / Π_{i, j} (x_i - y_j)
```

The Cauchy matrix has the property that **no entry is zero** (because
`x_i ≠ y_j` by disjointness). This is useful for software: you never
have to special-case division by zero.

### Cauchy RS vs Vandermonde RS

| Property | Vandermonde | Cauchy |
|---|---|---|
| Invertibility | Yes | Yes |
| Sparse form | No | No (but can be split into two halves) |
| Encoding speed | `O(kn)` | `O(kn)` (same) |
| Decoding speed | `O(k²)` | `O(k²)` (same) |
| Memory pattern | Sequential access | Random access |
| Numerical stability | Can be poor for near-α | Better |
| Parallelism | Per-row independent | Per-row independent |

The biggest practical difference: **Cauchy RS can be made systematic
trivially**. The systematic form has the first `k` symbols equal to
the message, so the receiver doesn't even need to decode if all
`k` data symbols arrive.

Commonware's `coding/src/reed_solomon.rs` uses the **NTT form** of
RS, not Cauchy. The reason: NTT enables the `O(n log n)` fast
encoding/decoding, which beats both Vandermonde and Cauchy for
`n > 64`.

### When you'd choose Cauchy over Vandermonde

- **Hardware implementations** where the constant factor matters.
- **Streaming decoders** that process symbols as they arrive.
- **Codes over small fields** (e.g., `GF(2^8)`) where Vandermonde
  becomes numerically fragile.

For Commonware's BFT workload (Goldilocks, `n ≤ 100`), neither matters.
NTT is the right choice.

## The ZODA implementation, line by line

Let's trace through the actual ZODA encoding to see how all the pieces
fit together. The relevant code is `coding/src/zoda/mod.rs` and
`coding/src/zoda/topology.rs`.

### Step 1: Choose parameters

```rust
// topology.rs
let topology = Topology::with_cols(
    data_bytes,
    min_shards,    // n
    extra_shards,  // k - n (so total = n + k - n = k... wait, k_total = n + extra_shards)
    data_cols,
)?;
```

The `data_cols` is the number of columns in the data matrix `X`. It's
chosen by `required_samples` based on the data size and the security
target.

### Step 2: Lay out the data as a matrix

The data `B` is laid out as a matrix `X` of shape `n*S × c`, where:

- `S = ceil(data_rows / n)` (samples per shard).
- `c = data_cols` (number of columns).
- `data_rows = ceil(data_bytes / (c × 8))`.

The matrix is conceptually `n*S` rows of `c` field elements each. The
total cells: `n*S × c ≈ data_bytes / 8`.

### Step 3: Reed-Solomon encode

The matrix is RS-encoded along the row axis. The result is `X'` of
shape `((n+k)*S) × c`, where:

- The first `n*S` rows are the original `X`.
- The remaining `k*S` rows are the parity.

The encoding uses NTT over Goldilocks for `O((n+k) × S × c × log((n+k)*S))`
work.

### Step 4: Merkle-commit

Build a Merkle tree over the rows of `X'`. The tree has `n*S` leaves
(the data rows) and extends to include the parity rows in the
authentication structure. The root is the vector commitment `V`.

### Step 5: Compute the public commitment

```rust
let com = hash(V, data_bytes);
```

This binds `V` (the row commitment) and the data size. The hash uses
BLAKE3 or another 256-bit hash.

### Step 6: Fiat-Shamir randomness

```rust
let h_seed = hash(com, "zoda-checksum");
let h_matrix = expand_to_field_elements(h_seed, data_cols * column_samples);
let h: Matrix = reshape(h_matrix, (data_cols, column_samples));
```

The `column_samples` is the number of Hadamard-check columns (typically
2 for 126-bit security). The hash expands deterministically into
field elements.

### Step 7: Compute the Hadamard checksum

```rust
let z = x.dot(&h);  // shape: (n*S) × column_samples
```

This is the heart of ZODA. Each row of `Z` is a linear combination of
the corresponding row of `X'` according to the rows of `H`. With
column_samples = 2 and data_cols = 2^16, each row of `Z` has 2
elements (128 bits total).

### Step 8: Build the shards

Each shard `i` contains:

- `data_bytes` (the original size, for verification).
- `V` (the vector commitment — same for all shards).
- `z[i*S..(i+1)*S]` (the Hadamard checksum slice for this shard).
- The rows `X'[i*S..(i+1)*S]` with Merkle proofs.

The total shard size: `(data_bytes / n) + V_overhead_per_shard +
checksum_per_shard + proofs`.

For a 10 MB block with `n = 67`: shard data is ~150 KB, plus ~32 KB of
checksums, plus ~16 KB of proofs. Total ~200 KB per shard.

### The check (per-shard verification)

```rust
fn check(commitment, index, shard) -> Result<CheckedShard, Error> {
    // 1. Verify the commitment
    let computed = hash(&shard.vector_commitment, shard.data_bytes);
    if computed != commitment { return Err(CommitmentMismatch); }

    // 2. Re-derive H via Fiat-Shamir
    let h = derive_h(commitment, shard.data_cols, shard.column_samples);

    // 3. Verify each row's Merkle proof
    for (row, proof) in shard.rows.iter().zip(&shard.merkle_proofs) {
        if !verify_proof(&shard.vector_commitment, row, proof) {
            return Err(InvalidProof);
        }
    }

    // 4. Verify the Hadamard check
    let z_computed = shard.rows.dot(&h);
    if z_computed != shard.checksum_slice {
        return Err(HadamardCheckFailed);
    }

    Ok(CheckedShard { rows: shard.rows.clone(), index })
}
```

If all four pass, the shard is provably from the committed encoding.

### Common gotchas in implementation

1. **Forgetting to bind `data_bytes` to the commitment.** Without it,
   the adversary could swap data of different sizes for the same
   `Com`.
2. **Using a deterministic RNG instead of Fiat-Shamir.** This makes
   the encoder's `H` predictable; an attacker could pre-compute a
   collision. Always hash `Com` for the seed.
3. **Skipping the row-count check.** The verifier should confirm
   `shard.rows.len() == expected_S`. Otherwise, a partial shard could
   pass.
4. **Caching `H` between calls without including `Com`.** Each call
   must re-derive `H` from `Com`.

## Common attacks and defenses

Beyond the Check Agreement bug, there are subtler attacks against
encoding schemes. Here's a taxonomy.

### The "short data" attack

The adversary commits to a small amount of data but produces shards
that contain padding. The verifier checks `data_bytes` against the
commitment but doesn't notice the padding.

**Defense**: ZODA's encoding includes `data_bytes` in the commitment,
and the decoder checks that the reconstructed data has the correct
size.

### The "replay" attack

The adversary takes a valid encoding from a previous round and
re-proposes it for a new round. The hash matches but the data is
stale.

**Defense**: Application-level logic checks the block height / view
number. The encoding layer alone can't prevent this.

### The "sybil" attack on the encoder

The adversary controls multiple nodes that all encode the same data
slightly differently, flooding the network with conflicting encodings.

**Defense**: The consensus protocol has a single designated leader
per view. The leader's encoding is canonical; conflicting encodings
from non-leaders are ignored.

### The "withholding" attack

The encoder gives different shards to different recipients, hoping
that some recipients can never reconstruct.

**Defense**: Per-shard checks (ZODA) and gossip-based shard
dissemination. If 67% of recipients get a working set of shards,
consensus proceeds.

### The "equivocation" attack

The encoder produces two valid encodings of the same data with
different commitments. The decoders split into two camps.

**Defense**: ZODA's commitment binds to the encoding via the Merkle
tree. An equivocating encoder has to forge the Merkle tree, which
requires breaking BLAKE3's collision resistance.

### The "small field" attack

Over a small field (e.g., `GF(2^8)`), the adversary can enumerate
all `2^8 = 256` possible polynomial evaluations and find a
collision cheaply.

**Defense**: Use a large field. Commonware uses Goldilocks with
`2^64` elements, giving 64-bit security per field element.

### The "lazy decoder" attack

A malicious decoder reconstructs only when convenient and signals
"failure" otherwise.

**Defense**: Simplex's `certify` hook is deterministic (chapter 02).
A lazy decoder either produces a result or doesn't; the protocol
handles both cases.

## Production deployment notes

Beyond the protocol correctness, here's what actually matters when
running ZODA in production.

### Parameter tuning

For BFT consensus, the parameters need to balance:

- **`min_shards`**: how many validators you want to recover the
  block. Higher = more redundancy but more bandwidth.
- **`extra_shards`**: how many parity shards. More = better fault
  tolerance but more bandwidth.
- **`data_cols`**: number of columns in the data matrix. More =
  smaller shards but more checksum overhead.

A typical setup for a 100-validator network with 10 MB blocks:

```
min_shards  = 67   (2f+1 with f=33)
extra_shards = 33  (1f)
data_cols   = 65536  (2^16, for 64 KB per shard column)
```

### Memory budget

For `n = 100, c = 65536`:
- `data_rows ≈ data_bytes × 8 / 65536`. For 10 MB, ~1280 rows.
- `samples = data_rows / n ≈ 13` rows per shard.
- `encoded_rows = (n + extra) × samples = 100 × 13 = 1300`.
- Memory for the encoding: `1300 × 65536 × 8 bytes = 650 MB`. Big.

ZODA uses lazy allocation and processes the matrix column-by-column
to keep memory bounded. The actual peak is much less than the
total matrix size.

### CPU budget

Encoding a 10 MB block:
- RS via NTT: ~50 ms on a 16-core machine.
- Merkle tree: ~20 ms.
- Hadamard check: ~100 ms.
- Total: ~170 ms.

For 1-second-per-block consensus, this is fine. For higher
throughput, you need either bigger machines or smarter
implementations.

### Network budget

Each validator receives ~200 KB per shard (with overhead). With 67
shards, that's ~13 MB total ingress — the data itself plus ~30%
overhead. Sender egress: ~13 MB total across the network.

The 30% overhead comes from:
- Merkle proofs: ~1 KB per row × 13 rows × 100 shards ≈ 1.3 MB
- Hadamard checksums: ~256 bytes per row × 13 rows × 100 shards ≈
  330 KB
- Vector commitment: 32 bytes per shard × 100 shards ≈ 3 KB

Total overhead: ~1.6 MB per 10 MB block. About 16%. Less than the
naive broadcast (which is 100% redundant: 990 MB sent for 10 MB).

### When ZODA isn't worth it

For blocks smaller than ~100 KB, the encoding overhead
(~1.6 MB absolute) becomes a significant fraction of the data
itself. For 10 KB blocks, ZODA's overhead is 16x the data!

In that regime, plain gossip (chapter 09) is more efficient: the
sender broadcasts the block, peers re-broadcast, everyone gets it.
The bandwidth cost is `O(N × data)`, but `data` is small.

Rule of thumb:
- **Blocks < 100 KB**: use plain gossip.
- **Blocks 100 KB - 10 MB**: ZODA is a clear win.
- **Blocks > 10 MB**: ZODA's CPU cost dominates; consider streaming
  + erasure coding (LT codes).

Commonware's Simplex is designed for the middle regime (kilobyte to
megabyte blocks), where ZODA shines.

## Where to look in the code (intermediate)

These pointers bridge the new sections above to the implementation.

- `coding/src/lib.rs:160-218` — `Scheme` trait (re-stated).
- `coding/src/lib.rs:131-150` — Check Agreement (now with three worked
  attacks in the chapter body).
- `coding/src/lib.rs:479-484` — `ValidatingScheme` (ZODA satisfies,
  plain RS doesn't).
- `coding/src/lib.rs:287-382` — `PhasedScheme` (strong/weak shards,
  covered in the new section above).
- `coding/src/reed_solomon.rs` — Vandermonde / NTT construction
  (deep dive in the new Vandermonde section).
- `coding/src/zoda/mod.rs` — ZODA implementation (line-by-line
  walkthrough above).
- `coding/src/zoda/topology.rs` — parameter selection (`SECURITY_BITS`,
  `data_cols`).
- `consensus/src/simplex/mod.rs:80-100` — the `CertifiableAutomaton`
  hook, where consensus and coding meet.
- `docs/blogs/carnot-bound.html` — theoretical limit.
- `eprint.iacr.org/2025/034` — the ZODA paper (Dziembowski et al.).

## If you only remember three things (expanded)

1. **Erasure coding cuts leader egress from `O(N × block)` to
   `O(block)`.** Send one shard per validator, validators gossip,
   everyone reconstructs. 100x bandwidth savings for big blocks.

2. **Plain Reed-Solomon has a Check Agreement bug.** A malicious
   encoder can give different validators different "valid" shards.
   ZODA fixes this with Hadamard checksums + Fiat-Shamir —
   per-shard checks now prove the shard is from the real encoding.

3. **The `certify` hook in Simplex is where consensus and coding
   meet.** Quorum notarizes the hash; application certifies the
   data after reconstruction; then quorum finalizes. ZODA makes
   the certify step deterministic and Byzantine-safe.

## Appendix A — Reed-Solomon codes, in math

Reed-Solomon is a **Maximum Distance Separable (MDS) code** based on
polynomial evaluation over a finite field.

### The basic idea

You have a message of `k` symbols (think of each symbol as a number in a
finite field `F_q`). You want to encode it into `n` symbols such that
**any `k` of the `n` can reconstruct the original**.

**Construction**:
- Pick `k` field elements `α_0, α_1, ..., α_{k-1}` (distinct).
- Treat the message as the polynomial `f(x) = m_0 + m_1*x + ... + m_{k-1}*x^{k-1}`
  of degree `k-1`.
- The encoded symbols are `f(α_0), f(α_1), ..., f(α_{n-1})`.

So you have `n` evaluations of a degree-`k-1` polynomial. **Any `k`
evaluations uniquely determine the polynomial** (Lagrange
interpolation).

**Decoding**: given any `k` evaluations `(α_i, y_i)`, use Lagrange
interpolation to recover `f(x)`, then read off the coefficients (which
are the original message).

### The math properties

- **MDS**: distance = `n - k + 1`. So the code can correct up to `n - k`
  erasures (missing symbols) OR `floor((n-k)/2)` errors (wrong symbols,
  with error correction).
- **Rate**: `k / n`. The "expansion factor" — how many bytes you ship
  vs the original message.
- **Field choice**: must be large enough to hold the message. Commonware
  uses `Goldilocks` (a 64-bit prime field from
  `commonware-math::fields::goldilocks`).

### Why Reed-Solomon works for block dissemination

A 10 MB block. Encode with `k = 67`, `n = 100` (i.e., any 67 of 100
shards reconstructs).

- Each shard is 10 MB / 67 ≈ 150 KB.
- Leader sends one shard to each of 100 validators (10 MB total egress).
- Each validator gossips shards; once you have 67, reconstruct.

Total bandwidth across the network: ~10 MB (the data itself), distributed
evenly. **Optimal.**

### The Check Agreement bug

`coding/src/lib.rs:131-150`:

> It should not be possible for parties A and B to both call `check`
> successfully on their own shards, but then have either of them fail
> when calling `check` on the other's shard.

With plain Reed-Solomon, this CAN happen:

**Setup**: Alice sends Bob shard #5, Charlie shard #7. Both are random
garbage (no polynomial behind them).

**Bob** receives shard #5, calls `check(shard_5)` — passes (any single
shard trivially "checks" against itself).
**Charlie** receives shard #7, calls `check(shard_7)` — passes.
**Bob** sends shard #5 to Charlie; Charlie calls `check(shard_5)` — fails
(it's not consistent with the polynomial Bob "should" have derived from
his "share").
**Or**: Bob and Charlie try to reconstruct from different subsets and
get different results.

The bug: with Reed-Solomon, a single shard tells you nothing about the
polynomial. You need `k` shards to even verify the polynomial exists.

**Commonware's ZODA fixes this.** See Appendix B.

## Appendix B — ZODA, in depth

`coding/src/zoda/mod.rs` and `coding/src/zoda/topology.rs`. ZODA (Zero-
Overhead Data Availability) is the novel construction from
`eprint.iacr.org/2025/034`. Let me walk through it more carefully.

### The data layout

Take `data_bytes` of data. Lay it out as a matrix `X`:

```
X is shape (n * S) x c, where:
    n = minimum_shards
    S = samples per shard (rows per shard)
    c = number of columns
    data_bytes determines c (so total cells ≈ data_bytes / 8)
```

Each shard `i` will contain rows `i*S..(i+1)*S` of the encoded matrix.

### The encoding

1. **Reed-Solomon encode** to get `X'` of shape `((n+k) * S) x c`.
   - `pad(((n+k) * S))` rounds up to the next power of two for efficient NTT.
2. **Commit** rows of `X'` via a Merkle tree, producing vector commitment `V`.
3. **Hash** `V + data_bytes` → public commitment `Com`.
4. **Derive randomness** from `Com` via Fiat-Shamir → Hadamard check matrix
   `H` of shape `c x S'`.
5. **Compute `Z = X @ H`** of shape `(n * S) x S'` — the checksum matrix.
6. **Shard `i`** contains: `data_bytes`, `V`, `Z[i*S..(i+1)*S]`, and rows
   `X'[i*S..(i+1)*S]` with Merkle proofs in `V`.

### The security parameter

`topology.rs:7`:

```rust
const SECURITY_BITS: usize = 126;
```

The protocol provides 126 bits of security. Why not 128? Because the
arithmetic is over `F_p` where `p` is the Goldilocks prime (2^64 -
2^32 + 1). Using a 64-bit field gives you 126 bits of "soundness
error" — the probability that an attacker can fool you is ≤ 2^(-126).

### The topology calculation

`topology.rs:34-49`:

```rust
fn with_cols(data_bytes: usize, n: usize, k: usize, cols: usize) -> Self {
    let data_els = F::bits_to_elements(8 * data_bytes);
    let data_rows = data_els.div_ceil(cols);
    let samples = data_rows.div_ceil(n);
    Self {
        data_bytes,
        data_cols: cols,
        data_rows,
        encoded_rows: ((n + k) * samples).next_power_of_two(),
        samples,
        column_samples: 0,
        min_shards: n,
        total_shards: n + k,
    }
}
```

The "how many columns" choice is non-trivial. More columns = more
checksum overhead but smaller proofs. Less columns = less overhead but
larger proofs. The protocol picks based on the data size and security
target.

`required_samples` (line 51-66) computes the number of checksum
columns needed for the security target. The math involves computing
log2 of a probability ratio (how likely is a malicious encoder to fool
the verifier?).

### The verification

To verify a shard:

1. **Check the commitment**: `Com == hash(V, data_bytes)`.
2. **Re-derive `H`** from `Com` (Fiat-Shamir makes this deterministic).
3. **Check `Z` dimensions**: shape `(n*S) x S'` as expected.
4. **Encode `Z`** to get `Z'` of shape `pad((n+k)*S) x S'`.
5. **Check shard data** has shape `S x c` and each row is in `V` (Merkle
   proof).
6. **Check Hadamard**: `X'[S_i] @ H == Z'[S_i]`.

If all pass, the shard is **provably from a valid encoding** — even with
just 1 shard.

### Why this fixes Check Agreement

The Hadamard check `X' @ H == Z` ties every row of `X'` to a deterministic
checksum. For a malicious encoder to satisfy the check with random data,
they'd need to find a polynomial `f` such that `f @ H = Z'` for their
chosen `Z'`. With 126 bits of security, this is computationally
infeasible.

So **any shard that passes `check` is provably from the encoding that
produced the commitment**. No more "different validators get different
valid shards" ambiguity.

## Appendix C — `PhasedScheme` in detail

`coding/src/lib.rs:287-382`. For gossip-based dissemination, you want two
levels of shard:

### Strong shard

The original from the encoder. Contains everything:
- `data_bytes`
- Vector commitment `V`
- Full checksum slice
- Rows + Merkle proofs

This is what the leader sends to each validator.

### Weak shard

Stripped-down for gossiping between validators:
- Just the rows
- Just the Merkle proofs
- (No commitment, no checksum — those are derived from the strong shard
  by the recipient)

Smaller, faster to forward, but only verifiable against the recipient's
`CheckingData` (which they derived from their strong shard).

### `weaken`

```rust
fn weaken(config, commitment, index, strong_shard)
    -> Result<(CheckingData, CheckedShard, WeakShard), Error>;
```

Takes a strong shard, derives:
- `CheckingData` — enough information to verify weak shards from peers.
- `CheckedShard` — the verified contribution to reconstruction.
- `WeakShard` — what to forward.

### `check`

```rust
fn check(config, commitment, checking_data, index, weak_shard)
    -> Result<CheckedShard, Error>;
```

Takes a weak shard, verifies against the local `CheckingData`, returns
a checked shard.

### `decode`

```rust
fn decode(config, commitment, checking_data, checked_shards, strategy)
    -> Result<Vec<u8>, Error>;
```

Reconstructs the data from the checked shards. Needs `n` checked shards
(any subset of size `n`).

## Appendix D — The `Strategy` parameter

`coding/src/lib.rs:178-181`:

```rust
fn encode(
    config: &Config,
    data: impl Buf,
    strategy: &impl Strategy,
) -> Result<(Self::Commitment, Vec<Self::Shard>), Self::Error>;
```

The `strategy: &impl Strategy` is `commonware_parallel::Strategy`. Two
implementations:

```rust
pub enum StrategyChoice {
    Sequential,    // single-threaded
    Rayon(RayonPool),  // parallel
}
```

`Sequential` — straight-line execution. Used in tests where you want
deterministic behavior.

`Rayon` — uses the rayon data-parallel library. Each shard is encoded
on a separate thread. The encode/decode are embarrassingly parallel —
each shard's computation is independent.

The `ThreadPooler` trait from chapter 02 spawns the rayon pool on the
runtime's threads. So you don't need a separate thread pool — it uses
the runtime's.

## Appendix E — The `Strategy::Parallel` and shard work distribution

When you use `Rayon`:

```rust
let shards: Vec<Shard> = data
    .chunks(rows_per_shard)
    .enumerate()
    .par_bridge()             // rayon: turn into parallel iterator
    .map(|(i, chunk)| encode_shard(config, chunk, i))
    .collect::<Result<_, _>>()?;
```

Each shard's encoding involves:
- Polynomial evaluation (for Reed-Solomon): O(rows × cols) field
  multiplications.
- Merkle tree construction (for ZODA): O(rows × log(rows)) hashes.
- Hadamard check computation (for ZODA): O(rows × cols × S') operations.

For a 10 MB block with `n = 67, k = 33` (100 shards total):
- Each shard: ~150 KB of data, ~32 KB of checksum.
- Encoding: ~1 ms per shard on a single core.
- Parallel: ~50 ms total on 67 cores, or ~100 ms on 33.

## Appendix F — The Certificate Scheme integration

When you use ZODA in a consensus system:

```rust
impl CertifiableAutomaton for MyApp {
    async fn certify(&mut self, round: Round, payload: Digest) -> Receiver<bool> {
        let key = payload_to_zoda_key(payload);
        if !self.shard_cache.contains(&key) {
            // Wait for shards via resolver
            let _ = self.resolver.fetch(key).await;
        }

        // Check that we have enough shards
        let checked_shards: Vec<_> = self.shard_cache.get(&key)
            .iter()
            .filter_map(|s| Zoda::check(&self.config, &self.commitments[&key], s).ok())
            .collect();

        if checked_shards.len() < self.config.min_shards {
            return false;  // can't reconstruct yet
        }

        // Try to reconstruct
        match Zoda::decode(&self.config, &self.commitments[&key],
                            &self.checking_data[&key],
                            checked_shards.iter(), &self.strategy).await {
            Ok(data) => hash(&data) == payload && self.app_verify(&data),
            Err(_) => false,
        }
    }
}
```

This is the **decoupling in action**: consensus agrees on the hash
(payload); application certifies by reconstructing from shards and
checking the hash matches.

## Appendix G — Edge cases and gotchas

### The "lazy encoder" attack

A malicious encoder could send out valid-looking shards but never have
committed to an actual encoding. With Reed-Solomon, this isn't
detectable. With ZODA, the commitment `Com = hash(V, data_bytes)` binds
the encoder to the data. Even if they don't reveal the data, the
commitment locks them in.

If the encoder later reveals different data, the commitment won't
match.

### The "withholding shards" attack

A malicious encoder sends different shards to different validators,
hoping to keep some validators from reconstructing. With ZODA, each
validator checks their shard and refuses to sign if it doesn't check.
Consensus halts; the encoder's attack fails.

### Shard staleness

If you receive a shard for an old commitment (e.g., from a previous
round), you might check it against the wrong `Com`. Always check
`commitment == expected_commitment` before using a shard.

## Where to look in the code (expanded)

- `coding/src/lib.rs:160-218` — the `Scheme` trait.
- `coding/src/lib.rs:131-150` — Check Agreement.
- `coding/src/lib.rs:479-484` — `ValidatingScheme`.
- `coding/src/lib.rs:287-382` — `PhasedScheme`.
- `coding/src/reed_solomon.rs` — Reed-Solomon implementation.
- `coding/src/zoda/mod.rs` — ZODA implementation.
- `coding/src/zoda/topology.rs` — topology calculations.
- `consensus/src/simplex/mod.rs:80-100` — the `CertifiableAutomaton` hook.
- `docs/blogs/carnot-bound.html` — the theoretical limit.

## If you only remember three things

1. **Erasure coding cuts leader egress from O(n × block) to O(block).** Send one shard per validator, validators gossip, everyone reconstructs. 100x bandwidth savings for big blocks.
2. **Plain Reed-Solomon has a Check Agreement bug.** A malicious encoder can give different validators different "valid" shards. ZODA fixes this with Hadamard checksums + Fiat-Shamir.
3. **The `certify` hook in Simplex is where consensus and coding meet.** Quorum notarizes the hash; application certifies the data after reconstruction; then quorum finalizes.

→ Next: **Chapter 08 — Resolver** (small) and **Chapter 09 — Broadcast**
(important). Resolver is "fetch the data behind a hash." Broadcast is the
high-level data dissemination primitive that uses coding under the hood.