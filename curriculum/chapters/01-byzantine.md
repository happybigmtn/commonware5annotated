# Chapter 01 — Byzantine Fault Tolerance

> The hardest part of distributed systems, made simple.

## The distributed systems landscape — where Byzantine fits

Before diving into Byzantine fault tolerance, it's worth knowing the four
canonical problems in distributed agreement, in order of difficulty.

### 1. Two Generals Problem (impossible)

Two generals must coordinate an attack across a hostile valley. They can
only communicate by messenger. The messenger might be captured (lost).
**They can never reach certainty about whether their message was received.**

Proof: general A sends "let's attack at dawn." For A to be certain B got
the message, B must acknowledge. For B to be certain A got the
acknowledgment, A must acknowledge the acknowledgment. And so on. There's
always one more "did they get it" to worry about.

This is fundamentally a problem of **no common knowledge in asynchronous
systems with unreliable channels.** It's why HTTP doesn't have a "did you
get my request" ACK beyond TCP — TCP itself never knows if the last ACK
arrived.

### 2. Byzantine Generals Problem (solvable with n ≥ 3f+1)

Same as Two Generals, but **multiple generals and at least one is a traitor**
who actively lies. With enough generals, the loyal ones can reach agreement
despite the traitor's lies. Commonware is built around this.

### 3. FLP Impossibility (determines what you can promise)

The Fischer-Lynch-Paterson result from 1985: in a purely asynchronous system
with even one crash failure, **there is no deterministic consensus protocol
that always terminates.** This is fundamental — it's not a limitation of
your code, it's a theorem about distributed systems.

The escape hatches:

- **Randomized protocols** (e.g., Ben-Or's): use coin flips to break
  symmetry. Eventually terminate with probability 1.
- **Partial synchrony** (DLS, PBFT): assume the network eventually delivers
  messages within some bounded delay `Δ`. Simplex uses this.
- **Synchrony**: assume a known upper bound on message delay. Too strong
  for real networks.

Commonware's Simplex is **partially synchronous**: it assumes messages
eventually arrive within `Δ`, but `Δ` is not known. The protocol guarantees
**safety always** (no two conflicting blocks finalized) and **liveness when
the network is synchronous** (progress happens within a bounded time).

### 4. CAP Theorem (which trade-off do you choose)

Eric Brewer's CAP theorem (2000): a distributed data store can have at
most **two** of:

- **C**onsistency: every read sees the latest write.
- **A**vailability: every request gets a response (no error).
- **P**artition tolerance: the system keeps working when the network splits.

In practice you must choose P (networks fail). So you choose between CP and AP:

- **CP** systems (e.g., etcd, ZooKeeper, Simplex): refuse writes during
  partitions. Linearizable.
- **AP** systems (e.g., Cassandra, DynamoDB): keep accepting writes during
  partitions. Eventually consistent.

Simplex is **CP**. During a network partition, validators in the minority
partition make no progress. They don't fork — they just stop. When the
partition heals, they catch up via the Resolver (chapter 08).

### The safety vs liveness distinction

These two properties show up everywhere:

- **Safety**: "nothing bad happens." No two conflicting blocks finalized.
  No two honest nodes disagree.
- **Liveness**: "something good eventually happens." Some block gets
  finalized. Progress happens.

Simplex guarantees:
- **Safety always** — works in any network.
- **Liveness under partial synchrony** — works when messages eventually
  arrive within some bounded delay `Δ`.

The deeper theorem (DLS — Keidar, Naor, Shapiro 2011): any consensus
protocol that is safe under all conditions can only be live under some
synchrony assumption. You can't have both unconditionally.

## What is Byzantine, and why does it make everything hard?

**The easy version of distributed systems:** "Crash fault tolerance." Some nodes
die. They just stop responding. Easy — if a node doesn't respond after a timeout,
assume it's dead, keep going.

**The hard version:** "Byzantine fault tolerance." Some nodes are **actively
malicious**. They respond fast, but they lie. They send different messages to
different people. They coordinate. They're the enemy.

Why is this so much harder? Let me give you the canonical puzzle — the
**Byzantine Generals Problem**.

### The Generals Puzzle

Imagine you're a general besieging a city. You have 4 generals total (including
you) camped on different hills. You can only communicate by messenger. You need
to all attack at dawn, **or** all retreat — but if some attack and some retreat,
you lose.

One of the generals is a traitor. He might tell General A "attack at dawn" and
General B "retreat at dawn." He'll do anything to make the loyal generals
disagree.

**Question:** Can you, a loyal general, always figure out what the loyal majority
decided?

The answer (proved by Lamport, Shostak, Pease in 1982) is yes — but you need a
specific number of messengers. **The magic formula is `n ≥ 3f + 1`** where `f`
is the number of traitors. So if you have 1 traitor, you need 4 generals total.
If you have 2 traitors, you need 7. Always 1/3 margin.

**Why 1/3?** Because the loyal generals need to outvote the traitors **plus** the
votes the traitors can fake. If you have `f` traitors, they can produce `f` fake
votes. The loyal generals need their `n - f` votes to overwhelm `2f` total (real
traitor votes + fake votes). So `n - f > 2f`, meaning `n > 3f`.

### Why n ≥ 3f+1, formally

Let `n` be the total number of nodes, `f` the number of Byzantine ones, and
`q` the quorum size for some decision.

