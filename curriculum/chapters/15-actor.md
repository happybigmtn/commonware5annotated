# Chapter 15 — Actor: mailboxes, supervision, and bounded concurrency

> How primitives talk to each other safely under load.

## The problem

Commonware has lots of moving parts. Simplex has Voter, Batcher, Resolver.
P2P has connection handlers. Broadcast has the cache engine. They all need
to send messages to each other.

Naive approach: shared `Arc<Mutex<VecDeque<Message>>>` and a condvar.
**Problems:**
- Unbounded memory growth under load.
- Deadlocks if the lock-holder calls back into the same component.
- Lost messages if the queue fills up and you drop.
- No way to know "is the receiver alive?"

Commonware's answer: **the actor model**, with bounded mailboxes and
explicit overflow handling.

## The mailbox — bounded queue + unbounded overflow

Open `actor/src/mailbox.rs:1-41`:

> The mailbox is split into two queues: a bounded `ready` queue that
> producers push to and the receiver pops from, and an unbounded `overflow`
> queue that holds messages displaced when ready is full.

```
                          senders
                             |
         +-------------------+--------------------+
         | overflow inactive                      | overflow active
         | and ready has room                     | or ready full
         v                                        v
     +----------+    refill front-to-back     +----------+
     |  ready   |<----------------------------| overflow |
     +----------+    after each ready pop     +----------+
         |
         | pop first
         v
      receiver
```

- **Ready** — fast path, bounded. Pop is fast, push is fast (when there's
  room).
- **Overflow** — slow path, unbounded. Holds messages that didn't fit.
  Receiver eagerly refills `ready` from `overflow` after each pop.
- **Receiver** — single consumer. Pops from `ready`, processes, repeats.

Why two queues? Because **bounded** memory is what you want, but bursts
happen. Without overflow, a burst would either fill ready and reject new
messages (data loss) or grow ready unbounded (OOM). With overflow, bursts
are absorbed; under sustained overload, you have a problem you can see
(overflow growing) instead of one you can't.

## The two policies

`mailbox.rs:91-140`. Two trait surfaces:

### `Policy` — reliable (always handle overflow)

```rust
pub trait Policy: Sized {
    type Overflow: Overflow<Self>;

    /// Reliably handle a message when it can't enter the ready queue.
    fn handle(overflow: &mut Self::Overflow, message: Self);
}
```

Use this when losing messages would be a correctness violation. For
example: consensus votes. Dropping a `notarize` message means missing
votes that could have formed a certificate.

The `handle` method:
- Can **retain** the message in overflow storage.
- Can **coalesce** with existing retained messages (e.g., two votes for the
  same view → keep only the latest).
- Can **drop** the message if it's "already satisfied, superseded, or no
  longer needed" (e.g., a request whose response channel is already
  closed).

Returns nothing because it's *reliable* — the message was either retained
or intentionally absorbed. Never rejected.

### `UnreliablePolicy` — may reject overflow

```rust
pub trait UnreliablePolicy: Sized {
    type Overflow: Overflow<Self>;

    /// Unreliably handle a message when it can't enter the ready queue.
    /// Returns true if the policy considered the message's effects.
    fn handle(overflow: &mut Self::Overflow, message: Self) -> bool;
}
```

Use this when overflow handling is best-effort. For example: gossip
messages. Dropping a gossip message is bad but not catastrophic — it'll
be re-gossiped by other peers.

Returns `bool`: `true` = handled (retained/coalesced/dropped intentionally),
`false` = rejected under backpressure.

## The `Overflow` trait

```rust
pub trait Overflow<T>: Default {
    fn is_empty(&self) -> bool;
    fn drain<F>(&mut self, push: F)
    where F: FnMut(T) -> Option<T>;
}
```

Storage for overflowed messages. Default impl is `VecDeque<T>`. You can
implement it to do smart things:

- **Latest-wins**: only keep the last message per key.
- **Coalescing**: merge messages of the same type.
- **Bounded**: keep at most N messages (then drop oldest).

The `drain` method is called by the receiver when ready has space. It
asks the storage to push messages back into ready. The `push` closure
returns `Some(msg)` if it refused (e.g., ready filled up again) — the
storage must put that message back.

## A warning about re-entry

`mailbox.rs:106-110`:

> Do not enqueue into the same mailbox from this method or from destructors
> triggered by editing `overflow`. This method runs while the mailbox
> holds its overflow lock, so same mailbox re-entry can deadlock.

This is the deadlock warning. If your `handle` method tries to call
`mailbox.send(...)` on the same mailbox, you deadlock instantly. The lock
is held; your call blocks; the receiver can't release it.

If you need to send to another component from within `handle`, send to
*that* component's mailbox, not your own.

## The feedback enum

From chapter 05 and `actor/src/lib.rs:13-39`:

```rust
pub enum Feedback {
    Ok,         // accepted into ready
    Backoff,    // overflowed (reliably handled)
    Closed,     // receiver is gone
}

pub enum Unreliable<T> {
    Outcome(T),     // result of enqueue (Ok, Backoff, Closed)
    Rejected,       // not handled at all
}
```

Callers check the feedback to know what happened. If `Closed`, the receiver
task died — you can stop sending. If `Backoff`, the system is under load —
you might want to slow down. If `Ok`, all good.

## Supervision trees (chapter 02 again, in detail)

`runtime/src/lib.rs:177-259`. The `Supervisor` trait:

```rust
pub trait Supervisor: Send + Sync + 'static {
    fn name(&self) -> Name;
    fn child(&self, label: &'static str) -> Self;
    fn with_attribute(self, key: &'static str, value: impl Display) -> Self;
}
```

Every task has a parent. When the parent finishes or aborts, all children
are aborted. This is the **mandatory supervision** that prevents orphan
tasks.

Look at the diagram in chapter 02's lib.rs (which is also in the actor
docs):

```
ctx_a
  |
  +-- child("worker") ---> ctx_c
  |                  |
  |                  +-- spawn() ---> Task C (ctx_d)
  |
  +-- spawn() ---> Task A (ctx_b)
                             |
                             +-- spawn() ---> Task B (ctx_e)

Task A finishes or aborts --> Task B and Task C are aborted
```

So if the Simplex Voter crashes, the Batcher's task gets cleaned up too.
No leaks.

## The mailbox + supervision together

The mailbox handles **load**. Supervision handles **crashes**. They work
together:

1. Task A spawns task B as a child (supervision).
2. Task B has a mailbox (bounded ready + overflow).
3. Senders send to B's mailbox. Under load, messages overflow.
4. B's `handle` policy decides what to do with overflow (retain, coalesce,
   drop).
5. If B panics, supervisor A gets notified; A can decide to restart B or
   shut down.

For Commonware's BFT context: a panicked consensus task is a serious
problem. Supervision triggers shutdown — better than continuing with a
half-dead consensus engine.

## Putting it together — Simplex's mailbox

Look at the Simplex Voter config:

```rust
let cfg = simplex::Config {
    ...
    mailbox_size: NZUsize!(1024),   // ready queue holds 1024 messages
    ...
};
```

This sets `ready` to 1024 messages. Overflow is unbounded, but in practice
should never grow unless the system is fundamentally overloaded.

If overflow does grow, the `metrics` (chapter 02) show it. You'd see
`backoff_total` climbing, indicating "we're under load, something's stuck."

## The `SupervisedMailbox` pattern

Some primitives use a **supervised mailbox**: the mailbox's receiver task
is itself a child of some parent. If the mailbox task panics, the parent
knows.

Simplex's voter does this. The voter is spawned as a child of the runtime
context. If the voter panics, the runtime sees it via supervision and
initiates shutdown.

## The "panic aborts descendants" property

From the supervisor docs (chapter 02):

> All tasks are supervised. When a parent task finishes or is aborted,
> all its descendants are aborted.

This is the **structural** deadlock prevention. If task A acquires lock
L1 and dies, task B (waiting for L1) would otherwise deadlock waiting
forever. With supervision, B is aborted when A dies — B's `select!` on
`A.aborted()` resolves, B returns an error, the system keeps moving.

## Where to look in the code

- `actor/src/mailbox.rs:1-41` — the mailbox architecture.
- `actor/src/mailbox.rs:91-140` — the two policy traits.
- `actor/src/lib.rs:13-39` — the `Feedback` and `Unreliable` types.
- `runtime/src/lib.rs:177-259` — the `Supervisor` trait.
- `runtime/src/lib.rs:262-356` — `Spawner` and the supervision model.
- `consensus/src/simplex/actors/voter/actor.rs` — how the Voter uses mailbox
  + supervision together.

## The actor model: Hewitt, Agha, and what it formalizes

The actor model is the foundational theory this chapter implements.
Two papers are load-bearing:

1. **Carl Hewitt, Peter Bishop, Richard Steiger (1973), "A Universal
   Modular ACTOR Formalism for Artificial Intelligence"** — the original
   construction. The three primitives an actor exposes are:
   - **Send**: deliver a message to another actor.
   - **Receive**: process a single message from its own mailbox.
   - **Create**: spawn a new actor (with its own identity, its own
     initial behavior, and its own fresh mailbox).

