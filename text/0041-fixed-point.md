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

A fixed point type would encode and keep track of the precision through arithmetic operations, as well as provide standard operations for converting values to and from fixed point representations.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

TODO

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This RFC proposes a library addition `amaranth.lib.fixed` with the following contents:

### `fixed.Shape`

`fixed.Shape` is a `ShapeCastable` subclass.
The following operations are defined on it:

- `fixed.Shape(shape, f_bits)`: Create a `fixed.Shape` with underlying storage `shape` and `f_bits` fractional bits.
- The signedness is inherited from `shape`, so a `fixed.Shape(signed(16), 12)` would be a signed fixed-point number, 16 bits wide with 12 fractional bits.
- A `fixed.Shape` may be constructed using the following aliases:
    - `SQ(i_bits, f_bits)` is an alias for `fixed.Shape(signed(i_bits + f_bits), f_bits)`.
    - `UQ(i_bits, f_bits)` is an alias for `fixed.Shape(unsigned(i_bits + f_bits), f_bits)`.
- `.i_bits`, `.f_bits`, `.signed`: Width and signedness properties of the `fixed.Shape`.
    - `.i_bits` includes the sign bit. That is, for `fixed.Shape(signed(16), 12)`, `.i_bits == 4`.
- `.const(value)`: Create a `fixed.Const` from `value`.
- `.as_shape()`: Return the underlying `Shape`.
- `.__call__(target)`: Create a `fixed.Value` over `target`.
- `min()`, `max()`: Returns a `fixed.Const` representing the minimum and maximum representable values of a shape.

### `fixed.Value`

`fixed.Value` is a `ValueCastable` subclass.
The following operations are defined on it:

- `fixed.Value(shape, target)`: Create a `fixed.Value` with `shape` over `target`.
- `fixed.Value.cast(value, f_bits=0)`: Cast `value` to a `fixed.Value`.
- `.i_bits`, `.f_bits`, `.signed`: Width and signedness properties.
- `.shape()`: Return the `fixed.Shape` this was created from.
- `.as_value()`: Return the underlying value.
- `.numerator()`: Return `as_value()` cast to the appropriate signedness.
- `.eq(value)`: Assign `value`.
  - If `value` is a `Value`, it'll be assigned directly to the underlying `Value`.
  - If `value` is an `int` or `float`, it'll be cast to a `fixed.Const` first.
  - If `value` is a `fixed.Value`, the precision will be extended or truncated as required.
- `.reshape(f_bits)`: Return a new `fixed.Value` with `f_bits` fractional bits, truncating or extending precision as required.
- `.reshape(shape)`: Return a new `fixed.Value` with shape `shape`, truncating or extending precision as required.
  - For example, `value1.reshape(SQ(4, 4))` * value2
- `.__add__(other)`, `.__radd__(other)`, `.__sub__(other)`, `.__rsub__(other)`, `.__mul__(other)`, `.__rmul__(other)`: Binary arithmetic operators.
  - If `other` is a `Value`, it'll be cast to a `fixed.Value` first.
  - If `other` is an `int`, it'll be cast to a `fixed.Const` first.
  - If `other` is a `float`, this is not permitted. The `float` must be explicitly cast to `fixed.Const`.
  - The result will be a new `fixed.Value` with enough precision to hold any resulting value without rounding or overflowing.
- `.__lshift__(other)`, `.__rshift__(other)`: Bit shift operators.
    - `other` only accepts integral types. For example, shifting by a `float` or `fixed.Value` is not permitted.
- `.__neg__()`, `.__pos__()`, `.__abs__()`: Unary arithmetic operators.
- `.__lt__(other)`, `.__le__(other)`, `.__eq__(other)`, `.__ne__(other)`, `.__gt__(other)`, `.__ge__(other)`: Comparison operators.
    - Comparisons between `fixed.Value` of matching size, or between `fixed.Value` and `int` are permitted.
    - Comparisons between `fixed.Value` of different `f_bits` are not permitted.
        - Users are guided by an exception to explicitly `reshape()` as needed.
    - Comparisons between `fixed.Value` and `float` are not permitted.
        - Users are guided by an exception to explicitly convert using `fixed.Const` as needed.

### `fixed.Const`

`fixed.Const` is a `fixed.Value` subclass.
The following additional operations are defined on it:

- `fixed.Const(value, shape=None, clamp=False)`: Create a `fixed.Const` from `value`. `shape` must be a `fixed.Shape` if specified.
  - If `value` is an `int` and `shape` is not specified, the smallest shape that will fit `value` will be selected.
  - If `value` is a `float` and `shape` is not specified, the smallest shape that gives a perfect representation will be selected.
    If `shape` is specified, `value` will be truncated to the closest representable value first.
  - If `shape` is specified and `value` is too large to be represented by that shape, an exception is thrown.
    - The exception invites the user to try `clamp=True` to squash this exception, instead clamping the constant to the maximum / minimum value representable by the provided `shape`.
- `.as_integer_ratio()`: Return the value represented as an integer ratio `tuple`.
- `.as_float()`: Return the value represented as a `float`.
- Operators are extended to return a `fixed.Const` if all operands are constant.

## Drawbacks
[drawbacks]: #drawbacks

