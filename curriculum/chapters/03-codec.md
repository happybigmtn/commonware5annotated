# Chapter 03 — Codec: bytes on the wire, safely

> How Commonware ships typed messages across the network without breaking.

## The problem

You have a Rust struct. You want to send it to another node. That node is on a
different machine, possibly a different architecture (32-bit vs 64-bit,
little-endian vs big-endian), and possibly **hostile** — it might send you
garbage that you have to safely reject.

Four things must be true:

1. **Round-trip deterministic.** Encode on machine A, decode on machine B, get
   the same value. Always.
2. **Platform-independent.** A 32-bit validator and a 64-bit validator must
   agree on the wire format. Forever.
3. **Safe against untrusted input.** If a malicious peer sends you a "length
   prefix" of `2^60`, you mustn't allocate 2^60 bytes.
4. **Stable over time.** If you change the encoding format, all the deployed
   nodes need to either still understand the old format or refuse cleanly.
   Silent format drift = consensus split = catastrophic.

This is the codec crate's job. And it does it with a tiny trait surface that's
surprisingly deep.

## The trait surface

Open `codec/src/codec.rs`. Five core traits, two convenience aliases:

```rust
pub trait FixedSize {                                       // "I always encode to N bytes"
    const SIZE: usize;
}

pub trait EncodeSize {                                       // "tell me how big I am before you write me"
    fn encode_size(&self) -> usize;
    fn encode_size_slice(values: &[Self]) -> usize          // O(1) for FixedSize, O(n) for variable
        where Self: Sized;
}

pub trait Write {                                           // "write yourself to this buffer"
    fn write(&self, buf: &mut impl BufMut);
    fn write_slice(values: &[Self], buf: &mut impl BufMut) where Self: Sized;
    fn write_bufs(&self, buf: &mut impl BufsMut);            // zero-copy path for Bytes fields
}

pub trait Read: Sized {                                      // "read yourself from this buffer"
    type Cfg: Clone + Send + Sync + 'static;                 // decode-time config
    fn read_cfg(buf: &mut impl Buf, cfg: &Self::Cfg) -> Result<Self, Error>;
}

pub trait Encode: Write + EncodeSize { ... }                 // everything Write + size
pub trait Decode: Read { ... }
pub trait Codec: Encode + Decode { ... }
```

Plus `EncodeShared`, `CodecShared`, `EncodeFixed`, `DecodeFixed`, `CodecFixed`
for types that need to cross thread boundaries or are guaranteed fixed-size.

The asymmetry is **deliberate**: `Write` is parameter-free, `Read::read_cfg` takes
a `Cfg`. That's how you safely decode untrusted data — supply the bounds at
decode time, not encode time.

## The four critical design decisions

### Decision 1: Big-endian, always

`codec/src/types/primitives.rs:18-19`:

> All fixed-size integers and floats are written big-endian to avoid host-
> endian ambiguity.

x86 is little-endian. ARM is sometimes little-endian, sometimes big-endian.
Network order is big-endian (the name "endian" comes from Gulliver's Travels —
the war over which end of an egg to crack). Commonware picks the network-order
side of this debate. No ambiguity, ever.

### Decision 2: `usize` is a varint (and capped at u32)

Also `primitives.rs:11-17`:

> `usize` is the lone exception: since most values refer to a length or size
> of an object in memory, values are biased towards smaller values. Therefore,
> it uses variable-length (varint) encoding to save space. **This means that
> it does not implement [FixedSize]. When decoding a `usize`, callers must
> supply a [RangeCfg] to bound the allowable value** — this protects against
> denial-of-service attacks that would allocate oversized buffers.

Look at `codec/src/varint.rs`. It's **Google's Protocol Buffers varint**:
each byte carries 7 data bits + 1 continuation bit. Small numbers fit in 1
byte. Big numbers take more. The encoder emits the shortest possible varint
and rejects "lazy" encodings (where someone pads a small number into a longer
varint).

Why? Because **the same number has exactly one canonical encoding**. No
ambiguity. And capping at u32 (4 bytes worst case) keeps the wire format
identical across 32-bit and 64-bit architectures.

### Decision 3: Every `Read::read_cfg` takes a Cfg

Look at `codec/src/codec.rs:173-189`:

```rust
pub trait Read: Sized {
    /// The `Cfg` type parameter allows passing configuration during the read process.
    /// This is crucial for safely decoding untrusted data, for example, by providing
    /// size limits for collections or strings.
    type Cfg: Clone + Send + Sync + 'static;
    fn read_cfg(buf: &mut impl Buf, cfg: &Self::Cfg) -> Result<Self, Error>;
}
```

For types that need no config (like `u32`), `Cfg = ()`. For collections,
`Cfg = (RangeCfg<usize>, T::Cfg)` — you supply the **allowed range** of lengths
at decode time. So if you're reading a `Vec<Transaction>` from the network, you
pass `0..=10_000` to refuse a "vector" of 2 billion txs.

This is a **fundamental safety pattern**: never trust the wire to tell you how
much to allocate.

### Decision 4: Containers are length-prefixed

Look at `codec/src/types/vec.rs:35-47`:

```rust
impl<T: Write> Write for &[T] {
    fn write(&self, buf: &mut impl BufMut) {
        self.len().write(buf);     // <-- length first
        T::write_slice(self, buf); // <-- then items
    }
}
```

A `Vec<u8>` on the wire is `[varint length][byte 1][byte 2]...[byte n]`. A
`Vec<MyMessage>` is `[varint count][encoded msg 1][encoded msg 2]...`. The
reader knows when to stop because the length prefix tells it how many items
to expect.

This is **streaming-friendly**: a decoder can read the length prefix
incrementally, allocate the right buffer, then fill it. No need to buffer
forever.

## The zero-copy optimization: `write_bufs`

`codec/src/codec.rs:142-169`. Most of the time, encoding means "copy bytes
into a target buffer." But what if your type contains a `Bytes` field that's
already a chunk of bytes? Copying is wasteful — you could just push the chunk
onto a list-of-buffers and avoid the copy.

That's what `BufsMut` is for. `Bytes` (the crate) has a `BytesMut` that can
hold multiple separate chunks; you can push a `Bytes` chunk onto it without
copying. The `write_bufs` hook lets your type opt into this:

```rust
fn write_bufs(&self, buf: &mut impl BufsMut) {
    self.write(buf)   // default falls back to inline copy
}
```

If your type has a `Bytes` field, override this to push the field as a chunk
and only write the inline header inline. For a 1 MB block, you save a 1 MB
copy. Across a consensus protocol, that's huge.

## The size hooks: `encode_size_slice`

`codec/src/codec.rs:44-51`:

```rust
fn encode_size_slice(values: &[Self]) -> usize
where Self: Sized
{
    values.iter().map(EncodeSize::encode_size).sum()   // default: O(n) walk
}
```

But for `FixedSize` types (`codec/src/codec.rs:96-101`):

```rust
impl<T: FixedSize> EncodeSize for T {
    fn encode_size(&self) -> usize { Self::SIZE }

    fn encode_size_slice(values: &[Self]) -> usize
    where Self: Sized
    {
        Self::SIZE * values.len()   // O(1)!
    }
}
```

If your `Vec<u32>` has 10 million elements, you don't want to walk all 10M to
compute the encoded size — you want `4 * 10_000_000 = 40_000_000` immediately.
The size hook lets FixedSize types skip the O(n) pre-pass.

## Conformance testing — catching drift

Open `codec/src/conformance.rs:28-60`. The trick:

```rust
pub fn generate_value<T>(seed: u64) -> T
where T: for<'a> Arbitrary<'a>,
{
    let mut rng = ChaCha8Rng::seed_from_u64(seed);
    let mut buffer = vec![0u8; INITIAL_BUFFER_SIZE];
    rng.fill(&mut buffer[..]);
    let mut unstructured = Unstructured::new(&buffer);
    T::arbitrary(&mut unstructured).unwrap()
}
```

And the `Conformance` impl:

```rust
impl<T> Conformance for CodecConformance<T>
where T: Encode + for<'a> Arbitrary<'a> + Send + Sync,
{
    async fn commit(seed: u64) -> Vec<u8> {
        let value: T = generate_value(seed);
        value.encode().to_vec()
    }
}
```

Here's the loop:

1. Take a deterministic seed.
2. Use it to seed a ChaCha RNG.
3. Use the RNG to generate a random instance of `T` via the `arbitrary` crate.
4. Encode it.
5. Hash all the encoded bytes together.
6. Compare against the stored hash in `conformance.toml`.

The hash IS your wire format. If you change `MyStruct` from
`{ a: u32, b: u8 }` to `{ a: u32, b: u16 }`, the encoding changes, the hash
changes, the test fails. **You have to explicitly regenerate the hash** (via
`just regenerate-conformance`) to acknowledge "yes, I changed the wire format."

This catches the worst class of bugs — silent format drift. Without it, you'd
deploy a new version and two thirds of the network wouldn't understand the
new format and consensus would split.

The `AGENTS.md` pattern for adding a new type:

```rust
commonware_conformance::conformance_tests! {
    CodecConformance<MyType>,                  // default # of cases
    CodecConformance<MyType2> => 1024,         // explicit # of cases
}
```

That's it. The macro wires up the rest.

## The error type

`codec/src/error.rs`. Every decode failure is one of:

| Error | Meaning |
|---|---|
| `EndOfBuffer` | Buffer ran out before reading enough bytes |
| `ExtraData(n)` | n bytes left over after decoding — buffer had too much |
| `InvalidVarint(n)` | Malformed varint of length n bytes |
| `InvalidUsize` | usize-specific varint problem |
| `InvalidBool` | Byte was neither 0 nor 1 |
| `InvalidEnum(v)` | Decoded enum variant `v` doesn't exist |
| `InvalidLength(n)` | Length prefix outside the allowed range |
| `Invalid(ctx, msg)` | Application-level validation (e.g., keys not sorted) |
| `Wrapped(ctx, err)` | Boxed downstream error |

Notice: every error is **specific**. There's no generic "decode failed." When
a consensus node rejects a message, you know exactly why. That's not a luxury
— it's how you debug BFT bugs at 3am.

## Putting it all together: how a message crosses the wire

Say Alice sends a `Vec<Signature>` to Bob, where each signature is 96 bytes
(BLS12-381 G2 point). Here's the wire format:

```
+--------+--------+--------+--------+-----+--------+
| LEN   |   SIG 1 (96 bytes)    |    SIG 2 (96 bytes)  | ...
| varint| b0..b95              | b0..b95             |
+--------+--------+--------+--------+-----+--------+
```

LEN is a varint (1 byte if there are < 128 signatures, 2 bytes otherwise).
Each signature is fixed-size (96 bytes). On a wire with 10 signatures: 1 byte
LEN + 960 bytes = 961 bytes total. Bob reads LEN, allocates 960 bytes, reads
10 fixed-size sigs.

On Bob's decode side, he calls:

```rust
let sigs: Vec<Signature> = Vec::decode_cfg(buf, &(RangeCfg::new(0..=10_000), ()))?;
```

That `0..=10_000` is the safety bound — refuse a "vector" of 2 billion
signatures.

## Where to look in the code

- `codec/src/codec.rs` — the trait surface.
- `codec/src/types/primitives.rs` — big-endian, varint for usize, u32 cap.
- `codec/src/types/vec.rs` — length-prefixed containers, `Cfg = (RangeCfg, T::Cfg)`.
- `codec/src/varint.rs` — the varint codec with canonical encoding enforcement.
- `codec/src/conformance.rs` — the conformance testing harness.
- `codec/src/types/btree_map.rs`, `hash_map.rs`, etc. — same patterns for other containers.

## Why big-endian (network byte order) — the long version

We said all fixed-size integers are big-endian (`primitives.rs:18-19`). This
choice is older than Commonware, older than Rust, older than the internet as
we know it. It's worth understanding why.