2. **Gul Agha (1986), "Actors: A Model of Concurrent Computation in
   Distributed Systems"** — the formalization. Agha's thesis turned
   the three primitives into a calculus with:
   - **Mailbox as identity**: each actor is identified by the address
     of its mailbox. The model is fundamentally about processing
     *messages*, not calling *functions*.
   - **No shared state**: actors communicate by sending messages.
     Anything mutable lives inside exactly one actor.
   - **No locks**: a single actor processes one message at a time,
     so there's nothing for two actors to fight over.
   - **Asynchronous send**: `actor ! msg` does not wait. (Some
     languages add synchronous request-reply as a derived primitive;
     Commonware's `Sender::send` is the asynchronous version.)

What the actor model **formalizes**:

- **Liveness**: under fair scheduling, every message sent to a live
  actor is eventually delivered.
- **No undefined nondeterminism**: any indeterminism is from
  interleaved message arrival order, not from language-level race
  conditions.
- **Composability**: you can reason about a system's properties
  by reasoning about its actors individually and the messages
  between them — without modeling the entire global state.

What the actor model **does not formalize**:

- **Mailbox capacity**. The original model has unbounded mailboxes;
  practical implementations must bound them.
- **Overflow policy**. The original model never drops messages;
  practical implementations often must.
- **Supervision/restart**. Agha added this later in life; Erlang's
  Joe Armstrong popularized it. Commonware's supervision tree is
  the Commonware form.
- **Bounded vs unbounded time**. "Eventually delivered" assumes fair
  scheduling; Commonware doesn't promise this.

Commonware's implementation is the **practical** actor model: bounded
mailbox + explicit overflow + supervision + deterministic scheduler
(chapter 02). When chapter 02's deterministic runtime replays a scenario,
it delivers messages in the same order to the same actors — actor-model
formal semantics hold as long as senders don't break scheduling.

## CSP vs actors

Hoare's **Communicating Sequential Processes** (CSP, 1978) is the
formal sibling of the actor model. The difference is foundational:

| | Actor model | CSP |
|---|---|---|
| Communication primitive | Mailbox (a per-actor FIFO queue) | Channel (a rendezvous or buffered point-to-point link) |
| Identity | Mailbox address | Process |
| Channels | Derived; multiple can point to one mailbox | First-class; explicitly typed |
| Send blocks? | No (asynchronous) | Yes (rendezvous) unless buffered |
| Receive blocks? | Yes (waits for next message) | Yes (waits for matching send) |
| Where it ships | Erlang/OTP, Akka, Commonware actor | Go (`chan`), Rust `tokio::sync::mpsc` |
| Best at | "Send and forget"; persistent servers | "Pipe this data through these stages"; pipeline processing |

Erlang-style actor: each process owns one inbox, senders hand it
messages, the process picks them up in order. Erlang lets you `link`
processes so a death in one propagates. This is Commonware's mailbox
plus supervision.

Go-style channel: a typed value-passing pipe between two (or more)
goroutines. Send blocks until the receiver is ready (or the buffer
has room). This is `tokio::sync::mpsc` and `tokio::sync::oneshot`.

The actor model fits Commonware's design because:

- **State lives in the actor** — the actor's identity is durable.
- **Multiple senders per actor** — many components (consensus voter,
  marshal, broadcast) all talk to the same actor.
- **Backpressure is per-actor** — one bounded queue, one overflow
  policy, one explicit decision per actor.

CSP would force each pair of sender/receiver to define its own
buffering. That's too much bookkeeping for a system with the
connectivity graph Commonware has.

## Tokio tasks vs actors

A Tokio task is a future running on the multi-threaded runtime. It
has:

- A `JoinHandle` you can await.
- No built-in mailbox.
- No built-in identity beyond `task::id()`.
- Cheap cancellation via `abort()`.

What's **added** by wrapping it in an actor:

| | Tokio task | Actor |
|---|---|---|
| Mailbox | None built-in | Bounded queue + overflow |
| Identity | `task::id()` (not addressable) | A typed `Sender` cloneable across modules |
| Spawn | `tokio::spawn(async { ... })` | `Engine::start(cfg, mailboxes)` |
| Shutdown | `task.abort()` | Multi-stop signals + sup tree |
| State ownership | Whatever you move into the future | Owned by the actor struct |
| Routing | Send `JoinHandle` around | Send `Sender::send(msg)` |

So a Tokio task is **the runtime primitive**. An actor is **the
programming model built on top of it** — a structured way to give
each task a state machine, an address, and a backpressure policy.

In practice every Commonware actor is implemented as one Tokio task
(or, for tests, one deterministic-runtime task — chapter 02) wrapping
the actor struct. The actor struct owns the **mailbox Sender** the
external world talks to. The actor's `run` future is the message
loop; the actor's fields are its private state. The two pieces —
mailbox-backed address plus per-actor state — are what distinguish an
actor from a plain future.

## Lock-free data structures — when Commonware uses them

The Commonware mailbox's hot path is a `Mutex<VecDeque<T>>`. That's
**not** lock-free. The design tradeoff:

- **Mutex path**: spin briefly; if contention, suspend the
  caller. Simplest possible code, slowest under heavy contention.
- **Lock-free path**: compare-and-swap-based, no suspending. Faster
  under contention but harder to reason about and limited in
  operations you can support.

Commonware uses **lock-free** in two places:

1. **Waker registration in `recv()`**: an atomic so the sender can
   wake the receiver without grabbing the ready-queue lock. Pattern
   from `crossbeam-channel` and `flume`.

2. **Counter metrics in the runtime** (chapter 02): `Arc<AtomicUsize>`
   for things like `mailbox_send_total` or `tasks_spawned_total`.
   CAS-add to bump them; no async.

Lock-free data structures in this chapter's vicinity that Commonware
**does not** use but might recognize:

### Michael-Scott queue

The classic bounded lock-free MPMC queue (1996). Compare-and-swap
linked list of "ticket" slots. Each sender atomically claims a
ticket, deposits, advances the head; each receiver atomically
retires, advances the tail. Commonware's mailbox is roughly a
bounded variant of this idea but with a `Mutex` for simplicity, since
the hot path throughput target is ~10–50 ns/call, far below the
Michael-Scott queue's cliff.

### Hazard pointers

Memory reclamation for lock-free structures. The Commonware mailbox
never frees the queue while any thread holds a reference into it;
its lifetime is bound to the `Arc<MailboxState>`. So hazard pointers
aren't needed.

### When Commonware uses `Mutex`

Everywhere else. The decision rule: if you can express your data
structure's invariants in terms of `lock; mutate; unlock`, use
`Mutex`. If the unlock leaves the structure in a state where another
thread could observe a partial invariant, *then* you need
lock-free. Commonware's mailboxes, journal buffers, QMDB
commit-and-prune paths — all lockable. Look at
`storage::journal::variable::Config { buffer_pool, write_buffer }`
in chapter 06 for an example.

## The full mailbox implementation, in detail

`actor/src/mailbox.rs`. The shape is:

```rust
pub struct Mailbox<T, M = Reliable> {
    state: Arc<MailboxState<T, M>>,
}

struct MailboxState<T, M> {
    ready: Mutex<VecDeque<T>>,       // bounded hot path
    overflow: Mutex<M::Overflow>,    // unbounded cold path
    capacity: NonZeroUsize,
    closed: AtomicBool,
    receiver_registered: AtomicBool,
    waker: Mutex<Option<Waker>>,
    // Metrics
    send_total: Arc<AtomicUsize>,
    send_backoff_total: Arc<AtomicUsize>,
    send_rejected_total: Arc<AtomicUsize>,
    recv_total: Arc<AtomicUsize>,
}
```

The `Sender` is a `Clone` of the `Arc<MailboxState>`. Send and recv
both go through one level of indirection. The `Receiver` is a single
owner (the actor task); clones aren't supported because that would
compete for messages.

### Send path

```
1. Acquire closed flag (acquire ordering).
2. If closed, return M::closed_feedback().
3. Lock ready.
4. If len() < capacity: push_back(msg); drop ready; wake receiver; return Ok.
5. drop ready.
6. Lock overflow.
7. Call M::handle(overflow, msg).
8. drop overflow.
9. Return M::overflow_feedback(handled).
```

### Recv path

```
1. Lock ready.
2. If pop_front returns Some(msg):
     a. drop ready.
     b. refill ready from overflow (lock overflow briefly; drain into ready; drop).
     c. return Some(msg).
3. drop ready.
4. check if closed; if so, return None.
5. register waker; await wakeup.
6. goto 1.
```

The `refill_ready_from_overflow` step is the key insight. By always
refilling after each pop, the actor keeps ready-as-full-as-possible
without holding the locks across the consumer's processing. If
overflow is empty, the refill is a no-op; if overflow is full, we
move one message into ready for every pop.

### Why this fast-path/slow-path split

The receiver is single-consumer. The hot path (ready has room, no
overflow check) takes **one Mutex acquisition and release**. The
slow path (overflow handling) adds a second Mutex acquisition. Under
load, the steady state is hot-path-only: producer pushes, consumer
pops, overflow stays empty.

## Backpressure: drop, block, coalesce, spill

When the actor's ready queue fills, four common policies.

| Policy | Overflow behavior | Best when |
|---|---|---|
| **Drop** | New messages discarded; sender's feedback says `Rejected` | Gossip, telemetry, anything idempotent |
| **Block** | Sender waits on a semaphore until ready has room | Real-time control loops, RPC handlers |
| **Coalesce** | New messages merged with existing retained ones | State updates (only the latest matters) |
| **Spill** (default in Commonware) | New messages buffered in overflow queue | Consensus messages (lose none) |

Commonware's `Reliable` mode is **spill** (default `Overflow<T> =
VecDeque<T>`); `Unreliable` mode is **drop** with explicit feedback.
The actor's `Policy::handle` is the place to implement
**coalesce**: hold a `HashMap<K, T>` keyed by some operation-id;
when a new message arrives with an existing key, merge; when the
queue refills into ready, drain the latest version per key.

```
+-------------------------------------+
|  overflow: HashMap<K, T>            |
|                                     |
|  { K1 -> msg_v3, K2 -> msg_v2 }     |
|                                     |
|  On refill: drain all entries into  |
|  ready; consume produces "latest    |
|  per K" messages.                   |
+-------------------------------------+
```

The collect chapter 10's `Originator::submit` is exactly this pattern
applied at a different granularity: coalescing requests for the same
key across many senders, then dispatching one request per key.

### Choosing the right policy

The rule of thumb from chapter 11's Simplex: votes **must** be
reliable (the protocol fails if you drop a notarization message);
gossip can be unreliable (the network re-gossips anyway); state
updates want coalescing (older ones are obsolete).

## Work stealing — and how actors fit

Tokio's work-stealing scheduler maintains N runqueues (one per
worker thread). When a worker runs out of local work, it steals a
runnable task from another worker. This is what gives Tokio its
"multi-threaded runtime" feel.

Actors fit into this nicely:

```
+------------------+     +------------------+
| Worker A          |     | Worker B          |
| +------------+   |     |  +------------+   |
| | Actor X    |   |     |  | Actor Y    |   |
| | mailbox    |   |     |  | mailbox    |   |
| +------------+   |     |  +------------+   |
+------------------+     +------------------+
        |                          |
        +--- work-stealing -------+
```

The actor's task is just another future. The actor's **mailbox** is
not a runqueue — it's a FIFO queue the actor will pop from when its
task is scheduled to run. So actors don't change Tokio's scheduler
behavior; they just structure what each task does.

What actor tasks add beyond plain `tokio::spawn(async { ... })`:

- **Email-like address**: senders hand `Sender`s around like URLs.
- **State locality**: each actor has its own state; no surprise
  contention between consumers.
- **Backpressure visibility**: `mailbox_send_backoff_total` makes
  "we're under load" queryable, not invisible.

## Supervision: Erlang/OTP's "let it crash"

Erlang/OTP's supervision trees are the canonical model. The
philosophy:

- **Don't try to write defensive code.** Trying to recover from every
  expected error produces tangled, untestable logic.
- **Instead, structure for restart.** Each process has a well-defined
  **initial state** and a way to reach it. If it crashes, the
  supervisor starts a new instance.
- **Restart strategies** (per the OTP docs):
  - **one-for-one**: only the failed child restarts.
  - **one-for-all**: if any child fails, all restart.
  - **rest-for-one**: the failed child and all children started
    after it restart.
- **Supervision intensity**: max N restarts in M seconds before the
  supervisor itself terminates (escalates).
- **Crash-only design**: every crash mode is a restart mode. Restart
  is the normal recovery path.

Commonware's `Supervisor` (chapter 02) implements this. The
differences from Erlang:

- **No automatic restart** in the BFT context: a Byzantine-faulting
  consensus task should *not* be transparently restarted, because
  the rest of the network already saw it stop voting. Restart is
  wired up but typically comes with a "stop the whole validator"
  decision.
- **Supervision is mandatory**: every task has a parent.
- **Mandatory abort-on-parent-failure**: a parent that crashes
  causes descendants to abort. This is what prevents the "task A
  holds lock, task A dies, task B waits forever" deadlock.

In Commonware: `ctx.spawn(...)` returns a `Handle<T>`; `ctx.child("...")`
creates a sub-supervisor. When a parent task aborts (panic, explicit
`.abort()`, or stop signal), all `child().spawn()` descendants
abort too. This is the analog of OTP's `trap_exit` + supervisor's
death propagation.

## The `actor!` macro

`commonware-macros::actor!` reduces boilerplate. A common pattern
across Simplex, Marshal, and Aggregation:

```rust
actor! {
    pub struct Voter {
        // fields of the actor
        config: Config,
        mailbox: Mailbox<Message>,
        scheme: Scheme,
        automaton: Automaton,
        reporter: Reporter,
        supervisor: Supervisor,
    }

    // The body of the actor loop, written against the mailbox
    // receiver and the actor's fields.
    async fn run(mut self) {
        loop {
            select! {
                msg = self.mailbox.receiver.recv() => {
                    self.handle(msg).await;
                }
                _ = self.stop_signal => break,
            }
        }
    }
}
```

The macro expansion:

1. Defines the `Voter` struct with all fields as named in the
   `actor!` body.
2. Defines a `Voter::start(cfg, mailboxes) -> (Mailbox<Message>,
   Handle)` that spawns the actor as a supervised task.
3. Defines a `Voter::handle` forwarding method.
4. Generates all the bookkeeping glue (Drop, Debug, etc.).

The handle may be `pub` or `pub(crate)`, depending on who needs to
await or abort the actor.

Why a macro rather than generics: the syntax is friendlier than
manually constructing a 200-line actor scaffolding, and the macro
produces a stable shape that all actors in the codebase share. Commonware
often has one actor per consensus role; this macro means each is
written in roughly 50 lines, not 200.

## Shutdown: `Stopper`, `Signal`, and the multi-stop dance

The `Stopper` from chapter 02 (look at `runtime/src/utils/signal.rs`)
manages the dance:

```rust
let stopper = Stopper::new();
let signal = stopper.stopped();           // 1. Create a stop-signal channel

let task_handle = ctx.spawn(|ctx| async move {
    select! {
        _ = work() => {}
        _ = signal => {
            // Begin draining; wait for outbound requests to complete
        }
    }
});

// Later:
stopper.stop(0, Some(Duration::from_secs(30))).await?;
```

The `Signal::stop(value, timeout)` initiates shutdown. The `Signal::stopped()`
calls return `Future`s that resolve when shutdown is initiated. Each
task holding a `Signal` reference has an "obligation" to complete
gracefully:

- Drain outbound work in flight.
- Persist state to disk.
- Notify counterparties (if there's a peer task on the other side).

The **multi-stop safety** property: `stop()` can be called from
multiple places (each task finishing, the runtime clock elapsing,
an outer shutdown supervisor). Only the first call counts; the rest
return `Result::Err(Error::Exited)` immediately. This is why you can
wire `Signal` into every task in a supervisor tree without worrying
about double-shutdown races.

`Signal::stopped()` (no argument) is the **passive** variant: it
returns a future that resolves when shutdown is initiated by
**any** caller. Use this in long-running tasks that should yield when
shutdown begins.

## Common patterns in production

### The "supervised tree" pattern

```rust
let voter_mailbox = ...;
let voter_handle = ctx.child("voter").spawn(move |ctx| async move {
    voter_loop(ctx, voter_mailbox).await
});

let batcher_handle = ctx.child("batcher").spawn(move |ctx| async move {
    batcher_loop(ctx, voter_mailbox.clone()).await
});
// voter_handle and batcher_handle are now siblings under the same supervisor
// if either aborts (or the parent does), the sibling is aborted too
```

### The "fan-in" pattern

Many producers, one consumer. The consumer's mailbox is cloned; each
producer holds a `Sender` to it. Backpressure is centralized: if the
consumer is slow, every producer sees `Backoff`.

```rust
let (collector_mailbox, mut collector_receiver) = mailbox::mailbox(1024);
for source in sources {
    let sender = collector_mailbox.clone();
    spawn_producer(source, sender);   // each producer gets a clone
}
// consumer handles messages from all sources
loop {
    let msg = collector_receiver.recv().await;
    ...
}
```

### The "fan-out" pattern

One producer, many consumers. Each consumer has its own mailbox; the
producer chooses which mailbox to send to (often by a routing key).

```rust
let (control_mailbox, mut control_recv) = mailbox::mailbox(1024);
let worker_handles: Vec<_> = (0..N).map(|i| {
    let sender = control_mailbox.clone();
    ctx.child(format!("worker-{i}")).spawn(move |ctx| async move {
        while let Some(msg) = ... { /* process */ }
    })
}).collect();
```

### The "graceful drain on stop" pattern

```rust
async fn consume(mut mailbox_receiver: Receiver, stop: Signal) {
    let drain_timeout = Duration::from_secs(5);
    loop {
        select! {
            msg = mailbox_receiver.recv() => {
                process(msg).await;
            }
            _ = stop.stopped() => {
                // Set the "draining" flag; refuse new sends; finish in-flight
                break;
            }
            _ = sleep(drain_timeout) => {
                // No more senders; drain
                break;
            }
        }
    }
}
```

## Rust pattern — `unsafe`, `Send`/`Sync`, and the toolbox

Three patterns in this chapter that recur throughout Commonware.

### `Send` and `Sync` in actor code

The rules:

- `T: Send` — safe to move T to another thread.
- `T: Sync` — safe to share `&T` across threads (i.e., `&T: Send`).
- `Mutex<T>` is `Send + Sync` iff `T: Send`.
- `Rc<T>` is **neither** `Send` nor `Sync`.

Commonware's actor types are **Send-bound** for fields, so the
entire actor struct is `Send`. The actor's `run` future is `Send`
because it captures `&mut self.field`-references through `Pin`.

Where things get tricky: when an actor holds `&mut field` across an
`.await`, that future **must not outlive the field it borrows**. The
borrow checker enforces this statically. If you see a `Send`-bound
error, the fix is usually to move the field into a temporary local
before the await — or to clone the field (Arc-cloneable).

### `parking_lot::Mutex` vs `std::sync::Mutex`

Two pragmatic choices:

```rust
// std::sync::Mutex
let g = mutex.lock().unwrap();   // returns Result (poisoning-aware)

// parking_lot::Mutex
let g = mutex.lock();            // returns LockGuard directly; faster
                                  // no poisoning; smaller code size
```

Commonware uses `parking_lot::Mutex` in hot paths (the `Mailbox`
implementation, the `journal` buffers) for the speed. The two trade-
offs: `parking_lot` panics on poisoning; `std::sync` returns `Err`.
Commonware's choice: don't poison — unwrap poisoning as a programming
bug, not a runtime recovery path.

For `RwLock`, `parking_lot` is also used. The reader-favoring policy
fits Commonware's read-heavy consensus read-patterns.

### `Arc<AtomicUsize>` for counters

```rust
struct ActorMetrics {
    pub send_total: Arc<AtomicUsize>,
    pub send_backoff: Arc<AtomicUsize>,
    pub recv_total: Arc<AtomicUsize>,
}

let metrics = ActorMetrics::new();
let counter_clone = metrics.send_total.clone();

thread_A:
  counter_clone.fetch_add(1, Ordering::Relaxed);

thread_B:
  counter_clone.fetch_add(1, Ordering::Relaxed);
```

No locks, no mutex contention. `Ordering::Relaxed` is sufficient for
counters (you don't care about ordering, just the count). For memory
publication (so the receiver sees the message before the count),
`Ordering::Release` on the counter and `Ordering::Acquire` on the
reader.

### `unsafe` in actor code

Commonware has a strict rule: every `unsafe` block has a `// SAFETY:
...` comment explaining why the operation is sound. The few uses
you'll see:

- `Pin::get_unchecked_mut` in actor fields after `pin-project`.
- `AtomicPtr` access in the runtime's task-state manipulation.
- `static mut` in the FFI bridge to libsodium (chapter 04).

Reviewer focus (per `AGENTS.md`): anything that accepts **untrusted
input** and uses `unsafe` is suspect. The state-of-the-art in BFT-
critical code is "minimum `unsafe`, maximum review."

## Where to look in the code (expanded preview)

The appendices that follow go deeper on each topic above. When you
see a `file:line` reference in main body, the appendix walks it
line by line — but the surrounding structure and motivation live
here.

## The actor model theory in full — Hewitt, Agha, and π-calculus

The actor model this chapter implements goes deeper than "mailbox +
supervision." It is a foundational model of concurrent computation
that pre-dates most of what we now call "async runtimes." Three
papers define the lineage; understanding them turns Commonware's
mailbox from "a queue" into "an interpretation of a 50-year-old
model of computation."

### Hewitt 1973 — the construction

Carl Hewitt, Peter Bishop, and Richard Steiger published "A
Universal Modular ACTOR Formalism for Artificial Intelligence" in
the proceedings of IJCAI '73. The paper's contribution was not a
library or a runtime; it was a **formalism**. Hewitt wanted a way
to talk about AI systems built out of independently behaving
sub-systems that exchange information, without the formalism itself
imposing a particular control structure.

The key innovation was to model each such sub-system as an
**actor**. An actor has:

- A **behavior** — what it does when next triggered.
- A **mailbox** — a queue of incoming messages waiting to trigger it.
- A **set of acquaintances** — other actors it knows about (by
  address), and can send messages to.
- (In some formulations) a **replacement behavior**, which is what
  the actor becomes after each step.

An "actor step" is a single, atomic transition: select one message
from the mailbox, compute a new behavior (which may involve
constructing new actors and sending messages to acquaintances),
and replace the old behavior with the new one.

This decomposes a concurrent system into actors — independent
agents — and the messages between them. The fundamental primitives
are:

1. **Send** — `actor_ref ! message` (or some syntax sugar). This
   enqueues `message` onto the target actor's mailbox. It never
   blocks the sender. Return semantics are language-defined
   (Erlang returns the message reference; Akka returns a Future;
   Commonware returns `Feedback`).

2. **Receive** — implicit in the next step: an actor dequeues one
   message at a time and chooses what to do with it. The
   *order* of dequeue is FIFO in the standard model, though
   Agha's later work allows priority mailboxes.

3. **Create** — an actor can manufacture a new actor: assign it an
   initial behavior, an empty mailbox, and report its address
   back to the parent. The new actor is independently
   addressable.

Notice what's *not* in those three: read/write, lock/unlock, fork/
join, receive-with-timeout, request/reply, supervisor. None of
those primitives are part of the model. They're all **derived**.
Supervision came later (Armstrong's 2003 dissertation formalized
it; Agha and others had been working on error recovery in the
actor setting since the late 1980s). Request-reply is sugar over
send + receive (and a reply-to address). Fork-join is sugar over
create + send + wait-for-all-replies.

### Agha 1986 — the formalization

Gul Agha's PhD thesis ("Actors: A Model of Concurrent Computation
in Distributed Systems," MIT 1986) did for actors what Hoare did
for CSP: turn the intuition into a calculus. The thesis introduces:

- **Actor mailboxes as the unit of identity.** Two actors are "the
  same" iff they have the same mailbox address. There is no
  notion of "the actor is over there" separate from "messages go
  to this address."

- **Single-step semantics.** An actor's behavior change is atomic.
  No other actor sees "half a step." This is what gives the model
  its composability: you can reason about each actor's behavior
  locally, and the composition's properties follow.

- **Reception ordering.** A "fair" scheduler guarantees that each
  actor's mailbox is eventually serviced. Agha showed that under
  fair scheduling, every message sent to a live actor is
  eventually delivered. ("Liveness," in the formal sense.)

- **No shared state.** Mutable state lives inside exactly one
  actor. Two actors that want to coordinate must do so by
  sending messages. This rules out an entire class of bugs
  (data races, lost updates, torn reads) by construction.

What Agha **didn't** formalize:

- **Mailbox capacity**. The calculus assumes unbounded mailboxes.
  Commonware's `Mailbox<T, M>` is bounded (see `mailbox.rs:1-41`)
  with an overflow storage. A bounded mailbox violates Agha's
  liveness guarantee by design — under sustained overload,
  messages will queue or be rejected.

- **Message dropping.** Agha's calculus never drops. Commonware's
  `UnreliablePolicy::handle` returns `false` to reject a message
  under backpressure. This is a deliberate break from the
  calculus for **practical reasons**: in a real network, drops
  are normal, and we'd rather shed load than block forever.

- **Supervision/restart.** Agha's later work ("Composable
  Finite-State Machines," 1996, with Mason, Smith, Talcott)
  introduces fault models but doesn't pin down a "let it crash"
  philosophy. That comes from Erlang/OTP.

### The three primitives, again — and what they imply

The three primitives have implications that drive Commonware's
whole design:

**Send is asynchronous.** The sender does not wait for the
receiver. This forces the receiver to be the unit of concurrency
control: an actor processes one message at a time, so its state
is automatically consistent. The sender never knows when the
message is processed (without a separate reply-to).

This is why Commonware's `Sender<T>` clones cheaply and never
awaits. You hand out `Sender` clones the way you'd hand out a
phone number — the recipient (the actor's mailbox) is opaque to
the caller. The caller can't deadlock the callee.

**Receive is single-message.** One actor, one message at a
time. This is what eliminates the lock. There's nothing for two
senders to fight over because the receiver's "ownership" of its
state is total during processing.

Contrast with multi-threaded code: a method called on a struct
that is shared across threads needs synchronization because two
threads can interleave. The actor model sidesteps that by making
"interleaving" mean "inter-message," which is coarse-grained and
serial.

**Create is identity-forming.** When you `create`, you're not
"spawning a worker" — you're manufacturing a new entity with a
fresh address. The new entity might outlive you, might be referred
to by other actors, has its own lifecycle. This is a more
fundamental claim than "I'm forking a thread."

In Commonware, `ctx.child("name").spawn(...)` does this: it
forges a new address for the new task, registers it with the
supervisor tree, and lets the application hold that address
(via the returned `JoinHandle`, which doubles as an address).

### Relationship to π-calculus

The actor model is closely related to **Robin Milner's
π-calculus** (1973, concurrent with Hewitt's work). The
π-calculus is also about independent processes that send
names-and-values along named channels; the principal
difference is:

- **π-calculus** models channels as first-class values that can
  be sent in messages. Channels can be passed around;
  `process1 < chan > = new_channel()` can hand the only handle to
  `process2`, after which `process1` can no longer send through
  it.
- **Actor model** models mailboxes as the unit. Actors can't be
  sent (their address can). Mailboxes are not values that get
  handed around; you give acquaintances to other actors by
  telling them your address.

The translation is straightforward: an actor in the actor model
corresponds to a π-calculus process that owns one "private"
channel, plus all the rest of the messaging infrastructure.
Conversely, a π-calculus channel corresponds to an actor whose
mailbox accepts messages from one logical sender.

Commonware uses an actor model flavor: actors have addresses
(mailboxes), mailboxes are clonable, mailboxes can be passed
into other mailboxes. The Erlang/OTP version of the actor model
allows pretty much the same flexibility — Akka disallows
mailbox cloning in some interfaces but is morally actor-shaped.

Milner's late 1990s work on the **join-calculus**, **ambient
calculi**, and **action structures** explored the family of
process calculi further; for distributed systems, the actor
model has remained the most popular formalism (with **CSP**
its main rival — covered next).

## CSP vs actors — in depth

Hoare's **Communicating Sequential Processes** (CACM 1978) is
the other foundational model of message-passing concurrency. The
two are similar enough that it's easy to confuse them; they're
different enough that picking one shapes how you design your
system.

### Hoare's primitive — synchronized rendezvous

CSP's primitive is **synchronous rendezvous**. To send on a
channel, you block until a receiver is ready. The classic CSP
notation:

```
X!e       -- send expression e on channel X (blocks)
X?y       -- receive into variable y from channel X (blocks)
```

This is like a Unix pipe with zero buffer capacity — the kernel
hands the data from writer to reader at the syscall boundary.
Each send and each receive blocks. The "algebra" of CSP defines
guarded commands, traces, refusals, and deadlocks in terms of
the channel topology: which channels connect which processes.

Hoare formalized this as a process algebra. Two processes
"communicating" produce a synchronizing event; the formal
semantics describe the set of traces (sequences of
communications) a system can engage in, and the set of states
it can refuse to leave.

Hoare's motivation: designing multi-process operating systems
where each process has well-defined inputs and outputs, and
where you can prove correctness of the composition without
modeling the entire process internals.

### Hewitt's primitive — asynchronous mailbox

The actor model uses **asynchronous mailboxes**. The send is
fire-and-forget; the message sits in the mailbox until the
receiver gets to it.

```
send(actor, msg)    -- enqueue msg onto actor's mailbox
```

CSP-with-buffering: even CSP admits buffered channels; the
standard kind (`buf = 3`-style) is an explicit capacity. But the
*primitive* is rendezvous.

### The trade-offs

The trade-offs cascade into language design and runtime
behavior:

| | CSP (rendezvous) | CSP (buffered) | Actors |
|---|---|---|---|
| Sender blocks? | Until receiver ready | Until buffer has room | Never |
| Receiver blocks? | Until sender ready | Until message arrives | Until message arrives |
| Capacity | 0 | N (declared) | Unbounded (Commonware: bounded + overflow) |
| Topology | Channels are explicit | Channels are explicit | Implicit in actor acquaintance graph |
| Failure handling | Channels don't fail; if a process dies, the channel is just gone | Same | Mailboxes fail at actor exit; supervision needed |
| Verification | Process algebra | Process algebra (simpler) | Trace semantics (Agha) |

**CSP is best at pipeline.** When you're streaming data through
stages ("parse → validate → store"), you think in terms of
stages with channels between them. The backpressure is implicit:
the sender waits for the receiver, so you can't overwhelm a
slow stage. The composition is naturally laid-out because the
channels are first-class and visible.

**Actors are best at services.** When the unit is a long-lived
"thing" that other things send requests to (an account, a
session, a consensus participant), the actor model fits. Each
thing has an address. Other things send messages. The thing
handles them at its own pace. The composition is structured
top-down because the actors are the units.

Commonware uses actors because consensus is service-shaped.
Every long-lived participant (Voter, Batcher, Resolver, Engine)
is naturally an actor. Each actor accepts requests from many
senders; CSP would force each sender-receiver pair to declare a
buffered channel, which is bookkeeping for the sake of
bookkeeping.

### Erlang vs Go — the language-level expression

Erlang's process model is actor-shaped: each `pid` has a mailbox;
messages are sent via `Pid ! Msg`; receive blocks until a message
arrives. Supervision trees are baked into OTP. Erlang was the
first widely-used language to take the actor model to
production; it still has the deepest tooling around the actor
model.

Go's goroutine model is CSP-shaped. `chan T` is a typed channel;
`<-chan` blocks until a value arrives; `chan <-` blocks until
received (or until buffered). Goroutines aren't actors; you give
them state by closure, but the state isn't addressable as an
independent thing.

Rust's `tokio::sync::mpsc::Sender` is CSP-shaped: it has a
buffered queue (capacity 1 by default, configurable). A
`tokio::sync::oneshot` is CSP-rendezvous: it has capacity 0.

Rust's actor crates (`actix`, `ractor`, `bastion`, `axiom`)
borrow the actor model from Erlang/Akka. Commonware's `actor!`
macro is its own actor scaffolding; the underlying mechanism is
a Tokio task with a bounded mailbox.

### Why Rust can pick

Rust doesn't pick a side. The standard library provides
**synchronization primitives** (channels, mutexes, atomics). The
`tokio` runtime adds an **MPSC channel**. Application code on
top picks: actor-shaped or CSP-shaped or hybrid.

Commonware picks **actor-shaped**. The actor's `Mailbox<T, M>`
is the abstraction. The library exposes
`mpsc::channel`/`oneshot` only when the actor needs a side-band
that doesn't fit the actor model (e.g., a `oneshot` to deliver
one result to a future waiting for the result).

## Tokio tasks vs actors — when each is right

A Tokio task is a future; an actor is a task plus a **mailbox**.
That difference is the entire distinction. But "task plus
mailbox" is not just an additive change — it transforms what
you can express. This section walks through what tasks give
you, what the actor model adds, when each is right, and the cost
of choosing actors.

### What a Tokio task gives you

Spawning `tokio::spawn(async { ... })` returns a `JoinHandle`.
The task runs on the multi-threaded executor. It has:

- **A future** that the scheduler polls.
- **Wakers** registered by the future's `await` points (chapter
  02's "waker protocol" section).
- **Cancellation** via `handle.abort()` — the task is told to
  stop; awaited futures get dropped; resources clean up.
- **No identity** beyond `JoinHandle` — you can't address this
  task from another task except by waiting for it or aborting
  it.
- **No mailbox** — sending the task a message means writing into
  one of its channels, which you must construct and pass into it
  when spawning.

The shape is right for "fire-and-forget work" — processing a
block of bytes, running an HTTP request, churning through a
queue. The future runs once and exits.

### What the actor model adds

Wrap a Tokio task with an actor struct and you get:

- **A typed address.** `Sender<T>` is a cloneable handle to the
  actor's mailbox. Send `Sender<T>` into other actors' mailboxes
  (or simply share it). They'll send messages. The recipient
  actor's mailbox sorts out ordering.

- **A state machine.** The actor's struct is the state;
  `run(self)` is the state-transition function. Each call to
  `await self.mailbox.recv()` is "wait for next message."

- **Backpressure.** A bounded mailbox means "the actor can
  refuse to accept more than N messages." The actor's
  `Overflow` policy decides what happens to excess.

- **Supervision.** Every actor task is a child of some parent.
  If the parent crashes, the actor is aborted. This means an
  actor's lifecycle is bounded by its parent's — there's no
  floating orphan actor left in the runtime.

- **Liveness observability.** `mailbox_send_total`,
  `mailbox_send_backoff_total`, `mailbox_recv_closed_total` —
  every actor has these metrics. You know "actor X is in
  backoff" without instrumenting the actor itself.

### When to use a plain task

Plain tasks are right when:

- The work is **finite** — runs to completion, no looping
  awaiting messages.
- The work is **independent** — doesn't need a controllable
  address.
- The work is **fire-and-forget** — you don't need to wait for
  completion before continuing.

Examples: an HTTP request handler (responds once, exits);
metrics emission (read counter, write to sink, exit); a one-off
database migration ("walk all entries, transform, write,
exit").

Spawning plain tasks is cheap. The overhead is creating a
`Future` trait object and registering it with the scheduler. On
modern Tokio, that's ~200 ns.

### When to use an actor

Actors are right when:

- The work is **long-lived** — the actor handles many messages
  over its lifetime.
- The work needs **an address** — other components send to it.
- The work has **state** — the actor's struct holds
  configurations, references, cursors, caches.
- The work participates in **supervision** — if the actor
  crashes, its parent should know.
- The work might be **overloaded** — bounded mailbox +
  overflow policy + backpressure.

Consensus actors (Voter, Batcher, Resolver) are long-lived,
addressed, stateful, supervised, and can be overloaded. Hence
Commonware uses actors for consensus.

A pipeline stage (validate → sign → broadcast) might be an
actor or might be a chain of futures inside one actor. The
chain-of-futures pattern keeps everything in one task; the
per-stage actor pattern isolates failures but adds mailbox
overhead. Commonware picks the chain-of-futures pattern in
Simplex's Voter because the stages share state
(`state.current_view`, `state.locked_certificate`, etc.).

### The cost of actors

Actors aren't free. The bookkeeping:

- **Mailbox memory.** A `MailboxState<T, M>` is `Arc<...>`. The
  `ready` queue holds up to `capacity` messages; each message
  has `T`'s size. Memory cost: O(capacity × sizeof(T)). At
  `capacity = 1024` and `T` is a small enum (~32 bytes), that's
  32 KiB per actor. Across hundreds of actors in a validator,
  that's a few megabytes — fine on any modern machine.

- **Mailbox CPU cost per send.** Acquire the `Mutex`, push to
  `VecDeque`, possibly wake the receiver, release. ~30–50 ns in
  the hot path. The `crossbeam-channel` data structure achieves
  ~10 ns; Commonware's mailbox is simpler, slightly slower.

- **Mailbox CPU cost per receive.** Acquire `Mutex`, pop from
  `VecDeque`, possibly refill from overflow (acquire second
  `Mutex`), release. ~40–80 ns per message. Burst reception
  bounds this: one wakeup delivers many messages.

- **Heap allocations on send.** None in the hot path. The
  message is moved into the queue. Overflow path may allocate
  only if you're holding a `VecDeque<T>` that's grown beyond
  its current capacity.

- **Receiver polling.** The Tokio task is `await`ing a waker;
  it's parked. Each wakeup + each scheduled poll costs ~5 µs
  (the cost of context-switching back into the task).

For comparison, a hand-rolled `Arc<Mutex<VecDeque<T>>>` +
condvar is roughly similar but loses the backpressure
visibility (`metrics`), the role-specific `Overflow` trait
(policy), and the supervision integration.

The actor model is **more expensive than raw shared state** but
**cheaper than a hand-rolled queue in correctness**. It's the
right default when the workload is address-shaped.

## Lock-free data structures — in depth

Commonware uses lock-free data structures in two narrow places
(see "Lock-free data structures — when Commonware uses them"
above). This section makes the choice principled by surveying
what "lock-free" actually delivers, the landmark structures in
the field, and where the trade-off lands.

### Why lock-free matters at all

Lock-free programming matters in three places:

1. **Latency under contention.** A `Mutex` that's contended
   blocks the caller. The thread suspends, the OS reschedules
   it later. Tail latency grows. A lock-free structure serves
   the request in bounded time — usually by retrying a CAS
   (compare-and-swap) a few times.

2. **Real-time guarantees.** A task that holds a `Mutex` can
   be preempted between lock and unlock. If another task needs
   the same `Mutex` in a bounded real-time window, it can't
   have it. Lock-free structures don't have this problem.

3. **Async / signal-safety.** A `Mutex` lock can call `futex`
   internally, which can park the thread. In a signal handler,
   that's a deadlock. Lock-free operations don't park.

For "regular" server code, contention reasons dominate. For
request-response middleware, latency tail is the killer
metric.

### Michael-Scott queue (1996)

The classic lock-free MPMC (multi-producer, multi-consumer)
bounded queue. The paper: Michael & Scott, "Simple, Fast, and
Practical Non-Blocking and Blocking Concurrent Queue
Algorithms," PODC 1996.

The shape: an array of slots, each slot a "ticket" with a flag
(empty / waiting / filled). Producers atomically claim a ticket,
write to it, set its flag to filled. Consumers atomically claim
a reading slot, read it, set its flag to empty. The slots form
a ring buffer; tickets wrap around.

Each operation uses one CAS to claim a slot. Reads and writes
to the slot's payload are then unsynchronized (the CAS-ordering
guarantees the visibility).

Variants:

- **Bounded MPMC** (the original): a fixed-size array,
  overflow causes producers to retry.
- **Linked list** (later work): an unbounded lock-free queue
  with malloced nodes.

Commonware's `Mailbox` is conceptually similar but uses
`Mutex<VecDeque<T>>` instead. Why? See "When Commonware uses
`Mutex`" below.

### Treiber stack (1986)

The classic lock-free LIFO (stack). Treiber, "Systems
Programming: Coping with Parallelism," 1986.

The shape: a linked list of nodes. Push atomically swaps the
head pointer to the new node (pointing to the old head). Pop
atomically swaps the head to the head's `next` pointer.

The simplest of all lock-free structures. Just one CAS per
op. The ABA problem: a pop sees head = X, computes
X.next = Y, but between "see head" and "swap to Y," another
pop swapped head from X to Y to X again (popped X, then
re-pushed). The reader's CAS succeeds but now points to
freed memory.

The ABA fix: tag the pointer with a version counter; the
CAS compares both. Commonware doesn't use Treiber stacks in
production code.

### Hazard pointers (2002)

The memory reclamation question: if a thread reads a pointer,
but another thread frees the pointed-to object, the reader is
using freed memory. (Use-after-free.) With `Arc<T>`, the
reference count prevents this — but reference-count updates
are not free; on contested counters, they can be the bottleneck.

**Hazard pointers** (Michael, "Safe Memory Reclamation for
Lock-Free Objects," IEEE TPDS 2002): each thread maintains a
list of pointers it's currently reading. Before freeing an
object, the reclaimer scans all threads' hazard lists; if any
thread is currently reading the object, defer the free to
later.

Equivalent reclamation schemes:

- **Epoch-based reclamation (EBR, 2003, Fraser & Harris)**:
  partition time into epochs; objects freed in epoch N are
  released only when all threads have left epoch N (or higher).
  Simpler than hazard pointers but slightly more aggressive on
  memory: a thread that's behind on progress can prevent
  reclamation of objects for arbitrarily long.
- **QSBR (Quiescent State-Based Reclamation, 2007)**: even
  cheaper than EBR; readers must "announce quiescent states"
  explicitly. Used in Linux kernel RCU.

Commonware's mailboxes don't need hazard pointers because the
backing storage lives in an `Arc<MailboxState<T, M>>` with a
lifetime tied to the actor's task. The structure is freed only
when no one holds a `Sender<T>`. Lifetime is managed
structurally, not via the lock-free engine.

### The Commonware lock-free mailbox design

Commonware's **canonical** mailbox (in `actor/src/mailbox.rs`)
uses a `Mutex<VecDeque<T>>` not because the team didn't know
about Michael-Scott, but because the engineering trade-off
favored simplicity. The numbers, in practice:

- The hot path of a Commonware `send` (push to ready queue,
  wake receiver) takes ~30–50 ns on a modern x86 with light
  contention. The Michael-Scott queue's hot path is similar
  (one CAS to claim a slot, write to it). Commonware's mailbox
  wins slightly when contention is light (Mutex is fast when
  uncontended); the Michael-Scott queue wins slightly under
  high contention.

- Commonware's mailbox is about 200 lines of Rust. A
  Michael-Scott queue is about 150 lines of `unsafe` Rust.
  Twice as much lifetime/aliasing reasoning for a marginal
  throughput gain.

- The `Overflow` trait is the actual innovation: it lets you
  swap in coalescing, latest-wins, deduplicating, batch-
  draining overflow strategies without changing the
  push/pop logic. That generality is hard to achieve with a
  Michael-Scott bounded array.

So: Commonware is correctness-first. A `Mutex<VecDeque<T>>` is
correct by construction. A Michael-Scott queue requires a
formal argument about CAS orderings; Commonware's
documentation states this argument elsewhere.

### When Commonware uses `Mutex`

The decision rule (echoed in chapter 06 and across `storage/`):

> If you can express your data structure's invariants in terms
> of "lock; mutate; unlock", use `Mutex`. If the unlock leaves
> the structure in a state where another thread could observe
> a partial invariant, *then* you need lock-free.

The mailboxes, journal buffers, QMDB commit-and-prune paths,
and most of `storage::qmdb` are lockable. They've all been
proven to satisfy the "lock-mutate-unlock" invariant. So they're
`Mutex`-based.

The **runtime counters** (`Arc<AtomicUsize>`) are lock-free
because they're counters: incrementing a counter is a single
CAS; the invariant ("the count has gone up by one") holds
atomically. The waker registration in `recv()` is similar.

### When Commonware uses `unsafe`

Lock-free structures are the main consumer of `unsafe` in
Commonware. Strict rule (per `AGENTS.md`): every `unsafe`
block has a `// SAFETY:` comment that names the Rust soundness
property being relied on (e.g., "this pointer is aligned
because it was just allocated as `align_of::<u64>()`," or "this
is the only writer at this point because the Mutex is held").

The reviewer focus (per `AGENTS.md`): any `unsafe` that
processes untrusted input is suspect. The state-of-the-art is
"minimum `unsafe`, maximum review."

## Backpressure patterns in full — and the TCP analogy

Section "Backpressure: drop, block, coalesce, spill" above
introduces the four policies. This section justifies each one
in terms of what TCP's flow control has taught us, and walks
through how the four interact when they meet in the same
system.

### The four policies, again — with examples

| Policy | Sender | Receiver feedback | Commonware embodiment |
|---|---|---|---|
| **Drop** | Doesn't wait; message gone | `Outcome(Closed)` / `Rejected` | `UnreliableMailbox` |
| **Block** | Waits on semaphore | Sender waits | `tokio::sync::mpsc` (buffered channel that blocks on full) |
| **Coalesce** | Replaces existing | Implicit (`Backoff`) | Custom `Overflow<K, T>` keyed by message-id |
| **Spill** | Sits in overflow queue | `Backoff` | Default `Mailbox<T, Reliable>` |

**Drop** is correct when:

- The message is idempotent (re-sent = same effect).
- The message can be re-derived later (TTL'd retry, gossip
  re-broadcast).
- Latency matters more than completeness (heartbeat
  messages, telemetry beacons).

Commonware uses drop with gossip (chapter 09): if a gossip
message is dropped, the network re-gossips it via other peers
within seconds.

**Block** is correct when:

- The message is part of a synchronous chain (request →
  response, with the caller waiting).
- Dropping is unacceptable.
- The system has a clear "what's the backpressure budget"
  answer (typically a real-time deadline).

The `tokio::sync::mpsc` channel `Sender::send(msg).await` is
the canonical block-on-full channel. Commonware uses this for
request/response channels between actors (the `oneshot` return
to the response channel is a single message, blocked at the
sender if the response channel is full).

**Coalesce** is correct when:

- Multiple messages can be summarized by one (latest state,
  latest commit, latest signature).
- The "old" message is strictly less useful than the "new"
  one.

The `Originator::submit` (chapter 10) coalesces requests for
the same key into a single downstream submission. State-update
protocols (PBFT's "view-change with highest-prepared-cert")
are coalescing by construction.

**Spill** is correct when:

- The receiver must eventually see every message.
- The receiver has a bounded rate of consumption and bursts
  happen.
- "Sustained overload" is a rare condition; "intermittent
  bursts" is the common case.

Consensus protocols fit this: votes must be retained, but the
rate of consensus messages is bounded by the view frequency,
not by sender demand.

### The TCP analogy

TCP's flow control is the canonical backpressure design:

1. **Sliding window** — the receiver tells the sender how much
   buffer it has (`rwnd` field in TCP headers). The sender
   doesn't send more until more buffer becomes available.
2. **Slow start / congestion avoidance** — the sender infers
   network capacity from packet loss and adjusts its sending
   rate accordingly.
3. **Block until room** — TCP's `write` to a socket whose
   receiver buffer is full blocks the user.

Commonware's `Mailbox<T, Reliable>` is TCP slide-window-style:
the sender's `send_backoff_total` climbs while the receiver's
ready queue is full; the implicit "wait" happens via the
sender's next step in its own actor (don't push more until you
have headroom).

The `Mailbox<T, Unreliable>` is closer to UDP-over-a-network:
the sender pushes, drops under overload, and asks "did the
receiver ack?" separately. UDP + your own reliability layer is
how real-time protocols (WebRTC, QUIC for media) do it.

`oneshot` channels are TCP-rendezvous (capacity 0); mpsc
buffered channels are TCP-buffered.

### Mixing policies

A real Commonware validator mixes all four:

- **Spill** for consensus messages (`Reliable` mailbox on
  Voter, Batcher).
- **Drop** for gossip (chapters 09).
- **Block** for `oneshot` replies to consumers waiting for an
  explicit response.
- **Coalesce** for state-update requests in collectors
  (chapter 10).

The patterns coexist by **layering**. Each layer (P2P,
Broadcast, Simplex) has its own mailboxes; the within-layer
messaging uses the policy appropriate for that layer's
semantics.

## Work stealing in depth

Section "Work stealing — and how actors fit" above introduces
Tokio's work-stealing scheduler and notes that actors fit into
it. This section goes deeper into the alternatives — worker
threads, thread-per-core, share-nothing, share-everything — and
how each interacts with the actor model.

### Tokio's strategy

Tokio's multi-threaded runtime uses a **work-stealing**
scheduler:

- N worker threads. Each owns a **runqueue** of runnable tasks.
- When a worker has work, it polls its queue.
- When a worker runs out, it **steals** half the other workers'
  queues (round-robin).
- Wakes (`Waker::wake`) put tasks on the current worker's
  queue (so a task that wakes another task tends to keep
  them co-located).

This gives **load balancing** without explicit migration: hot
spots eventually redistribute.

The cost: **cache coherence** pressure. A task running on
worker A wakes a task that was last running on worker B; the
worker's task runs on B; the data may not be in B's cache.
Work-stealing limits but doesn't eliminate this.

### Actors fit naturally

An actor's task is just a future. The actor's `Mailbox` is
not a runqueue — it's the queue the actor's future polls when
running. So the actor's task participates in work-stealing
like any other task:

- When the actor has messages to process, it's scheduled
  (woken by the sender), runs on a worker, processes, yields,
  waits again.
- When the actor has no messages, it sleeps (its waker is
  registered, and `recv().await` suspends).

The mailbox does not change Tokio's scheduling. It changes
**what the task does** when it runs.

### Thread-per-core alternative

The opposite of work-stealing is **thread-per-core**: pin each
task to a specific OS thread; rely on the application to
distribute work across cores. Seastar (ScyllaDB's framework)
does this aggressively.

Pros:

- **Worst-case latency bound:** a task on thread T runs alone on
  that thread; no other task can preempt it. Tail latency is
  bounded by the task's own compute.
- **Cache warmth:** each thread's tasks share an L2 cache, and
  you control placement.

Cons:

- **Load balancing is the application's problem:** an
  unbalanced workload leaves some cores idle while one is
  saturated.
- **Cross-thread communication is expensive:** no shared state
  between threads → must copy or use shared memory with
  expensive synchronization.

For Commonware, which has many small cooperating tasks
(voters, batchers, resolvers all in one process), work-
stealing is the better fit. For a monolithic database server,
thread-per-core is the better fit.

### Mailboxes and locality

The actor model adds **state locality**: each actor's state
lives on the actor's worker thread (whichever worker is
running the actor right now). When other actors send messages,
they don't access the actor's state directly; they go through
the mailbox, which is shared (an `Arc<MailboxState<T, M>>`).

This trade-off:

- **Reads of the actor's state** are single-threaded (the
  actor's task); no contention on that side.
- **Writes of the actor's state** happen in the actor's
  message-processing loop, single-threaded; no contention.
- **Cross-actor reads** are impossible (state is owned by
  one actor). To read, you send a message; the data structure
  is the message.

If two actors want to share read-only data, they each hold a
clone of an `Arc<T>` where `T: Sync`. (This is the "shared
immutable, exclusive mutable" pattern, but at actor
granularity.)

## Erlang/OTP supervision in depth

Section "Supervision: Erlang/OTP's 'let it crash'" above
introduces the philosophy and maps it to Commonware. This
section goes through OTP's restart strategies in detail and
explains how Commonware's `Supervisor` (chapter 02) compares.

### "Let it crash"

Erlang's design philosophy (Armstrong 2003 dissertation):

> Don't write defensive code that anticipates every failure.
> Instead, structure the code so that any failure is a restart
> trigger. The supervisor starts a new instance with a fresh
> state.

This works because:

1. Functional code has **no hidden state.** An Erlang process
   has its mailbox and its function arguments; nothing else.
   A "restart" is just "spawn a new process with the same
   initial function."

2. Supervision is **hierarchical.** A supervisor can supervise
   other supervisors; each level has well-defined restart
   rules.

3. **Failures are observed.** A process crashing emits an exit
   signal; a supervisor's restart is observable; you can see
   "process X restarted 5 times in 1 minute" and act.

Commonware keeps the philosophy but adapts it for the BFT
context. The differences:

- Commonware doesn't auto-restart a consensus actor. A
  Byzantine-faulting consensus engine should not silently
  respawn; the rest of the network already saw it fail. A
  validator restart is a *separate* event (peers learn about
  it via the network layer), not a local recovery.

- Commonware's supervisor **does** abort descendants when a
  parent crashes. This is the structural deadlock prevention
  — if the parent that held lock L died, descendants waiting
  on L are aborted, not stuck.

- The Commonware `Stopper` API (chapter 02) provides explicit
  shutdown signaling for graceful drain.

### OTP restart strategies

Erlang's supervisor supports three restart strategies (per the
OTP `supervisor` docs):

**`one_for_one`.** Only the failed child restarts. Sibling
children are untouched.

```
supervisor
  |
  +-- child A
  +-- child B  -- crashed
  +-- child C
```

After B crashes:

```
supervisor
  |
  +-- child A
  +-- (child B' just spawned)
  +-- child C
```

Use when children are independent (B and C don't need B's
state to function). Commonware's typical structure (one actor
per consensus role): each actor is independent, so
`one_for_one` semantics apply.

