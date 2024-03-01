- Start Date: 2024-03-04
- RFC PR: [amaranth-lang/rfcs#50](https://github.com/amaranth-lang/rfcs/pull/50)
- Amaranth Issue: [amaranth-lang/amaranth#1186](https://github.com/amaranth-lang/amaranth/issues/1186)

# `Print` statement and string formatting

## Summary
[summary]: #summary

A `Print` statement is added, providing for simulation-time printing. A `Format` object, based on Python format strings, is added to support the `Print` statement. The existing `Assert`, `Assume`, and `Cover` statements are modified to make use of `Format` functionality as well.

## Motivation
[motivation]: #motivation

This functionality has been requested multiple times, for debugging purposes. While debug printing can be provided externally via Python testbenches, having it as a statement allows it to be embedded directly into the module, without having to carefully plumb the required state into the simulator process. Since it can be passed to yosys, it is also usable for cross-language simulation.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A `Print` statement can be used to print design state during simulation:

```
ctr = Signal(16)
m.d.sync += [
    ctr.eq(ctr + 1),
    Print("counter:", ctr),
]
```

```
counter: 0
counter: 1
counter: 2
...
```

The `Print` statement is modeled after Python's `print` function and likewise supports `sep=` and `end=` keyword arguments.

For more advanced formatting, `Print` can be paired with `Format`, which corresponds to Python string formatting and uses a subset of the same syntax:

```
ctr = Signal(16)
m.d.sync += [
    ctr.eq(ctr + 1),
    Print(Format("Counter: {ctr:04x}", ctr=ctr)),
]
```

```
...
Counter: fffe
Counter: ffff
Counter: 0000
Counter: 0001
Counter: 0002
...
```

The `Format` functionality supports printing `Value`s as integers, in `b`, `d`, `o`, `x`, `X` format types, with most of the relevant formatting options. The `c` format, which prints the `Value` treating it as a Unicode code pointer, is also supported.

In addition, the Amaranth-specific `s` format is supported, which prints a `Value` as a Verilog-style string (but with opposite byte ordering to Verilog): the `Value` is treated as a string of octets starting from LSB, any NUL bytes are trimmed, and the octets are interpreted as UTF-8.

Formatting `ValueCastable` is not supported, and is left for a future RFC. Formatting anything other than `Value` and `ValueCastable` will be done at elaboration time by defering to normal Python string formatting.

The `Assert`, `Assume`, and `Cover` statements are extended to take an optional message (which can be a `Format` object) that will be printed when the statement is triggered:

```
m.d.sync += Assert(ctr < 10, message=Format("ctr value {} is out of bounds", ctr))
```

```
assertion failed at foo.py:13: ctr value 17 is out of bounds
```

When message is not included, `Assert` and `Assume` will get a default message, while `Cover` will execute silently.

The `Print`, `Assert`, `Assume`, and `Cover` statements are supported in the Python simulator, in CXXRTL simulator, and (to the best of target language's ability) in Verilog output.

In pysim, `Print` prints to standard output. `Assert` and `Assume` will throw an `AssertionError` when the test is false, with the formatted message included in the exception message. The `Cover` statement prints the message to standard output together with its source location, and the actual coverage tracking functionality is not implemented yet.

While `Assert` executes identically to `Assume` (with the exception of failure message), `Assert` is meant for checking post-conditions, while `Assume` is meant for checking pre-conditions.


## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### `Format` objects

A new class is introduced for the formatting functionality:

- `amaranth.hdl.Format(format_string, /, *args, **kwargs)`

The format string uses exactly the same format string grammar as `str.format`. Any arguments that are not `Value` or `ValueCastable` will be formatted immediately when the `Format` object is constructed and essentially inlined into the format string. Any `ValueCastable` arguments are an error (as a placeholder for a future RFC which will specify `ValueCastable` formatting). Any `Value` arguments will be stored, to be formatted at simulation time.

For `Value`, the following subset of standard format specifiers is supported:

- fill character: supported
- alignment: `<`, `>`, `=` (but *not* `^`)
- sign: `-`, `+`, ` `
- `#` and `0` options: supported
- width: supported, but must be an elaboration-time constant (ie. cannot be another `Value`)
- grouping option: `_` is supported, `,` is *not* supported
- type:
  - `b`, `c`, `d`, `o`, `x`, `X`: supported, with Python semantics
  - `s`: the corresponding `Value` must be a multiple of 8 bits wide; the value is converted into an UTF-8 string LSB-first, any 0 bytes present in the string are removed, and the result is printed as a string

Two `Format` objects can be concatenated together with the `+` operator.

The `Format` class is added to the prelude.

### The `Print` statement

A `Print` statement is added:

- `amaranth.hdl.Print(*args, sep=" ", end="\n")`

Any argument that is not a `Format` instance will be implicitly converted by wrapping it in `Format("{}", arg)`.

When the statement is executed in simulation, all `Format` objects will be evaluated to strings, the results will be joined together with the `sep` argument, the `end` argument will be appended at the end, and the resulting string will be printed directly to standard output.

A `Print` statement is considered active iff all `If`, `Case`, and `State` constructs in which it is contained are active.

`Print` statements contained in `comb` domains will be executed:

- at the beginning of the simulation, if active at that point
- whenever they become active after being previously inactive
- whenever any referenced `Value` changes while they are active

`Print` statements contained in non-`comb` domains will be executed whenever they are active on the relevant clock edge.

The `Print` statement is added to the prelude.

### `Assert`, `Assume`, `Cover`

The statements become:

- `amaranth.hdl.Assert(test, message=None)`
- `amaranth.hdl.Assume(test, message=None)`
- `amaranth.hdl.Cover(test, message=None)`

The `name` argument is deprecated and removed.

The `message` argument can be either `None`, a string, or a `Format` object. If it is a string, it's equivalent to passing `Format("{}", s)`.

Whenever an `Assert` or `Assume` is active and `test` evaluates to false, the simulator throws an `AssertionError` and includes the formatted `message`, if any, in the payload.

Whenever a `Cover` is active and `test` evaluates to true, and the `Cover` has a message attached, the formatted message is printed to standard output along with the source location.

The `Assert` statement is added to the prelude. The `Assume` and `Cover` statements need to be imported manually from `amaranth.hdl`. Their old location in `amaranth.asserts` is deprecated.

### Miscellaneous

To guard against accidentally using plain Python formatting, an implementation of `__format__` is added to `Value` that always raises a `TypeError` with a message directing the user to `Format`. If the old behavior of printing the AST serialization is desired for some reason, `"{value!r}"` can still be used.

## Drawbacks
[drawbacks]: #drawbacks

This is some quite complex functionality. It cannot be fully represented in Verilog output, since Verilog is quite limitted in its formatting capabilities.

It is not fully clear what `Assume` and especially `Cover` semantics should be, and they are very much formal verification oriented.

Unfortunately the `f""` syntax is not supported. The hacks that would be necessary to make this work are too horrifying even for the authors of this RFC.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The design follows Python prior art, not leaving much space for discussion. The exact subset of supported format specifiers is, of course, to be bikeshedded.

The `s` format specifier is included to provide a way to implement data-dependent string printing, eg. for printing an enum value. It is expected this may be used for lowering future `ValueCastable` printing.

Signals that are enum-typed could get special default formatting. However, this will be solved by future `ValueCastable` hooks for Amaranth enums.

## Prior art
[prior-art]: #prior-art

The design of `Print`, `Format` and `Assert` closely follows Python, as much as is easily possible.

The `s` format specifier is based directly on Verilog strings.

The `Print` statement corresponds and synthesizes to RTLIL `$print` cell, aligning closely to its capabilities.

The `Print` statement is loosely modelled on Verilog's `$display` and related task. The `Assert`, `Assume`, and `Cover` statements are loosely based on corresponding SystemVerilog statements.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- What exact format specifiers should be supported?

## Future possibilities
[future-possibilities]: #future-possibilities

`ValueCastable` formatting can and should be implemented by an extension hook on the `ValueCastable` class.

`Format` could be made into a (non-synthesizable) `Value` itself, corresponding to `$sformat` in Verilog. This would be useful for creating more complex formatting for `ValueCastables`.

Some kind of output redirection or output hook could be implemented in pysim.

Likewise, a hook for assertion failure could be desirable.

Actual coverage tracking could be implemented in pysim for `Cover`.

More exotic formatting could be useful (eg. for the proposed fixed point numbers).

Some way to print the current simulation time could be useful.
