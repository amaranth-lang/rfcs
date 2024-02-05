- Start Date: 2024-02-05
- RFC PR: [amaranth-lang/rfcs#46](https://github.com/amaranth-lang/rfcs/pull/46)
- Amaranth Issue: [amaranth-lang/amaranth#1081](https://github.com/amaranth-lang/amaranth/issues/1081)

# Change `Shape.cast(range(1))` to `unsigned(0)`

## Summary
[summary]: #summary

Change `Shape.cast(range(1))` to return `unsigned(0)` instead of `unsigned(1)`.

## Motivation
[motivation]: #motivation

Currently, `Shape.cast(range(1))` returns `unsigned(1)`. This is inconsistent with the expectation that casting `range` to a shape will return the minimal shape capable of representing all elements of that range, which is clearly `unsigned(0)`.

This behavior is an accidental result of using `bits_for` internally to determine required width for both endpoints, which returns `1` for an input of `0`.

The behavior introduces edge cases in unexpected places. For example, one may expect that `Memory(depth=depth).read_port().addr.width == ceil_log2(depth)`. This is currently false for `depth` of 1 for no particularly good reason.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

`Shape.cast(range(1))` is changed to return `unsigned(0)`. The same applies to any other range whose only element is `0`, like `Shape.cast(range(0, 2, 2))`.

Arguably, this change requires a negative amount of exlanation, since it removes an edge case and brings the behavior into alignment with the language reference.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

See above.

## Drawbacks
[drawbacks]: #drawbacks

This is a minor backwards compatibility hazard.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The change itself is simple enough that it cannot really be done any other way.

It would be possible to introduce a compatibility warning to the 0.4 branch. However, a `range()` signal having a shape that's slightly too large is unlikely to cause problems in the first place, so the warning would cause a mass of false positives without a nice way to turn it off.

## Prior art
[prior-art]: #prior-art

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

The `bits_for` function, which led to this issue in the first place, could be deprecated and removed from public interface to avoid introducing similar problems to external code. Otherwise, it should at the very least be documented and loudly call out this special case.
