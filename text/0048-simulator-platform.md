- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#48](https://github.com/amaranth-lang/rfcs/pull/48)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Allow passing a platform to the simulator

## Summary
[summary]: #summary

Add a `platform` argument to the simulator, allowing a platform to be passed through to the elaboration of the DUT.

## Motivation
[motivation]: #motivation

Having the ability to pass in a platform allows creating a mock platform with simulated resources, making it possible to simulate components using `platform.request()` or other platform methods/properties without change.

Note that the mock platform itself is out of scope for this RFC.

## Guide- and reference-level explanation
[guide-level-explanation]: #guide-level-explanation

`Simulator.__init__()` gets a `platform=None` keyword-only argument.
This value is passed to the elaboration of the DUT.

## Drawbacks
[drawbacks]: #drawbacks

None.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This is a simple and obvious two line change; the simulator is already passing `platform=None` to the elaboration of the DUT and all we need is to expose that argument one layer higher.

## Prior art
[prior-art]: #prior-art

Not relevant.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

In the future it would make sense to also provide a standard mock platform with simulated IO registers.
