- Start Date: (fill in with date at which the RFC is merged, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#66](https://github.com/amaranth-lang/rfcs/pull/66)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Simulation time

## Summary
[summary]: #summary

Add a type for representing time durations and simulator methods to query the elapsed time.

## Motivation
[motivation]: #motivation

In [RFC #36](0036-async-testbench-functions.md), there were a plan to introduce a `.time()` method to query the current simulation time, but it got deferred due to the lack of a suitable return type.

Internally simulation time is an integer number of femtoseconds, but exposing that directly is not ergonomic and making it a `float` of seconds sacrifices precision the longer a simulation runs.

Also, to quote @whitequark: Time is not a number.

## Guide- and reference-level explanation
[guide-level-explanation]: #guide-level-explanation

A new type `amaranth.sim.Period` is introduced.
It is immutable, has no public constructor and is instead constructed through the following classmethods:
- `.s(duration: numbers.Real)`
- `.ms(duration: numbers.Real)`
- `.us(duration: numbers.Real)`
- `.ns(duration: numbers.Real)`
- `.ps(duration: numbers.Real)`
- `.fs(duration: numbers.Real)`
- `.Hz(frequency: numbers.Real)`
- `.kHz(frequency: numbers.Real)`
- `.MHz(frequency: numbers.Real)`
- `.GHz(frequency: numbers.Real)`
  - The argument will be scaled according to the SI prefix on the method and the `duration` or reciprocal of `frequency` then used to calculate the closest integer femtosecond representation.

To convert it back to a number, the following properties are available:
- `.seconds -> fractions.Fraction`
- `.milliseconds -> fractions.Fraction`
- `.microseconds -> fractions.Fraction`
- `.nanoseconds -> fractions.Fraction`
- `.picoseconds -> fractions.Fraction`
- `.femtoseconds -> int`

The following operators are defined:
- `.__lt__(other: Period) -> bool`
- `.__le__(other: Period) -> bool`
- `.__eq__(other: Period) -> bool`
- `.__ne__(other: Period) -> bool`
- `.__gt__(other: Period) -> bool`
- `.__ge__(other: Period) -> bool`
- `.__hash__() -> int`
- `.__bool__() -> bool`
- `.__neg__() -> Period`
- `.__pos__() -> Period`
- `.__abs__() -> Period`
- `.__add__(other: Period) -> Period`
- `.__sub__(other: Period) -> Period`
- `.__mul__(other: numbers.Real) -> Period`
- `.__rmul__(other: numbers.Real) -> Period`
- `.__truediv__(other: numbers.Real) -> Period`
- `.__truediv__(other: Period) -> fractions.Fraction`
- `.__floordiv__(other: Period) -> int`
- `.__mod__(other: Period) -> Period`
  - Operators on `Period`s are performed on the underlying femtosecond values.
  - Operators involving `numbers.Real` operands have the result rounded back to the closest integer femtosecond representation.
  - Operators given unsupported operand combinations will return `NotImplemented`.

`Simulator` and `SimulatorContext` both have a `.time() -> Period` method added that returns the elapsed time since start of simulation.

The methods that currently take seconds as a `float` are updated to take a `Period`:
- `.add_clock()`
- `.delay()`

The ability to pass seconds as a `float` is deprecated and removed in a future Amaranth version.

## Drawbacks
[drawbacks]: #drawbacks

- Deprecating being able to pass seconds as a `float` creates churn.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- `.add_clock()` is usually passed the reciprocal of a frequency, so being able to construct a `Period` from a frequency instead of a duration is useful.
  - We could add a separate `Frequency` type, but in practice it would likely almost exclusively be used to calculate the reciprocal `Period`.

- Accepting a `numbers.Real` when we're converting from a number lets us pass in any standard suitable number type.

- When converting to a number, `fractions.Fraction` is a standard `numbers.Real` type that is guaranteed to represent any possible value exactly, and is convertible to `int` or `float`.
  - Returning `int` would force a potentially undesirable truncation/rounding upon the user.
  - Returning `float` would lose precision when the number of femtoseconds are larger than the significand.
  - `decimal.Decimal` is decimal floating point and thus has the same issue as `float`.
  - `.femtoseconds` returns `int` because that's a perfect representation and returning a fraction with a denominator of 1 adds no value.

- The supported set of operators are the ones that have meaningful and useful semantics:
  - Comparing, adding and subtracting time periods makes sense.
  - Multiplying a time period with or dividing it by a real number scales the time period.
  - Dividing time periods gives the ratio between them.
  - Modulo of time periods is the remainder and thus still a time period.
    Potentially useful to calculate the phase offset between a timestamp and a clock period.
  - Reflected operators that only accept `Period` operands are redundant and omitted.

## Prior art
[prior-art]: #prior-art

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- The usual bikeshedding.
  - Case, e.g. `.MHz` vs `.mhz`.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
