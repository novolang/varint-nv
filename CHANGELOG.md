# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.3

Documentation: the reference is generated from the code, and the
examples in it are doctests.  No code changed — every byte on the wire
is what 0.1.2 produced, and both dependency ranges are unchanged.

- **Every `pub` item is documented under Go's rule**, the comment block
  directly above the declaration, its first sentence the summary a
  reader meets before opening anything.  `VarintError.message`, which
  had no comment at all, has one.  `novo doc` turns the lot into
  [the package's page](https://novo-lang.org/packages/varint-nv).
- **Eleven worked examples, and they run.**  Both widths, both pairs
  and the offset walk are shown where they are declared, with the bytes
  written out in hex; so is the difference between a truncated varint
  and a well-formed one too wide for a 32-bit reader.  A fenced `novo`
  block in a documentation comment is compiled by `novo doc` and run by
  `novo test src/varint.nv`, so an example that stopped being true is a
  failing test rather than a reader's afternoon.

## 0.1.2

Developed in its own repository from this version.  `novolang/varint-nv` is
where the sources live, where CI runs and where releases are tagged;
the novo-lang monorepo no longer carries a copy.  No code changed —
every signature and every byte on the wire is what 0.1.1 shipped.

- **`LICENSE` ships with the package.**  It is on the publish
  allow-list, so the tarball now carries the Apache-2.0 text rather
  than only naming it in the manifest.

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
