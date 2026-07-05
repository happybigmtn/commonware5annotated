# Chapter 13 — Aggregation: quorum certificates over an external sequencer

> The same "collect 2f+1 signatures" pattern, but for any ordered stream of items.

## The problem

You have an **external sequence** of items — maybe transaction batches, maybe
state roots, maybe block hashes from another chain. You want validators to
**collectively sign each item** so that an external observer can prove "this
many validators agreed on item N."

This is different from Simplex:
- Simplex **orders** messages from a leader.
- Aggregation **certifies** items that are already ordered by something else.

Use cases (from `aggregation/mod.rs:8-12`):

> The primary use case for this primitive is to allow blockchain validators
> to agree on a series of state roots emitted from an opaque consensus
> process. Because some chains may finalize transaction data but not the
> output of said transactions during consensus, agreement must be achieved
> asynchronously over the output of consensus to support state sync and
> client balance proofs.

So you might have:
- An L1 that finalizes transactions.
- An aggregation layer that signs the state roots after each batch.
- Light clients that read those signatures as proof of state.

## The architecture

`aggregation/mod.rs:32-46`:

> The core of the module is the Engine. It manages the agreement process by:
> - Requesting externally synchronized digests
> - Signing said digests with the configured scheme's signature type
> - Multicasting signatures/shares to other validators
> - Assembling certificates from a quorum of signatures
> - Monitoring recovery progress and notifying the application layer of recoveries

Five responsibilities. The flow:

```
External sequencer emits item at height H (digest D).
  │
  ▼
Aggregation Engine receives D from external source.
  │
  ▼
Engine signs D with local key (or threshold share).
  │
  ▼
Engine multicasts signature to peers.
  │
  ▼
Receivers verify signature, add to in-progress set for height H.
  │
  ▼
Once 2f+1 signatures collected: form Certificate(D, H).
  │
  ▼
Notify application via Reporter.
```

## The signature schemes — same four as Simplex

`aggregation/mod.rs:18-30` — supports the same four schemes from chapter 04:

| Scheme | Aggregation | Attribution |
|---|---|---|
| ed25519 | No | Yes |
| secp256r1 | No | Yes |
| bls12381_multisig | Yes | Yes |
| bls12381_threshold | Yes | **No** |

Same trade-offs as chapter 04. BLS12-381 wins on aggregation; threshold
loses attribution.

## Missing Certificate Resolution — the design choice

`aggregation/mod.rs:51-58`:

> The engine does not try to "fill gaps" when certificates are missing.
> When validators fall behind or miss signatures for certain indices, the
> tip may skip ahead and those certificates may never be emitted by the
> local engine.

This is **deliberately simpler than Simplex**. Aggregation doesn't try to
backfill. If you fall behind, your local tip advances, you skip the
certificates you missed, and you start signing the new tip.

The safety property:

> Before skipping ahead, we ensure that at-least-one honest validator has
> the certificate for any skipped height.

So even if you skip a height, you know the consensus exists somewhere.
You just don't have it locally. (Applications that need the full history
implement their own recovery — typically using the resolver from chapter
08.)

## TipAcks — the gossip pattern

`aggregation/mod.rs:60-66`:

> In aggregation, participants never gossip recovered certificates. Rather,
> they gossip TipAcks with signatures over some height and their latest tip.

Why? Because gossiping full certificates is expensive. Instead, validators
gossip "I've signed height H; my current tip is H+5." Recipients can infer
that height H is safe (one honest validator signed it) without seeing the
full set of signatures.

This is the chapter 09 broadcast caching applied to certificates.

## The `safe_tip` mechanism

`aggregation/safe_tip.rs` — the algorithm for advancing the local tip.

The tip advances when **at least `f+1` validators report a higher tip**. (You
need `f+1` to be sure at least one of them is honest.) This is the same
principle as Quorum intersection from chapter 01: `2f+1` quorums always
intersect in at least `f+1` honest members.

## Epoch-Independent Signatures

`aggregation/mod.rs:69-82`:

> The attestation in an Ack covers only the Item (height and digest), not
> the epoch, and the ack namespace does not rotate per epoch. This is
> intentional: participants attest to an externally agreed-upon log that
> does not change across epochs, so an attestation to an item is valid
> regardless of the epoch in which it was produced.

So a signature from epoch 5 on item (height=10, digest=X) is valid in
epoch 10 too. No signature invalidation on epoch transition. The
aggregation's log is **continuous across epochs**, even as the validator
set rotates.

## Where it lives in the stack

```
External sequencer (could be Simplex itself, or a bridge, or anything)
        │
        ▼
   Aggregation Engine
   ├─ Receives digests
   ├─ Signs locally
   ├─ Gossip via P2P (chapter 05)
   ├─ Use Broadcast (chapter 09) or direct multicast
   ├─ Form Certificates via the Collector pattern (chapter 10)
   └─ Store in archive (chapter 06)
        │
        ▼
   Application (light clients, bridges, downstream chains)
```

## Where to look in the code

- `consensus/src/aggregation/mod.rs:1-82` — the architecture.
- `consensus/src/aggregation/engine.rs` — the main loop.
- `consensus/src/aggregation/safe_tip.rs` — the safe tip advancement.
- `consensus/src/aggregation/scheme.rs` — the signature schemes.
- `consensus/src/aggregation/types.rs` — the message types.

## BLS aggregation, in detail

Aggregation works because BLS12-381 admits a **bilinear pairing**: a map
`e: G1 × G2 -> GT` (a third group, sometimes called the "target group")
with the property that

```
e([a]P, [b]Q) = e(P, Q)^(a*b) = e([b]P, [a]Q)
```

where `[a]P` means scalar multiplication of point P by scalar a.

Concretely:

- A signature on a message m is `[sk] H(m)` in G1, where H hashes m into
  G1 and `sk` is the secret key.
- Verifying a single signature checks that `e(pk, H(m)) == e(sig, g2)`,
  where `pk = [sk] g2` and g2 is the G2 generator.
- **A signature is a point on the curve.** Points add the way elliptic
  curve points always add. This is what "aggregation" exploits.
- Given n signatures `sigma_1, sigma_2, ..., sigma_n` on the *same*
  message m, the aggregated signature is `sigma_1 + sigma_2 + ... + sigma_n`.

Verification of the aggregate collapses to a single pairing check:

```
e(aggregated_sigma, g2) == e(pk_1 + pk_2 + ... + pk_n, H(m))
```

which the verifier computes with one pairing per side of the equation
instead of n. **This is O(1) verification regardless of how many
signatures you combined.** Aggregate size is still one curve point (48
bytes compressed), independent of n.

The full deepening of G1, G2, pairings, and hashing-to-curve lives in
chapter 04's appendix; the central point is that bilinear structure lets
us add signatures the way we add integers, and a single pairing recovers
"is this aggregate valid on this message" without re-doing n pairings.

The `Scheme::Certificate` type that `assemble_certificate` returns is
exactly this aggregate. For `bls12381_multisig` it is the same aggregate
plus a bitfield of which validator indices contributed (so attribution
survives); for `bls12381_threshold` it is the same aggregate with no
attribution (because Lagrange interpolation hides the contributors).

## Certificate transparency: the same shape, in production

Google's **Certificate Transparency** (CT) project is the largest
deployed example of "collect signatures from a quorum to certify an
append-only log." The shape is identical to Aggregation; the
application domain is web PKI rather than consensus.

The CT problem (RFC 9162, "Certificate Transparency Version 2.0"):

- Every TLS certificate that a browser trusts must be recorded in one or
  more **append-only logs** run by independent operators.
- The log publishes a **Signed Tree Head (STH)** periodically — a Merkle
  root over the entries so far, plus a signature by the log operator's
  key.
- **Monitors** watch the logs to detect mis-issuance: a CA that issued
  a certificate the CA shouldn't have.
- **Auditors** verify that logs are append-only by checking consistency
  proofs between STHs.

Aggregation maps onto this almost term-for-term:

| Certificate Transparency | Commonware Aggregation |
|---|---|
| Append-only `Merkle Tree Hash (MTH)` log | Externally-ordered log of digests |
| `Signed Tree Head (STH)` | Aggregation `Certificate` |
| Log-shard quorum monitors | Validators' quorum signatures |
| Browser (auditor) verifies SCT against root | Light client verifies Certificate against expected root |

The reason CT uses Merkle trees, signed tree heads, and consistency
proofs (not full validator replay) is exactly the same reason Aggregation
gossips `TipAck`s instead of full certificates: bandwidth. The
aggregation engine's "TipAck: I'm at height H with signature over H"
matches a CT monitor's "I'm at STH R with signature over R" — both
protocols compress a quorum-of-signatures into one short, signature-bearing
status message that's checkable against the same expected root.

If you want a textbook treatment of CT in Commonware terms, see "Merkle
Mountain Range" (`docs/blogs/mmr.html`) for the data structure, RFC 9162
for the protocol, and chapter 21's "Where to look" table for the
relevant Commonware source.

## Cross-chain bridges: the security model and why BLS helps

A **cross-chain bridge** is a way to make an action on chain A appear to
take effect on chain B (or vice versa). The security model has three
threat surfaces:

1. **Sequencer / message source.** Who is allowed to say "this event
   happened on chain A"? If they're trusted, the bridge is "trusted."
   Most early bridges (Wormhole, the original Multichain, the Ronin
   hack of 2022) put a small multisig here and got robbed.
2. **Validator / witness.** A committee (often a PoS chain's validator
   set, or a dedicated bridge set) signs attestations: "yes, event E
   happened on chain A." If aggregation is used, the attestations are
   BLS-aggregated into one short Certificate that the destination
   chain verifies cheaply.
3. **Destination relay.** Who actually posts the attestation to chain B?
   It can be the relayer (a single party that pays gas), or a quorate
   relayer set, or anyone with an incentive to do it.

In the BLS case the bridge's `aggregation` engine gathers
`2f+1` signatures from the witness set per event, produces a
Certificate, and stores it. The destination chain's light-client
contract verifies the Certificate (a single pairing) against the
witness-set public key, which is itself a single curve point. **One
verification per event, regardless of the witness-set size.** That's
why Ethereum's beacon chain, IBC's light clients, and the various
bridges shipping this design all use BLS12-381: cheap verification, no
per-block-quorum verifier-code duplication.

Commonware aggregation's "epoch-independent signatures" property (next
section) is what makes this design tenant of long-lived bridge
infrastructure: the witness set can rotate without invalidating the
historical Certificate stream. Old Certificates still verify against
the same fixed destination-chain-side pubkey snapshot, even though the
set that produced them has long since moved on.

Wormhole (Solana), LayerZero's oracle+relayer pair, IBC's light clients
(Cosmos), ChainX's Polkadot bridge, and the `examples/bridge` in
Commonware itself are all real instances of this pattern.

## Why epoch-independent signatures

The argument, in full:

An Aggregation `Ack` signs `(namespace, height, digest)` — and **only**
that tuple. It does **not** sign the epoch. It does **not** sign any
"current validator set" commitment. It does **not** sign the policy
that defines quorum, the network's BLS public parameters, or anything
that varies across epochs.

Because the signed payload is fixed across epochs, an Ack from epoch 5
on `(height = 10, digest = D)` is bit-identical to an Ack from epoch 10
on `(height = 10, digest = D)`. A Certificate assembled in epoch 5 is
the same curve point — verifiable in the same way by the same code
path — in epoch 10.

The verifier side does need a "current validator set" to know **which
pubkeys to sum** before the pairing. That's where the per-epoch pubkey
snapshot enters: a destination-chain-side light client reads the
validator set for epoch 5 (which it snapshotted at the time), sums
those pubkeys, and verifies the Certificate.

So the lifetime of a Certificate is decoupled from the lifetime of any
one validator set. Old Certificates stay verifiable as long as the
verifier keeps the corresponding validator-set snapshot. New
Certificates from new sets are still compatible because they sign the
same fixed tuple.

The alternative — epoch-bound signatures — would force every consumer
on the destination chain to also track per-epoch signature namespaces
and rotation logic. In an Ethereum-like setting with millions of
historical logs, that overhead compounds fast.

The `Epoch` field that does appear on the `Ack` struct is
**unauthenticated metadata** (sender-supplied; receiver verifies the
signed payload, not the epoch field). It exists so the receiver can
select the right scheme + namespace + pubkey snapshot for the
acknowledgement without doing round-trips; it does not affect whether
the acknowledgement is valid.

This is also why aggregation keys don't need rotation. The
"aggregation public key" the destination chain verifies against is a
**per-epoch snapshot**, derived from the per-epoch validator pubkeys.
Each epoch has its own snapshot; rotation is "use the new snapshot for
new Certificates." Old Certificates still verify against their snapshot.
No key-update ceremony, no in-protocol handover, no signature
invalidation window.

## The `safe_tip`, formally

The safe tip is the height `H*` such that **at least one honest validator
has certificates for every height `h <= H*`** somewhere in the network.
This is not "the certificate for H* exists locally"; it's "the
certificate exists in the world."

The math, in two lines:

```
Quorum = 2f+1 (from chapter 01)
Honest = n - f (from chapter 01)
Quorum ∩ Honest = (2f+1) + (n - f) - n = f+1
```

So any quorum of size `2f+1` shares at least `f+1` members with the
honest set. Therefore **any quorum contains at least one honest
validator.**

The protocol collects tips: each validator gossips "I'm at height H"
periodically. The receiver collects these into a multiset, sorts them,
and takes the `(f+1)`-th highest as `safe_tip`. Why the `f+1`-th and
not the `2f+1`-th?

- The `2f+1`-th highest is "the lowest tip among a quorum." A quorum
  of size `2f+1` exists, but you cannot reach that conclusion from a
  multiset of reports — you only see reports, not proofs of them.
- The `(f+1)`-th highest is the highest tip such that "either I'm
  reporting this, or at least `f+1` other validators are reporting
  higher." Either way, at least one of those reporters is honest, and
  either way, that honest reporter has (or will have) the Certificate
  for `safe_tip + 1` at the latest.

Concretely: pick the `(f+1)`-th highest report `H*`. Sort all reports
descending. If you pick this as `safe_tip`, then at least `f+1` reports
are at or above `H*`. Of those `f+1` reporters, at least one is honest
(because `f+1` exceeds the maximum Byzantine set size `f`). That honest
reporter either (a) has the certificate for `H*`, or (b) is at a height
above `H*` because they have *all* certificates through that height
already. Either way the Certificate for `H*` exists.

This is the same argument as the quorum intersection from chapter 01;
rephrased as "we are looking for a quorum-of-reports-from-distinct-sets
to implicitly find an honest sub-quorum."

`aggregation/safe_tip.rs` lives in this file. Look at how the algorithm
is written: `tips.sort(); tips.get(n - f - 1)` (roughly), where `n -
f - 1` is the index of the `(f+1)`-th highest when 0-indexed. The off-
by-one in the source matches the off-by-one in the prose above; both
are correct.

The skip-when-missing property is then immediate: **once `safe_tip`
advances past `local_tip`, you can confidently start signing the new
tip because the certificates for the gap are guaranteed to exist
somewhere** — even though you yourself don't have them. If you later
need to fill the gap (e.g., to serve a state-sync request), you ask the
resolver (chapter 08).

## The `Ack` and `TipAck` types, in full lifecycle

Both types appear in `consensus/src/aggregation/types.rs`. Here is
their full purpose and lifecycle:

### `Ack<S: Scheme>`

```rust
pub struct Ack<S: Scheme> {
    pub height: Height,
    pub digest: Digest,
    pub signer: PK,
    pub signature: S::Signature,    // signs (namespace, height, digest)
    pub epoch: Epoch,               // unauthenticated metadata
}
```

Lifecycle:

1. **Genesis**: formed by the local engine when it observes a new item
   at height H. The engine checks `H > local_tip`, updates `local_tip`
   to `H`, signs `(namespace, H, D)` with its own scheme key, and
   prepends the signature to its own `acks[H]` map keyed by `signer =
   local_pubkey`.
2. **Send**: the engine multicasts the `Ack` to peers (or pushes it
   onto the broadcast cache from chapter 09).
3. **Receive**: a peer receives the `Ack`, looks up the scheme+pubkey
   for the claimed epoch, verifies the signature on
   `(namespace, H, D)`, then inserts the `Ack` into its `acks[H]` map
   under `signer`.
4. **Quorum formation**: when `acks[H].len() >= 2f+1`, the engine calls
   `S::assemble_certificate(acks[H].values().map(|a| &a.signature))`
   producing an `S::Certificate` and emits a `Certificate { height:
   H, digest: D, certificate }` notification to the `Reporter`.

### `TipAck<S: Scheme>`

```rust
pub struct TipAck<S: Scheme> {
    pub height: Height,
    pub digest: Digest,             // (height, digest) = the validator's *tip*
    pub signer: PK,
    pub signature: S::Signature,    // signs (tip_namespace, height, digest)
}
```

`TipAck` is structurally similar but semantically different: it certifies
the validator's **current tip**, not an intermediate item. It's used in
the safe-tip gossip instead of full certificates. Lifecycle:

1. **Tick**: periodically (governed by `tip_gossip_frequency`), the
   local engine produces a `TipAck` over `(tip_namespace, local_tip,
   local_tip_digest)` with its own key.
2. **Send**: multicasted.
3. **Receive**: a peer verifies the signature against the same
   scheme+pubkey, then updates its `tips` map with `(signer, (height,
   digest, TipAck))`.
4. **Safe-tip computation**: the engine re-runs the `safe_tip` algorithm
   every time the `tips` map changes.

### `Certificate<S: Scheme>`

```rust
pub struct Certificate<S: Scheme> {
    pub height: Height,
    pub digest: Digest,
    pub certificate: S::Certificate,  // aggregated form (one curve point for BLS)
}
```

The terminal product. Formed once per height. Emitted to the
`Reporter` and stored (typically in the `archive` from chapter 06 or
the `qmdb::keyless` log from chapter 16). Consumers verify the
`S::Certificate` against the expected pubkey and proceed.

### Why two ack shapes?

