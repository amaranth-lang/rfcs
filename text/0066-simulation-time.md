- Start Date: 2024-07-29
- RFC PR: [amaranth-lang/rfcs#66](https://github.com/amaranth-lang/rfcs/pull/66)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/1469)

# Simulation time

## Summary
[summary]: #summary

Add a type for representing time durations (including clock periods) and simulator methods to query the elapsed time.

## Motivation
[motivation]: #motivation

In [RFC #36](0036-async-testbench-functions.md), there were a plan to introduce a `.time()` method to query the current simulation time, but it got deferred due to the lack of a suitable return type.

Internally simulation time is an integer number of femtoseconds, but exposing that directly is not ergonomic and making it a `float` of seconds sacrifices precision the longer a simulation runs.

Also, to quote @whitequark: Time is not a number.

## Guide- and reference-level explanation
[guide-level-explanation]: #guide-level-explanation

A new type `Period` is introduced.
It is exported from `amaranth.hdl` and also re-exported from `amaranth.sim`.
It is immutable and has a constructor accepting at most one named argument, giving the following valid forms of construction:
- `Period()`
  - Constructs a zero period.
- `Period(s: numbers.Real)`
- `Period(ms: numbers.Real)`
- `Period(us: numbers.Real)`
- `Period(ns: numbers.Real)`
- `Period(ps: numbers.Real)`
- `Period(fs: numbers.Real)`
  - The argument will be scaled according to its SI prefix and used to calculate the closest integer femtosecond representation.
- `Period(Hz: numbers.Real)`
- `Period(kHz: numbers.Real)`
- `Period(MHz: numbers.Real)`
- `Period(GHz: numbers.Real)`
  - The argument will be scaled according to its SI prefix and its reciprocal used to calculate the closest integer femtosecond representation.
  - A value of zero will raise `ZeroDivisionError`.
  - A negative value will raise `ValueError`.

To convert it back to a number, the following properties are available:
- `.seconds -> float`
- `.milliseconds -> float`
- `.microseconds -> float`
- `.nanoseconds -> float`
- `.picoseconds -> float`
- `.femtoseconds -> int`

To calculate the reciprocal frequency, the following properties are available:
- `.hertz -> float`
- `.kilohertz -> float`
- `.megahertz -> float`
- `.gigahertz -> float`
  - Accessing these properties when the period is zero will raise `ZeroDivisionError`.
  - Accessing these properties when the period is negative will raise `ValueError`.

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
- `.__truediv__(other: Period) -> float`
- `.__floordiv__(other: Period) -> int`
- `.__mod__(other: Period) -> Period`
  - Operators on `Period`s are performed on the underlying femtosecond values.
  - Operators involving `numbers.Real` operands have the result rounded back to the closest integer femtosecond representation.
  - Operators given unsupported operand combinations will return `NotImplemented`.
- `.__str__() -> str`
  - Equivalent to `.__format__("")`
- `.__format__(format_spec: str) -> str`
  - The format specifier format is `[width][.precision][ ][unit]`.
  - An invalid format specifier raises `ValueError`.
  - If `width` is specified, the string is left-padded with space to at least the requested width.
  - If `precision` is specified, the requested number of decimal digits will be emitted.
    Otherwise, duration units will emit as many digits required for an exact result, while frequency units will defer to default `float` formatting.
  - If a space is present in the format specifier, the formatted string will have a space between the number and the unit.
  - `unit` can be specified as any of the argument names accepted by the constructor.
    If a unit is not specified, the largest duration unit that has a nonzero integer part is used.
    Formatting frequency units have the same restrictions and exception behavior as accessing frequency properties.

`SimulatorContext` have an `.elapsed_time() -> Period` method added that returns the elapsed time since start of simulation.

These methods that has a `period` argument currently taking seconds as a `float` are updated to take a `Period`:
- `Simulator.add_clock()`
- `SimulatorContext.delay()`

These methods that has a `frequency` argument currently taking Hz as a `float` are updated to take a `Period`:
- `ResourceManager.add_clock_constraint()`
- `Clock.__init__()`

The ability to pass seconds or Hz directly as a `float` is deprecated and removed in a future Amaranth version.

The `Clock.period` property is updated to return a `Period` and the `Clock.frequency` property is deprecated.

Consequently, `Platform.default_clk_frequency()` is also deprecated and replaced with a new method `Platform.default_clk_period() -> Period`.


## Drawbacks
[drawbacks]: #drawbacks

- Deprecating being able to pass seconds or Hz directly as a `float` creates churn.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- `.add_clock()` is usually passed the reciprocal of a frequency, so being able to construct a `Period` from a frequency instead of a duration is useful.
  - We could add a separate `Frequency` type, but in practice it would likely almost exclusively be used to calculate the reciprocal `Period`.

- Accepting a `numbers.Real` when we're converting from a number lets us pass in any standard suitable number type.

- The supported set of operators are the ones that have meaningful and useful semantics:
  - Comparing, adding and subtracting time periods makes sense.
  - Multiplying a time period with or dividing it by a real number scales the time period.
  - Dividing time periods gives the ratio between them.
  - Modulo of time periods is the remainder and thus still a time period.
    Potentially useful to calculate the phase offset between a timestamp and a clock period.
  - Reflected operators that only accept `Period` operands are redundant and omitted.

- `.elapsed_time()` is only available from within a testbench/process, where it is well defined.

- Instead of named constructor arguments, we could use a classmethod for each SI prefix.
  This was proposed in an earlier revision of this RFC.

- Instead of returning `float`, we could return `fractions.Fraction` to ensure no precision loss when the number of femtoseconds is larger than the `float` significand.
  This was proposed in an earlier revision of this RFC.
  - Rounding to integer femtoseconds is already lossy and a further precision loss from converting to a `float` is negligible in most real world use cases.
    For any applications requiring an exact conversion of a `Period`, the `.femtoseconds` property is always exact.

## Prior art
[prior-art]: #prior-art

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
