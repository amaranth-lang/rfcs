- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Deprecate non-FWFT FIFOs

## Summary
[summary]: #summary

Deprecate non-first-word-fall-through FIFOs.

## Motivation
[motivation]: #motivation

Currently, FIFOs in `amaranth.lib.fifo` have two incompatible interfaces: FWFT (first word fallthrough) and non-FWFT. The incompatibility concerns only the read half. FWFT FIFOs have `r_data` valid when `r_rdy` is asserted. Non-FWFT FIFOs have `r_data` valid only after strobing `r_en`, if the FIFO was empty previously.

Non-FWFT interface is awkward and is essentially never used. It is a holdover from Migen and its implementation details that was included for compatibility. There are three downsides to having it:
1. Having non-FWFT FIFOs requires every consumer of the FIFO interface to check for `fwft` when interacting with the FIFO and either asserting that it is `True`, or adding a code path to handle it. No one does this.
2. The FWFT interface is directly compatible with streams and allows us to add e.g. `r_stream` and `w_stream` to existing FIFOs without adding a wrapper such as `stream.FIFO`. It also makes any custom FIFOs defined downstream of Amaranth stream-enabled.
3. The notion of FWFT vs non-FWFT FIFOs is confusing and difficult to understand. E.g. the author of this RFC wrote both `lib.fifo` and the Glasgow FIFO code, and she misused the `fwft` argument in the latter.

## Guide-level and reference-level explanation
[guide-level-explanation]: #guide-level-explanation

In the next version, instantiating `SyncFIFO(fwft=False)` emits a deprecation warning. In addition, `FIFOInterface`'s `fwft` parameter now defaults to `True`. Other FIFOs have no non-FWFT variant in the first place.

In the version after that, there is no way to instantiate `SyncFIFO(fwft=False)`. The feature and all references to it are removed in their entirety.

## Implementation considerations
[reference-level-explanation]: #reference-level-explanation

At the moment, `SyncFIFOBuffered` is implemented as a register in the output of `SyncFIFO(fwft=False)`. The implementation will need to be rewritten.

## Drawbacks
[drawbacks]: #drawbacks

- Churn.
- There will be no alternative to `SyncFIFO(fwft=False)`.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- It is feasible to extract `SyncFIFO(fwft=False)` into its own module that may be used by downstream code that needs non-FWFT FIFOs. It would not implement `FIFOInterface`.
  - There is no reason the `SyncFIFO` class could not be copied into downstream code as it is.
- It is possible to wrap FIFOs in the stream library in a way that ensures only FWFT FIFOs are used.
  - Let's not create pointless wrappers.

## Prior art
[prior-art]: #prior-art

Not relevant.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

This RFC primarily exists to enable better stream interface integration.
