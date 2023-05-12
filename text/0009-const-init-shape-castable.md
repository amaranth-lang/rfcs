- Start Date: 2023-05-11
- RFC PR: [amaranth-lang/rfcs#9](https://github.com/amaranth-lang/rfcs/pull/9)
- Amaranth Issue: [amaranth-lang/amaranth#771](https://github.com/amaranth-lang/amaranth/issues/771)

# Constant initialization for shape-castable objects

## Summary
[summary]: #summary

Add an extension point to shape-castable objects, for converting constant initializers (typically Python literals) to Amaranth constant expressions.

## Motivation
[motivation]: #motivation

[RFC 1]: 0001-aggregate-data-structures.md

[RFC 1] specifies that the `reset=` argument of a `View` accepts structured data:

```python
flt_neg_reset = data.View(float32_layout, reset={"sign": 1})
```

This structured data is internally turned into an integer constant that is supplied to `Signal(reset=)`. This mechanism is not exposed to Amaranth developers. However, this creates a clear unmet need, since at the moment there is no way to turn a layout and a field-to-value mapping into a constant integer for use elsewhere.

For example, if a signal is created manually and not through the `View`, this will not work despite looking reasonable (the layout is shape-castable and can be supplied to `Signal`):

```python
flt_neg_reset_signal = Signal(float32_layout, reset={"sign": 1})
```

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Shape-castable objects must, in addition to the mandatory `.as_shape()` method, implement a mandatory `.const(value)` method to define how a constant initializer (the reset value of a `Signal` or `View`) is converted to an Amaranth constant.

This method is defined by shape-castable objects to convert arbitrary Python objects into Amaranth constants. For example, if a shape-castable object has complex internal structure, it can accept a dictionary with the values to be filled into various bits of this structure. If `Shape` implemented `ShapeCastable`, the method would be defined as `def const(self, value): return Const(value, self)`.

This method can also be directly called by the developer to construct a constant using a given shape-castable object.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A method `def const(self, obj):` is added on `ShapeCastable`.

The `Signal(shape, reset=)` constructor is changed so that if `isinstance(shape, ShapeCastable)`, then `shape.const(reset)` is used instead of `reset`.

The `.const()` instance method is implemented on `Layout` to accept a `Sequence` or `Mapping` instance and returns a `Const` with a bit pattern that has the fields set to the given values. Overlapping fields are written in the order of iteration of the input. If the field shape is a shape-castable object, then the value for that field is computed using `Const.cast(value, field.shape)`.

The `.const()` method is implemented on the metaclass of `Struct` and `Union` as `return self.as_shape().const(obj)`.

The `View(..., reset=)` constructor is changed to pass `reset` through to the `Signal()` constructor.

## Drawbacks
[drawbacks]: #drawbacks

* Additional method on `ShapeCastable`.
  * It was clear from the beginning that this functionality will likely be necessary, and we are unlikely to ever add more.
* The `reset=` argument becomes dependently typed.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Given:

```python
class Point(Struct):
    x: 16
    y: 16
```

it is clear that there needs to be some way to go from `{"x": 123, "y": 456}` to `Cat(C(123, 16), C(456, 16))` without manually writing out the concatenation.

There are two main options for this:
1. Implement a new method, such that `Point.const({"x": 123, "y": 456})` returns a constant of some kind (either an `int` or a `Const` or a constant-castable expression).
2. Implement an additional `.__init__()` override, such that `Point({"x": 123, "y": 456})` returns a view that has a constant-castable expression as its target.

Option (1) has the benefit of making it clear when downstream code is creating a constant (and expects an argument where the nested data must all be constant), and of minimizing useless wrapping/unwrapping of views that would otherwise happen. It is an explicit type conversion (from a literal to `Const`).

Option (2) avoids introducing new names. It is an implicit type conversion (from a literal to a view, which in this case is `Point`).

In the end, option (1) seems preferable here since implicit type conversions are easy to unintentionally misuse. It also avoids any clashes with proposed RFC 8.

## Prior art
[prior-art]: #prior-art

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
