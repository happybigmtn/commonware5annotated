# Chapter 04 — Cryptography: keys, signatures, and the magic of aggregation

> The crypto layer underneath every consensus message.

## What we need from crypto in distributed systems

Three things, roughly in order of importance:

1. **Identity.** When validator X says "I voted for block B," how do you know
   it's really X?
2. **Integrity.** When X says "the previous block was B-1," how do you know X
   didn't change B-1's hash to point to a different chain?
3. **Aggregation.** When 67% of validators all sign the same thing, you want a
   single small proof — not 67 individual signatures.

Item 3 is the killer feature. Commonware supports it.

## The trait surface

Open `cryptography/src/lib.rs:82-192`. Five core traits:

```rust
pub trait Signer: Random + Send + Sync + Clone + 'static {
    type Signature: Signature;
    type PublicKey: PublicKey<Signature = Self::Signature>;
    fn public_key(&self) -> Self::PublicKey;
    fn sign(&self, namespace: &[u8], msg: &[u8]) -> Self::Signature;
}

pub trait Verifier {
    type Signature: Signature;
    fn verify(&self, namespace: &[u8], msg: &[u8], sig: &Self::Signature) -> bool;
}

pub trait PublicKey: Verifier + Sized + ReadExt + Encode + PartialEq + Array {}
pub trait Signature: Sized + Clone + ReadExt + Encode + PartialEq + Array {}

pub trait BatchVerifier {
    type PublicKey: PublicKey;
    fn new() -> Self;
    fn add(&mut self, namespace: &[u8], message: &[u8],
           public_key: &Self::PublicKey,
           signature: &<Self::PublicKey as Verifier>::Signature) -> bool;
    fn verify<R: CryptoRngCore>(self, rng: &mut R, strategy: &impl Strategy) -> bool;
}
```

Notice the **`namespace` argument** to `sign` and `verify`. This is **domain
separation**. If you sign `b"transfer 100"` for the consensus layer, you don't
want that signature to also be valid on the execution layer. The namespace
prefixes the message before signing, so `sign(b"L1", b"transfer 100")` and
`sign(b"L2", b"transfer 100")` produce completely different signatures.

`Secret<T>` (`lib.rs:54-65` and `cryptography/src/secret.rs`) wraps raw key
material and **zeroizes on drop**. You can't accidentally leak a private key
through a stack clone.

## The four signature schemes

`certificate.rs:11-26` lays out the four schemes Commonware ships:

| Scheme | Speed | Aggregation | HSM | Trusted Setup | Attribution |
|---|---|---|---|---|---|
| `ed25519` | Fast | No | Yes | No | Yes |
| `secp256r1` | Medium | No | Yes | No | Yes |
| `bls12381_multisig` | Slow sign, fast verify | Yes (multi-sig) | No | No | Yes |
| `bls12381_threshold` | Slow sign, fast verify | Yes (threshold) | No | Yes (DKG) | **No** |

Let me explain what these mean.

### Ed25519 — the default

Ed25519 is what most blockchain projects use by default. Signatures are 64
bytes, public keys 32 bytes. Verification is fast. Hardware security modules
support it. No trusted setup.

**Downside:** signatures don't aggregate. If 100 validators sign the same
message, your certificate is 100 × 64 = 6,400 bytes. Verifying 100 signatures
is 100× the cost of verifying 1.

### BLS12-381 — the magic curve

BLS12-381 is an **elliptic curve pairing**. Its magic property: signatures
are points on a curve, and you can **multiply** them.

```
sig = sk * H(msg)              # sk is private key, H maps msg to curve point
verify(pk, msg, sig):          # pk is public key
    e(sig, G2) == e(H(msg), pk)    # pairing check
```

Now the trick: signatures are points. **Adding two points is a point.**

```
sig_a + sig_b = sk_a * H(msg) + sk_b * H(msg) = (sk_a + sk_b) * H(msg)
```

So if Alice and Bob both sign the same message `msg`, their signatures add up
to the signature that `(sk_a + sk_b)` would produce. You verify **one** combined
signature against the combined public key `pk_a + pk_b`. **2 signatures → 1
verification.**

For 100 signers: 100 signatures → 1 aggregated signature → 1 verification.
That's the magic.

### Two BLS modes: multisig vs threshold

Both modes use BLS12-381. Both produce aggregable signatures. The difference
is what the "key" is:

**Multisig:** Each signer has their own (sk_i, pk_i). They each produce a
real signature. Anyone can aggregate any subset of signatures. The
certificate contains all individual signatures AND the aggregate (for
verification) AND the indices of who signed (for attribution).

**Threshold:** A group of `n` parties **collectively** hold a private key `S`.
No individual party knows `S`. They run a **Distributed Key Generation (DKG)**
protocol to set it up. Each party ends up with a `share_i` of `S`. They can
each produce a **partial signature** on a message. Once you collect **any `t`
partials** (where `t` is the threshold, usually `2f+1`), you can reconstruct
the full signature. You don't need the original parties — anyone can combine
`t` partials.

The certificate in threshold mode is **constant size** regardless of `n`.
One signature, one group public key. Verify and you're done.

But there's a critical property: **`t` signers can forge any missing
signer's partial signature**. Because each partial is just `(sk_i / S) *
H(msg)`, and you can compute any other partial by interpolating through the
ones you have. So threshold signatures are **non-attributable** — you can't
prove "Alice signed this" to a third party. You can only prove "a quorum of
the committee signed this."

`certificate.rs:28-47` makes this distinction explicit:

> **Attributable Schemes** (ed25519, secp256r1, bls12381_multisig): Individual
> signatures can be presented to some third party as evidence of either
> liveness or of committing a fault.
>
> **Non-Attributable Schemes** (bls12381_threshold): Individual signatures
> cannot be presented as evidence because they can be forged by other players.

This matters for slashing design. If you want to slash validators who don't
sign, use attributable. If you want compact certificates and don't need
fine-grained slashing, use threshold.

### Why batch verification, and why randomness

Look at `BatchVerifier::verify` (`lib.rs:178-191`):

> When performing batch verification, it is often important to add some
> randomness to prevent an attacker from constructing a malicious batch of
> signatures that pass batch verification but are invalid individually.
>
> Abstractly, think of this as there existing two valid signatures (`c_1` and
> `c_2`) and an attacker proposing (`c_1 + d` and `c_2 - d`).

The batch verification equation is linear. Without randomness, an attacker
can construct a linear combination of valid signatures that sums correctly
but contains garbage. Adding randomness (a random scalar per signature)
breaks this attack.

The link in the doc points to ethresear.ch for the full attack. The TL;DR:
**always add randomness to batch verification**.

## DKG — Distributed Key Generation

How do `n` parties end up sharing a private key without anyone learning it?
This is the DKG problem, and Commonware ships two implementations
(`bls12381/dkg/mod.rs:1-19`):

| Protocol | Rounds | Sync assumption | Cost |
|---|---|---|---|
| `feldman_desmedt` | 2 | Synchronous (bounded dealer messages) | Cheap |
| `golden` | 1 | Asynchronous (uses encryption + ZK proofs) | More expensive |

`feldman_desmedt` is the textbook approach. A "dealer" sends encrypted shares
to each player along with public commitments. Players verify their share is
on the committed polynomial. After all players acknowledge, the threshold
key is set.

`golden` is a recent Commonware contribution (`docs/blogs/golden.html`):
it's a one-round DKG that works without synchrony, at the cost of more
cryptography (public-key encryption per share, zero-knowledge proofs of
correctness).

For most apps, `feldman_desmedt` is fine. For networks where you can't bound
message delays (e.g., adversarial network conditions), use `golden`.

## Timelock Encryption (TLE)

`bls12381/tle.rs:1-30`. Here's the problem it solves:

> Encrypt a message so it can ONLY be decrypted when a valid signature over a
> specific target (e.g., a future timestamp or round number) becomes
> available.

In other words: I want to send an encrypted bid to an auction. The bid
should be invisible until the auction closes (round 1000). After round 1000,
everyone can see all bids.

TLE does this with **Identity-Based Encryption (Boneh-Franklin)**. The
"identity" is the target (e.g., `b"round-1000"`). The master secret key is
held by a threshold committee. Once they produce a signature on round 1000,
that signature acts as the decryption key for all ciphertexts encrypted to
that identity.

The implementation uses the **Fujisaki-Okamoto transform** to get
CCA-security (resistance to chosen-ciphertext attacks).

Use cases: sealed-bid auctions, threshold decryption of user transactions
(MEV protection), commit-reveal schemes.

## Transcripts — the safe way to hash things

`transcript.rs:1-20`:

> A Transcript is useful for hashing data, committing to it, and extracting
> secure randomness from it. The API evades common footguns when doing these
> things in an ad hoc way.

The footguns: re-using a hash context for different things (cross-protocol
attack), forgetting to commit before extracting randomness, etc.

Look at `transcript.rs:91-98`:

```rust
enum StartTag {
    New = 0,     // brand new transcript
    Resume = 1,  // pick up where we left off
    Fork = 2,    // split the transcript into two independent children
    Noise = 3,   // generate randomness from this transcript
}
```

Different start modes get different tags (`0`, `1`, `2`, `3`). So even if you
happen to commit the same data, a `New` transcript and a `Resume` transcript
produce different hashes. **Domain separation baked into the API.**

The transcript internally uses Blake3, which is fast and has a clean
extensibility story (the `OutputReader` lets you derive many outputs from
one transcript state).

## Putting it together — the certificate abstraction

`certificate.rs:189-264` defines the `Verifier` trait, used by Simplex and
other consensus primitives:

```rust
pub trait Verifier: Clone + Debug + Send + Sync + 'static {
    type Subject<'a, D: Digest>: Subject;
    type PublicKey: PublicKey;
    type Certificate: Clone + Debug + PartialEq + Eq + Hash + Send + Sync + Codec;

    fn verify_certificate<R, D, M>(
        &self,
        ...
    ) -> Result<Verification<Self>, ...>;
}
```

This is what makes Simplex **cryptography-agnostic**. You can write a
consensus instance with ed25519 (simple, attributable, big certificates),
BLS multisig (compact, attributable), or BLS threshold (compactest,
non-attributable). The consensus logic doesn't care.

The Simplex config from chapter 01 has a `scheme: Scheme` field. That's the
type parameter. Pick the scheme that fits your deployment.

## Why this matters for chapter 01's `2f+1`

Now re-read chapter 01 with crypto in hand.

In **ed25519 mode**, collecting `2f+1` votes means collecting `2f+1`
individual signatures. The "certificate" is literally those `2f+1`
signatures. Verifying it means running `verify` on each one. ~2f+1 individual
verification operations.

In **BLS threshold mode**, collecting `2f+1` partial signatures means
collecting `2f+1` shares of the threshold key. You **reconstruct** the full
threshold signature from those `2f+1` shares (Lagrange interpolation).
Verify once against the static group public key. The certificate is one
signature.

That's a 100x bandwidth and verification cost reduction for large committees.
The tradeoff: trusted setup (DKG), non-attributability.

## Where to look in the code

- `cryptography/src/lib.rs:82-192` — Signer, Verifier, PublicKey, Signature, BatchVerifier.
- `cryptography/src/certificate.rs:11-58` — the four schemes compared.
- `cryptography/src/ed25519/` — the ed25519 implementation.
- `cryptography/src/bls12381/scheme.rs` — the BLS12-381 implementation.
- `cryptography/src/bls12381/dkg/feldman_desmedt.rs` — synchronous DKG.
- `cryptography/src/bls12381/dkg/golden/` — asynchronous 1-round DKG.
- `cryptography/src/bls12381/tle.rs` — timelock encryption.
- `cryptography/src/transcript.rs` — the Transcript abstraction.
- `consensus/src/simplex/scheme/` — how Simplex uses these schemes.

## A number theory primer — the math under the magic

The signature schemes we use (Ed25519, BLS12-381) all rest on a small
set of number-theory facts. Once you have these, the rest of the chapter
becomes "this scheme is the math applied to a particular problem."

### Modular arithmetic

The integers `0, 1, 2, ..., p-1` with addition and multiplication modulo
a prime `p` form a **finite field** `F_p`. Every arithmetic identity
that holds for the integers holds in `F_p`, modulo `p`. For example,
in `F_7`:

```
3 + 5 = 8 mod 7 = 1
3 * 5 = 15 mod 7 = 1
```

A consequence: every non-zero element `a` in `F_p` has a **multiplicative
inverse** `a^(-1)` such that `a * a^(-1) = 1 mod p`. This is because `p`
is prime, so `F_p` is a field.

This gives us a tiny, closed arithmetic world where we can do
cryptography. The "hardness" assumption: it's easy to compute
`a * b mod p`, but given `a * b mod p` and `a`, recovering `b` requires
computing the modular inverse. That's the **discrete logarithm problem**.

### Fermat's little theorem

For any `a` not divisible by `p`:

```
a^(p-1) ≡ 1 (mod p)
```

A direct consequence: `a^(-1) ≡ a^(p-2) (mod p)`. So inverting is just
exponentiation, which is `O(log p)` operations via square-and-multiply.

This is the engine that powers every elliptic-curve signature: signing
and verifying both reduce to a handful of exponentiations modulo a large
prime.

### The discrete logarithm problem (DLP)

In a group `G` (e.g., `F_p^*` under multiplication, or an elliptic curve
under addition), given `g` and `h = g^x`, find `x`.

For well-chosen groups, this is believed to be **exponentially hard**.
The fastest known algorithms (index calculus for `F_p^*`, baby-step
giant-step or Pollard's rho for elliptic curves) take `O(sqrt(N))` time,
where `N` is the group size. So a 256-bit group gives `2^128` security,
which is unbreakable.

This is the basis for Diffie-Hellman key exchange, ElGamal signatures,
DSA, Schnorr, Ed25519, and BLS.

### Why elliptic curves work

An elliptic curve `E` over `F_p` is a set of points satisfying

```
y^2 = x^3 + a*x + b  (mod p)
```

with an additional "point at infinity" `O`. These points form a group
under a chord-and-tangent addition operation.

The DLP on elliptic curves — given `P` and `Q = k*P`, find `k` — is
**harder** than the DLP on `F_p^*`. Index calculus attacks work for
`F_p^*` but not for elliptic curves. The fastest known attacks are the
generic `O(sqrt(n))` algorithms, where `n` is the curve order.

Practical consequence: a 256-bit elliptic curve gives the same security
as a 3072-bit RSA modulus. Smaller keys, smaller signatures, faster
operations. This is why Ed25519 (32-byte keys, 64-byte signatures) can
replace RSA-3072 (384-byte keys, 384-byte signatures).

Ed25519 uses **Curve25519**, an Edwards curve. The math is the same;
the Edwards form (`-x^2 + y^2 = 1 + d*x^2*y^2`) makes addition
formulas faster and **complete** (no special cases for `P = Q` or
`P = -P`). That last property is what makes Ed25519 immune to a class
of side-channel attacks.

BLS12-381 is different: it's a **pairing-friendly** curve, which means
it has extra structure (a small embedding field) that supports the
bilinear pairing operation. That structure is why BLS can aggregate
signatures and Ed25519 cannot. Trade-off: a pairing-friendly curve is
slightly weaker per bit than a non-pairing curve, so BLS12-381 uses a
381-bit modulus (instead of 256).

That's the entire math foundation. The rest of the chapter is applying
it.

## Why elliptic curves — the security-size comparison

A useful table (security levels from NIST SP 800-57 and the 2023
recommendations):

| Security (bits) | RSA / DSA modulus | ECC key size | Hash output |
|---|---|---|---|
| 112 | 2048 | 224 | 224 |
| 128 | 3072 | 256 | 256 |
| 192 | 7680 | 384 | 384 |
| 256 | 15360 | 512 | 512 |

For BFT consensus at the 128-bit security level:

- Ed25519: 32-byte public key, 64-byte signature. Total ~96 bytes per
  validator per signature.
- BLS12-381 MinSig: 96-byte signature, 96-byte public key. Per-signer
  overhead is larger, but signature aggregation collapses `n` signatures
  into one.

The ECC win is enormous at the protocol level. A 100-validator committee
sending 64-byte signatures: 6.4 KB per certificate. Same committee with
BLS aggregation: 96 bytes. That's a 67x bandwidth reduction.

### Edwards curves vs Weierstrass curves

Most textbook ECC uses **Weierstrass form**: `y^2 = x^3 + a*x + b`. The
addition formulas require handling special cases (point at infinity,
doubling vs adding, `y = 0`). Implementations must guard against these
to avoid timing leaks.

Edwards curves use `-x^2 + y^2 = 1 + d*x^2*y^2`. Their addition formulas
are **complete**: they handle all inputs uniformly. No special cases.
This means constant-time implementations are easier, and the formulas
are faster (fewer multiplications).

Ed25519 uses the twisted Edwards form of Curve25519 (the Montgomery
form of the same curve, `v^2 = u^3 + 486662*u^2 + u`, is what X25519 key
exchange uses — same curve, different coordinate system).

BLS12-381 is in Weierstrass form (and the `blst` library uses optimized
formulas to compensate for the special cases). The trade-off: BLS12-381
needs to be a **pairing-friendly** curve, and the known constructions
happen to produce Weierstrass curves.

## BLS12-381 — the pairing-friendly construction

Not every elliptic curve supports pairings. Pairing-friendly curves
require **embedding degree** `k > 1`: the smallest `k` such that the
curve's points embed into the multiplicative group of some extension
field `F_{p^k}`.

For a "random" elliptic curve, the embedding degree is enormous —
effectively, the curve doesn't embed usefully. For a "pairing-friendly"
curve, `k` is small (typically 6, 10, 12, or 18). BLS12-381 has
**embedding degree 12**, which is why it's named BLS**12**.

### Why BLS12-381 specifically

The "12" gives you a target group `GT` in `F_{p^12}`. Pairing computation
goes through a tower of field extensions:

```
G1 ⊂ E(F_p)              -- 381-bit prime field
G2 ⊂ E'(F_{p^2})         -- square extension
GT ⊂ F_{p^12}            -- degree-12 extension
```

Pairing costs are dominated by the cost of arithmetic in `F_{p^12}`,
which is much more expensive than in `F_p`. But the security is also
much higher per bit. The sweet spot at the 128-bit security level is
a 381-bit `p`, hence the name.

BLS12-381's specific parameter set (`r` is the group order, `p` is the
field modulus):

```
r = 0x1a0111ea397fe69a4b1ba7b6434bafd667fb22d7748e6ba6f4b1ba7b6c9a0d6e
    ... (about 381 bits)
