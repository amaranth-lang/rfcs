- Start Date: 2025-06-02
- RFC PR: [amaranth-lang/rfcs#79](https://github.com/amaranth-lang/rfcs/pull/79)
- Amaranth Issue: [amaranth-lang/amaranth#1612](https://github.com/amaranth-lang/amaranth/issues/1612)

# Add `PortLike.with_direction`

## Summary
[summary]: #summary

Add a `with_direction` method to `lib.io.PortLike`, allowing for "downcasting" a bidirectional port to an input port or an output port.

## Motivation
[motivation]: #motivation

[RFC 55](https://amaranth-lang.org/rfcs/0055-lib-io.html) has made way for a variety of generic and reusable I/O components.  Many such components can be parametrized by the "signature" of I/O ports used, including their directionality.  One such motivating example is Glasgow's `IOStreamer`.

It is often convenient to treat a bidirectional port as an input-only port, or an output-only port.  `lib.io` already allows such usage by instantiating a `*Buffer` with a requested direction.  However, for generic components, it's oftern convenient to determine requested buffer direction by introspecting the provided port — in such case, we need some means to explicitly "downcast" the port before providing it to the component.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Given a `PortLike` with a direction of `Direction.Bidir`, you can use the `with_direction` method to obtain an input-only or output-only version of it:

```
port_io = platform.request("io0", dir="-")
assert port_io.direction == Direction.Bidir
port_i = port_io.with_direction("i")
assert port_i.direction == Direction.Input
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A new method is added to `PortLike` and its implementations:

- `with_direction(self, direction: Direction) -> PortLike`: returns a new `PortLike` equivalent to this one, but with a limitted direction
  - if the `direction` provided is the same as the port's direction, the returned `PortLike` is fully equivalent to `self`
  - if the `direction` is `Direction.Input` or `Direction.Output` and `self.direction is Direction.Bidir`, the returned `PortLike` has the given direction
  - any other combination of `direction` and `self.direction` is an error

## Drawbacks
[drawbacks]: #drawbacks

This adds a new required method to implement on a `PortLike`, and is thus a breaking change for users implementing their own `PortLike`s.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The only obvious alternative to this RFC is providing the information to the generic component out-of-band.  However, this is inconvenient, particularly when groups of ports are involved.

We can have the usual bikeshedding about the name.

## Prior art
[prior-art]: #prior-art

Most languages allow using an `inout` as an `input` or `output`.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