For safety, two quorums must overlap in at least one honest node (so they
can't disagree). With `f` Byzantine nodes, an attacker controls up to `f`
votes. Two quorums of size `q` might be entirely disjoint if:

- Quorum 1 contains `q` nodes, including up to `f` Byzantine.
- Quorum 2 contains `q` nodes, including up to `f` Byzantine.

For overlap: `2q > n + f`. We want this to hold in the worst case. Setting
`q = 2f + 1` and `n = 3f + 1`: `2q = 4f + 2`, `n + f = 4f + 1`. So
`4f + 2 > 4f + 1` — overlaps by at least 1.

Smaller quorums break this:
- If `n = 3f`, then `q ≤ f`. Two disjoint `f`-sized quorums exist.
- If `n = 3f + 1`, you need `q ≥ 2f + 1` to guarantee overlap.

This is also why **public blockchains with 1/3 hostile stake halt** — they
can't safely reach quorum.

### How Commonware encodes this

Look at `consensus/src/simplex/mod.rs` — when it says "2f+1" everywhere, that's
the quorum. With `n = 3f+1`, you have `2f+1` honest votes out of `3f+1` total.
That's just over 2/3 — enough to overwhelm `f` traitors.

Look at the `N3f1` and `Faults` types in `commonware-utils`:

```rust
N3f1::quorum(n)   // returns the smallest count that beats f traitors
```

That's the math showing up in the code.

### A worked example: 4 nodes, 1 traitor

Say we have `n=4` validators. `f = floor((4-1)/3) = 1`. We tolerate 1 traitor.
We need `2f+1 = 3` votes to make progress.

| Scenario | What happens |
|---|---|
| Leader is honest, everyone votes the same | 4 `notarize` messages → 1 notarization |
| Leader is honest, 1 honest node offline | 3 `notarize` messages → 1 notarization ✓ |
| Leader is the traitor | Honest nodes time out → `nullify` → 3 `nullify` messages → skip view |
| Traitor sends different blocks to different nodes | Each honest node broadcasts its received block; 3 honest votes converge on one block; traitor isolated |

### Adversary models — what the traitor can do

The Byzantine Generals paper assumes a powerful adversary. Commonware
assumes the same — but the assumptions vary. Here's the taxonomy:

| Model | Capability | Used in |
|---|---|---|
| **Crash-only** | Node stops responding. No lies. | Paxos, Raft. Easier to handle. |
| **Byzantine, static** | A fixed set of `f` nodes are adversarial. They can do anything — lie, equivocate, collude. The set is chosen at startup and doesn't change. | Most BFT protocols including Simplex. |
| **Byzantine, adaptive** | The adversary can corrupt any node at any time, up to `f` total corruptions. | Stronger model, harder to handle. |
| **Byzantine, mobile** | The set of corrupted nodes changes over time. | Rarely used in practice. |
| **Byzantine, with rushing** | Adversary can see your message before deciding what to send in response. (All real networks are rushing.) | Always assumed. |
| **Sleepy** | Honest nodes can be temporarily offline (sleep). Adversary corrupts only awake nodes. | Some newer protocols. |
| **Rational** | Nodes are self-interested, follow protocol only when profitable. | Mechanism design. |

Commonware's Simplex uses **static Byzantine with rushing**. The set of `f`
adversarial nodes is fixed at startup (or per epoch). They can read your
messages before responding (the network is rushing).

This is the same model PBFT, Tendermint, HotStuff use. The "right" model for
blockchain consensus.

### The communication complexity trade-off

Different BFT protocols have different message counts per decision:

| Protocol | Messages per decision (best case) | Notes |
|---|---|---|
| **PBFT** (Castro-Liskov 1999) | O(n²) | Pre-prepare, prepare, commit. Every node talks to every node. |
| **Tendermint** (Buchman 2016) | O(n²) | Same as PBFT, but with rounds and locked blocks. |
| **HotStuff** (Yin et al. 2019) | O(n) per view × O(log n) views | Linear per view via threshold signatures. |
| **Simplex** (Das-Xiang-Ren 2023) | O(n) per view × O(1) views | Linear per view. One-hop notarize, one-hop finalize. |

Simplex's O(n) per view (instead of O(n²)) comes from BLS threshold
signatures: one signature can be verified as "at least 2f+1 of these
public keys signed." No need for every node to forward every vote.

This is the engineering difference. PBFT with 100 nodes sends 10,000
messages per decision. Simplex with 100 nodes sends 300 messages per
decision (100 votes, 100 forwarders, ~100 ack — but BLS lets you compress
to ~3 messages per view if you aggregate).

### Simplex Consensus: the "fast BFT" recipe

Now you have `2f+1` honest nodes. How do you actually agree on something? Look
at `consensus/src/simplex/mod.rs:30-68` — that's the entire algorithm. Let me
walk you through it.

You have a **view** (think: round number). In each view, there's a **leader** —
one node picked to propose a block. The leader broadcasts a `notarize(c, v)`
message — "I propose blob `c` for view `v`."

Every honest node is sitting there with two timers running:

- `t_l = 2Δ` — "if the leader hasn't proposed after 2 network hops, skip them"
- `t_a = 3Δ` — "if we can't even agree after 3 network hops, give up on this view"

Here's the dance, view by view:

1. **Enter view v.** Find the leader. Start timers. If you're the leader,
   propose a blob and broadcast `notarize(c, v)`.

2. **Hear the leader's `notarize(c, v)`.** Verify `c` (check its parent is
   legit, etc.). If good, broadcast your own `notarize(c, v)`. If bad, broadcast
   `nullify(v)` — "this view is dead to me."

3. **Collect `2f+1` `notarize(c, v)` messages.** That's a **quorum**. Cancel
   `t_a`. You've got a `notarization(c, v)` — a *certificate* that everyone (or
   near-everyone) agrees `c` is the proposed block. Now ask your application:
   "can I certify this?" (more on this in a sec). If yes, broadcast
   `finalize(c, v)`. If no, broadcast `nullify(v)`.

4. **Collect `2f+1` `finalize(c, v)` messages.** That's a
   `finalization(c, v)` — **irreversible**. Block `c` is canon. Enter view
   `v+1`.

5. **If `t_l` fires** (leader is silent or slow): broadcast `nullify(v)`.

6. **If you collect `2f+1` `nullify(v)` messages**: that's a
   `nullification(v)`. Move to `v+1`. The view was a wash; try again.

That's it. **Three message types** (notarize, nullify, finalize), **two
timers**, **two quorums**. The genius is that **2 hops gets you a notarization**
(a "soft" agreement), and **3 hops gets you finalization** (irreversible).
Compare to PBFT (Practical Byzantine Fault Tolerance) which needs 5+ hops.

### The "certify" hook — the magic escape hatch

Step 3 has that weird line: "ask your application if you can certify." Why?

Because sometimes `2f+1` nodes say "notarize this block" — but they may not
have the actual block data yet (imagine a big block, still being broadcast). You
don't want to **finalize** a block whose contents you haven't verified. So
`certify` is your chance to say "I have the block, I've checked it, it's valid
— yes, finalize."

If your application says "no, I don't have the block" — you nullify the view
instead. Try again. The leader in `v+1` will propose a block you can probably
get.

This is the **decoupling** in action: consensus doesn't know what `c` is. It
just knows `c`'s hash. The application gets to decide when to commit to the
contents.

### The asymmetric insight — "Unchained Finalization"

Look at this part of the spec (`mod.rs:156-168`):

> A payload notarized in view `v` can be finalized while the network is in
> view `v+k` for any `k >= 1`. There is no requirement that a particular view
> after `v` succeeds.