TBD

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- `fixed.Value.eq()` could cast a `Value` to a `fixed.Value` first, and thereby shift the value to the integer part instead of assigning directly to the underlying value.
  However, `Value.eq()` would always grab the underlying value of a `fixed.Value`, and for consistency both `.eq()` operations should behave in the same manner.
  - If we wanted to go the other way, this RFC could be deferred until another RFC allowing `ValueCastable` to override reflected `.eq()` have been merged.
    However, having to explicitly do `value.eq(fp_value.round())` when rounding is desired is arguably preferable to having `value.eq(fp_value)` do an implicit rounding.

- Unlike `.eq()`, it makes sense for arithmetic operations to cast a `Value` to `fixed.Value`.
  Multiplying an integer with a fixedpoint constant and rounding the result back to an integer is a reasonable and likely common thing to want to do.

- There's two slightly different [Q notation](https://en.wikipedia.org/wiki/Q_(number_format)) definitions, namely whether the bit counts includes the sign bit or not.
    - Not having the sign bit included seems more common, and has the advantage that a number has the same fractional precision whether `i_bits` is 0 or not.
    - While Q notation names the signed type `Q`, it's more consistent for Amaranth to use `SQ` since other Amaranth types defaults to unsigned.
    - vk2seb@: Having the sign bit included is the dominant notation in the audio ASIC world (citation needed, comment from samimia-swks@). As of now, this RFC uses this notation as it is also a little simpler to reason about the size of underlying storage on constructing an `SQ`.

## Prior art
[prior-art]: #prior-art

[Q notation](https://en.wikipedia.org/wiki/Q_(number_format)) is a common and convenient way to specify fixed point types.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- What should we do if a `float` is passed as `other` to an arithmetic operation?
  - We could use `float.as_integer_ratio()` to derive a perfect fixed point representation.
    However, since a Python `float` is double precision, this means it's easy to make a >50 bit number by accident by doing something like `value * (1 / 3)`, and even if the result is truncated afterwards, the lower bits can affect rounding and thus won't be optimized out in synthesis.
  - We could use the same width for `other` as for `self`, adjusted to the appropriate exponent for the value.
  - We could outright reject it, requiring the user to explicitly specify precision like e.g. `value * Q(15).const(1 / 3)`.
  - vk2seb@: I would lean toward outright rejecting this, with an explicit cast necessary (now reflected above).

- How should we handle rounding?
  - Truncating and adding the most significant truncated bit is cheap and is effectively round to nearest with ties rounded towards positive infinity.
  - Simply truncating is free, rounds towards negative infinity.
  - IEEE 754 defaults to round to nearest, ties to even, which is more expensive to implement.
  - Should we make it user selectable?
    - We still need a default mode used when a higher precision number is passed to `.eq()`.
  - samimia-swks@: In most DSP applications, simple truncating is done (bit picking, which is equivalent to a floor()) because it's free. I would vote for that being the default behavior at least.
  - ld-cd@: (...) Truncate is still a reasonable default for most applications.
  - ld-cd@: (...) I think a better approach would be to leave rounding and several other common operations that require platform dependent lowering to a subsequent RFC (...).
  - vk2seb@: Both truncation and simple rounding (round to nearest) are commonly used in DSP algorithms. For now, we provide only `reshape()` (truncation, now reflected above). Additional rounding strategies may be added in a future RFC, however we will always need a default rounding strategy, and truncation seems like a sane default.

- Are there any other operations that would be good to have?
  - From ld-cd@: `min()`, `max()` on `fixed.Shape` (vk2seb@: agree, heavily use this)
  - From ld-cd@: `numerator()` as a signedness-preserving `as_value()` (vk2seb@: agree, heavily use this)
  - vk2seb@: Let's add both of these operations (now reflected above)

- Are there any operations that would be good to *not* have?
  - This API (`fixed.Shape.cast()`) seems confusing and difficult to use. Should we expose it at all? (@whitequark)
  - vk2seb@: It can survive as a building block but I don't see why it needs to be exposed. Propose removal (reflected above).

- `Decimal` and/or `Fraction` support?
  - This could make sense to have, but both can represent values that's not representable as binary fixed point.
    On the other hand, a Python `float` can perfectly represent any fixed point value up to a total width of 53 bits and any `float` value is perfectly representable as fixed point.
  - vk2seb@: As both can represent values that aren't representable as binary fixed point, I don't see a huge benefit. I also can't immediately think of an application that would need >53 bit constants. I would propose for now we leave `Decimal` and `Fraction` out of scope of this RFC.

- Name all the things.
  - Library name:
    - Bikeshed: `lib.fixed`, `lib.fixnum`. (@whitequark)
  - Type names:
    - `fixed.Shape` and `fixed.Value` are one option, though I can see why others may object to it. (@whitequark)
  - I feel like the `i_width` and `f_width` names are difficult enough to read that it's of more importance than bikeshedding to come up with something more readable. (@whitequark)
    - `.int_bits`, `.frac_bits`?
    - cursed option: `int, frac = x.width`?
  - `.round()` is a bit awkwardly named when it's used both to increase and decrease precision.
  - vk2seb@: The existing modifications address this:
    - Library name: `lib.fixed`
    - Type names and shapes (from zyp@): signature has now been updated to use `i_bits`, `f_bits` and the explicit underlying storage in the constructor for `fixed.Shape`.
    - We now have `.reshape()`, which better represents increasing and decreasing precision. However, I'm open to new names.

- Should `__div__` be permitted?
    - zyp@: (...) To avoid scope creep, I'm inclined to leave inferred division out of this RFC. We could instead do a separate RFC later for that.
    - vk2seb@: agree, let's leave it out of this RFC.

## Future possibilities
[future-possibilities]: #future-possibilities

TBD
