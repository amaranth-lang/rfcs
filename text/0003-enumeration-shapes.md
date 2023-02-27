- Start Date: 2023-02-27
- RFC PR: [amaranth-lang/rfcs#3](https://github.com/amaranth-lang/rfcs/pull/3)
- Amaranth Issue: [amaranth-lang/amaranth#756](https://github.com/amaranth-lang/amaranth/issues/756)

# Summary
[summary]: #summary

Allow explicitly specifying shape for enumerations as an alternative to implicitly inferring it.

# Motivation
[motivation]: #motivation

Hardware development includes a lot of enumerated values, so first class support for enumerations is important, and so is integration with the standard Python mechanisms for specifying enumerations.

Amaranth accepts `enum.Enum` subclasses anywhere a shape is expected, and `enum.Enum` instances anywhere a value is expected:

```python
>>> from amaranth import *
>>> from enum import Enum
>>> class Kind(Enum):
...     MUL = 0
...     ADD = 1
...     SUB = 2
...
>>> Shape.cast(Kind)
unsigned(2)
>>> Value.cast(Kind.SUB)
(const 2'd2)
```

However, this does not cover an important use case: a enumeration where many values are reserved. For example, if the `Kind` enumeration above may need to be extended in the future, it would be necessary to reserve space for additional values, which may require additional storage bits. Right now there is no way to specify that `Kind` should be cast to e.g. `unsigned(4)`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Amaranth standard library module, `amaranth.lib.enum` can be used as a drop-in replacement for the Python standard library `enum` module. It exports the same classes as the ones provided by Python's `enum` (namely `Enum`, `Flag`, `IntEnum`, and `IntFlag`) and provides the same functionality, adding the possibility of specifying a shape for the enumeration when it is defined:

```python
>>> from amaranth.lib.enum import Enum
>>> class Kind(Enum, shape=unsigned(4)):
...    MUL = 0
...    ADD = 1
...    SUB = 2
...
>>> Shape.cast(Kind)
unsigned(4)
>>> Value.cast(Kind.SUB)
(const 4'd2)
```

If the `shape=` keyword argument is not specified, the enumeration is treated exactly the same as the corresponding standard Python class.

If the values specified for the members are not representable with the explicitly provided shape, a warning is emitted:

```python
>>> class Funct3(Enum, shape=unsigned(3)):
...     SUB = 8
...
<stdin>:1: RuntimeWarning: Value of enumeration member <Funct3.SUB: 8> will be truncated to enumeration shape unsigned(3)
>>> class Funct3(Enum, shape=unsigned(3)):
...     SUB = -1
...
<stdin>:1: RuntimeWarning: Value of enumeration member <Funct3.SUB: -1> is signed, but enumeration shape is unsigned(3)
```

A shape that is specified for a base class will be inherited in subclasses:

```python
>>> class Enum3(Enum, shape=unsigned(3)): pass
...
>>> class Funct3(Enum3):
...     SUB = 2
...
>>> Shape.cast(Funct3)
unsigned(3)
```

If a enumeration without an explicitly defined shape is used in a concatenation, a warning is emitted:

```python
>>> class Kind(Enum):
...     ADD = 1
...
>>> Cat(Kind.ADD)
<stdin>:1: SyntaxWarning: Argument #1 of Cat() is an enumeration Kind.ADD without a defined shape used in bit vector context; define the enumeration by inheriting from the class in amaranth.lib.enum and specifying the 'shape=' keyword argument
(cat (const 1'd1))
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The Amaranth standard library module, `amaranth.lib.enum`, exports all of the public names of the Python standard library `enum` module. The `EnumMeta` class adds the functionality for storing and casting to shapes, and inherits from `ShapeCastable`. The `Enum`, `Flag`, `IntEnum`, and `IntFlag` classes in this module derive from `enum.Enum`, `enum.Flag`, `enum.IntEnum`, and `enum.IntFlag` respectively, and use `amaranth.lib.enum.EnumMeta` as their metaclass, which makes subclasses of `amaranth.lib.enum.Enum`, etc be instances of `ShapeCastable`.

When a new `amaranth.lib.enum.Enum` subclass is defined, `amaranth.lib.enum.EnumMeta.__new__` checks that the enumeration members are valid (currently, Amaranth requires these to be integers), and if the `shape=` argument is provided, stores it in an internal attribute. Importantly, the attribute is only set if the argument is provided, making it possible to distinguish these cases later. It also checks that all of the members can be represented by the given shape.

When an `amaranth.lib.enum.Enum` subclass is cast to a shape, if the internal attribute is set, the shape in it is returned. Otherwise it is cast to a shape using exactly the same logic as what `Shape.cast` uses for `enum.Enum` subclasses.

When an instance of a `enum.Enum` subclass is used in a concatenation, and it is not an instance of `ShapeCastable`, or if it lacks the `_amaranth_shape_` attribute, a warning is emitted. This approach avoids a circular dependency between `amaranth.hdl.ast` and `amaranth.lib.enum`.

# Drawbacks
[drawbacks]: #drawbacks

* Introducing a new standard library module increases the API surface.
* The names of enumeration base classes are the same as the standard library enumeration base classes, which may be confusing.
* Deriving from a different class requires changes to the enumeration at its point of definition, meaning that it is not possible to annotate a enum that comes from an external library with an Amaranth shape.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Ultimately, this feature boils down to defining an internal variable on an enum, which is then used by `Shape.cast` and other core Amaranth code. There are a few possible options for doing this.

1. Special class variable:

    ```python
    class SomeEnum(enum.Enum):
        _amaranth_shape_ = unsigned(4)
    ```

2. Class decorator:

    ```python
    @amaranth.shape(unsigned(4))
    class SomeEnum(enum.Enum):
    ```

3. Class keyword argument (this proposal):

    ```python
    class SomeEnum(amaranth.lib.enum.Enum, shape=unsigned(4)):
    ```

Alternative (1) has the following drawbacks:
  * It is not possible to check that the enumeration members can be represented by the specified shape at the point of definition.
  * It exposes what should be an implementation detail to the user.
  * The documentation for the standard `enum` module does not specify whether it's OK to use `_sunder_` names for one's own purposes, but it would have to be a part of the stable API.

Its advantages are:
  * No additional methods or classes in the API surface.
  * `_amaranth_shape_` makes it immediately clear what's going on.
  * The variable can be defined on any enum, even an external one.

Alternative (2) has the following drawbacks:
  * It's not clear where the `shape` decorator should be. It can only be applied to enums, but there's no enum-specific namespace in Amaranth core to put it into.
  * `SomeEnum` would inherit from the standard `Enum` class and therefore `isinstance(SomeEnum, ShapeCastable)` would be `False` unless `ShapeCastable.__instancecheck__` is overridden to fix that.

Its advantages are:
  * The decorator can be applied to an external enum.

Alternative (3) has the drawbacks specified above, and the following advantages:
  * `isinstance(SomeEnum, ShapeCastable)` naturally works.
  * As a consequence, no additional code is required in the core language. All of the functionality necessary for the feature to work lives in `amaranth.lib.enum`.
  * The `shape` argument matches `Signal(shape=)` (even though no one uses the keyword form) and works the way one would naturally expect.
  * Uses of `import enum` can be transparently replaced with `from amaranth.lib import enum` without updating the call sites, making the migration as easy as the other alternatives.

# Prior art
[prior-art]: #prior-art

None.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

# Future possibilities
[future-possibilities]: #future-possibilities

None.