### The historical accident

The name "endian" comes from Jonathan Swift's *Gulliver's Travels* (1726),
where two empires go to war over which end of a boiled egg to crack. In
computer science, the term was popularised by Danny Cohen in his 1980 paper
*"On Holy Wars and a Plea for Peace"* (`https://www.ietf.org/rfc/ien/ien137.txt`),
which compared the two-byte-order camps to Lilliput and Blefuscu.

Cohen argued that someone had to blink. In 1980, the ARPANET community
decided it would be the Internet community. RFC 791 (1981, the IP spec)
chose **big-endian** as "the standard network byte order." RFC 1700 (1994)
codified it. Every subsequent protocol — TCP, UDP, DNS, IP itself — followed.

The reason was not technical superiority. It was that the dominant
"big iron" of the day (IBM mainframes, Motorola 68000, SPARC) shipped
big-endian, and ARPA wanted to pick a side and move on.

### Why x86 is little-endian but networks are big-endian

When Intel designed the 8086 in 1978, they chose little-endian. The reason
was hardware: little-endian lets you do `inc eax` to add 1 to a 16-bit value
at any offset without needing to know the operand's width. Big-endian needs
extra arithmetic to know whether the byte you're looking at is the high
byte or the low byte. So little-endian wins on microcode simplicity.

That tiny hardware decision got cemented by the IBM PC in 1981, then by
every clone. x86 became the desktop. ARM was little-endian by default in
most configurations. Today, nearly every CPU you touch is little-endian.

But networks stayed big-endian. Two reasons:

1. **Backward compatibility.** Once millions of hosts had shipped IP
   stacks expecting big-endian, you cannot flip the switch.
2. **Hash functions and integer parsing.** Reading a multi-byte integer in
   big-endian is a left-to-right scan that matches how humans write numbers
   on paper. Little-endian reverses the bytes, which complicates byte-level
   parsing.

### The cost on Commonware

`codec/src/types/primitives.rs:18-19` emits big-endian. On a little-endian
x86 host, every `write_u32` needs to byte-swap. Modern CPUs handle this in
a single cycle (the `BSWAP` instruction on x86, `REV` on ARM). The
throughput hit is negligible for typical consensus traffic.

What we get in exchange is **wire-format portability**. A block produced by
a little-endian validator can be parsed byte-for-byte by a big-endian
validator (or an FPGA, or a hardware wallet, or a debug tool) without any
endianness negotiation. That's worth more than the cycles.

### How this differs from Bincode and Postcard

Most Rust serialization libraries are **little-endian by default** because
they were built for x86-native in-process use. Bincode's default config emits
little-endian. Postcard follows. This is fine for caching, IPC, and
disk-on-the-same-machine. It is a hazard for cross-machine wire formats.

Commonware, by contrast, optimises for "the bytes will cross trust
boundaries and CPU architectures forever." Big-endian is the boring,
correct choice.

For reference, the IETF's official position is in RFC 1700 ("Assigned
Numbers"), which says: "All binary numbers in network protocols are
big-endian." Commonware picks that side of the egg.

## A type-theory perspective on Encode and Decode

Let's step back and look at `Encode` and `Decode` from a different angle.

```rust
pub trait Write {                                         // parameter-free
    fn write(&self, buf: &mut impl BufMut);
    fn write_slice(values: &[Self], buf: &mut impl BufMut) where Self: Sized;
    fn write_bufs(&self, buf: &mut impl BufsMut);
}

pub trait Read: Sized {                                    // takes a Cfg
    type Cfg: Clone + Send + Sync + 'static;
    fn read_cfg(buf: &mut impl Buf, cfg: &Self::Cfg) -> Result<Self, Error>;
}

pub trait EncodeSize {                                     // size before write
    fn encode_size(&self) -> usize;
    fn encode_size_slice(values: &[Self]) -> usize where Self: Sized;
}
```

Viewed as mathematical functions:

```
Write:   (T, BufMut) -> ()            -- side-effecting; "produces bytes"
Read:    (Buf, Cfg) -> Result<T, Error> -- fallible; "consumes bytes"
EncodeSize: T -> usize                -- total; "predicts bytes"
```

The asymmetry is **not accidental**:

- **`Encode` is a total function** from a value of type `T` plus a buffer to
  bytes in that buffer. The same `T` always produces the same bytes. No
  error path. If you call `Write::write` on `T`, you will not get a
  half-written buffer. (Implementation-wise, the buffer might run out of
  capacity, but that's a buffer concern, not a `T` concern.)

- **`Decode` is a partial function** from bytes to a `T`. It can fail for
  many reasons: not enough bytes, malformed varint, length prefix out of
  range, value violates an invariant. The error is non-trivial because
  the bytes could be **anything**, including bytes crafted by an
  adversary.

This asymmetry has a name in type theory: **product types, not sum types**.
The encode direction is a "read-only" view of `T` — given a value, emit
its witness. The decode direction is "constructive" — given a witness,
build a value, but the witness might be invalid.

### The `Cfg` is what makes decoding safe

Look at `Read::Cfg`. It is *not* part of the encoded bytes; it is supplied
at decode time. For `u32`, `Cfg = ()` because every 4 bytes is a valid
`u32`. For `Vec<u8>`, `Cfg = (RangeCfg<usize>, ())` — you supply the
allowable length range. For `HashMap<K, V>`, `Cfg = (RangeCfg<usize>, K::Cfg, V::Cfg)`.

This is the **decode-time bounds** pattern from chapter 03 main text, but
now seen from a type theory angle: the `Cfg` parameter lets the function
type of `read_cfg` be **parametric** in the safety policy. The same type
can be safely decoded under different policies by supplying different
`Cfg`s.

In effect: **`Encode` is parametric over the type but not over policy;
`Decode` is parametric over both.** A nice reflection of the trust
asymmetry between encoder and decoder.

### Variance and lifetime subtleties

If you write generic code that holds a `T: Encode`, you'll sometimes see
this bound:

```rust
where T: for<'a> Read<Cfg = &'a [u8]>
```

That's a **higher-ranked trait bound (HRTB)**: `T::read_cfg` must work for
*any* lifetime `'a`, not just one specific lifetime. This is needed when
the configuration is borrowed from the input buffer (e.g., a slice that
points into the buffer being decoded). The trait must accept any
lifetime the caller provides.

Commonware's codec uses this pattern for its `Bytes` newtype and similar
zero-copy types, where the decoded value's lifetime is tied to the input
buffer's lifetime.

You won't usually write HRTBs by hand; if you see `for<'a>` in a `where`
clause, it's because some trait was designed to be flexible over lifetimes.

## Self-describing vs non-self-describing formats

Codec is **non-self-describing**. That means: given a stream of bytes,
you cannot decode it without knowing the type `T` you expected.

Contrast with **self-describing** formats like JSON, XML, or MessagePack:

```
# JSON: the schema is in the bytes
{"name": "Alice", "age": 30}

# Protobuf wire format: NOT in the bytes
0x08 0x1e    # field 1 (age), varint value 30
0x12 0x05 0x41 0x6c 0x69 0x63 0x65   # field 2 (name), length-prefixed string "Alice"
```

In JSON, the receiver sees `"name"` and `"age"` as strings, knows to look
for `"Alice"` and `30`. In Protobuf, the receiver sees byte `0x08` —
which it must interpret as "field number 1, wire type 0 (varint)." The
field *name* is gone; only the field *number* is in the wire. To decode
correctly, you need the `.proto` schema.

Commonware's codec is **further** along this spectrum: the field numbers
themselves are not in the bytes. The wire format is just a sequence of
fixed types in a fixed order. To decode, you need the type definition.

### Trade-offs

| Property | Self-describing (JSON) | Non-self-describing (Codec) |
|---|---|---|
| Readable by humans | Yes | No |
| Decode without schema | Yes | No |
| Wire size | Large (field names, tags) | Small (no metadata) |
| Schema evolution | Add fields freely | Must coordinate with all peers |
| Concatenation safety | High (delimiters in the bytes) | Low (must add framing) |
| Type safety at decode | Loose (cast strings to anything) | Strict (the type IS the schema) |

The codec's choice of non-self-describing is **an explicit trade**. It buys:

1. **Minimum wire size.** No field tags, no type names, no schema URL.
   For a 96-byte BLS signature, the bytes are the bytes.
2. **Zero ambiguity.** No "is this `i32` or `i64`?" The type is known.
3. **Compile-time guarantee.** If the Rust type compiles, the wire format
   is well-defined. No runtime schema validation.

It costs:

1. **Schema evolution requires type-level versioning.** You can't add
   fields without coordinating with all peers.
2. **Schema documentation must be external.** Commonware relies on
   `conformance.toml` to pin down the exact bytes.
3. **You can't pipe `MyMessage` and `YourMessage` over the same channel
   without a length prefix.** Self-describing formats handle this
   implicitly.

The deciding factor: in a BFT consensus protocol, you want **the smallest
possible wire format** because the network is your bottleneck. Every byte
of schema overhead is a byte of bandwidth spent on metadata instead of
work.

## Schema evolution — adding fields without breaking the world

A non-self-describing format has a tricky property: if you add a field to
the type, old readers don't know how to skip it. Old readers might
interpret the new field's bytes as the next field's bytes, which would
be wrong.

But field additions are sometimes unavoidable. Commonware handles this
via the **stability** annotation system (from `AGENTS.md`):

```rust
#[stability(BETA)]
pub struct MyMessage {
    pub a: u32,
    pub b: u8,
}
```

When you change `MyMessage` from `(a: u32, b: u8)` to `(a: u32, b: u8, c: u16)`:

- At ALPHA stability: breaking changes are expected; no migration needed.
- At BETA or higher: **a migration path is required.** The CI fails the
  build (`just check-stability`), forcing you to provide a translation
  between the old and new encodings.

This isn't schema evolution in the Protobuf sense. It's stricter: you
either don't break, or you explicitly add migration logic.

### The Protobuf-style alternative

If Commonware wanted Protobuf-style evolution, the wire format would have
to encode field numbers:

```rust
impl Write for MyMessage {
    fn write(&self, buf: &mut impl BufMut) {
        (1u32, &self.a).write(buf);   // tag 1, value a
        (2u32, &self.b).write(buf);   // tag 2, value b
        // tag 3 (c) absent - new field
    }
}
```

Then old readers, given a new payload, see tag 1 and tag 2 (the fields
they know about) and can skip tag 3 (unknown). This is **forward
compatibility**: old code reads new data. The reverse — new code reads
old data — is **backward compatibility** and requires that the new code
default missing fields to something reasonable.

Commonware deliberately does not take this path. The reasoning:

1. **Forward compat is a constant overhead.** Every field gets a tag, every
   decode gets a switch. Bytes-per-message goes up.
2. **Backward compat is a constant headache.** What's the default for a
   missing field? Different types have different defaults (zero, empty
   list, None, etc.), and getting it wrong silently corrupts state.
3. **In BFT, you upgrade as a group.** Coordinated upgrades are the norm.
   Schema-less evolution gives you the smallest bytes; you pay for
   coordination instead.

When you really do need evolution (e.g., a 6-month migration window), the
recommended pattern is to add a `version: u8` field at the start of the
message and use the conformance harness to enforce the boundary.

### Defaults

Self-describing formats allow you to skip missing fields; non-self-
describing formats either refuse or assume zero. Commonware assumes zero
at the **decode-time `Default` bound**, but does not enforce this for
production types — it's the developer's job to provide a `Default` if
they want optional fields.

## Comparison table — when to use what

| Format | Self-describing | Sizes | Schema | Use case |
|---|---|---|---|---|
| **Codec (Commonware)** | No | Smallest (no tags) | Rust type = schema | BFT consensus, adversarial network |
| **Bincode** | No | Small | Rust type = schema | IPC, in-process caching, same-arch disk |
| **Postcard** | No | Small (varint) | Rust type = schema | Embedded systems, constrained devices |
| **Protobuf** | Partial (field numbers) | Medium | `.proto` schema | Cross-language, slow-evolution systems |
| **MessagePack** | Yes | Medium | Implicit | Dynamic-typed APIs, JSON-like with types |
| **FlatBuffers** | No (zero-copy) | Medium | `.fbs` schema | Game engines, in-place reads |
| **CBOR** | Yes | Medium | Self-describing | IoT, similar to JSON but binary |
| **JSON** | Yes | Largest | None | Logs, public APIs, debugging |

**When to use Commonware's codec:**

- You're shipping bytes across a network where you don't trust the receiver.
- You want the smallest possible wire size for repeated types (signatures,
  certificates, votes).
- Your Rust types are your schema; you don't need cross-language support.
- You want the conformance harness to catch drift.

**When NOT to use Commonware's codec:**

- You need cross-language support (use Protobuf or FlatBuffers).
- Your types evolve often and you can't coordinate upgrades (use Protobuf
  with field tags).
- You want zero-copy access to nested fields (use FlatBuffers, which lets
  you read a field without decoding the whole message).
- You're caching data in-process and want to avoid the codec overhead
  (use Bincode or raw memory).

The unifying principle: **codec is for adversarial networks with
coordinated upgrades**. Anything else, pick a format that matches your
operational reality.

## Length-prefixed framing — why every record has its size

The wire format for a `Vec<T>` or any composite structure is:

```
[varint length][record 1][record 2]...[record N]
```

Why? Three reasons:

### Reason 1: Streaming-friendly

Without a length prefix, the receiver doesn't know when to stop reading.
They'd need either:
- A delimiter (and the bytes must not contain the delimiter), or
- A fixed-size record, or
- A length prefix.

Length prefixes win because they don't constrain the payload's contents
and don't require fixed sizes. A receiver can allocate exactly the right
buffer, then read `length` more bytes, then decode.

### Reason 2: Concatenation-safe

Two messages can be sent back-to-back on a stream if and only if the
receiver can find the boundary between them. A length prefix makes this
trivial: read length, skip those bytes, the next message starts at the
next byte.

Without a length prefix, you'd need to **parse** to find the boundary.
Parse one message, check if more bytes are queued, parse another, etc.
This works but couples parsing to framing. Length-prefixed framing
**separates** the two concerns.

Codec's `Vec<T>::Write` writes `[length][items]`. On the read side,
`Vec<T>::Read` reads length, validates against `RangeCfg`, allocates,
fills. Done.

### Reason 3: Safety under untrusted input

The `RangeCfg` (chapter 03 main text, `codec/src/config.rs`) turns the
length prefix into a **safety check**. A malicious peer sends a length
of `2^60`. The decoder sees this, rejects it because it's outside the
range, and the attack fails.

Without length-prefixed framing, a peer could craft a message that's
"valid" but enormous. The decoder might buffer gigabytes before realising
the message is too big. With length-prefixed framing, the malicious
length is rejected **before any allocation**.

This is the same reason `Vec<T>::write` includes the length even when the
underlying `T` is fixed-size: it's a defensive framing, not a redundancy.

### The streaming Decoder

Look at `varint.rs:90-93` (`Decoder<U: UPrim>`). For streaming protocols
where you receive bytes one at a time, you don't have a full buffer to
feed `usize::read`. Instead, you feed bytes to the `Decoder` one at a
time and it tells you when the varint is complete.

This is what `Stream::recv(len)` does (chapter 02): read bytes until the
varint is done, then read `length` more bytes, then hand the full record
to the application layer.

The cost of this style: an extra state machine. The benefit: arbitrary
fragmentation. TCP can split a single message across many reads; the
`Decoder` reassembles it.

## Rust patterns you'll see in the codec

A few patterns are worth flagging because you'll see them again and
again in Commonware code.

### `PhantomData<T>` for unused type parameters

Some types are generic over `T` but don't actually store a `T`:

```rust
pub struct MyCodec<T> {
    config: Config,
    _phantom: PhantomData<T>,
}
```

Why? To let the type system enforce that `MyCodec<X>` is distinct from
`MyCodec<Y>` even when the field types are the same. Without
`PhantomData`, the compiler complains about unused type parameter `T`.

`PhantomData<T>` is zero-cost: it takes no space. It's a marker for the
type system only. Common variants:

- `PhantomData<T>` — "I logically own a `T`" (affects variance and drop
  checking).
- `PhantomData<fn() -> T>` — "I'd return a `T` if asked" (covariant, no
  drop check).
- `PhantomData<*const T>` — "I'd reference a `T`" (invariant, no drop
  check).

Most codec code uses `PhantomData<T>` or `PhantomData<fn() -> T>`. The
distinction matters when you have type-parameter drop semantics; codec
usually doesn't, so the variance-friendly variants are common.

### `where T: Codec` and variance

When you write `T: Codec`, you're saying `T` can be encoded and decoded.
The compiler will infer variance from how `T` is used:

- `&T` in a struct field — covariant (you can pass a `&'long T` where a
  `&'short T` is expected).