**`one_for_all`.** All children restart when any one fails.

```
supervisor (one_for_all)
  |
  +-- child A
  +-- child B  -- crashed
  +-- child C
```

After B crashes:

```
supervisor (one_for_all)
  |
  +-- (just spawned A', B', C')
```

Use when children share state tightly (B's correctness depends
on A being restarted too). Commonware rarely needs this.

**`rest_for_one`.** The failed child and all children started
after it restart.

```
supervisor (rest_for_one)
  |
  +-- child A
  +-- child B  -- crashed
  +-- child C  (started after B)
```

After B crashes:

```
supervisor (rest_for_one)
  |
  +-- child A
  +-- (just spawned B', C')
```

Use when there's a startup-order dependency. Commonware
sometimes uses this in deployment code: a JWT-authenticated
child depends on the JWT issuer being set up.

### Supervision intensity and period

OTP also specifies **supervision intensity**: the maximum
number of restarts within `Period`. If exceeded, the
supervisor itself terminates.

This is a **liveness safeguard**: a child that crashes in a
loop is failing in a way that's not recoverable; the
supervisor gives up and the system is supposed to escalate.

Commonware's analog: the runtime aborts the parent task when a
child fails too often (chapter 02). The exact number depends
on the deployment; default is set conservatively.

### Commonware's tree

