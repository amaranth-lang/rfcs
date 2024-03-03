- Start Date: (fill me in with today's date, 2024-03-04)
- RFC PR: [amaranth-lang/rfcs#51](https://github.com/amaranth-lang/rfcs/pull/51)
- Amaranth Issue: [amaranth-lang/amaranth#1187](https://github.com/amaranth-lang/amaranth/issues/1187)

# Add `ShapeCastable.from_bits` and `amaranth.lib.data.Const`

## Summary
[summary]: #summary

A new user-definable method is added to `ShapeCastable`: `from_bits(bits: int)`. It is used to convert constants into user-defined types, for example when a user-typed value is read in simulation.

A new type is added: `amaranth.lib.data.Const`. It is used as the return type of `Layout.from_bits`. `Layout.const` is also changed to return it instead of `View`.

## Motivation
[motivation]: #motivation

The current Amaranth model provides a useful set of primitives for manipulating opaque signals and values of custom types, via `ShapeCastable` and `ValueCastable`.  However, there is little support for manipulating constants of custom types.

This is reasonable, as Amaranth mostly deals with elaborating a design hierarchy that operates on symbolic values that won't have actual values assigned before the circuit is operational. However,  this need comes up in the Amaranth simulator, where the values have to be actually materialized.

Specifically, the simulator requires two operations to usefully support `ValueCastable` expressions:

1. Conversion from user-facing constant value to raw bits, used when a `ValueCastable` experession is set to a value, or a cell of `ShapeCastable`-typed `Memory` is written by a user process. This need is already covered by the existing `ShapeCastable.const`.
2. Conversion from raw bits to user-facing constant value, used when a `ValueCastable` expression is requested, or a cell of `ShapeCastable`-type `Memory` is read by a user process. This need will be covered by the proposed `from_bits` method.

For `amaranth.lib.enum`, the implementation of `from_bits` is trival: the value is looked up in the enum, and the enum value is returned. However, we have no good type to return from `from_bits` for `amaranth.lib.data`. Thus, a new `lib.data.Const` class is proposed.


## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A new method is added to `ShapeCastable`: `from_bits(bits: int)`, which, given a constant, interprets the constant as a value of the given `ShapeCastable` and returns an arbitrary Python value. The returned value must be round-trippable back to the original constant via `ShapeCastable.const`. This method will be used whenever there is a need to translate a raw bit pattern through a `ShapeCastable`, such as when reading a `ShapeCastable`-typed value in simulation.

```
>>> class Abc(amaranth.lib.enum.Enum, shape=unsigned(2)):
...     X = 0
...     Y = 1
...     Z = 2
... 
>>> Abc.from_bits(2)
<Abc.Z: 2>
```

To support this functionality in `amaranth.lib.data`, a new `lib.data.Const` class is added, which wraps a constant. Like `View`, this class provides attribute-based and indexing-based access to `Layout` fields. However, when a field is accessed, `lib.data.Const` returns a raw int value (for plain `Shape`-typed fields), or calls `from_bits` on the underlying bit pattern (for `ShapeCastable` fields). The class is also `ValueCastable`.

```
>>> class Def(amaranth.lib.data.Struct):
...     a: Abc
...     b: unsigned(2)
... 
>>> a = Def.from_bits(9)
>>> a
Const(StructLayout({'a': <enum 'Abc'>, 'b': unsigned(2)}), 9)
>>> a.a
<Abc.Y: 1>
>>> a.b
2
```

Since it is a better fit, the return type of `Layout.const` is changed to `lib.data.Const`.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A new method is added on `ShapeCastable`: `from_bits(bits: int)`. The default implementation raises an error. A warning is raised when this method is not overriden in a subclass. The warning will become a hard error in Amaranth 0.6.

The argument to the method must be an `int` that is a valid constant value of the `ShapeCastable`'s underlying shape.

The method must return something acceptable to the `const` method on the same `ShapeCastable`, and the value must round-trip, that is:

```
# assume `bits` is an int that is a valid const of `shape_castable.as_shape()`
value = shape_castable.const(shape_castable.from_bits(bits)).as_value()
assert isinstance(value, Const)
assert value.value == bits
```

It is recommended, but not strictly required, that the returned value is a `ValueLike`.

The `sim.get` method in the upcoming RFC 36 will call this method when called on a `ValueCastable` with a custom shape. The `sim.set` method will likewise call `ShapeCastable.const`.

A `from_bits` implementation is added onto `amaranth.lib.enum.EnumMeta`. If the given bit pattern has a corresponding enum value, the enum value is returned. Otherwise, the argument is returned unchanged.

A new class is added:

- `amaranth.lib.data.Const(layout, bits: int)`: bits must be a valid constant of the underlying shape; derives from `ValueCastable`
  - `shape()`: returns the underlying shape
  - `as_value()`: returns `Const(bits, len(layout))`
  - `__getitem__`:
    - resolves the field like `View.__getitem__`
    - extracts the corresponding field from `bits`
    - if the field is a `ShapeCastable`, returns the result of calling its `from_bits` with the extracted value
    - otherwise, returns the raw field value as an `int`
  - `__getattr__`: resolves the field like `View.__getattr__`, then proceeds as above
  - `__eq__` and `__ne__`:
    - if the other value is a `lib.data.Const` with the same layout, compares the underlying `bits` and returns a Python `bool`
    - if the other value is a `View` with the same layout, delegates to `Value.__eq__` (and thus returns a 1-bit `Value`)
    - otherwise, raises `TypeError`
  - all other operators: raises `TypeError` (to prevent the default `Value` operator implementations)

A `from_bits` implementation is added to `Layout`, `Struct`, and `Union`. It returns a `lib.data.Const` instance.

`Layout.const`, `Struct.const`, and `Union.const` are changed to return a `lib.data.Const` of matching layout. They are also changed to accept such constants if given, and return them unchanged.

## Drawbacks
[drawbacks]: #drawbacks

Adds an extra hook onto `ShapeCastable` that everyone must implement. However, it is not clear how this could be avoided.

When a `Struct` (or `Union`) is involved, `isinstance(sim.get(my_struct_signal), MyStruct)` will be false. This could be avoided by horrifying metaclass hacks, but it's unclear if it's a good idea.

The change to `Layout.const` etc is not fully backwards-compatible. It is believed that this is not much of an issue, since its main user is `Signal` `init=` argument, for which it doesn't matter.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The proposed design of `from_bits` is essentially the minimal interface needed to support the needs of simulation. Additionally, it is designed so that it can be called essentially recursively by aggregate `ShapeCastable`s, such as `Layout`.

An alternative (proposed originally by RFC 36) would be to let `ValueCastable` implement `get`/`set` methods for simulation. However, this has obvious drawbacks:

- two methods needed per `ShapeCastable`
- doesn't work for reading or writing memories, as they don't have underlying signals
- when the value is an aggregate, there's no obvious way to delegate to construct the values of fields

The `lib.data.Const` design closely follows `View`. An alternative would be for `from_bits` to return a list or dict of fields (recursively processed by `from_bits`), paralleling the accepted argument format of `const`. However, this results in inferior user experience.

`lib.data.Const` conceivably could be made mutable, providing `__setattr__` / `__setitem__` in the future. However, Amaranth tries to avoid mutable data types, and it was felt that keeping it explicitly immutable was the best way forward. Its main use is believed to be simulation, and in simulation one can just call `sim.set()` on the `View` field directly if needed. Outside of simulation, we can add a `.like()`-like constructor for `lib.data.Const` to modify some fields of the aggregate.

## Prior art
[prior-art]: #prior-art

The author believes user-defined types usually come in three parts:

- a `ShapeCastable`, ie. the type itself
- a `ValueCastable` proxying arbitrary `Value`s
- a class representing constant values of the type

For enums, these correspond to `EnumMeta`, `EnumView`, and the `Enum` itself. For plain values, they correspond to `Shape`, `Value`, and `int`. The proposed RFC 41 likewise includes `fixed.Shape`, `fixed.Value`, and `fixed.Const`.

The first half of this RFC provides a generic way to construct the third part from a bit pattern. The second half of this RFC fills the missing third part for `amaranth.lib.data`.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

The `lib.data.Const` class can be expanded in functionality:

- alternative constructors could be added, to let user create and manipulate const struct-typed values