Because bandwidth cost grows with the **number of items per period**,
not with the **number of validators**. If you re-gossip full
Certificates on every gossip tick, you re-send `2f+1` signatures worth
of data. With TipAcks, you send one signature per validator per tick.
At 100 validators and 1-tick-per-second, that's the difference between
~10KB/s and ~5KB/s — modest, but at 10,000 validators it is the
difference between the protocol working and not. This is the same
status-vector-vs-full-state trick from gossip subprotocols (GossipSub,
SWIM); Chapter 09's broadcast engine implements it for cached data,
and Aggregation uses it for quorum-of-signatures.

## Test patterns for Aggregation

Integration tests live in `consensus/src/aggregation/tests/` and the
`mocks` directory. The canonical patterns:

### Happy path: progress under ideal conditions

```rust
#[test]
fn test_aggregation_progress() {
    let runner = deterministic::Runner::default();
    runner.start(|context| async move {
        // 4 honest validators, deterministic network (no loss)
        let mut oracle = ...;
        let (engines, supervisors) = build_aggregation_stack(
            context, &mut oracle, PARTICIPANTS, scheme,
        );

        // External sequencer emits 100 items
        let mut sequencer = external_sequencer(context);
        for h in 0..100 {
            sequencer.emit(h, random_digest(&mut rng));
            context.sleep(Duration::from_millis(10)).await;
        }

        // Wait for each engine to report 100 certificates
        for supervisor in &supervisors {
            let mut latest = 0;
            while latest < 100 {
                let report = supervisor.subscribe().await;
                let next = report.certificates.last();
                latest = next.height;
            }
        }
    });
}
```

The pattern: deterministic runtime + simulated P2P oracle (chapter 05 +
chapter 21) + supervisor monitor + assertion that finalization
progressed. This is exactly what
`consensus/src/aggregation/tests/simulated.rs` does.

### Gap path: skip when behind

```rust
#[test]
fn test_safe_tip_skip() {
    let runner = deterministic::Runner::timed(Duration::from_secs(60));
    runner.start(|context| async move {
        let (engines, supervisors, mut oracle) = build_aggregation_stack(
            context.clone(), PARTICIPANTS, scheme,
        );

        // Three validators run normally; one (validator A) is held offline.
        let a_idx = 0;
        for h in 0..50 {
            sequencer.emit(h, random_digest(&mut rng));
            context.sleep(Duration::from_millis(10)).await;
        }

        // Cut the link from A to the others.
        for pk in PARTICIPANTS.iter().skip(1) {
            oracle.remove_link(pk_by(a_idx), *pk).await;
        }

        // Force progress on B/C/D, then re-link A.
        for _ in 0..50 { sequencer.emit(...).await; sleep().await; }
        for pk in PARTICIPANTS.iter().skip(1) {
            oracle.add_link(pk_by(a_idx), *pk, default_link()).await;
        }
        for h in 50..100 {
            sequencer.emit(h, random_digest(&mut rng)).await;
        }

        // A's safe_tip should jump to ~100 quickly.
        let a_supervisor = &supervisors[a_idx];
        let safe_tip_after_recovery = wait_for_safe_tip(a_supervisor, 80).await;
        assert!(safe_tip_after_recovery >= 80);

        // A may not have local certificates for 50-79. That's the design.
        let local_count = a_supervisor.local_certificates().len();
        assert!(local_count < 100, "design says we skip, not backfill");
    });
}
```

This validates "safe tip advances past missing heights" and "local
certificates may be missing for the skipped range." Look for the
equivalent pattern in `tests/skip_recovery.rs`.

### Byzantine path: invalid signatures get blocked

```rust
#[test]
fn test_byzantine_acks() {
    let runner = deterministic::Runner::default();
    runner.start(|context| async move {
        let mut oracle = ...;
        // Replace validator 0 with a Conflicter that sends invalid acks.
        let conflicter = mocks::conflicter::Conflicter::new(context.child("conflicter"), ...);
        conflicter.start(pending);

        let (supervisors, _) = build_aggregation_stack_for_others(context, PARTICIPANTS, scheme);

        // Run for a while.
        for h in 0..100 { sequencer.emit(...).await; sleep().await; }

        // The other 3 validators should reach quorum from the honest 3.
        for i in 1..PARTICIPANTS.len() {
            let cert_count = supervisors[i].certificate_count();
            assert!(cert_count >= 99, "honest validators must progress despite Byzantine");
        }
    });
}
```

The Conflicter is a standard mock from `consensus/src/simplex/mocks/`;
it sends signed messages that don't verify. The aggregation engine's
verify-on-receive path (`scheme.verify(...)`) should reject these
without dropping the Byzantine peer (since the engine's block-list
lives elsewhere — see chapter 05).

## Rust pattern — trait objects, marker types, `PhantomData`

Aggregation's design relies on a few Rust patterns that are worth
flagging because they appear throughout Commonware.

### Trait-object polymorphism for `Scheme`

The four signature schemes share a `Scheme` trait:

```rust
pub trait Scheme: Send + Sync + 'static {
    type Signature: ...;
    type PublicKey: PublicKey;
    type Certificate: ...;
    fn sign(&self, namespace: &[u8], message: &[u8]) -> Self::Signature;
    fn verify(pk: &Self::PublicKey, namespace: &[u8], message: &[u8], sig: &Self::Signature) -> bool;
    fn assemble_certificate<'a, I: Iterator<Item = &'a Self::Signature>>(
        signatures: I,
    ) -> Self::Certificate;
}
```

Downstream code can take a `Box<dyn Scheme>` or, more commonly, a
generic `S: Scheme`. The Aggregation `Engine` uses generic
dispatch (`Engine<S>`) because the scheme's `Signature` and
`Certificate` types appear in messages and storage — boxing via
`dyn Trait` would impose an extra indirection on every message and an
allocation per message. **The aggregation hot path needs monomorphized
static dispatch; storage and tests can afford dyn if they want.**

The general rule in Commonware: generics on message-carrying types,
`Box<dyn Trait>` only at the edges (configuration, factory functions,
plugin registries).

### `PhantomData` for marker types

The `Mailbox` in chapter 15 uses `PhantomData` to carry a `Policy`
type without owning an instance:

```rust
struct Mailbox<T, P> {
    state: Arc<MailboxState<T, P>>,
    _marker: PhantomData<P>,    // P is a tag, not stored
}
```

`P` encodes which overflow-handler trait the mailbox uses (Reliable vs
UnreliablePolicy). Without `PhantomData<P>`, you'd have no field of
type `P` for the compiler to track the type parameter through the
struct. `PhantomData<T>` is the zero-cost way to say "this type
parameter exists for the type system; do not allocate storage for it."

You'll see this pattern in aggregation too: the `Ack<S>` type carries
`S::Signature` which lives in a real field, but `S` itself is a type
parameter with no instance at the struct level — Rust would treat `S`
as unused and warn, except the `S::Signature` field uses it.

### Why Rust trait objects lose on the hot path

When you write `Box<dyn Scheme>`, every signature verifies through a
vtable lookup: `(*scheme).verify(...)` reads the function pointer from
the heap, indirects, calls. Each message's verification is then:

```
fn verify_ack(scheme: &dyn Scheme, ack: &Ack<dyn Scheme>) {
    let sig_ok = scheme.verify(...);  // one vtable jump
    ...
}
```

vs. with generics:

```
fn verify_ack<S: Scheme>(scheme: &S, ack: &Ack<S>) {
    let sig_ok = S::verify(...);  // inlined, no jump
}
```

For 10,000 acks/second, the difference is roughly 10,000 nanoseconds
of vtable traffic — small in absolute terms, but real, and on a
hot-loop code path in BFT consensus. Commonware consistently picks
generics for protocol logic, `Box<dyn>` only at API boundaries.

## Where to look in the code (expanded preview)

The appendices that follow go deeper on each topic above. When you see
a `file:line` reference in main body, the appendix walks it line by
line — but the surrounding structure and motivation live here.

---


## Deepening 1 — BLS signature aggregation in full

Aggregation works because BLS12-381 admits a **bilinear pairing**.
This section reconstructs the math from groups to applications.

### The bilinear pairing, formalized

Three groups:

- `G1`: an additive group of prime order `r`, with generator `g1`.
- `G2`: an additive group of prime order `r`, with generator `g2`.
- `GT`: a multiplicative group of prime order `r`.

A **bilinear map** (pairing) `e: G1 x G2 -> GT` satisfies:

```
e([a]P, [b]Q) = e(P, Q)^(a*b) = e([b]P, [a]Q)
```

where `[a]P` is `P + P + ... + P` (a times) — scalar multiplication
of `P` by `a`.

The map is computed via **Miller's loop** followed by a
**final exponentiation**. For BLS12-381, this is roughly 1 ms on a
modern CPU.

### The cryptographic hardness assumption

**co-CDH** (co-Diffie-Hellman with pairing): given
`g1, [a]g1, g2, [b]g2`, computing `[ab]g1` is hard (i.e., discrete
log on the group is hard).

