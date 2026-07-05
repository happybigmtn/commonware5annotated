# Chapter 02 — The Abstract Runtime

> How Commonware makes distributed systems deterministically testable.

## The problem

Distributed systems are notoriously hard to test.

Why? Because every test has to wrangle **time**, **scheduling**, **randomness**, and
the **network**. Real time is non-deterministic. The OS scheduler might pick tasks
in any order. RNG draws aren't reproducible. Network messages drop, reorder, arrive
at random.

If you run the same test twice, you can get different results. Bugs hide in the
gaps. You write a test, it passes, you ship, it explodes in production at 3am.

## The trick

Don't fight the chaos. **Replace it.**

Take the entire "outside world" — time, scheduling, network, even storage — and
make it an **interface** (a trait). Then implement that interface **twice**. Once
with real production code (Tokio). Once with a simulator that you completely
control.

Now your code is written against the interface. In production, it talks to Tokio.
In tests, it talks to the simulator. **Same code, both worlds.** And in tests,
you can replay the same scenario a thousand times with different seeds, get
different interleavings, and any time something breaks it's reproducible down to
the SHA-256 of the runtime state.

That's the whole trick.

## The trait surface

Look at `runtime/src/lib.rs`. The whole file is basically a list of traits. Here
are the big ones:

```rust
pub trait Runner {                                  // drives execution
    fn start<F, Fut>(self, f: F) -> Fut::Output
    where F: FnOnce(Self::Context) -> Fut, Fut: Future;
}

pub trait Spawner: Supervisor {                    // task spawning
    fn spawn<F, Fut, T>(self, f: F) -> Handle<T>;
    fn shared(self, blocking: bool) -> Self;       // hints for scheduler
    fn dedicated(self) -> Self;                    // pin to a thread
    fn stop(self, value: i32, timeout: Option<Duration>) -> ...;
}

pub trait Supervisor: Send + Sync + 'static {       // task hierarchy
    fn child(&self, label: &'static str) -> Self;   // mandatory supervision
    fn with_attribute(self, key, value) -> Self;   // dynamic attrs (epoch, peer)
}

pub trait Clock {                                   // virtualized time
    fn current(&self) -> SystemTime;
    fn sleep(&self, duration: Duration) -> ...;
    fn sleep_until(&self, deadline: SystemTime) -> ...;
    fn timeout<F, T>(&self, duration, future) -> ...;  // with auto-cancel
}

pub trait Network {                                 // abstract network
    type Listener: Listener;
    fn bind(&self, socket: SocketAddr) -> ...;
    fn dial(&self, socket: SocketAddr) -> ...;
}

pub trait Storage {                                 // persistent blobs
    type Blob: Blob;
    fn open_versioned(...) -> ...;                 // versioned for migrations
    fn remove(...) -> ...;
    fn scan(...) -> ...;
}

pub trait Metrics: Supervisor {                     // Prometheus metrics
    fn register(...) -> Registered<M>;
    fn encode(&self) -> String;                     // dump all metrics
}
```

Notice the layering: `Spawner: Supervisor`. You can't spawn without inheriting
the supervision tree. Every task has a parent. Parents killing themselves kills
their children. That's how they keep deadlocks and orphan tasks under control.

The `Metrics: Supervisor` is clever too — registered metrics get prefixed with
the current supervision tree path. So `orchestrator.engine.votes` is automatically
the metric for "votes inside orchestrator → engine." You don't have to manually
namespace anything.

## How `async`/`.await` actually works (Rust internals)

Before we go deeper into the runtime, you need to understand what `async` and
`.await` do to your code. This is intermediate-to-advanced Rust and is the
reason Commonware's design is possible.

### The state-machine transformation

When you write:

```rust
async fn fetch_and_verify(url: String) -> Result<Bytes, Error> {
    let response = http_get(url).await?;       // suspension point
    let body = response.body().await?;          // suspension point
    verify_signature(&body)?;                    // synchronous
    Ok(body)
}
```

The Rust compiler transforms this into a state machine — without you seeing
it. The generated code looks roughly like:

```rust
enum FetchAndVerify {
    Start { url: String },
    WaitingOnHttp { url: String, fut: HttpGetFuture },
    WaitingOnBody { fut: BodyFuture, response: Response },
    Verifying { body: Bytes },
    Done,
}

impl Future for FetchAndVerify {
    type Output = Result<Bytes, Error>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
        loop {
            match &mut *self {
                FetchAndVerify::Start { url } => {
                    let url = std::mem::take(url);
                    let fut = http_get(url);
                    *self = FetchAndVerify::WaitingOnHttp { url, fut };
                }
                FetchAndVerify::WaitingOnHttp { fut, .. } => {
                    match fut.poll(cx) {
                        Poll::Ready(Ok(response)) => {
                            let fut = response.body();
                            *self = FetchAndVerify::WaitingOnBody { fut, response };
                        }
                        Poll::Ready(Err(e)) => return Poll::Ready(Err(e.into())),
                        Poll::Pending => return Poll::Pending,
                    }
                }
                FetchAndVerify::WaitingOnBody { fut, .. } => {
                    match fut.poll(cx) {
                        Poll::Ready(Ok(body)) => {
                            if let Err(e) = verify_signature(&body) {
                                return Poll::Ready(Err(e));
                            }
                            *self = FetchAndVerify::Verifying { body };
                        }
                        Poll::Ready(Err(e)) => return Poll::Ready(Err(e.into())),
                        Poll::Pending => return Poll::Pending,
                    }
                }
                FetchAndVerify::Verifying { body } => {
                    let body = std::mem::take(body);
                    *self = FetchAndVerify::Done;
                    return Poll::Ready(Ok(body));
                }
                FetchAndVerify::Done => panic!("polled after completion"),
            }
        }
    }
}
```

The compiler turns your `async fn` into an `enum` with one variant per
suspension point, plus a `Future::poll` impl that walks the state machine.
Each `.await` becomes a `match` arm that either advances to the next state
or returns `Pending`.

### What `Pin` is for

The state machine above contains self-referential fields in some cases
(when a future holds a reference to data owned by another field of itself).
The compiler can generate those for things like `select!` macros or futures
that borrow from their own state.

Self-referential types can't be safely moved. If you move a struct, all
its pointers (including self-pointers) become dangling. `Pin<P>` is a
wrapper that prevents moving: `Pin<Box<T>>` puts the future on the heap
and `Pin<&mut T>` is a borrow that says "you can't move out of this."

The auto-trait `Unpin` lets types opt out — most types are `Unpin` because
they don't have internal references. The compiler auto-implements `Unpin`
for all "normal" types. Futures that contain self-references are `!Unpin`.

Why you care: when you spawn a future with `tokio::spawn`, it has to be
`Send + 'static` and pinned to the heap (via `Box::pin`). The runtime
needs the future to stay in one place for the entire duration of its
execution — it can't be moved around while suspended.

### Tokio's executor architecture

Tokio is a **multi-threaded, work-stealing** scheduler. Here's how it works:

1. **Worker threads**: by default, one worker per CPU core.
2. **Task queue**: each worker has a local queue. When a worker has nothing to
   do, it tries to **steal** tasks from other workers.
3. **Reactor**: a single thread (or pool) that polls OS-level event sources
   (epoll on Linux, kqueue on macOS, IOCP on Windows). When a file descriptor
   becomes ready, the reactor wakes the task that's waiting on it.
4. **Time driver**: a thread that manages timer wheels. When a `sleep()`
   expires, the time driver wakes the waiting task.

The four major components:

| Component | Job |
|---|---|
| **Worker threads** | Run tasks. Poll them. When a task yields (`Pending`), put it on the local queue and poll something else. |
| **Reactor** | Translate OS-level I/O readiness (epoll/kqueue/IOCP) into Rust task wakeups. |
| **Time driver** | Manage timers. Wake tasks whose `sleep()` has expired. |
| **Blocking pool** | A separate thread pool for `spawn_blocking` — runs CPU-heavy or blocking work that would starve the worker threads. |

### The waker protocol

When a task is suspended, it returns `Poll::Pending` and (usually) registers
its `Waker` with whatever it was waiting on. When the waker is called, the
runtime knows to put the task back in the queue.

