# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.1

A patch: the wire format is the wire format.  The Protocol Buffers
`sint32` / `sint64` table, the error vocabulary and the 32-bit range
checks all still pass.

- **The manifest carries the fields the registry browses by.**
  `category`, `tags`, `repository` and `maintainers` were added after
  `0.1.0` was published, and a published version is never replaced, so
  this release is the first one the packages page can shelve and
  filter.
- **The suite's constants are written with the operators.**  `1 << 63`
  and `1 << shift` where they were `bits.shl` calls.  The module itself
  has no bit arithmetic to rewrite: it is the ZigZag fold and the
  LEB128 byte loop composed, and both live in the packages under it.

## 0.1.0

First release: `max_len`, `encoded_len`, `encode_into`, `encode`,
`decode_from`, `decode`, `decode_at`, each with a 32-bit twin, plus
`fits32`, `min32`, `max32` and the `VarintError` trio.

- **The Protocol Buffers pairing as one call.**  ZigZag then LEB128, in
  the order `sint32` and `sint64` are defined by, so the bytes are the
  bytes every other implementation writes.
- **The in-place pair allocates nothing.**  `encode_into` and
  `decode_from` work over a cursor the caller owns.
- **A malformed input is a value.**  `Truncated`, `Overflow` and
  `NotInt32` each carry the byte count reached; nothing about bad input
  panics.  `NotInt32` is separate from `Overflow` on purpose — the bytes
  are a valid varint, so the same input still decodes at 64 bits.
- **The vectors are Protocol Buffers'**, so the suite is evidence about
  the encoding rather than about this implementation.
- **Depends on `leb128-nv ^0.1.0` and `zigzag-nv ^0.1.0`**, the widest
  ranges this package supports: it calls only what each of them shipped
  first, so a consumer already holding a `0.1.x` of either keeps it.