This is what BLS signatures reduce to: a forger without `sk` cannot
produce a valid signature pair `(m, sigma)` unless they can solve
co-CDH. The reduction is tight (Pollard's rho attack) for the
scheme's security level.

### Sign and verify, formally

Setup (per signer):

- `sk`: a random scalar in `Z_r`.
- `pk = [sk] g2` in `G2`.

Sign (for any message `m`):

- `H = H_BLS(m)`: a hash of `m` to `G1` (RFC 9380 hashing-to-curve).
- `sigma = [sk] H`: scalar multiplication in `G1`.

Verify:

```
e(sigma, g2) ?= e(H, pk)
```

Expanded:

- LHS = `e([sk] H, g2) = e(H, g2)^(sk)`.
- RHS = `e(H, [sk] g2) = e(H, g2)^(sk)`.

Match. ✓

### The aggregation: O(1) verification

For N signatures on the same message `m`:

```
sigma_agg = sigma_1 + sigma_2 + ... + sigma_N     (G1 addition)
```

Verify:

```
e(sigma_agg, g2) ?= e(H, pk_1 + pk_2 + ... + pk_N)
```

LHS = `e(sigma_agg, g2) = e(H, g2)^(sum sk_i) = e(H, sum sk_i * g2)`.
RHS = `e(H, sum sk_i * g2)`. They match by bilinearity.

**O(1) verification**: one pairing on each side (not N). Aggregate
size = one curve point (96 bytes in compressed BLS12-381 G1).

### The security argument for aggregation

Without aggregation, an attacker that controls `f + 1` signers
can produce `f + 1` signatures, but they're isolated to those
signers. With aggregation, the attacker's contribution is
*folded* into the aggregate with the others.

But aggregation only makes sense for **the same message**. If two
different messages are involved, you'd need multisig (separate
aggregates per message).

Threshold BLS (l = (t, n) Pedersen DKG):

- `n` participants each get a share `sk_i`.
- Any `t` participants can jointly produce a signature.
- The aggregated signature is indistinguishable from one produced
  by any specific subset of `t` (Lagrange folds the choice).

### The multisig vs threshold trade-off

| Property | Multisig | Threshold |
|---|---|---|
| Attribution | Yes (bitfield) | No |
| Cert size | 96 B + 100 bits | 96 B |
| Per-signer secret | Full `sk_i` | Full `sk_i` (lives with signer) |
| Verification | O(n) (verify each sig) or O(1) (one pairing + bitfield) | O(1) |
| Use case | Slashing, accountability | Cheap verification, BLS-friendly bridges |

Commonware's BLS12-381 threshold is a true `(2f+1, 3f+1)` Pedersen
DKG. Multisig is BLS12-381 with no threshold, just summing.

### The pairing-pair check optimization

For batch verification, you can batch many pairing-pair checks
together. Commonware's `BatchVerifier` does this:

```rust
let mut rng = OsRng;
let mut bv = scheme::BatchVerifier::new();
for (msg, sig) in &messages {
    bv.add(namespace, msg, pk, sig);
}
let valid = bv.verify(&mut rng, &self.strategy);
```

The `strategy` (Random, Deterministic, Concurrent) controls how the
batch is parallelized. Random is fastest; Deterministic is
reproducible.

### The hashing-to-curve subtlety

BLS signatures require hashing arbitrary messages into G1 points.
RFC 9380 (which BLS12-381 follows) defines two methods:

- `hash_to_curve m`: optimized for hashing short messages.
- `encode_and_hash`: hash with a tag.

Commonware uses `hash_to_curve` with a domain-separation tag
(`namespace` parameter to all signing operations). The tag prevents
cross-protocol attacks (where a signature in protocol A could be
re-interpreted as a signature in protocol B).

### Where BLS aggregation breaks

1. **Different messages**: aggregation only works for sigs on the
   *same* message. Cross-message aggregation is non-trivial
   (Boneh-Gentry-Lynn-Shacham 2003 covers this case via different
   schemes).
2. **Different public keys per signer**: not a problem for BLS
   threshold / multisig (the verifier knows `pk_1, ..., pk_n` and
   sums them).
3. **Rogue key attacks**: an attacker submits a public key that
   *looks* like another pubkey minus the attacker's. The naive
   verification `e(sigma, sum pk_i) == e(H, sum sk_i g2)` then
   trivially passes for the attacker. Defenses:
   - **Proof of possession**: each pubkey comes with a zero-knowledge
     proof that the signer knows `sk_i`.
   - **Hashed keys**: the actual pubkey is `H(K_i)` where `K_i` is
     a curve point; this makes rogue keys infeasible.
   - **Knowledge-of-secret-key (KOSK)**: trusted DKG ensures each
     signer knows `sk_i`.
   Commonware uses trusted setup (DKG) which is the strongest
   defense.

---

## Deepening 2 — Certificate Transparency in full

Google's Certificate Transparency (CT) is the largest deployed
example of "collect signatures from a quorum to certify an
append-only log." This section shows the term-for-term mapping onto
Aggregation.

### The CT problem (RFC 9162, "Certificate Transparency Version 2.0")

Every TLS certificate that a browser trusts must be recorded in one or
more **append-only logs** run by independent operators. The log:

1. Receives certificate submissions from CAs.
2. Constructs a Merkle Tree over recent entries.
3. Periodically publishes a **Signed Tree Head (STH)** — a Merkle
   root over the entries so far, plus a signature by the log
   operator's key.

There are three classes of participants:

