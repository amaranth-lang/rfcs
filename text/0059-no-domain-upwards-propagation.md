- Start Date: 2024-03-25
- RFC PR: [amaranth-lang/rfcs#59](https://github.com/amaranth-lang/rfcs/pull/59)
- Amaranth Issue: [amaranth-lang/amaranth#1242](https://github.com/amaranth-lang/amaranth/issues/1242)

# Get rid of upwards propagation of clock domains

## Summary
[summary]: #summary

Upwards propagation of clock domains is removed, `local=True` effectively becomes the default.

## Motivation
[motivation]: #motivation

Currently, a clock domain created anywhere in the hierarchy is, by default, propagated everywhere within that hierarchy. The other option is marking a clock domain with `local=True`, which restricts propagation to only downwards from the point of definition.

The default behavior constitutes the ultimate implicit spooky action at a distance, making it hard to reason about clock domain origins in complex Amaranth design. It is also rarely used. This dangerous default should be changed to something more reasonable.

## Guide and reference-level explanation
[explanation]: #explanation

In version 0.5, upwards domain propagation becomes deprecated. Any use of a domain that wouldn't be valid without upwards propagation (ie. wouldn't be valid if the domain were `local=True`) triggers a deprecation warning.

In version 0.6, upwards domain propagation is removed entirely. The `local` argument to `ClockDomain` constructor becomes redundant and is deprecated.

In version 0.7, the `local` argument is removed.

## Drawbacks
[drawbacks]: #drawbacks

A little bit of churn for designs with clock generator modules.

When propagating a domain upwards is actually desired (eg. for clock generator modules), a workaround is now necessary to route the domain to the toplevel. The simplest way is to set an eg. `cd_sync` attribute on the module in the constructor, then do `m.domains.sync = clkgen.cd_sync` in the top-level. This has the slight disadvantage that the clock generator module cannot decide on the attributes of the clock domain (clock polarity, reset asyncness) based on the platform.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The core proposal is straightforward. The main question is how to handle the usecases currently served by purposeful use of upwards domain propagation. There are three alternatives:

1. None. Clock generator modules must export their domains through some other existing Amaranth functionality. This is the option proposed above.
2. Keep the current mechanism as-is, but make it opt-in, such by requiring `ClockDomain(global=True)`.
3. Add a mechanism to explicitly import a domain from a submodule. A draft is proposed here, for discussion purposes:

   ```py
   m.submodules.clkgen = ClockGenerator(...)
   m.domains.sync = ImportedDomain("clkgen", "sync") # Imports a domain defined in the submodule "clkgen" named "sync".
   ```

We feel that 1. is the best option to take, as introducing a new mechanism when a full redesign of the system is pending is likely to just result in another thing that will have to be deprecated later.

## Prior art
[prior-art]: #prior-art

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

The concept of "private" clock domains has been proposed: clock domains that don't propagate at all, including downwards.

It's clear that clock domain overhaul with a larger scope needs to be done. This RFC is (among other things) intended to enable such overhaul, by removing a feature likely to make it infeasible.
