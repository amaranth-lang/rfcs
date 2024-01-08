- Start Date: 2024-01-08
- RFC PR: [amaranth-lang/rfcs#39](https://github.com/amaranth-lang/rfcs/pull/39)
- Amaranth Issue: [amaranth-lang/amaranth#1021](https://github.com/amaranth-lang/amaranth/issues/1021)

# Change semantics of no-argument `m.Case()`

## Summary
[summary]: #summary

Change the semantics of `with m.Case():` (without any arguments) from always-true conditional to always-false conditional. Likewise, change `value.matches()` from returning `C(1)` to returning `C(0)`.

## Motivation
[motivation]: #motivation

Currently, `with m.Case():` results in an always-true conditional, and `value.matches()` likewise returns a const-1 value. However, this is not consistent with what would be expected from extrapolating the non-empty case.

In all non-empty cases, the semantics are equivalent to an OR of equality comparisons with all specified values:

`value.matches(1, 2, 3) =def= (value == 1) | (value == 2) | (value == 3)`

`value.matches(1, 2, 3) =def= Const(0) | (value == 1) | (value == 2) | (value == 3)`

Extrapolating from this, one would expect `value.matches()` to be the empty OR, ie. `Const(0)`.

It is unlikely that any manually written code will rely on this, but this can be a dangerous trap for machine-generated code that doesn't take the empty case into account.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The semantics of `m.Case()` change from always matching to never matching. Likewise, the semantics of `value.matches()` change from always-1 to always-0. The change is committed to the current `main` branch and will be included in Amaranth 0.5.

Amaranth 0.4.1 is released with the old semantics, but a deprecation warning is emitted whenever `m.Case()` or `value.matches()` is used.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

See above.

## Drawbacks
[drawbacks]: #drawbacks

Obviously backwards-incompatible, changes the semantics of a language construct to the direct opposite.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

It is unlikely anyone actually uses `value.matches()` directly, since this is just a constant. For generated code, the current semantics is much more likely to be a bug that intended behavior.

For `m.Case()` the situation is similar: it is redundant with `m.Default()`, which should be used instead. It is somewhat possible that there is code out there written by someone who didn't know about `m.Default()` and ended up using `m.Case()` instead (the official documentation didn't include either for a long time). This code will need to be fixed.

An alternative, if the empty case is deemed too confusing or insufficiently useful, is to make the semantics a hard error instead.

## Prior art
[prior-art]: #prior-art

The current behavior is likely taken directly from RTLIL, which exhibits a similar inconsistency.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

Should we include more warnings about the change? This RFC proposes a warning in the 0.4.1 release, but this will never be seen by someone always using amaranth from git main.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