- **Log operators**: maintain the log, publish STHs.
- **Monitors**: watch the log to detect mis-issuance (a CA that
   issued a cert the CA shouldn't have).
- **Auditors**: verify consistency proofs between STHs to ensure
   the log is append-only.

### The Merkle Tree and the Signed Tree Head

A **Merkle Tree** (RFC 9162 §2) is a binary tree where each internal
node is the hash of its children. The leaves are the entries
(certificates); the root is a single 32-byte hash. Inclusion
proofs: the path from a leaf to the root, with sibling hashes.

An **STH** is `(tree_size, root_hash, signature_by_log_operator)`.
A monitor verifying an STH:

1. Verifies the signature against the log's public key.
2. Maintains a local view of "what's in the log" (received
   certificates).
3. Checks "has the log issued a certificate I haven't seen?" via
   inclusion proofs.

If the log misbehaves (issues conflicting STHs, deletes entries,
etc.), monitors and auditors detect it via the cryptographic
structure.

### Mapping to Commonware Aggregation

| CT concept | Commonware Aggregation |
|---|---|
| Append-only log of certificate hashes | Externally-ordered log of digests (could be CT's itself) |
| Signed Tree Head (STH) | Aggregation `Certificate` |
| Log operator's signature (25519) | Validator's BLS signature on (h, D) |
| Monitors (track STHs) | Aggregation's Reporter + safe-tip-gossip |
| Auditors (verify append-only) | Light clients verifying aggregator's BLS pubkey snapshot |

The translation is exact:

- Each STH = "this log has seen N entries with root R, signed by
  log_op_pk" is equivalent to a Certificate of (height = N,
  digest = R) signed by the validator set's BLS pubkey.
- A monitor's job is "look at all STHs I have, deduce what entries
  the log has, find missing certificates" which is exactly what
  Aggregation's safe-tip algorithm does.
- An auditor's job is "verify that STH_k+1 extends STH_k, i.e., the
  log is append-only" which is what BLS aggregation's certification
  enforces: the quorum-signed certificate is the only way entries
  can be "in" the log.

### Why CT works

The economic + cryptographic trick:

- **Cryptographic**: any log operator who issues a "fork" STH
  (publishes two STHs at the same height with different roots) is
  detected because both STHs would have valid signatures by the
  same key, but only one can be the "true" continuation. A monitor
  who has both can prove the fork.
- **Economic**: browsers refuse to accept certificates not in a
  CT log. So CAs *must* submit to CT, and the logs are observable
  by all participants.

### Aggregation's replication of CT's reliability

CT requires each log's STH to be widely distributed — otherwise a
log could "fork" locally and the monitors wouldn't see both. CT
uses gossip; Aggregation uses BLS quorum-certification. The
end-result is similar: the certified state is durably recorded
multiple times, and any divergence is detectable.

A malicious CT log could publish:

```
STH_1 = (size=100, root=R1, sig_log_op)
STH_2 = (size=100, root=R2, sig_log_op)
```

Two STHs at the same size but different roots = a fork. Monitors
publish both. Browsers reject both until the fork is resolved (the
log operator creates an inconsistency proof to determine which is
"real").

Aggregation's analog: a validator can't fork the aggregated log
because the BLS public key sum is fixed per epoch. Any certificate
that doesn't match is rejected by light clients.

### CT vs. Aggregation's "epoch-independent signatures"

CT signatures (per-STH) are tied to the log operator's key. If the
key rotates, old STHs become unverifiable (modulo "log key
rotation" procedures in CT v2).

Aggregation's "epoch-independent signatures" (chapter 13 main text
+ deepens below) decouple the BLS key from any one epoch. Old
certificates remain verifiable with the old epoch's pubkey
snapshot, indefinitely. New certificates use the new pubkey.

This is what makes Aggregation suitable for long-running bridges:
the witness set can rotate without invalidating historical
certificates. CT doesn't quite achieve this without ceremony.

---

## Deepening 3 — Cross-chain bridges in depth

A **cross-chain bridge** makes an action on chain A appear to take
effect on chain B. The security model has three threat surfaces,
and BLS aggregation addresses each.

### The three threat surfaces

1. **Sequencer / message source.** Who says "this event happened
   on chain A"? If it's a trusted party, the bridge is "trusted."
   Most early bridges (Wormhole, original Multichain, the Ronin hack
   of 2022) put a small multisig here and got robbed when one key
   was compromised.
2. **Validator / witness.** A committee signs attestations: "yes,
   event E happened on chain A." If aggregation is used, attestations
   are BLS-aggregated into one Certificate that the destination
   chain verifies cheaply.
3. **Destination relay.** Who posts the attestation to chain B? A
   single relayer, a quorum of relayers, or anyone-incentivized.

### Why BLS helps

On the destination chain (typically Ethereum, IBC, or Solana), the
bridge's verification contract must:

1. Verify the Certificate (one BLS pairing, ~50 µs in EVM via
   precompiles).
2. Update the destination state.

Without BLS, the destination would need to verify N individual
signatures (~50 µs × N = 5 ms for N=100). At 10 events/sec per
bridge, that's 50 ms of verification per second — fine for one
bridge, but a problem if many bridges share a chain.

With BLS: ~500 µs for 10 events — much better. This is why
Ethereum's beacon chain, IBC's light clients, and production
bridges use BLS12-381.

### The attack vectors

Common attack vectors against cross-chain bridges:

1. **Witness compromise** (`f + 1` or more witness keys stolen).
   Attack: forge a withdrawal from chain A. Defense: BLS threshold
   requires `2f + 1` participants, so `f + 1` stolen keys is
   useless.
2. **Relayer censorship**. Relayer refuses to post a valid
   certificate. Defense: open relayer set with incentives.
3. **Bridge contract bug**. Logic error in the destination chain's
   bridge. Defense: careful audits, formal verification, time-
   locked upgrades.
4. **Replay attack**. Submitting the same certificate twice.
   Defense: nonce or unique-cert-id on the destination chain.
5. **Reorg on chain A**. The bridge commits to a state that later
   reorgs. Defense: wait for "finality confirmation depth" on
   chain A.

### The classic Ethereum/IBC/Wormhole comparison

**Wormhole (Solana)** uses a multisig guardian set with 19 nodes
(needing 13-of-19 to sign). This is fast (signature aggregation is
optimized but not BLS) but suffers key management concerns.

**IBC (Cosmos)** uses Tendermint + light clients. Each chain runs
its own validator set; the destination chain verifies the source
chain's signatures via the light client. Honest-majority assumption
per chain.

**LayerZero** uses oracle+relayer pair (Chainlink + nominated
relayer). Each message needs both to attest; if they collude, the
bridge is broken.

**ChainX (Polkadot bridge)** uses Polkadot's GRANDPA finality +
BLS aggregation. The advantage over Wormhole: one signature
verification for any committee size.

**Commonware's `examples/bridge`** uses Simplex + Aggregation. The
advantage:

- BLS threshold on the witness side: cheap verification.
- Epoch-independent: witness set rotation doesn't invalidate
  historical certificates.
- Pluggable scheme: rotate BLS scheme if a vulnerability emerges.

### The bridge's "honest-majority" assumption

Bridges inherit their security from the witness's honest-majority
assumption. If 2f+1 witnesses are honest (acting correctly per the
protocol), the bridge is safe.

This is weaker than the chain's assumption: the *chain* requires
2f+1 honest validators on chain A AND 2f+1 honest validators on
chain B AND 2f+1 honest witnesses AND 1 honest relayer. Four
honest-majority assumptions; the weakest link matters.

Aggregation can't reduce this; it just makes the witness
verification cheap.

### Why epoch-independence matters for bridges

A bridge running for years needs to rotate its witness set
(membership changes, key custody refresh, etc.). With epoch-bound
signatures, every rotation invalidates historical certificates.

With epoch-independent signatures (the Aggregation design), the
*destination chain* keeps a per-epoch snapshot of the witness set's
BLS public key. Old certificates verify against the old snapshot;
new certificates use the new snapshot. No ceremony, no invalidation
window.

For a 5-year-running bridge with 12 epoch rotations per year, the
savings are vast: no per-cert update logic on the destination
chain, just "look up the snapshot for cert.epoch_id."

---


## Deepening 4 — Why epoch-independent signatures work

The earlier section on epoch-independent signatures gave the high
level. This section grounds the claim in math and shows why it's
robust.

### The argument, formally

An Aggregation `Ack` signs `(namespace, height, digest)` — and
*only* that tuple. It does NOT sign:

- The epoch.
- Any "current validator set" commitment.
- Network identity policy.
- Per-epoch BLS public parameters.

Because the signed payload is fixed across epochs, two Acks — one
from epoch 5, one from epoch 10 — are bit-identical if they cover
the same `(height, digest)`. A Certificate assembled in epoch 5 is
the same curve point as one assembled in epoch 10.

The verifier side does need a "current validator set" to know
*which pubkeys to sum* before the pairing. That's where the
per-epoch pubkey snapshot enters: a destination chain's light
client reads the validator set for epoch 5 (which it snapshotted
at the time), sums those pubkeys, and verifies the Certificate.

So the lifetime of a Certificate is decoupled from the lifetime of
any one validator set. Old Certificates stay verifiable as long as
the verifier keeps the corresponding validator-set snapshot. New
Certificates from new sets are still compatible because they sign
the same fixed tuple.

### The math

Let's denote:

- `Set(e)` = the validator set for epoch `e`.
- `pk_agg(e)` = `sum_{i in Set(e)} sk_i * g2` = the BLS public key
  for epoch `e`.
- `Cert(h, d)` = the certificate for height `h`, digest `d`. It's
  `sum_{i in Set(e)} sk_i * H(h, d)` for some epoch `e` (any epoch
  that emitted an Ack for `(h, d)`).

A verifier at time `T` checks Cert:

```
e(Cert, g2) == e(H(h, d), pk_agg(e))
```

For verification, the verifier needs `pk_agg(e)`. It looks it up in
its per-epoch snapshot. As long as the snapshot exists (e.g., the
verifier stored the epoch's validator set when it was active), the
verification succeeds.

For each epoch `e`:
- The pubkey is `pk_agg(e) = sum_{i in Set(e)} pk_i`.
- Snapshotting `pk_agg(e)` requires only storing one curve point
  per epoch (96 bytes).

### The key rotation considerations

When a validator set rotates from epoch `e` to `e'`:

- Old `Set(e)` members may not be in `Set(e')`.
- Their old signatures (AcKs in epoch `e`) remain valid forever
  (signed with `sk_i` for epoch `e`, not the new key).
- New `Set(e')` members' signatures require `pk_agg(e')` for
  verification.

The light client's snapshot maintenance:

```rust
struct EpochSnapshot {
    epoch: Epoch,
    pubkeys: Vec<PublicKey>,    // for verification
    aggregated: PublicKey,       // pre-summed for fast verification
}
```

Stored once per epoch, never updated. As long as the light client
keeps snapshots from enough historical epochs, it can verify
historical certificates.

### The destination-chain cost

For an Ethereum-like chain with millions of historical certificates
across many epochs:

- Storage: one snapshot per epoch = ~96 bytes × num_epochs.
  For 5 years × 12 epochs/year = 60 snapshots × 96 bytes = ~6 KB.
  Negligible.
- Verification per cert: 1 pairing + 1 epoch-snapshot lookup = ~50 µs
  + ~100 ns lookup.

Compare to multisig (BLS without epoch-independence):

- Storage: one key per validator per epoch = 64 bytes × 100 × 60
  = ~400 KB.
- Verification: 100 pairings per cert (or 1 pairing + bitfield) =
  ~5 ms.

Epoch-independence is dramatically cheaper at storage and slightly
faster per-verification.

### The alternative (epoch-bound signatures)

If signatures included the epoch:

```
Ack(h, d, e) -> S = sk * H(namespace, h, d, e)
```

Then the destination chain must:
1. Snapshot the validator set for epoch `e` (same).
2. Track the signed-payload format (it changes as the protocol
   evolves).
3. Re-verify all historical AcKs if any protocol-level change
   happens.

The third point is brutal. A bugfix requires re-signing every
historical ack, which is impossible. So the protocol can't
upgrade. Epoch-bound signatures lock the protocol.

### Why BLS makes this work

The reason this is BLS-specific: BLS supports aggregation
*cleanly*. With Ed25519 or secp256r1, you'd need multisig (which
requires per-signer verification), and the verifiers would need
to keep track of individual pubkeys per validator per epoch. The
storage scales linearly with set size and epoch count.

BLS lets you *compress* the per-epoch set into a single point.
The per-epoch snapshot is one curve point, not a list of
pubkeys. The compression is *exact* (verifies in one pairing).

### The aggregation guarantee

The deeper theorem: for any two subsets `S1, S2` of a validator set,
each of size >= 2f+1, the BLS aggregate of signatures from `S1`
equals the BLS aggregate from `S2`. This is because:

```
sum_{i in S1} sk_i * H = sum_{j in S2} sk_j * H
?=
```

No, that's not right. The aggregates from S1 and S2 are NOT generally
equal unless both sets contain at least 2f+1 members — and there's
no guarantee they overlap exactly.

What's true:

- For any single message `m`, the *threshold* aggregate from any
  2f+1-subset equals the threshold aggregate from any other
  2f+1-subset. (Lagrange interpolation forces this.)
- For multisig, the aggregate is unique *given the set*. Different
  sets give different aggregates.

The epoch-independence property uses the threshold flavour: any
2f+1-subset's aggregate works. We don't have to know *which*
2f+1-subset; any will do.

---

## Deepening 5 — The safe_tip formal definition

The earlier section on `safe_tip` gives an intuitive description.
This section formalizes the math and connects it to PBFT's commit
certificate.

### The quorum intersection math

Two quorums of size `2f+1` from a set of size `n = 3f+1` overlap in
at least `f+1` members:

```
| Q1 ∩ Q2 | = |Q1| + |Q2| - |Q1 ∪ Q2|
            >= 2f+1 + 2f+1 - (3f+1)
            = 4f+2 - 3f - 1
            = f+1
```

(Using `|Q1 ∪ Q2| <= n = 3f+1`.)

So any two 2f+1 quorums share at least `f+1` members. Of those `f+1`
shared members, at least one is honest (because the maximum
Byzantine set size is `f`).

Therefore: **any 2f+1-quorum contains at least one honest
member**. This is the safety argument that underlies every BFT
protocol.

### The safe_tip algorithm

The engine collects tips: each validator gossips "I'm at height H"
periodically. The engine receives these reports and sorts them.

```rust
fn compute_safe_tip(reports: &BTreeMap<PK, (Height, Digest)>) -> Height {
    // Sort reports by height, descending.
    let mut heights: Vec<Height> = reports.values().map(|(h, _)| *h).collect();
    heights.sort_by(|a, b| b.cmp(a));

    // Take the (f+1)-th highest (0-indexed: index n - f - 1).
    let idx = heights.len().saturating_sub(self.f + 1);  // wait, careful here
    // Actually: want the highest H such that >= f+1 reports are >= H.
    // That H is the (f+1)-th highest when sorted descending.
    // In a sorted-descending vec of size m, that's heights[f].
    heights.get(self.f).copied().unwrap_or(Height::zero())
}
```

The detailed math: among all reports, at least `f+1` are at height
≥ H*. Of those `f+1` reporters, at least one is honest (quorum
intersection argument). That honest reporter has Certificates for
all heights ≤ its current height.

### The `(f+1)`-th highest justification

Why `(f+1)`-th, not `(2f+1)`-th?

`(2f+1)`-th: "the lowest tip among a quorum." This requires
proof that a quorum is exactly at this height. Without such proof
(we only see reports, not proofs), we can't conclude.

`(f+1)`-th: "the highest H such that at least `f+1` reports are
≥ H." We don't need a quorum at exactly H; we just need at least
`f+1` reporters *above* H (or at H). One of them is honest.

So `(f+1)`-th is the conservative choice — "I'm sure that at
least one honest validator has height H or higher."

### The comparison with PBFT's commit certificate

PBFT's commit certificate says: "2f+1 replicas have voted to
commit (v, n, d)." That's an explicit `2f+1` quorum with explicit
signatures. The Certificate is a directly verifiable proof.

Aggregation's `safe_tip` is *implicit*: it's inferred from a
collection of tip reports. There's no Certificate for "everyone
agrees the tip is H." Instead, the algorithm deduces "at least one
honest validator has H or higher" — which is sufficient for
safety but doesn't require 2f+1 explicit attestations.

The cost: PBFT's commit is **explicit** (you can prove to a third
party "this is committed"). Aggregation's safe-tip is **internal**
(it tells the local engine when it's safe to advance, but the
engine doesn't publish this to others).

If you need to publish safe-tip externally (say, to a downstream
protocol), you need a Certificate — which Aggregation *also*
produces, separately from safe-tip.

### Why both safe-tip AND Certificate exist

Aggregation has two safety artifacts:

1. **safe_tip**: a *local* marker saying "I'm sure there's a
   quorum somewhere at height H."
2. **Certificate**: a *public* proof that 2f+1 validators agreed
   on a specific (h, d).

A peer who learns the safe_tip can advance locally. A peer who
learns the Certificate can prove the same to a third party.

The Certificate is more expensive (requires forming an actual
2f+1-quorum of signatures). The safe_tip is cheaper (just a
collection of tips).

In practice:
- The Certificate is formed once per height (when the engine
  observes 2f+1 Acks).
- The safe_tip is updated continuously (every tip message
  received).

The Certificate is for cross-chain bridges; the safe_tip is for
local progress.

### The skip-when-missing property, formally

Once safe_tip advances past local_tip, the algorithm declares "you
may start signing the new tip."

Let `safe_tip = H*` and `local_tip = H`. If `H* > H`, then
`H* - H` is the skip range. The algorithm's correctness:

For each `h in (H, H*)`, at least one honest validator has
signed `h`. That validator has the Certificate. So the Certificate
exists somewhere in the network.

The local engine can safely advance its local_tip to H* and start
signing H* + 1, confident that the Certificates for
`(H, H*]` exist somewhere, even though the local engine doesn't
have them.

If the local engine later needs to fill the gap (e.g., to serve
a state-sync request), it asks peers via the resolver.

### The (f+1)-th-highest edge cases

Edge cases:

- **Fewer than f+1 reporters**: H* = 0 (genesis). Wait until more
  reports arrive.
- **All reporters at same height**: H* = that height.
- **One Byzantine reporter at very high height**: doesn't affect
  H*; the algorithm uses the (f+1)-th highest, so a single high
  report is discarded.
- **Network partition**: reports may not arrive. Safe_tip stalls
  at the last known H*. Once partition heals, reports resume,
  safe_tip catches up.

The algorithm is **partition-tolerant**: it makes progress when
the network is healthy and stalls under partition. No unsafe
advances.

---

## Deepening 6 — Ack and TipAck types in depth

The two ack shapes are subtle. This section walks the full
lifecycle, the encoding, and the verification.

### Ack full lifecycle

```rust
pub struct Ack<S: Scheme> {
    pub height: Height,
    pub digest: Digest,
    pub signer: PK,
    pub signature: S::Signature,
    pub epoch: Epoch,        // unauthenticated metadata
}
```

Step 1: genesis at the local engine.

The engine observes an item at height H (via the external
sequencer). It checks `H > local_tip`. If yes, updates
`local_tip = H`, then signs:

```rust
let to_sign = encode_ack_payload(&self.namespace, height, digest);
let signature = self.scheme.sign(&self.ack_namespace, &to_sign);
let ack = Ack { height, digest, signer: my_pk, signature, epoch: current_epoch };
self.acks[height].insert(my_pk, ack.clone());
```

Step 2: send.

The engine multicasts the `Ack` to peers (or to the broadcast
cache from chapter 09 for fan-out).

Step 3: receive.

A peer receives, looks up the scheme + pubkey for the claimed
epoch, verifies:

```rust
let payload = encode_ack_payload(&namespace, ack.height, ack.digest);
let valid = S::verify(&epoch_pubkey, &ack_namespace, &payload, &ack.signature);
if !valid {
    return Err(InvalidAck);
}
```

After verification, the peer inserts:

```rust
self.acks[ack.height].insert(ack.signer.clone(), ack.clone());
```

Step 4: quorum formation.

When `self.acks[height].len() >= 2f+1`:

```rust
let sigs = self.acks[height].values().map(|a| &a.signature);
let cert = S::assemble_certificate(sigs).expect("BLS agg");
let certificate = Certificate { height, digest, certificate: cert };
self.reporter.report(Activity::Certificate(certificate)).await;
```

### TipAck full lifecycle

```rust
pub struct TipAck<S: Scheme> {
    pub height: Height,
    pub digest: Digest,
    pub signer: PK,
    pub signature: S::Signature,
}
```

Note: no `epoch` field. The TipAck's `height` is the validator's
*current tip*, not a specific item being acked.

Step 1: tick.

Periodically (governed by `tip_gossip_frequency`):

```rust
let payload = encode_tipack_payload(&self.namespace, self.local_tip, self.local_tip_digest);
let signature = self.scheme.sign(&self.tip_namespace, &payload);
let tipack = TipAck { height: local_tip, digest: local_tip_digest, signer: my_pk, signature };
self.tips.insert(my_pk.clone(), (local_tip, local_tip_digest, tipack.clone()));
self.sender.send(Recipients::All, Encode::encode(&tipack), false).await;
```

Step 2: send.

Same as Ack.

Step 3: receive.

A peer verifies:

```rust
let payload = encode_tipack_payload(&namespace, tipack.height, tipack.digest);
let valid = S::verify(&epoch_pubkey, &tip_namespace, &payload, &tipack.signature);
if !valid { return Err(InvalidTipAck); }

self.tips.insert(tipack.signer.clone(), (tipack.height, tipack.digest, tipack));
self.compute_safe_tip().await;   // re-run on every tip
```

Step 4: safe-tip computation.

The local engine recomputes safe_tip whenever the tips map
changes. Implementation:

```rust
async fn compute_safe_tip(&mut self) -> Height {
    let mut heights: Vec<Height> = self.tips.values().map(|(h, _, _)| *h).collect();
    heights.sort_by(|a, b| b.cmp(a));
    let h_star = heights.get(self.f).copied().unwrap_or(Height::zero());
    if h_star > self.safe_tip {
        self.safe_tip = h_star;
        // Maybe also emit a Certificate for h_star if local knows
    }
    h_star
}
```

### Why two different shapes?

If you only had `Ack`s, you'd need to gossip every Ack to every
peer. With 100 validators and 1000 items, that's 100,000 messages
(total). At 1 KB each, 100 MB.

If you gossip *only* TipAcks (one per validator per period), the
same information propagates much more cheaply:
- 100 validators.
- Tip-period = 1 sec.
- TipAck = 200 bytes (height + digest + sig).
- Traffic: 100 × 200 bytes = 20 KB/sec.

That's ~5,000× cheaper for similar progress-tracking.

The TipAck is a "status-vector" message — like GossipSub's IHAVE
or SWIM's failure-detection pings. Aggregating them all would
duplicate work; sending them only when the tip changes would lose
liveness info.

The "tip every X seconds" tradeoff:

- Higher X → less traffic, but slower safe_tip advancement.
- Lower X → faster advancement, more traffic.

Commonware's default is 100 ms — sufficient for most deployments.

### The encoding details

Both types use Commonware's Codec (chapter 03). The encoding:

```rust
impl<S: Scheme> Encode for Ack<S> {
    fn encode(&self) -> Bytes {
        let mut buf = BytesMut::new();
        self.height.encode_to(&mut buf);
        self.digest.encode_to(&mut buf);
        self.signer.encode_to(&mut buf);
        self.signature.encode_to(&mut buf);
        self.epoch.encode_to(&mut buf);
        buf.freeze()
    }
}
```

Fields:

- `height`: varint (typically 1-3 bytes).
- `digest`: 32 bytes (SHA-256).
- `signer`: 32 bytes (Ed25519 / secp256r1) or 96 bytes (BLS).
- `signature`: 64 bytes (Ed25519), 96 bytes (BLS G2), etc.
- `epoch`: varint (typically 1 byte).

Total: ~200 bytes for Ack, ~150 bytes for TipAck.

### The verification

Both Acks and TipAcks sign a payload constructed as:

```rust
fn encode_ack_payload(namespace: &[u8], height: Height, digest: &Digest) -> Vec<u8> {
    let mut buf = Vec::with_capacity(namespace.len() + 8 + 32);
    buf.extend_from_slice(namespace);
    height.encode_to(&mut buf);
    digest.encode_to(&mut buf);
    buf
}
```

The namespace separates "ack namespace" from "tip namespace" (and
from any other namespaces the protocol uses). Mismatched namespace
=> verification fails. This is the cross-protocol attack defense
(chapter 04).

---

## Deepening 7 — Test patterns for Aggregation

Three full test scaffolds with Byzantine mocks.

### Test 1: Happy path (5 honest validators, fast progress)

```rust
#[test]
fn test_aggregation_progress() {
    let runner = deterministic::Runner::default();
    runner.start(|ctx| async move {
        let mut oracle = setup_oracle(ctx.clone(), 5);
        let mut sequencer = external_sequencer(ctx.clone());

        let (engines, supervisors) = build_aggregation_stack(
            ctx.clone(),
            &mut oracle,
            PARTICIPANTS.iter().cloned().collect::<Vec<_>>(),
            Scheme::BLS_THRESHOLD,
        );

        // External sequencer emits 100 items.
        for h in 0..100 {
            let digest = random_digest(&mut ctx.rng());
            sequencer.emit(h, digest);
            for engine in &engines {
                engine.new_digest(h, digest).await;
            }
            ctx.sleep(Duration::from_millis(10)).await;
        }

        // Each engine should report 100 certificates.
        for sup in &supervisors {
            let mut last_h = Height::zero();
            while last_h < 99 {
                let update = sup.subscribe().await;
                last_h = update.last;
            }
        }
    });
}
```

What's checked: every engine observes 100 certificates, no crashes
or Byzantine behavior.

### Test 2: Skip-when-behind

```rust
#[test]
fn test_safe_tip_skip() {
    let runner = deterministic::Runner::timed(Duration::from_secs(60));
    runner.start(|ctx| async move {
        let mut oracle = setup_oracle(ctx.clone(), 4);
        let (engines, supervisors) = build_aggregation_stack(
            ctx.clone(), &mut oracle, PARTICIPANTS.clone(), Scheme::BLS_THRESHOLD,
        );

        // Start 3 validators at normal speed.
        for h in 0..50 {
            emit_digest(ctx, h);
            ctx.sleep(Duration::from_millis(10)).await;
        }

        // Hold validator A offline by removing its network links.
        for pk in &PARTICIPANTS[1..] {
            oracle.remove_link(PARTICIPANTS[0], *pk).await;
        }

        // Force 50 more items on the remaining 3.
        for h in 50..100 {
            emit_digest(ctx, h);
            ctx.sleep(Duration::from_millis(10)).await;
        }

        // Re-link A.
        for pk in &PARTICIPANTS[1..] {
            oracle.add_link(PARTICIPANTS[0], *pk, default_link()).await;
        }

        // Wait for A's safe_tip to catch up.
        let safe_tip = wait_for_safe_tip(&supervisors[0], 80).await;
        assert!(safe_tip >= 80, "safe_tip didn't advance enough");

        // A's local certificates may be sparse — that's the design.
        let local_count = supervisors[0].local_certificates().len();
        assert!(local_count < 100, "design says we skip, not backfill");
    });
}
```

What's checked: safe_tip advances past gap; the engine knows about
certificates it doesn't have locally.

### Test 3: Byzantine mocks

```rust
#[test]
fn test_byzantine_acks() {
    let runner = deterministic::Runner::default();
    runner.start(|ctx| async move {
        let mut oracle = setup_oracle(ctx.clone(), 4);

        // Replace validator 0 with a Conflicter that sends invalid sigs.
        let conflicter = mocks::conflicter::Conflicter::new(
            ctx.child("conflicter"),
            mock::ConflicterConfig {
                invalid_signature_rate: 1.0,  // always invalid
                ..Default::default()
            },
        );
        conflicter.start(oracle.take(PARTICIPANTS[0]).unwrap());

        // Three honest engines.
        let (_, supervisors) = build_aggregation_stack(
            ctx.clone(), &mut oracle, PARTICIPANTS[1..].to_vec(), Scheme::BLS_THRESHOLD,
        );

        // Emit 100 items.
        for h in 0..100 {
            emit_digest(ctx, h);
            ctx.sleep(Duration::from_millis(10)).await;
        }

        // Honest engines should each reach at least 99 certificates.
        for i in 1..4 {
            let count = supervisors[i].certificate_count();
            assert!(count >= 99, "honest validator {} only got {} certs", i, count);
        }
    });
}
```

What's checked: honest quorum forms despite Byzantine attempts.


## Exercises — Aggregation

A set of exercises covering Aggregation's deeper behaviors.

### Exercise 13.1 — Aggregation bandwidth math

For 100 validators, 1000 items, BLS threshold:

1. **Ack traffic (gossiped)**: with BLS threshold, each Ack is ~200
   bytes. If you gossip all Acks to all peers: 100 × 1000 × 200 = 20 MB.
2. **TipAck traffic (gossiped)**: 1 TipAck per validator per second
   for 10 seconds, 200 bytes each: 100 × 10 × 200 = ~200 KB.

Compare to a Setup where you re-gossip full Certificates
(constructed from BLS threshold = ~96 bytes): 1000 × 96 = ~96 KB.
But that doesn't tell you about the *progress* safely.

Why is the TipAck design a good trade-off?

### Exercise 13.2 — safe_tip progression

For 4 validators (n=4, f=1), with tips `[v=10, v=12, v=11, v=15]`:

1. Sort descending: `[15, 12, 11, 10]`.
2. f+1-th (0-indexed f = 1): `11`.
3. `safe_tip = 11`.

Concretely: at least `f+1 = 2` validators are at height ≥ 11.
Of those 2, at least 1 is honest (max Byzantine = 1). The honest
validator has progress ≥ 11.

### Exercise 13.3 — Epoch-independent signature verification

Given:
- A Certificate for (height = 100, digest = D).
- The corresponding epoch = 5.
- The light client has a snapshot of `Set(5)` and `pk_agg(5)`.

Verify the Certificate:

```rust
let cert = load_cert(height = 100, epoch = 5);
let snapshot = epoch_snapshots.get(&Epoch::new(5)).expect("snapshot");
let valid = e(cert.aggregate, g2) == e(H(100, D), snapshot.aggregated);
```

Note: the snapshot's `aggregated` is the *sum* of all
`Set(5)` validator pubkeys. Pre-summing avoids per-validator cost
at verification time.

### Exercise 13.4 — Cross-chain bridge cost analysis

For a bridge from Solana to Ethereum, processing 10 events/sec:

1. **BLS threshold (1 pairing per cert)**: ~500 µs/event.
2. **Ed25519 multisig (100 sigs/cert)**: ~5 ms/event.
3. **secp256r1 (100 sigs/cert)**: ~3 ms/event.

For Ethereum's gas budget, the BLS pairing costs ~150k gas (with
precompile). The multisig requires 100 individual signature
verifications (~3M gas each), totaling ~300M gas = cost-prohibitive.

Why does this affect which scheme bridges adopt?

### Exercise 13.5 — CT to Aggregation mapping

For each CT concept, find the Aggregation equivalent:

| CT concept | Commonware Aggregation |
|---|---|
| Log entry | ? |
| STH | ? |
| SCT (signed certificate timestamp) | ? |
| Consistency proof | ? |
| Audit | ? |

(Answer: entry = `(h, D)`; STH = `Certificate`; SCT = `Ack`;
consistency proof = the BLS public key snapshot; audit =
light-client verification.)

### Exercise 13.6 — Aggregation test debugging

Given a failing test (5 validators, 1 Byzantine, no progress):

```rust
// Test code:
let (supervisors, _) = build_aggregation_stack(...);
for h in 0..100 {
    emit_digest(h);
    ctx.sleep(Duration::from_millis(10)).await;
}
// At this point, supervisors[0..5] each should report 100 certificates.
for sup in &supervisors {
    assert_eq!(sup.certificate_count(), 100);
}
```

The assertion fails for 4 of 5 supervisors (only #2 reports 100).
What's wrong? (Answer: validator 2 is Byzantine, sends invalid
Acks. The other 4 are honest but never form a quorum because the
Byzantine one is included.)

How would you fix the test? (Answer: ensure the quorum is among
the honest 4, not all 5. The BLS threshold scheme should still
produce a Certificate from any 3 of 4 honest.)

---


## Appendix A — The Aggregation Engine, in detail

`consensus/src/aggregation/engine.rs`. State and main loop:

```rust
struct AggregationEngine<S: Scheme> {
    // Configuration
    config: Config,
    scheme: S,
    automaton: Automaton,
    reporter: Reporter,

    // Local state
    local_tip: Height,
    safe_tip: Height,
    my_signatures: HashMap<Height, S::Signature>,  // signatures I've sent
    acks: HashMap<Height, BTreeMap<PK, Ack<S>>>,  // acks received, per height
    tips: BTreeMap<PK, (Height, S::Signature)>,    // latest tips from peers

    // Mailboxes
    mailbox: Mailbox<Message>,
    sender: Sender,
    // ...
}
```

### Main loop

```rust
async fn run(mut self) {
    loop {
        select! {
            // External: new digest at height H
            digest = self.automaton.next_digest() => {
                self.on_new_digest(digest).await;
            }

            // P2P: incoming ack
            ack = self.ack_receiver.recv() => {
                self.on_ack(ack).await;
            }

            // P2P: incoming tip
            tip = self.tip_receiver.recv() => {
                self.on_tip(tip).await;
            }

            // Epoch transition
            epoch = self.epoch_signal => {
                self.on_epoch_change(epoch).await;
            }

            // Shutdown
            _ = self.stop_signal => break,
        }
    }
}
```

## Appendix B — The `on_new_digest` flow

When the external sequencer emits a new digest at height H:

```rust
async fn on_new_digest(&mut self, digest: Digest) {
    if H <= self.local_tip {
        return;  // Already processed
    }

    self.local_tip = H;

    // Sign and broadcast
    let sig = self.scheme.sign(namespace, encode_item(H, digest));
    self.my_signatures.insert(H, sig.clone());
    self.sender.send(Recipients::All, Ack { height: H, digest, signature: sig }, false).await;
}
```

Now I have a signature for `(H, digest)`. I broadcast to peers.

## Appendix C — The `on_ack` flow

When a peer responds with their ack:

```rust
async fn on_ack(&mut self, ack: Ack<S>) {
    // Verify the signature
    if !self.scheme.verify(&ack.signer, namespace, encode_item(ack.height, ack.digest), &ack.signature) {
        // Block the peer
        return;
    }

    // Insert into acks
    let entry = self.acks.entry(ack.height).or_insert_with(BTreeMap::new);
    entry.insert(ack.signer, ack);

    // Check for quorum
    if entry.len() >= QUORUM {
        let cert = assemble_certificate(entry.values(), &self.scheme);
        self.reporter.report(Certificate { height: ack.height, cert }).await;
    }
}
```

Same Collector pattern as Simplex's Batcher. Quorum → assemble
certificate → notify Reporter.

## Appendix D — The `safe_tip` advancement

`consensus/src/aggregation/safe_tip.rs`. The safe tip is the height
above which we trust certificates exist somewhere in the network.

```rust
fn advance_safe_tip(&mut self) {
    // Get all reported tips from peers
    let tips: Vec<Height> = self.tips.values().map(|(h, _)| *h).collect();
    tips.sort();

    // The f+1-th highest tip is the safe tip (any f are Byzantine)
    let safe_tip = tips.get(tips.len() - (F + 1)).copied().unwrap_or(0);

    if safe_tip > self.safe_tip {
        self.safe_tip = safe_tip;
    }
}
```

The **f+1-th highest** ensures at least one honest validator has reached
that height. Even if `f` validators are lying about being at height
`safe_tip`, the remaining `f+1` (at least one honest) actually are.

## Appendix E — The `Ack` and `TipAck` types

`consensus/src/aggregation/types.rs`:

```rust
pub struct Ack<S: Scheme> {
    pub height: Height,
    pub digest: Digest,
    pub signer: PK,
    pub signature: S::Signature,
    pub epoch: Epoch,         // unauthenticated metadata
}

pub struct TipAck<S: Scheme> {
    pub height: Height,
    pub digest: Digest,
    pub signer: PK,
    pub signature: S::Signature,  // over TipAck's own content
}

pub struct Certificate<S: Scheme> {
    pub height: Height,
    pub digest: Digest,
    pub certificate: S::Certificate,  // aggregated
}
```

`Ack` certifies `(height, digest)`. `TipAck` certifies the validator's
**current tip** (height + digest). `Certificate` is the aggregated form
(emitted when quorum reached).

## Appendix F — Why epoch-independent signatures work

`mod.rs:69-82`:

> The attestation in an Ack covers only the Item (height and digest),
> not the epoch, and the ack namespace does not rotate per epoch. This
> is intentional: participants attest to an externally agreed-upon log
> that does not change across epochs, so an attestation to an item is
> valid regardless of the epoch in which it was produced.

So a signature `(H, digest)` from epoch 5 is still valid in epoch 10.
The validator set can change; the attestations don't.

The `epoch` field on `Ack` is unauthenticated metadata. It's used by
the receiver to select the right scheme/verifier, but isn't part of
the signed content.

## Appendix G — The safe tip and skipping

When `safe_tip` advances past `local_tip`:

- We've confirmed "certificates exist up to safe_tip somewhere."
- We may not have those certificates ourselves.
- We won't backfill (per design).

```rust
fn on_safe_tip_advance(&mut self) {
    // Skip past missing heights
    while self.local_tip < self.safe_tip {
        self.local_tip += 1;
        // Don't expect a certificate — the network has it, but we don't
    }
}
```

The local state advances; we won't sign or expect certificates for
skipped heights.

## Appendix H — Test patterns

```rust
fn test_aggregation_progress() {
    // Setup: 4 validators, aggregation engine each
    // External sequencer emits 100 items
    // Verify: each validator reports 100 certificates
    // Verify: all certificates agree on (height, digest)
}

fn test_safe_tip_skip() {
    // Setup: 4 validators
    // Validator A offline while B/C/D progress 1-50
    // Validator A comes back at height 50
    // Verify: A's safe_tip advances to 50 quickly
    // Verify: A doesn't have certificates for 1-49 (skip is OK)
    // Verify: A signs new items starting from 50
}

fn test_byzantine_acks() {
    // Setup: 4 validators, one is Byzantine
    // Byzantine sends invalid signatures
    // Verify: Byzantine is blocked
    // Verify: Honest validators reach quorum from remaining 3
}
```

## Appendix I — Common gotchas

### The signature aggregation mismatch

If you use threshold BLS, `S::Certificate` is a threshold sig. If you
use multisig, it's an aggregated multisig + signer indices. The reporter
must handle both. Use the `Scheme::Certificate` type to abstract.

### The `Epoch` field is not authenticated

Don't use the unauthenticated `epoch` field for anything that requires
trust. It's a hint, not a commitment.

### The activity_timeout

`activity_timeout` (a `ViewDelta`) controls how long the engine waits
before considering a height "skipped." If too small, you skip heights
that are about to produce certificates. If too large, you wait forever
for stalled peers. Tune for your network.

## Where to look in the code (expanded)

- `consensus/src/aggregation/mod.rs:1-82` — the architecture.
- `consensus/src/aggregation/engine.rs` — the main loop.
- `consensus/src/aggregation/safe_tip.rs` — safe tip advancement.
- `consensus/src/aggregation/scheme.rs` — the signature schemes.
- `consensus/src/aggregation/types.rs` — the message types.
- `consensus/src/aggregation/config.rs` — the config knobs.

## If you only remember three things

1. **Aggregation = "collect signatures over an externally-ordered log."** Same Collector pattern as Simplex, different source of items.
2. **No backfill.** Tips skip forward; at least one honest validator has each skipped certificate.
3. **Epoch-independent signatures.** Attestations don't bind to an epoch; they survive validator set changes.

→ Next: **Chapter 14 — Ordered Broadcast**. The third consensus-flavored
primitive: a sequencer broadcasts a chain of chunks, validators link them
with certificates, observers can prove availability.