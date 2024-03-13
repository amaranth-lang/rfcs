- Start Date: 2024-03-25
- RFC PR: [amaranth-lang/rfcs#58](https://github.com/amaranth-lang/rfcs/pull/58)
- Amaranth Issue: [amaranth-lang/amaranth#1243](https://github.com/amaranth-lang/amaranth/issues/1243)

# Core support for `ValueCastable` formatting

## Summary
[summary]: #summary

`Format` hooks are added, allowing custom formatting to be implemented for `ValueCastable`s.

This RFC is only about adding hook support for custom `ShapeCastable`s. Providing actual formatting implementation for `lib.data` and `lib.enum` is left for a future RFC.

## Motivation
[motivation]: #motivation

Custom types, like enums and data structures, will not make immediate sense to the user when printed as raw bit patterns. However, it is often desirable to print them out with the `Print` statement. Thus, an extension point for custom formatting needs to be added to the `Format` machinery.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

`ShapeCastable` subtypes can define their own ways of formatting values by implementing the `format` hook:

```py
class FixedPoint(ShapeCastable):
    ...

    def format(self, value, format_desc):
        if format_desc == "b":
            # Format a fixed-point value as binary.
            return Format("{value.int:b}.{value.fract:0{fract_bits}b}}", value=value, fract_bits=len(value.fract))
        elif format_desc == "x":
            # Format a fixed-point value as hexadecimal (very simplified implementation).
            assert len(value.fract) % 4 == 0
            return Format("{value.int:x}.{value.fract:0{fract_digits}x}}", value=value, fract_digits=len(value.fract) // 4)
        else:
            ...

# A fixed-point number with 8 integer and 8 fractional bits.
num = Signal(FixedPoint(8, 8))
m.d.comb += [
    num.eq(0x1234)
    Print(Format("Value in binary: {:b}", num)),
    Print(Format("Value in hexadecimal: {:x}", num)),
]
```

prints:

```
Value in binary: 00010010.00110100
Value in hexadecimal: 12.34
```

However, sometimes it is also useful to print the raw value of such a signal. A shorthand is provided for that purpose:

```py
m.d.comb += Print(Format("Value: {num:x} (raw: {num!v:x})", num=num))
```

```
Value: 12.34 (raw: 1234)
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A new overridable method is added to `ShapeCastable`:

- `format(self, value: ValueCastable, format_desc: str) -> Format`

When a `ValueCastable` is formatted without specifying a conversion (ie. `"!r"` or `"!v"`):

- `shape()` is called
- if the shape is a `ShapeCastable` and has a `format` method, the value-castable being formatted and the format descriptor (the part after `:` in `{}`, if any) are passed directly to `shape.format()`, and the result (which must be a `Format` object) is included directly in place of the format specifier
- otherwise (the shape is a plain `Shape`, or doesn't have `format`), `Value.cast()` is called on the value-castable, and formatting proceeds as if it was a plain value

A new conversion, `!v`, is added to `Format` (with syntax analogous to `!r`). When specified, the value being formatted is converted through `Value.cast` before proceeding with further formatting. It can be used to print the raw value of value-castables. It is a no-op when used with a plain `Value`.

An implementation of `__format__` is added to the `ValueCastable` base class that always raises a `TypeError`, directing the user to `Format` instead (like the one that already exists on plain `Value`).

## Drawbacks
[drawbacks]: #drawbacks

A new reserved name, `format`, is added to `ShapeCastable`, which is intended to be a fairly slim interface.

The `__format__` on `ValueCastable` is the first time we have a method with an actual implementation.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The `format` hook is added on `ShapeCastable` instead of `ValueCastable`. This ensures that `ValueCastable`s without a custom shape automatically get plain formatting.

The default behavior proposed in this RFC ensures that a formatting implementation is always available, allowing generic code to print arbitrary values without worrying about an exception. Eg. something like `lib.fifo` could use debug prints when an element is added, automatically using rich formatting for shapes with `format`, while falling back to plain integers when rich formatting is not available.

alternative default behaviors possible are:

- raise `TypeError` (disallow formatting value-castables without explicitly implemented formatting)
- no default, require every `ShapeCastable` to implement `format` (compatibility breaking)

To avoid reserving a name and interfering with user code, `format` could be renamed to `_amaranth_format`.

## Prior art
[prior-art]: #prior-art

This RFC is modelled directly on the `__format__` extension point in Python.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

Bikeshed: what should `!v` be called? Alternatives proposed:

- `!v` (cast to value)
- `!i` (print as integer)
- `!n` (print as number)
- `!R` (print raw)
- `!l` (lower to a value)
- `!av` (as value)

## Future possibilities
[future-possibilities]: #future-possibilities

Actual formatting will need to be implemented for `lib.data` and `lib.enum`.
