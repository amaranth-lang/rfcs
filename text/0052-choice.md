- Start Date: 2024-07-01
- RFC PR: [amaranth-lang/rfcs#52](https://github.com/amaranth-lang/rfcs/pull/52)
- Amaranth Issue: [amaranth-lang/amaranth#1445](https://github.com/amaranth-lang/amaranth/issues/1445)

# Add `amaranth.hdl.Choice`, a pattern-based `Value` multiplexer

## Summary
[summary]: #summary

A new type of expression is added: `amaranth.hdl.Choice`. It is essentially a variant of `m.Switch`
that returns a `Value` using the same patterns as `m.Case` for selection.

## Motivation
[motivation]: #motivation

We currently have several multiplexer primitives:

- `Mux`, selecting from two values
- `Array` indexing, selecting from multiple values by a simple index
- `.bit_select` and `.word_select`, selecting from slices of a single value by a simple index
- `m.Switch` together with combinatorial assignment to an intermediate `Signal`, selecting from multiple values by pattern matching

It is, however, not possible to select from multiple values by pattern matching without using an intermediate `Signal` and assignment (which can be a problem in contexts where a `Module` is not available). This RFC aims to close this hole.

This feature is generally useful and has been on the roadmap for a while. The immediate impulse for writing this RFC was using this functionality to implement string formatting for `lib.enum` values.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `Choice` expression can be used to select from among several values via pattern matching:

```py
abc = Signal(8)
a = Signal(8)
b = Signal(8)
sel = Signal(4)
m.d.comb += abc.eq(Choice(sel)
    # any pattern or tuple of patterns usable in `Value.matches` or `m.Case` is valid as key
    .case(1, a)
    .case(2, b)
    .case((3, 4), a + b)
    .case("11--",  a - b)
    .case(("10--", "011-"), a * b)
    .default(13)
)
```

is equivalent to writing:

```py
with m.Switch(sel):
    with m.Case(1):
        m.d.comb += abc.eq(a)
    with m.Case(2):
        m.d.comb += abc.eq(b)
    with m.Case(3, 4):
        m.d.comb += abc.eq(a + b)
    with m.Case("11--"):
        m.d.comb += abc.eq(a - b)
    with m.Case("10--", "011-"):
        m.d.comb += abc.eq(a * b)
    with m.Default():
        m.d.comb += abc.eq(13)
```

`Choice` can also be used on the left-hand side of an assignment:

```py
a = Signal(8)
b = Signal(8)
c = Signal(8)
d = Signal(8)
sel = Signal(2)
m.d.sync += (Choice(sel)
    .case(0, a)
    .case(1, b)
    .case(2, c)
    .default(d)
    .eq(0))
```

which is equivalent to:

```py
with m.Switch(sel):
    with m.Case(0):
        m.d.sync += a.eq(0)
    with m.Case(1):
        m.d.sync += b.eq(0)
    with m.Case(2):
        m.d.sync += c.eq(0)
    with m.Default():
        m.d.sync += d.eq(0)
```

If `default=` is not used, the default value is 0 when on right-hand side, and no assignment happens when on left-hand side.

In addition, `Mux` becomes assignable if the second and third argument are both assignable.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A new expression type is added:

- `amaranth.hdl.Choice(sel: ValueLike)`: creates a new `Choice` expression with no cases
  - `.case(self, patterns: int | str | tuple[int | str], value: ValueLike) -> Choice`: creates a new `Choice` based on this one, adding anoter case to it
  - `.default(self, value: ValueLike) -> Choice`: creates a new `Choice` based on this one, adding a default case to it

The expression evaluates `sel`, then matches it to `patterns` of every `.case()` in turn. If a match is found, the expression evaluates to the corresponding `value` of the first found match. If no match is found, the expression evaluates to the `value` of `.default()`, or to `Cat()` with no arguments if no `.default()` was used. The expression is assignable if all `.case()` values and `.default()` value (if any) are assignable.

Neither `.case()` nor `.default()` can be called on a `Choice` that already has a `.default()`.

The shape of the expression is determined as follows:

- if all `value` arguments are `ShapeCastable`, and it is the same `ShapeCastable` for all of them (as determined by `__eq__` on the `ShapeCastable`), the resulting value is transformed through `ShapeCastable.__call__` of that shape-castable
- if all `value` arguments have a plain `Shape`, the minimum shape that can represent the shapes of all `cases` values and `default` (ie. determined the same as for `Array` proxy or `Mux` tree).
- otherwise, an exception is raised

The default when `.default()` is not specified is `Cat()` to ensure the correct semantics for assignment (ie. discarding the assigned value). This also happens to provide the default 0 when on right-hand side.

`Choice` is also added to the Amaranth prelude.

In addition, the existing `Mux` expression is made valid on the left-hand side of an assignment, as if it was lowered as follows:

```py
def Mux(sel, val1, val0):
    return Choice(sel).case(0, val0).default(val1)
```

`ArrayProxy` (ie. the type currently returned by `Array` indexing) is changed from a native `Value` to a `ValueCastable` that lowers to `Choice` (removing the odd case where we can currently build an invaid `Value`). To avoid problems with lowering the out-of-bounds case, the value returned for out-of-bounds `Array` accesses is changed to 0.

`__eq__` is added to the `ShapeCastable` protocol and documented (we already have suitable implementations in `lib.data` and `lib.enum`).

## Drawbacks
[drawbacks]: #drawbacks

The language gets slightly more complex.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The core functionality is fairly obvious. However, the syntax is not. Other possibilities include:

- `*args` (or perhaps iterable) of `(key, value)` tuples:

  ```py
  Choices(sel,
      (1, a),
      (2, b),
      ((3, 4), c),
      ("11--", d),
      default=e
  )
  ```

- *args of newly-defined `amaranth.hdl.Case` object (not to be confused with `m.Case`):

  ```py
  Choices(sel,
      Case(1, a),
      Case(2, b),
      Case((3, 4), c),
      Case("11--", d),
      default=e,
  )
  ```

The syntax proposed has been selected to have extension space (in the form of keyword arguments) for e.g. optional guard conditions.

## Prior art
[prior-art]: #prior-art

This feature is inspired by Rust `match` construct.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

The name is subject to bikeshed. An obvious alternative is `Match`, though this RFC avoids using this name, as it suggests much more advanced pattern matching (with variable capture) than is currently available.

## Future possibilities
[future-possibilities]: #future-possibilities

Optional guard conditions could be added to `Choice` and `m.Switch` cases (like Rust's `if` guards on `match` branches).
