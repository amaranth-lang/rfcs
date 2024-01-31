- Start Date: 2024-02-12
- RFC PR: [amaranth-lang/rfcs#43](https://github.com/amaranth-lang/rfcs/pull/43)
- Amaranth Issue: [amaranth-lang/amaranth#1110](https://github.com/amaranth-lang/amaranth/issues/1110)

# Rename `reset=` to `init=`

## Summary
[summary]: #summary

Rename the `reset=` keyword argument to `init=` in `Signal()`, `In()`, `Out()`, `Memory()`, etc.

## Motivation
[motivation]: #motivation

The value specified by the `reset=` keyword argument is called an "initial value" in our language guide, never a "reset value", because when the signal is driven combinatorially, it does not get *reset* to that value (as it holds no state), but rather *initialized* to that value whenever the value of the signal is computed.

Calling it a "reset value" (even implicitly, by the name of the keyword argument) makes teaching Amaranth more difficult and is a point of confusion. All of our documentation already has to carefully avoid calling it a "reset value", and similarly, any Amaranth experts would have to avoid that in speech. Tutorial authors [have to call it out explicitly](https://github.com/RobertBaruch/amaranth-tutorial/blob/6a7ebe9cb3b904177df876df01552099e89c031f/3_modules.md#resetdefault-values-for-signals).

`Memory` already does not have a `reset=` argument or accessor; it uses `init=`. `Memory` should be consistent with `Signal`.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

All instances of `reset=` keyword argument in Amaranth are changed to use `init=`. `reset_less=`, `async_reset=`, etc remain as they are. Using `reset=` raises a deprecation warning but continues working for a long time, perhaps Amaranth 1.0.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The following entry points have their `reset=` argument and attribute changed to `init=`:
- `Signal(reset=)`
- `Signal.like(reset=)`
- `with m.FSM(reset=):`
- `FFSynchronizer(reset=)`
- `Member(reset=)` (which handles `In(reset=)`, `Out(reset=)`)

Specifically:
- At most one of `init=` and `reset=` keyword arguments are accepted. Using `reset=` prints a deprecation warning. The semantics is exactly the same.
- Wherever there was an accessible `.reset` attribute, a getter and a setter are provided that read/write `.init`.
- No specific deprecation timeline is established, unlike with many other features. We could do this, perhaps, in two years, or by Amaranth 1.0.

## Drawbacks
[drawbacks]: #drawbacks

Churn.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The primary alternative is to not do this. Amaranth is steadily gaining popularity, so the earlier we do it the better.

There are no good alternatives to the `init=` name, especially given our already written documentation and its use for `Memory`.

## Prior art
[prior-art]: #prior-art

Verilog has `initial x = 1;`, though that does not result in a reset being inferred.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

When exactly do we remove `reset=`? It seems valuable to do it as late as possible to minimize breakage of lightly maintained Amaranth code.

## Future possibilities
[future-possibilities]: #future-possibilities

None.