What does this mean in plain English? Say view 5 finalized block B. Then views
6, 7, 8, 9 all go to hell — timeouts, partitions, whatever. The system is now
stuck in view 9. But the finalization for B **still lands** because those
`finalize(B, 5)` messages are independent of what happens later. They're like
a separate pile of mail that's already en route.

This is **why Simplex is so fast** — failed views don't block earlier successes.

### Forced Inclusion (Tail-Forking Resistance)

The other brilliant property (`mod.rs:124-137`):

> A notarized payload in view `v` must appear in the canonical chain if no
> nullification certificate exists for `v`.

Why? Because to propose a block in view `v+5`, the leader must reference a
certified parent in some earlier view `v_p` AND have nullification certificates
for every view between `v_p` and `v+5`. If view `v` only has a notarization
(not a nullification), the leader **must include** that notarized block as an
ancestor.

This means a malicious leader can't "skip over" your block. If it gets
notarized, it stays in the chain. Tail-forking resistance for free.

### Optimistic Finality

`mod.rs:139-152` reveals yet another trick:

> Once a notarization certificate is observed for view `v` (without any timeout
> having fired), the notarized payload can be treated as speculatively final.

This "speculative finality" is available after just 2 network hops (proposal +
notarization), compared to the 3 hops required for full finalization.

In plain English: if you see `2f+1` votes and no timeouts fired, you can bet
the block is going to be finalized. You can act on it. If you're wrong, you
lose money — but in the common case (no faults, no timeouts), you're right.

## The architecture (one diagram, one sentence)

```
                 +------------+          +++++++++++++++
                 |            +--------->+             +
                 |  Batcher   |          +    Peers    +
                 |            |<---------+             +
                 +-------+----+          +++++++++++++++
                     |   ^
                     |   |
                     v   |
+---------------+   +---------+   +++++++++++++++
|               |<--+         +-->+             +
|  Application  |   |  Voter  |   +    Peers    +
|               +-->|         |<--+             +
+---------------+   +--+------+   +++++++++++++++
                     |   ^
                     |   |
                     v   |
                 +-------+----+   +++++++++++++++
                 |            +-->+             +
                 |  Resolver  |   +    Peers    +
                 |            |<--+             +
                 +------------+   +++++++++++++++
```

Three components: **Batcher** (collect votes, batch-verify at quorum),
**Voter** (drive participation in the current view), **Resolver** (fetch missing
certificates). The **Application** is yours — it proposes blocks and certifies
them.

## Where to look in the code

- `consensus/src/simplex/mod.rs:1-67` — the entire protocol spec.
- `consensus/src/simplex/mod.rs:124-168` — the three deep properties
  (forced inclusion, optimistic finality, unchained finalization).
- `consensus/src/simplex/mod.rs:180-207` — the architecture diagram in ASCII.
- `consensus/src/lib.rs` — the trait surface (Automaton, CertifiableAutomaton,
  Reporter, Monitor, etc.) — what YOU have to implement to plug your
  application in.
- `consensus/src/simplex/scheme/` — the four supported signature schemes
  (ed25519, bls12381 multisig, secp256r1, bls12381 threshold).
- `consensus/src/simplex/elector.rs` — leader election (round-robin or
  VRF-based random).

## The full Byzantine Generals derivation

The Lamport-Shostak-Pease 1982 paper ("The Byzantine Generals Problem",
ACM TOPLAS) is the foundation. Let's walk through it formally.

### Setup

There are `n` generals, each commanding a division of the Byzantine army.
They encircle a city. They communicate only by messenger. They must
agree on a common plan of action: **attack** or **retreat**. Some
generals (`f` of them) are traitors. The loyal generals must all reach
the same decision; otherwise the attack fails.

The traitors can do anything: lie, equivocate, collude, withhold
messages. The loyal generals follow the protocol exactly.

We want:
- **Agreement**: all loyal generals decide on the same value.
- **Validity**: if all loyal generals start with the same initial value
  `v`, they all decide `v`.

In the oral-messages (OM) model, messages are not signed. The traitors
can fabricate messages. We want a protocol that works in this model.

### OM(0) — base case

If `f = 0` (no traitors), the protocol is trivial: the commanding
general sends his value to all lieutenants, they all decide on it. They
agree because they all saw the same message.

### OM(m) — induction step

For `f = m`, the commanding general sends his value `v_C` to every
lieutenant. Each lieutenant then acts as the "commander" in a recursive
protocol, gathering values from every other lieutenant via OM(m-1).

In detail: commander `C` sends his value `v_C` to all `n-1`
lieutenants. Each lieutenant `L_i` collects the value `v_i` he received
from `C` (or "no value" if he got nothing). Then `L_i` runs OM(m-1) with
all other lieutenants to compute `act_i(j)` — "the value that lieutenant
`L_j` says `L_i` should act on." (Each lieutenant plays "commander" of
his own sub-protocol.)

Finally, `L_i` takes the majority of `{v_i, act_i(1), act_i(2), ..., act_i(n-1)}`.

### Proof of correctness

By induction on `m`. The base case `m=0` is trivial. For the inductive
step: assume OM(m-1) works with `3(m-1)+1 = 3m-2` lieutenants (so
`3m-1` total participants). For OM(m) we need to show:

**Case 1: The commander is loyal.** He sends his value `v` to all
lieutenants. All lieutenants receive `v`. In the OM(m-1) sub-protocols,
each lieutenant `L_i` reports the value he got from `C`. Because `C` is
loyal, all loyal lieutenants receive `v` from `C`. Since the OM(m-1)
protocol works, all lieutenants agree on `act_i(j) = v` for every `j`.
Majority of `{v, v, ..., v}` is `v`. All decide `v`. ✓

**Case 2: The commander is a traitor.** He may send different values to
different lieutenants. The loyal lieutenants don't know the commander
is a traitor — they think they're in Case 1 with some unknown value `v`.
The OM(m-1) sub-protocols reveal what each loyal lieutenant thinks it
received. By the inductive hypothesis, all loyal lieutenants agree on
`act_i(j)` for each `j`. So all loyal lieutenants compute the same
majority and decide the same value. ✓

### Why `n ≥ 3f + 1`

For the OM(m-1) sub-protocol to work with `m-1` traitors among the
`n-1` lieutenants, we need `n-1 ≥ 3(m-1) + 1 = 3m - 2`. So `n ≥ 3m - 1`.

The correct count: each OM(m) instance needs `3m + 1` generals total.
Of these, at most `m` can be traitors. The commander is one of the
`3m + 1`. The lieutenants are the other `3m`. Of these `3m`
lieutenants, at most `m-1` can be traitors (one traitor was the
commander). The sub-protocol is OM(m-1), which needs `3(m-1) + 1 = 3m - 2`
lieutenants. But we have `3m` lieutenants. The extra `2` are buffer.

