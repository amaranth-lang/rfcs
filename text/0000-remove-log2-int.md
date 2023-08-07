- Start Date: 2023-08-07
- RFC PR: [amaranth-lang/rfcs#17](https://github.com/amaranth-lang/rfcs/pull/17)
- Amaranth Issue: None

# Remove `log2_int`

## Summary
[summary]: #summary

Remove the `log2_int(n, need_pow2=True)` function.

## Motivation
[motivation]: #motivation

`log2_int` is a helper function that was copied from Migen in the early days of Amaranth.

It behaves like so:

* `n` must be an integer, and a power-of-2 unless `need_pow2` is `False`;
* if `n == 0`, it returns `0`;
* if `n != 0`, it returns `(n - 1).bit_length()`.

Its differences with the `math.log2` function are summarized in the following table:

|   `n` |  `math.log2(n)`   |  `log2_int(n)`    | `log2_int(n, need_pow2=False)` |
| ----- |  ---------------- |  ---------------- | ------------------------------ |
|   `4` |  `2.0`            |  `2`              | `2`                            |
|   `3` |  `1.58496250...`  |  `ValueError`     | `2`                            |
| `0.5` |  `-1.0`           |  `AttributeError` | `AttributeError`               |
|   `0` |  `ValueError`     |  `0`              | `0`                            |
|  `-1` |  `ValueError`     |  `ValueError`     | `2`                            |


In practice, developers seem to use `log2_int` as a variant of `math.log2` whose result is rounded-up to an integer, in order to know how many bits are needed to represent a number.

However, the fact that it transparently accepts `0` and rejects non-integer values such as `0.5` makes it inadequate for use cases outside bit manipulation.

## Explanation
[explanation]: #explanation

The `log2_int` function is deprecated and removed in favor of `math.log2`.

The following suggestions are made to help migration:

* `int(log2(n))` can be used instead of `log2_int(n)`;
* `ceil(log2(n))` can be used instead of `log2_int(n, need_pow2=False)`;
* if `n == 0` or `n < 0` are valid, the caller must handle them explicitly.

## Drawbacks
[drawbacks]: #drawbacks

This is a breaking change.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could keep `log2_int`. The author believes that the suggested replacements are more effective at communicating intent.

## Prior art
[prior-art]: #prior-art

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
