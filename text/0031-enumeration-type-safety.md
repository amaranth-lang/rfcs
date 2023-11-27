- Start Date: 2023-11-27
- RFC PR: [amaranth-lang/rfcs#31](https://github.com/amaranth-lang/rfcs/pull/31)
- Amaranth Issue: [amaranth-lang/amaranth#972](https://github.com/amaranth-lang/amaranth/issues/972)

# Enumeration type safety

## Summary
[summary]: #summary

Make Amaranth `Enum` and `Flag` use a custom `ValueCastable` view class, enforcing type safety.

## Motivation
[motivation]: #motivation

Python `Enum` provides an opaque wrapper over the underlying enum values,
providing type safety and guarding against improper usage in arithmetic
operations:

```pycon
>>> from enum import Enum
>>> class EnumA(Enum):
...     A = 0
...     B = 1
...
>>> EnumA.A + 1
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unsupported operand type(s) for +: 'EnumA' and 'int'
```

Likewise, `Flag` values can be used in bitwise operations, but only within
their own type:

```pycon
>>> from enum import Flag
>>> class FlagA(Flag):
...     A = 1
...     B = 2
... 
>>> class FlagB(Flag):
...     C = 1
...     D = 2
... 
>>> FlagA.A | FlagA.B
<FlagA.A|B: 3>
>>> FlagA.A | FlagB.C
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unsupported operand type(s) for |: 'FlagA' and 'FlagB'
```

However, these safety properties are not currently enforced by Amaranth
on enum-typed signals:

```pycon
>>> from amaranth import *
>>> from amaranth.lib.enum import *
>>> class FlagA(Flag):
...     A = 1
...     B = 2
... 
>>> class FlagB(Flag):
...     C = 1
...     D = 2
... 
>>> a = Signal(FlagA)
>>> b = Signal(FlagB)
>>> a | b
(| (sig a) (sig b))
```


## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Like in Python, `Enum` and `Flag` subclasses are considered strongly-typed,
while `IntEnum` and `IntFlag` are weakly-typed. Enum-typed Amaranth values
with strong typing are manipulated through `amaranth.lib.enum.EnumView`
and `amaranth.lib.enum.FlagView` classes, which wrap an underlying `Value`
in a type-safe container that only allows a small subset of operations.
For weakly-typed enums, `Value` is used directly, providing full
interchangeability with other values.

An `EnumView` or a `FlagView` can be obtained by:

- Creating an enum-typed signal (`a = Signal(MyEnum)`)
- Explicitly casting a value to the enum type (`MyEnum(value)`)

The operations available on `EnumView` and `FlagView` include:

- Comparing for equality to another view of the same enum type (`a == b` and `a != b`)
- Assigning to or from a value
- Converting to a plain value via `Value.cast`

The operations additionally available on `FlagView` include:

- Binary bitwise operations with another `FlagView` of the same type
  (`a | b`, `a & b`, `a ^ b`)
- Bitwise inversion (`~a`)

A custom subclass of `EnumView` or `FlagView` can be used for a given enum
type if so desired, by using the `view_class` keyword parameter on enum
creation.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

`amaranth.lib.enum.EnumView` is a `ValueCastable` subclass.  The following
operations are defined on it:

- `EnumView(enum, value_castable)`: creates the view
- `shape()`: returns the underlying enum
- `as_value()`: returns the underlying value
- `eq(value_castable)`: delegates to `eq` on the underlying value
- `__eq__` and `__ne__`: if the other argument is an `EnumView` of the same
  enum type or a value of the enum type, delegates to the corresponding
  `Value` operator; otherwise, raises a `TypeError`
- All binary arithmetic, bitwise, and remaining comparison operators: raise
  a `TypeError` (to override the implementation provided by `Value` in case
  of an operation between `EnumView` and `Value`)

`amaranth.lib.enum.FlagView` is a subclass of `EnumView`.  The following
additional operations are defined on it:

- `__and__`, `__or__`, `__xor__`: if the other argument is a `FlagView`
  of the same enum type or a value of the enum type, delegates to the
  corresponding `Value` operator and wraps the result in `FlagView`;
  otherwise, raises a `TypeError`
- `__invert__`: inverts all bits in this value corresponding to actually
  defined flags in the underlying enum type, then wraps the result in
  `FlagView`

The behavior of `EnumMeta.__call__` when called on a value-castable
is changed as follows:

- If the enum has been created with a `view_class`, the value-castable
  is wrapped in the given class
- Otherwise, if the enum type is a subclass of `IntEnum` or `IntFlag`, the
  value-castable is returned as a plain `Value`
- Otherwise, if the enum type is a subclass of `Flag`, the value-castable
  is wrapped in `FlagView`
- Otherwise, the value-castable is wrapped in `EnumView`

The behavior of `EnumMeta.const` is modified to go through the same logic.

## Drawbacks
[drawbacks]: #drawbacks

This proposal increases language complexity, and is not consistent with
eg. how `amaranth.lib.data.View` operates (which has much more lax type
checking).

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Do nothing. Operations on mismatched types will continue to be silently
allowed.

Equality could work more like Python equality (always returning false
for mismatched types).

Assignment could be made strongly-typed as well (with corresponding hook
added to `Value`).

## Prior art
[prior-art]: #prior-art

This feature directly parallels the differences between Python's
`Enum`/`Flag` and `IntEnum`/`IntFlag`.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

Instead of having an extension point via `view_class`, we could instead
automatically forward all otherwise unknown methods to the underlying enum
class, providing it the `EnumView` as `self`.

## Future possibilities
[future-possibilities]: #future-possibilities

None.