The exact bound `3m + 1` comes from a counting argument:

- We need a quorum of size `q` such that any two quorums intersect in
  at least one *loyal* member.
- With `f` traitors, two quorums of size `q` overlap by at least
  `2q - n`. To guarantee at least 1 loyal overlap: `2q - n > f`, i.e.,
  `q > (n+f)/2`.
- Setting `q = 2f + 1` (the smallest integer satisfying this), we get
  `2f + 1 > (n+f)/2`, i.e., `n < 3f + 2`, i.e., `n ≤ 3f + 1`.
- So `n ≥ 3f + 1` is necessary *and* sufficient.

That's the derivation. The intuition: traitors can fake `f` votes.
The loyal majority needs `2f + 1` votes to overwhelm `f` real traitors
+ `f` fake votes. So you need `n - f ≥ 2f + 1`, hence `n ≥ 3f + 1`.

### Signed messages (SM)

The paper also defines the *signed messages* model, where loyal
generals' messages cannot be forged by traitors. In SM, the bound is
weaker: `n ≥ f + 2` (i.e., you can tolerate up to `n-2` traitors).
This is why real BFT systems use signatures extensively — they shrink
the required replication factor. PBFT, Tendermint, HotStuff, Simplex
all use signatures.

Commonware's Simplex uses ed25519 (small signatures) or BLS12-381
threshold (aggregate signatures), both of which give the *signed*
model: `n ≥ 2f + 1`. (The `3f+1` bound is the *oral* model with no
signatures; with signatures, you only need 2f+1.)

## The FLP impossibility in full

Fischer, Lynch, Paterson (1985). "Impossibility of Distributed
Consensus with One Faulty Process." The result: in a fully asynchronous
network with even a single crash fault, *no deterministic* consensus
protocol can guarantee termination.

### Setup

- `n` processes, one of which may crash (stop responding).
- Asynchronous network: messages can be delayed arbitrarily.
- Each process proposes a value; they must agree on one.
- Required: agreement (all non-faulty decide the same), validity
  (if all propose `v`, all decide `v`), termination (every non-faulty
  process eventually decides).

### Proof sketch

The proof is a classic distributed-systems gem. Sketch:

**Step 1: Define *configurations*.** A configuration is the local state
of every process, plus the set of undelivered messages.

**Step 2: Define *decision values*.** A configuration `C` has decision
value `v` if from `C` onwards, every possible execution results in
all non-faulty processes deciding `v`. (Configurations are committed
to a decision.)

**Step 3: Show there must be an *initial bivalent* configuration.**
A configuration is *bivalent* if both decision values `0` and `1` are
still possible. A *univalent* configuration has only one possible
decision value.

Claim: starting from any initial configuration where not all processes
propose the same value, there exists some sequence of events leading
to a bivalent configuration. Why: if all initial configurations were
univalent, then by varying which process is the "first to take a step"
or which messages are "first delivered," you can drive the protocol to
both decisions — contradiction.

**Step 4: Show that from any bivalent configuration, there is a step
that keeps the configuration bivalent.** Take a bivalent `C`. There
exist two single-event extensions `C s` and `C t` such that `C s`
is 0-valent and `C t` is 1-valent. The argument gets technical here,
but the punch line: if the network delays the messages, the protocol
can be kept bivalent indefinitely.

**Step 5: If the protocol is to terminate, some process must decide
while bivalent.** But deciding commits the configuration to one value.
The adversary chooses the message order to prevent commitment. So
termination is impossible.

### What FLP doesn't say

FLP proves impossibility of *deterministic* consensus under
*asynchrony*. It doesn't say consensus is impossible in practice.
Three escape hatches:

1. **Randomization.** Ben-Or 1983: use coin flips to break symmetry.
   Expected termination; probability 1 of eventual termination.
2. **Partial synchrony.** DLS 1988: assume messages eventually arrive
   within `Δ`, but `Δ` is unknown. Safety holds always; liveness holds
   after GST. Commonware's Simplex uses this.
3. **Synchrony.** Strong assumption — known bound on `Δ`. Easier
   protocols possible, but unrealistic for global networks.

Commonware uses partial synchrony. Simplex is *safe* always (no forks
ever); *live* under partial synchrony (when messages arrive within
the bound, progress happens).

## The DLS impossibility

Dwork, Lynch, Stockmeyer (1988). "Consensus in the Presence of
Partial Synchrony." Three models:

- **Asynchronous (ASYNCH)**: no bounds on message delays.
- **Partially synchronous (PART-SYNCH)**: there is a Global
  Stabilization Time (GST) after which messages arrive within known
  bound `Δ`, but GST and `Δ` are unknown.
- **Synchronous (SYNCH)**: messages always arrive within known `Δ`.

The paper proves:

**Theorem 1.** There is no consensus protocol that is *live* in
ASYNCH. (This is FLP, refined to the partial-synchrony setting.)

**Theorem 2.** Any consensus protocol that is *safe* in all three
models can be *live* in at most one of them. I.e., you can have
ASYNCH + safe, or PART-SYNCH + safe + live, or SYNCH + safe + live —
but not ASYNCH + safe + live.

**Theorem 3.** The minimum number of processes for PART-SYNCH BFT
consensus is `n ≥ 2f + 1`. The minimum for SYNCH is `n ≥ f + 1` (or
even lower with more assumptions).

**Theorem 4.** No consensus protocol can solve the *Two Generals*
problem under any model. (This is unsurprising — Two Generals is a
single-round coordination problem, and there's no way to guarantee
a single message gets through.)

### Why DLS matters for Commonware

Simplex matches the DLS PART-SYNCH upper bound exactly: `n ≥ 2f + 1`,
with `f = floor((n-1)/3)` for safety. (The 3f+1 comes from the oral-
messages model; with signatures, you only need 2f+1.) Simplex is
safe in all three models; live in PART-SYNCH only. That's the DLS
trade-off made explicit.

## A worked Simplex example with n=4, f=1

Let's trace a full Simplex run with 4 validators: A, B, C, D.
Assume leader rotation: view 1 = A, view 2 = B, view 3 = C, view 4 = D.
Assume all four are honest except D, which is Byzantine.

### View 1 (leader A) — successful

**Time t=0:**
- A is leader. A's application produces a payload, hashes it to `c1`,
  and broadcasts `notarize(c1, 1)` to B, C, D.