The standard pattern is:

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
    if self.data_is_ready() {
        return Poll::Ready(self.take_data());
    }
    // Register our waker so the runtime knows to wake us.
    self.set_waker(cx.waker().clone());
    Poll::Pending
}
```

The `Waker` is `Send + Sync + Clone` so it can be sent to whatever thread
will eventually signal readiness. Commonware's deterministic runtime uses a
`NoopWaker` because everything happens in one thread.

### Tokio vs async-std vs smol

Commonware chose Tokio. Here's why, and what the alternatives look like:

| Runtime | Concurrency | I/O | Maturity |
|---|---|---|---|
| **Tokio** | Multi-threaded work-stealing. | epoll / kqueue / IOCP / io_uring. | Production-grade. Used by Discord, AWS, Cloudflare, Linkerd. |
| **async-std** | Multi-threaded work-stealing (similar to Tokio). | Same OS primitives. | Less actively maintained than Tokio. |
| **smol** | Single-threaded by default. | Same OS primitives. | Smaller, simpler. Used in some GUI frameworks. |
| **glommio** | Thread-per-core (no work-stealing). | io_uring. | Specialized for high-throughput I/O. |
| **monoio** | Thread-per-core. | io_uring. | Used by Cloudflare Workers. |

Tokio wins on ecosystem (every Rust library supports it), io_uring
integration (`tokio-uring`), and battle-testing. Commonware's choice lets
them use `tokio-uring` for high-performance storage and network on Linux.

### Loom — permutation testing for concurrency

The `loom` crate (`runtime/src/lib.rs` uses it) is a permutation tester for
concurrent code. It's the concurrency analog of the deterministic runtime:
same code, but explores every possible interleaving.

```rust
#[test]
fn test_concurrent_increment() {
    loom::model(|| {
        let counter = Arc::new(AtomicUsize::new(0));
        let c1 = counter.clone();
        let c2 = counter.clone();

        let t1 = loom::thread::spawn(move || c1.fetch_add(1, Ordering::SeqCst));
        let t2 = loom::thread::spawn(move || c2.fetch_add(1, Ordering::SeqCst));

        t1.join().unwrap();
        t2.join().unwrap();

        assert_eq!(counter.load(Ordering::SeqCst), 2);
    });
}
```

Loom explores all 2 possible interleavings of the two `fetch_add` operations.
For more complex tests, it can explore thousands.

Commonware uses loom in the actor crate to test the mailbox's concurrent
enqueue/dequeue paths. A real test with 4 concurrent senders and 1 receiver
might have hundreds of interleavings; loom checks them all.

### Cooperative vs preemptive scheduling

| Aspect | Cooperative (Tokio, async-std) | Preemptive (Go goroutines, Java threads) |
|---|---|---|
| **Task switch** | Only when task yields (`.await`, `Pending`). | OS / runtime can switch at any instruction. |
| **Predictability** | High — you know when a task might pause. | Low — a task can be paused anywhere. |
| **Latency for hot tasks** | Other tasks can starve if a task never yields. | OS scheduler enforces fairness. |
| **Implementation** | Compiler-transformed state machines + wakers. | OS thread + context switch. |
| **Memory** | ~100 bytes per task (state machine). | ~8 KB per thread (default stack). |

Commonware is cooperative. That's why you see `context.sleep()` and explicit
yields — without them, a hot loop could starve other tasks. The deterministic
runtime enforces this: if no task is `Ready` and no task is sleeping, the
runtime panics with "runtime stalled."

### Concurrent vs parallel

These two words are often used interchangeably. They mean different things:

- **Concurrent**: tasks make progress in overlapping time periods, but not
  necessarily at the same instant. (One task runs for a bit, then another.)
- **Parallel**: tasks make progress at the same instant, on different CPU
  cores.

A single-threaded async runtime is concurrent but not parallel. A
multi-threaded runtime can be both.

Commonware code can be either, depending on the runtime config:
`tokio::Runtime::new()` with default settings uses all cores (parallel).
You can configure single-threaded with `tokio::runtime::Builder::new_current_thread()`.

The `commonware_parallel` crate gives you a `Strategy` trait to control
parallelism explicitly:

```rust
pub trait Strategy {
    fn execute<F, R>(self, work: F) -> Result<Vec<R>, Error>
    where F: FnMut(Self::Item) -> R;
}

pub struct Sequential;          // no parallelism
pub struct Rayon { pool: ThreadPool };  // parallel via rayon
```

Simplex uses `strategy: Sequential` (no parallelism) or `strategy: Rayon(8)`
(8-way parallelism). For consensus, parallelism is mostly in batch
verification (chapter 11).

### Threads vs tasks vs futures

| Concept | Created by | Cost to create | Cost to switch | Used by |
|---|---|---|---|---|
| **OS thread** | `std::thread::spawn` | ~1 ms (mmap a stack) | ~1-10 us (syscall + register save) | Operating system |
| **Tokio task** | `tokio::spawn` | ~100 ns (heap alloc) | ~100 ns (waker) | Tokio runtime |
| **Future** | `async fn` | 0 ns (compile time) | 0 ns (no runtime) | Async I/O |

Commonware creates ~thousands of tasks per node (one per actor, one per
mailbox). That would be untenable with OS threads. With tasks, it's free.

### The async stack trace problem

One real downside of async: **stack traces are useless.** When a future is
suspended and resumed on a different worker thread, the original stack is
gone. The new stack starts at `poll`, with no trace of how the future was
created.

Tools that help:

- **`tokio-console`**: shows you which task is waiting on what.
- **`tracing` spans**: manually instrument your code so trace data shows
  the call chain.
- **`async-backtrace`**: a custom hack that captures spans in futures.

Commonware uses `tracing` (chapter 21 cross-references). Every actor loop
has a span; every cross-actor message carries its parent's span. This is
why the AGENTS.md is so emphatic about Span-as-field.

## The Auditor — a SHA-256 receipt for every execution

The key insight that makes this whole thing **trustworthy**: the deterministic
runtime has an `Auditor` (`runtime/src/deterministic.rs:144-183`):

```rust
pub struct Auditor {
    digest: Mutex<Digest>,   // 32 bytes, SHA-256
}

