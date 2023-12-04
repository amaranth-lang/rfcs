- Start Date: 2023-12-04
- RFC PR: [amaranth-lang/rfcs#35](https://github.com/amaranth-lang/rfcs/pull/35)
- Amaranth Issue: [amaranth-lang/amaranth#986](https://github.com/amaranth-lang/amaranth/issues/986)

# Add `ShapeLike`, `ValueLike`

## Summary
[summary]: #summary

Two special classes are added to the language: `ShapeLike` and `ValueLike`. They cannot be constructed, but can be used to determine with `isinstance` and `issubclass` to determine whether something can be cast to `Shape` or a `Value`, respectively.

## Motivation
[motivation]: #motivation

As it stands, we have multiple types of objects that can be used as shapes (`Shape`, `ShapeCastable`, `int`, `range`, `EnumMeta`) and values (`Value`, `ValueCastable`, `int`, `Enum`). These types have no common superclass, so there's no easy way to check if an object can be used as a shape or a value, save for actually calling `Shape.cast` or `Value.cast`. Introducing `ShapeLike` and `ValueLike` provides an idiomatic way to perform such a check.

Additionally, when type annotations are in use, there is currently no simple type that can be used for an argument that takes an arbitrary shape- or value-castable object. These new classes provide such a simple type.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In Amaranth, multiple types of objects can be cast to shapes:

- actual `Shape` objects
- `ShapeCastable` objects
- non-negative integers
- `range` objects
- `Enum` subclasses with const-castable values

To check whether an object is of a type that can be cast to a shape, `isinstance(obj, ShapeLike)` can be used. To check whether a type can be, in general, cast to a shape, `issubclass(cls, ShapeLike)` can be used.

Likewise, multiple types of objects can be cast to values:

- actual `Value` objects
- `ValueCastable` objects
- integers
- values of `Enum` subclasses with const-castable values

To check whether an object is of a type that can be cast to a value, `isinstance(obj, ValueLike)` can be used. To check whether a type can be, in general, cast to a value, `issubclass(cls, ValueLike)` can be used.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A `ShapeLike` class is provided. It cannot be constructed, and can only be used with `isinstance` and `issubclass`, which are overriden by a custom metaclass.

`issubclass(cls, ShapeLike)` returns `True` for:

- `Shape`
- `ShapeCastable` and its subclasses
- `int` and its subclasses
- `range` and its subclasses
- `enum.EnumMeta` and its subclasses

`isinstance(obj, ShapeLike)` returns `True` for:

- instances of `Shape`
- instances of `ShapeCastable` and its subclasses
- non-negative `int` values (and `int` subclasses)
- `enum.Enum` subclasses where every value is a `ValueLike`

Similarly, a `ValueLike` class is provided.

`issubclass(cls, ValueLike)` returns `True` for:

- `Value` and its subclasses
- `ValueCastable` and its subclasses
- `int` and its subclasses
- `enum.Enum` subclasses where every value is a `ValueLike`

`isinstance(obj, ValueLike)` returns `True` iff `issubclass(type(obj), ValueLike)` returns `True`.

## Drawbacks
[drawbacks]: #drawbacks

More moving parts in the language.

`isinstance(obj, ShapeLike)` does not actually guarantee that `Shape.cast(obj)` will succeed — the instance check looks only at surface-level information, and an exception can still be thrown. `issubclass(cls, ShapeLike)` is, by necessity, even more inaccurate.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There are many ways to implement the instance and subclass checks, some more precise (and complex) than others. The semantics described above are a compromise.

For `isinstance`, a simple variant would be to just try `Shape.cast` or `Value.cast` and see if it raises an exception. However, this will sometimes result in `isinstance(MyShapeCastable(), ShapeLike)` returning `False`, which may be very unintuitive and hide bugs.

The check for a valid shape-castable enum described above is an approximation — the actual logic used requires all values of an enum to be *const*-castable, not just value-castable. However, there is no way to check this without actually invoking `Value.cast` on the enum members.

## Prior art
[prior-art]: #prior-art

Python has the concept of abstract base classes, such as `collections.abc.Sequence`, which can be used for subclass checking even if they are not actual superclasses of the types involved. `ShapeLike` and `ValueLike` are effectively ABCs, though they do not use the actual ABC machinery (due to having custom logic in instance checking).

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should the exact details of the instance and subclass checks be changed?

## Future possibilities
[future-possibilities]: #future-possibilities

A similar ABC-like class has been proposed for `lib.wiring` interfaces.