**Time t=Δ (one network hop later):**
- B receives `notarize(c1, 1)`. B's Voter verifies `c1` (calls
  application to check parent, signatures, etc.). B broadcasts
  `notarize(c1, 1)` to A, C, D.
- Same for C: receives, verifies, broadcasts.
- D (Byzantine): receives but does nothing. Or: receives and broadcasts
  `notarize(c2, 1)` where `c2 != c1` — trying to fork.

**Time t=2Δ (two network hops):**
- A has received from B, C, D. So A has votes: B's `c1` + C's `c1` +
  D's `c2` = 2 votes for `c1` + 1 for `c2`.
- B has A's `c1` + C's `c1` + D's `c2` = 2 for `c1`.
- C has A's `c1` + B's `c1` + D's `c2` = 2 for `c1`.

So 3 honest nodes have 2 votes each for `c1`. They need a 3rd honest
vote — but there are only 3 honest nodes total. Wait: with n=4, f=1,
quorum = 2f+1 = 3. So we need 3 *valid* votes. The honest 3 form a
quorum among themselves.

**Time t=2Δ + ε (slightly later):**
- A, B, C each have 3 votes for `c1` (from the honest set {A, B, C}).
  Each forms a `Notarization { view: 1, payload: c1, certificate:
  thr_sig(A, B, C) }`.
- Each broadcasts `finalize(c1, 1)`.
- D receives the finalizes but doesn't broadcast its own (it's
  Byzantine).

**Time t=3Δ (three hops):**
- A, B, C have received 2 finalizes for `c1` from the other honest
  nodes. Each needs 3 to form a `Finalization`.
- A has its own `finalize` + B's + C's = 3. Form `Finalization { view: 1,
  payload: c1 }`.
- Same for B and C.
- View 1 is FINALIZED.

**Time t=3Δ + ε:**
- All nodes enter view 2. Leader is B.

The full cycle: 3 hops, 2 certificate constructions (notarization +
finalization), 1 Byzantine node isolated (D's vote for `c2` is
silently dropped).

### View 2 (leader B) — leader is slow

**Time t=0:**
- B is leader but slow (perhaps B's CPU is busy). No `notarize`
  broadcast yet.

**Time t=2Δ (timeout `t_l` fires):**
- A, C, D have not received B's `notarize`. Timer fires.
- A broadcasts `nullify(2)`.
- C broadcasts `nullify(2)`.
- D broadcasts `nullify(2)` (even though Byzantine — D can vote
  anything; let's say D votes nullify this time).

**Time t=3Δ:**
- A, C have 2 `nullify(2)` votes from each other + D's. That's 3
  (well, with the local vote, 4). Form `Nullification { view: 2 }`.
- Enter view 3.

Notice: a slow leader wastes a view but doesn't fork the chain. The
cost is one view's worth of time (3Δ for the timeout).

### View 3 (leader C) — Byzantine leader tries to fork

**Time t=0:**
- C is leader. Suppose C is honest.
- C proposes `c3`. Broadcasts `notarize(c3, 3)`.

**Time t=Δ:**
- A, B, D receive. A and B broadcast `notarize(c3, 3)`. D broadcasts
  `notarize(c4, 3)` (Byzantine fork attempt).

**Time t=2Δ:**
- A, B, C have 3 votes for `c3` (from honest nodes). Form notarization.
  D's vote for `c4` is ignored (doesn't reach quorum).
- A, B, C broadcast `finalize(c3, 3)`.

**Time t=3Δ:**
- Finalization forms. View 3 is finalized.

D's `c4` vote goes nowhere — it can never reach quorum of 3 because
only 1 node (D itself) supports it.

This is **safety under split-brain**: two competing payloads can't
both reach `2f+1` because that would require `f+1` honest nodes to
equivocate (impossible — each honest node votes once per view).

## A worked PBFT example with n=4, f=1

PBFT (Castro-Liskov 1999) is the historical baseline. Same n=4, f=1.
Three phases, each O(n²) messages.

### Setup

View 0. Leader is A. A's application produces request `r`.

### Pre-prepare phase

A assigns sequence number `n=1` to `r`. A broadcasts
`<<PRE-PREPARE, v=0, n=1, d=r>, σ_A>` to B, C, D.

B, C, D receive, verify A's signature, verify sequence number is
expected, verify request digest. Each accepts.

### Prepare phase

B broadcasts `<<PREPARE, v=0, n=1, d=r, i=B>, σ_B>` to A, C, D.
C broadcasts same with i=C.
D broadcasts same with i=D.

A receives B's, C's, D's prepares. So A has 3 prepares (its own counts
too? In PBFT, yes, the primary counts its own prepare).

After 2 hops, each honest node has 3 prepares for (v=0, n=1, d=r):
- A: B, C, D's (plus its own implicit).
- B: A, C, D's.
- C: A, B, D's.

This is the **prepared certificate**. Each node enters the *prepared*
state.

### Commit phase

A broadcasts `<<COMMIT, v=0, n=1, d=r, i=A>, σ_A>` to B, C, D.
B, C, D do the same.

After 2 more hops, each honest node has 3 commits. Each enters the
*committed-local* state, then executes the request, then sends a
**reply** to the client.

### Message count