- `&mut T` in a struct field — invariant (you can't substitute).
- `T` by value (e.g., a `Vec<T>`) — covariant in the `T` parameter but
  the field is invariant in lifetime.

For codec types, you'll typically see `&T` (read-only encode) and
`T::read_cfg(buf, &cfg)` (constructive decode). Variance rarely matters
in practice for codec code; it's a footnote.

### Higher-ranked trait bounds (HRTBs)

If you see this:

```rust
where T: for<'a> Read<Cfg = &'a [u8]>
```

That's a **higher-ranked trait bound**: the bound must hold for *every*
lifetime `'a`, not just one specific lifetime. HRTBs appear when the
configuration type borrows from the input buffer. The `&'a [u8]` config
means: "I have a slice borrowed from somewhere with lifetime `'a`; please
support that lifetime."

You'll usually see this when defining a `Bytes` newtype or other
zero-copy decoded type whose lifetime is tied to the input.

### Default type parameters

Some traits have default type parameters, like:

```rust
pub trait Read: Sized {
    type Cfg: Clone + Send + Sync + 'static = ();
    ...
}
```

The `= ()` says "if you don't specify a Cfg, it's `()`." Most codec
implementations just use the default. Types that need a Cfg (like
`Vec<T>`) override it.

### `impl Trait` in trait return positions

`fn write(&self, buf: &mut impl BufMut)` uses `impl Trait` to say "any
type that implements `BufMut`." This is sugar for a generic parameter.

A more advanced variant: `fn write(&self) -> impl BufMut`. Commonware
doesn't use this often, but you'll see it in codec adapters.

## A formal view of Encode and Decode

The codec trait pair looks mundane: one function turns a value into
bytes, another turns bytes into a value. But if you sit with it for
ten minutes, it starts to feel like one of the most interesting
type signatures in the entire codebase. It is doing real work: it is
encoding the boundary between the **trusting world** (your process,
where types are sound and values mean what their types say) and the
**untrusting world** (the network, the disk, the future, the past)
where types are a fiction enforced only by your ability to parse
bytes.

This section treats `Encode` and `Decode` as the *type-theoretic
machines* they really are.

### `Encode` is a total function `T → bytes`

Strip away the syntax and `Encode` is just a function. Given a value
of type `T`, it produces a sequence of bytes that represents `T`.

```
Encode::encode : T → Vec<u8>
```

Two properties make this function *good*:

1. **Totality.** `Encode` cannot fail. Every value of `T` produces
   some byte string. There is no `Result` because there is nothing
   that can go wrong — encoding is purely a transformation from an
   in-memory value into an in-memory byte buffer. If your `Encode`
   impl can panic, that is a bug.

2. **Determinism.** For the same `T` and the same value, you always
   get the same bytes. `Encode` is a *pure* function (modulo
   mutable state captured by the implementation, which Commonware
   forbids — there is no `&mut self`).

Together, totality and determinism mean `Encode` is a *code* in the
information-theoretic sense: a bijection between the set of
distinguishable values of `T` and a subset of `Vec<u8>`.

### `Decode` is a partial function `bytes → Result<T, Error>`

`Decode` is the harder sibling. It is *not* total: there exist byte
strings that are not valid encodings of any `T`. Those strings are
the ones `Decode` rejects.

```
Decode::decode : &[u8] → Result<T, Error>
```

Crucially, `Decode` is also *not* a left inverse of `Encode` for
arbitrary input. There are at least three classes of inputs where
`Decode(bytes) = Err(_)`:

1. **Truncated input.** You tried to decode `Vec<u8>` but only got 3
   bytes and the varint says the length is 1000.
2. **Structurally invalid.** You tried to decode a `bool` and got
   byte `0x05`. There is no `bool` for that.
3. **Semantically invalid.** You tried to decode a `Block` whose
   `parent_hash` refers to a block your ledger does not know about.
   The bytes parse, but the value is rejected by the application
   logic.

`Decode` returns `Err(_)` for the first two cases. The third case is
the application's responsibility — `Decode` doesn't know what
"valid" means in your domain.

The **partiality** of `Decode` is the entire reason it returns
`Result` instead of `T`. If `Decode` were total (no `Result`),
then `Decode::decode(&[])` would have to return *some* `T`, and
that `T` would have to be a sensible default — but there is no
sensible default for most types. Empty bytes do not encode any
non-trivial value. So `Decode` is forced to be partial.

### Why partiality matters

If you are a careful Rust programmer, you already know to handle
`Result`. But here is the deeper reason partiality is *central* to
how Commonware's codec works:

- The codec is **untrusted input**. Bytes come in from the network
  and from disk. Both can be corrupted, malicious, truncated, or
  from an old version of the protocol. A *partial* `Decode` lets the
  system say "no, this is not a valid encoding of any `T` I know
  about" without panicking or producing garbage.

- Partiality is a **type-level commitment to the unknown**. The
  return type `Result<T, Error>` says, *out loud*, "there are byte
  strings from which no `T` can be reconstructed." That commitment
  is what makes the rest of the system safe.

- A codec that returns `T` directly (no `Result`) would force you to
  invent a sentinel for every error case. For a `u32`, that might
  work (`0` is unused). For a `Vec<Block>`, it is impossible. So
  `Result` is the only honest signature.

This is why the trait is `Decode`, not `DecodeOrPanic` or `TryDecode`.
The return type is the trait.

### A type-level state machine

Look at `read_cfg` carefully:

```rust
fn read_cfg(buf: &impl Buf, cfg: &Self::Cfg) -> Result<Self, Error>;
```

The fact that it takes `&impl Buf` (anything implementing `Buf`)
suggests you could read a value from any byte source. But here is
the catch: in practice, you almost always call it with a `&[u8]` or
a `Reader` over a stream. Why does the signature accept a generic
`Buf`?

Because `Buf` is the trait for "any byte source I can advance
through." `Reader` is `Buf`. `Bytes` is `Buf`. A cursor wrapping a
shared buffer is `Buf`. By accepting `&impl Buf`, the codec layer
sits *above* the choice of byte source.

There is also a deeper, less obvious reason. The codec's `read`
operation is, in spirit, a *state machine*:

```
    +-----------------+    read    +------------+
    | Initial: buf@0  | ---------> | After:    |
    |                 |            | buf@size  |
    +-----------------+            +------------+
            |                            |
            |   partial reads            |   exhausted
            v                            v
      Error::Short                     Ok(T)
```

Each call to `read` either consumes some bytes and produces a `T`
(with the buffer advanced), or refuses the bytes (with the buffer
untouched). This is the same shape as every parser combinator
library. Commonware just doesn't expose it as `State<S>` — it
exposes it as `(&Buf) -> Result<T, Error>`, which is morally
equivalent.

### Generic associated types (GATs)

The trait has:

```rust
trait Read: Sized {
    type Cfg;
    fn read_cfg(buf: &impl Buf, cfg: &Self::Cfg) -> Result<Self, Error>;
}
```

`Self::Cfg` is a *generic associated type*. In pre-GAT Rust
(before 1.65), you would have had to write something like:

```rust
trait ReadFamily {
    type Cfg;
    fn read_cfg<T: ...>(buf: &impl Buf, cfg: &Self::Cfg) -> Result<T, Error>;
}
```

and pass `T` as a separate generic parameter. GATs let the
associated type be *attached to the impl for `Self`*, which is what
you want — `Cfg` is a property of `T`, not of the wire.

Why does this matter? Because `Vec<T>::Cfg` should be `T::Cfg` (or
some composition of it). With GATs:

```rust
impl<T: Read> Read for Vec<T> {
    type Cfg = T::Cfg;
    ...
}
```

is the natural thing to write. Without GATs, you would have to
define a separate family trait, plus an `impl` per type, and
every call site would have to thread the type parameter through.

GATs are not just a convenience. They let the trait *have a
configuration knob* without exploding the type-parameter graph.

### Variance — covariant, contravariant, invariant

`Self::Cfg` is *invariant*. That means if you have a value of type
`Vec<T>` and you try to substitute `Cfg = T::Cfg` with `Cfg =
&'static T::Cfg` or `Cfg = T::Cfg<'a>`, the compiler will refuse.
This is *correct* for a configuration parameter: a wider or narrower
configuration is a different configuration, and the encoding rules
depend on the exact configuration.

Let me explain what variance *means* in this context.

A type `F<T>` is **covariant** in `T` if `T <: U` implies `F<T> <:
F<U>`. Read: "if `T` is a subtype of `U`, then `F<T>` is a subtype
of `F<U>`." Most "container" types in Rust (`Vec<T>`, `Box<T>`) are
covariant: `Vec<Cat>` can be used wherever `Vec<Animal>` is
expected, because `Cat <: Animal`.

A type `F<T>` is **contravariant** in `T` if `T <: U` implies `F<U>
<: F<T>`. The classic example is `fn(T) -> ()`: a function that
accepts `Animal` can be used wherever a function that accepts `Cat`
is expected, because the `Animal` function can also handle `Cat`.

A type `F<T>` is **invariant** in `T` if neither holds — `F<T>` and
`F<U>` are unrelated when `T ≠ U`. Mutable references (`&mut T`),
`*mut T`, and most configuration parameters are invariant.

For `Cfg`, invariance is exactly what you want. The configuration
parameter is the *protocol version*, the *maximum allowable length*,
the *validator set*. These are not substitutable for one another.
A protocol version of 2 cannot be coerced into a protocol version of
3. A maximum length of 1000 is not interchangeable with a maximum
length of 10000. Invariance in `Cfg` is the type system telling
you, *statically*, "you cannot accidentally substitute one config
for another."

If `Cfg` were covariant, you could end up encoding with config A but
the receiver expecting config B, and the two might disagree on what
a valid `T` is. Invariance prevents this class of bug at compile
time.

### `Read` has a lifetime; `Encode` does not

The two traits have subtly different signatures:

```rust
trait Encode {
    fn encode(&self, buf: &mut impl BufMut);     // no lifetime on output
    fn encode_size(&self) -> usize;
    fn write_bufs(&self) -> impl WriteBuf;        // zero-copy
}

trait Read: Sized {
    type Cfg;
    fn read_cfg(buf: &impl Buf, cfg: &Self::Cfg) -> Result<Self, Error>;
    //                                                       ^^^^
    //                                                       no lifetime
}
```

`Read` returns `Self` by value (no lifetime), and the input `&impl
Buf` is also a borrow. So both are tied to the borrow `buf`, but
the returned value is owned. Why?

Because `Read` *must* produce an owned value. If it produced a
borrowed value, the borrow's lifetime would be tied to the input
buffer's lifetime, and the input buffer goes away as soon as the
caller drops it. So the returned value would be unusable.

`Encode`, by contrast, takes `&self` (the borrowed input) and writes
into `buf: &mut impl BufMut`. There is no `Encode::encode(&self) ->
Vec<u8>` because we want the *caller* to control allocation, not
the trait.

The asymmetry (input borrowed on encode, output owned on decode) is
not accidental. It is the right shape for a streaming codec: the
encoder wants to write into a buffer the caller chose (zero alloc),
and the decoder wants to produce a value the caller owns (zero
borrow).

### The Cfg type parameter as a "schema"

If you squint, `Cfg` is doing the same job as a *schema* in a typed
data system. A schema says: "these are the types, in this order,
with these widths." `Cfg` says: "here are the maximum lengths, the
protocol version, the limits."

For most types (`u32`, `[u8; 32]`, `bool`), `Cfg = ()`. There is
nothing to configure.

For `Vec<T>`, `Cfg = T::Cfg` (or, in some cases, a richer struct
that includes a max-length bound).

For application-level types like `Block`, `Cfg` is typically a
`(protocol_version: u32, max_block_size: u32)` tuple — or the
application defines its own `BlockCfg` struct.

This is Commonware's answer to Protobuf's `Descriptor` or
FlatBuffers' `.fbs` file: a type-level description of the protocol
shape, propagated through the codec. The difference is that
Commonware's Cfg is a *Rust type*, not a separate IDL. So the
schema lives in the same module as the impl, and the compiler
verifies that the schema is consistent.

### Why not just pass `cfg` as a runtime parameter?

You might wonder why `Cfg` is a *type* and not just a struct passed
at runtime. The answer: types are checked by the compiler; runtime
parameters are checked by you. If `Cfg` is a type, then a function
that takes a `Vec<u8>` cannot accidentally call it with a `Vec<u32>`
whose Cfg is a different shape. The compiler enforces the
relationship.

If `Cfg` were a runtime struct, you would have to write defensive
code everywhere: "what if the runtime Cfg has a max-length that
disagrees with what I expected?" With types, the question never
arises.

This is the same logic that drives `PhantomData<T>` and the entire
typestate pattern in Rust: move the invariant from runtime to
compile time.

### Recap

- `Encode` is total, deterministic, `&self → &mut BufMut`. It is a
  *code*.
- `Decode` is partial, `&Buf → Result<T, Error>`. Partiality is
  *structural*: empty bytes do not encode a value.
- `Cfg` is a *generic associated type* that lets the trait carry
  configuration without an explosion of generic parameters.
- `Cfg` is *invariant* on purpose: configuration values are not
  substitutable.
- `Read` returns owned values (no lifetime) because the alternative
  is a borrow with the wrong lifetime; `Encode` writes into a
  caller-provided buffer for the same reason.
- `Cfg` is Commonware's type-level "schema" — the answer to
  Protobuf's `.proto` and FlatBuffers' `.fbs`.


## Schema evolution as a formal protocol

Chapter 03 already covers the basics: a "stable" message type uses
the `#[stability(BETA)]` (or higher) attribute, and adding a field
to a stable type is a backward- and forward-compatible change.
This section goes deeper. We treat schema evolution as a *protocol*
— a set of rules that two independently-deployed processes
follow — and we use that lens to compare Commonware's approach to
the way Protobuf, FlatBuffers, and Avro handle the same problem.

### The problem, restated

You ship `BlockV1`. Six months later, you want to ship `BlockV2`
with an extra field. The old software is still running. You cannot
force everyone to upgrade at once. So:

- Old nodes must keep working when they receive `BlockV2` messages
  from new nodes. (Backward compat — old reader, new writer.)
- New nodes must keep working when they receive `BlockV1` messages
  from old nodes that haven't upgraded yet. (Forward compat — new
  reader, old writer.)
