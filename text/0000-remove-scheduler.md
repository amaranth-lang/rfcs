- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#19](https://github.com/amaranth-lang/rfcs/pull/19)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Remove `amaranth.lib.scheduler`

## Summary
[summary]: #summary

Remove `amaranth.lib.scheduler` and the only class `RoundRobin` in it.

## Motivation
[motivation]: #motivation

This module is not used in the sole place for which it was added (Amaranth SoC), it is not especially useful, and it has not undergone proper community review when it was added.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The module `amaranth.lib.scheduler` and the sole class `RoundRobin` in it is removed. To continue using it, copy the contents of the module into your own project.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The class `amaranth.lib.scheduler.RoundRobin` is deprecated in Amaranth 0.4 and removed in Amaranth 0.5.

## Drawbacks
[drawbacks]: #drawbacks

- Churn.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- This module is out of place in the standard library.
- It has not seen much use and is trivially implemented outside of it.
- Downstream consumers tend to inline the logic anyway.
- It does not seem like there would be any other uses for the `amaranth.lib.scheduler` module since any other scheduling algorithm would be more closely tied to the consumer.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