impl Auditor {
    pub(crate) fn event<F>(&self, label: &'static [u8], payload: F)
    where F: FnOnce(&mut Sha256) {
        let mut hasher = Sha256::new();
        hasher.update(self.digest.lock().as_ref());
        hasher.update(label);
        payload(&mut hasher);
        *self.digest.lock() = hasher.finalize().into();
    }

    pub fn state(&self) -> String { hex(&self.digest.lock()) }
}
```

Every meaningful event in the runtime — every `send_attempt`, `recv_success`,
`open`, `write_at`, `rand`, `process_task` — feeds its details into a rolling
SHA-256 hash. At the end of a test, `context.auditor().state()` gives you a hex
string.

If you run the same test twice with the same seed and get the same hex string,
**the executions were byte-for-byte identical** at the level of "what was done,
in what order, with what data." If the strings differ, you have non-determinism
somewhere.

This isn't just a debug aid. Look at the tests in `consensus/src/simplex/mod.rs`
from chapter 01 — they verify `state1 == state2` to assert determinism across
runs. That's the level of trust this gives you.

Look at `runtime/src/network/audited.rs:14-30` — even a simple `send()`:

```rust
async fn send(&mut self, bufs: impl Into<IoBufs> + Send) -> Result<(), Error> {
    let bufs = bufs.into();
    self.auditor.event(b"send_attempt", |hasher| {
        hasher.update(self.remote_addr.to_string().as_bytes());
        bufs.for_each_chunk(|chunk| hasher.update(chunk));
    });
    self.inner.send(bufs).await.inspect_err(|e| { ... })?;
    self.auditor.event(b"send_success", |hasher| { ... });
    Ok(())
}
```

The auditor records the remote address AND the actual bytes sent. So if your
code serializes a message differently based on some condition, the hash diverges
and you know.

## The deterministic executor — time as a virtualized quantity

`runtime/src/deterministic.rs:188-353`. The `Config` struct defines the
simulation:

```rust
pub struct Config {
    rng: BoxDynRng,                                    // randomness source
    cycle: Duration,                                   // how much time advances per iteration
    start_time: SystemTime,                            // virtual clock starts here
    timeout: Option<Duration>,                         // panic if test runs longer
    catch_panics: bool,                                // capture or propagate?
    storage_fault_cfg: FaultConfig,                    // inject random storage faults
    network_buffer_pool_cfg: BufferPoolConfig,         // buffer pool sizes
    storage_buffer_pool_cfg: BufferPoolConfig,
}
```

The defaults (`deterministic.rs:236-247`): seed 42, cycle 1ms, start at Unix
epoch, no timeout, no panics caught, no faults, default buffer pools. All
deterministic.

The executor (`deterministic.rs:355-440`) holds:

- An RNG (seeded) — every random choice goes through this
- A virtual clock — advanced by `cycle` each iteration
- A binary heap of sleeping tasks — sorted by deadline
- A `Stopper` — for clean shutdown coordination
- The `Auditor` — recording every event

**The event loop** is the heart of it. Each iteration:

1. **Advance time** by `cycle` (or jump to next sleep deadline if idle)
2. **Wake sleepers** whose deadlines have elapsed
3. **Pick a ready task at random** (RNG-sampled) and poll it once
4. **Assert liveness** — if no task is ready AND no sleeper is pending, panic

The random ordering matters. Every test run picks tasks differently. Every test
run is reproducible (same seed). So you get **diversity with reproducibility** —
exactly what fuzz-testing needs.

## Checkpoints — crash recovery testing

`runtime/src/deterministic.rs:442-463` defines `Checkpoint`. You can snapshot
the executor's state — RNG, time, storage, auditor — and resume from it. The
test pattern from `AGENTS.md`:

```rust
let mut checkpoint = None;
loop {
    let runner = if let Some(checkpoint) = checkpoint.take() {
        deterministic::Runner::from(checkpoint)
    } else {
        deterministic::Runner::timed(Duration::from_secs(30))
    };
    let (complete, next_checkpoint) = runner.start_and_recover(f);
    if complete { break; }
    checkpoint = Some(next_checkpoint);
}
```

This pattern runs your code, **drops the runner at some point to simulate
unclean shutdown**, restores from checkpoint, continues. Lets you verify that
recovery is correct: data is consistent, no lost messages, no double-broadcast,
etc.

## The deterministic network — just channels

`runtime/src/network/deterministic.rs` is a simple in-memory network:

```rust
pub struct Network {
    ephemeral: Arc<Mutex<u16>>,                            // next ephemeral port
    listeners: Arc<Mutex<HashMap<SocketAddr, Dialable>>>,  // bound sockets
}
```

`bind` registers a sender under a `SocketAddr`. `dial` looks up that sender,
creates two in-memory channels (one for each direction), and returns `Sink`/
`Stream` handles. No actual TCP. No actual sockets. Just mpsc channels wrapped
to look like network primitives.

That's why these tests are **fast** — milliseconds instead of seconds — and why
you can run thousands of them in CI.

## The simulated P2P network — where it gets serious

But the real magic is in `p2p/src/simulated/`. This isn't part of the `runtime`
crate; it sits **on top** of the runtime and gives you a faithful model of a
real network.

Look at `p2p/src/simulated/ingress.rs:134` — the `Link` struct:

```rust
pub struct Link {
    pub latency: Duration,
    pub jitter: Duration,
    pub success_rate: f64,
}
```

That's per-link latency, jitter, and packet loss. Not a constant — a sample on
each message. With latency=10ms and jitter=3ms, each message takes 7-13ms to
arrive.

Now look at `p2p/src/simulated/mod.rs:12-57` — the bandwidth model. It's
**really** faithful:

> 1. **Collect Active Flows**: gather every active transfer
> 2. **Compute Progressive Filling**: raise every flow's rate in lock-step
>    until a sender or receiver limit saturates
> 3. **Wait for the Next Event**: compute when the first flow finishes
> 4. **Deliver Message**: pass it to the receiver

This is **max-min fairness** — a real networking concept. If peer A's egress is
the bottleneck, every flow through A gets equal share. The simulator also
distinguishes **transmission delay** (bytes on the wire) from **latency**
(propagation). A 10KB message on a 10KB/s link takes 1 second to send,
regardless of latency.

**Why this matters:** consensus nodes can be bandwidth-bound. A 10MB block being
shipped from a leader to 100 validators eats 1GB of egress. If you don't model
that, your consensus test passes but production melts down.

## Dynamic network changes

The `Oracle` (`p2p/src/simulated/ingress.rs:150`) is the test's control plane:

```rust
oracle.add_link(pk1, pk2, Link { ... }).await;     // connect two peers
oracle.remove_link(pk1, pk2).await;                 // partition
oracle.update_link(pk1, pk2, Link { ... }).await;   // change characteristics
oracle.limit_bandwidth(pk, Some(egress), Some(ingress)).await;  // throttle
```

You can **partition** the network mid-test. You can simulate a degraded link.
You can flip a peer from "good" to "Byzantine" mid-consensus. The test pattern
from `AGENTS.md`:

```rust
fn separated(n: usize, a: usize, b: usize) -> bool {
    let m = n / 2;
    (a < m && b >= m) || (a >= m && b < m)   // split into two halves
}
link_validators(&mut oracle, &validators, Action::Unlink, Some(separated)).await;
```

This unlinks every connection that crosses the partition. Two islands of
validators, both still trying to make progress, neither one aware (yet) of the
partition. **Then** you can verify that consensus recovers when the partition
heals.

## The faulty storage — fault injection you can trust

`runtime/src/storage/faulty.rs` is a `Storage` wrapper that randomly returns
errors. The fault schedule is drawn from the shared RNG, so:

```rust
let cfg = deterministic::Config::new()
    .with_seed(42)
    .with_storage_fault_config(FaultConfig {
        read_failure_rate: 0.01,
        write_failure_rate: 0.001,
        ..Default::default()
    });
```

Run twice with seed 42 → same faults happen at the same moments. Now you can
test "what if my disk sometimes fails to write?" and have confidence the test
exercises the same paths every time.

## The full picture

Pull all of this together and you get:

```
                    ┌─────────────────────────────────────┐
                    │  Your protocol code                 │
                    │  (consensus::simplex, broadcast::,  │
                    │   storage::journal, etc.)           │
                    └────────────┬────────────────────────┘
                                 │ uses traits
                                 ▼
        ┌─────────────────────────────────────────────────┐
        │  Trait surface:                                │
        │    Spawner, Clock, Network, Storage,           │
        │    Metrics, Supervisor, BufferPooler, ...       │
        └────────────┬────────────────────┬───────────────┘
                     │                    │
        ┌────────────▼──────┐    ┌─────────▼────────────┐
        │ Production impl:  │    │ Test impl:           │
        │   tokio + io_uring│    │   deterministic.rs   │
        │   + real sockets  │    │   + AuditedNetwork   │
        │   + real disk     │    │   + MemStorage       │
        └───────────────────┘    │   + FaultyStorage    │
                                 └─────────┬────────────┘
                                           │ underneath
                                           ▼
                                 ┌─────────────────────┐
                                 │ Simulated p2p       │
                                 │   (latency, jitter,  │
                                 │    partitions, BW)  │
                                 └─────────────────────┘
```

The same line of code — say, `context.sleep(Duration::from_millis(100)).await` —
runs in production against the OS scheduler for 100ms, and in tests against the
deterministic cycle counter. Same code, both worlds.

**Why this matters for the 93% test coverage + 1500 benchmarks:** you can write
a consensus test that runs in 50ms but exercises 1000 distinct adversarial
scenarios (different seeds → different task orderings → different message
timing → different partition patterns). Every failure mode is reproducible.
Every test is a unit test, not an integration test against a flaky network.

## Where to look in the code

- `runtime/src/lib.rs:152-165` — the `Runner` trait.
- `runtime/src/lib.rs:262-356` — `Spawner` with the supervision model.
- `runtime/src/lib.rs:447-500` — `Clock` with `timeout` combinator.
- `runtime/src/lib.rs:513-606` — `Network`, `Sink`, `Stream`, `Listener`.
- `runtime/src/lib.rs:627-779` — `Storage` and `Blob` traits.
- `runtime/src/deterministic.rs:144-183` — the `Auditor`.
- `runtime/src/deterministic.rs:355-440` — the executor state.
- `runtime/src/network/audited.rs:1-100` — how every network event feeds the auditor.
- `runtime/src/storage/audited.rs:31-160` — how every storage event feeds it too.
- `p2p/src/simulated/mod.rs:12-57` — the bandwidth model.
- `p2p/src/simulated/ingress.rs:134-150` — `Link` and `Oracle`.

## Rust for distributed systems — the patterns that matter

Before we go deeper into the runtime internals, let's lay down the
Rust concepts that show up *constantly* in Commonware's source. If
you're already a Rust ace, skim. If you've written Rust but not
production async code, this is your reference.

### Ownership and borrowing — the heart of Rust

Every Rust value has exactly one *owner*. When the owner goes out of
scope, the value is dropped (its `Drop` runs, memory is freed). The
borrow checker enforces this statically — no garbage collector, no
runtime overhead, no manual `free`.

```rust
let v = Vec::new();      // v owns the Vec
let v2 = v;              // ownership MOVED to v2; v is invalid
// println!("{:?}", v);  // compile error: v is gone
v2.push(1);              // v2 owns the Vec, can mutate
```

Borrowing lets you share without moving:

```rust
let v = vec![1, 2, 3];
let r = &v;              // immutable borrow
println!("{:?}", r);     // r lives as long as needed
println!("{:?}", v);     // v still valid; immutable borrows can stack
```

The two flavors: `&T` (shared/immutable) and `&mut T` (exclusive).
The rules:
- You can have *any number* of `&T` at once, OR
- *Exactly one* `&mut T`, OR
- Neither.

```rust
let mut v = vec![1, 2, 3];
let r1 = &v;             // OK: shared borrow
let r2 = &v;             // OK: another shared borrow
// let m = &mut v;       // compile error: shared borrows live
println!("{:?}", r1);    // r1, r2 last used here
let m = &mut v;          // OK: shared borrows gone
m.push(4);               // mutate via exclusive borrow
```

**Why this matters for Commonware.** Consensus messages are
serialized bytes (`Vec<u8>`); they pass through channels, get
deserialized into typed structs, get verified, get signed. Every
step involves ownership transfer. If the borrow checker rejects your
code, it's usually because you're holding a borrow longer than you
should or trying to mutate through a shared reference.

The `Automaton` trait in `consensus/src/lib.rs:25-79` is a great
example. `verify` takes `&self` (immutable view of the application);
`propose` takes `&mut Context` (mutable access to spawn children,
register metrics); `certify` takes shared references. The borrow
checker enforces that you can't simultaneously verify and certify a
block with overlapping state — they must be sequential.

### Lifetimes — how long references live

A lifetime `'a` is the scope during which a reference is valid.
Most lifetimes are inferred (lifetime elision); when they're not, the
compiler asks you to write them.

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

The signature says: "the returned reference lives as long as the
shorter of `x` and `y`."

**Static lifetimes.** `'static` means "lives for the entire program
duration." String literals are `'static`. `&'static str` is a string
slice that the program can use forever. Commonware uses this heavily
for labels, span names, error messages — anything that's a constant.

**Why this matters for Commonware.** Tasks spawned with `tokio::spawn`
must be `'static` (the task could outlive the function that spawned
it). The runtime abstraction deals with this: `Context::spawn` takes
a closure that returns a `Future + 'static`. Most Commonware code
finesses this with `Arc<...>` (atomic reference counting) to give
spawned tasks their own owned copies.

### `Send` and `Sync` — concurrency's foundation

Two *auto traits* that govern concurrency:

- **`Send`**: a value can be *moved* to another thread.
- **`Sync`**: a value can be *shared* (via `&T`) across threads.
  Equivalently: `&T: Send`.

The compiler auto-derives `Send` and `Sync` based on the type's
fields. A type is `Send` iff all its fields are `Send`. A type is
`Sync` iff all its fields are `Sync` (and similarly for `&T`).

```rust
let v = vec![1, 2, 3];
std::thread::spawn(move || { println!("{:?}", v); });  // Vec<i32>: Send

let s = Rc::new(5);                                     // Rc<i32>: !Send
// std::thread::spawn(move || { println!("{}", s); });  // compile error
```

`Rc` (reference counting) is `!Send` because it doesn't synchronize
the count atomically. Use `Arc` (atomic Rc) for cross-thread sharing.

**Why this matters for Commonware.** Async runtimes (Tokio) move
tasks between worker threads. Tasks that hold non-`Send` data can't
be moved — they're stuck on the thread that created them. Commonware
cares about this because: (a) Tokio tasks spawned with `tokio::spawn`
require `Send + 'static` futures; (b) channels between actors are
`Send`-bounded by what flows through them; (c) the deterministic
runtime doesn't need `Send` because it's single-threaded.

The `commonware_runtime::Spawner` trait abstracts over this — the
production Tokio backend requires `Send`, the deterministic backend
doesn't. Same code, two regimes.

### `async`/`await` basics — cooperative concurrency

`async` turns a function into a state machine. `.await` is a
suspension point. We've seen this in detail in the main text; here's
the brief version:

```rust
async fn fetch_user(id: u64) -> Result<User, Error> {
    let response = http_get(format!("/users/{}", id)).await?;
    let user: User = serde_json::from_slice(&response.bytes().await?)?;
    Ok(user)
}
```

Three `.await` calls — three potential suspension points. While one
task is waiting on `http_get`, another task can run on the same
thread.

The key invariant: `await` points are *the only* places where the
runtime can switch tasks. If you have a synchronous loop that takes
100ms, it blocks the entire thread for 100ms. Commonware's
`context.sleep` and explicit `yield_now` are how you avoid this.

### Generics vs trait objects

Rust has two flavors of polymorphism:

**Generics** (`<T: Trait>`): monomorphized at compile time. Each
concrete type gets its own copy of the code. Zero runtime cost, but
bigger binary size.

```rust
fn process<T: Encode>(item: T) -> Vec<u8> {
    let mut buf = Vec::new();
    item.encode(&mut buf);
    buf
}
```

**Trait objects** (`Box<dyn Trait>`): a fat pointer (data pointer +
vtable pointer). Runtime dispatch via vtable lookup. Smaller binary,
small runtime cost.

```rust
fn process(item: Box<dyn Encode>) -> Vec<u8> {
    let mut buf = Vec::new();
    item.encode(&mut buf);
    buf
}
```

**Commonware's choice.** Hot-path code uses generics (`Scheme`,
`Digest`, `PublicKey` are all generic over the signature scheme). The
performance-critical paths can't afford vtable lookups. Cold paths
and configuration use trait objects (e.g., `Box<dyn Network>`).

