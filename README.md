# varint-nv

A signed varint is a signed integer written in as few bytes as its
magnitude needs. It is two transforms applied in order. The ZigZag fold
maps a signed integer onto an unsigned one, and base 128 groups write
that unsigned number seven bits at a time. The pairing is what Protocol
Buffers calls `sint32` and `sint64`, and it is described in the
[Protocol Buffers encoding documentation](https://protobuf.dev/programming-guides/encoding/).
This package brings it to novo-lang at both widths. It is built on
[leb128-nv](https://novo-lang.org/packages/leb128-nv) for the byte loop
and [zigzag-nv](https://novo-lang.org/packages/zigzag-nv) for the fold.

## What a signed varint is

A number is cut into groups of seven bits, least significant group
first. Each group travels in the low seven bits of one byte. The top
bit of that byte is the **continuation bit**. It is 1 on every byte but
the last, so a reader finds the end of a number without being told its
length beforehand. Protocol Buffers calls that a base 128 varint, and
the same encoding is specified in the
[DWARF standard](https://dwarfstd.org/) section 7.6 under the name
LEB128.

Those groups carry an unsigned number. A signed number written straight
into them travels as its two's complement bit pattern, which sets the
high bits of every negative number. `-1` is then 2^64 − 1, and it takes
all ten bytes. Every negative number costs the maximum, however small
it is.

The **ZigZag fold** is what removes that cost. It renumbers the
integers 0, −1, 1, −2, 2, … onto 0, 1, 2, 3, 4, …, so that magnitude
rather than sign decides how many groups a number needs. The fold is
`(n << 1) ^ (n >> 63)` at 64 bits, and it is a bijection onto the
unsigned range, so nothing is lost and no number is refused. Folding
first is what makes `-1` one byte instead of ten.

One byte carries seven bits of the folded number rather than of the
original, so one byte holds −64 to 63. The length steps up at every
seventh power of two of the folded value and nowhere else.

There are two widths, and they are the two Protocol Buffers types. The
64-bit pair covers the whole signed range of `Int` and takes up to ten
bytes. The 32-bit pair is defined over the signed 32-bit range and
takes up to five. Inside that range both write the same bytes, because
the fold is one renumbering and the narrower half simply has less room
above it.

| Quantity | Value |
| --- | --- |
| Bits one byte carries | 7 |
| Values one byte covers | −64 to 63 |
| Bytes a 64-bit value takes, at most | 10 |
| Bytes a 32-bit value takes, at most | 5 |
| The range the 32-bit pair is defined over | −2^31 to 2^31 − 1 |
| The order the fold renumbers in | 0, −1, 1, −2, 2, … |
| `-1` encoded | `01` |
| `-75` encoded | `95 01` |
| `150` encoded | `ac 02` |

## Install

```
novo pkg add varint-nv
```

## Example

```novo
use varint

fn main() [io]
    // Encode a signed number into a fresh buffer of its own length.
    let wire = varint.encode(0 - 75)          // 95 01 — two bytes
    // Read it back. A malformed input arrives as an `Err`.
    match varint.decode(wire)
        Ok(v)  => println("${v}")             // -75
        Err(e) => println(e.message())
```

Build and test with `novo pkg build` and `novo test tests/varint_tests.nv`.

## What the package contains

| Module | Contents |
| --- | --- |
| `varint` | The size calls, `max_len`, `max_len32`, `encoded_len` and `encoded_len32`. The encoders, `encode`, `encode_into`, `encode32` and `encode32_into`. The decoders, `decode`, `decode_from`, `decode_at`, `decode32`, `decode32_from` and `decode32_at`. The 32-bit range, `min32`, `max32` and `fits32`. The `Decoded` pair and the `VarintError` enum. |

The API reference is on
[the package's page](https://novo-lang.org/packages/varint-nv). `novo
doc` generates it from these sources: every `pub` declaration with its
signature, its effect row and the comment block written above it. Each
entry carries a worked example that is compiled and run as a test.

## How to choose an entry point

**`encode` and `decode` take the whole value at once.** `encode`
allocates a buffer of exactly the length the value needs and answers
it. `decode` reads the varint at the start of a buffer and ignores
whatever follows. This is the pair for a program that is building or
reading one message.

**`encode_into` and `decode_from` work over a cursor the caller owns.**
Each allocates nothing, and each leaves the cursor on the byte after
the number. This is the pair for a writer filling a frame, and for a
reader walking a run of varints with one cursor.

**`decode_at` asks by offset instead of by cursor.** It answers a
`Decoded`, which carries the value and the number of bytes it occupied,
so the next offset is `off + d.len`. This is the call for a table of
varint-encoded entries, or a format that hands out each record's start.

**Each of those has a 32-bit twin, named with `32`.** Use the 64-bit
form unless the protocol says the number is 32-bit. The 32-bit form
takes at most five bytes rather than ten, and a number outside the
signed 32-bit range is refused on the way in.

**`max_len` and `encoded_len` are the sizes**, with `max_len32` and
`encoded_len32` beside them. `encoded_len` answers what one value will
take. `max_len` answers what any value can take, which is the room to
reserve for a value that is not yet computed.

**`fits32` is the range check**, and `min32` and `max32` are its two
ends. Ask it once, where the number arrives.

## The rules a user needs

1. **The 32-bit pair is defined over `fits32` and nowhere else.**
   `encode32` does not check, and a value outside that range folds to a
   well-formed number that every other implementation reads as
   something else. The check belongs at the edge where the number
   arrives rather than at every write. The 64-bit pair needs no such
   check, because every `Int` is in its range.
2. **A malformed input is a value and never a panic.** `Truncated` says
   the input ended with a continuation bit still set. `Overflow` says
   the groups read exceed 64 bits. `NotInt32` says the number is well
   formed but outside the signed 32-bit range. Each carries the byte
   count the reader reached, so a caller dropping a bad frame can say
   where it went wrong.
3. **`NotInt32` is not the same answer as `Overflow`.** The bytes are a
   valid varint, so the fault is the reader's width rather than the
   encoding. A caller told which one it is can read the same bytes
   again at 64 bits. The cursor is left after the number, so a caller
   that reads again moves it back to where the number started, and a
   caller that skips the field reads on from where it is.
4. **A cursor with less room than the value needs is a panic.**
   `encode_into` writes through `put_u8`, which panics on a short
   buffer exactly as `xs[i]` does on an index out of range. A buffer
   the caller sized wrong is a mistake in the program rather than
   input. Reserve `max_len()`, or test `dst.remaining()` first.
5. **The cursor's position after a call is where the next number
   starts.** A caller reading a run of varints needs nothing else. It
   calls `decode_from` again.
6. **An offset outside the buffer is `Truncated`, not a panic.** An
   offset that came out of the data being decoded is input, and input
   is never a reason to stop the program. `decode_at` answers
   `Truncated(after: 0)` for it.
7. **The two widths write the same bytes wherever both are defined.** A
   number inside the signed 32-bit range has one encoding, and either
   pair reads what the other wrote. What differs is the ceiling, five
   bytes against ten.
8. **Only `encode` and `encode32` allocate**, for their one result
   buffer. The fold is two shifts and an xor with no branch, so it
   costs the same for every input, and the byte loop is one shift, one
   mask and one write per output byte. The arithmetic is `Int`
   throughout, so nothing here needs a heap, and no function in the
   package performs input or output.

## What is not included

- **Unsigned varints.** They are
  [leb128-nv](https://novo-lang.org/packages/leb128-nv), which this
  package is built on and which a consumer can add on its own.
- **Signed LEB128.** Its top group is sign-extended rather than folded,
  which makes it a different encoding of the same number. A reader
  expecting the fold misreads it, with nothing in the bytes to say that
  something is wrong.
- **Field tags, wire types and message framing.** This is the integer
  encoding, and not Protocol Buffers.
- **A refusal for a padded encoding.** Groups of zeroes may be added on
  the end without changing the value, so two byte strings can carry the
  same number. This package writes the bytes the value needs, and it
  reads a padded encoding as the value it means.
- **Streaming across buffer boundaries.** A varint must be contiguous
  in one buffer. A reader holding part of one gets `Truncated`, and it
  reads the number again from its first byte once the rest has arrived.
- **Any input or output.** Every function here is arithmetic over bytes
  the caller already holds.

## Related packages

- [leb128-nv](https://novo-lang.org/packages/leb128-nv) is the base 128
  groups on their own, over the full unsigned 64-bit range. It is the
  package to reach for when the numbers are unsigned, as a length, an
  index or an offset is. This package calls it for the byte loop.
- [zigzag-nv](https://novo-lang.org/packages/zigzag-nv) is the fold on
  its own, at 32 and 64 bits, with the range check and the two ends of
  the 32-bit range. It answers a number rather than bytes, so a caller
  that already has a varint writer can fold with it and write the
  result itself. This package calls it for the fold.

Both are dependencies of this package, so adding it brings them with
it. They are pinned in your `novo.lock` beside it, and you do not list
them yourself.

## Tests

```
novo test tests/varint_tests.nv
```

The suite is 20 tests. The vectors in it are the bytes Protocol Buffers
writes for `sint32` and `sint64`: the first five values by magnitude,
the one-byte boundary at a magnitude of 64, and both ends of both
widths. Asserting the bytes another implementation writes is what makes
the suite evidence about the encoding rather than about this
implementation.

Beside the vectors the suite asserts a round trip over every power of
two at both widths, that the two widths agree wherever both are
defined, that `encoded_len` answers what `encode` writes, that nothing
exceeds the advertised maximum, that a run of varints is read with one
cursor and walked by offset, and each of the four ways an input can be
refused. Every example in a documentation comment is compiled by `novo
doc` and run by `novo test`, so an example that stopped being true is a
failing test.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