```
ctx_root                                  // root
  |
  +-- child("voter")                      // consensus
  |    |
  |    +-- spawn() -> voter_task          // the actor
  |
  +-- child("batcher")
  |    |
  |    +-- spawn() -> batcher_task
  |
  +-- child("resolver")
       |
       +-- spawn() -> resolver_task
```

If `voter_task` aborts: nothing else is aborted (because the
voter is a child of `ctx_root`, not of `batcher_task` or
`resolver_task`). If `ctx_root`'s task aborts: all children
abort.

This is `one_for_one` semantics at the level of the runtime:
each actor has its own fate; the root owns the propagation.

### Comparison

| Property | OTP | Commonware |
|---|---|---|
| Auto-restart | Yes (configurable strategy) | No (typically; BFT context) |
| Hierarchical | Yes (supervisor tree) | Yes (`ctx.child(...).spawn(...)`) |
| Restart intensity | Max N in M seconds | Implicit via runtime abort policy |
| Process model | Process per actor | Task per actor (lighter than OTP process) |
| Process isolation | VM-level isolation | Single-process, multi-task |
| Crash propagation | Link + monitor | Mandatory (parent abort → descendant abort) |
| Restart types | `permanent`, `transient`, `temporary` | n/a; abort propagation always |
| State recovery | Re-init from config | Lazy replay (chapter 17) |

