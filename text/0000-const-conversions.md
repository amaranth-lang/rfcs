- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Const-castable Amaranth value to Python value conversions

## Summary
[summary]: #summary

Allow transparent conversion of constant-castable Amaranth values to Python values via the built-in `__int__`, `__bool__`, and `__index__` methods.

## Motivation
[motivation]: #motivation

[RFC 4] introduced constant-castable objects and provided the following motivating example (edited for clarity here):

```python
class Funct(enum.Enum):
    ADD = 0
    ...

class Op(enum.Enum):
    REG = 0
    IMM = 1

class Instr(enum.Enum):
    ADD  = Cat(Funct.ADD, Op.REG)
    ADDI = Cat(Funct.ADD, Op.IMM)
    ...
```

However, this doesn't actually work:

```
>>> from amaranth import *
>>> import enum
>>> class Funct(enum.Enum):
...     ADD = 0
...     ...
...
>>> class Op(enum.Enum):
...     REG = 0
...     IMM = 1
...
>>> class Instr(enum.Enum):
...     ADD  = Cat(Funct.ADD, Op.REG)
...     ADDI = Cat(Funct.ADD, Op.IMM)
...     ...
...
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/lib/python3.9/enum.py", line 277, in __new__
    if canonical_member._value_ == enum_member._value_:
  File ".../amaranth/amaranth/hdl/ast.py", line 175, in __bool__
    raise TypeError("Attempted to convert Amaranth value to Python boolean")
TypeError: Attempted to convert Amaranth value to Python boolean
```

It fails because the `enum.Enum` class attempts to check the members for uniqueness. Doing this, it attempts to convert `Cat(Funct.ADD, Op.REG) == Cat(Funct.ADD, Op.IMM)` (which is an Amaranth value, `(== (cat (const 1'd0) (const 1'd0)) (cat (const 1'd0) (const 1'd1)))`) to boolean.

[RFC 4]: 0004-const-castable-exprs.md

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Constant-castable values may be directly converted to integers and used in Python expressions and conditionals:

```python
>>> Const(1) == Const(1)
(== (const 1'd1) (const 1'd1))
>>> bool(Const(1) == Const(1))
True
>>> if Const(1) == Const(1):
...     print("yes")
...
yes
```

Values that are not constant-castable will cause an exception to be raised if they are used in the same way:

```
>>> bool(Signal(1) == Const(1))
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File ".../amaranth/amaranth/hdl/ast.py", line 175, in __bool__
    raise TypeError("Attempted to convert non-constant-castable Amaranth value to a Python value")
TypeError: Attempted to convert non-constant-castable Amaranth value to a Python value
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The builtin methods `__int__`, `__bool__`, and `__index__` are implemented on the `Value` class. They return the numeric value as a Python `int` or `bool` for constant-castable values, and raise TypeError for any other values.

## Drawbacks
[drawbacks]: #drawbacks

* This RFC represents a dramatic departure from the status quo, where (during the runtime of the Python code) Amaranth values are used solely to describe computations, and Python values are used solely to perform computations.
  * Great effort has been put to ensure that Amaranth `Value`s numerically behave exactly the same as Python `int`s.
* This RFC makes it possible to pass Amaranth values to unsuspecting third-party code and have it mostly work (so far as only common numeric operations are performed, like `a + b`), only to break at a later point where an operation that is defined on `int` but not on `Value` is done (like `.bit_length()`, or `.__str__()`)
  * A typechecker should catch this.
* This RFC makes it possible to write code operating on constant-castable Amaranth values that breaks when a non-constant-castable Amaranth value is introduced.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Python values are implicitly lifted to Amaranth values as necessary. Although the Amaranth representation is richer (it has an explicit shape), for arithmetic expressions on constants, the Amaranth operations are intentionally designed in a way that exactly matches Python semantics. Now that [RFC 4] added explicit lowering to Amaranth, it is reasonable to consider also performing implicit lowering as necessary. Any case where `Const.cast(x.__op__(y))` returns a different result depending on whether `x` and `y` are Amaranth or Python values is already considered a bug, so introducing this functionality should not create additional hazards.

Doing this solely to enable the motivating `Enum` example is clearly overkill, although it is not trivial to work around this and it is highly desirable to enable the motivating example to work due to its usefulness in building CPU-like blocks. However, it seems like a viable solution since the language as it exists already does everything to make such conversions safe.

[RFC 3]: 0003-enumeration-shapes.md

Alternative: Do not do this.
* Amend [RFC 3] to only allow constant-castable member values when inheriting from `amaranth.lib.enum.Enum`.
  * It's not really feasible to hack the `Enum` implementation to allow AST nodes to be stored in enums if they are not evaluatable as described above.
  * Implement this by irreversibly converting constant-castable expressions to their numeric value and storing that.

## Prior art
[prior-art]: #prior-art

Lifting and lowering values in this manner is a widely used approach. It is particularly common in functional languages. The values are typically not implicitly lifted and/or lowered, however, but implicit type conversions are idiomatic in Python.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

Which other unintended consequences could this change have?

## Future possibilities
[future-possibilities]: #future-possibilities

We could define other `int` methods on `Value` to make the use of constant-castable values as integers more transparent.

  * Already defined: `__invert__`, `__neg__`, `__add__`, `__radd__`, `__sub__`, `__rsub__`, `__mul__`, `__rmul__`, `__mod__`, `__rmod__`, `__floordiv__`, `__rfloordiv__`, `__lshift__`, `__rlshift__`, `__rshift__`, `__rrshift__`, `__and__`, `__rand__`, `__xor__`, `__rxor__`, `__or__`, `__ror__`, `__eq__`, `__ne__`, `__lt__`, `__le__`, `__gt__`, `__ge__`, `__abs__`
  * How do we not already define it?: `__pos__`
  * Probably: `__divmod__`, `__rdivmod__`, `bit_length`
  * Maybe: `__pow__`, `__rpow__`
    * The backend actually supports these still.
    * There were some issues matching sign inference for `__pow__` between Verilog and Python, and it was removed.
  * Probably not: `__ceil__`, `__float__`, `__floor__`, `__round__`, `__rtruediv__`, `__truediv__`, `__trunc__`, `as_integer_ratio`, `denominator`, `numerator`, `conjugate`, `imag`, `real`
    * These are all floating point related.
  * Unclear: `to_bytes`, `from_bytes`