### The newtype pattern

`struct Meters(f64);` — a tuple struct with one field. Looks like a
type alias (`type Meters = f64;`) but isn't: the newtype is a
*distinct* type, so you can't accidentally pass `Meters` where
`Seconds` is expected.

Commonware uses newtypes for:
- **Type-safe IDs**: `ValidatorIndex(u32)`, `Epoch(u64)`, `Height(u64)`.
- **Domain separation**: `Namespace(&'static [u8])` so different
  protocols can use the same hash function without collision.
- **Units**: separate `Bytes` from `Messages` from `Blocks`.

### `PhantomData` — telling the compiler about unused type parameters

Sometimes you have a generic parameter you don't actually store:

```rust
struct TaggedHandle<T> {
    inner: SomeNonGenericType,
    _marker: PhantomData<T>,
}
```

`PhantomData<T>` tells the compiler "treat this as if it owned a
`T`" — which affects `Send`/`Sync` derivation, variance, and
auto-trait inference. Without `PhantomData<T>`, the compiler infers
the wrong variance and the wrong `Send`/`Sync` status.

Commonware uses `PhantomData` for:
- Type-level tagging (e.g., `Certificate<S>` tagged with the scheme
  type even though it doesn't store an `S`).
- Variance control (e.g., making a struct invariant in `T` to
  prevent lifetime issues).

### Why these patterns matter for Commonware

Distributed systems code is full of invariants:
- "this signature is over THIS exact message"
- "this vote is for view V"
- "this certificate comes from 2f+1 distinct signers"
- "this task owns its context, no one else mutates it"

Rust's type system enforces all of these at compile time. The
borrow checker enforces ownership flow. `Send`/`Sync` enforces
thread-safety boundaries. Newtypes prevent accidental swaps.
`PhantomData` controls variance to prevent subtle lifetime bugs.

The cost: a learning curve. The benefit: distributed-systems bugs
that would take weeks to track down in C++ or Go become compile errors.

## The full async/await state-machine transformation

We sketched this in the main text; let's go deeper. The Rust
compiler's *MIR* (mid-level intermediate representation) is where
the transformation happens. The high-level steps:

**Step 1: Identify suspension points.** Each `.await` is a potential
suspension point. The compiler inserts bookkeeping (state variable,
local-variable storage) at each one.

**Step 2: Generate an enum.** For an `async fn` with three `.await`
points, the compiler generates an enum with 5 variants (initial +
3 awaiting + complete).

**Step 3: Generate the `poll` method.** A loop that matches on the
current state, advances to the next, returns `Pending` when blocked
on a future, returns `Ready` when done.

**Step 4: Track captured state.** Local variables that are alive
across suspension points become fields of the enum. The compiler
figures out which variables are "live across" which await points
via liveness analysis on MIR.

### A concrete example

Consider:

```rust
async fn fetch_and_count(url: String, count: Arc<AtomicUsize>) -> Result<usize, Error> {
    let response = http_get(url).await?;
    let body = response.body().await?;
    let parsed: Vec<i32> = serde_json::from_slice(&body)?;
    let total: i32 = parsed.iter().sum();
    count.fetch_add(total as usize, Ordering::Relaxed);
    Ok(parsed.len())
}
```

The compiler analyzes which locals are live across each await:
- `url` is consumed by `http_get(url).await` → only live in state 0.
- `response` is live from first await completion to second await start
  → lives in state 2.
- `body` is live from second await completion to `serde_json` call
  → lives in state 3.
- `count` is borrowed for the entire function → lives in all states.
- `parsed`, `total` are computed after all awaits → live in state 4
  only.

The generated enum (approximately):

```rust
enum FetchAndCount {
    Start { url: String, count: Arc<AtomicUsize> },
    WaitingOnHttp {
        url: String,
        count: Arc<AtomicUsize>,
        fut: HttpGetFuture,
    },
    WaitingOnBody {
        count: Arc<AtomicUsize>,
        fut: BodyFuture,
        response: Response,
    },
    WaitingOnParse {
        count: Arc<AtomicUsize>,
        body: Bytes,
    },
    Done,
}
```

Note: `url` is moved into `http_get` but only in state 0; once the
future is constructed, `url` is gone. `response` only exists after
the first future resolves. The compiler is careful about *when* each
local is "alive."

### Why `Pin` exists

`Pin<P>` is a wrapper that prevents moving the wrapped value. It's
needed because the generated state-machine enum can be
*self-referential*: a variant might hold a future that borrows from
another field of the same enum.

Example: `select!` macro generates code that polls multiple futures
while keeping references to all of them. If the enum is moved,
those references become dangling.

```rust
let mut state = MyFuture::Start { ... };
let pinned: Pin<&mut MyFuture> = Pin::new(&mut state);
// pinned's fields can't be moved out, even though &mut MyFuture could
// theoretically be moved
```

The auto-trait `Unpin` lets types opt out of pinning. Most types
are `Unpin` because they don't have internal references. The
generated async state machine is `!Unpin` if any of its variants
contains a self-reference.

**Tokio's spawn requires `Unpin` or heap-allocation.** When you call
`tokio::spawn(future)`, the future must be `Send + 'static`. The
runtime pins it via `Box::pin` (heap allocation, then pinning).
The future lives on the heap for its entire lifetime, never moved.

### Stack usage and async

Each `async fn` invocation creates an instance of the generated
enum. The size of that enum is the sum of its largest variant's
fields. For a function with three large local variables, that's
a sizable allocation. For deeply nested async call chains, the
total memory can be substantial.

Commonware mitigates this by:
- Boxing large internal state (`Box<dyn Future>` for some hot paths).
- Using `Pin<Box<...>>` explicitly when futures need to be self-referential.
- Keeping async functions short and focused (each function adds one
  layer to the state machine).

### The compiler optimizations

Modern Rust compilers do significant optimization on async state
machines:
- **State inlining**: if a state is unreachable, the variant is
  eliminated.
- **Variant merging**: two states with identical fields are merged.
- **Tail-call optimization**: if a state ends by immediately calling
  another future, the states are collapsed.
- **Drop elision**: if a variant's fields are all `Drop`, the
  compiler may optimize the drop path.

The result: hand-written state machines are rarely faster than
`async`/`await`. The compiler does a better job than humans.

## The full Tokio architecture

Tokio is a multi-component async runtime. Five major pieces:

### 1. The reactor — translating OS events to Rust tasks

The reactor is the bridge between the OS's I/O event system and
Rust's task waker system. On Linux, it uses `epoll`. On macOS/BSD,
`kqueue`. On Windows, IOCP. On Linux with the `io-uring` feature,
Tokio can use `io_uring` for true async I/O.

**How epoll works.** `epoll_wait` blocks until one or more file
descriptors become ready (readable, writable, or have errors). The
reactor calls `epoll_wait`, gets back a list of ready FDs, and wakes
the tasks that registered interest in those FDs.

**The Tokio reactor's structure:**
- A single OS thread (or thread pool) dedicated to `epoll_wait`.
- A registry mapping FD → list of wakers.
- When an FD becomes ready, walk the waker list and wake each one.

The reactor integrates with the scheduler: when a waker is called,
the corresponding task is pushed to the worker's queue.

### 2. The scheduler — work-stealing multi-thread

Tokio's default scheduler is a work-stealing multi-thread scheduler
(see `tokio/src/runtime/scheduler/multi_thread/`).

**Structure per worker:**
- A local run queue (LIFO slot for the most recently woken task).
- A local queue (FIFO for older tasks).
- A handle to the global injector queue.
- A handle to other workers' queues (for stealing).

**The flow:**
1. Worker pops from local queue.
2. If empty, tries local LIFO slot.
3. If still empty, checks global injector.
4. If still empty, half-steals from another worker's queue.
5. If still empty, parks (sleeps) until a waker fires.

**Stealing mechanics.** Each worker has a fixed-size local queue
(typically 256 slots). When a worker runs out of work, it
randomly picks another worker and tries to steal half its queue.
This amortizes load across cores while keeping cache locality
(common tasks stay on one worker).

**Yield mechanisms.**
- `task::yield_now()` — re-pushes the current task to the back of
  the local queue, returns `Pending`.
- `task::consume_budget()` — checks if the task has exceeded its
  128-iteration budget; if so, yields automatically.

The "task budget" prevents a single infinite-loop task from starving
all others. Every 128 polls, the task is forcibly yielded.

### 3. The time driver — hierarchical timing wheel

The time driver manages `tokio::time::sleep` and other time-based
operations. Internally, it's a **hierarchical timing wheel**: a
multi-level array of "slots," each slot representing a time range.

**Why a wheel?** Insertion and removal of timers are O(1) per
operation. A heap-based priority queue would be O(log n). For a
runtime that handles millions of timers per second, O(1) matters.

**Hierarchical wheels:** the lowest level has fine granularity
(milliseconds); higher levels have coarser granularity (seconds,
minutes). A timer "fires" by walking the wheels.

The time driver integrates with the reactor: when a timer fires, it
wakes the corresponding task. This means timers don't need their own
thread — they piggyback on the reactor's event loop.

### 4. The blocking pool — for CPU-heavy or sync-I/O work

`tokio::task::spawn_blocking` runs work on a separate thread pool.
This is for code that can't be made async (e.g., `std::fs::File`
operations, CPU-heavy computation, blocking C library calls).

