- Start Date: (fill in with date at which the RFC is merged, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#73](https://github.com/amaranth-lang/rfcs/pull/73)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Stricter connections

## Summary
[summary]: #summary

Make it possible for `lib.wiring.connect()` to reject connections between ports with shapes that despite having equal widths are incompatible.

## Motivation
[motivation]: #motivation

The primary motivating use case is to be able to prevent connecting stream interfaces with incompatible payload shapes.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

`lib.stream` defines a basic stream interface with `ready`, `valid` and `payload` members.
Additional metadata signals can be added by making the `payload` shape an aggregate.
This has the advantage that such streams can be passed through standard components like FIFOs without them having to know or care about these signals.

Consider how we can make a byte-wide packetized stream by defining the `payload` shape as `StructLayout({"data": 8, "last": 1})`.
The `data` field contains the data byte and the `last` field is a flag indicating that the current byte is the last in the packet.

This is not the only common way to make a packetized stream.
Instead of (or in addition to) the `last` flag, the payload could have a `first` flag, indicating that the current byte is the first in the packet.
Both are valid options with different semantics for different usecases.

The problem is that `lib.wiring.connect()` currently only checks that port members have matching widths, so a connection between a stream interface with `last` semantics and a stream interface with `first` semantics will not be rejected, despite being incompatible.

Currently, `lib.data.View.eq()` does no checks on the passed value and immediately forwards to `self.as_value().eq()`, making this legal:

```python
>>> a = Signal(StructLayout({"data": 8, "last": 1}))
>>> b = Signal(StructLayout({"data": 8, "first": 1}))
>>> a.eq(b)
(eq (sig a) (sig b))
```

This RFC proposes adding a check to `View.eq()` that would reject direct assignments from another aggregate data structure with a non-identical layout.
If such an assignment is desired, the other aggregate data structure can be explicitly passed through `Value.cast()` first.

Currently `lib.wiring.connect()` passes every signal through `Value.cast()` before assigning them.
This results in a `ValueCastable`'s `.eq()` not being called, and thereby bypassing the check proposed above.

This RFC proposes removing the `Value.cast()` so a `ValueCastable`'s `.eq()` will be called directly.

This RFC proposes updating `lib.enum.EnumView` in the same manner, for the same reason, as `lib.data.View`.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Modify `lib.wiring.connect()` to not pass port members through `Value.cast()`, so that a `ValueCastable`'s `.eq()` will be called, allowing it to perform compatibility checks.
- If a `ValueCastable` doesn't define `.eq()`, reject the assignment.

Modify `lib.data.View.eq(other)` to add the following checks:
- If `other` is a `ValueCastable`, do `Layout.cast(other.shape())`
  - If a valid `Layout` is returned, reject the assignment if it doesn't match `self.shape()`.
  - If `Layout.cast()` raises, reject the assignment.
- Otherwise, proceed as normal.

Modify `lib.enum.EnumView.eq(other)` to add the following checks:
- If `other` is an `EnumView`, reject the assignment if enum types are not identical.
- Otherwise, if `other` is another `ValueCastable`, reject the assignment.
- Otherwise, proceed as normal.

Rejected assignments are a warning in Amaranth 0.6 and becomes a hard error in Amaranth 0.7.

## Drawbacks
[drawbacks]: #drawbacks

- This will add an implied requirement for a `ValueCastable` to implement `.eq()` to be usable with `lib.wiring`. Currently a `ValueCastable` is not required to implement `.eq()` at all.
  - We could fall back to `Value.cast().eq()` when `.eq()` is not defined.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Signals like `first` and `last` could be added as separate ports in the stream signature, preventing incompatible connections.
  - This was already discussed in [RFC 61](0061-minimal-streams.md) and decided against.
- An explicit hook for checking compatibility between interfaces could be added, instead of relying on `.eq()`.
  - This RFC proposes picking the low-hanging fruits first, making use of already existing mechanisms.
    If this is not enough, another RFC can propose an explicit hook later.

## Prior art
[prior-art]: #prior-art

[RFC 61](0061-minimal-streams.md) already discussed the need to eventually make `lib.wiring.connect()` stricter to prevent the connection of stream interfaces with incompatible payloads.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None (yet).

## Future possibilities
[future-possibilities]: #future-possibilities

- A hook can still be added to allow a signature/interface to perform an explicit compatibility check, for cases where signatures have identical members but still have metadata indicating they are incompatible.
  - This hook could also allow overriding the existing checks, allowing connections where interfaces are compatible despite differences in members.
