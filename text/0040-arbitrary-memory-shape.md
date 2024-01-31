- Start Date: 2024-01-22
- RFC PR: [amaranth-lang/rfcs#40](https://github.com/amaranth-lang/rfcs/pull/40)
- Amaranth Issue: [amaranth-lang/amaranth#1048](https://github.com/amaranth-lang/amaranth/issues/1048)

# Arbitrary `Memory` shapes

> **Obsoleted by**
> This RFC is obsoleted by [RFC 45](0045-lib-memory.md).

## Summary
[summary]: #summary

Extend `Memory` to support arbitrary element shapes.

## Motivation
[motivation]: #motivation

`Memory` currently only supports plain unsigned elements, with the width set by the `width` argument.
Extending this to allow arbitrary shapes eliminates the need for manual conversion when used to store signed data and value-castables.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `width` argument to `Memory()` is replaced with `shape`, accepting anything that is `ShapeLike`.
Since a plain bit width is `ShapeLike`, this is a direct superset of existing functionality.

If `shape` is shape-castable, each element passed to the `init` argument is passed through `shape.const()`.

Example:
```python
RGB = StructLayout({"r": 8, "g": 8, "b": 8})

palette = Memory(shape = RGB, depth = 16, init = [
    {"r": 0, "g": 0, "b": 0},
    {"r": 255, "g": 0, "b": 0},
    # ...
])
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

`Memory.__init__()` gets a new `shape` argument, accepting any `ShapeLike`.

The `width` argument to `Memory.__init__()` deprecated and removed in a later Amaranth version. Passing both `width` and `shape` is an error.

The `Memory.shape` attribute is added.

The `Memory.width` attribute is made a read-only wrapper for `Shape.cast(self.shape).width`.

The `Memory.depth` attribute is made read-only.

`ReadPort.data` and `WritePort.data` are updated to be `Signal(memory.shape)`.

`WritePort.__init__()` raises an exception if `granularity` is specified and `shape` is not an unsigned `Shape`.

`DummyPort.__init__()` gets a new `data_shape` argument. `data_width` is deprecated and removed in a later Amaranth version.

## Drawbacks
[drawbacks]: #drawbacks

Churn.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- This could also be accomplished by adding a wrapper around `Memory`.
  - A wrapper would result in more code to maintain than simply updating `Memory`, since both the memory object itself and the port objects would have to be wrapped.

## Prior art
[prior-art]: #prior-art

Being able to make a `Memory` with an arbitrary element shape is analogous to being able to make an array with an arbitrary element type in any high level programming language.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

- Once `Memory` is extended to support arbitrary shapes, it is natural that higher level constructs building on `Memory` like FIFOs gets the same treatment.

- `granularity` could later be allowed to be used with other kinds of shapes.
  - This is desirable for e.g. `lib.data.ArrayLayout`, but is not currently possible since `Memory` lives in `hdl.mem`, and `hdl` can't depend on `lib`.
