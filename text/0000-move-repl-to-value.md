- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Move `Repl` to `Value.replicate`

## Summary
[summary]: #summary

Replace the standalone `Repl(value, count)` node with `value.replicate(count)`.

## Motivation
[motivation]: #motivation

`Repl` is a [rarely used](https://github.com/search?q=%2F%5CbRepl%5Cb%2F+amaranth+language%3Apython&type=code) construct (it's mostly useful for manual sign extension).

It is currently a first-class entity that has its own AST node and a name in the prelude, mostly for historical reasons (`Repl(v, n)` is analogous to Verilog's `{x{n}}`).

`Repl` does not need to be a first-class entity; `Repl(x, n)` is almost exactly equivalent to `Cat(x for x in range(n))`. It especially does not need a name in the prelude.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Use of `Repl` is deprecated. To replicate a value multiple times, use `value.replicate()`.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Direct use of `Repl` is deprecated. Its implementation is replaced with `def Repl(value, count): return Value.cast(value).replicate(count)`.

A function `Value.replicate(count)` is added. It is implemented as `Cat(value for _ in range(count))`. The `Repl` AST node is removed.

## Drawbacks
[drawbacks]: #drawbacks

* Churn.
* The proposed implementation makes `Value.replicate` valid on left-hand side of assignment, with potentially surprising behavior. However, this can be handled by prohibiting multiple assignment to the same bit of a signal in general.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Rationale:

* Fewer names in the prelude is always good.
* Unlike with `Cat` (where `Cat()` makes sense), `Repl` does not make sense as a standalone node any more than `Part` does (and we do not currently export `Part`).
* Despite existing by analogy with `{x{n}}`, it is currently turned into a concatenation before it reaches the Verilog backend *anyway*, and any future work will have to reconstruct replication from concatenation in any case.
* `Repl` being a dedicated node complicates AST processing for no reason.

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

* The Verilog backend currently bitblasts what could be a replication. We could detect these and convert them to replications proper.
* We could detect code like `Cat(x, x).eq(0b11)` and warn or reject it.
