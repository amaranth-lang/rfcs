- Start Date: 2023-10-30
- RFC PR: [amaranth-lang/rfcs#0028](https://github.com/amaranth-lang/rfcs/pull/0028)
- Amaranth Issue: [amaranth-lang/amaranth#0929](https://github.com/amaranth-lang/amaranth/issues/0929)

# Allow overriding Value operators

## Summary
[summary]: #summary

Allow overriding binary `Value` operators with reflected operators in a value-castable type.

## Motivation
[motivation]: #motivation

A value-castable type can define operators that return another value-castable.
However, if the left side operand is a `Value`, its operator will be called first, casting the right side operand to a plain `Value`.
This creates a mismatch in behavior depending on the type and order of operands.

As an example, consider the multiplication of a fixed point value-castable with an integral type:
```
>>> Q(7).const(0.5) * 255
(fixedpoint Q8.7 (* (const 8'sd64) (const 8'd255)))
>>> 255 * Q(7).const(0.5)
(fixedpoint Q8.7 (* (const 8'sd64) (const 8'd255)))
>>> Q(7).const(0.5) * C(255)
(fixedpoint Q8.7 (* (const 8'sd64) (const 8'd255)))
>>> C(255) * Q(7).const(0.5)
(* (const 8'd255) (const 8'sd64))
```

## Explanation
[guide-level-explanation]: #guide-level-explanation

When a binary `Value` operator is called with a value-castable `other`, check whether the value-castable implements the reflected variant of the operator first and defer to it when present.

## Drawbacks
[drawbacks]: #drawbacks

Extra logic required around every `Value` operator.

## Prior art
[prior-art]: #prior-art

This is standard behavior for inheritance in [Python](https://docs.python.org/3/reference/datamodel.html#emulating-numeric-types):

> Note: If the right operand’s type is a subclass of the left operand’s type and that subclass provides a different implementation of the reflected method for the operation, this method will be called before the left operand’s non-reflected method. This behavior allows subclasses to override their ancestors’ operations.

We don't get this behavior automatically because `Value` is not an ancestor of `ValueCastable`, but it would make sense for it to behave as it were.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

As an alternative, `Value` and `ValueCastable` could be rearchitected so that `ValueCastable` inherits from either `Value` or a common base that implements the `Value` operators.
This would make Python do the right thing w.r.t. operator overriding, but is a larger change with more potential for undesirable consequences.


## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

`Value.eq()` could in the same manner check for and defer to a `.req()` method, i.e. reflected `.eq()`, to allow a value-castable to override how assignment from it to a `Value` is handled.
