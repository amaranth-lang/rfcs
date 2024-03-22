- Start Date: 2024-04-08
- RFC PR: [amaranth-lang/rfcs#63](https://github.com/amaranth-lang/rfcs/pull/63)
- Amaranth Issue: [amaranth-lang/amaranth#1292](https://github.com/amaranth-lang/amaranth/issues/1292)

# Remove `amaranth.lib.coding`

## Summary
[summary]: #summary

Remove `amaranth.lib.coding` and all classes in it.

## Motivation
[motivation]: #motivation

This module has been essentially inherited from Migen and doesn't meet the bar for inclusion in Amaranth standard library. Most of the functionality included is so simple that it's essentially easier to just inline an implementation. The rest of it would be better served by a function than a module.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The module `amaranth.lib.coding` and all classes in it are removed. To continue using it, copy the contents of the module into your own project.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

All classes within `amaranth.lib.coding` are deprecated in Amaranth 0.5 and removed (along with the module) in Amaranth 0.6. The Gray encoder is moved to `lib.fifo` as a private implementation detail.

## Drawbacks
[drawbacks]: #drawbacks

- Churn.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- This module is out of place in the standard library.
- It has not seen much use and is trivially implemented outside of it.
- Downstream consumers tend to inline the logic anyway.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

The functionality could be brought back at a future point with a more suitable lightweight interface (as functions instead of modules), when the core language is flexible enough to support that.
