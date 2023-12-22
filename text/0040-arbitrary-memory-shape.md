- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#40](https://github.com/amaranth-lang/rfcs/pull/40)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Arbitrary `Memory` shapes

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

The `Memory.width` attribute is deprecated and removed in a later Amaranth version.

`ReadPort.data` and `WritePort.data` are updated to be `Signal(memory.shape)`.

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

- `Memory.width` is still meaningful, but is it useful enough that we should consider keeping it?

## Future possibilities
[future-possibilities]: #future-possibilities

Once `Memory` is extended to support arbitrary shapes, it is natural that higher level constructs building on `Memory` like FIFOs gets the same treatment.  