OTP's `permanent`/`transient`/`temporary` restart types are
about what kind of child deserves restart. Commonware skips
that distinction because it doesn't auto-restart.

## The `actor!` macro in depth

Section "The `actor!` macro" above shows the surface syntax.
This section walks through what the macro expansion looks like
and the trade-offs of choosing a macro vs. a trait.

### Surface syntax

```rust
actor! {
    pub struct Voter<E: Rng + Spawner + Metrics + Clock> {
        // The actor's fields. Each becomes a member of `Voter`.
        config: Config,
        mailbox: Mailbox<Message>,
        scheme: Scheme,
        automaton: Automaton,
        reporter: Reporter,
        supervisor: Supervisor,
        state: State,
    }

    // The actor's run loop.
    async fn run(mut self) {
        let mut receiver = self.mailbox.receiver();
        loop {
            select! {
                msg = receiver.recv() => {
                    self.handle(msg).await;
                }
                _ = self.stop_signal => break,
            }
        }
    }
}
```

### What the macro produces

The `actor!` macro expansion generates roughly:

```rust
pub struct Voter<E: Rng + Spawner + Metrics + Clock> {
    config: Config,
    mailbox: Mailbox<Message>,
    scheme: Scheme,
    automaton: Automaton,
    reporter: Reporter,
    supervisor: Supervisor,
    state: State,
    // ...
}

impl<E: Rng + Spawner + Metrics + Clock> Voter<E> {
    pub fn start(
        config: Config,
        supervisor: &dyn Supervisor,
        // possibly other args
    ) -> (Mailbox<Message>, JoinHandle) {
        let (mailbox, receiver) = mailbox::mailbox(config.mailbox_size);
        let actor = Voter { config, mailbox, /*...*/, /*state init*/ };
        let handle = supervisor.child("voter").spawn(move |ctx| async move {
            // The body the user wrote:
            let mut self_actor = actor;
            let mut receiver = receiver;
            loop { /* select! body */ }
        });
        (mailbox, handle)
    }
}

impl<E: Rng + Spawner + Metrics + Clock> Mailbox for Voter<E> {
    type Message = Message;
    fn mailbox_handle(&self) -> &Mailbox<Message> { &self.mailbox }
}
```