- In the best case, both. (Full compat — old reader ↔ new writer,
  in either direction.)

This is the *cross-version* requirement. It is the central
constraint of any long-lived wire protocol.

### Backward compatibility — old reader, new writer

The old reader doesn't know about the new field. So the new writer
must put the new field somewhere the old reader will ignore. There
are two ways:

1. **Append-only.** Put the new field at the *end* of the encoding.
   The old reader parses the fields it knows and stops. Anything
   after the end is either ignored or causes a parse error.

   This is what Protobuf does: fields are tagged with a field
   number, and the old reader just skips unknown field numbers.

2. **Default-on-read.** If the old reader expects a *count* (e.g.,
   "how many fields follow"), bump the count and let the old reader
   notice it has more fields than it knows about, then ignore the
   extras.

   This is what XDR and early JSON-RPC variants did.

Commonware uses strategy 1 for `#[stability(BETA)]` types: you
append a field, and the old reader parses up to the field count it
expects, then ignores the rest. (The exact mechanics are described
in the existing schema-evolution section in the main body.)

### Forward compatibility — new reader, old writer

The new reader doesn't know the old writer might omit a field it
expects. So:

- If the field has a *default value* (e.g., 0 for a `u32`), the new
  reader uses the default when the field is absent.
- If the field has no default, the new reader rejects the message.
  This is a forward-incompatible change — new software can't talk
  to old software.

Commonware's approach: every field in a `#[stability(BETA)]` struct
*must* have a `#[serde(default)]`-like default in the codec. If you
add a field without a default, the `conformance_tests!` macro will
warn (and possibly fail).

### Full compatibility — both

If you can append a field with a default and the old reader ignores
the unknown tag, you get full compatibility: old ↔ new, in either
direction. This is the gold standard.

The Protobuf community's mantra is: "never change the tag number,
never change the wire type, never reuse a tag number with a
different meaning." If you follow that, your messages evolve
gracefully for years.

### Protobuf field tags in detail

Protobuf's wire format is a sequence of `(tag, value)` pairs. The
tag is a varint that packs a field number and a wire type. Wire
types are: `VARINT`, `FIXED64`, `LENGTH_DELIMITED`, `FIXED32`,
`START_GROUP`, `END_GROUP`, plus the now-deprecated
`START_GROUP`/`END_GROUP`.

A Protobuf decoder scans the byte stream, reads each `(tag, value)`
pair, and dispatches based on the tag. Unknown field numbers are
skipped (their length is encoded for `LENGTH_DELIMITED` fields, so
the skip is bounded).

This is why Protobuf's evolution story is "field tags are forever."
You can add tag 17 and remove tag 3 (reserving 3's tag number from
reuse), and the world keeps working.

Commonware's approach is similar in spirit (fields are positional,
not tagged, but new fields go at the end). The mechanics differ:
Commonware uses *positional* ordering (the struct field order
matters), while Protobuf uses *tagged* ordering (field numbers
matter). Commonware's choice is simpler but less flexible — you
can't reorder fields without breaking compatibility. Commonware's
docs explicitly warn against reordering.

### FlatBuffers schema evolution

