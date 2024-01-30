- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#42](https://github.com/amaranth-lang/rfcs/pull/42)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# `Const` from shape-castable

## Summary
[summary]: #summary

Allow passing a shape-castable to `Const`.

## Motivation
[motivation]: #motivation

We currently have two incompatible syntaxes for making a constant, depending on whether it's made from a `Shape` or a shape-castable.
The former uses `Const(value, shape)`, while the latter requires `shape.const(value)`.

Making `Const` accept shape-castables means we'll have a syntax that works for all shape-likes, reducing the need to special case for shape-castables.

## Guide- and reference-level explanation
[guide-level-explanation]: #guide-level-explanation

`Const(value, shape)` checks whether `shape` is a shape-castable and returns `shape.const(value)` when this is the case.

## Drawbacks
[drawbacks]: #drawbacks

- A `Const()` constructor sometimes returning non-`Const` objects can be confusing.
  - `Signal()` already behaves this way.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- This is consistent with how `Signal()` handles shape-castables.

Alternatives:
- Do not do this. Require code that makes constants from a passed shape-like to check whether it got passed a shape-castable or not and pick the appropriate syntax.

## Prior art
[prior-art]: #prior-art

[RFC #15](0015-lifting-shape-castables.md) added the equivalent behavior to `Signal()`.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
