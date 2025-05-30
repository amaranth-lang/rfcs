- Start Date: 2025-06-02
- RFC PR: [amaranth-lang/rfcs#78](https://github.com/amaranth-lang/rfcs/pull/78)
- Amaranth Issue: [amaranth-lang/amaranth#1639](https://github.com/amaranth-lang/amaranth/issues/1639)

# Internal I/O ports

## Summary
[summary]: #summary

`SimulationPort` is renamed to `InternalPort`, and using it outside of simulation is allowed.  It allows for connecting I/O port-expecting components to internal signals instead of external I/O ports.

## Motivation
[motivation]: #motivation

[RFC 55](https://amaranth-lang.org/rfcs/0055-lib-io.html) has made way for a variety of reusable I/O components, capable of attaching to arbitrary `PortLike`s.  However, their reusability is limitted by requiring external ports for all of their interface.  This limitation comes up in two scenerios:

1. The component is used internally, to interface with other gateware within the same design.
2. One of the optional pins of the I/O interface is unused and should be discarded (for outputs), or tied to a constant value (for inputs).

Having a way to connect such gateware to internal signals resolves both issues.  Since we already have such capability in simulation, we only need to extend it to allow synthesis usage.

## Guide-level and reference-level explanation
[guide-level-explanation]: #guide-level-explanation

`lib.io.SimulationPort` is renamed to `lib.io.InternalPort`.  The old name is deprecated and will be removed in Amaranth 0.7.

Use of `InternalPort` is allowed with `lib.io.Buffer` and `lib.io.FFBuffer` regardless of the platform (and bypassing the platform hooks).

Use of `InternalPort` with `lib.io.DDRBuffer` is forbidden in synthesis. If a need to abstract over unknown implementations of DDR buffer exists for a component, this should be done by using `io.DDRBuffer.signature` in the component interface rather than using `InternalPort` or similar objects.

Use of `InternalPort` with `lib.io.DDRBuffer` is allowed in simulation. The buffer will elaborate to an unspecified glitch-free model driving the port triple.

To simplify tying off unused input ports to constant values, `init=` keyword-only argument is added to the `InternalPort` constructor. Is is passed directly to the `init=` argument of the port's `i` signal.  It is an error to use this argument with an output-only port.

## Drawbacks
[drawbacks]: #drawbacks

- more complex I/O subsystem
- the tie-off case could be better served by other means

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This is a fairly simple solution leveraging existing code.  For the internal connection case, no other design is apparent.  For tie-off, there are two other possibilities:

1. Have a separate `TieoffPort` class.

   Such class could serve as a marker that the port output will always be discarded, or that input will always have the tieoff value, allowing for simplification in controller logic.

   Further, fancy platform-specific buffer types (such as high data rate SERDES) could be trivially instantiated for tie-off ports (by simply doing nothing for discarded output, or replicating the tieoff value for input), which is not in general possible for arbitrary `InternalPort`.

2. Have an out-of-band mechanism that removes the unused port from the component's signature.

## Prior art
[prior-art]: #prior-art

This RFC reuses existing `SimulationPort` machinery.

Passing around a triple of `(I, O, OE)` is a fairly standard way of connecting an I/O controller to other logic.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None.