- 1 pre-prepare (from A) = 1 message.
- 3 prepares (one per other node, plus A's implicit) = 4 messages.
  Actually: A doesn't broadcast a prepare; A goes straight from
  pre-prepare to receiving prepares. So 3 prepare broadcasts (B, C, D).
- 4 commit broadcasts (A, B, C, D).
- 3 reply broadcasts (to client; client only needs f+1 = 2).

Total per request: ~11 messages + 1 client reply = ~12 messages,
spread over 4 hops (pre-prepare + prepare + commit + reply).
With signatures, every message is signed and verified.

This is O(n²) in messages per request. The 3 phases are 3 sequential
broadcast rounds; you can't pipeline requests within a view easily.

Compare to Simplex: O(n) per view for notarize (each node sends 1
vote, threshold-sig aggregates), O(n) per view for finalize. Total
~2n messages per finalized block.

The performance gap: with n=100, PBFT sends ~10,000 messages per
block; Simplex sends ~200. (And Simplex can pipeline across views
because each view's certificate is independent.)

## A worked Tendermint example with n=4, f=1

Tendermint (Buchman 2016) is closer to Simplex but with explicit
locking. Same n=4, f=1.

### View structure

Each view has two phases: *propose* and *prevote*. After collecting
prevotes, the leader proposes a *precommit* (essentially "I'm locked
on this block for this round"). After collecting precommits, the
block is committed (or the view is skipped).

### Round 0, step 1 (propose)

A is leader. A broadcasts `PROPOSAL { round: 0, step: propose,
block: B1, valid_round: -1 }`.

### Round 0, step 2 (prevote)

B, C, D receive the proposal. They verify B1. Each broadcasts
`PREVOTE { round: 0, block_id: hash(B1) }`.

If the leader was slow, nodes prevote `nil` (empty block).

After 1 hop: A has 3 prevotes for B1. A broadcasts
`PRECOMMIT { round: 0, block_id: hash(B1) }`.
B, C, D also broadcast precommits if they see 2f+1 prevotes for the
same block.

### Round 0, step 3 (precommit)

After 2 hops, each honest node has 3 precommits for B1. Block B1 is
*committed*.

### Locking

Crucially: when a node sees 2f+1 precommits (a *polka*), it locks on
that block. The lock means: in future rounds, the node will only
prevote blocks with `valid_round >= the lock round`.

### Round change

If the leader is slow, after timeout T, nodes broadcast
`TIMEOUT { round: 0, height: H, valid_round: -1 }`. After 2f+1
timeouts, enter round 1 with new leader.

If a node has a lock, it broadcasts
`TIMEOUT { round: 0, valid_round: lock_round }`. This informs the
new leader "I have a lock on round X."

### Difference from Simplex

Tendermint uses **polka + polka+1** for locking (2f+1 prevotes +
another 2f+1 prevotes in the next round). Simplex uses separate
notarize/nullify messages and no explicit locking. Simplex's
approach is simpler but loses some of Tendermint's responsiveness in
adversarial scenarios.

In Tendermint, a node can be "stuck" on its lock for several rounds
if the network is unreliable. Simplex avoids this by using
nullifications (which can occur without committing to a block).

## The Simplex paper in detail

Das, Xiang, Ren (2023). "Simplex Consensus: A Simple and Efficient
BFT Consensus." Cryptology ePrint Archive 2023/463. The Commonware
implementation is based on this paper with documented deviations
(`consensus/src/simplex/mod.rs:102-119`).

### The algorithm, in pseudocode

```
state: { view, last_finalized }
On enter view v:
  leader(v) = round_robin(v)   # or VRF
  if I'm leader(v):
    propose c = my_payload()
    broadcast notarize(c, v)
  start timer t_l = 2Δ, t_a = 3Δ

On receive notarize(c, v) from p:
  if verify(p, c, v) and v is current view:
    broadcast notarize(c, v)

On receive 2f+1 notarize(c, v):
  cancel timer t_a
  notarization = cert(c, v)
  if certify(notarization):  # application hook
    broadcast finalize(c, v)
  else:
    broadcast nullify(v)

On receive 2f+1 finalize(c, v):
  finalization = cert(c, v)
  state.last_finalized = (c, v)
  enter view v+1

On timer t_l:
  broadcast nullify(v)

On receive 2f+1 nullify(v):
  nullification = cert(v)
  enter view v+1
```

### Why 2 hops for block, 3 for finality

- **2 hops for notarize**: leader proposes (hop 1), others vote
  (hop 2). After hop 2, all honest nodes have 2f+1 votes (if everyone
  voted the same way, which happens when the leader is honest and
  the network is synchronous).
- **3 hops for finalize**: after notarize, each honest node
  broadcasts `finalize`. After this hop, all honest nodes have 2f+1
  finalizes. The block is *irreversible*.

Why not 1 hop? Because 1 hop means only the leader has signed — not
a quorum. Why not 2 hops for finalize? Because finalize requires
every node to be aware of the notarization; that takes at least 1
hop after the notarization forms.

### Tail-forking resistance

The "forced inclusion" property from the main text:

> A notarized payload in view `v` must appear in the canonical chain
> if no nullification certificate exists for `v`.

Proof: the leader of view `v+1` must include as parent either (a) the
notarization from view `v` or (b) a later notarization. If the leader
skips view `v`, they need a nullification certificate for `v` to prove
`v` is "skipped." A nullification requires 2f+1 `nullify` votes.
Honest nodes only nullify on timeout or certify failure. If view `v`
completed cleanly, no honest node nullifies, so no nullification
certificate exists, so the leader of `v+1` MUST include the view-v
notarization.

This makes Simplex's safety argument stronger than naive "longest
chain" consensus. A malicious leader cannot reorg out a notarized
block.

## Why Simplex is fast — the role of BLS threshold signatures

The PBFT-to-Simplex speedup comes from BLS threshold signatures.

### How BLS threshold works

A BLS threshold scheme has:
- A single group public key `pk`.
- Each of `n` signers holds a private key share `sk_i`.
- Any `t` signers can produce a valid signature on a message `m`.
- The signature is the *same size* as a single BLS signature, regardless
  of how many signers participated.
- Verifying checks that *at least* `t` signers signed.

In Simplex, `t = 2f + 1`. So 3 signers (out of 4) can produce a
threshold signature, and a verifier checks the signature against the
group public key.

### Why this is O(n) per view

In PBFT (no aggregation): a notarization is `n` individual signatures
or `O(n)` proof size. The collector must gather `n` votes, verify
each, and assemble.

In Simplex (with BLS threshold): a notarization is *one* threshold
signature. The collector gathers 2f+1 votes, asks them to produce a
threshold signature (which they can do with their private key
shares), and verifies the result against the group public key. The
proof size is constant (32-48 bytes), regardless of `n`.

The per-view message complexity:
- PBFT: leader broadcasts to `n-1` others (1 msg), each broadcasts
  to `n-1` others (n-1 msgs), so `n-1 + n*(n-1) = n² - 1` msgs.
- Simplex: leader broadcasts (1 msg), each broadcasts vote (n-1 msgs),
  so `n` msgs. (Plus finalize round: another `n` msgs.)

So per view, Simplex is O(n) vs PBFT's O(n²). With n=100, that's
100 messages vs 10,000.

### The trade-off

BLS threshold signatures require a **Distributed Key Generation
(DKG)** ceremony at the start of each epoch. Each validator holds a
key share; the group public key is known to all. Generating the key
shares securely is itself a multi-round protocol (see
Gennaro-Jarecki 2003, Kate-Koblitz-Goldberg 2010, and the Simple-2PC
DKG of Das-Xiang-Ren 2022).

Commonware's `cryptography::bls12381::threshold` (chapter 04) wraps
this. The DKG runs once per epoch; the resulting key shares are used
for all consensus messages until the next epoch.

If DKG is too expensive, Commonware's `bls12381_multisig` scheme uses
BLS multisignatures (no DKG, but signature size grows with signer
count). The choice is yours: threshold = O(1) proof size, multisig =
no DKG but O(n) proof size.

### Leader rotation cost

Simplex rotates leaders per view. With round-robin, leader selection
is `O(1)`. With VRF-based random selection, each node evaluates a
VRF on the view number and epoch randomness to determine the leader.
VRF evaluation is `O(1)` per node (constant-time pairing).

The cost is *coordination of randomness* across the epoch. VRF
inputs include the epoch's random beacon, which itself requires
aggregation (Dfinity-style or threshold-DKG-style). Commonware uses
the BLS threshold scheme for this aggregation: at epoch boundaries,
nodes contribute randomness via threshold signatures.

## Exercises

**Exercise 1.1 — Recite the `3f+1` formula.** Without looking, write
the formula and a 3-sentence proof. Hint: "two quorums must overlap
in at least one loyal node."

**Exercise 1.2 — Trace a 4-validator Simplex view with the Byzantine
node as leader.** Show every message. Identify which messages the
Byzantine node could forge and how the honest nodes detect it.

**Exercise 1.3 — Count messages in PBFT and Simplex.** For n=10, f=3,
count the messages exchanged for one block in PBFT (full path) and
in Simplex (notarize + finalize). Compare. Which scales better?

**Exercise 1.4 — Read the Lamport 1982 paper.** Get "The Byzantine
Generals Problem" from a library. Read section 4 ("The OM(m)
Algorithm"). Implement OM(1) in pseudocode with 4 generals, one of
whom is a traitor. Show the messages for one round.

**Exercise 1.5 — Reproduce the FLP argument.** The proof uses
"bivalent configurations." Define a bivalent configuration in your
own words. Show why one must exist in the initial state if not all
processes start with the same value.

**Exercise 1.6 — Compare Tendermint and Simplex.** Both are BFT, both
use rounds. List 3 specific differences. Hint: locking vs no locking,
nil votes vs separate nullify messages, polka+1 vs simple quorum.

## Appendix A — The deep math: FLP impossibility

We claimed Simplex works. But there's a famous result that says **no
deterministic protocol can solve consensus in an asynchronous network with
even one crash fault**. That's the FLP impossibility (Fischer, Lynch,
Paterson, 1985).

The intuition: in an asynchronous network, a node can't tell the
difference between "the leader is dead" and "the leader's message is just
slow." If you decide to proceed without the leader's message, you might
be wrong — and now two nodes have made different decisions. If you wait,
you might wait forever.

Simplex **sidesteps** FLP by assuming **partial synchrony**: messages
are delivered eventually, but the bound is unknown. Simplex adds timers
(`t_l = 2Δ`, `t_a = 3Δ`). When the network is being well-behaved,
timers expire after messages have already arrived, and consensus
proceeds fast. When the network is being adversarial, timers expire
before messages arrive, and consensus proceeds slowly (with nullifications
that may be wasted work, but never wrong work).

The trade-off: Simplex is **safe under all conditions**, **live when the
network is partially synchronous** (i.e., eventually messages get
through within the time bound). This is the standard BFT trade-off; FLP
just says you can't have safety AND liveness unconditionally.

## Appendix B — Round / View / Epoch / Height: the four counters

Open `consensus/src/types.rs` and the Simplex config. There are **four
related but distinct counters** in any BFT system. Conflating them is a
common source of confusion.

```rust
pub type Height = u64;     // sequential position in the chain (block 1, 2, 3, ...)
pub type View = u64;       // consensus attempt counter (try 1, try 2, try 3, ...)
pub type Round = (Epoch, View);  // view within a specific epoch
pub struct Epoch(u64);    // the validator set in effect
```

**Height** — the actual block height. Block 1 has height 1. Block 2 has
height 2. Monotonically increasing over the chain's lifetime.

**View** — the consensus attempt. View 5 means "this is our 5th attempt to
agree on the next block." Different views can produce the same height
(if a view is skipped/empty) or different heights (if a view finalizes).

**Epoch** — the validator set in effect. Epoch 5 has a different committee
than epoch 4. Epochs change when the validator set changes.

**Round** — `(epoch, view)` combined. The "view within an epoch." Most
BFT systems use round to mean "this specific attempt under this specific
validator set."

In Simplex:
- Each view attempts to produce **one block** (a notarization).
- Successful notarizations can be **certified** (chapter 07: erasure coding
  check).
- Certified notarizations advance to the next view.
- Unsuccessful views (timeouts, nullifications) skip to the next view
  without finalizing.
- Finalized blocks advance **height** by 1.

Simplex's `Viewable` trait (`consensus/src/lib.rs:39-42`):

```rust
pub trait Viewable {
    fn view(&self) -> View;
}
```

Every consensus message has a view. The Voter rejects messages with
views outside the active range (chapter 11, the `interesting()` function).

## Appendix C — Certificate construction, in detail

From chapter 01: a certificate is "a quorum of 2f+1 votes with
matching content." Let's make this precise.

For Simplex with ed25519 scheme:

```rust
pub struct Notarization<D: Digest, S: Scheme> {
    pub proposal: Proposal<D>,    // { payload, signature over (view, parent) }
    pub certificate: Certificate<S>,  // aggregated signatures over (notarize(c, v))
}
```

The `Certificate<S>` is a scheme-specific type:
- For **ed25519** / **secp256r1**: a `Vec<Attestation>` with one entry per
  signer.
- For **bls12381_multisig**: one multisig + the signer indices.
- For **bls12381_threshold**: a single threshold signature + the static
  group public key.

The voter only emits a notarization if **all** of these hold:

1. ≥ `2f+1` `notarize(c, v)` messages received from **unique** signers.
2. All signatures verify against the validator set for the current epoch.
3. All signers' votes are for **the same** `c` (otherwise they don't form
   a quorum on anything).
4. The Voter has verified `c` itself (or has been told to skip verification
   via the `verify=false` path, which exists for testing).

If any check fails, the votes are dropped silently (logged but not
broadcast). No `nullify` is emitted for a notarization failure — that's
a local concern, not a view-timeout.

The certificate construction process:

```rust
// Hypothetical voter code
let mut votes_by_payload: HashMap<Digest, Vec<Attestation<S>>> = HashMap::new();
for msg in notarizes_for_view_v {
    if S::verify(&msg.signer_pk, &namespace, &encode_notarize(c, v), &msg.signature) {
        votes_by_payload.entry(msg.payload).or_default().push(msg.attestation);
    }
}
for (payload, attestations) in votes_by_payload {
    if attestations.len() >= quorum {
        let cert = S::assemble_certificate(attestations)?;
        emit Notarization { proposal: Proposal { payload, ... }, certificate: cert };
    }
}
```

Notice: votes for **different** payloads at the same view are collected
separately. Only the payload that reaches `2f+1` produces a notarization.
Other payloads' votes are discarded.

This is what makes Simplex **safe under split-brain**: two competing
payloads can't both reach `2f+1` because that would require `f+1` honest
nodes to vote for both (impossible — each honest node votes once per
view).

## Appendix D — The Simplex deviation from Simplex Consensus paper

The reference paper is `eprint.iacr.org/2023/463` (Simplex Consensus by
Sourav Das, Zhuolun Xiang, Ling Ren). The Commonware implementation
**deviates in specific ways** (see `mod.rs:102-119`):

1. **Fetch missing certificates on demand.** The paper assumes proposals
   contain all historical notarizations/nullifications. Commonware's
   implementation asks the Resolver (chapter 08) for missing ones.

2. **Distinct messages for `notarize` and `nullify`.** The paper uses
   "votes for a block" and "votes for a dummy block." Commonware uses
   separate message types, which is cleaner for network analysis.

3. **Leader timeout** — early view transition if the leader is slow. The
   paper waits for the activity timeout. Commonware adds a faster
   `t_l = 2Δ` for unresponsive leaders specifically.

4. **Skip timeout** — if the leader hasn't been active in `r` views,
   trigger an immediate transition.

5. **Message rebroadcast** — periodically re-send votes to overcome
   dropped messages without requiring a heavyweight reliable broadcast.

6. **Local proposal failure = immediate timeout** — if you can't propose
   (e.g., not synced), nullify immediately rather than waiting.

7. **Local verification failure = immediate timeout** — symmetric.

8. **Certification gating** — the `certify` hook (chapter 01 main text
   and chapter 07) is Commonware's addition, not in the paper. It
   separates hash agreement from data verification.

These deviations are Commonware's engineering judgment: tighter timeouts,
fewer assumptions about peer behavior, decoupled verification.

## Appendix E — Forced inclusion, in more depth

The "tail-forking resistance" property from chapter 01:

> A notarized payload in view `v` must appear in the canonical chain
> if no nullification certificate exists for `v`.

Proof sketch:

1. To propose in view `v+1`, the leader must have a **certified** parent
   in some view `v_p < v+1` and must possess **nullification certificates**
   for every view from `v_p+1` to `v`.
2. A nullification certificate requires `2f+1` `nullify` votes.
3. Honest nodes only `nullify` on timeout (`t_l` or `t_a`) or on certify
   failure.
4. If view `v` completed without timeout AND certification succeeded,
   no honest node broadcast `nullify(v)`. At most `f` Byzantine nodes
   could have voted `nullify`, which is < `2f+1`. So **no nullification
   certificate for v exists**.
5. Without a nullification certificate for `v`, the leader of view `v+1`
   must reference the notarized block at `v` as an ancestor.
6. By induction, all subsequent proposals include it.

The crux: a malicious leader **cannot skip over a notarized block** to
double-spend, censor, or rewrite history. The notarization is a
permanent commitment.

This is what makes Simplex safer than naive "longest chain" consensus,
where a sufficiently powerful attacker CAN reorg out recent blocks.

## Appendix F — Optimistic finality and speculative execution

> Once a notarization certificate is observed for view `v` (without any
> timeout having fired), the notarized payload can be treated as
> speculatively final.

The conditions:
- `2f+1` notarizes received.
- No timeout has fired (so no `nullify` votes).
- No certification failures.

Why is this safe? Because the only way the block at view `v` could
**not** be in the canonical chain is:
- `f+1` or more honest nodes timed out, OR
- Certification failed for all of them.

In the common case (no faults, no timeouts), neither happens. So you can
act on the block at view `v` after just 2 network hops, not 3.

This enables **speculative execution**: start executing the block
immediately after notarization. If finalization fails (rare), roll back.
Most of the time, the speculation was correct, and you saved one network
hop of latency.

(Alto uses this — see chapter 19 / the blog `buffered-signatures.html`.)

## Appendix G — Why Simplex's nullifications don't cause forks

In PBFT and similar protocols, a nullification at view `v` might be
"vetoed" by some nodes that already saw a notarization at `v`. This can
cause temporary forks that resolve later. Simplex's design avoids this:

- `2f+1` notarizes form a notarization. If you see it, you're locked in.
- `2f+1` nullifies form a nullification. If you see it, you move on.
- A node can see BOTH (if it sees votes from both sides after the
  decision was already made). It must pick the notarization if
  available, else the nullification.

The `interesting()` function (`mod.rs:362-381`) handles this:

```rust
pub(crate) fn interesting(
    activity_timeout: ViewDelta,
    last_finalized: View,
    current: View,
    pending: View,
    allow_future: bool,
) -> bool {
    if pending.is_zero() { return false; }                // genesis, skip
    if pending < min_active(activity_timeout, last_finalized) { return false; }
    if !allow_future && pending > current.next() { return false; }
    true
}
```

A vote for a view that's too far behind `last_finalized` is dropped. A
vote for a view too far ahead of `current` is dropped. So old votes
don't cause retroactive forks.

## Where to look in the code (expanded)

- `consensus/src/simplex/mod.rs:30-119` — the full protocol spec.
- `consensus/src/simplex/mod.rs:124-168` — the three "free" properties.
- `consensus/src/simplex/mod.rs:180-207` — the four-actor architecture.
- `consensus/src/types.rs` — `Height`, `View`, `Epoch`, `Round`.
- `consensus/src/lib.rs:25-79` — the trait surface.
- `consensus/src/simplex/scheme/` — the four supported signature schemes.
- `consensus/src/simplex/elector.rs` — `RoundRobin` and `Random` (VRF)
  leader election.

## If you only remember three things

1. **`n ≥ 3f + 1`**, quorum is `2f + 1`. That's the math; everything else is bookkeeping.
2. **Simplex is 3 message types, 2 timers, 2 quorums.** 2 hops for a block, 3 hops for finality.
3. **The `certify` hook is the seam** between consensus and your application. Consensus doesn't know what's in `c`. That's the whole point.

→ Next: **Chapter 02 — Runtime**. The abstract runtime is what makes all this
testable deterministically. Without it, you can't actually trust the 93% coverage
number.