Roughly. The exact form varies; this is the shape.

### The trade-offs

**Why a macro instead of a trait?**

In Rust, you can't write:

```rust
trait Actor {
    fn run(self) -> impl Future<Output = ()>;
}
```

…and have callers `spawn` it, because `impl Trait` in trait
methods has limited support, and the future type would be
unnameable.

You could write:

```rust
trait Actor {
    fn run<'a>(&'a mut self, mailbox: Receiver)
        -> Pin<Box<dyn Future<Output = ()> + Send + 'a>>;
}
```

…but boxing the future allocates per-call, which kills the
hot path.

The macro solves both: it inlines the future's body into the
spawn site, so the future is a concrete (named) type, fully
inlined, no allocation, no boxing. The cost: the user writes
the future syntax inside the macro, and macro errors can be
cryptic.

**Why not just write it out manually?**

You could. The full `Voter` actor (mailbox creation, supervisor
spawn, waker wiring, drop glue) is roughly 200 lines of
boilerplate. The macro reduces it to ~50. Across a dozen actors
in a single application, that's 1,500 lines of duplication
eliminated.

### The pattern's limitations

- **Each `actor!` is a separate struct.** You can't share
  fields between two actors via an `actor!` macro; you'd
  extract a struct module and have both actors hold a clone.

- **Generic actors.** `actor!` does support generic
  parameters, but the syntax is non-trivial — you have to
  write the bounds yourself, and the macro doesn't add bounds
  you didn't specify.

- **`Debug`, `Clone`, etc.** The macro generates `Debug` and
  `Clone` derives for fields that implement them, but if a
  field doesn't implement those, you must implement them
  yourself.

### When to use `actor!` vs hand-roll

- **`actor!`** when the actor's pattern is "receive, handle,
  repeat." Most actors fit this.

- **Hand-roll** when the actor's run loop has a different
  shape (e.g., many concurrent awaits with cancellation,
  custom cleanup). E.g., the `Resolver` actor has an
  unusual pattern: it sends to multiple peers simultaneously
  and waits for a quorum; the macro's natural-shape `select!`
  doesn't fit cleanly, so `Resolver` is hand-rolled.

## Shutdown coordination in depth

Section "Shutdown: `Stopper`, `Signal`, and the multi-stop
dance" above introduces the API. This section walks through
the protocol in detail, with the multi-stop safety argument.

### The shutdown protocol

The goal: when one part of the system decides "we're done," all
parts agree and shut down cleanly. The pieces:

```rust
let stopper = Stopper::new();            // 1. Create
let signal = stopper.stopped();            // 2. Get a 'Signal'
let signal_clone = signal.clone();         // 3. Each task holds a clone

// Task A:
ctx.spawn(|_| async move {
    select! {
        _ = work() => {},
        _ = signal_clone.stopped() => { /* begin drain */ }
    }
});

// Task B: (a different task, also holding the clone)
// Same shape

// Initiator:
stopper.stop(0, Some(Duration::from_secs(30))).await?;
//   ... after this returns, all tasks have observed, drained, ...
```

### The multi-stop safety argument

The interesting case: `stop()` is called twice. From two
shutdown paths — a deadline expires, AND a peer reports an
unrecoverable error. Both want to "stop the world."

Without multi-stop safety, the second call might:

- Re-fire `waiters.notify_all`, causing some tasks to run
  their drain logic twice.
- Block forever, holding the rest of the system.

Both would be bugs. The `Stopper::stop` implementation:

```rust
pub fn stop(&self, value: i32, timeout: Option<Duration>) {
    // 1. Try to set the inner value (atomic CAS).
    let mut guard = self.inner.value.lock();
    if guard.is_some() {
        // Already stopped.
        return Err(Error::Exited);
    }
    *guard = Some(value);
    drop(guard);

    // 2. Notify all waiters.
    self.inner.waiters.notify_all();

    // 3. Wait for outstanding Signal clones to drop (bounded by timeout).
    wait_for_drain(timeout).await;

    Ok(())
}
```

The atomic-first-step ensures that exactly one `stop()` call
succeeds. All other calls return `Error::Exited` immediately.
This is the **multi-stop safety property**.

### Drain semantics

The "wait for outstanding `Signal` clones to drop" step is the
subtle bit. Each `Signal` is an `Arc<StopperInner>` clone. When
the last `Signal` is dropped, the count goes to zero; the
`stop()` future can resolve.

What does "drop" mean? It means the task that owns the `Signal`
clone has returned (or panicked, or been aborted). The
`stop()` future resolving means "all tasks have acknowledged the
shutdown."

If a task holds the `Signal` forever (infinite loop, never
observes `signal.stopped()`), the `stop()` future waits for
the timeout, then returns `Result::Err(Error::Timeout)`. This
is the **bounded-shutdown** property.

### Practical usage patterns

**Pattern 1: cooperative drain.**

```rust
async fn run_loop(signal: Signal) {
    loop {
        select! {
            msg = mailbox.recv() => process(msg).await,
            _ = signal.stopped() => {
                // Drain remaining work
                while let Some(msg) = mailbox.try_recv() {
                    process(msg).await;
                }
                return;
            }
        }
    }
}
```

The task drains the mailbox before returning. Other tasks
finish their own drains; `stop()` resolves.

**Pattern 2: ignore and exit.**

```rust
async fn run_loop(signal: Signal) {
    loop {
        select! {
            msg = mailbox.recv() => process(msg).await,
            _ = signal.stopped() => return,
        }
    }
}
```

The task drops mid-processing. Useful when the task has no
state that needs flushing.

**Pattern 3: critical-section abort.**

```rust
async fn run_loop(signal: Signal) {
    let mut in_critical = false;
    loop {
        select! {
            biased;
            _ = signal.stopped() => {
                if !in_critical {
                    return;
                }
                // Else: continue critical section, then return
            }
            msg = mailbox.recv() => {
                in_critical = true;
                process(msg).await;
                in_critical = false;
            }
        }
    }
}
```

The `biased` keyword evaluates the shutdown branch first. A
task in the middle of a critical section finishes it before
returning, avoiding torn writes.

## Common patterns in full — recap with examples

Section "Common patterns in production" above lists four.
This section walks through each in production-grade form,
with realistic Commonware code.

### Supervisor tree (full)

```rust
fn spawn_validator_stack(
    ctx: &mut Context,
    cfg: ValidatorConfig,
) -> Result<(Mailbox<VoterMsg>, Mailbox<BatcherMsg>, ...)> {

    let voter_mailbox = Voter::start(ctx, cfg.clone());

    let (batcher_tx, batcher_rx) = mailbox::mailbox(cfg.batcher_mailbox_size);
    ctx.child("batcher").spawn(move |ctx| {
        Batcher::new(batcher_rx, voter_mailbox.clone(), cfg.clone()).run(ctx)
    });

    let (resolver_tx, resolver_rx) = mailbox::mailbox(cfg.resolver_mailbox_size);
    ctx.child("resolver").spawn(move |ctx| {
        Resolver::new(resolver_rx, voter_mailbox.clone(), cfg.clone()).run(ctx)
    });

    Ok((voter_mailbox, batcher_tx, resolver_tx))
}
```

If `voter_mailbox`'s task aborts (panic), only that task dies.
If `ctx` (the root) ends, all three abort. The mailbox clones
let components send to each other; the supervision tree ensures
no orphans.

### Fan-in (full)

```rust
// Many producers, one consumer.
let (collector_tx, mut collector_rx) = mailbox::mailbox(4096);
for producer in producers {
    let tx = collector_tx.clone();
    ctx.child(producer.name()).spawn(move |ctx| {
        producer_loop(ctx, tx).await
    });
}
ctx.child("collector").spawn(move |ctx| {
    collector_loop(ctx, &mut collector_rx).await
});
```

The collector's mailbox is shared across all producers. If
the collector is slow, every producer sees `Backoff`. The
collector can also implement a custom `Overflow` (e.g.,
latest-wins) for sources that don't need every message.

### Fan-out (full)

```rust
// One dispatcher, many workers.
let (control_tx, mut control_rx) = mailbox::mailbox(1024);
let worker_handles: Vec<_> = (0..N).map(|i| {
    let tx = control_tx.clone();
    ctx.child(format!("worker-{i}")).spawn(move |ctx| {
        worker_loop(ctx, &mut control_rx_clone, ...).await
    })
}).collect();
```

Each worker has its own mailbox; the dispatcher routes by
key. The "rabbit-mq pattern": topic-based fan-out without a
broker.

### Graceful drain (full)

```rust
async fn consumer_loop(
    mut receiver: Receiver,
    signal: Signal,
) {
    loop {
        select! {
            msg = receiver.recv() => {
                process(msg).await;
            }
            _ = signal.stopped() => {
                // Phase 1: stop accepting new sends (close the
                // mailbox so producers get Feedback::Closed)
                receiver.close();
                // Phase 2: drain
                while let Some(msg) = receiver.try_recv() {
                    process(msg).await;
                }
                return;
            }
        }
    }
}
```

The two-phase drain: close the mailbox so no new messages
arrive, then drain the existing queue. Senders see `Closed`
and stop sending.

### Request-response (full)

```rust
let (tx, mut rx) = mailbox::mailbox(1024);
let (reply, reply_rx) = oneshot::channel();
tx.send(Request { payload, reply });
let response = reply_rx.await?;
```

The request includes the reply channel. The actor's handler
gets the message, does work, sends the reply. The original
sender awaits the reply.

This pattern is how many of Commonware's RPC-style calls
work: it's CSP-shaped (the `oneshot` is a CSP rendezvous)
embedded in an actor-shaped component.

### State machine (full)

Many Commonware actors are state machines with explicit
transition functions:

```rust
enum VoterState {
    Idle,
    WaitingForProposal { view: View },
    WaitingForNotarize { view: View, proposal: Proposal },
    WaitingForNullify { view: View },
}

async fn handle(&mut self, msg: Message) {
    let next = match (&self.state, msg) {
        (VoterState::Idle, Message::Proposal(p)) => {
            VoterState::WaitingForNotarize { view: p.view, proposal: p }
        }
        (VoterState::WaitingForNotarize { proposal, .. },
            Message::Notarize(n)) if proposal.matches(&n) => {
            VoterState::Idle
        }
        // ... other transitions ...
        (state, msg) => {
            // Unexpected message in this state.
            log_warn!(state, msg);
            return;  // no transition
        }
    };
    self.state = next;
}
```

The `match` ensures type-safe transitions. The actor's `run`
loop calls `handle` for each message; the state machine
evolves.

## Exercises

These exercises ask you to **implement parts of the actor
model** that Commonware uses. They build a working mailbox and
supervisor from scratch.

### Exercise 1 — Build a minimal bounded mailbox

Without using `actor::mailbox`, write a `SimpleMailbox<T>` with:

- `Sender<T>::send(&self, msg: T) -> Result<(), ()>` — push to
  a `VecDeque<T>` bounded by capacity; return `Err(())` when
  full.
- `Receiver<T>::recv(&mut self) -> Option<T>` — block on a
  condvar until a message arrives; return the message.
- A `close()` method that returns the receiver to a state
  where `recv()` returns `None`.

Tests:

- Sender → receiver round-trip.
- Two senders → one receiver, interleaved.
- Capacity 1; two consecutive sends; second returns `Err`.
- Close; receiver sees `None`.

This is roughly `actor::mailbox` minus the `Overflow` and
`Policy` machinery.

### Exercise 2 — Add an overflow policy

Extend Exercise 1's mailbox with a `Policy`:

- When the `VecDeque` is full, instead of returning `Err`,
  push to an `overflow: VecDeque<T>`.
- When the receiver pops a message from `ready`, refill
  `ready` from `overflow`.
- `send()` returns `Ok` always.

Tests:

- Capacity 1; ten sends; the first lands in `ready`; the next
  nine in `overflow`.
- Receiver pops 10 messages (in order: the one in `ready`,
  then the nine in `overflow`).

### Exercise 3 — Build a hierarchical supervisor

Write a `Supervisor` that:

- Spawns N child tasks on an executor.
- Each child reports "alive" or "dead" via a oneshot.
- If any child dies, the supervisor aborts all siblings.
- After all children have exited (naturally or by abort), the
  supervisor exits.

Test with a child that panics; verify siblings abort.

### Exercise 4 — Implement the `actor!` macro by hand

Without using `commonware-macros::actor!`, write the same
Scaffold for a `Counter` actor that has:

- A `count: u64` field.
- A `Message::Increment` and `Message::Get(oneshot::Sender<u64>)`.
- A `run()` future that processes messages until shutdown.

Compare line counts to using `actor!`.

### Exercise 5 — Add metrics to your mailbox

Extend Exercise 2's mailbox with `Arc<AtomicUsize>` counters
for `send_total`, `send_overflow_total`, `recv_total`.

Compare the throughput to a `Mutex<VecDeque<T>>`-only version.
What do you observe?

→ Next: **Chapter 16 — Advanced Storage** (QMDB, MMR, MMB).
The authenticated data structures that make verifiable state
possible.

## Appendix A — The full mailbox implementation, line by line

`actor/src/mailbox.rs`. The Mailbox is the central type:

```rust
pub struct Mailbox<T, M = Reliable> {
    state: Arc<MailboxState<T, M>>,
    // ...configuration...
}

struct MailboxState<T, M> {
    // The bounded ready queue
    ready: Mutex<VecDeque<T>>,
    // The unbounded overflow queue (may contain policy-specific data)
    overflow: Mutex<M::Overflow>,
    // Configuration
    capacity: NonZeroUsize,
    // Shutdown signal
    closed: AtomicBool,
    // Wake notification for the receiver
    waker: Mutex<Option<Waker>>,
}
```

### The send path

```rust
impl<T, M> Mailbox<T, M> {
    pub fn send(&self, msg: T) -> M::Feedback {
        // Check closed state
        if self.state.closed.load(Ordering::Acquire) {
            return M::closed_feedback();
        }

        // Try fast path: push to ready
        let mut ready = self.state.ready.lock();
        if ready.len() < self.state.capacity.get() {
            ready.push_back(msg);
            drop(ready);
            // Wake receiver
            self.wake_receiver();
            return M::ready_feedback(Feedback::Ok);
        }
        drop(ready);

        // Slow path: handle overflow
        let mut overflow = self.state.overflow.lock();
        let handled = M::handle(&mut overflow, msg);
        drop(overflow);

        if handled {
            M::overflow_feedback(true)
        } else {
            M::overflow_feedback(false)
        }
    }
}
```

The flow: try ready (fast, no overflow work). If full, go to the
overflow policy (slow, may reject).

### The recv path