FlatBuffers takes a different approach: the schema is the
`.fbs` file, and the binary format is *generated* from the schema.
Adding a field is a backward-compatible change (the old reader
doesn't know about it), but only if you append it. Removing a
field is also backward-compatible (the old reader ignores
unknown fields, even if it knows the field's tag).

FlatBuffers is more flexible than Protobuf for some use cases: it
supports nested tables (objects with variable-length fields), and
the binary format supports random access (you can read a field
without parsing the whole message). Commonware's codec does *not*
support random access — decoding a `Vec<Block>` requires decoding
every `Block`. This is a deliberate trade-off: random access is
expensive (size overhead per field) and Commonware optimizes for
*streaming* decode.

### Avro schema evolution

Avro's evolution story is the most interesting because Avro is
*self-describing*: every message includes a schema ID, and the
writer's schema and the reader's schema are both resolved at
decode time. Avro can apply a *projection* from the writer's schema
to the reader's schema, automatically filling in default values
for missing fields.

Commonware does not do this. The `Cfg` parameter is the schema, and
two versions of the same type with different `Cfg`s will not
*interoperate* — they will produce different byte streams, and a
mix-up will fail at decode time.

This is a deliberate trade-off. Avro's projection is flexible but
expensive: every message carries a schema ID, and the reader must
have the schema. Commonware's `Cfg` is implicit (the reader knows
its own `Cfg` by construction), so there is no per-message
overhead.

### Worked example: adding a field

Suppose you have:

```rust
#[derive(Encode, Decode)]
#[stability(BETA)]
pub struct BlockV1 {
    pub height: u64,
    pub parent: Hash,
    pub payload: Vec<u8>,
}
```

You want to add `proposer: PublicKey` for a new consensus protocol.

**Step 1.** Add the field with a default:

```rust
#[derive(Encode, Decode)]
#[stability(BETA)]
pub struct BlockV2 {
    pub height: u64,
    pub parent: Hash,
    pub payload: Vec<u8>,
    #[codec(default)]   // ← mandatory for new fields
    pub proposer: PublicKey,
}
```

The `#[codec(default)]` attribute tells the codec to use
`PublicKey::default()` (or whatever `Default` impl you provide) when
the field is absent.

**Step 2.** Regenerate the conformance file:

```bash
just regenerate-conformance
```

This re-encodes the test values with the new code, updates
`conformance.toml`, and you commit the change.

**Step 3.** On rollout:

- Old nodes (still running `BlockV1`) parse `BlockV2` messages by
  decoding the fields they know (`height`, `parent`, `payload`) and
  ignoring the trailing `proposer`. They never look at it.
- New nodes (running `BlockV2`) parse `BlockV1` messages by
  decoding the fields they know and using `PublicKey::default()`
  for `proposer`.

Both directions work. This is full compatibility.

### Worked example: removing a field

You can't remove a field that was in a stable type. Once a field
has been on the wire, old readers expect to find it. If you remove
it, an old reader will mis-parse the message (it'll read the
"wrong" field as the now-removed one's value).

The Commonware docs say: deprecate, don't remove. Mark the field
`#[codec(deprecated)]` and continue to encode/decode it. It will
take up space forever.

### Worked example: changing a field's type

You cannot change a field's type in a stable type. If `height:
u64` becomes `height: u128`, an old reader will mis-parse
(reading 16 bytes where it expected 8) or reject the message.

If you need a type change, you must introduce a new message
(`BlockV2`) and migrate consumers via a version negotiation
handshake. Commonware does not do automatic type migration.

### The protocol as a state machine

If you step back, schema evolution is a *state machine* over the
type's version:

```
        add field (with default)
   BlockV1  ------------------->  BlockV2
       ^                              ^
       |                              |
       |       drop field             |
       |     (NOT ALLOWED)            |
       |                              |
       +----- can rollback if no ----+
              field was used
```

The transitions are:
- **add field (with default)**: backward- and forward-compatible.
- **remove field**: not allowed.
- **change field type**: not allowed; requires a new message.
- **rename field**: allowed if the rename is purely a Rust-level
  change (the wire-level field order is unchanged). The codec uses
  positional ordering, so renaming a field is fine as long as you
  don't reorder.

### Why Commonware's design is the way it is

Commonware chose a *simple, conservative* evolution model because:

1. **BFT protocols need predictable behavior.** If a field can
   disappear, the consensus state machine has to handle missing
   fields everywhere. Commonware says: no, fields don't disappear.

2. **The codec is small and trusted.** Commonware's codec is a
   ~1,000-line file. A schema-evolution policy encoded in
   attribute macros is auditable in a day. Protobuf's evolution
   policy is encoded in dozens of `.proto` files plus
   `protoc`'s output — auditable in a week.

3. **Test-first.** Every change to a stable type is gated by the
   conformance test. The conformance file is the source of truth
   for the wire format. If you change the wire format, you change
   the conformance file, and reviewers see the diff.

The trade-off: Commonware's evolution model is *less flexible*
than Protobuf's. You can't reorder fields, you can't remove them,
you can't change their types. But it is *simpler and easier to
reason about*, which is the right trade-off for a BFT library
where correctness matters more than flexibility.


## The wire-format decision matrix

This section is the long version of the existing "when to use what"
comparison. The original is a table; this is a tour. We pick
seven wire formats that are likely to come up in a Rust project,
explain the implementation story for each, and end with a decision
matrix that helps you pick.

### Bincode — the naive default

`bincode` is the wire format that most Rust tutorials reach for.
The implementation is short, the integration is `#[derive]`, the
performance is OK. The problem is the *evolution story*: bincode
1.x is not stable. The byte format depends on the order of struct
fields, the choice of integer types, and the specific version of
bincode you used to encode.

For an internal tool that reads its own writes within a single
binary, bincode is fine. For a long-lived wire protocol where the
binary on the wire today must be decodable by next year's code,
bincode is dangerous.

**Implementation cost:** trivial. `#[derive(Encode, Decode)]`.

**Wire format:** raw little-endian, length-prefixed for slices,
no tag numbers.

**Compatibility:** none across versions. A struct reordering
breaks decode.

**When to use:** scratch serialization in tests, single-process
state dumps, internal RPCs in a single binary.

### Postcard — the modernized bincode

`postcard` is a bincode descendant that takes the design more
seriously. It uses varint-encoded lengths (like Commonware's
`usize` encoding) and is stable across versions of the library.
Its evolution story is "append-only fields," like Commonware's.

Postcard is what `bincode` should have been. It is small, fast, and
has a credible evolution story. The catch: it doesn't have a
type-level `Cfg` like Commonware, so configuration (like max
length) is a runtime parameter, not a type.

**Implementation cost:** trivial. `#[derive(Encode, Decode)]`.

**Wire format:** varint lengths, explicit endianness (configurable,
defaults to native).

**Compatibility:** backward-compatible within major versions (the
"append-only fields" rule applies).

**When to use:** embedded serialization, embedded RPCs, sensor
networks, anything where Commonware's stricter model is overkill.

### Commonware Codec — the long-lived BFT choice

The codec you are reading about. It is the most disciplined of the
lot: explicit endianness (big-endian, always), type-level `Cfg`,
canonical encoding (no lazy zero-padding), incremental decoders
for streaming, conformance tests as the source of truth.

The cost: every type needs an explicit `Encode`/`Decode` impl (or a
derive that supports the `Cfg` parameter, which is a bit fiddly for
types that take a `Cfg`). The benefit: the wire format is fixed,
auditable, and frozen across deployments.

**Implementation cost:** moderate. `#[derive]` works for most types,
but you must be careful with `Cfg`.

**Wire format:** big-endian, varint lengths (capped at u32/i32),
canonical, deterministic.

**Compatibility:** backward-compatible within `#[stability(BETA)]`
(or higher) types; the evolution rules are precise.

**When to use:** long-lived wire protocols, BFT consensus,
cross-version compatibility for years.

### Protobuf — the cross-language interop choice

`protobuf` is the wire format of the internet. It is the only one
in this list with first-class support for 20+ languages, with a
formal schema language (`.proto`), and with industry-standard
tooling (`protoc`, `buf`, `grpc`).

The cost is the build system. `protobuf-rs` requires you to invoke
`protoc` (or a Rust-native parser) on every `.proto` file, and the
generated code is large and verbose. The benefit is portability:
your Rust service can talk to your Python service, your Go service,
your C++ service, all with the same wire format.

The Rust codegen has historically lagged behind the Go and C++
codegen, but the gap is closing. `prost` is a clean, idiomatic
codegen that produces normal-looking Rust types.

**Implementation cost:** high. `.proto` files, `build.rs`,
`prost`/`protobuf-rs` integration.

**Wire format:** tagged, varint, length-delimited, schema-driven.

**Compatibility:** very good within the schema. Field tags are
forever; new fields can be added freely.

**When to use:** cross-language RPC, public APIs, anything that
needs to integrate with non-Rust clients.

### MessagePack — the JSON-like compact format

`msgpack` is a self-describing binary format that is essentially
JSON with shorter encodings for common values. The wire format
includes type tags in every byte sequence, so it is parseable
without a schema.

The trade-off: the type tags cost bytes (an integer takes 1 byte in
Commonware, 1-9 bytes in msgpack depending on size), and the
schema-less design means you lose the compiler's help — a typo in
a field name silently produces an `nil` at decode time.

**Implementation cost:** trivial to moderate. `serde` integration
is straightforward.

**Wire format:** self-describing, with type tags.

**Compatibility:** very good. Schema-less formats are by definition
forward- and backward-compatible as long as the data model is.

**When to use:** when you need a self-describing format but JSON is
too verbose. Less common in Rust these days because bincode /
postcard cover most of the same use cases with smaller size.

### FlatBuffers — the zero-copy choice

`flatbuffers` is the only format in this list that supports *zero-copy
deserialization*. You can read a field from the encoded bytes
without parsing the whole message. The trade-off: the wire format
is verbose (offsets everywhere), the codegen is heavy, and the
schema evolution story is good but not trivial.

The killer feature is performance: if you have a 10MB message and
you want to read one field, FlatBuffers does it in O(1) per field
access, regardless of message size. Commonware's codec, by
contrast, requires you to decode the entire message.

**Implementation cost:** high. `.fbs` files, codegen, build
integration.

**Wire format:** offset-based, table-oriented, schema-driven.

**Compatibility:** good within the schema. New fields can be
added; old fields can be deprecated.

**When to use:** game engines (the original use case), large
messages with sparse field access, scenarios where the message
size dominates parse time.

### Cap'n Proto — the flatbuffers sibling

`capnp` is the spiritual successor to FlatBuffers, designed by the
same author (Kenton Varda). It improves on FlatBuffers in several
ways: better support for streaming RPC, better tooling, a more
ergonomic Rust API.

The wire format is similar to FlatBuffers but with a richer type
system (capabilities, generics, etc.). The trade-offs are
similar: high implementation cost, verbose wire format, great
performance.

**Implementation cost:** high. `.capnp` files, codegen.

**Wire format:** offset-based, schema-driven.

**Compatibility:** good within the schema.

**When to use:** long-lived RPC frameworks, distributed systems
where message sizes are large. The `capnp-rpc` library is a
complete RPC system with capabilities.

### CBOR — the IETF self-describing format

`cbor` is the IETF's answer to msgpack. It is a self-describing
binary format designed to be a drop-in replacement for JSON in
constrained environments. It is RFC 8949.

CBOR is widely used in IoT (CoAP uses it) and in certificate
transparency (RFC 9162 uses a CBOR-flavored encoding for some
structures). It has the same trade-offs as msgpack: self-describing
means larger messages, and you lose the compiler's help.

**Implementation cost:** trivial to moderate. `ciborium` and
`serde_cbor` are the main Rust crates.

**Wire format:** self-describing, type-tagged, with deterministic
encoding mode (RFC 8949 §4.2).

**Compatibility:** very good (self-describing).

**When to use:** IETF protocols, IoT, when you need an IETF
self-describing format.

### Decision matrix

| Format              | Schema       | Endianness     | Self-desc | Streaming | Zero-copy | Rust cost | Evolve cost |
|---------------------|--------------|----------------|-----------|-----------|-----------|-----------|-------------|
| Bincode             | positional   | native         | no        | yes       | no        | trivial   | none        |
| Postcard            | positional   | configurable   | no        | yes       | no        | trivial   | append-only |
| Commonware Codec    | positional + | big (always)   | no        | yes       | partial¹  | moderate  | strict rules|
| Protobuf            | tag-based    | n/a            | partial²  | yes       | no        | high      | append-only |
| MessagePack         | n/a          | n/a            | yes       | yes       | no        | low       | trivially   |
| FlatBuffers         | offset-based | little         | no        | partial   | yes       | high      | append-only |
| Cap'n Proto         | offset-based | little         | no        | yes       | yes       | high      | append-only |
| CBOR                | n/a          | n/a            | yes       | yes       | no        | low       | trivially   |

¹ `write_bufs` is zero-copy for the encoder (writes into caller's
buffer). Decoding is not zero-copy.

² Protobuf is "schema-driven" rather than self-describing: the
schema is external to the bytes, but the field tags are in the
bytes, so a partial schema can decode.

### When to reach for each

The decision is rarely about which format is "best." It is about
which trade-offs you can afford.

- **Cross-language RPC** → Protobuf. The codegen cost is worth it
  for the cross-language portability.
- **Long-lived wire protocol with strict evolution** → Commonware
  Codec. You pay the implementation cost once and you get the
  conformance tests forever.
- **Internal serialization, single binary** → Bincode or Postcard.
- **Self-describing format, small messages** → MessagePack or
  CBOR.
- **Zero-copy access to large messages** → FlatBuffers or
  Cap'n Proto.
- **Embedded / constrained devices** → Postcard.

The Commonware Codec is the right choice when your primary
constraint is *the wire format must not change unexpectedly*. The
conformance tests enforce this; the Cfg parameter makes it
type-safe; the deterministic encoding rules make it
reproducible. If you need any of those properties, the
implementation cost is worth it.


## Length-prefixed framing: the deeper view

The existing "length-prefixed framing" section in the main body
covers the basics: every record on the wire starts with its length,
which makes the wire format concatenation-safe, streaming-friendly,
and safe under untrusted input. This section goes deeper into the
*theory* of framing and the alternatives.

### Why framing matters

A wire protocol is a *stream of records*. The question is how to
demarcate the boundaries between records. There are four common
strategies:

1. **Fixed-size records.** Every record is `N` bytes. The boundary
   is implicit: record `i` is bytes `[iN, (i+1)N)`.
2. **Delimiter-based.** Records are separated by a special byte
   sequence (`\n\n`, `<record>`, etc.).
3. **Length-prefixed.** Every record starts with a length, then the
   payload.
4. **Self-describing.** The wire format includes type tags or
   schema IDs that implicitly delimit records.

Each has trade-offs. Fixed-size is the simplest but the most
inflexible. Delimiter-based is human-readable but ambiguous
(what if the delimiter appears in the payload?). Length-prefixed
is robust but adds overhead. Self-describing is flexible but
expensive.

### Why length-prefixed is the right default

For a BFT consensus protocol, the wire records are messages from
peers. The peers are potentially malicious. So the wire format
must be *unambiguous* even when the input is adversarial.

Consider delimiter-based framing. If the protocol is
`record1 || "\n\n" || record2 || "\n\n" || ...`, then a malicious
peer can send a record that contains "\n\n" inside its payload,
and the receiver will mis-parse the next record. The fix is to
escape the delimiter in the payload, but then the escape character
itself can be ambiguous, and you have the same problem one level
up. The classic escape-handling bugs (CVE-2017-3735 in OpenSSL is
one example) come from this kind of complexity.

Length-prefixed framing has none of this. The length prefix
unambiguously says "the next `N` bytes are this record." There is
no way to construct an input that is interpreted as a different
record boundary, because the length is read before the payload.

### The streaming implication

Length-prefixed framing also lets you *stream*. You do not need to
buffer an entire message before you start decoding it. You read
the length prefix, allocate (or reserve) that many bytes, and
then read the payload.

For very large messages (say, a 10MB block), this is critical.
With delimiter-based framing, you cannot start parsing until you
see the next delimiter, which may be far away. With
length-prefixed framing, you know the size up front and can
allocate a buffer of the right size.

Commonware's incremental varint decoder (`codec/src/varint.rs`) is
the streaming piece. It reads bytes one at a time and tells you
when the varint is complete. You can compose it with a streaming
reader to decode messages without buffering the whole stream.

### Concatenation safety

Length-prefixed framing is *concatenation-safe*: you can write
`record1 || record2 || ...` to a single byte buffer, and the
receiver can decode each record independently without needing to
know where the boundaries were in the original stream.

This is why Commonware's `Write` writes length-prefixed records:
any number of records can be appended to a buffer (or a file, or a
socket), and the receiver will correctly demultiplex them.

This is *not* true for delimiter-based framing unless the delimiter
is carefully chosen and the payload is escaped. Even then, bugs
abound.

### Fixed-size records: when they are right

For records of fixed size, you don't need a length prefix. The
boundary is implicit: record `i` is at offset `i * N`.

Commonware's `Fixed` journal uses this. The record size is part of
the journal's configuration, and every record is exactly that
size. There is no length prefix. The trade-off: if a record's
content is smaller than the configured size, you waste space; if
larger, you cannot fit it.

Fixed-size records are great for:

- **Telemetry records.** Every sample is the same shape.
- **Heartbeats.** Every heartbeat is a fixed-size "I'm alive"
  message.
- **Consensus votes.** Every vote is the same shape (block hash,
  height, signature).

For variable-size records (blocks with different payload sizes,
messages with optional fields), fixed-size is the wrong choice.

### The Commonware length-prefix format

Commonware's length prefix is a varint (LEB128) capped at 5 bytes
(32 bits). So the prefix is between 1 and 5 bytes long.

```
+--------+--------+--------+--------+--------+
| byte 0 | byte 1 | byte 2 | byte 3 | byte 4 |
+--------+--------+--------+--------+--------+
   |
   |--- 7 data bits + continuation
   |--- high bit set means "more bytes follow"
```

The 5-byte cap means the maximum record size is `2^32 - 1` bytes
(~4 GiB). For most consensus messages, that's plenty.

The cap is enforced canonically: a 32-bit length must be encoded
in exactly 5 bytes (no "lazy zero-padding" like protobuf allows).
This is to ensure that the encoding is canonical — given a length,
there is exactly one valid encoding. (See Appendix A in the main
body for the canonicalization rules.)

### A worked example: concatenation

Suppose three messages:

```
Message A: 100 bytes
Message B: 5 bytes
Message C: 1000 bytes
```

The wire stream is:

```
[A_len=100, 100 bytes of A, B_len=5, 5 bytes of B, C_len=1000, 1000 bytes of C]
```

Total bytes on the wire: `len(A) + 100 + len(B) + 5 + len(C) + 1000`.

The receiver reads the stream in order:

1. Read varint → 100. Read 100 bytes → A.
2. Read varint → 5. Read 5 bytes → B.
3. Read varint → 1000. Read 1000 bytes → C.

If the stream is truncated (say, only the first 50 bytes of A
arrive), the receiver knows to expect 100 bytes but got 50, and
returns `Error::Short`. The next message is not parsed; the
receiver waits for more bytes or gives up.

This is the streaming decoder: incremental, safe, no ambiguity.

### Why not delimiter-based in practice

The historical reason delimiter-based framing is rare in modern
protocols is the *parsing complexity*. HTTP/1.1 used `\r\n\r\n`
to delimit headers, and the parser had to handle chunked
transfer encoding, content-length, and connection-close as three
different ways to know when the body ended. The complexity
produced bugs.

HTTP/2 (binary, length-prefixed) and HTTP/3 (QUIC, length-prefixed
inside QUIC frames) both moved away from delimiter-based framing.
QUIC's framing is entirely length-prefixed. So the modern trend
is: use length prefixes, escape nothing.

### Why not self-describing

A self-describing format (msgpack, CBOR, JSON) puts a type tag on
every value. This makes the wire format longer (an integer takes
1 byte in Commonware, 1-9 bytes in msgpack), but it makes the
decoder more flexible.

For a BFT consensus protocol, the wire format is fixed and known
in advance. The flexibility of self-describing is not needed.
The shorter encodings of Commonware's codec are a win.

For a public API where clients come and go with different
versions of the schema, self-describing (or Protobuf) is the
better choice. Commonware is not trying to be that.


## Variance and lifetime tricks in the codec

This section zooms in on a few of the more subtle type-level
choices in the codec's API, and explains the reasoning.

### Why `Cfg` is invariant

`Self::Cfg` is an associated type, which means each `impl Read`
sets its own `Cfg`. The compiler treats associated types as
*invariant* by default: if `Vec<T>::Cfg = T::Cfg` and you have a
`Vec<u32>` and a `Vec<u64>`, the `Cfg`s are not interchangeable.

This is the right behavior. `Cfg` is configuration, and two
different configurations are *different*. A `Cfg` that says "max
length 1000" is not interchangeable with a `Cfg` that says "max
length 10000."

If `Cfg` were covariant (the default for type parameters, not
associated types), then `Vec<ShortVec>` could be used where
`Vec<LongVec>` is expected, because `ShortVec <: LongVec` (if
such a subtyping relationship existed). But for configuration,
that's wrong: the lengths are part of the protocol contract.

### Why `Read` has a `&impl Buf` parameter but `Encode` does not

The asymmetry between `Encode` and `Read` is intentional.

`Encode::encode(&self, buf: &mut impl BufMut)` takes a *mutable*
buffer reference. The caller chooses the buffer; the encoder
writes into it. This is the streaming-friendly shape.

`Read::read_cfg(buf: &impl Buf, cfg: &Self::Cfg)` takes a
*shared* buffer reference and produces an owned value. The
decoder reads from the buffer and returns a value the caller owns.

The asymmetry comes from the *direction of data flow*. Encoding
goes *out* (into the buffer); decoding goes *in* (from the
buffer). The trait signatures mirror this.

### Why `Read` returns `Result<Self, Error>`

`Self` is an owned value, not a borrow. This is because the
decode operation *must* produce an owned value: the input buffer
may be a temporary (e.g., a socket read buffer that gets reused
for the next message), and a borrowed return value would have a
lifetime tied to a buffer that's about to be dropped.

If `Read` returned `&Self` with the input buffer's lifetime, the
returned reference would be unusable after the input is dropped.
So `Read` returns owned `Self`.

### Why `Encode` does not return a value

`Encode::encode(&self, buf: &mut impl BufMut)` returns `()`. The
encoder does not return a value because the caller does not
need one: the bytes are in `buf`.

If `Encode` returned `Vec<u8>`, every call would allocate. The
trait is designed for *streaming*: the caller provides the
buffer (possibly a reusable scratch buffer), and the encoder
writes into it. No allocation per call.

### Why `write_bufs` returns `impl WriteBuf`

`Encode::write_bufs(&self) -> impl WriteBuf` is a zero-copy
optimization. It returns a *description* of the bytes to write,
not the bytes themselves. The description is a sequence of
`(ptr, len)` pairs pointing into `&self`. The caller can then
issue `writev`-style syscalls that copy directly from `self`'s
memory into the socket.

The trick is that `impl WriteBuf` is an opaque type. The caller
doesn't need to know the layout; they just iterate over the
buffers and write them. This is the same idea as `bytes::Buf`
but specialized for the encode direction.

### Read-once types and `Read`

A *read-once* type is a value that can only be read from a byte
source once. After the read, the buffer is empty (or advanced
past the consumed bytes), and the value has been produced.

Commonware's codec is *not* read-once for `Self`, because `Self`
is returned by value. But the *byte source* (`&impl Buf`) is
read-once: each call to `read_cfg` advances the source past the
consumed bytes.

The reason this matters: if you have a single byte source and
you want to read multiple values from it, you must call `read_cfg`
multiple times, each call advancing the source. This is the
streaming pattern.

### The lifetime of `&impl Buf`

`Buf` is a trait that abstracts over byte sources. `&[u8]` is a
`Buf`. `Cursor<&[u8]>` is a `Buf`. `Reader` over a stream is a
`Buf`. The `&` in `&impl Buf` means the trait method takes a
*borrow* of the source, not ownership. The borrow's lifetime is
the lifetime of the call, not the lifetime of the returned value.

This is what lets `Read::read_cfg(buf: &impl Buf, ...)` produce
an owned `Self` even though `buf` is borrowed. The borrow is
released as soon as `read_cfg` returns; the returned `Self` lives
as long as the caller keeps it.

### Putting it together

The codec's type signatures are *not* arbitrary. They are the
result of three constraints:

1. **Streaming.** The codec must work on streams, not just in-memory
   buffers. Hence `&impl Buf` and `&mut impl BufMut`.

2. **Zero allocation in the encoder.** The encoder writes into a
   caller-provided buffer. Hence no `Vec<u8>` return.

3. **Owned values in the decoder.** The decoder produces owned
   values, because borrowed values would be tied to the input
   buffer's lifetime. Hence `Result<Self, Error>`.

If you remember those three constraints, you can re-derive the
trait signatures from scratch. The signatures are *forced* by the
constraints; they are not stylistic choices.


## The conformance machinery in detail

The conformance test is the source of truth for Commonware's wire
format. This section explains the machinery: the macro, the
deterministic random values, the hash, the `conformance.toml`
file, and the `just regenerate-conformance` workflow.

### The `conformance_tests!` macro

The macro lives in `codec/src/conformance.rs`. It generates a
test that:

1. Constructs a `CodecConformance<T>` wrapper.
2. Encodes a set of deterministic random values.
3. Decodes the bytes back.
4. Verifies the round-trip equals the original.
5. Computes a SHA-256 hash of the encoded bytes.
6. Compares the hash to a stored value in `conformance.toml`.

If the hashes match, the wire format is unchanged. If they don't,
the wire format has changed.

The macro looks like:

```rust
conformance_tests! {
    u8, u16, u32, u64,
    String, Vec<u8>,
    Block,
    Certificate,
}
```

You give it a list of types. For each type, it generates one test
function. The test function name is `conformance_<type_name>`.

### Deterministic random values

The test uses `arbitrary` to generate random values, but the
randomness is *seeded* by a deterministic value (the type name or
a fixed seed). So every run of the test produces the same random
values, and the resulting encoded bytes are identical run-to-run.

This is what makes the conformance test reproducible: you don't
need a snapshot of the random values; you just need the seed.

The seed is typically the type's name as bytes. So `Block` and
`Certificate` produce different random values. Within a type, the
same seed is used every run, so the encoded bytes are identical.

### The SHA-256 hash

After encoding, the test computes SHA-256 over the encoded bytes.
The hash is the *fingerprint* of the wire format for that type.
Two implementations of the same type should produce the same hash
for the same input.

The hash is stored in `conformance.toml`:

```toml
[u32]
hash = "0x..."

[Block]
hash = "0x..."

[Certificate]
hash = "0x..."
```

### Why a hash and not the bytes themselves

Storing the hash (32 bytes) is much smaller than storing the
encoded bytes (could be KB or more). The hash is enough to
verify that the wire format has not changed: if the hash
matches, the bytes match.

The trade-off: if the wire format changes, you have to regenerate
the hash, not just regenerate the bytes. But the hash is a
function of the bytes, so regenerating one regenerates the other.

### What `conformance.toml` stores

For each type, `conformance.toml` stores:

- The SHA-256 hash of the encoded bytes.
- The seed used to generate the random values.
- The version of the codec that produced the bytes.

The version field matters: if you upgrade Commonware's codec, the
version field changes, and old conformance files are rejected as
mismatched. This forces you to regenerate after a codec upgrade.

### `just regenerate-conformance`

When you intentionally change the wire format (add a field,
change an encoding), you update the code and then run:

```bash
just regenerate-conformance
```

This runs the conformance tests with a special flag that
overwrites `conformance.toml` instead of comparing against it.
You then commit the updated `conformance.toml` along with your
code change.

The workflow is:

1. Change the codec (e.g., add a field).
2. Run `just regenerate-conformance`.
3. Inspect the diff in `conformance.toml` (reviewers see the
   hash change).
4. Commit both the code change and the conformance change.

The reviewer sees a hash change in the diff, knows that the wire
format changed, and can audit the change.

### Why this is so important

The conformance test is the *contract* between Commonware's
implementations across versions. If you change the wire format
without updating the conformance test, the test fails, and you
notice before you ship.

If you change the wire format *and* update the conformance test,
the diff in `conformance.toml` is the audit trail. Anyone reading
the git history can see what changed and when.

This is a much stronger guarantee than "we have unit tests." Unit
tests can pass for many reasons. The conformance test passes for
exactly one reason: the wire format matches the stored hash.

### Limitations

The conformance test only covers types you explicitly add to the
`conformance_tests!` macro. If you add a new message type and
forget to add it to the macro, the type's wire format is not
covered. This is a maintenance burden but a small one.

The conformance test also doesn't cover *interoperability* with
other implementations. It only checks that *this* implementation's
encoding matches the stored hash. If another implementation
encodes differently, you won't know unless you test against it.

For a BFT library, this is OK: the wire format is Commonware's,
and you test against Commonware's own implementation. Cross-
implementation interop is a separate concern, handled by
integration tests.


## Exercises

These exercises build hands-on familiarity with the codec. They
are ordered roughly by increasing depth.

### Exercise 1: Implement a custom Codec for a simple struct

Define a `Point { x: i32, y: i32 }` struct and implement `Encode`
and `Decode` for it manually (no derive). Encode it as
big-endian, fixed size (8 bytes). Use `Cfg = ()`.

Then derive `Encode` and `Decode` and compare the implementations.
They should produce the same bytes.

### Exercise 2: Write a conformance test for your own type

Pick a type you've defined (something more interesting than
`Point`, maybe a struct with a `Vec<u8>` field). Add it to the
`conformance_tests!` macro. Run the test. Run
`just regenerate-conformance`. Inspect the diff in
`conformance.toml`. Revert the change and run the test again
to see it fail.

### Exercise 3: Implement a Cfg-parameterized type

Define a `BoundedVec<T> { items: Vec<T> }` whose Cfg includes a
`max_len: u32`. The decoder should reject any encoding with more
items than `max_len`. Write a test that tries to encode 10 items
with a `max_len: 5` Cfg and verifies it fails.

### Exercise 4: Compare against Bincode

Take the same struct from Exercise 1. Encode it with bincode and
with Commonware's codec. Compare the byte representations. Note:
bincode is little-endian, Commonware is big-endian. The bytes
should be reversed for the integer fields.

### Exercise 5: Schema evolution

Take a struct with two fields. Mark it `#[stability(BETA)]`. Add
a third field with `#[codec(default)]`. Verify the old bytes
still decode correctly with the new code, and the new bytes still
decode correctly with the old code (you'll need to keep a copy
of the old struct for the second test).
## Appendix A — Varint encoding, in detail

`codec/src/varint.rs`. The encoding is Google's Protocol Buffers LEB128,
with three refinements specific to Commonware:

1. **Canonical encoding enforced.** No "lazy" zero-padding.
2. **Capped at `u32` / `i32`** for size types — same on-wire format across
   32-bit and 64-bit architectures.
3. **Incremental decoder** for streaming protocols (length-prefix
   reading without buffering).

### The encoding

Each byte carries 7 data bits + 1 continuation bit (the high bit):

```
   7 6 5 4 3 2 1 0
  ┌─┬─┬─┬─┬─┬─┬─┬─┐
  │c│  data     │   c = 1 if more bytes follow, 0 if last
  └─┴─┴─┴─┴─┴─┴─┘
```

The integer is reconstructed little-endian across the data bits:

```
value 300 = 0b100101100

Byte 0: 0xAC = 1010_1100  →  data bits: 010_1100 = 0x2C = 44, continuation bit set
Byte 1: 0x02 = 0000_0010  →  data bits: 000_0010 = 0x02 = 2,  no continuation
Result: 44 + (2 << 7) = 44 + 256 = 300 ✓
```

Sizes: 1 byte for 0-127, 2 bytes for 128-16383, 3 bytes for 16384-2M, etc.
Most lengths in Commonware fit in 1 byte; only very long ones need more.

### The canonicalization check

`varint.rs:124-128`:

```rust
if byte == 0 && self.bits_read > 0 {
    return Err(Error::InvalidVarint(U::SIZE));
}
```

If you've already read some bytes and the next one is exactly zero,
that's **invalid**. Because the previous byte had continuation set, but
this byte has all data bits zero — which would have been better
represented by NOT continuing at all. Every value has exactly one
canonical encoding. Two encodings for the same number = attack surface
for ambiguous parsing.

### The four varint types

`varint.rs` provides four sealed trait implementations:

| Type | Range | Use case |
|---|---|---|
| `UInt(u128)` | 0 to 2^128-1 | Channel IDs, sizes, opaque integers |
| `SInt(i128)` | -2^127 to 2^127-1 | Signed offsets, deltas |
| `U32(U32)` | 0 to 2^32-1 | Reserved for specific use |
| `S32(i32)` | -2^31 to 2^31-1 | Signed 32-bit values |

`UInt` uses **unsigned** encoding. `SInt` uses **ZigZag** encoding
(varint.rs:18-21):

```
value  0 → 0
value -1 → 1
value  1 → 2
value -2 → 3
value  2 → 4
```

Map `(n << 1) ^ (n >> 63)` for 64-bit, then varint-encode. Small-magnitude
numbers (positive or negative) encode to small byte counts.

### The incremental `Decoder`

`varint.rs:90-93`:

```rust
pub struct Decoder<U: UPrim> {
    result: U,
    bits_read: usize,
}
```

For streaming protocols where you receive bytes one at a time and need
to know when a varint is complete. The `feed(byte)` method returns
`Ok(Some(value))` when the varint ends, `Ok(None)` when more bytes are
needed, `Err(InvalidVarint)` on malformed input.

This is what lets `Stream::recv(len)` (chapter 02) do length-prefixed
framing: read bytes, feed to `Decoder<u32>`, get length, then read that
many more bytes.

## Appendix B — `RangeCfg` — the safety knob

`codec/src/config.rs` (and `extensions.rs:72-99`). Every collection decode
takes a `RangeCfg<usize>` to bound the allocation:

```rust
pub struct RangeCfg<T> {
    pub min: Bound<T>,    // inclusive lower bound
    pub max: Bound<T>,    // inclusive upper bound
}

impl<T> RangeCfg<T> {
    pub fn new(range: impl RangeBounds<T>) -> Self { ... }
}
```

Examples:

```rust
// A Vec with 0 to 1000 elements
let cfg: RangeCfg<usize> = (0..=1000).into();
let v: Vec<u8> = Vec::decode_cfg(buf, &(cfg, ()))?;

// A non-empty Vec with at most 100 elements
let cfg: RangeCfg<usize> = (1..=100).into();
```

The decode validates the length prefix against `cfg` BEFORE allocating:

```rust
// In Vec::read_cfg
let len = usize::read_cfg(buf, range)?;     // checks cfg first
T::read_vec(buf, len, cfg)                    // then allocates len items
```

This is the safety pattern: **decode-time bounds, not encode-time
trust**. The encoder can lie. The decoder enforces.

`RangeCfg` also exists for `HashMap`, `BTreeMap`, `HashSet`, `BTreeSet`,
and `Bytes` (the latter bounds the byte length, not the count).

## Appendix C — The full codec variant taxonomy

`codec/src/codec.rs`. There are 8 distinct traits, organized by axis:

### By variability

- **`FixedSize`** — encoded size is a `const SIZE: usize`. Known at compile
  time.
- **`EncodeSize`** — encoded size is a runtime method. May depend on the
  value (e.g., for variable-size types).

`FixedSize: EncodeSize` is automatic (lines 89-110 of `codec.rs`).

### By direction

- **`Write`** — encode to bytes.
- **`Read`** — decode from bytes.

### By combination

- **`Encode = Write + EncodeSize`** — full encode (size + write).
- **`Decode = Read`** — full decode.
- **`Codec = Encode + Decode`** — both directions.

### By thread safety

- `Encode` / `Codec` — single-threaded use.
- **`EncodeShared: Encode + Send + Sync`** — cross-thread use (e.g., shared
  types inside async tasks).
- **`CodecShared: Codec + Send + Sync`** — both directions, cross-thread.

### By size guarantee

- **`EncodeFixed: Write + FixedSize`** — encoded size is `const SIZE`.
- **`DecodeFixed: Read<Cfg = ()> + FixedSize`** — decoder doesn't need Cfg
  because the size is known.
- **`CodecFixed: Codec + FixedSize`** — both directions, fixed size.
- **`CodecFixedShared: CodecFixed + Send + Sync`** — fixed size, cross-thread.

These are useful for fixed-size cryptographic primitives (`Sha256` digest
= 32 bytes, `Ed25519` signature = 64 bytes) where compile-time size
guarantees matter.

### The `Lazy` type

`codec/src/types/lazy.rs`. For types whose encoding is expensive
(computing it eagerly would slow down the encode path):

```rust
pub struct Lazy<T>(OnceCell<T>);

impl<T: Encode> Write for Lazy<T> {
    fn write(&self, buf: &mut impl BufMut) {
        // Write the cached encoding if available
        if let Some(encoded) = self.0.get() {
            encoded.write(buf);
        } else {
            // Or a placeholder, with the assumption that something
            // will fill in the real encoding later
            buf.put_slice(&PLACEHOLDER);
        }
    }
}
```

Used by `Attestation` (chapter 04, `cert.rs:81-87`):

```rust
pub struct Attestation<S: Scheme> {
    pub signer: Participant,
    pub signature: Lazy<S::Signature>,  // computed lazily
}
```

When the same `Attestation` is included in multiple certificate
structures, the signature is computed once and cached.

## Appendix D — Hashes and digests

The codec crate doesn't define a hash function; it provides traits for
**types that act as digests**. From `codec/src/lib.rs` (look for `Digest`):

```rust
pub trait Digest: Array { }    // fixed-size byte array, content-addressed
```

Implementors include:
- `Sha256Digest` (32 bytes) — SHA-256 output.
- `Blake3Digest` (32 bytes) — Blake3 output.
- `Ed25519PublicKey` (32 bytes) — public key as digest.
- `Bls12381PublicKey` (48 / 96 bytes) — public key as digest.

The codec layer treats these as fixed-size arrays. The hash function
lives in the cryptography crate.

## Appendix E — The conformance harness internals

`codec/src/conformance.rs`. The flow:

```rust
pub fn generate_value<T>(seed: u64) -> T
where T: for<'a> Arbitrary<'a>,
{
    let mut rng = ChaCha8Rng::seed_from_u64(seed);
    let mut buffer = vec![0u8; INITIAL_BUFFER_SIZE];  // 4 KB
    rng.fill(&mut buffer[..]);
    let mut unstructured = Unstructured::new(&buffer);
    T::arbitrary(&mut unstructured).unwrap()
}
```

The `arbitrary` crate (`docs.rs/arbitrary`) is a structured fuzzing
library. `Arbitrary::arbitrary` takes unstructured bytes and tries to
construct a valid `T`. If it can't, it returns `IncorrectFormat`
(discard these bytes, try the next) or `NotEnoughData` (double the buffer,
try again).

The flow:

1. Generate 4 KB of ChaCha-seeded random bytes.
2. Feed them to `T::arbitrary`.
3. If success, hash the encoded bytes. If failure, double the buffer.
4. Up to 16 MB; panic if even that fails.

The encoding step:

```rust
async fn commit(seed: u64) -> Vec<u8> {
    let value: T = generate_value(seed);
    value.encode().to_vec()
}
```

The `Conformance` trait (from `commonware-conformance`, NOT codec):

```rust
pub trait Conformance: Send + Sync {
    async fn commit(seed: u64) -> Vec<u8>;
}
```

This is what the `conformance_tests!` macro (in `commonware-conformance`)
implements. The macro wraps a `Conformance<T>` and generates a test that:

1. Calls `commit(seed)` for many seeds.
2. Hashes all the bytes together.
3. Compares against stored hash in `conformance.toml`.

The hash isn't `SHA256(seed || encoded)`; it's `SHA256(SHA256(...) || encoded)` —
chained, so order doesn't matter but presence does.

### Where conformance files live

Each crate that implements `Encode` for its types has a
`conformance.toml` file at its root:

```
cryptography/conformance.toml
consensus/conformance.toml
p2p/conformance.toml
storage/conformance.toml
... etc.
```

Each contains `[TypeName]` sections with the expected hash and case count.

### Regenerating after an intentional change

```bash
# Make your encoding change
just regenerate-conformance -p commonware-cryptography

# Review the diff in conformance.toml
git diff cryptography/conformance.toml

# Commit
git commit -am "Update cryptography conformance for new Sha256 encoding"
```

The workflow ensures that encoding changes are **explicit** in code
review. You can't accidentally break wire compatibility without
realizing it.

## Appendix F — The `ReadExt` trait — `Cfg = ()` convenience

`codec/src/extensions.rs:15-56`:

```rust
pub trait ReadExt: Read<Cfg = ()> {
    fn read(buf: &mut impl Buf) -> Result<Self, Error> {
        Self::read_cfg(buf, &())
    }

    fn read_vec(buf: &mut impl Buf, len: usize) -> Result<Vec<Self>, Error> {
        Self::read_cfg_vec(buf, len, &())
    }
    // ... many more convenience methods ...
}
```

For types with `Cfg = ()`, `ReadExt` provides no-config read methods.
Used everywhere; the most common case for simple types.

## Appendix G — The `ReadRangeExt` for sized reads

`codec/src/extensions.rs:72-99`:

```rust
pub trait ReadRangeExt<X: IsUnit>: Read<Cfg = (RangeCfg<usize>, X)> {
    fn decode_range(buf: impl Buf, range: impl RangeBounds<usize>)
        -> Result<Self, Error>;
}
```

For collections where you want to specify a range without manually
constructing `Cfg`:

```rust
let v: Vec<u8> = Vec::decode_range(encoded_buf, 1..=100)?;
// equivalent to: Vec::decode_cfg(buf, &((1..=100).into(), ()))
```

Just syntactic sugar.

## Appendix H — Why big-endian specifically

`codec/src/types/primitives.rs:18-19`:

> All fixed-size integers and floats are written big-endian to avoid host-
> endian ambiguity.

Network byte order is big-endian (RFC 1700, STD 2). When you `recv(4)` on
a TCP connection, the first byte you see is the most significant byte of
the first field — by convention, since ARPANET.

Modern systems use little-endian (x86, ARM in most configs). If Commonware
emitted little-endian, every cross-architecture handoff would need
swapping. By emitting big-endian, the wire format is portable AND
matches the de-facto network standard.

Cost: `put_u32` on big-endian requires a byte-swap on little-endian
hardware. CPUs handle this in 1 cycle (BSWAP instruction). Negligible.

## Appendix I — The `Tuples` and primitive impls

`codec/src/types/tuple.rs` and `types/primitives.rs`. Tuple implementations:

```rust
impl<A: Write, B: Write> Write for (A, B) {
    fn write(&self, buf: &mut impl BufMut) {
        self.0.write(buf);
        self.1.write(buf);
    }
}
```

Up to some max arity (typically 12). The tuple's encoding is the
concatenation of its elements' encodings. Size = sum of sizes (or
runtime sum for variable-size elements).

