- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#41](https://github.com/amaranth-lang/rfcs/pull/41)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Fixed point types

## Summary
[summary]: #summary

Add fixed point types to Amaranth.

## Motivation
[motivation]: #motivation

Fractional values in hardware are usually represented as some form of fixed point value.
Without a first class fixed point type, the user has to manually manage how the value needs to be shifted to be represented as an integer and keep track of how that interacts with arithmetic operations.

A fixed point type would encode and keep track of the precision through arithmetic operations, as well as provide standard operations for converting values to and from fixed point representations with correct rounding.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

TODO

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This RFC proposes a library addition `amaranth.lib.fixedpoint` with the following contents:

`FixedPoint` is a `ShapeCastable` subclass.
The following operations are defined on it:

- `FixedPoint(f_width, /, *, signed)`: Create a `FixedPoint` with `f_width` fractional bits.
- `FixedPoint(i_width, f_width, /, *, signed)`: Create a `FixedPoint` with `i_width` integer bits and `f_width` fractional bits.
- `FixedPoint.cast(shape)`: Cast `shape` to a `FixedPoint` instance.
- `.i_width`, `.f_width`, `.signed`: Width and signedness properties.
- `.const(value)`: Create a fixed point constant from an `int` or `float`, rounded to the closest representable value.
- `.as_shape()`: Return the underlying `Shape`.
- `.__call__(target)`: Create a `FixedPointValue` over `target`.

`Q(*args)` is an alias for `FixedPoint(*args, signed=True)`.

`UQ(*args)` is an alias for `FixedPoint(*args, signed=False)`.

`FixedPointValue` is a `ValueCastable` subclass.
The following operations are defined on it:

- `FixedPointValue(shape, target)`: Create a `FixedPointValue` with `shape` over `target`.
- `FixedPointValue.cast(value)`: Cast `value` to a `FixedPointValue`.
- `.i_width`, `.f_width`, `.signed`: Width and signedness properties.
- `.shape()`: Return the `FixedPoint` this was created from.
- `.as_value()`: Return the underlying value.
- `.eq(value)`: Assign `value`.
  - If `value` is a `FixedPointValue`, the precision will be extended or rounded as required.
  - If `value` is an `int` or `float`, the value will be rounded to the closest representable value.
  - If `value` is a `Value`, it'll be assigned directly to the underlying `Value`.
- `.round(f_width=0)`: Return a new `FixedPointValue` with precision changed to `f_width`, rounding as required.
- `.__add__(other)`, `.__radd__(other)`, `.__sub__(other)`, `.__rsub__(other)`, `.__mul__(other)`, `.__rmul__(other)`: Binary arithmetic operations.
  - If `other` is a `Value` or an `int`, it'll be cast to a `FixedPointValue` first.
  - If `other` is a `float`: TBD
  - The result will be a new `FixedPointValue` with enough precision to hold any resulting value without rounding or overflowing.
- `.__neg__()`, `.__pos__()`, `.__abs__()`: Unary arithmetic operations.

## Drawbacks
[drawbacks]: #drawbacks

TBD

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- `FixedPointValue.eq()` could cast a `Value` to a `FixedPointValue` first, and thereby shift the value to the integer part instead of assigning directly to the underlying value.
  However, `Value.eq()` would always grab the underlying value of a `FixedPointValue`, and for consistency both `.eq()` operations should behave in the same manner.
  - If we wanted to go the other way, this RFC could be deferred until another RFC allowing `ValueCastable` to override reflected `.eq()` have been merged.
    However, having to explicitly do `value.eq(fp_value.round())` when rounding is desired is arguably preferable to having `value.eq(fp_value)` do an implicit rounding.

- Unlike `.eq()`, it makes sense for arithmetic operations to cast a `Value` to `FixedPointValue`.
  Multiplying an integer with a fixedpoint constant and rounding the result back to an integer is a reasonable and likely common thing to want to do.

## Prior art
[prior-art]: #prior-art

[Q notation](https://en.wikipedia.org/wiki/Q_(number_format)) is a common and convenient way to specify floating point types.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- What should we do if a `float` is passed as `other` to an arithmetic operation?
  - We could use `float.as_integer_ratio()` to derive a perfect fixed point representation.
    However, since a Python `float` is double precision, this means it's easy to make a >50 bit number by accident by doing something like `value * (1 / 3)`, and even if the result is rounded or truncated afterwards, the lower bits can affect rounding and thus won't be optimized out in synthesis.
  - We could use the same width for `other` as for `self`, adjusted to the appropriate exponent for the value.
  - We could outright reject it, requiring the user to explicitly specify precision like e.g. `value * Q(15).const(1 / 3)`.

- There's two slightly different [Q notation](https://en.wikipedia.org/wiki/Q_(number_format)) definitions, namely whether the bit counts includes the sign bit or not.
  `UQ(15)` and `UQ(7, 8)` would be 15 bits in either convention, but `Q(15)` and `Q(7, 8)` would be 15 or 16 bits depending on the convention. Which do we pick?

- Are there any other operations that would be good to have?

- Bikeshed all the names.

## Future possibilities
[future-possibilities]: #future-possibilities

TBD
