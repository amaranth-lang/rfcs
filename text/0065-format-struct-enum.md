- Start Date: 2024-04-08
- RFC PR: [amaranth-lang/rfcs#65](https://github.com/amaranth-lang/rfcs/pull/65)
- Amaranth Issue: [amaranth-lang/amaranth#1293](https://github.com/amaranth-lang/amaranth/issues/1293)

# Special formatting for structures and enums

## Summary
[summary]: #summary

Extend `Format` machinery to support structured description of aggregate and enumerated data types.

## Motivation
[motivation]: #motivation

When a `lib.data.Layout`-typed signal is included in the design, it is often useful to treat it as multiple signals for debug purposes:

- in pysim VCD output, it is useful to have each field as its own trace
- in RTLIL output, it is useful to include a wire for each field (in particular, for CXXRTL simulation purposes)

Likewise, `lib.enum.Enum`-typed signals benefit from being marked as enum-typed in RTLIL output.

Currently, both of the above usecases are covered by the private `ShapeCastable._value_repr` interface. This interface has been added in a rush for the Amaranth 0.4 release as a stopgap measure, to ensure switching from `hdl.rec` to `lib.data` doesn't cause a functionality regression ([amaranth-lang/amaranth#790](https://github.com/amaranth-lang/amaranth/issues/790)). Such forbidden coupling between `lib` and `hdl` via private interfaces is frowned upon, and the time has come to atone for our sins and create a public interface that can be used for all sorts of aggregate and enumerated data types.

Since both needs relate to presentation of shape-castables, we propose piggybacking the functionality on top of `ShapeCastable.format` machinery. At the same time, we propose implementing formatting for `lib.data` and `lib.enum`.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Shape-castables implementing enumerated types can return a `Format.Enum` instance instead of a `Format` from their `format` method:

```py
class MyEnum(ShapeCastable):
    ...

    def format(self, obj, spec):
        assert spec == ""
        return Format.Enum(Value.cast(obj), {
            0: "ABC",
            1: "DEF",
            2: "GHI",
        })
```

For formatting purposes, this is equivalent to:

```py
class MyEnum(ShapeCastable):
    ...

    def format(self, obj, spec):
        assert spec == ""
        return Format(
            "{:s}",
            Mux(Value.cast(obj) == 0, int.from_bytes("ABC".encode(), "little"),
            Mux(Value.cast(obj) == 1, int.from_bytes("DEF".encode(), "little"),
            Mux(Value.cast(obj) == 2, int.from_bytes("GHI".encode(), "little"),
            int.from_bytes("[unknown]".encode(), "little")
            )))
        )
```

with the added benefit that any `MyEnum`-shaped signals will be automatically marked as enums in RTLIL output.

Likewise, shape-castables implementing aggregate types can return a `Format.Struct` instance instead of a `Format`:

```py
class MyStruct(ShapeCastable):
    ...

    def format(self, obj, spec):
        assert spec == ""
        return Format.Struct({
            # Assume obj.a, obj.b, obj.c are accessors that return the struct fields as ValueLike.
            "a": Format("{}", obj.a),
            "b": Format("{}", obj.b),
            "c": Format("{}", obj.c),
        })
```

For formatting purposes, this is equivalent to:

```py
class MyStruct(ShapeCastable):
    ...

    def format(self, obj, spec):
        assert spec == ""
        return Format("{{a: {}, b: {}, c: {}}}", obj.a, obj.b, obj.c)
        return Format.Struct(Value.cast(obj), {
            # Assume obj.a, obj.b, obj.c are accessors that return the struct fields as ValueLike.
            "a": Format("{}", obj.a),
            "b": Format("{}", obj.b),
            "c": Format("{}", obj.c),
        })
```

with the added benefit that any `MyStruct`-shaped signal will automatically have per-field traces included in VCD output and per-field wires included in RTLIL output.

Implementations of `format` are added as appropriate to `lib.enum` and `lib.data`, making use of the above features.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Three new classes are added:

- `amaranth.hdl.Format.Enum(value: Value, /, variants: EnumType | dict[int, str])`
- `amaranth.hdl.Format.Struct(value: Value, /, fields: dict[str, Format])`
- `amaranth.hdl.Format.Array(value: Value, /, fields: list[Format])`

Instances of these classes can be used wherever a `Format` can be used:

- as an argument to `Print`, `Assert`, ...
- included within another `Format` via substitution
- returned from `ShapeCastable.format`
- as a field format within `Format.Struct` or `Format.Array`

When used for formatting:

- `Format.Enum` will display as whichever string corresponds to current value of `value`, or as `"[unknown]"` otherwise.
- `Format.Struct` will display as `"{a: <formatted field a>, b: <formatted field b>}"`.
- `Format.Array` will display as `"[<formatted field 0>, <formatted field 1>]"`

Whenever a signal is created with a shape-castable as its shape, its `format` method is called with `spec=""` and the result is stashed away.

VCD output is done as follows:

1. When a signal or field's format is just `Format("{}", value)`: value is emitted as a bitvector to VCD.
2. Otherwise, when a signal or field's custom format is not `Format.Struct` nor `Format.Array`: the format is evaluated every time the value changes, and the result is emitted as a string to VCD
3. When the custom format is `Format.Struct` or `Format.Array`:
   - the `value` as a whole is emitted as a bitvector
   - each field is emitted recursively (as a separate trace, perhaps with subtraces)

RTLIL output is done as follows:

1. When a signal or field's format is a plain `Format` and contains exactly one format specifier: a wire is created and assigned with the specifier's value.
2. When a signal or field's format is a plain `Format` that doesn't conform to the above rule: no wire is created.
3. When a signal or field's format is a `Format.Enum`: a wire is created and assigned with the format's `value`. RTLIL attributes are emitted on it, describing the enumeration values.
4. When a signal or field's format is a `Format.Struct` or `Format.Array`: a wire is created and assigned with the format's `value`, representing the struct as a whole. For every field of the aggregate, the rules are applied recursively.

## Drawbacks
[drawbacks]: #drawbacks

More language complexity.

A shape-castable using `Format.Struct` to mark itself as aggregate is forced to use the fixed `Format.Struct` display when formatted.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design ties together concerns of formatting with structural description. An alternative would be to have separate hooks for those, like the current `_value_repr` interface.

## Prior art
[prior-art]: #prior-art

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

The current `decoder` interface on `Signal` could be deprecated and retired.