`bool` is special:

```rust
impl Write for bool {
    fn write(&self, buf: &mut impl BufMut) {
        buf.put_u8(if *self { 1 } else { 0 });
    }
}

impl Read for bool {
    type Cfg = ();
    fn read_cfg(buf: &mut impl Buf, _: &()) -> Result<Self, Error> {
        match u8::read(buf)? {
            0 => Ok(false),
            1 => Ok(true),
            _ => Err(Error::InvalidBool),
        }
    }
}
```

One byte. `0` = false, `1` = true, anything else = `InvalidBool` error.

## Appendix J — The `net.rs` types

`codec/src/types/net.rs`. Network-related fixed types:

- `IpAddr` — `IPv4` or `IPv6` variant (1-byte tag + 4 or 16 bytes).
- `SocketAddr` — `IpAddr` + `u16` port (variable, 7 or 19 bytes).
- These have a `Read::Cfg = ()` because they're fixed-size.

Used by P2P and runtime for socket addresses.

## Appendix K — The `Hash`/`HashMap`/`HashSet` impls

`codec/src/types/hash_map.rs`. Like Vec, but with a twist:

```rust
impl<K: Encode, V: Encode> Write for HashMap<K, V> {
    fn write(&self, buf: &mut impl BufMut) {
        self.len().write(buf);          // length prefix
        for (k, v) in self.iter() {     // sorted by key!
            k.write(buf);
            v.write(buf);
        }
    }
}

impl<K: Decode, V: Decode> Read for HashMap<K, V> {
    type Cfg = (RangeCfg<usize>, K::Cfg, V::Cfg);
    fn read_cfg(buf: &mut impl Buf, (range, k_cfg, v_cfg): &Self::Cfg)
        -> Result<Self, Error>
    {
        let len = usize::read_cfg(buf, range)?;
        let mut map = HashMap::with_capacity(len);
        for _ in 0..len {
            let k = K::read_cfg(buf, k_cfg)?;
            let v = V::read_cfg(buf, v_cfg)?;
            if map.insert(k, v).is_some() {
                return Err(Error::Invalid("HashMap", "duplicate key"));
            }
        }
        Ok(map)
    }
}
```

