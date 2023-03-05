- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Aggregate data structure extensibility

## Summary
[summary]: #summary

Provide well-defined extension points for the aggregate data structure library.

## Motivation
[motivation]: #motivation

[RFC 1]: 0001-aggregate-data-structures.md

[RFC 1] introduces the aggregate data structure library, which allows using any shape-castable object as the shape of a field. Layouts do not consider the specific type of the shape-castable object. However, views do, and depending on whether it's a layout, a subclass of an aggregate class (`Struct` or `Union`), or any other shape-castable object, behavior differs:

```python
>>> from amaranth import *
>>> from amaranth.lib.data import *
>>> class S(Struct):
...     x: unsigned(1)
...
>>> layout = StructLayout({
...     "a": unsigned(1),
...     "b": ArrayLayout(unsigned(2), 4),
...     "c": S
... })
...
>>> View(layout).a
(slice (sig $signal) 0:1)
>>> View(layout).b
<amaranth.lib.data.View object at 0x7f53e46934c0>
>>> View(layout).c
<__main__.S object at 0x7f53e4693a30>
```

At the moment this behavior is not well-defined and it special-cases the aggregate classes defined in `amaranth.lib.data`.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Any shape-castable object can be used as the shape of a field in a layout. This includes another layout. If the object is a callable (provides a `__call__` method), then when a view is accessed, the `__call__` method will be called with a single argument, the slice of the underlying value, which will be returned by the view. A `Layout` is a callable that constructs a `View` from itself and the value.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `Layout` class has a method `__call__`. `layout(value)` is equivalent to `View(layout, value)`.

The `View.__getitem__` method (and by extension `View.__getattr__`), after extracting a `field` from the layout, attempts to call `field.shape.__call__(value_slice)`, where `value_slice` is the slice of the underlying value corresponding to the field. If there is no such method, it iteratively calls `as_shape()` on `field.shape` or the result of the previous call to `as_shape()` until an object is returned that has a `__call__` method, or until an instance of `Shape` is returned.

## Drawbacks
[drawbacks]: #drawbacks

* The syntax may be confusing.
  * Using `__call__` to implement construction is a widespread pattern in Python. Moreover, `Struct` and `Union` are classes, whose `__call__` method forwards to `__new__`, so implementing this behavior would remove the special case for aggregate base classes without additional code.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design is, as far as the author knows, the smallest possible addition that provides the largest possible extensibility and removes all special casing of aggregate base classes. That it requires no additional functionality to be implemented on the aggregate base classes indicates that it fits well into the existing design.

Alternatives:
* Do not do this.

## Prior art
[prior-art]: #prior-art

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
