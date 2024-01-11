- Start Date: 2024-01-08
- RFC PR: [amaranth-lang/rfcs#17](https://github.com/amaranth-lang/rfcs/pull/17)
- Amaranth Issue: [amaranth-lang/amaranth#1025](https://github.com/amaranth-lang/amaranth/issues/1025)

# Remove `log2_int`

## Summary
[summary]: #summary

Replace `log2_int` with two functions: `ceil_log2` and `exact_log2`.

## Motivation
[motivation]: #motivation

`log2_int` is a helper function that was copied from Migen in the early days of Amaranth.

It behaves like so:

* `n` must be an integer, and a power-of-2 unless `need_pow2` is `False`;
* if `n == 0`, it returns `0`;
* if `n != 0`, it returns `(n - 1).bit_length()`.

#### Differences with `math.log2`

In practice, `log2_int` differs from `math.log2` in the following ways:

1. its implementation is restricted to integers only;
2. if `need_pow2` is false, the result is rounded up to the nearest integer;
3. it doesn't raise an exception for `n == 0`;
4. if `need_pow2` is false, it doesn't raise an exception for `n < 0`.

#### Observations

* *1)* is a desirable property; coercing integers into floating-point numbers is fraught with peril, as the latter have limited precision.
* *2)* has common use-cases in digital design, such as address decoders.
* *3)* and *4)* are misleading at best. Despite being advertised as a logarithm, `log2_int` doesn't exclude 0 or negative integers from its domain.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Amaranth provides two log2 functions for integer arithmetic:

* `ceil_log2(n)`, where `n` is assumed to be any non-negative integer
* `exact_log2(n)`, where `n` is assumed to be an integer power-of-2

For example:

```python3
ceil_log2(8) # 3
ceil_log2(5) # 3
ceil_log2(4) # 2

exact_log2(8) # 3
exact_log2(5) # raises a ValueError
exact_log2(4) # 2
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Use of the `log2_int` function is deprecated.

A `ceil_log2(n)` function is added, that:

* returns the integer log2 of the smallest power-of-2 greater than or equal to `n`;
* raises a `TypeError` if `n` is not an integer;
* raises a `ValueError` if `n` is lesser than 0.

An `exact_log2(n)` function is added, that:

* returns the integer log2 of `n`;
* raises a `TypeError` if `n` is not an integer;
* raises a `ValueError` if `n` is not a power-of-two.

## Drawbacks
[drawbacks]: #drawbacks

This is a breaking change.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The following alternatives have been considered:

1. Do nothing. Callers of `log2_int` may still need to restrict its domain to positive integers.
2. Restrict `log2_int` to positive integers. Downstream code relying on the previous behavior may silently break.
3. Remove `log2_int`, and use `math.log2` as replacement:
    * `log2_int(n)` would be replaced with `math.log2(n)`
    * `log2_int(n, need_pow2=False)` would be replaced with `math.ceil(math.log2(n))`

Option *3)* will give incorrect results, as `n` is coerced from `int` to `float`:

```
>>> log2_int((1 << 64) + 1, need_pow2=False)
65
>>> math.ceil(math.log2((1 << 64) + 1))
64
```

## Prior art
[prior-art]: #prior-art

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None.

## Acknowledgements

[@wanda-phi] provided valuable feedback while this RFC was being drafted.

[@wanda-phi]: https://github.com/wanda-phi