Why a separate pool? Because blocking work hogs the worker thread
it runs on. If you block a worker for 1 second, no other task on
that worker makes progress for 1 second. The blocking pool isolates
this damage.

### 5. The I/O driver — unifying everything

The I/O driver is the public-facing API for I/O. It owns the
reactor, the time driver, and the signal driver (for `Ctrl-C`). When
you call `tokio::net::TcpStream::connect`, you go through the I/O
driver, which registers with the reactor and returns a future that
resolves when the connection is established.

### How they interact

A typical I/O op (e.g., reading from a TCP socket):

1. **Task polls the I/O future.** The future checks the reactor's
   readiness map. If the FD isn't ready, registers a waker and
   returns `Pending`.
2. **OS notifies epoll.** The reactor's thread wakes up with a list
   of ready FDs.
3. **Reactor calls wakers.** For each ready FD, walks the waker list
   and calls each one.
4. **Wakers push tasks to worker queues.** The woken task is now in
   some worker's local queue.
5. **Worker pops and polls.** The worker picks up the task, calls
   its `poll` method. The future re-checks the readiness map, sees
   the FD is ready, returns `Ready(value)`.

The total coordination overhead: 1 `epoll_wait` wakeup + 1 task
poll + 1 scheduler queue op. ~microseconds.

### Tokio's metrics hooks

Tokio exposes runtime metrics via `RuntimeMetrics`:

```rust
let handle = runtime.handle();
let metrics = handle.metrics();
metrics.num_workers();          // worker count
metrics.active_tasks_count();   // currently polling
metrics.blocking_queue_depth(); // spawn_blocking queue
metrics.io_driver_ready_count();// FD readiness events
```

Commonware doesn't directly expose these, but the trait surface
(`runtime/src/telemetry/metrics.rs`) wraps Prometheus metrics
similarly.

## The full Tokio scheduler internals

Let's get into the weeds. Tokio's multi-thread scheduler has
three queue layers:

### Layer 1: local LIFO slot

A single-task slot. When a task wakes up (its waker is called),
the waker pushes the task onto the current worker's LIFO slot.
The next time the worker needs work, it pops from the LIFO slot
*first* — before checking any queue.

Why LIFO? Because of cache locality. The most recently woken task
is likely to have warm CPU caches. By running it next, you reduce
cache misses.

### Layer 2: local FIFO queue

A fixed-size queue (256 slots by default). When the LIFO slot is
empty, the worker pops from this queue.

The local queue is FIFO so older tasks don't starve. If you always
pushed to the front and popped from the front, you'd run hot tasks
repeatedly and starve cold ones.

### Layer 3: global injector

A multi-producer single-consumer queue (MPSC) shared by all
workers. When a worker spawns a new task or a waker fires on a
worker that can't run the task immediately, the task is pushed
to the injector.

The injector is a Chase-Lev work-stealing deque variant. Workers
push to the back; thieves steal from the front (half at a time).

### Stealing mechanics

When a worker has no local work:

1. It tries to pop from the global injector (FIFO from the front).
2. If empty, it picks a random victim worker and tries to steal
   half their local queue.
3. If still empty, it tries stealing from another worker.
4. After a few rounds of empty stealing, it parks (sleeps).

When a waker fires on a parked worker, the waker pushes to the
injector *and* sends a "wake up" notification to the parked
worker via a per-worker notify mechanism.

### The cost of task creation

Task creation is `~100 ns` for a simple task (heap allocation +
metadata). Compare to OS thread creation at `~1 ms` (mmap a stack).
The 10000x speedup is why Commonware can spawn thousands of tasks
per second without thinking about it.

But task creation isn't free. Every task allocates:
- The future (state machine enum) — boxed via `Box::pin`.
- A task header (waker, queue pointers, etc.).
- A slot in the worker's local queue.

For high-throughput servers, this matters. Commonware's actor model
(chapter 15) helps by reusing tasks for long-lived components.

## The full Tokio reactor internals

The reactor (`tokio/src/runtime/io/`) is the part that talks to
the OS. It's based on the `mio` crate, which provides cross-platform
async I/O primitives.

### `mio` — the cross-platform abstraction

`mio` exposes:
- `Poll` — the epoll/kqueue/IOCP handle.
- `Registry` — a clonable handle to register/unregister FDs.
- `Interest` — bit flags for read/write readiness.
- `Token` — a per-FD identifier (usize).
- `Source` — a trait wrapping a raw FD.

You register an FD with `Registry::register(&mut source, token,
interest)`. You call `Poll::poll(&mut events, timeout)` to wait for
events. When events arrive, you walk the list and dispatch.

### Epoll under the hood

On Linux, `mio::Poll` wraps `epoll_create1` + `epoll_wait`. Each
registered FD gets an entry in the kernel's interest list. When the
FD's readiness state changes (data arrives, buffer drains, etc.),
the kernel marks the FD ready.

`epoll_wait` returns a list of `(token, interest)` pairs for FDs
that became ready. The reactor walks the list, looks up the token
in its registry, finds the associated waker(s), and fires them.

### Readiness-based vs completion-based I/O

Linux `epoll` is *readiness-based*: it tells you when an FD is
*ready to be operated on*, but the actual operation (`read`,
`write`) is still synchronous. You have to call `read` again to
actually get the data.

Windows IOCP and Linux `io_uring` are *completion-based*: they tell
you when an operation *completes*. You submit an operation, the
kernel does it asynchronously, and notifies you when done.

Readiness-based is simpler but has more overhead per I/O op (you
submit, you re-check, you read). Completion-based has lower per-op
overhead but more setup cost.

Tokio primarily uses readiness-based (epoll). With the
`io-uring` feature, it can use completion-based for some
operations. Commonware's `tokio-uring` integration gives you
both options.

### `io_uring` for true async I/O

`io_uring` is a Linux 5.1+ feature for asynchronous I/O. The
kernel maintains a pair of ring buffers (submission queue + completion
queue). User space pushes submission queue entries (SQEs) and reads
completion queue entries (CQEs). No syscalls per operation after
the initial setup.

