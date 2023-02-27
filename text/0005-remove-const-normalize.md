- Start Date: 2023-02-07
- RFC PR: [amaranth-lang/rfcs#5](https://github.com/amaranth-lang/rfcs/pull/5)
- Amaranth Issue: [amaranth-lang/amaranth#754](https://github.com/amaranth-lang/amaranth/issues/754)

# Summary
[summary]: #summary

Remove `Const.normalize(value, shape)`.

# Motivation
[motivation]: #motivation

From the name it is not clear what it is supposed to achieve (it's truncation and inversion according to the shape) and it does not check types of arguments.

We already have `Const(value, shape).value` and most developers should be aware of it. Having `Const.normalize(value, shape)` as well provides no benefit over the former. It's also longer.

# Explanation
[explanation]: #explanation

The `Const.normalize` method is deprecated (with the suggestion to use `Const().value`) and removed.

# Drawbacks
[drawbacks]: #drawbacks

- Churn.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could keep it. Removing it reduces the API surface and makes the language a bit more elegant.

# Prior art
[prior-art]: #prior-art

None.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

# Future possibilities
[future-possibilities]: #future-possibilities

None.