p = 0x1a0111ea397fe69a4b1ba7b6434bafd667fb22d7748e6ba6f4b1ba7b6c9a0d6f
    ... (slightly larger, also 381 bits)
```

The exact values are in the Commonware source (cryptography/src/bls12381/primitives/).
The reason these specific numbers are chosen:

1. **`r` is prime** (so the groups are cyclic of prime order).
2. **`p` is prime** and `p ≡ 1 (mod 12)` (so the embedding-degree-12
   extension exists cleanly).
3. **`r` divides `p - 1` with very large cofactor** (so the extension
   arithmetic has nice properties for pairing computation).
4. **The curve constants `a, b` are small** (so curve arithmetic is
   fast — specifically, `a = 0`).

These are not random choices. They're tuned to make the pairing cheap
and the security tight. Commonware uses `blst` (Supranational), an
audited, optimized implementation that knows all of these tricks.

### Why MinSig vs MinPk

A BLS signature can live in either `G1` or `G2`. The pair stays the
same (`e: G1 × G2 → GT`), but you flip which side has the signature.

| Variant | Signature size | Public key size |
|---|---|---|
| MinSig | 48 bytes (G1) | 96 bytes (G2) |
| MinPk | 96 bytes (G2) | 48 bytes (G1) |

If you have lots of signatures (e.g., one per transaction), MinSig saves
bandwidth. If you have lots of public keys (e.g., one per validator),
MinPk saves bandwidth.

For BFT consensus where validators' public keys appear in the validator
set and signatures appear in every certificate, MinPk is the common
choice.

## Pairings explained — the bilinear map

A **bilinear pairing** `e: G1 × G2 → GT` is a function that takes two
points (one from each group) and returns a third (in the target group).
The "bilinear" part:

```
e(a*P, b*Q) = e(P, Q)^(a*b)
```

This is the magic that makes BLS aggregation work. Why?

In `G1` (or `G2`), the discrete log is hard: given `P` and `Q = a*P`,
you cannot recover `a`. In `GT`, the discrete log might be easier (or
harder) depending on the choice of `GT`. Pairings give us a "lift" from
`G1 × G2` into a different algebraic world.

### Computational vs decisional Diffie-Hellman

Standard DLP-based crypto uses **decisional** hardness: it's hard to
distinguish `(g, g^a, g^b, g^(ab))` from `(g, g^a, g^b, g^c)` for
random `a, b, c`. This is the CDH assumption.

Pairing-based crypto uses **co-CDH** (co-gap Diffie-Hellman): given
`g` in `G1` and `g^a` in `G2`, computing `g^(ab)` in `G1` is hard.
The pairing `e(g, g^a)^b = e(g^b, g^a) = e(g, g^(ab))` is the trapdoor.

The XDH assumption (external Diffie-Hellman) says that DDH is hard in
`G1` and `G2` separately. This is what makes BLS signatures secure;
without XDH, the signature could be forged via the pairing.

### Why aggregation works

BLS signing: `sig = sk * H(m)` (where `H(m)` is a point in `G1`,
`sk` is a scalar, and the result is a point in `G1`).

If Alice signs with `sk_a` and Bob signs with `sk_b`, on the same `m`:

```
sig_a + sig_b = sk_a * H(m) + sk_b * H(m) = (sk_a + sk_b) * H(m)
```

This is exactly the signature that the **combined secret key**
`(sk_a + sk_b)` would produce. And verifying it works because of
bilinearity:

```
e(sig_a + sig_b, g_2) = e((sk_a + sk_b) * H(m), g_2)
                      = e(H(m), g_2)^(sk_a + sk_b)

e(H(m), pk_a + pk_b) = e(H(m), (sk_a + sk_b) * g_2)
                     = e(H(m), g_2)^(sk_a + sk_b)   ✓
```

So you verify **one** combined signature against **one** combined
public key. The size of the certificate is independent of the number
of signers.

This is why BLS is the backbone of modern BFT: 100-vote committees
collapse to 1 verification.

## Hash functions in depth — SHA-256 vs SHA-3 vs BLAKE3

The cryptography crate ships three hash families. They differ in
construction, properties, and performance.

### Merkle-Damgard (SHA-256)

SHA-256, SHA-384, SHA-512 all use the **Merkle-Damgard** construction:
split the input into 512-bit blocks, process each block into an
intermediate state, hash the state to produce the final digest.

```
state_0 = IV
state_1 = compress(state_0, block_1)
state_2 = compress(state_1, block_2)
...
digest = state_n
```

Property: the compression function is collision-resistant, and the
construction preserves that. Collision-resistant means: it's hard to
find two inputs with the same digest.

Vulnerability: **length extension**. Given `H(m)` and `len(m)`, you can
compute `H(m || padding || m')` without knowing `m`. SHA-256 is
vulnerable to this. So is SHA-512. So is anything Merkle-Damgard.

Workaround for Commonware: the `Transcript` abstraction (chapter 04 main
text) length-prefixes every commit, eliminating length-extension attacks.

### Sponge construction (SHA-3, Keccak)

SHA-3 (Keccak) uses a **sponge**: absorb input into the state, then
"squeeze" output. The state has a "rate" portion (data goes in/out) and
a "capacity" portion (stays internal).

```
state = IV (capacity bits + rate bits)

absorb:
  for each block:
    state[rate] ^= block
    permute(state)

squeeze:
  for each output block:
    output_block = state[rate]
    permute(state)
```

Property: collision-resistant, **and** length-extension-resistant
(because the capacity never leaks). SHA-3 is also indifferentiable
from a random oracle, which is a strong formal property.

Cost: Keccak's permutation is slower than SHA-256's compression on
most CPUs (despite being the NIST winner).

### Tree mode (BLAKE3)

BLAKE3 takes a different approach: a **Merkle tree** of chunks, with
each chunk independently compressed.

```
        root
       /    \
    H(A,B)  H(C,D)
    /  \    /  \
   A    B  C    D
   |    |  |    |
 chunks of input (1 KB each)
```

Properties:

- **Parallelizable.** Each chunk can be hashed independently on a
  different core. Trivially scales to 16+ cores.
- **Incremental.** Updating one byte only re-hashes the chunks along the
  path to the root. For a 1 MB file, that's about 10 chunks
  (`log2(1024)`), not all 1024.
- **No length-extension.** Same argument as SHA-3 — the structure is
  sponge-like internally.
- **Fast.** Roughly 4-8x faster than SHA-256 on modern hardware due
  to parallelism and a tree structure that's SIMD-friendly.

This is why Commonware uses BLAKE3 for **transcripts** (chapter 04 main
text). When you have a sequence of items to hash, tree mode lets you
parallelize the absorb. When you need to derive multiple sub-hashes from
one transcript state, BLAKE3's `OutputReader` (a streaming output
mode) is more flexible than SHA-3's sequential squeeze.

### When SHA-256 is still right

SHA-256 isn't dead. It's the conservative choice for **interoperability**:
every language has a SHA-256 library, every FIPS-compliant system
supports it, every hardware security module can compute it in hardware.
When you want maximum compatibility (e.g., bridging to Bitcoin, which
double-hashes SHA-256), use SHA-256.

Commonware's choice: `Transcript` uses BLAKE3 by default. The
`Sha256` module is available for cases that need interop.

## Merkle trees in depth — RFC 9162 and beyond

A Merkle tree is a binary tree of hashes:

```
        root = H(H(AB) || H(CD))
       /                          \
  H(A || B)                    H(C || D)
   /     \                      /       \
  A       B                    C         D
  |       |                    |         |
leaves (data blocks)
```

To prove `A` is in the tree, provide `B` and `H(CD)` — the verifier
recomputes `H(A||B)`, then `H(H(A||B) || H(CD))`, and checks against
the root.

Properties:

- **O(log n) proof size.** For `n` leaves, a proof is `log2(n)` hashes.
- **O(log n) verification.** The verifier does `log2(n)` hashes.
- **Fixed root size.** Always the same hash size, regardless of `n`.

### Second-preimage resistance

A Merkle tree is **second-preimage resistant** as long as the hash
function is collision-resistant. The proof: to find two trees with the
same root but different leaves, you'd need to find a collision in the
hash function at some level. SHA-256, BLAKE3, and SHA-3 are all
collision-resistant.

### Inclusion vs exclusion proofs

An **inclusion proof** shows that a specific leaf is in the tree (you
walk up from the leaf to the root using sibling hashes).

An **exclusion proof** shows that a specific leaf is **not** in the
tree. For a sorted Merkle tree (chapter 16), this requires proving that
the leaf's "neighbors" are present and the target is absent.

Commonware's authenticated databases (chapter 16) use sorted Merkle
trees via MMR (Merkle Mountain Range), which has slightly different
properties but the same underlying hash structure.

### RFC 9162 — Certificate Transparency

RFC 9162 ("Certificate Transparency Version 2.0") defines a specific
Merkle tree construction for logging TLS certificates. The interesting
property: the tree is **append-only**, and a "Single Leaf" proof is
logarithmic in the tree size at the time the leaf was added (not the
current size).

Commonware's MMR (chapter 16) has a similar append-only property.
Inclusion proofs over historical MMRs are still valid forever, even
as new leaves get added.

## Shamir's secret sharing — the math behind threshold reconstruction

Shamir's 1979 scheme is the foundation for threshold cryptography.

### The construction

A **threshold** `t` and a **total** `n`. We want to share a secret `S`
among `n` parties such that any `t` parties can reconstruct `S`, but
`t - 1` learn nothing.

Pick a random polynomial `f(x)` of degree `t - 1` with `f(0) = S`:

```
f(x) = S + a_1*x + a_2*x^2 + ... + a_{t-1}*x^{t-1}
```

Distribute shares: party `i` gets `f(i)` for `i = 1, ..., n`.

### Why this works

Any `t` shares uniquely determine `f(x)` (Lagrange interpolation). The
constant term `f(0) = S` is the secret.

Any `t - 1` shares determine `f(x)` only up to the missing terms. There
are `p` equally-likely possibilities for the secret (where `p` is the
field size). Information-theoretically, no information leaks.

### Lagrange interpolation

Given `t` points `(x_i, y_i)` with distinct `x_i`, recover the
coefficients of `f`:

```
f(x) = Σ y_i * L_i(x)

L_i(x) = Π (x - x_j) / (x_i - x_j)  for j ≠ i
```

The Lagrange basis polynomial `L_i(x)` is "1 at x_i, 0 at every other
x_j." So at `x = 0`:

```
S = f(0) = Σ y_i * L_i(0)
       = Σ y_i * Π (-x_j) / (x_i - x_j)
```

For Shamir, you don't even need the full polynomial. You just compute
`S = Σ y_i * L_i(0)`.

### The DKG twist

In a real threshold protocol (like Commonware's BLS threshold), the
shares are not scalars — they're **scalar values modulo the curve
order `r`**. The polynomial has coefficients in `F_r`, and each share
is a scalar.

`secret.rs` wraps scalar shares in `Zeroizing<...>` so they get wiped
when dropped (see "Rust pattern" section below).

## Threshold BLS — Lagrange in the exponent

Plain BLS aggregation works when each signer has a full private key.
Threshold BLS is different: **no signer has the full key**, but `t`
partial signatures can be combined.

### The setup (DKG)

The committee of `n` parties runs a DKG (chapter 04 main text, the
`feldman_desmedt` or `golden` protocols). At the end:

- Each party `i` holds a **share** `sk_i ∈ F_r` (a scalar).
- The "master secret" `S` is the constant term of a polynomial `f(x)`.
- `f(i) = sk_i` (party `i`'s share).
- The polynomial is degree `t - 1` (threshold `t`).
- The master public key is `PK = S * G_2` (a point in `G2`).
- Each party's individual public key is `pk_i = sk_i * G_2` (also `G2`).

No one knows `S`. The polynomial is implicit; only the share/individual-
pk pairs are visible.

### Signing

Party `i` produces a partial signature on message `m`:

```
partial_sig_i = sk_i * H(m)   ∈ G1
```

Verify against `pk_i`:

```
e(partial_sig_i, G2) == e(H(m), pk_i)
```

### Combining `t` partials

Given partials from parties `i_1, ..., i_t`, reconstruct the full
signature:

```
full_sig = Σ L_{i_k}(0) * partial_sig_{i_k}    ∈ G1

where L_i(0) = Π (-i_j) / (i - i_j)    for j ≠ k
```

This is Lagrange interpolation **in the exponent**. The scalars
`L_i(0)` are the same Lagrange basis coefficients as in Shamir, but
now they multiply **curve points**, not field elements.

### Verifying

```
e(full_sig, G2) == e(H(m), PK)
```

This works because:

```
full_sig = Σ L_i(0) * (sk_i * H(m))
        = (Σ L_i(0) * sk_i) * H(m)
        = S * H(m)             (by Lagrange interpolation of f(0))
        = f(0) * H(m)
```

And `e(S * H(m), G2) = e(H(m), S * G2) = e(H(m), PK)` by bilinearity.
The verification equation holds.

### Why aggregated threshold sigs are non-attributable

Here's the punchline: the `full_sig` is a single point in `G1`. It's
the **same point regardless of which `t` partials you combined**. The
set of partials you used to produce it is not visible in the output.

Even worse: given `full_sig` and any `t - 1` partials, you can compute
the missing `t`'th partial (just Lagrange-interpolate). So any `t`
signers can forge any missing signer's partial.

This is what `certificate.rs:28-47` means by "non-attributable." You
can prove "a quorum of the committee signed this," but you cannot prove
"Alice signed this" to a third party.

For slashing, you want **attributable** schemes (Ed25519, BLS
multisig). For compact certificates, you accept non-attributability.

## Quantum resistance — what happens when Shor ships

The bad news for current BFT: **Shor's algorithm** (1994) breaks all
DLP-based crypto (including RSA, DSA, Ed25519, BLS12-381) in
polynomial time on a sufficiently large quantum computer.

### What Shor does

Given `g` and `g^x mod p` (or `P` and `k*P` on an elliptic curve),
Shor's algorithm finds `x` in `O((log p)^3)` quantum operations. This
is **exponentially faster** than the best classical algorithm.

For a 256-bit elliptic curve, classical security is `2^128`. Quantum
security with Shor drops to `O(256^3) ≈ 2^24` — trivially breakable.

A sufficiently large quantum computer (thousands of logical qubits,
millions of physical qubits with error correction) doesn't exist yet,
but it's a credible long-term threat. "Harvest now, decrypt later" is
already a concern: an adversary recording encrypted traffic today can
decrypt it in 15 years when quantum computers are ready.

### What survives

Two families of post-quantum crypto are believed to resist Shor:

1. **Lattice-based.** CRYSTALS-Dilithium (signatures), CRYSTALS-Kyber
   (KEMs). Based on the hardness of finding short vectors in lattices.
   Large keys (~1 KB) and signatures (~2 KB).
2. **Hash-based.** XMSS (eXtended Merkle Signature Scheme) and SPHINCS+.
   Based only on the security of the underlying hash function. SPHINCS+
   signatures are ~8-50 KB.

Commonware's current posture: **not post-quantum**. The four schemes
shipped (Ed25519, secp256r1, BLS12-381 multisig, BLS12-381 threshold)
are all DLP-based. BLS threshold in particular depends on pairings,
which Shor breaks.

### Migration strategy

When migration happens, the playbook is:

1. **Hybrid signatures.** Sign with both an Ed25519 and a SPHINCS+ key.
   Verify both. As long as either is secure, you're safe.
2. **New curve types.** Replace BLS12-381 with a pairing-equivalent
   over post-quantum-friendly groups (an active research area).
3. **Hash-based state commitments.** State roots are Merkle hashes;
   those survive Shor. Only signatures need replacement.

The `stability` system in Commonware is designed for this: at ALPHA,
you can swap Ed25519 for SPHINCS+ in a single release. At BETA+, you'd
need a migration window where both are accepted.

For now, the practical guidance: if you're deploying for a 5-year
timeframe, classical is fine. If you're deploying for 20+ years, plan
for post-quantum migration.

## Random number generation — ChaCha20 in tests, CSPRNG in prod

The cryptography crate distinguishes **test RNG** from **production
RNG**. The distinction matters.

### NIST SP 800-90A — DRBGs

NIST's standard for deterministic random bit generation defines three
algorithms: Hash_DRBG, HMAC_DRBG, CTR_DRBG. They're all designed to
take entropy input and produce a stream of unpredictable bits.

The catch: **entropy input must be unpredictable**. If your seed comes
from a weak source (current time, PID, etc.), the output is weak too.
For this reason, modern OSes expose `getrandom()` (Linux) or
`BCryptGenRandom` (Windows), which gather entropy from hardware
interrupts, disk timings, etc.

Commonware's `Signer::from_seed` (in `cryptography/src/ed25519/`) takes
a seed and produces a deterministic private key. That's fine for tests
(seed known, output reproducible) but **must not be used in production**
for actual signing keys (the seed is the private key; if it's leaked,
the key is leaked).

### ChaCha20-based CSPRNG

The `ChaCha20Rng` (from `rand_chacha`) is the standard test RNG.
ChaCha20 is a stream cipher; using it as an RNG means:

```
key = some seed
counter = 0
output = ChaCha20(key, counter=0) || ChaCha20(key, counter=1) || ...
```

The output is indistinguishable from random as long as the key is
secret and `counter` doesn't repeat. ChaCha20's counter is 64 bits,
which means you can generate `2^64` blocks (~256 TB at 64 bytes/block)
before the counter wraps. Plenty for tests.

Commonware's `commonware_utils::test_rng()` returns a `ChaCha20Rng`
seeded by `Runner::seeded(seed)`. For multiple independent RNG streams
in one test, use `test_rng_seeded(N)` (chapter 02 testing pattern).

### Why ChaCha20 in tests specifically

Three reasons:

1. **Speed.** ChaCha20 is fast on any CPU, with or without hardware AES.
2. **Determinism.** Given a seed, the output is deterministic. Same seed,
   same RNG state, same output. Tests can be reproduced.
3. **Simplicity.** No hardware dependency. Runs identically on x86, ARM,
   RISC-V.

In production, Commonware uses `OsRng` (or whatever the runtime's RNG
is). The cryptography crate's `PrivateKey::random(&mut rng)` works with
any `CryptoRngCore`, so production code passes the OS RNG and test
code passes `test_rng()`.

### The Commonware pattern

`commonware_utils::test_rng()` is the canonical way to get a
deterministic RNG. From `cryptography/src/ed25519/`:

```rust
let mut rng = test_rng();                      // seeded from runtime
let sk = PrivateKey::generate(&mut rng);       // deterministic
```

Two RNGs with the same seed produce the same `sk`. This is critical
for **simulator determinism**: the same seed should produce the same
state machine trace.

## Rust pattern — `Zeroize` for key material

`Secret<T>` (`cryptography/src/secret.rs`) wraps key material in a
`Zeroizing<T>`:

```rust
pub struct Secret<T: Zeroize>(Zeroizing<T>);

impl<T: Zeroize> Drop for Secret<T> {
    fn drop(&mut self) {
        self.0.zeroize();
    }
}
```

When `Secret<PrivateKey>` goes out of scope, the underlying bytes are
overwritten with zeros before being freed. This defends against:

- **Memory dumps.** A post-mortem dump doesn't contain the key.
- **Swap files.** If the page is swapped out and later swapped in, the
  key bytes are gone.
- **Cold-boot attacks.** Reading DRAM immediately after power-off can
  sometimes recover data. Zeroizing removes the data.

The `Zeroize` trait (from the `zeroize` crate) is implemented for
primitive integer types, arrays, and `Vec<u8>`. For custom types, you
implement it manually.

For scalars, `Secret<Scalar>` is common. The `Zeroizing` wrapper from
the `zeroize` crate ensures the scalar's bytes are wiped on drop.

### `subtle` for constant-time operations

The `subtle` crate provides constant-time wrappers for secret-dependent
operations:

```rust
use subtle::ConstantTimeEq;

let a: [u8; 32] = ...;
let b: [u8; 32] = ...;

if a.ct_eq(&b).into() {     // constant-time equality
    // ...
}
```

Why this matters: a regular `==` on byte arrays short-circuits on the
first differing byte. The time-to-compare reveals how many leading
bytes match — leaking information about the secret. `ct_eq` always
takes the same time, regardless of where the difference is.

Commonware uses `subtle` for signature verification: comparing a
computed value to an expected value in constant time. The `blst`
library also handles this internally for pairing checks.

### Pattern summary

For cryptographic code in Commonware:

1. **Wrap secret material in `Secret<T>` or `Zeroizing<T>`.** It zeroes
   on drop.
2. **Use `subtle` for any comparison that involves a secret.** Don't
   `==` on private keys, expected hashes, etc.
3. **Use `OsRng` or `test_rng()` — never raw byte arrays as keys.**
   Even if you "intend" them to be random, you'll accidentally seed
   with non-random data.
4. **Use `Zeroizing` to bind secrets to a scope:**

```rust
let secret_key = Zeroizing::new(Scalar::random(&mut rng));
let sig = sign(secret_key.as_ref(), namespace, msg);
// secret_key automatically zeroed at end of scope
```

This pattern appears throughout `cryptography/src/bls12381/`. It's the
defense-in-depth for key material.

## Number theory in depth

The main-body section "A number theory primer" introduces the
basic concepts: modular arithmetic, Fermat's little theorem, the
discrete log problem. This section goes deeper into the
*algebraic structure* that makes the cryptography work: the group
(Z/nZ)*, subgroups, the structure of Z_p*, the Chinese Remainder
Theorem, and the proofs of the theorems.

### Groups, rings, fields

A **group** is a set G with a binary operation * that satisfies
four axioms:

1. **Closure.** For all a, b ∈ G, a * b ∈ G.
2. **Associativity.** For all a, b, c ∈ G, (a * b) * c = a * (b *
   c).
3. **Identity.** There exists e ∈ G such that for all a ∈ G, a *
   e = e * a = a.
4. **Inverse.** For every a ∈ G, there exists a⁻¹ ∈ G such that a
   * a⁻¹ = a⁻¹ * a = e.

If the operation is also commutative (a * b = b * a for all a,
b), the group is **abelian**.

A **ring** is a set R with two operations, + and *, such that
(R, +) is an abelian group, * is associative, and * distributes
over +. If * is also commutative and has an identity (1), the
ring is a **commutative ring with identity**.

A **field** is a commutative ring with identity where every
non-zero element has a multiplicative inverse. Examples: the
rationals Q, the reals R, the complex numbers C, and the integers
modulo a prime Z/pZ.

The integers modulo a composite Z/nZ are *not* a field, because
not every non-zero element has a multiplicative inverse. For
example, in Z/4Z, 2 has no inverse: 2 * 1 = 2, 2 * 2 = 0, 2 * 3 =
2. So Z/4Z is a ring but not a field.

### Modular arithmetic in Z/nZ

The set {0, 1, ..., n-1} with addition and multiplication modulo
n is the **ring of integers modulo n**, denoted Z/nZ or Z_n.

The **additive group** (Z/nZ, +) is always a cyclic group of
order n, generated by 1. Every element a has order n / gcd(a, n).

The **multiplicative group** (Z/nZ)* is the set of elements with
a multiplicative inverse. An element a has an inverse modulo n
iff gcd(a, n) = 1.

For n = p (a prime), Z/pZ is a field, and (Z/pZ)* has order p - 1.
For n = pq (a product of two distinct primes), (Z/nZ)* has order
(p-1)(q-1), but its structure is more complex (it's Z/(p-1) ×
Z/(q-1), by the Chinese Remainder Theorem).

### Fermat's little theorem (with proof)

**Theorem.** If p is prime and gcd(a, p) = 1, then a^(p-1) ≡ 1
(mod p).

**Proof.** Consider the set S = {a, 2a, 3a, ..., (p-1)a} modulo
p. Each element is distinct: if ia ≡ ja (mod p) for 0 < i < j <
p, then (j - i)a ≡ 0 (mod p), so p divides (j - i)a. Since p
doesn't divide a (gcd(a, p) = 1), p divides (j - i), but 0 < j -
i < p, contradiction.

So S has p - 1 distinct non-zero elements modulo p. The set {1, 2,
..., p - 1} also has p - 1 distinct non-zero elements modulo p.
Since both sets are the same size and live in the same universe,
they must be equal as sets:

{a, 2a, ..., (p-1)a} = {1, 2, ..., p-1} (mod p)

Multiplying all elements of both sides:

a^(p-1) * (p-1)! ≡ (p-1)! (mod p)

Since gcd((p-1)!, p) = 1, we can cancel (p-1)!:

a^(p-1) ≡ 1 (mod p). ∎

This is the foundation of RSA and Diffie-Hellman.

### Euler's theorem and the totient function

**Euler's totient function** φ(n) counts the number of integers
in {1, ..., n} that are coprime to n.

- φ(p) = p - 1 for prime p.
- φ(pq) = (p-1)(q-1) for distinct primes p, q.
- φ(p^k) = p^k - p^(k-1) = p^(k-1)(p-1) for prime p.
- φ is multiplicative: φ(ab) = φ(a)φ(b) for coprime a, b.

**Euler's theorem.** If gcd(a, n) = 1, then a^φ(n) ≡ 1 (mod n).

This generalizes Fermat's little theorem: when n = p, φ(p) = p -
1, and we recover Fermat.

### The Chinese Remainder Theorem

**Theorem.** If m and n are coprime, then the system

```
x ≡ a (mod m)
x ≡ b (mod n)
```

has a unique solution modulo mn.

**Proof sketch.** Since gcd(m, n) = 1, there exist integers u, v
such that um + vn = 1 (Bezout's identity). Set x = a * vn + b *
um. Then x ≡ a * 1 = a (mod m), and x ≡ b * 1 = b (mod n).
Uniqueness modulo mn follows from the fact that the solution
lives in a ring of size mn.

The CRT has two implications for cryptography:

1. **RSA modulus structure.** The RSA modulus n = pq has
   (Z/nZ)* ≅ (Z/pZ)* × (Z/qZ)*, which has order (p-1)(q-1). This
   is why knowing p and q lets you compute φ(n) and break RSA.

2. **Computation speedup.** Operations modulo n = pq can be done
   in parallel modulo p and modulo q, then combined via CRT.
   This is roughly 4× faster than working modulo n directly,
   and is used in production RSA implementations.

### The discrete logarithm problem (DLP), formally

Let G be a cyclic group of order n, generated by g. The **discrete
logarithm** of h ∈ G is the integer x ∈ {0, 1, ..., n-1} such
that g^x = h.

The DLP is the problem: given g, h, and the group, find x.

The best general-purpose algorithm for the DLP in a generic group
is **Pollard's rho**, which runs in O(√n) time and O(1) space.
For specific groups (Z/pZ)* or elliptic curves), there are
faster algorithms using index calculus or number field sieve, but
for elliptic curve groups, no such algorithms are known, and the
best attack is Pollard's rho.

This is why elliptic curves are preferred for DLP-based
cryptography: with a 256-bit curve, Pollard's rho takes 2^128
operations, which is infeasible. With a 256-bit prime p for DLP in
(Z/pZ)*, the index calculus method runs in sub-exponential time,
which is much faster than 2^128.

### Why DLP is hard in prime-order groups

A **prime-order group** is a group whose order is a prime number.
In such a group, every non-identity element is a generator (has
order equal to the group order).

This matters for the DLP because Pollard's rho runs in O(√p)
time, where p is the group order. If p is large (say, 2^256),
Pollard's rho is infeasible.

In a group of composite order n, Pollard's rho runs in O(√d),
where d is the smallest prime factor of n. So a group of order
2^256 with a factor of 2^16 is breakable in O(2^8) time.

This is why Commonware uses prime-order subgroups for DLP-based
cryptography: BLS12-381's G1 and G2 are prime-order subgroups
of larger elliptic curve groups.

### The structure of (Z/pZ)*

For p prime, (Z/pZ)* is a cyclic group of order p - 1. Its
structure depends on the factorization of p - 1.

If p - 1 = 2q where q is prime, then (Z/pZ)* has a subgroup of
order q, and DLP in that subgroup is roughly as hard as DLP in
the full group (Pollard's rho: O(√q) = O(√(p-1)/√2)).

This is the structure used in some DLP-based protocols (e.g.,
the Schnorr group): a prime p such that p - 1 = 2q for prime q,
with operations in the subgroup of order q.

### Quadratic residues

An element a ∈ (Z/pZ)* is a **quadratic residue** if there exists
b such that b² ≡ a (mod p). Otherwise, a is a **quadratic
non-residue**.

The set of quadratic residues forms a subgroup of index 2 in
(Z/pZ)*. By Euler's criterion, a is a quadratic residue iff
a^((p-1)/2) ≡ 1 (mod p).

This matters for the Rabin cryptosystem and for the Legendre
symbol used in some DLP variants.

### The structure of (Z/nZ)* for composite n

For n = pq (distinct primes), (Z/nZ)* has order (p-1)(q-1). The
structure is (Z/pZ)* × (Z/qZ)* by the CRT, which is Z/(p-1) ×
Z/(q-1).

The DLP in (Z/nZ)* is as hard as DLP in (Z/pZ)* plus DLP in
(Z/qZ)*, which is roughly √p and √q respectively. So if p and q
are both ~2^1024, the DLP takes ~2^512 operations.

The factorization of n is a different problem: the best general
algorithm is the **General Number Field Sieve (GNFS)**, which runs
in sub-exponential time. For a 2048-bit RSA modulus, GNFS is
estimated to take ~2^112 operations, which is feasible for a
nation-state but not for an individual.

This is why RSA key sizes are much larger than elliptic curve
key sizes: RSA-2048 has roughly the same security as
ECC-224. ECC gets more security per bit.

### Subgroups and cofactors

A **subgroup** of G is a subset H ⊆ G that is itself a group
under the same operation. The order of H divides the order of G.

If |G| = n and |H| = d, then d divides n, and the **index** of H
in G is n/d.

The **cofactor** of H in G is n/d. So in a group of order n with
a subgroup of prime order q, the cofactor is n/q.

Cofactors matter for cryptographic security: a small cofactor
makes subgroup attacks easier, so a cofactor of 1 (i.e., the
group itself is prime-order) is ideal. BLS12-381 achieves this
in G1 and G2 (both are prime-order groups).

### Putting it all together

The cryptography in Commonware relies on:

- The hardness of the DLP in prime-order groups (BLS12-381's G1
  and G2, Ed25519's underlying curve).
- The bilinearity of the pairing map (for BLS aggregation).
- The algebraic structure of Z/pZ for the hash-to-curve step.

All of these rest on the number theory we just covered. The
"magic" of pairing-based signatures is just the bilinear map
e: G1 × G2 → GT, combined with the elliptic curve group law, all
of which is grounded in the integer arithmetic we just
described.


## Elliptic curves in depth

The main-body section "Why elliptic curves work" introduces the
basic idea: an elliptic curve is a set of points with a group
structure, and the DLP on an elliptic curve group is harder per
bit than the DLP in Z/pZ. This section goes deeper: the geometry
of the group law, the algebraic structure, the different curve
forms, and the comparison of curve25519, secp256k1, and BLS12-381.

### The Weierstrass form

An elliptic curve in **Weierstrass form** is given by the
equation:

```
y² = x³ + ax + b
```

over a field F. (For characteristic 2 or 3, the form is
slightly different, but we will focus on characteristic > 3.)

For the curve to be non-singular (no cusps or self-intersections),
the discriminant Δ = -16(4a³ + 27b²) must be non-zero.

The curve is the set of points (x, y) ∈ F² satisfying the
equation, plus a "point at infinity" denoted O.

The famous visualization of an elliptic curve over the reals
looks like a noodle:

```
       |
       |
     _,_)
    /   |
   /    |
  /     |
 (------+----------
  \     |
   \    |
    \   |
     `-_)
       |
       |
```

This is for y² = x³ - x over the reals.

### The group law geometrically

Given two points P and Q on the curve, the group law P + Q is
defined as:

1. Draw the line through P and Q.
2. The line intersects the curve at a third point R (counting
   multiplicities).
3. Reflect R across the x-axis to get -R.
4. Define P + Q = -R.

If P = Q, the "line" is the tangent at P. If P = -Q (i.e., Q is
the reflection of P across the x-axis), the line is vertical and
P + Q = O (the point at infinity).

This construction is the *geometric* definition of the group
law. It satisfies the group axioms:

- **Associativity.** This is the hard part to prove geometrically;
  the proof is non-trivial.
- **Identity.** O (the point at infinity) is the identity.
- **Inverse.** The inverse of P is its reflection across the
  x-axis.

### The group law algebraically

The same group law can be expressed algebraically. For two
distinct points P = (x₁, y₁) and Q = (x₂, y₂), the sum P + Q =
(x₃, y₃) is given by:

```
λ = (y₂ - y₁) / (x₂ - x₁)
x₃ = λ² - x₁ - x₂
y₃ = λ(x₁ - x₃) - y₁
```

For doubling (P + P), the slope is the tangent:

```
λ = (3x₁² + a) / (2y₁)
x₃ = λ² - 2x₁
y₃ = λ(x₁ - x₃) - y₁
```

For adding O (the identity), the result is the other point.

These formulas are what the actual code implements. The
geometric picture is just intuition; the code does the
arithmetic.

### The discrete log problem on elliptic curves

The **elliptic curve discrete log problem (ECDLP)** is: given P
and Q = nP on an elliptic curve, find n.

For a curve over F_p with about p points, the best general
algorithm is Pollard's rho, which takes O(√p) operations. There
is no known sub-exponential algorithm for ECDLP (unlike DLP in
(Z/pZ)*).

So a 256-bit elliptic curve (p ~ 2^256) gives ~128 bits of
security, comparable to a 3072-bit RSA modulus. This is the
*security per bit* advantage of ECC.

### Why ECC gives smaller keys

The best known algorithm for ECDLP runs in O(√p) time. The best
known algorithm for DLP in (Z/pZ)* runs in sub-exponential time
(L(1/3, c) for some constant c). For a 256-bit modulus, L(1/3,
c) is much smaller than 2^128. So to get 128 bits of security
against DLP in (Z/pZ)*, you need a much larger modulus.

This is why:

| Security | RSA / DH (bits) | ECC (bits) |
|----------|-----------------|------------|
| 80       | 1024            | 160        |
| 112      | 2048            | 224        |
| 128      | 3072            | 256        |
| 192      | 7680            | 384        |
| 256      | 15360           | 512        |

For the same security level, ECC keys are ~10× smaller than
RSA/DH keys.

### The Edwards form

**Edwards curves** are a different parameterization of elliptic
curves, given by:

```
x² + y² = 1 + dx²y²
```

For d not equal to 0 or 1, this defines an elliptic curve (under
some conditions). The group law on Edwards curves is *complete*:
the addition formula works for all pairs of inputs, including
duplicates and inverses. This means no special cases, which is
faster and less error-prone.

The addition law is:

```
(x₁, y₁) + (x₂, y₂) = (
  (x₁y₂ + y₁x₂) / (1 + dx₁x₂y₁y₂),
  (y₁y₂ - x₁x₂) / (1 - dx₁x₂y₁y₂)
)
```

This is used in Ed25519 and in the BLS12-381 G1.

### The Montgomery form

**Montgomery curves** are given by:

```
By² = x³ + Ax² + x
```

The most famous use of Montgomery form is **Curve25519**, designed
by Daniel Bernstein in 2006. Curve25519 uses the Montgomery
form to enable *constant-time* scalar multiplication, which is
resistant to timing attacks.

Montgomery curves have a fast scalar multiplication using the
**Montgomery ladder**, which always performs the same sequence
of operations regardless of the scalar. This is a critical
property for avoiding side-channel attacks.

Curve25519 is used for Diffie-Hellman key exchange, not for
signatures. Ed25519 (which is the same underlying curve but in
Edwards form) is used for signatures.

### The short Weierstrass form

Many elliptic curves use the **short Weierstrass form**:

```
y² = x³ + ax + b
```

This is what we described earlier. Examples: secp256k1 (used in
Bitcoin), P-256 (used in TLS), P-384, P-521.

The short Weierstrass form is the standard for many
cryptographic protocols because of its simplicity.

### curve25519 vs secp256k1 vs BLS12-381

The three curves used in Commonware:

**Curve25519 (Ed25519 for signatures):**
- Edwards form for signatures: -x² + y² = 1 + (121665/121666)
  x²y²
- 256-bit security
- Fast, constant-time, no cofactor issues
- Used for peer authentication in the P2P layer

**secp256k1:**
- Short Weierstrass form: y² = x³ + 7
- 128-bit security
- Used in Bitcoin, Ethereum
- Not used by Commonware directly, but is the de facto standard
  in blockchain contexts

**BLS12-381:**
- Pairing-friendly curve, two curves (one for G1, one for G2)
- 128-bit security
- The only curve in this list that supports pairing-based
  cryptography
- Used for signature aggregation and threshold signatures

Commonware uses Curve25519 (Ed25519) for peer authentication and
signing individual messages, and BLS12-381 for consensus
signatures where aggregation or threshold is needed.

### Scalar multiplication

The fundamental operation on an elliptic curve is **scalar
multiplication**: given a point P and a scalar n, compute nP (P
added to itself n times).

The naive algorithm is repeated addition: O(n) operations. But
there is a much faster algorithm, **double-and-add**, which runs
in O(log n) operations:

```
def scalar_mult(n, P):
    result = O  # identity
    addend = P
    while n > 0:
        if n & 1:
            result = result + addend
        addend = addend + addend
        n >>= 1
    return result
```

This is the same algorithm as binary exponentiation in (Z/pZ)*.

### Windowed scalar multiplication

For large scalars, the double-and-add algorithm can be sped up
using **windowing**: precompute small multiples (P, 2P, 3P, ...,
15P) and process the scalar in 4-bit windows. This reduces the
number of point additions at the cost of a small precomputation.

Commonware's `bls12381` uses windowed scalar multiplication with
a window size of 4 or 8 bits, depending on the platform.

### Multi-scalar multiplication

For BLS signature aggregation, you need to compute:

```
R = s₁P₁ + s₂P₂ + ... + sₙPₙ
```

Naively, this is n scalar multiplications, which is O(n log s).
But with **Pippenger's algorithm** (also known as bucket
aggregation), it can be done in roughly O(n / log n) point
operations, which is much faster for large n.

Commonware's BLS12-381 implementation uses Pippenger's algorithm
for multi-scalar multiplication, which is critical for signature
aggregation performance.

### Point compression

A point (x, y) on an elliptic curve can be compressed to just x
plus a single bit indicating the sign of y (since y is
determined up to sign by the curve equation). This reduces the
size from 64 bytes (for a 256-bit curve) to 33 bytes.

Ed25519 public keys are 32 bytes, not 33, because they use a
*y-coordinate* format: the y coordinate plus a sign bit is
sufficient, and the format puts the sign bit in the high bit of
the y coordinate. (This works because the curve equation is
edwards form, where y uniquely determines x up to sign.)

Commonware's `Ed25519::PublicKey` is 32 bytes (the y coordinate
plus a sign bit). Its `Bls12381G1::PublicKey` is 48 bytes (the
x coordinate plus a sign bit, with the x coordinate padded to
46 bytes for field alignment).

### Cofactor and subgroup membership

A cofactor h in a curve of order n = hq (where q is prime) means
there are h points for every point in the prime-order subgroup.
If you don't check subgroup membership, an attacker can give you
points of small order, and your protocols may break.

BLS12-381's G1 has a cofactor of h = (q - 1) / r where r is the
prime subgroup order. The full curve order has many small
subgroups; you must check that a point is in the prime-order
subgroup before using it.

Commonware's BLS12-381 implementation does subgroup membership
checks for all public keys. This is a security-critical check;
skipping it is a known vulnerability class.

### Putting it together

Elliptic curves give us a *group structure* that is:

- Compact (256-bit points for 128-bit security).
- Has a hard DLP (no sub-exponential algorithm known).
- Supports fast operations (constant-time scalar mult).
- Supports pairings (for BLS12-381).

This combination makes elliptic curves the workhorse of modern
public-key cryptography. Commonware uses two curves: Ed25519 for
peer-to-peer signatures and BLS12-381 for consensus signatures.


## Pairings in depth

The main-body section "Pairings explained — the bilinear map"
introduces the basic idea: a pairing is a map e: G1 × G2 → GT
that is bilinear, meaning e(aP, bQ) = e(P, Q)^(ab). This section
goes deeper: the three types of pairings, the Weil and Tate
pairings, the embedding degree, and the algebraic structure of
BLS12-381.

### The bilinear map

Let G1, G2, GT be cyclic groups of prime order r. A **bilinear
map** (pairing) is a function e: G1 × G2 → GT satisfying:

1. **Bilinearity.** For all P ∈ G1, Q ∈ G2, and a, b ∈ Z:
   e(aP, bQ) = e(P, Q)^(ab).

2. **Non-degeneracy.** e(P, Q) ≠ 1 for at least one pair (P, Q).

Bilinearity is the magic property. It says: scalar multiplications
in the source groups become exponentiations in the target group,
and the exponents multiply.

This is why aggregation works. If you have signatures σᵢ = sᵢ * H(mᵢ)
where H is a hash-to-curve and sᵢ is a private key, then:

```
e(g1, Σσᵢ) = e(g1, ΣsᵢH(mᵢ)) = Π e(g1, sᵢH(mᵢ))^... 
```

Actually, the standard aggregation uses a slightly different
construction:

```
Aggregate = Σσᵢ
Verify: e(Aggregate, g2) = e(H(m), Σpkᵢ)
```

This works because of bilinearity.

### The three types of pairings

Pairings are classified by the relationship between G1 and G2:

**Type 1 (symmetric):** G1 = G2. The pairing is symmetric:
e(P, Q) = e(Q, P). Examples: BLS12-381 with the G1 = G2
subgroup, but this isn't actually used because Type 1 curves
are slow.

**Type 2 (asymmetric, G2 ≠ G1):** G1 ≠ G2, but there is an
efficient distortion map φ: G1 → G2. Examples: some older
pairing curves.

**Type 3 (asymmetric, no distortion map):** G1 ≠ G2, and there
is no known efficient map between them. Examples: BLS12-381.

BLS12-381 is Type 3. G1 is over a smaller field (Fp), G2 is
over a larger field (Fp²). The pairing is asymmetric: e(P, Q) ≠
e(Q, P) in general (although the values are related).

Type 3 is preferred for cryptographic security because Type 1
and Type 2 curves have been shown to have weaknesses (the
MOV attack and the Frey-Rück attack reduce the ECDLP to DLP in
a smaller field).

### The embedding degree

For a pairing e: G1 × G2 → GT to exist, the groups G1, G2 must
embed into a larger group GT. The **embedding degree** k is the
smallest integer such that the order r of G1 divides p^k - 1
(where p is the field characteristic of G1's base field).

For BLS12-381, the embedding degree is 12. This means:

- G1 is over Fp (381-bit prime).
- G2 is over Fp² (762-bit prime).
- GT is over Fp^12 (4572-bit prime extension).

The 12 in BLS12-381's name refers to this embedding degree.

### The Weil pairing

The **Weil pairing** is one of the earliest constructions of a
bilinear map on elliptic curves. Given points P and Q of order r
on a curve E, the Weil pairing is:

```
e(P, Q) = (-1)^r * (f_P(Q) / f_Q(P))
```

where f_P and f_Q are certain rational functions determined by P
and Q.

The Weil pairing is symmetric (Type 1) and not particularly
efficient for cryptographic use. It is mainly of theoretical
interest.

### The Tate pairing

The **Tate pairing** is more efficient and is the basis for
most modern pairing-based cryptography. It is defined as:

```
e(P, Q) = f_P(Q)^((p^k - 1)/r)
```

where f_P is a rational function determined by P, and the
exponentiation projects into the r-torsion subgroup of Fp^k.

The Tate pairing is asymmetric: e(P, Q) ≠ e(Q, P) in general.
For Type 3 curves, this is the standard pairing.

BLS12-381 uses a variant of the Tate pairing, optimized using
the **optimal Ate pairing** algorithm, which reduces the number
of field operations.

### BLS12-381: a Type 3 pairing curve

BLS12-381 is defined by:

- A base field Fp where p is the prime:
  ```
  p = 0x1a0111ea397fe69a4b1ba7b6434bad3e0b5c9b2a
        7c6e1f4b3a8b5c0e1e5e5e5e5e5e5e5e5e5e5e5e5e5e5e5e5e5e5e5e5e5e5e5e5
  ```
  (a 381-bit prime)

- The curve equation: y² = x³ + 4 (over Fp).

- An embedding degree of 12.

- A subgroup of prime order r:
  ```
  r = 0x73eda753299d7d483339d80809a1d80553bda402
        fffe5bfeffffffff00000001
  ```
  (~255 bits)

The curve is chosen so that:
- r is prime (so the subgroup is prime-order).
- p is prime (so the base field is a prime field).
- The embedding degree is small (12 is "small enough").
- The security level is ~128 bits.

The "12" in BLS12-381 is the embedding degree; the "381" is the
bit length of the base prime.

### The pairing equation for BLS

The BLS signature scheme uses the pairing as follows:

**Key generation:** Private key sk ∈ Z_r, public key pk = sk *
G1 (a point in G1).

**Signing:** σ = sk * H(m), where H is a hash-to-curve function
mapping messages to points in G2.

**Verification:** e(G1, σ) = e(pk, H(m)).

The verification equation works because:

```
e(G1, σ) = e(G1, sk * H(m)) = e(G1, H(m))^sk
e(pk, H(m)) = e(sk * G1, H(m)) = e(G1, H(m))^sk
```

Both equal e(G1, H(m))^sk, so the equation holds iff σ is a
valid signature.

**Aggregation:** σ_agg = σ_1 + σ_2 + ... + σ_n. Verification:

```
e(G1, σ_agg) = e(pk_1 + pk_2 + ... + pk_n, H(m))
```

This works because of bilinearity:

```
e(G1, σ_agg) = e(G1, Σ sk_i * H(m_i)) = Π e(G1, H(m_i))^sk_i
e(Σ pk_i, H(m_i)) = ...
```

(For distinct messages, the verification is slightly more
involved, but the principle is the same.)

### Why BLS dominates the alt_bn128

Before BLS12-381, the standard pairing curve was BN254 (also
called alt_bn128). BN254 has a 254-bit base field and a 12
embedding degree, but the prime subgroup order is only 254 bits
(~128 bits of security in theory, but less in practice due to
recent improvements).

BLS12-381 is preferred over BN254 because:

1. **Larger prime subgroup.** BLS12-381's r is 255 bits; BN254's
   r is ~254 bits. The difference matters for security margins.

2. **Conservative parameters.** BLS12-381 was designed by Sean
   Bowe and others to be safe against known attacks on pairing
   curves, including recent improvements to the special tower
   number field sieve.

3. **Higher security margin.** With r ≈ 2^255, BLS12-381 has a
   clear security margin over 128 bits, while BN254 has been
   estimated to provide only ~110 bits of security in light of
   recent attacks.

For Commonware, the choice of BLS12-381 over BN254 is about
*future-proofing*. The cost is slightly slower operations, but
the security gain is worth it.

### Why pairings enable signature aggregation

Without pairings, you cannot aggregate BLS signatures because
the verification equation does not factor. With pairings, the
bilinearity lets you "move" the secret key from the signature
to the public key (or vice versa), and the aggregation falls
out.

Other signature schemes (like ECDSA) do not have this property.
You can construct multi-signatures (multiple signatures on the
same message) using more complex protocols, but you cannot
simply *add* signatures together.

This is why BLS is the standard for consensus protocols: each
validator produces a signature, the signatures are aggregated
into one, and the aggregate is broadcast. The bandwidth savings
are enormous: instead of N 96-byte signatures, you have one 96-byte
aggregate.

### Putting it all together

Pairings are a deep algebraic construction. The key idea: there
exists a bilinear map e: G1 × G2 → GT that "multiplies" the
scalars when you multiply the points. This enables BLS signature
aggregation, threshold BLS, and a host of other protocols
(universal accumulators, KZG commitments, SNARKs).

Commonware uses BLS12-381 because it is the most secure and
efficient pairing curve in production today. The cost is
slightly slower than Ed25519, but the gain (aggregation,
threshold) is worth it for consensus.


## Hash functions in depth

The main-body section "Hash functions in depth — SHA-256 vs
SHA-3 vs BLAKE3" introduces the three constructions: Merkle-
Damgård, sponge, and tree. This section goes deeper into each:
the formal definitions, the security arguments, the length-
extension attack, and the role of each construction in Commonware.

### What we need from a hash function

A cryptographic hash function H takes an input of arbitrary
length and produces a fixed-length output (the **digest**). The
three security properties we need:

1. **Preimage resistance.** Given h, find m such that H(m) = h.
   The attacker should need ~2^n operations, where n is the
   digest length.

2. **Second-preimage resistance.** Given m₁, find m₂ ≠ m₁ such
   that H(m₁) = H(m₂). The attacker should need ~2^n operations.

3. **Collision resistance.** Find any m₁ ≠ m₂ such that H(m₁) =
   H(m₂). The attacker should need ~2^(n/2) operations (by the
   birthday paradox).

These are three different properties with three different
security levels. Collision resistance is the strongest and the
hardest to achieve; the birthday attack reduces its effective
security to n/2 bits.

For 128-bit security against collision attacks, you need a
256-bit digest. SHA-256 and BLAKE3 both provide this.

### The Merkle-Damgård construction

SHA-256 is built using the **Merkle-Damgård construction**. The
idea:

1. Pad the input to a multiple of the block size (64 bytes for
   SHA-256).
2. Split the padded input into blocks M₁, M₂, ..., M_n.
3. Initialize an IV (initial vector) h_0.
4. For each block, apply a compression function f:
   ```
   h_i = f(h_{i-1}, M_i)
   ```
5. Output h_n as the digest.

The compression function f is a keyed permutation: it takes the
current state and the next block, and produces the next state.

The construction's security reduces to the compression function:
if f is collision-resistant, then the whole construction is
collision-resistant. The proof is in Merkle's 1979 paper and
Damgård's 1989 paper (hence the name).

**Length-extension attack.** Merkle-Damgård has a known weakness:
given H(m) and len(m), an attacker can compute H(m || padding ||
m') for any m' without knowing m. This is because the
compression function takes the current state as input, so the
attacker can resume the computation from H(m).

To prevent length-extension, SHA-256 uses the **Merkle-Damgård
with strengthening** (also called the **Davies-Meyer**
construction): the IV is set to the digest of the input length,
not a fixed constant. This forces the attacker to know the
length, which is just len(m) (already known), but more
importantly, the finalization step uses a different
transformation.

Wait, that's not quite right. Let me correct: SHA-256 uses the
standard Merkle-Damgård construction, and it IS vulnerable to
length-extension attacks. The fix is to use HMAC, which hashes
the key twice and breaks the length-extension chain.

Commonware uses SHA-256 (via BLAKE3's underlying compression)
but is careful to use it in HMAC-like contexts (transcripts)
where length-extension is not a concern.

### The sponge construction

SHA-3 is built using the **sponge construction**. The idea:

1. The state is `r + c` bits, where r is the rate and c is the
   capacity.
2. The input is absorbed in r-bit blocks: each block is XORed
   into the r-bit part of the state, then a permutation f is
   applied to the entire state.
3. After absorbing, the output is squeezed in r-bit blocks: each
   block is taken from the r-bit part of the state, then f is
   applied.
4. The total output length is arbitrary (you can squeeze as much
   as you want).

The security of the sponge is determined by the capacity c: the
attacker needs ~2^(c/2) operations for a collision (birthday
attack on c bits). For SHA-3-256, c = 512 bits, so collision
resistance is 2^256.

The sponge is *not* vulnerable to length-extension. Once the
input is absorbed, the state depends on all input bits, but the
attacker cannot extend the input without knowing the state. The
sponge is provably secure against length-extension.

This is the main reason SHA-3 was designed: to provide a
hash function that is structurally different from SHA-2, in
case SHA-2 was ever broken. SHA-3 won the NIST hash competition
in 2012, beating out many other candidates on the strength of
its clean security proof.

### The tree construction

BLAKE3 uses the **tree construction**. The idea:

1. The input is split into chunks (1 KB each).
2. Each chunk is hashed using a compression function, with a
   counter indicating its position in the tree.
3. The chunk hashes are combined pairwise using the compression
   function, forming a binary tree.
4. The root of the tree is the digest.

The tree construction enables *parallelism*: multiple chunks can
be hashed simultaneously on multiple cores. This is a major
performance advantage over Merkle-Damgård and sponge, which are
inherently sequential.

BLAKE3 also uses a *flag* byte in the compression function to
distinguish between chunk compression, parent compression, and
finalization. This is the same domain separation idea used in
RFC 9162 (Certificate Transparency) for the 0x00 and 0x01
prefixes in Merkle trees.

### The BLAKE3 compression function

BLAKE3 uses the same compression function as BLAKE2 (its
predecessor), but with a few tweaks:

- 7 rounds instead of 10.
- A "domain separation flag" that distinguishes chunk, parent,
  root, and key derivation modes.
- The full state is 64 bytes (8 × u64), enabling SIMD parallelism.

The compression function operates on a 4×4 matrix of u64 words:

```
| v0  v1  v2  v3  |
| v4  v5  v6  v7  |
| v8  v9  v10 v11 |
| v12 v13 v14 v15 |
```

Each round consists of 8 "G" function applications, where G is a
simple mixing function. The 7 rounds provide sufficient
diffusion for cryptographic security.

### Why Commonware uses BLAKE3 for transcripts

Commonware's transcript module (chapter 04 already covers this)
uses BLAKE3 because:

1. **Speed.** BLAKE3 is ~10× faster than SHA-256 on modern CPUs,
   due to the tree construction and SIMD parallelism.

2. **Streaming.** BLAKE3 supports incremental hashing: you can
   absorb input in chunks and produce the digest at the end. The
   tree structure means the digest is available as soon as all
   chunks are absorbed, without a separate finalization.

3. **Domain separation.** The flag byte allows multiple
   "namespaces" within the same hash function: the digest of
   "consensus" is structurally different from the digest of
   "network", even if the inputs are the same. This is the same
   trick as the 0x00/0x01 prefixes in RFC 9162.

4. **Security margin.** BLAKE3 has not been the subject of
   significant cryptanalysis (it's relatively new, from 2020),
   but its design is conservative and based on well-studied
   primitives.

### Transcript construction

A **transcript** is a structured hash of a sequence of events.
Commonware's transcript module allows you to:

1. Append items of different types (each type has a domain
   separator).
2. Commit to the current state of the transcript (producing a
   digest that commits to all items so far).

The structure is:

```rust
let mut t = Transcript::new(namespace);
t.append(&message_1);
t.append(&message_2);
let digest_1 = t.commit();  // commits to message_1 and message_2
t.append(&message_3);
let digest_2 = t.commit();  // commits to message_1, message_2, and message_3
```

This is the standard pattern for transcript hashing, similar to
the STH (Signed Tree Head) in Certificate Transparency (RFC 9162)
or the Fiat-Shamir transform in zero-knowledge proofs.

### Putting it together

Commonware uses:

- **BLAKE3** for general-purpose hashing, including transcripts.
- **SHA-256** (via BLAKE3's compression) for compatibility with
  external systems that expect SHA-256 digests.
- **Poseidon** (or similar algebraic hashes) for hash-to-curve
  operations in BLS12-381.

The choice of BLAKE3 over SHA-256 is primarily about speed: in
the hot path of a consensus protocol, hashing is a significant
fraction of CPU time, and BLAKE3 is much faster.

The trade-off: BLAKE3 is newer and has less cryptanalysis than
SHA-256. For most applications this is fine, but for
high-security contexts (e.g., signing root keys), SHA-256 may
be preferred for its longer track record.


## BLS12-381 in depth

The main-body section "BLS12-381 — the pairing-friendly
construction" introduces the curve. This section goes deeper:
the curve equation in detail, the embedding degree
justification, the hash-to-curve construction, the multi-scalar
multiplication, and the comparison with other pairing curves.

### The curve equation

BLS12-381 is defined over a prime field Fp where p is:

```
p = 0x1a0111ea397fe69a4b1ba7b6434bad3e0b5c9b2a7c6e1f4b3a8b5c0e1e5e5e5e
```

(a 381-bit prime, chosen for its specific algebraic properties)

The curve is:

```
E: y² = x³ + 4 (mod p)
```

This is the *short Weierstrass form*. The coefficient `a = 0`,
`b = 4`.

The curve has a cofactor of h = (p + 1 - t) / r, where t is the
trace of Frobenius. For BLS12-381, the cofactor in G1 is small
(~2^32 or so), which is acceptable but does require subgroup
membership checks.

### The two groups

BLS12-381 has two source groups:

- **G1**: a subgroup of E(Fp). Points in G1 are 48 bytes when
  compressed (a 381-bit x-coordinate plus a sign bit, padded to
  48 bytes for alignment).

- **G2**: a subgroup of E'(Fp²). Points in G2 are 96 bytes when
  compressed (two Fp elements plus a sign bit).

The target group **GT** is a subgroup of Fp^12*. This is a
4572-bit field (12 × 381 = 4572). Operations in GT are
expensive (~10× slower than G1).

### The embedding degree

The embedding degree k = 12 is the smallest integer such that
r | p^k - 1, where r is the order of G1. This means the r-th
roots of unity exist in Fp^12 but not in any smaller extension.

The pairing e: G1 × G2 → GT maps into the r-torsion subgroup of
Fp^12*. This subgroup has size r, which is prime.

The choice of k = 12 is a trade-off:

- Smaller k (like 6 or 4) would give faster pairings but
  weaker security (the MOV attack becomes more effective).
- Larger k (like 24 or 48) would give stronger security but
  slower pairings.

For BLS12-381, k = 12 was chosen as the sweet spot: pairings
are fast enough for production use, and the security level is
~128 bits.

### Why these specific parameters

The prime p was chosen by Sean Bowe (and others) for several
properties:

1. **The prime has a special form.** p ≡ 1 (mod 6) and
   p ≡ 1 (mod 2^32), which enables fast modular arithmetic
   using Montgomery multiplication.

2. **The curve order has a specific structure.** The order of
   E(Fp) is `p + 1 - t`, where t is the trace of Frobenius. For
   BLS12-381, the order factors as h * r where h is a small
   cofactor and r is a 255-bit prime.

3. **The embedding degree is 12.** This enables efficient
   pairings while maintaining ~128 bits of security.

The choice is not arbitrary; it is the result of a search over
many possible primes to find one with all these properties.

### The hash-to-curve

For BLS signatures, messages are hashed to points in G2. The
standard construction is **hash-to-curve** (RFC 9380), which
takes an arbitrary byte string and produces a point on the
curve.

The construction works by:

1. Hashing the message to a field element (using a wide hash,
   to ensure uniform distribution).
2. Using the **SSWU** (Simplified Shallue-van de Woestijne-
   Ulas) method to map field elements to curve points.
3. Clearing the cofactor (multiplying by the inverse of h) to
   ensure the result is in the prime-order subgroup.

This is more complex than hashing to a field element, because
not every field element is a valid x-coordinate on the curve.
The SSWU method gives a deterministic mapping that is
collision-resistant.

Commonware uses the standard hash-to-curve construction for
BLS12-381 G2.

### The pairing equation in code

In Commonware, a BLS signature verification looks like:

```rust
let sig = Signature::deserialize(&sig_bytes)?;
let pk = PublicKey::deserialize(&pk_bytes)?;
let message = b"some message";
let message_hash = hash_to_curve(message);

if !verify(&pk, &message_hash, &sig) {
    return Err(Error::InvalidSignature);
}
```

The `verify` function computes:

```
e(G1::generator(), sig) == e(pk, message_hash)
```

Both sides of the equation are computed in GT. The pairing is
implemented using the optimal Ate pairing algorithm.

### Multi-scalar multiplication

For signature aggregation, you need to compute:

```
agg_sig = Σ s_i * H(m_i)
```

or equivalently, when verifying:

```
agg_pk = Σ pk_i
e(G1, agg_sig) == e(agg_pk, H(m))
```

The bottleneck is the multi-scalar multiplication (MSM). For N
signatures, the naive algorithm is N scalar multiplications,
each O(log r) point operations, so O(N log r) total.

**Pippenger's algorithm** reduces this to O(N / log N) point
operations, by bucket-aggregation:

1. Split each scalar into windows of w bits.
2. For each window position, bucket the points by their window
   value (0 to 2^w - 1).
3. Sum each bucket's points.
4. Sum the bucket sums (with appropriate shifts) to get the
   result.

For w = 4 (16 buckets), this is roughly 4× faster than naive.
For w = 8 (256 buckets), it's roughly 8× faster. The trade-off
is memory: more buckets means more memory.

Commonware's BLS12-381 implementation uses Pippenger's algorithm
with a window size chosen at runtime based on the input size.

### Why BLS dominates alt_bn128

The earlier pairing curve, BN254 (also called alt_bn128), has
the same embedding degree (12) but a smaller prime (254 bits
instead of 381). The security level of BN254 has been revised
downward over the years as attacks have improved:

- In 2010, the original analysis estimated BN254 at ~128 bits.
- By 2016, improvements to the special tower number field sieve
  reduced the estimate to ~110 bits.
- Recent work suggests BN254 may provide only ~100 bits of
  effective security.

BLS12-381, designed in 2017, was built to provide a clear
~128-bit security margin, with parameters chosen to be robust
against known attacks. The slightly slower operations are
acceptable for the security gain.

This is why Commonware uses BLS12-381 over BN254.

### The Zexe and zk-SNARK context

BLS12-381 is the pairing curve used in **Zexe** and many
zk-SNARK constructions. The reason: zk-SNARKs need pairings to
verify polynomial commitments (KZG commitments use pairings),
and BLS12-381 is the most efficient pairing curve with ~128-bit
security.

If Commonware ever adds zk-SNARK support (e.g., for light
clients or private transactions), it will use BLS12-381 to
maintain consistency with the existing cryptography.

### Putting it together

BLS12-381 is the result of careful parameter selection to
provide:

- ~128 bits of security.
- Efficient pairings (k = 12).
- A prime-order subgroup (no cofactor issues in the prime
  subgroup).
- Compatibility with zk-SNARKs.

The trade-offs (slower than Ed25519 for non-pairing operations,
more complex hash-to-curve) are acceptable for the consensus
protocol use case, where the aggregation property is
indispensable.


## Threshold cryptography in depth

The main-body sections "Shamir's secret sharing" and "Threshold
BLS — Lagrange in the exponent" cover the basics. This section
goes deeper: the formal security model, the DKG construction,
Feldman and Pedersen VSS, and the threshold BLS reconstruction
in full detail.

### Shamir's secret sharing — the full math

Shamir's secret sharing lets you split a secret s into n shares
s_1, s_2, ..., s_n such that:

- Any t shares can reconstruct s.
- Any t-1 shares reveal nothing about s.

The construction:

1. Choose a random polynomial f(x) of degree t - 1 with
   f(0) = s.
2. Distribute shares s_i = f(i) for i = 1, ..., n.

Reconstruction uses Lagrange interpolation:

```
s = f(0) = Σ s_i * L_i(0)
```

where L_i(x) is the i-th Lagrange basis polynomial:

```
L_i(x) = Π_{j ≠ i} (x - j) / (i - j)
```

At x = 0:

```
L_i(0) = Π_{j ≠ i} (0 - j) / (i - j) = Π_{j ≠ i} (-j) / (i - j)
```

This is the standard Lagrange interpolation formula.

### Security of Shamir's secret sharing

The security argument: given t-1 shares, you have t-1 points on
the polynomial f(x). A polynomial of degree t-1 is determined by
t points, so t-1 points leave one degree of freedom. The secret
s = f(0) is then uniformly distributed over all possible values,
independent of the t-1 shares.

Formally: if the polynomial's coefficients are sampled uniformly
at random from a finite field, and you know t-1 evaluations, the
remaining evaluation f(0) is independent of the t-1 values and
uniformly distributed. This is information-theoretic security:
no amount of computation can recover s from t-1 shares.

### The DKG twist

A DKG (Distributed Key Generation) protocol produces a Shamir
secret sharing *without* a trusted dealer. The standard
construction:

1. Each participant i samples a random polynomial f_i(x) of
   degree t - 1 with f_i(0) = s_i (their own secret share).
2. Each participant i broadcasts Feldman commitments to f_i:
   ```
   C_{i,j} = g^{f_{i,j}}  for each coefficient f_{i,j}
   ```
3. Each participant i sends f_i(j) to participant j privately
   (encrypted or using a secure channel).
4. Each participant j verifies that f_i(j) is consistent with
   the Feldman commitments.
5. Each participant j's share of the joint secret is
   s_j = Σ_i f_i(j).
6. The joint public key is Σ_i g^{s_i} = g^Σ s_i.

The result is a (t, n) Shamir sharing of the secret s = Σ s_i,
where no single participant knows s, but any t participants can
reconstruct it via Lagrange interpolation.

### Feldman VSS

**Feldman's Verifiable Secret Sharing (VSS)** lets participants
verify that the dealer distributed correct shares, without
revealing the shares themselves. The dealer:

1. Samples a polynomial f(x) of degree t - 1 with f(0) = s.
2. Publishes commitments C_j = g^{f_j} for each coefficient
   f_j.
3. Sends share s_i = f(i) to participant i privately.

Each participant verifies:

```
g^{s_i} = Π_j C_j^{i^j}
```

If the verification passes, the share is consistent with the
public commitments.

Feldman VSS assumes a private channel between the dealer and
each participant. Pedersen's VSS removes this assumption.

### Pedersen VSS

**Pedersen's VSS** uses two commitments to each coefficient:

1. The dealer samples two random polynomials f(x) and f'(x) of
   degree t - 1.
2. Publishes commitments C_j = g^{f_j} h^{f'_j} for each
   coefficient.
3. Sends share s_i = (f(i), f'(i)) to participant i.

Verification is similar to Feldman, but uses both commitments.
The advantage: Pedersen VSS achieves *unconditional* privacy
(the shares are information-theoretically hidden), while Feldman
VSS has computational privacy (the shares are hidden under the
DLP).

Commonware's DKG uses Pedersen-style commitments for stronger
privacy.

### The threshold BLS reconstruction

In threshold BLS, the secret key sk is shared via Shamir's
scheme among n participants, with threshold t. Each participant
i has a share sk_i.

To sign a message m, each participant produces a partial
signature:

```
σ_i = sk_i * H(m)
```

To reconstruct the full signature from t partial signatures:

```
σ = Σ λ_i * σ_i
```

where λ_i is the Lagrange coefficient at x = 0:

```
λ_i = Π_{j ≠ i} (0 - j) / (i - j)
```

Verification is the same as for regular BLS:

```
e(G1, σ) == e(pk, H(m))
```

where pk is the joint public key (Σ pk_i, where pk_i = sk_i * G1).

### Why this works

The reconstruction works because of linearity in the exponent:

```
σ = Σ λ_i * σ_i = Σ λ_i * sk_i * H(m)
            = (Σ λ_i * sk_i) * H(m)   [by distributivity]
            = sk * H(m)                [by Shamir reconstruction]
```

So σ is exactly the BLS signature that would have been produced
by the (hypothetical) full secret key sk.

### Adaptive vs static adversary

A **static adversary** chooses which participants to corrupt at
the start of the protocol. An **adaptive adversary** can corrupt
participants dynamically during the protocol.

Pedersen VSS is secure against static adversaries. To handle
adaptive adversaries, you need stronger tools: non-committing
encryption, or proactive secret sharing.

Commonware's DKG is secure against static adversaries with up
to t - 1 corruptions (where t is the threshold). This is the
standard security model for DKG.

### Putting it together

Threshold BLS is the workhorse of consensus:

- The signing key is shared among n validators.
- Any t can sign (no single point of failure).
- The signature is a single BLS signature (96 bytes).
- Verification is the same as for non-threshold BLS.

The trade-off: DKG is complex and requires multiple rounds of
communication. But for a long-running consensus protocol, the
operational simplicity (one signing key, no key rotation ceremony)
is worth the upfront complexity.


## Cryptographic implementation pitfalls

The main-body section "Rust pattern — Zeroize for key material"
covers the basics: zeroize keys, use subtle for constant-time
operations. This section goes deeper into the *pitfalls* that
turn "secure on paper" into "insecure in code."

### Timing attacks

A **timing attack** is a side-channel attack where the attacker
measures the time taken by an operation to learn a secret.

Classic example: comparing a MAC. The naive code:

```rust
if computed_mac == expected_mac {
    Ok(())
} else {
    Err(Error::InvalidMac)
}
```

This is **not** constant time. The `==` operator on byte slices
(short-circuits) returns early when the first mismatched byte is
found. The attacker can measure how long the comparison took
and learn *which* byte failed, gradually reconstructing the
expected_mac byte by byte.

The fix is `subtle::ConstantTimeEq`:

```rust
use subtle::ConstantTimeEq;

if computed_mac.ct_eq(&expected_mac).into() {
    Ok(())
} else {
    Err(Error::InvalidMac)
}
```

The `ct_eq` runs in constant time regardless of the input.

This is not a theoretical concern. Remote timing attacks have
been demonstrated against TLS implementations (e.g., the
Lucky 13 attack on CBC-mode ciphers). Constant-time code is
not optional.

### Side-channel attacks

A **side-channel attack** uses information leaked from the
physical implementation of a cryptographic operation, not the
algorithm itself. Common side channels:

- **Timing.** As above. The most common.
- **Power consumption.** An attacker with physical access can
  measure the power drawn by a smart card and recover keys.
- **Electromagnetic radiation.** Same idea, different physical
  quantity.
- **Cache.** CPU caches leak information via access patterns.
  Flush+Reload attacks have recovered RSA keys from co-located
  processes in cloud environments.
- **Acoustic.** The high-pitched whine of a CPU can leak
  information under laboratory conditions.

Commonware's runtime environment is a server, not a smart card
or an embedded device, so power/EM/acoustic attacks are not
relevant. Timing and cache attacks are: Commonware uses
constant-time scalar multiplication (Montgomery ladder for
Curve25519) and avoids data-dependent memory accesses in
critical paths.

### Constant-time scalar multiplication

The scalar multiplication `n * P` on an elliptic curve must be
constant-time in `n`. Otherwise, the attacker can recover `n` by
measuring which branches are taken.

The standard algorithm (double-and-add) is **not** constant-time:
the `if n & 1 { result += addend }` branch depends on `n`.

The constant-time version (Montgomery ladder) processes every
bit:

```rust
let mut r0 = O;  // accumulator
let mut r1 = P;  // running doubled
for i in (0..256).rev() {
    if (n >> i) & 1 == 1 {
        r0 = r0 + r1;
        r1 = r1 + r1;
    } else {
        r1 = r0 + r1;
        r0 = r0 + r0;
    }
}
```

This is constant-time: every iteration does the same operations,
regardless of the bit.

Commonware's BLS12-381 implementation uses constant-time
algorithms. The `blst` library (which Commonware uses for
BLS12-381) is constant-time.

### The `subtle` crate

The `subtle` crate provides primitives for constant-time
operations:

- `Choice`: a "boolean" type that defers evaluation.
- `ConstantTimeEq`: `ct_eq` for byte slices.
- `ConstantTimeGreater`: `ct_gt` for integers.
- `ConditionallySelectable`: select between two values based on a
  `Choice`.

The pattern is:

```rust
use subtle::{Choice, ConstantTimeEq, ConditionallySelectable};

let computed: [u8; 32] = ...;
let expected: [u8; 32] = ...;
let equal: Choice = computed.ct_eq(&expected);

let result: u8 = if equal.into() { 1 } else { 0 };
```

The `ct_eq` returns a `Choice` (not a `bool`) so the compiler
cannot optimize the branch away. The `into()` extracts the
boolean value at the very end.

### What's constant-time, what's not

Common operations in Commonware's cryptography:

- **Constant-time:** MAC comparison, point decompression,
  BLS signature verification, hash comparison, signature
  aggregation checks.
- **Not constant-time:** Hashing (SHA-256, BLAKE3), public-key
  serialization, generic comparisons on public types.

The "not constant-time" operations are OK because they don't
involve secrets. The "constant-time" operations are critical.

### Secure on paper vs secure in code

A cryptographic algorithm can be *secure on paper* (the math is
correct) but *insecure in code* (the implementation leaks via
side channels).

Examples of "secure on paper, insecure in code":

- OpenSSL's Heartbleed (2014): a buffer over-read leaked private
  keys. The TLS protocol was secure; the implementation wasn't.
- RSA with Chinese Remainder Theorem: CRT speedup can leak via
  fault attacks (one bit of a CRT intermediate value, observed
  via timing or power).
- AES T-tables: a software optimization using lookup tables
  that leak cache timing information.

The lesson: every cryptographic implementation must be
*audited* for side channels, not just correctness. Commonware's
cryptography uses audited libraries (blst for BLS12-381, ed25519
for Ed25519) and writes custom code carefully.

### Bad randomness

If the random number generator is biased or predictable,
cryptographic operations become insecure:

- Predictable ECDSA nonces (k values) leak the private key. (See
  the 2010 Sony PS3 break: Sony used a static `k` value, allowing
  the private key to be recovered from any two signatures.)
- Biased DSA nonces: even a 1-bit bias can leak the key after
  many signatures.
- Reused nonces: if the same `k` is used for two different
  messages, the private key can be recovered.

Commonware's RNG is ChaCha20-based (covered in the main body),
seeded from the OS CSPRNG (`OsRng`). It is deterministic for
testing (using a seeded `test_rng()`) but uses real CSPRNG
randomness in production.

### The pitfalls checklist

When implementing cryptographic code:

1. **Use audited libraries.** Don't roll your own crypto.
2. **Use `subtle` for any comparison involving a secret.**
3. **Use `Zeroizing` or `Secret<T>` for any secret material.**
4. **Use `OsRng` for any randomness.** Never seed with
   `SystemTime::now()` or other low-entropy sources.
5. **Use constant-time algorithms for all secret-dependent
   operations.** Montgomery ladder for scalar mult, `subtle::ct_eq`
   for comparisons.
6. **Verify subgroup membership for any received point.** Not
   just any curve point — a point in the prime-order subgroup.
7. **Check signature malleability.** Some signature schemes have
   multiple valid encodings of the same signature.
8. **Use unique nonces.** For ECDSA, generate nonces
   deterministically (RFC 6979) to avoid reuse.

Commonware follows all these rules. But the rules are worth
knowing because cryptographic code review is unforgiving: a
single bug can compromise the entire system.


## Exercises

These exercises build hands-on familiarity with the cryptography.
They are ordered roughly by increasing depth.

### Exercise 1: Implement modular exponentiation

Write a function `mod_exp(base: u64, exp: u64, modulus: u64) -> u64`
that computes `base^exp mod modulus` using the square-and-multiply
algorithm. Test it against `(base as u128).pow(exp as u32) as u64 %
modulus` for correctness.

### Exercise 2: Verify Fermat's little theorem

Pick a prime p (say, p = 101). For each a in {1, ..., p-1}, verify
that `a^(p-1) ≡ 1 (mod p)`. Use your `mod_exp` from Exercise 1.

### Exercise 3: Generate a BLS key pair

Use Commonware's `bls12381` module to generate a key pair. Sign a
message. Verify the signature. Inspect the sizes: private key is
32 bytes, public key is 48 bytes, signature is 96 bytes.

### Exercise 4: Aggregate BLS signatures

Generate 10 key pairs. Have each sign the same message. Aggregate
the signatures using the `aggregate` function. Verify the
aggregate against the aggregate public key.

### Exercise 5: Threshold BLS signing

Run a DKG ceremony (or simulate it by manually distributing
shares). Sign a message with t-of-n threshold. Reconstruct the
signature from any t partial signatures. Verify it against the
joint public key.
## Appendix A — BLS12-381 math, in enough detail to understand the code

BLS12-381 is a **pairing-friendly elliptic curve**. The math is non-trivial,
but you only need to understand a few concepts.

### The two groups

BLS12-381 has two cyclic groups, `G1` and `G2`, plus a third group `GT`
(the "target"). Each is a set of points on a curve, with an addition
operation.

Key property: there's a **bilinear pairing** `e: G1 × G2 → GT`. This
pairing has a magical property:

```
e(g1^a, g2^b) = e(g1, g2)^(a*b)
```

Where `g1 ∈ G1`, `g2 ∈ G2`, `a` and `b` are scalars. The pairing is
**bilinear**: exponents multiply across the pairing.

### BLS signatures

A BLS signature is a curve point. Specifically:

```
sk = secret key (random scalar)
pk = sk * G2 = public key (a point in G2)

sign(sk, msg):
    h = H(msg)    # hash msg to a point in G1
    sig = sk * h  # signature is a point in G1

verify(pk, msg, sig):
    h = H(msg)
    assert e(sig, G2) == e(h, pk)
```

The verification works because:

```
e(sig, G2) = e(sk * h, G2) = e(h, G2)^sk
e(h, pk)   = e(h, sk * G2) = e(h, G2)^sk   ✓
```

### Why aggregation works

Now the magic: BLS signatures are **points**. Adding two points is a point.

```
sig_a + sig_b = sk_a * h_msg + sk_b * h_msg = (sk_a + sk_b) * h_msg
```

So if Alice and Bob both sign the same message `msg`:

```
aggregated_sig = sig_a + sig_b          # a single G1 point
aggregated_pk  = pk_a + pk_b           # a single G2 point

verify(aggregated_pk, msg, aggregated_sig):
    e(aggregated_sig, G2) = e((sk_a + sk_b) * h_msg, G2) = e(h_msg, G2)^(sk_a + sk_b)
    e(h_msg, aggregated_pk) = e(h_msg, (sk_a + pk_b))        = e(h_msg, G2)^(sk_a + sk_b)   ✓
```

So `2f+1` individual signatures → one aggregated signature, one
verification. **That's the speedup.** Same math, just linearly
composed.

### The two BLS12-381 variants

Commonware supports both:
- **`MinSig`** — signatures in `G1`, public keys in `G2`. Smaller sigs
  (48 bytes), larger keys (96 bytes).
- **`MinPk`** — signatures in `G2`, public keys in `G1`. Larger sigs
  (96 bytes), smaller keys (48 bytes).

Choose based on what's more important — saving signature bytes
(MinSig, common for blockchain transactions) or key bytes (MinPk,
common for validator identity).

The cryptography crate (`cryptography/src/bls12381/primitives/variant.rs`)
defines this:

```rust
pub trait Variant {
    type PublicKey: Group;
    type Signature: Group;
    // ... pairing constants ...
}
```

`MinPk` and `MinSig` are two impls of this trait.

### The hash-to-curve

To sign a message, you need `h = H(msg)` to be a point in `G1` (or `G2`
for MinSig). This is **not** a normal hash function. It's a
**hash-to-curve** function that produces a valid curve point.

Commonware uses RFC 9380 (hash-to-curve standard). The signature scheme
also incorporates the **namespace** into the hash:

```rust
fn sign(&self, namespace: &[u8], msg: &[u8]) -> Signature {
    // Hash to curve: namespace || msg → G1 point
    let h = hash_to_curve(namespace, msg);
    self.sk * h
}
```

The `namespace` prevents cross-protocol attacks (chapter 04 main text).

### Implementation: `blst`

Commonware uses the `blst` crate (Supranational). It's the fastest
audited BLS12-381 implementation. The wrapper layer in
`cryptography/src/bls12381/scheme.rs` and `primitives/ops.rs` provides
the trait surface.

## Appendix B — Threshold BLS, in depth

Plain BLS gives you aggregation but **every signer holds a full key**.
Threshold BLS is different: **no one holds the full key**.

### The setup: a shared secret via DKG

`n` parties run a Distributed Key Generation (DKG) protocol. At the
end, each party `i` holds a share `sk_i` of a master secret `S`. **No one
knows S.** The master public key `PK = S * G2` is publicly known.

This is achieved using **Shamir Secret Sharing** over an elliptic curve:

```
Polynomial f(x) = a_0 + a_1*x + a_2*x^2 + ... + a_(t-1)*x^(t-1)

f(0) = a_0 = S         (the master secret, never directly revealed)
f(i) = sk_i             (party i's share)

PK = S * G2             (master public key)
pk_i = sk_i * G2         (party i's public verification key)
```

The polynomial is of degree `t-1` (where `t` is the threshold). Any `t`
shares can reconstruct `S` via Lagrange interpolation. Any `t-1` shares
reveal nothing.

### Signing with a share

Party `i` produces a partial signature:

```
partial_sig_i = sk_i * h(msg)         # their share of the sig
```

Verify `partial_sig_i` against `pk_i` (their individual public key):

```
e(partial_sig_i, G2) == e(h(msg), pk_i)
```

### Reconstruction with `t` partials

Given `t` partial signatures `partial_sig_i`, reconstruct the full
signature:

```
full_sig = Σ (L_i * partial_sig_i)    where L_i = Lagrange coefficient at x=i
```

The Lagrange coefficient for share `i` in a `t-1`-degree polynomial:

```
L_i = Π (x_j / (x_j - x_i))     for j ≠ i, x_0...x_(t-1) being t known x-coords
```

For consensus, the x-coordinates are usually `0, 1, 2, ..., t-1`
(party indices), which simplifies the formula but doesn't change the
idea.

### Verify with the static group public key

```
e(full_sig, G2) == e(h(msg), PK)
```

No knowledge of who signed — `full_sig` looks the same regardless of
which `t` partials were combined. **Non-attributable.**

### The trade-off in full

**Plain BLS**: each party has a real key. Signatures attribute to signers.
DKG needed only to refresh keys (e.g., on validator set change).

**Threshold BLS**: no party has the real key. Signatures don't attribute.
DKG needed at setup (mandatory). Resharing needed on validator set
change.

Plain BLS: certificate size = N individual sigs (or 1 multisig).
Threshold BLS: certificate size = 1 sig, ~96 bytes.

For 100-validator committees, threshold saves ~6 KB per certificate.
For 1000-validator, it's 64 KB.

## Appendix C — The DKG protocols, in detail

`cryptography/src/bls12381/dkg/`. Two implementations.

### Feldman-Dersmedt (synchronous, 2-round)

The classic. Each of the `n` parties acts as a "dealer" in turn (or in
parallel if you allow it).

Round 1 — each dealer `d` broadcasts:
- Polynomial commitment: `C_d(x) = c_{d,0} + c_{d,1}*x + ... + c_{d,t-1}*x^(t-1)`
  where `c_{d,k} = a_{d,k} * G2` (commitment to polynomial coefficient).
- Encrypted share for party `i`: `share_{d→i} = a_d(i)` (scalar).

Round 2 — each party `i`:
- Verifies its share `share_{d→i}` against dealer `d`'s commitment:
  `share_{d→i} * G2 == C_d(i)` (the polynomial evaluated at `i`).
- If valid, keeps the share.
- If invalid, broadcasts a complaint.

After all rounds: each party has shares from each dealer. Master secret
`S = Σ (dealer_d's constant term)`. Each party's combined share:

```
sk_i = Σ share_{d→i}     for d in 1..=n
```

The synchronous assumption: bounded dealer-to-party message delay. If
a dealer's messages don't arrive in time, the protocol stalls.

### Golden (asynchronous, 1-round)

Novel to Commonware. `docs/blogs/golden.html` (May 2026):

> We've completed an initial implementation of the Golden protocol,
> which lets you perform a DKG in just a single round.

How does a 1-round DKG work without synchrony? It uses **public-key
encryption** per share:

1. Each party `i` publishes an encryption public key `e_pk_i` (e.g.,
   ECIES on BLS12-381 G1).
2. Each dealer `d` encrypts `share_{d→i}` to `e_pk_i` for each `i`.
3. Dealer broadcasts encrypted shares + polynomial commitments.
4. Each party decrypts its shares using its private key.
5. Zero-knowledge proofs (ZKPs) attached to the encrypted shares prove
   they were correctly encrypted to the right public key.

Cost: more crypto (ECIES, ZKPs) per round. Benefit: no synchrony
assumption.

For Commonware's target use case (Byzantine consensus with potentially
adversarial network timing), Golden is the safer choice.

## Appendix D — Threshold reconstruction in code

`cryptography/src/bls12381/primitives/ops.rs`. The signature is computed
and verified like:

```rust
pub fn sign_message<V: Variant>(sk: &Private, namespace: &[u8], msg: &[u8]) -> Signature {
    let h = hash_to_curve::<V>(namespace, msg);    // h ∈ G1
    sk.0 * h                                      // scalar mult → G1 point
}

pub fn verify_message<V: Variant>(
    pk: &PublicKey,        // G2 point
    namespace: &[u8], msg: &[u8], sig: &Signature
) -> bool {
    let h = hash_to_curve::<V>(namespace, msg);
    pairing(sig, G2_GENERATOR) == pairing(h, pk)
}
```

For threshold, the DKG output is a set of `(share_index, share_scalar)`
pairs. Signing is the same — each share is a valid scalar. Verification
works against the share's individual public key.

For threshold **verification** of a reconstructed full sig, you'd use
the static group public key (derived from all commitments).

The implementation handles both schemes via the `Scheme` trait in
`cert.rs`.

## Appendix E — Timelock Encryption (TLE) in depth

`cryptography/src/bls12381/tle.rs:1-30`:

```
1. Generating a random sigma value
2. Deriving encryption randomness r = H3(sigma || message)
3. Computing the ciphertext components:
   - U = r * G (commitment in G1)
   - V = sigma ⊕ H2(e(P_pub, Q_id)^r) (masked random value)
   - W = M ⊕ H4(sigma) (masked message)

Where Q_id = H1(target) maps the target to a point in G2.
```

The scheme is Boneh-Franklin Identity-Based Encryption. The
"identity" is the target (e.g., round number). The master secret key
(known to the threshold committee) can produce a signature on the
target, which is then used as the decryption key.

The flow for sealed-bid auctions:

1. **Setup**: threshold committee generates master secret `S`, public key
   `PK`.
2. **Bidding phase**: bidders encrypt bids to `target = auction_end`.
3. **End phase**: committee produces a signature on `auction_end`.
4. **Reveal phase**: anyone with the signature can derive the decryption
   key and reveal all bids.

The `sigma` value is included in the ciphertext so the bidder can prove
"this bid was encrypted with this sigma" — useful for disputes.

### CCA security

The scheme is CPA-secure (chosen-plaintext attack). Commonware wraps it
with the **Fujisaki-Okamoto transform** to get CCA security (chosen-
ciphertext attack):

```
encryption randomness r is derived from (sigma || message) deterministically
+ integrity check on the ciphertext
```

This makes the scheme safe against active attackers who can submit
malformed ciphertexts to learn information.

## Appendix F — Transcript internals

`cryptography/src/transcript.rs:1-100`. The `Transcript` API:

```rust
pub struct Transcript {
    inner: blake3::Hasher,
    tag: StartTag,    // New=0, Resume=1, Fork=2, Noise=3
    pending: u64,     // bytes pending flush
}
```

Operations:

- **`new(namespace)`** — start a new transcript with a domain
  separator.
- **`append(&mut self, label: &[u8], data: &[u8])`** — commit to data
  with a labeled slot. Labels prevent ambiguity ("which 'hash' did I
  mean?").
- **`commit(&mut self)`** — finalize the current state. After commit,
  you must call `new()` or `fork()` to keep using the transcript.
- **`fork(&mut self) -> Transcript`** — split into an independent
  child. Used for sub-protocols that need their own transcript state.
- **`noise(&mut self) -> impl CryptoRng`** — deterministically extract a
  `CryptoRng` from the committed state. Useful for Fiat-Shamir and
  deterministic-but-secure randomness.
- **`read(&mut self) -> [u8; N]`** — read bytes (for domain-separated
  values).

The `tag` field ensures different start modes get different hashes even
if you commit identical data. Domain separation at the type level.

The `flush` mechanism (line 84):

```rust
fn flush(hasher: &mut blake3::Hasher, pending: u64) {
    let mut pending_bytes = [0u8; 9];
    let pending = UInt(pending);
    pending.write(&mut &mut pending_bytes[..]);
    hasher.update(&pending_bytes[..pending.encode_size()]);
}
```

When you commit, pending bytes are length-prefixed into the hash. This
prevents length-extension attacks where "abc" + commit could be confused
with "ab" + commit + "c" + commit.

## Appendix G — The Certificate Scheme trait

`cert.rs:189-264`. The most important trait in the crypto crate:

```rust
pub trait Verifier: Clone + Debug + Send + Sync + 'static {
    type Subject<'a, D: Digest>: Subject;
    type PublicKey: PublicKey;
    type Certificate: ... Codec;

    fn verify_certificate<R, D, M>(
        &self,
        verifier_message: M,
        subject: Self::Subject<'_, D>,
        certificate: &Self::Certificate,
    ) -> Result<Verification<Self>, ...>;
}
```

This is what makes consensus **scheme-agnostic**. Simplex (chapter 11)
takes a `Scheme: certificate::Scheme` parameter. Plug in
`ed25519::Scheme`, `bls12381_multisig::Scheme`, `bls12381_threshold::Scheme`,
or your own.

The `Subject` trait (`cert.rs:174-183`):

```rust
pub trait Subject: Clone + Debug + Send + Sync {
    type Namespace: Namespace;
    fn namespace<'a>(&self, derived: &'a Self::Namespace) -> &'a [u8];
    fn message(&self) -> Bytes;
}
```

A `Subject` is "something you sign a certificate over." For Simplex:

- `Notarization` subject: the proposal (block payload + parent).
- `Nullification` subject: just the view number.
- `Finalization` subject: the proposal.

Each has its own `Namespace` so the same signature doesn't validate for
multiple subject types.

## Appendix H — Common patterns in the cryptography crate

### Determinism in tests

`commonware-utils::test_rng()` (chapter 04 main text) is the standard way
to get a deterministic RNG in tests. For the cryptography crate, use
the seeded `ChaCha20Rng`:

```rust
let mut rng = ChaCha20Rng::seed_from_u64(42);
let sk = PrivateKey::random(&mut rng);   // deterministic for seed 42
```

### Zeroization

`cryptography/src/secret.rs`. The `Secret<T>` type wraps key material:

```rust
pub struct Secret<T: Zeroize>(Zeroizing<T>);

impl<T: Zeroize + Drop> Drop for Secret<T> {
    fn drop(&mut self) {
        self.0.zeroize();    // explicit memory wipe
    }
}
```

When a `Secret<PrivateKey>` goes out of scope, the key material is
overwritten with zeros before being freed. Defends against memory dumps,
swap files, and cold-boot attacks.

### The `Display` and `FromStr` for keys

Keys are encoded as **hex strings** by default (via `commonware-formatting`):

```rust
let sk = PrivateKey::from_seed(42);
println!("{}", sk);      // 32-byte hex representation
let sk2: PrivateKey = "abc123...".parse()?;
```

Used everywhere — CLI args, config files, logs.

## Where to look in the code (expanded)

- `cryptography/src/lib.rs:82-200` — `Signer`, `Verifier`, `PublicKey`,
  `Signature`, `BatchVerifier`.
- `cryptography/src/certificate.rs:11-58` — the four schemes compared.
- `cryptography/src/certificate.rs:189-264` — the `Verifier` trait.
- `cryptography/src/bls12381/scheme.rs` — BLS12-381 implementation.
- `cryptography/src/bls12381/primitives/ops.rs` — pairing, hashing,
  signing primitives.
- `cryptography/src/bls12381/primitives/variant.rs` — MinPk vs MinSig.
- `cryptography/src/bls12381/dkg/feldman_desmedt.rs` — synchronous DKG.
- `cryptography/src/bls12381/dkg/golden/` — 1-round asynchronous DKG.
- `cryptography/src/bls12381/tle.rs` — timelock encryption.
- `cryptography/src/transcript.rs` — the transcript abstraction.
- `cryptography/src/ed25519/` — the ed25519 implementation.
- `cryptography/src/secp256r1/` — the secp256r1 implementation.
- `cryptography/src/sha256/` — SHA-256.
- `cryptography/src/blake3/` — Blake3.
- `cryptography/src/secret.rs` — zeroization.

## If you only remember three things

1. **BLS12-381 aggregates.** Ed25519 doesn't. That's the entire reason BLS exists in consensus — `2f+1` votes collapse to one signature.
2. **Threshold = non-attributable.** With threshold BLS, any `t` signers can forge any missing partial. You can prove "a quorum signed" but not "Alice signed." Matters for slashing.
3. **Namespaces prevent cross-protocol attacks.** Always sign with a namespace. Always verify with the same namespace. The cost is zero; the safety is real.

→ Next: **Chapter 05 — P2P**. Now we know how to sign things. But how do you
ship those signed things to specific peers over an authenticated, encrypted
channel? That's the P2P layer.