```rust
impl<T, M> Mailbox<T, M> {
    pub async fn recv(&mut self) -> Option<T> {
        loop {
            // First, drain ready
            {
                let mut ready = self.state.ready.lock();
                if let Some(msg) = ready.pop_front() {
                    // Refill ready from overflow before returning
                    self.refill_ready_from_overflow();
                    return Some(msg);
                }
            }

            // No message — register waker and yield
            self.register_waker().await;
        }
    }

    fn refill_ready_from_overflow(&self) {
        // For reliable: drain overflow into ready
        // For unreliable: try drain; if rejected, message goes back to overflow
        let mut overflow = self.state.overflow.lock();
        let mut ready = self.state.ready.lock();
        overflow.drain(|msg| {
            if ready.len() < self.state.capacity.get() {
                ready.push_back(msg);
                None  // msg accepted
            } else {
                Some(msg)  // msg refused, back to overflow
            }
        });
    }
}
```

The receiver pops one, then **eagerly refills** ready from overflow.
This keeps ready as full as possible (for the next pop) without
holding the locks across the consumer's processing.

## Appendix B — The two modes, side by side

```rust
mod mode {
    pub(super) struct Reliable;     // marker
    pub(super) struct Unreliable;   // marker
}

trait Mode<T>: Sized {
    type Overflow: Overflow<T>;
    type Feedback;

    fn handle(overflow: &mut Self::Overflow, message: T) -> bool;
    fn ready_feedback(feedback: Feedback) -> Self::Feedback;
    fn overflow_feedback(handled: bool) -> Self::Feedback;
    fn is_backoff(feedback: &Self::Feedback) -> bool;
    fn is_closed(feedback: &Self::Feedback) -> bool;
}

// Reliable: always handles, never rejects
impl<T: Policy> Mode<T> for mode::Reliable {
    fn handle(overflow: &mut Self::Overflow, message: T) -> bool {
        T::handle(overflow, message);
        true
    }

    fn overflow_feedback(_handled: bool) -> Self::Feedback {
        Feedback::Backoff
    }

    fn is_backoff(feedback: &Self::Feedback) -> bool {
        *feedback == Feedback::Backoff
    }
}

// Unreliable: may reject
impl<T: UnreliablePolicy> Mode<T> for mode::Unreliable {
    fn handle(overflow: &mut Self::Overflow, message: T) -> bool {
        T::handle(overflow, message)
    }

    fn overflow_feedback(handled: bool) -> Self::Feedback {
        Unreliable::Outcome(if handled { Feedback::Backoff } else { Feedback::Closed })
    }

    fn is_backoff(feedback: &Self::Feedback) -> bool {
        matches!(feedback, Unreliable::Outcome(Feedback::Backoff))
    }

    fn is_closed(feedback: &Self::Feedback) -> bool {
        matches!(feedback, Unreliable::Outcome(Feedback::Closed))
    }
}
```

## Appendix C — Custom Overflow implementations

The default `Overflow<T>` is `VecDeque<T>` (chapter 15 main text). You
can implement your own:

```rust
struct LatestOnlyOverflow<T> {
    latest: Option<T>,
}

impl<T> Overflow<T> for LatestOnlyOverflow<T> {
    fn is_empty(&self) -> bool {
        self.latest.is_none()
    }

    fn drain<F>(&mut self, mut push: F)
    where F: FnMut(T) -> Option<T>
    {
        if let Some(msg) = self.latest.take() {
            if let Some(returned) = push(msg) {
                // Receiver refused; put back
                self.latest = Some(returned);
            }
        }
    }
}
```

This overflow keeps only the latest message. Older messages are
discarded. Useful when newer supersedes older (e.g., state updates).

```rust
struct CoalescingOverflow<T, F> {
    items: VecDeque<T>,
    coalesce: F,  // closure that merges two items
}
```

This overflow coalesces incoming messages with existing ones. Useful
when multiple identical requests can be deduplicated.

## Appendix D — The `actor!` macro

`commonware-macros` provides an `actor!` macro for spawning actors with
boilerplate reduction:

```rust
actor! {
    pub struct MyActor {
        // fields
        config: Config,
        sender: Sender,
        receiver: Receiver,
        // ...
    }

    // The message handling logic
    async fn handle(msg: Message) -> Result<(), Error> {
        match msg {
            Message::Foo(x) => { ... }
            Message::Bar(y) => { ... }
        }
        Ok(())
    }
}
```

The macro generates:
- The actor struct with all your fields.
- A `start()` method that spawns the actor as a supervised task.
- A mailbox.
- An `handle()` function for messages.

Used extensively in Simplex, Marshal, and other consensus primitives
to avoid repetitive actor scaffolding.

## Appendix E — Supervision internals

`runtime/src/utils/supervision/`. The supervision tree:

```rust
pub struct Tree {
    name: Name,             // path of labels
    metrics: Arc<Metrics>,
    parent: Option<Arc<Tree>>,
    children: Mutex<Vec<Arc<Tree>>>,
}
```

Each task in the runtime has a `Context` that holds an `Arc<Tree>`
representing its position in the supervision tree.

When a task panics:

1. The panic is caught by the runtime's panic handler.
2. The runtime aborts the panicking task.
3. The runtime aborts all descendants (children of the panicking task).
4. The runtime notifies the parent of the abort (so it can decide
   whether to spawn a replacement).

The `Tree` is **observability**: you can walk it to see "what tasks are
alive and how are they organized." Useful for debugging.

## Appendix F — The shutdown coordination

```rust
pub struct Stopper {
    inner: Arc<StopperInner>,
}

struct StopperInner {
    value: Mutex<Option<i32>>,
    waiters: Condvar,
}

impl Stopper {
    pub fn stop(&self, value: i32, timeout: Option<Duration>) -> impl Future<Output = Result<(), Error>> {
        async move {
            let mut guard = self.inner.value.lock();
            if guard.is_some() {
                return Err(Error::Exited);  // Already stopped
            }
            *guard = Some(value);
            self.inner.waiters.notify_all();
            // ... wait for all `Signal` references to drop ...
        }
    }

    pub fn stopped(&self) -> Signal {
        Signal::new(self.inner.clone())
    }
}
```

`Stopper::stop(value)` initiates shutdown. `Signal` resolves when
shutdown is initiated. Tasks hold `Signal` references; when they all
drop, shutdown completes.

## Appendix G — The `Handle<T>` returned by spawn

```rust
pub struct Handle<T> {
    inner: JoinHandle<T>,
}

impl<T> Handle<T> {
    pub fn abort(&self) {
        self.inner.abort();
    }

    pub async fn join(self) -> Result<T, Error> {
        self.inner.await
    }
}
```

Spawn returns a `Handle<T>` you can use to:
- `abort()` — kill the task immediately (and its descendants).
- `await` — wait for completion.

## Appendix H — The metrics emitted by the mailbox

For each mailbox:

- `mailbox_ready_count` — current size of ready queue.
- `mailbox_overflow_count` — current size of overflow.
- `mailbox_send_total` — total sends (cumulative).
- `mailbox_send_backoff_total` — sends that went to overflow.
- `mailbox_send_rejected_total` — sends that were rejected (unreliable only).
- `mailbox_recv_total` — total receives.
- `mailbox_recv_closed_total` — receives after close.

These let you monitor backpressure (`send_backoff_total` climbing) and
detect overload (overflow growing).

## Appendix I — Test patterns

```rust
fn test_mailbox_basic() {
    let (mailbox, mut receiver) = mailbox::mailbox(NonZeroUsize::new(10).unwrap());
    mailbox.send(1);   // Ok
    mailbox.send(2);   // Ok
    // ...
    assert_eq!(receiver.recv().await, Some(1));
}

fn test_mailbox_overflow() {
    let (mailbox, mut receiver) = mailbox::mailbox(NonZeroUsize::new(2).unwrap());
    mailbox.send(1);   // Ok
    mailbox.send(2);   // Ok
    mailbox.send(3);   // Backoff (overflow)
    // Receiver drains; eventually gets all 3
}

fn test_mailbox_unreliable_rejects() {
    let (mailbox, mut receiver) = mailbox::unreliable_mailbox(NonZeroUsize::new(2).unwrap());
    // ... with an UnreliablePolicy that returns false for overflow ...
    let fb = mailbox.send(3);
    assert!(matches!(fb, Unreliable::Outcome(Feedback::Closed)));
}

fn test_mailbox_closed() {
    let (mailbox, mut receiver) = mailbox::mailbox(NonZeroUsize::new(10).unwrap());
    drop(mailbox);
    assert_eq!(receiver.recv().await, None);
}

fn test_supervision_kills_children() {
    let parent_handle = ctx.spawn(|ctx| async move {
        let child_handle = ctx.child("worker").spawn(|_| async move {
            loop { ctx.sleep(Duration::from_secs(1)).await; }
        });
        child_handle  // returned but kept alive
    });
    parent_handle.abort();
    // Child should be aborted too
}
```

## Appendix J — Common patterns in production

### The "send before await" anti-pattern

```rust
// BAD — sends while still holding state
for x in items {
    mailbox.send(x);
    self.state.update(x);  // blocks sender if mailbox is full
}

// GOOD — drop state before sending
let new_state = self.state.compute(items);
for x in items {
    mailbox.send(x);
}
self.state = new_state;
```

The mailbox's send may block (in synchronous mode) or backpressure
(async mode). Don't hold locks or references that the receiver might
need.

### The "mailbox for control plane, channel for data plane" pattern

Some apps use mailboxes for **control messages** (small, frequent,
must-not-drop) and direct channels (mpsc, oneshot) for **data
messages** (large, can be dropped).

```rust
let (control_mailbox, mut control_receiver) = mailbox::mailbox(...);
let (data_tx, data_rx) = mpsc::channel(...);

control_mailbox.send(ControlMsg::Start);
data_tx.send(DataMsg::BigPayload(...));  // never blocks control
```

This keeps the critical-path control messages flowing even if data
messages are backed up.

### The "spawn per topic" pattern

```rust
for topic in topics {
    let mailbox = topic_mailboxes[topic].clone();
    ctx.child(topic).spawn(move |ctx| async move {
        loop {
            let msg = mailbox.recv().await;
            process(msg);
        }
    });
}
```

One mailbox per topic, one task per topic. Easy to reason about,
easy to monitor, easy to scale.

## Appendix K — Common gotchas

### Re-entering the mailbox from a policy

`mailbox.rs:106-110` warns explicitly. If your `Policy::handle` calls
`mailbox.send(...)` on the **same** mailbox, you deadlock instantly.

If you need to send to another component from `handle`, use a
**different** mailbox or a oneshot channel.

### Holding overflow lock during expensive work

The overflow lock is held during `handle`. If your `handle` does
expensive work (network I/O, file I/O), all other senders to this
mailbox are blocked.

Keep `handle` fast. Defer expensive work to a separate task.

### Using `Mailbox::send` from a sync context

```rust
// BAD — blocks the executor
let result = mailbox.send_sync(msg);  // doesn't exist, but if it did...

// GOOD — async
mailbox.send(msg);  // non-blocking
```

The `Mailbox::send` API is async-aware. Don't try to make it sync.

## Where to look in the code (expanded)

- `actor/src/lib.rs` — the `Feedback`, `Unreliable` types.
- `actor/src/mailbox.rs:1-2280` — the full mailbox implementation.
- `actor/src/mailbox.rs:91-140` — the policy traits.
- `actor/src/mailbox.rs:142-200` — the mode machinery.
- `actor/src/mailbox.rs:201-500` — the Mailbox struct and send/recv paths.
- `runtime/src/lib.rs:152-356` — Spawner, Supervisor, supervision.
- `runtime/src/utils/supervision/` — the supervision tree.
- `runtime/src/utils/signal/` — the Stopper.
- `commonware-macros/src/actor.rs` — the `actor!` macro.

## If you only remember three things

1. **Bounded ready + unbounded overflow.** Bursts get absorbed; sustained overload is visible (overflow grows).
2. **`Policy` is reliable (always handle), `UnreliablePolicy` may reject.** Choose based on whether dropping messages is a correctness violation.
3. **Supervision kills descendants when a parent dies.** Structural deadlock prevention — no task waits forever on a dead peer.

→ Next: **Chapter 16 — Advanced Storage** (QMDB, MMR, MMB). The authenticated
data structures that make verifiable state possible.