Two important properties:

1. **Sorted by key.** Map encoding is deterministic regardless of
   `HashMap` iteration order (which is randomized).
2. **Duplicate detection.** If a key appears twice in the input,
   decoding fails with `Invalid("HashMap", "duplicate key")`. Catches
   malicious peers trying to game things via duplicate entries.

## Appendix L — Common pitfalls

### Forgetting `?Sized`

If your type has a `?Sized` marker, the auto-derived `EncodeSize` for
`Vec<T>` won't work. You'll need explicit impls.

### Forgetting `Read<Cfg = ()>` for simple types

If your `Read::Cfg` is something other than `()`, every call site needs to
construct the Cfg. Annoying. Default to `Cfg = ()` when possible.

### Encoding stateful types

If your type has interior state that affects encoding (e.g., a cache),
the encoding is non-deterministic. Run `cargo test` with the auditor
test to catch this.

### Encoding `HashMap` without sorting

Already covered above. Don't do this.

## Where to look in the code (expanded)

- `codec/src/codec.rs:1-380` — all the trait definitions.
- `codec/src/varint.rs:1-797` — the varint codec.
- `codec/src/conformance.rs` — the conformance harness.
- `codec/src/config.rs` — `RangeCfg` and friends.
- `codec/src/extensions.rs` — `ReadExt`, `DecodeExt`, `ReadRangeExt`.
- `codec/src/types/` — impls for primitives, Vec, HashMap, tuples, etc.
- `commonware-conformance/` — the macro and runner.

## If you only remember three things

1. **Big-endian, varint for `usize`, every `Read` takes a `Cfg`.** Those are the three decisions that make the format portable, compact, and safe against hostile input.
2. **Length-prefixed containers + `RangeCfg`.** Decoders bound the allocation before they allocate. No `O(n)` surprises from a malicious peer.
3. **Conformance tests hash your wire format.** Change the encoding intentionally → regenerate the hash. Change it accidentally → CI fails before you deploy.

→ Next: **Chapter 04 — Cryptography**. The crypto in chapter 01 was hand-wavy
(`2f+1` votes, signed by validators). Now we'll see what those signatures
actually are, how BLS12-381 collapses `2f+1` partial signatures into a single
verifiable aggregate, and what threshold cryptography buys you.