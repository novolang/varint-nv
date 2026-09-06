> Developed and published from this repository.  `varint-nv` began under
> `orbit/varint-nv` in the [novo-lang](https://github.com/novolang) monorepo
> and graduated out of it with its history; issues and pull requests
> belong here.

# varint-nv

Signed varints on a wire. LEB128 spends one byte per seven bits, so it
is cheap for small numbers — but two's complement makes every negative
number large, and a plain LEB128 of `-1` is ten bytes. ZigZag renumbers
the integers first, `0, -1, 1, -2, 2, …`, so that magnitude rather than
sign decides the size. Fold, then encode: that pairing is what Protocol
Buffers calls `sint32` and `sint64`, and this package is the two halves
as one call.

```novo
use varint

fn main() [io]
    let wire = varint.encode(0 - 75)          // 95 01 — two bytes
    match varint.decode(wire)
        Ok(v)  => println("${v}")             // -75
        Err(e) => println(e.message())
```

```
novo pkg add varint-nv
```

## What it gives you

Each function has a 32-bit twin, named with `32`. The two widths write
the same bytes wherever both are defined; the 32-bit pair simply cannot
be asked for a number outside its range, and takes at most five bytes
instead of ten.

The API is on [the package's page](https://novo-lang.org/packages/varint-nv),
generated from these sources: every `pub` declaration with its signature,
its effect row and the comment block written above it. A table of names
here would be a second original, and the second original is the one that
goes stale.

`VarintError` has three variants and all three are input faults:
`Truncated` when the input ends with a continuation bit still set,
`Overflow` when the groups read exceed 64 bits, and `NotInt32` when a
well-formed number is outside the signed 32-bit range. Each carries the
byte count reached, so a receiver dropping a bad frame can say where it
went wrong.

`NotInt32` is deliberately not the same answer as `Overflow`: the bytes
are a valid varint, so the fault is the reader's width rather than the
encoding, and a caller told which one it is can re-read the same bytes
at 64 bits.

## Writing into a buffer you already hold

`encode_into` and `decode_from` take a cursor and allocate nothing. The
offset lives in the cursor rather than in a third parameter because a
`Bytes` passed as a parameter is borrowed — the caller still holds it,
so a write through it lands in a copy the caller never sees. A cursor
owns its buffer, so a write through one lands where the caller can read
it. That is the same rule the standard library's own writers follow.

```novo
use std.bytes
use varint

// Three signed fields into one record, back to back.
fn record(a: Int, b: Int, c: Int) -> Bytes
    var w = bytes.cursor_le(bytes.zeros(3 * varint.max_len()))
    let n = varint.encode_into(w, a) + varint.encode_into(w, b)
              + varint.encode_into(w, c)
    bytes.slice(w.finish(), 0, n)
```

Reading is the mirror image, and the cursor's position after each call
is where the next number starts:

```novo
fn first_two(wire: Bytes) -> Result<Int, VarintError>
    var r = bytes.cursor_le(wire)
    let a = varint.decode_from(r)!
    let b = varint.decode_from(r)!
    Ok(a + b)
```

`decode_at` asks the same question by offset rather than by cursor —
for a table of varint-encoded entries, or a format that hands out each
record's start. It answers a `Decoded` carrying `value` and `len`, so
the next offset is `off + d.len`:

```novo
fn sum(buf: Bytes) -> Result<Int, VarintError>
    var off = 0
    var total = 0
    while off < bytes.len(buf)
        let d = varint.decode_at(buf, off)!
        total = total + d.value
        off = off + d.len
    Ok(total)
```

## Choosing a width

Use the 64-bit pair unless the protocol says the number is 32-bit. The
32-bit pair is defined over `min32() .. max32()` and `fits32` is the
question to ask at the edge where the number arrives — not at every
write. Handing `encode32` a number outside that range folds it to
something no other implementation would produce, exactly as ZigZag's own
32-bit fold does; the check belongs where the value enters the program.

## What it costs

Two shifts and an xor for the fold, branch-free, so it costs the same
for every input; then one shift, one mask and one write per output byte.
`encode_into` and `decode_from` allocate nothing; `encode` allocates its
one result buffer. The arithmetic is `Int` throughout — no table, no
state, nothing that needs a heap.

`encode_into` panics when the cursor has less room than the value needs,
exactly as `c.put_u8` and `xs[i]` do: a buffer the caller sized wrong is
a mistake in the program. Reserve `max_len()`, or test
`dst.remaining()` first. Malformed **input** is never a panic — that is
what `VarintError` is for.

## What it does not do

No unsigned varints: those are `leb128-nv`, which this package is built
on and which a consumer can add on its own. No signed LEB128, whose top
group is sign-extended rather than folded — a different encoding, and
one a ZigZag reader will silently misread. No field tags, no wire types
and no message framing: this is the integer encoding, not Protocol
Buffers.

## Dependencies

It depends on [`leb128-nv`](https://novo-lang.org/packages/leb128-nv) for
the byte loop and [`zigzag-nv`](https://novo-lang.org/packages/zigzag-nv)
for the fold. The ranges are on
[the package's page](https://novo-lang.org/packages/varint-nv), read from
this manifest, with the whole closure folded under them.

Both ranges are the widest this package supports, because it calls only
what each of them shipped in its first release. That matters to you
rather than to us: one version of a package is built into a program, so
a range that excluded a release you already hold would be a version
conflict you had to resolve. The floor rises only when this package
needs something a later release added, and `CHANGELOG.md` says so when
it does.

Adding this package brings both of them with it. You do not list them,
and they are pinned in your `novo.lock` alongside it.

## Tests

```
novo test tests/varint_tests.nv
```

The vectors are the bytes Protocol Buffers writes for `sint32` and
`sint64` — the first values by magnitude, the one-byte boundary at 64,
the documentation's own `150` and `-75`, and both ends of both widths.
Beyond those: a round trip over every power of two, `encoded_len`
checked against what `encode` actually writes, and each of the four ways
an input can be refused.
