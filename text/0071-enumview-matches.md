- Start Date: 2024-07-29
- RFC PR: [amaranth-lang/rfcs#71](https://github.com/amaranth-lang/rfcs/pull/71)
- Amaranth Issue: [amaranth-lang/amaranth#1468](https://github.com/amaranth-lang/amaranth/issues/1468)

# `EnumView.matches`

## Summary
[summary]: #summary

A `matches` method is added to `EnumView`, performing a similar function to `Value.matches`.

## Motivation
[motivation]: #motivation

Amaranth currently has `Value.matches`, which checks whether a value matches any of the given patterns. An obvious application of such a feature is using it with an enumerated value. However, as it stands, `EnumView` doesn't provide a corresponding method natively, requiring a cast to `Value`. Additionally, casting the `EnumView` to `Value` erases the enumeration type, providing no protection against matching with values from the wrong type.

This RFC proposes to fill that gap by adding a type-safe `EnumView.matches` method.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `EnumView.matches` method can be used to check whether an enumerated value belongs to a given set:

```
class Color(enum.Enum, shape=3):
    BLACK = 0
    RED = 1
    GREEN = 2
    BLUE = 4
    WHITE = 7

s = Signal(Color)
m.d.comb += is_grayscale.eq(s.matches(Color.BLACK, Color.WHITE))
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The following method is added:

- `EnumView.matches(*values)`: equivalent to `self.as_value().matches(*values)`, but requires all `values` to be a member of the same enum as the `EnumView`, raising a `TypeError` otherwise

## Drawbacks
[drawbacks]: #drawbacks

- more moving parts in the language
- more general pattern matching facilities have been requested on multiple occasions, which this facility could end up redundant with

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The proposed functionality is an obvious extension of `Value.matches` to the `EnumView` class. The proposed typechecking is consistent with current `EnumView` behavior.

## Prior art
[prior-art]: #prior-art

`Value.matches`, Rust `matches!()`.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None for `EnumView`. However, there have been discussions of having more advanced pattern matching facilities in the language in general.