Tokio's `io_uring` support (via the `tokio-uring` crate, used by
Commonware for high-performance paths) submits operations like
`read`, `write`, `accept` directly to io_uring. The reactor polls
the completion queue and wakes tasks as completions arrive.

The performance difference can be 10-100x for high-throughput
servers, especially for storage (file I/O) and high-bandwidth
network (10+ Gbps).

### The waker integration

The reactor doesn't know about Rust futures or tasks — it just
maintains a map from `Token` to a list of `Waker`s. When an FD
becomes ready, it walks the list and calls each `Waker::wake`.

The waker's job: push the task onto some scheduler's queue.
The scheduler decides which worker runs the task next.

### The cost of reactor operations

- `epoll_ctl(EPOLL_CTL_ADD)` — first-time FD registration. ~1µs.
- `epoll_ctl(EPOLL_CTL_MOD)` — interest modification. ~1µs.
- `epoll_wait` — block until events. Variable; tens of microseconds
  in practice.
- Waker firing — push to queue. ~100ns.

For a server doing 100K req/s, total reactor overhead is ~10% of
CPU. The rest is actual I/O and protocol work.

## The full Tokio time driver

The time driver (`tokio/src/runtime/time/`) manages timers. Its
data structure is a **hierarchical timing wheel**.

### Anatomy of a timing wheel

A flat timing wheel is an array of slots, indexed by time modulo
the wheel size. Each slot holds a list of timers that fire in that
time slot. Insertion is O(1); firing all timers in the current slot
is O(k) for k timers in that slot.

A hierarchical timing wheel has multiple levels:
- Level 0: 256 slots, granularity 1ms → covers 256ms.
- Level 1: 64 slots, granularity 256ms → covers ~16s.
- Level 2: 64 slots, granularity ~16s → covers ~17 minutes.
- Level 3+: similar, covers days/years.

When a timer is inserted:
- Compute its deadline relative to current time.
- Place it in the appropriate level + slot.

When the time advances:
- Pop all timers from the current level-0 slot and fire them.
- If level 0 is empty and we advance past level 1's slot, "cascade"
  level 1's timers down to level 0.
- Similarly for higher levels.

The result: O(1) insertion, O(1) firing (amortized over many
operations), with low memory overhead.

### Why time-driven tasks are O(1)

In a naive priority queue (binary heap), every `sleep(duration)`
would be O(log n). In a timing wheel, it's O(1). For a runtime
handling millions of timers, the difference is significant.

### Integration with the reactor

The time driver is *one* of the sources the reactor polls. When a
timer's deadline arrives, the time driver wakes the corresponding
task via the same waker mechanism as I/O readiness.

This means: the reactor's single `epoll_wait` call wakes on I/O OR
timer events OR both. No separate thread needed for timers.

### Tokio's `time::sleep` in action

```rust
async fn example() {
    tokio::time::sleep(Duration::from_secs(1)).await;
    println!("1 second elapsed");
}
```

Under the hood: the future registers itself with the time driver
at the requested deadline. The time driver adds an entry to the
timing wheel. When the wheel turns past the entry, the time driver
wakes the future. The task is re-polled, sees the wake, returns
`Ready(())`.

## A worked example of a Commonware actor

Let's build a complete Commonware actor: a simple "message of the
day" service. Each client sends a request; the actor responds with
a hash of the current request count.

### The trait

First, define the message and context types:

```rust
use commonware_runtime::{Actor, Context, Handle, Spawner, Clock};
use commonware_macros::stability_scope;

pub enum Message {
    Request { id: u64, reply: oneshot::Sender<u64> },
    Shutdown,
}

stability_scope!(BETA {
    pub struct MotdActor;

    impl Actor for MotdActor {
        type Message = Message;
        type Output = ();

        async fn run(self, ctx: Context) -> () {
            self.loop_body(ctx).await
        }
    }
});
```

### The actor loop

The actor maintains internal state (request count, supervisor
relations) and processes messages. Here's the full implementation:

```rust
impl MotdActor {
    async fn loop_body(self, ctx: Context) {
        let mut count: u64 = 0;
        let metrics = ctx.register("motd",
            "message of the day service",
            motd_metrics::Metrics::default());

        loop {
            // Wait for the next message; this blocks the actor's task
            // but doesn't block the worker thread (cooperative yield).
            let msg = ctx.mailbox().recv().await;

            match msg {
                Message::Request { id, reply } => {
                    // Increment metric
                    metrics.requests.inc();

                    // Increment counter
                    count += 1;
                    let hash = blake3::hash(&count.to_le_bytes());

                    // Send response via oneshot channel
                    let _ = reply.send(hash.as_bytes()[0] as u64);

                    // Trace the work
                    tracing::debug!(id, count, "served request");
                }
                Message::Shutdown => {
                    tracing::info!(count, "shutdown");
                    break;
                }
            }
        }
    }
}
```

### Spawning the actor

```rust
let handle: Handle<()> = ctx.child("motd").spawn(|child_ctx| {
    MotdActor.run(child_ctx)
});
```

`ctx.child("motd")` creates a child supervisor (chapter 15). The
child gets its own metrics namespace ("motd_*") and tracing span.
Spawning returns a `Handle<()>` that resolves when the actor
terminates.

### Supervision

If the actor panics, the supervisor tree handles it:

```rust
let handle = ctx.child("motd")
    .with_attribute("restart_policy", "always")
    .spawn(|child_ctx| MotdActor.run(child_ctx));
```

Commonware's supervisor (chapter 15) can restart, abort, or escalate
based on policy. The default is "abort" (kill the supervisor if a
child fails). For a stateless service like MotdActor, "always
restart" is appropriate.

### Metrics and tracing

The metrics struct uses `prometheus-client`:

```rust
mod motd_metrics {
    use prometheus_client::metrics::counter::Counter;

    #[derive(Default)]
    pub struct Metrics {
        pub requests: Counter,
    }
}
```

The supervisor's label ("motd") auto-namespaces every metric:
`motd_requests_total`. The Prometheus exporter sees this name.

Tracing spans are auto-created from the context tree. Every message
processed by MotdActor logs under the `motd` span; nested spans
(reply send, hash compute) inherit the parent.

### Why this design works

The actor pattern gives us:
- **State encapsulation**: `count` lives inside the actor; no
  external mutation.
- **Backpressure**: the mailbox has a bounded size; if full, senders
  block (or get an error, depending on policy).
- **Single-threaded processing**: messages are processed one at a
  time, in order. No locks needed.
- **Supervision**: failures are contained; parent can restart child.
- **Observable**: metrics + tracing built in.

Compare to a "global singleton with a mutex" approach: race
conditions, no metrics, no supervision, no observability.

### Production patterns

In production, you typically:
- Run multiple instances of the same actor (one per shard, one per
  peer).
- Use the `Overflow` policy to drop messages under load rather than
  block.
- Add circuit breakers for external dependencies.
- Trace every message with a request ID for end-to-end observability.

These are commonware's defaults — the actor framework makes them
ergonomic.

## Loom in depth

We mentioned `loom` briefly in the main text. Let's go deeper.

### How loom permutes thread interleavings

`loom` is a permutation tester for concurrent Rust code. When you
write:

```rust
loom::model(|| {
    let a = loom::thread::spawn(|| { ... });
    let b = loom::thread::spawn(|| { ... });
    a.join().unwrap();
    b.join().unwrap();
});
```

`loom` explores *every possible interleaving* of the atomic
operations in the two threads. For simple code, that's 2-4
interleavings. For complex code, it can be thousands.

**What's "atomic" to loom?** All atomic operations (`fetch_add`,
`compare_exchange`, `load`, `store`) on `AtomicT`. All `Mutex::lock`
and `RwLock::read/write`. All `Sender::send` and `Receiver::recv`.
All `Waker::wake`.

For each permutation, loom runs the test from scratch and checks
the assertions. If any permutation fails an assertion, loom reports
the failing interleaving as a counter-example.

### Why this is the gold standard for concurrency tests

Real concurrent tests have a fundamental problem: most bugs are
intermittent. They only manifest under specific scheduling
conditions. Running a test 1000 times might miss the bug. Running
it 1,000,000 times might miss it too (if the scheduler doesn't
hit the right interleaving).

Loom solves this by *exhaustively exploring* all interleavings. If
there's a bug, loom finds it. The cost: exponential runtime. A
test with 10 atomic ops can have ~3.6M permutations. Loom is
practical for small, focused tests (waker registration, lock
contention) but not for large systems.

### When to use loom vs the deterministic runtime

**Loom:** for the lowest-level concurrency primitives where races
are most likely. Commonware uses loom for:
- Waker registration in the runtime (the trickiest concurrency).
- Mailbox overflow with multiple senders.
- RNG locking in the deterministic executor.

**Deterministic runtime:** for protocol-level testing (consensus,
broadcast, storage). High-level simulation of multi-node systems.

The two are complementary. Loom verifies *intra-task* concurrency
correctness. The deterministic runtime verifies *inter-task*
protocol correctness.

### Performance implications

A loom test can be 100x-10000x slower than the equivalent real test.
A test that takes 1µs in production might take 1ms under loom.

