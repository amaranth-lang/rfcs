- Start Date: 2023-05-15
- RFC PR: [amaranth-lang/rfcs#15](https://github.com/amaranth-lang/rfcs/pull/15)
- Amaranth Issue: [amaranth-lang/amaranth#784](https://github.com/amaranth-lang/amaranth/issues/784)

# Lifting shape-castable objects

## Summary
[summary]: #summary

Make `Signal(shape_castable, ...)` return `shape_castable(Signal(shape_castable.as_shape(), ...))`.

## Motivation
[motivation]: #motivation

When Amaranth was a very new language, it did not have any facilities for ascribing type information to data. It had shapes (width and signedness), and it had special handling for `range()` in the shape position, as well as enumerations. Back then it made sense to have `Signal`, the single way to define new storage of any kind, to only operate on values (numbers / bit containers).

Today the situation is completely different. Amaranth has first-class support for enumerations in the standard library as well as the standard range of data structures (structs, unions, arrays) via [RFC 1] and [RFC 3]. It provides extensibility through [RFC 8] and [RFC 9]. Using the existing hooks alone it is possible to extend Amaranth with rich numeric types (fixed-point, complex, potentially even floating-point), and some of these are very likely to end up in the standard library.

All of this new functionality internally wraps a `Value`. It is so common and useful to initialize e.g. a struct view with a fresh `Signal` that `data.View` reexports all of the arguments of the `Signal` constructors and automatically constructs a `Signal` if no view target is provided. This works, but ties the two together more than would be ideal, and requires every similar facility to reimplement the functionality itself. What is worse is that it seems to be quite confusing to programmers, since it's not apparent that calling `data.View(foo_layout)` internally creates a `Signal`. Furthermore, people want to call `Signal(foo_layout)` to construct some storage for `foo_layout`, and that works (`foo_layout` is shape-castable), but does the wrong thing: the returned object is a `Signal`, not a `data.View`.

It would make teaching a lot easier if we could draw an equivalence between a `Signal` and a variable in a general purpose programming language, and between its shape and a type in a general purpose programming language. Then, no matter what shape-castable object it is, the way to make some storage is `Signal(x)`. It will also simplify the internals a fair bit.

This change wasn't practical before [RFC 8] and [RFC 9] laid the groundwork for it, but now it is an obvious extension.

[RFC 1]: 0001-aggregate-data-structures.md
[RFC 3]: 0003-enumeration-shapes.md
[RFC 8]: 0008-aggregate-extensibility.md
[RFC 9]: 0009-const-init-shape-castable.md

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

To include state in a design, use the `Signal(shape)` constructor, where `shape` describes the bit layout and possible operations on that state. The `reset=` argument and the returned value depend on the `shape` that is provided. If it is `signed(N)` or `unsigned(N)` or a built-in enumeration or a `range`, then a plain `Value` is returned, and the `reset=` argument accepts a number, an enumeration member, or a constant. If it is a `data.Layout`, then a `data.View` is returned, and the `reset=` argument accepts a sequence or a mapping, potentially nested for nested layouts. Other shape-castable classes will have their own behavior.

> **Warning**
> The existing syntax for creating a `View` with a new `Signal` underlying it will be removed immediately (it has never been in a release) to resolve an ambiguity over the semantics of `__call__`.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A method `def __call__(self, value):` is added on `ShapeCastable`. It must return `Value` or a `ValueCastable` instance with the right shape. (Such a method is opportunistically used by `data.View` for nested views since [RFC 8], however this RFC makes it mandatory for all shape-castable objects.)

The `Signal.__call__(shape, ...)` method is overridden (on the metaclass) to consider `shape`. First, a `signal` is constructed normally with all of the arguments. Afterwards, if `shape` is a `ShapeCastable` instance, then `shape(signal)` is returned. Otherwise `signal` is returned.

## Drawbacks
[drawbacks]: #drawbacks

* Increase in language complexity.
* More metaclasses.
  * `Signal` is a final class so this is unlikely to go wrong.
* A `Signal()` constructor sometimes returning non-`Signal` objects can be confusing.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There are several arguments in favor of the design:
* It does not de facto introduce any new methods on protocols, since `ShapeCastable.__call__` is expected to be implemented by essentially everyone after [RFC 8].
* It does not introduce new complexity to `Signal.__init__`; the logic for handling non-integer reset exists since [RFC 9].
* It eliminates unnecessary coupling between `data.View` (and other similar facilities) and `Signal()`.
* It is a natural extension of the language and has clear parallels to the notion of variables in other languages.
* It has been repeatedly requested by users, almost every time someone became familiar with the aggregate data structure design.

All of these points are compelling but the last one perhaps the most. The author did not find it a stark enough necessity to introduce themselves but it does seem to be one.

Alternatives:
  * Do not do this. The status quo is acceptable.

## Prior art
[prior-art]: #prior-art

This RFC brings the semantics of `Signal` to be very close to semantics of typed variables in other languages.

"Lifting" in the title of this RFC refers to a [concept in functional programming](https://wiki.haskell.org/Lifting) of the same name where a higher order function (`Signal`, here) is used to generalize an operation over a set of other functions (`data.View` and other shape-castable objects that implement the `__call__` protocol, here).

## Unresolved questions
[unresolved-questions]: #unresolved-questions

* How does this interact with typechecking?
  * This is a straightforward higher order function so it's probably fine.

## Future possibilities
[future-possibilities]: #future-possibilities

This RFC is the final one in a chain that started with [RFC 1].

Enumerations and ranges could be adjusted such that something other than `Value` is returned. This creates backwards compatibility concerns though.
