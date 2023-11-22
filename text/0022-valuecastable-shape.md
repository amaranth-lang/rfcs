- Start Date: 2023-08-22
- RFC PR: [amaranth-lang/rfcs#22](https://github.com/amaranth-lang/rfcs/pull/22)
- Amaranth Issue: [amaranth-lang/amaranth#876](https://github.com/amaranth-lang/amaranth/issues/876)

# Define `ValueCastable.shape()`

## Summary
[summary]: #summary

Require value-castable objects to have a method that returns their high-level shape.

## Motivation
[motivation]: #motivation

[RFC #15][] advanced the extensibility of the language significantly, but broke constructs like `Signal.like(Signal(data.StructLayout(...)))`. In addition, not having a well-defined point for returning the high-level shape (i.e. a shape-castable object from which this value-castable object was creaed rather than a `Shape` instance) causes workarounds such as `data.Layout.of` to be added to the language and standard library.

[RFC #15]: 0015-lifting-shape-castables.md

## Explanation
[guide-level-explanation]: #guide-level-explanation

The `ValueCastable` interface has a method `.shape()`. This method returns a shape-castable object. Where possibe, this object should, when passed to `Signal`, create a value-castable object of the same type.

`amaranth.lib.data.Layout.of` is removed immediately.

## Drawbacks
[drawbacks]: #drawbacks

- Increased API surface area
  - At one point a commitment was made that the only method `ValueCastable` will ever define will be `as_value`. However, radical changes to the language such as [RFC #15][] make it reasonable to revisit this.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This change is the minimal possible one that fixes the problem systemically. Some minor variations in the design are possible:
- Instead of requiring `shape()` to be defined (which is a breaking change), this method can be added optionally in the next release and be required in the release after that.
  - This is difficult to do with `ValueCastable` and will require workarounds both in the core language implementation and in downstream code that operates on `ValueCastable` objects.
  - `ValueCastable` is not very widely used yet and the breakage will likely be minimal.

## Prior art
[prior-art]: #prior-art

It is typical for a programming language to have a way of retrieving the type of a value. The mechanism being added here is equivalent.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