Commonware's strategy: use loom only for *small focused tests* of
the most race-prone code paths. Don't try to loom-test an entire
consensus protocol.

## The Deterministic Executor's event loop

We sketched the event loop in the main text; here's the full detail.
From `runtime/src/deterministic.rs:500-700`, paraphrased:

```rust
loop {
    // Step 1: Advance time
    let now = {
        let mut t = self.time.lock().unwrap();
        if self.tasks.ready_count() == 0 {
            // No work to do. Skip to next alarm if any.
            if let Some(next_alarm) = self.sleeping.peek() {
                let next_time = next_alarm.time.max(*t + self.cycle);
                *t = next_time;
            }
        } else {
            // Have work. Advance by `cycle`.
            *t += self.cycle;
        }
        *t
    };

    // Step 2: Wake expired sleepers
    {
        let mut sleeping = self.sleeping.lock().unwrap();
        while let Some(alarm) = sleeping.peek() {
            if alarm.time <= now {
                let alarm = sleeping.pop().unwrap();
                alarm.waker.wake();
            } else {
                break;
            }
        }
    }

    // Step 3: Pick a random ready task and poll it
    if let Some(task) = self.tasks.random_ready(&mut *self.rng.lock().unwrap()) {
        self.tasks.poll_task(task, now);
    }

    // Step 4: Liveness check
    if self.tasks.ready_count() == 0 && self.sleeping.lock().unwrap().is_empty() {
        panic!("runtime stalled: no work and no waiters");
    }

    // Step 5: Check deadline
    if let Some(deadline) = self.deadline {
        if now >= deadline {
            panic!("runtime exceeded deadline");
        }
    }

    // Step 6: Check shutdown signal
    if self.shutdown.is_signaled() {
        break;
    }
}
```

### What each step does

**Step 1 (advance time):** advances the virtual clock by `cycle`
(default 1ms) when there's work to do. If no work, jump to the
next alarm's deadline. This compresses long sleeps into single
iterations.

**Step 2 (wake sleepers):** pops all alarms whose deadlines have
elapsed and wakes their associated tasks. Tasks are pushed back
onto the ready queue (which step 3 will pick from).

**Step 3 (pick and poll):** RNG-driven selection of a ready task
to poll. This is the *deterministic* part — every test with the
same seed picks tasks in the same order.

**Step 4 (liveness check):** if no tasks are ready AND no sleepers
are pending, the runtime is stuck. Panic. This catches deadlock
in tests: if your code deadlocks, the test panics immediately
with a clear message.

**Step 5 (deadline check):** if the test exceeded its configured
deadline (e.g., 30 seconds of virtual time), panic. Prevents tests
from running forever.

**Step 6 (shutdown):** if a `Stopper::stop()` was called, exit the
loop cleanly.

### Why RNG-driven task selection

Tokio-style work-stealing is fast in production but introduces
non-determinism into tests: which task runs next depends on
work-stealing decisions, which depend on timing.

RNG-driven selection makes the *ordering* of task polls
deterministic given the seed. Every test run with seed 42 picks
tasks in the exact same order. This is what makes test failures
reproducible.

The RNG is the seeded `BoxDynRng` from `Config::rng`. Every random
choice in the runtime (task selection, fault injection, message
reordering) goes through it.

### Why virtual time is virtual

The virtual clock is just a `u128` representing nanoseconds since
the configured `start_time`. It advances in increments of `cycle`
(by default 1ms). Real wall-clock time is irrelevant.

This means:
- 10,000 simulated seconds can run in milliseconds of wall clock.
- A test that "sleeps for 1 hour" doesn't take 1 hour.
- Determinism is preserved across runs (no wall-clock influence).

The `external` feature (appendix G in the main text) is the
exception: it makes the deterministic runtime yield to real time
when interacting with external processes.

## Exercises

**Exercise 2.1 — Trace an async state machine.** Take this async
function and write out the enum that the compiler generates:

```rust
async fn triple(input: u64) -> Result<u64, ()> {
    let a = compute_a(input).await?;
    let b = compute_b(a).await?;
    let c = compute_c(b).await?;
    Ok(c)
}
```

How many variants? Which locals are live in which variants?

**Exercise 2.2 — Count operations per Tokio I/O.** Trace through
the steps for reading from a TCP socket. How many `epoll_wait`
calls? How many waker fires? How many scheduler queue ops?

**Exercise 2.3 — Design a hierarchical timing wheel.** For a wheel
with 4 levels (1ms granularity at level 0, 256x multipliers),
what's the maximum time range it covers? How many slots total?

**Exercise 2.4 — Implement a tiny Commonware actor.** Write the
`Message` enum, `Actor` impl, and supervisor setup for an actor
that counts received messages and exposes the count via a metric.
Make it shut down gracefully on `Shutdown` message.

**Exercise 2.5 — Use loom to test a small concurrent primitive.**
Write a loom test for a simple `Mutex<u32>` wrapper that supports
`increment` and `get`. Verify that after N concurrent increments,
the value is exactly N.

**Exercise 2.6 — Modify the event loop.** Take the pseudocode above
and add support for "high-priority" tasks that always run before
normal tasks. How does this interact with the RNG-driven selection?
How would you test the priority behavior?

## Appendix A — The deterministic executor's state machine, in detail

`runtime/src/deterministic.rs:355-440` is the heart. Let me walk through
it field by field.

```rust
pub struct Executor {
    registry: Registry,                      // Prometheus metrics registry
    cycle: Duration,                          // time advance per iteration
    deadline: Option<SystemTime>,            // panic if test exceeds this
    metrics: Arc<Metrics>,                    // runtime-specific counters
    auditor: Arc<Auditor>,                    // SHA-256 of all events
    rng: Arc<Mutex<BoxDynRng>>,               // shared RNG
    time: Mutex<SystemTime>,                  // virtual clock
    tasks: Arc<Tasks>,                        // task pool
    sleeping: Mutex<BinaryHeap<Alarm>>,       // scheduled wake-ups
    shutdown: Mutex<Stopper>,                 // shutdown coordination
    panicker: Panicker,                       // panic handling
    dns: Mutex<HashMap<String, Vec<IpAddr>>>, // mocked DNS
}
```

Each field has a job. Let me unpack the interesting ones.

### `Tasks` — the task pool

A custom `Arc<Tasks>` type (defined later in the file). It tracks:
- **Ready tasks** — tasks whose waker has been called and are ready to be polled.
- **Running tasks** — tasks currently being polled.
- **Blocked tasks** — tasks whose waker hasn't been called yet (waiting on a future).
- **Finished tasks** — completed; their `JoinHandle` can resolve.

The executor loop interacts with `Tasks` via:
- `tasks.ready() -> usize` — how many ready tasks right now.
- `tasks.poll_next(rng)` — randomly pick one ready task to poll (RNG-driven).
- `tasks.spawn(future)` — register a new task.
- `tasks.abort(handle)` — abort a task and propagate to descendants.

