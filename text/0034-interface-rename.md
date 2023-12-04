- Start Date: 2023-12-04
- RFC PR: [amaranth-lang/rfcs#34](https://github.com/amaranth-lang/rfcs/pull/34)
- Amaranth Issue: [amaranth-lang/amaranth#985](https://github.com/amaranth-lang/amaranth/issues/985)

# Rename `amaranth.lib.wiring.Interface` to `PureInterface`

## Summary
[summary]: #summary

The `Interface` class in `amaranth.lib.wiring` is renamed to `PureInterface`, to avoid the impression that it is used for *all* interfaces.

## Motivation
[motivation]: #motivation

The current naming of the `Interface` class wrongly suggests that it is the base class to be used for all interfaces, and that `isinstance(foo, Interface)` is a valid check for an interface. However, this is in stark contrast to how `lib.wiring` works: any object can be an interface, as long as it has a `signature` property and compliant members. This misleads users (and, on at least two occasions, amaranth developers), making them write buggy code.

Additionally, the naming makes spoken language ambiguous in a bad way, as it is impossible to tell apart "an interface" and "an Interface".

Therefore, this RFC proposes to rename `Interface` to something more specific and reflecting its function.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `Interface` class in `amaranth.lib.wiring` is renamed to `PureInterface`.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `Interface` class in `amaranth.lib.wiring` is renamed to `PureInterface`.

## Drawbacks
[drawbacks]: #drawbacks

Minor churn.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The new name is, of course, subject to bikeshedding. The names that have been proposed are:

- `PureInterface`
- `BareInterface`

## Prior art
[prior-art]: #prior-art

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

The name `Interface` that has just been freed up can be reused for an ABC-like class representing all valid interfaces.