This is the "select which task to run next" scheduler. It's a deliberate
contrast to work-stealing (tokio's default) — for deterministic testing,
RNG-driven selection gives you different orderings across runs without
introducing non-determinism from physical time.

### `BinaryHeap<Alarm>` — the sleep heap

A priority queue of (deadline, waker) pairs. When a task awaits
`Clock::sleep(duration)`, an `Alarm` is pushed. When the executor's
clock advances past the deadline, the alarm fires, the waker is woken,
the task becomes ready.

The heap ordering is **min-heap by deadline** — the next-firing alarm
is at the top. So `peek()` is the next wake-up.

When idle (no ready tasks), the executor jumps time forward to the next
alarm's deadline (`skip_idle_time` in `deterministic.rs:393-415`). This
**compresses long sleeps** — if you sleep for 1 hour in the test, the
runtime skips ahead 1 hour instead of polling 3.6 million cycles.

### `Stopper` — shutdown coordination

`runtime/src/utils/signal/`. The `Stopper` is a one-shot signal that
resolves when someone calls `Spawner::stop(value)`. Tasks that hold a
`Signal` from `Spawner::stopped()` get notified and can do graceful
shutdown.

This is how Simplex and other long-running primitives shut down cleanly
on `Ctrl-C`. The shutdown value (i32) is propagated so the runtime
exit code reflects the shutdown reason.

### `Panicker` — controlled panic propagation

If `Config::catch_panics = false` (default), any task panic propagates up
to the root, killing the whole runtime. If `catch_panics = true`, the
panic is captured and returned via the `JoinHandle::join()`.

Production default is `false` — a panic in your consensus task should
kill the process, not be silently caught. Tests sometimes use `true` to
verify "this input causes a panic."

## Appendix B — The event loop in pseudocode

The actual loop (paraphrased from `deterministic.rs:500-700`):

```rust
loop {
    // 1. Advance time
    if tasks.ready() == 0 {
        // Skip to next sleep deadline if no work
        time = min(time + cycle, next_alarm_time());
    } else {
        time = time + cycle;
    }

    // 2. Wake expired sleepers
    while let Some(alarm) = sleeping.peek() {
        if alarm.time <= time {
            alarm.waker.wake();
            sleeping.pop();
        } else { break; }
    }

    // 3. Pick a random ready task and poll it
    if let Some(task) = tasks.random_ready(&mut rng) {
        tasks.poll_task(task);
    }

    // 4. Liveness check
    if tasks.ready() == 0 && sleeping.is_empty() {
        panic!("runtime stalled: no work and no waiters");
    }

    // 5. Check deadline
    if let Some(d) = deadline {
        if time >= d { panic!("runtime exceeded deadline"); }
    }
}
```

That's the entire event loop. ~30 lines of logic. Every async primitive
you use (`spawn`, `sleep`, `select!`, `await`, etc.) is built on top of
this loop via the trait surfaces.

## Appendix C — `Context` — what every spawned task receives

When you call `context.spawn(|child_ctx| async move { ... })`, the closure
gets a new `Context` that is a **child** of the parent context. The child:
- Has the same underlying executor.
- Has a new `Supervisor` node (chapter 15 supervision tree).
- Inherits `Clock`, `Network`, `Storage`, `Metrics` traits.
- Has its own waker registration (independent of parent).

```rust
let handle = context.child("worker").spawn(|ctx| async move {
    // ctx is a new Context with "worker" label
    // ctx.current() returns the same clock
    // ctx.register() registers metrics prefixed with "worker_"
    // ctx.auditor() feeds into the same Auditor as parent
    loop { ctx.sleep(Duration::from_secs(1)).await; }
});
```

The supervisor tree (chapter 15) means aborting the parent aborts the
child. The label is used for metric namespacing and for tracing spans.

## Appendix D — Buffer pools

`runtime/src/iobuf/`. Network I/O and storage I/O allocate buffers. Naive
allocation (one `Vec<u8>` per message) wastes memory and fragments the
allocator. Commonware uses **size-class buffer pools**.

The pattern:
- Pages of fixed size (e.g., 4 KB).
- Free pages are kept in per-thread caches.
- Allocation: pop a free page, or allocate a new one.
- Deallocation: push back to the free list.

`BufferPoolConfig` (chapter 02's `Config::network_buffer_pool_config` and
`storage_buffer_pool_config`) lets you tune:

```rust
pub struct BufferPoolConfig {
    pub page_size: NonZeroU32,        // typically 4096
    pub max_per_class: NonZeroU32,   // cap on per-size-class free pages
    pub thread_cache_size: NonZeroU32, // per-thread cache depth
}
```

In tests (deterministic runtime), the pool is **disabled** (`with_thread_cache_disabled()`)
to keep test runs deterministic — otherwise the pool's behavior depends on
which thread allocated which page.

In production, the pool is **enabled** and tuned for the workload. For a
consensus engine sending 1KB messages, you'd want small classes (256B,
512B, 1K) more heavily cached than large classes (16K, 64K).

## Appendix E — Telemetry: metrics + tracing

`runtime/src/telemetry/`. Two parts.

### Prometheus metrics

`runtime/src/telemetry/metrics.rs`. The `Metrics` trait:

```rust
pub trait Metrics: Supervisor {
    fn register<N, H, M>(&self, name: N, help: H, metric: M) -> Registered<M>;
    fn encode(&self) -> String;
}
```

- `register` — add a counter/gauge/histogram with a name + help string.
- `encode` — dump all metrics in Prometheus text format.

The supervision-tree labels are auto-prepended. So a metric registered
in `orchestrator.engine` as `votes` becomes `orchestrator_engine_votes`
in Prometheus. Plus the `with_attribute` values become Prometheus
**labels**, not name components.

### Tracing

`runtime/src/telemetry/traces/`. Uses the `tracing` crate. Spans are
auto-created from the context tree.

From `AGENTS.md`, the tracing rules:

- Span names use dot-separated paths (e.g., `consensus.voter.process`).
- Never `::` in span names.
- `#[tracing::instrument(name = "...")]` on functions performing work.
- Manual `info_span!` only when the name depends on call-site context.
- `tracing`'s implicit context is **task-local** — it doesn't survive a
  mailbox channel. Messages must carry their `Span` explicitly across
  actor boundaries.

The latter is a subtle but important detail. When actor A sends a
message to actor B, B's tracing context is fresh. To preserve the trace
hierarchy, A includes its current Span in the message, and B does
`.instrument(span)` when processing.

## Appendix F — Loom: permutation testing for concurrent code

`loom` is a Rust crate that **systematically permutes** concurrent
operations to find races. Commonware uses it for the lowest-level
primitives where races are most likely.

```rust
#[test]
fn test_no_race() {
    loom::model(|| {
        let a = loom::thread::spawn(|| { ... });
        let b = loom::thread::spawn(|| { ... });
        a.join().unwrap();
        b.join().unwrap();
    });
}
```

Loom tries **all possible interleavings** of the two threads' atomic
operations. It's exhaustive — if there's a race, loom finds it. The
trade-off: exponential runtime, only practical for small code paths.

Commonware uses loom for the runtime's waker registration (the trickiest
concurrency in the codebase), the mailbox overflow (where two senders
contend), and the deterministic executor's RNG locking.

## Appendix G — The `external` feature: testing with real external systems

`runtime/src/deterministic.rs:69` mentions the `external` feature:

> Applications that interact with external processes (or are able to mock
> them) should never need to enable this feature. It is commonly used when
> testing consensus with external execution environments that use their
> own runtime (but are deterministic over some set of inputs).

What it does: when `external` is enabled, the deterministic runtime
**actually sleeps** for `cycle` duration each iteration, AND respects
`pace()` latency hints on external futures.

The use case: testing Simplex with an "external" execution engine (e.g.,
a separate runtime driving state transitions). The external engine has
its own timing; the deterministic runtime must not progress in virtual
time past what the external engine has actually observed.

Without `external`: virtual time can race ahead of real time, causing
your "deterministic" test to make assumptions that hold in simulation
but fail in production.

With `external`: the test still uses the deterministic executor for
consensus logic, but yields to real time when interacting with the
external engine. Slow but accurate.

## Appendix H — `deterministic::Runner` vs `tokio::Runner`

The two `Runner` implementations look identical at the trait level but
behave very differently:

| Aspect | `tokio` | `deterministic` |
|---|---|---|
| Time | Real (wall clock) | Virtual (advances by cycle) |
| Scheduling | Work-stealing | RNG-driven |
| RNG | Real `OsRng` or seeded | Always seeded |
| Network | Real TCP | In-memory channels |
| Storage | Real filesystem | In-memory HashMap |
| Panic | Propagates | Propagates (or caught) |
| Speed | Real | Milliseconds |
| Reproducibility | No | Yes (with seed) |

The same code, written against the trait surface, runs in either.

The migration path: in tests, use `deterministic::Runner::seeded(N)`.
In production, use `tokio::Runner::new(tokio::Config::new())`. No code
changes elsewhere.

## Appendix I — The Auditor's hash chain

`runtime/src/deterministic.rs:144-183`. Let me show the exact hash chain:

```rust
pub(crate) fn event<F>(&self, label: &'static [u8], payload: F)
where F: FnOnce(&mut Sha256),
{
    let mut digest = self.digest.lock();
    let mut hasher = Sha256::new();
    hasher.update(digest.as_ref());  // <-- previous hash
    hasher.update(label);             // <-- event name
    payload(&mut hasher);             // <-- event-specific data
    *digest = hasher.finalize().into();
}
```

The chain: `H_n = SHA256(H_{n-1} || label || data)`. Each event includes
the previous hash, so any two executions that diverge at any point
produce different final hashes.

In test code:

```rust
executor.start(|context| async move {
    // ... do stuff ...
    let auditor_state = context.auditor().state();  // 64-char hex string
    // assert_eq!(auditor_state, "expected_hash_for_this_seed_and_this_code");
});
```

If `auditor_state` doesn't match the expected hash, you have a
non-determinism somewhere. The test fails. The bug is reproducible.

## Where to look in the code (expanded)

- `runtime/src/lib.rs:152-779` — all the trait surfaces.
- `runtime/src/deterministic.rs:355-500` — the Executor and Runner state.
- `runtime/src/deterministic.rs:500-700` — the event loop.
- `runtime/src/network/audited.rs` — every network call goes through the Auditor.
- `runtime/src/storage/audited.rs` — every storage call too.
- `runtime/src/utils/supervision/` — the supervisor tree implementation.
- `runtime/src/utils/signal/` — the Stopper (graceful shutdown).
- `runtime/src/telemetry/metrics.rs` — Prometheus integration.
- `runtime/src/iobuf/` — buffer pool.
- `runtime/src/iouring/` — Linux io_uring backend.
- `runtime/src/tokio/` — Tokio backend.

## If you only remember three things

1. **The runtime is an interface.** Same code, two implementations: Tokio for prod, a deterministic simulator for tests.
2. **The Auditor SHA-256s every event.** Same seed + same code → same final hash. That hex string is your reproducibility receipt.
3. **The simulated network is realistic.** Latency, jitter, packet loss, bandwidth limits, max-min fairness, mid-test partitions. Tests fail in CI the same way they'd fail in prod.

## Cross-reference

The full `Context` API surface (every method on `Supervisor`, `Spawner`,
`Clock`, `Metrics`, `Storage`, `Blob`) lives in **Chapter 21 —
Cross-cutting patterns** (`chapters/21-cross-cutting.md`). When you start
writing code and need "what was that method called again," go there.

→ Next: **Chapter 03 — Codec**. Bytes on the wire need to round-trip into Rust
types and back, deterministically, with conformance tests that detect drift.