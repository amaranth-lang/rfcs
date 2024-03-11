- Start Date: 2024-03-11
- RFC PR: [amaranth-lang/rfcs#53](https://github.com/amaranth-lang/rfcs/pull/53)
- Amaranth Issue: [amaranth-lang/amaranth#1195](https://github.com/amaranth-lang/amaranth/issues/1195)

# Low-level I/O primitives

## Summary
[summary]: #summary

A new first-class concept is introduced to the language, I/O ports, representing top-level ports in synthesis flows. I/O ports can be obtained from the platform, or can be manually constructed by the user in raw Verilog flows that don't use a supported platform.

I/O ports can be connected to `Instance` ports. This becomes the only thing that can be connected to `Instance` ports with `io` directionality.

A new `IOBufferInstance` primitive is introduced that can consume an I/O port without involving vendor-specific cells.

## Motivation
[motivation]: #motivation

The current process for creating top-level ports is rather roundabout and involves the platform calling into multiple private APIs:

1. The user calls `platform.request`, which information about the requested pin, and returns an interface object.
   1. A `Signal` for the raw port is created and stored on the platform, together with metadata.
   2. If raw port is requested (`dir='-'`), that signal is returned (in a wrapper object).
   3. Otherwise, an I/O buffer primitive is instantiated that drives the raw port, and the returned interface contains signals controlling that buffer. Depending on the platform, this could be either a vendor-specific cell (via `Instance`) or a generic tristate buffer (via `IOBufferInstance`, which is currently a private API).
2. The hierarchy is elaborated via `Fragment.get`.
3. Platform performs first half of design preparation via private `Fragment` APIs (domains are resolved, missing domains are created).
4. Platform installs all instantiated I/O buffers into the elaborated hierarchy and gathers all top-level port signals.
5. Platform finishes design preparation via more private APIs, then calls RTLIL or Verilog backend.
6. Platform uses the gathered list of top-level ports to create a constraint file.

If the `io` directionality is involved, the top-level port is a cursed kind of `Signal` that doesn't follow the usual rules:

- it effectively has two drivers (the `Instance` or `IOBufferInstance` and the external world)
- on the Amaranth side, it can only be connected to at most one `Instance` or `IOBufferInstance` (ie. it cannot be peeked at)

This proposal aims to:

- provide a platform-independent way to instantiate and use top-level ports
- significantly reduce the amount of private APIs used in platform code
- provide a stricter model of I/O ports, no longer overloading `Signal`

## Scope and roadmap
[scope-roadmap]: #scope-roadmap

The proposal is only about low-level primitives implemented in `amaranth.hdl`. It is to be followed with:

1. Another RFC proposing generic I/O buffer components with platform hooks in `lib.io`.
2. Overhaul of the platform API (as of now undetermined).


## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### `IOPort`s and their use

When creating a synthesizable top-level design with Amaranth, top-level ports are represented by the `IOPort` class. If you're not using the Amaranth platform machinery, you can instantiate a top-level port like this:

```py
abc = IOPort(8, name="abc") # The port is 8 bits wide.
```

If a platform is in use, `IOPort`s should be requested via `platform.request` instead of created manually — the platform will associate its own metadata with ports.

To actually use such a port from a design, you need to instantiate an I/O buffer:

```py
abc_o = Signal(8)
abc_i = Signal(8)
abc_oe = Signal()
m.submodules += IOBufferInstance(abc, i=abc_i, o=abc_o, oe=abc_oe)
# abc_o and abc_oe can now be written to drive the port, abc_i can be read to determine the state of the port.
```

This automatically creates an `inout` port on the design. You can also create an `output` port by skipping the `i=` argument, or an `input` port by skipping the `o=` and `oe=` arguments.

If the `o=` argument is passed, but `oe=` is not, a default of `oe=Const(1)` is assumed.

Alternatively, `IOPort`s can be connected directly to `Instance` ports:

```py
# Equivalent to the above, using Xilinx IOBUF cells.
for i in range(8):
    m.submodules += Instance("IOBUF",
        i_I=abc_o[i],
        i_T=~abc_oe,
        o_O=abc_i[i],
        io_IO=abc[i],
    )
```

Just like values, `IOPort`s can be sliced with normal Python indexing and concatenated with `Cat`.

`IOPort`s can only be consumed by `IOBufferInstance` and `Instance` ports — they cannot be used as plain values. Every `IOPort` bit can be consumed at most once.

Only `IOPort`s (and their slices or concatenations) can be connected to `Instance` ports with `io` directionality. Ports with `i` and `o` directionalities can be connected to both `IOPort`s and plain `Value`s.

### General note on top-level ports

Amaranth provides many ways of specifying top-level ports, to be used as appropriate for the design:

1. For a top-level synthesizable design using a platform, ports are created by `platform.request` which either returns a raw `IOPort` or immediately wraps it in an I/O buffer.
2. For a top-level synthesizable design without using a platform, `IOPort`s can be created manually as above.
3. For a partial synthesizable design (to be used with eg. a Verilog top level), the top elaboratable can be a `lib.wiring.Component`, and the ports will be automatically derived from its signature + any unresolved domains.
4. For a partial synthesizable design without using `lib.wiring.Component`, the list of signals to be used as top-level ports can be specified out-of-band to the backend via the `ports=` argument.
5. For simulation, top-level ports are not used at all.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### `IOPort` and `IOValue`

Two public classes are added to the language:

- `amaranth.hdl.IOValue`: represents a top-level port, or a slice or concatenation thereof. Analogous to `Value`. No public constructor.
  - `__len__(self)`: returns the width of this value in bits. I/O values have no shape or signedness, only width.
  - `__getitem__(self, index: int | slice)`: like slicing on values, returns an `IOValue` subclass.
  - `metadata`: a read-only attribute returning a tuple of platform-specific objects; the tuple has one element per bit.
  - `cast(obj)` (class method): converts the given object to `IOValue`, or raises an exception; the only non-`IOValue` object that can be passed is a 0-length `Value`, as per below.
- `amaranth.hdl.IOPort(width, *, name, attrs={}, metadata=None)`: represents a top-level port. A subclass of `IOValue`. Analogous to `Signal`.
  - `metadata` is an opaque field on the port that is not used in any way by the HDL core, and can be used by the platform to hold arbitrary data. It is normally used to store associated constraints to be emitted to the constraint file. The value can be either a tuple of arbitrary Python objects with length equal to `width`, or `None`, in which case an all-`None` tuple of the right width will be filled in.

The `Cat` function is changed to work on `IOValue`s in addition to plain `Value`s:

- all arguments to `Cat` must be of the same kind (either all `Value`s or all `IOValue`s)
- the result is the same kind as the arguments
- if no arguments at all are passed, the result is a `Value`

When `IOValue`s are sliced, the `metadata` attribute of the slicing result is likewise sliced in the same way from the source value. The same applies for concatenations.

As a special allowance to avoid problems in generated code, `Cat()` (empty concatenation, which is defined to be a `Value`) is also allowed wherever an `IOValue` is allowed.

### `IOBufferInstance`

A new public class is added to the language:

- `amaranth.hdl.IOBufferInstance(port, *, i=None, o=None, oe=None)`

The `port` argument must be an `IOValue`.

The `i` argument is used for the input half of the buffer. If `None`, the buffer is output-only. Otherwise, it must be an assignable `Value` of the same width as the `port`. Like for `Instance` outputs, the allowed kinds of `Value`s are `*Signal`s and slices or concatenations thereof.

The `o` argument is used for the output half of the buffer. If `None`, the buffer is input-only. Otherwise, it must be a `Value` of the same width as the `port`.

The `oe` argument is the output enable. If `o` is `None`, `oe` must also be `None`. Otherwise, it must be either a 1-bit `Value` or `None` (which is equivalent to `Const(1)`).

At least one of `i` or `o` must be specified.

The `IOBufferInstance`s are included in the hierarchy as submodules in the same way as `Instance` and `MemoryInstance`.

### `Instance`

The rules for instance ports are changed as follows:

1. `io` ports can only be connected to `IOValue`s.
2. `o` ports can be connected to either `IOValue`s or assignable `Value`s. Like now, acceptable `Value`s are limitted to `*Signal`s and their slices and concatenations.
3. `i` ports can be connected to either `IOValue`s or `Value`s.

A zero-width `IOValue` or `Value` can be connected to any port regardless of direction, and such connection is ignored.

### Elaboration notes

Every `IOPort` used in the design will become a top-level port in the output, with the given name (subject to the usual name deduplication).

If the `IOPort` is used only as an input (connected to an `Instance` port with `i` directionality, or to `IOBufferInstance` without `o=` argument), it becomes a top-level `input` port.

If the `IOPort` is used only as an output (connected to an `Instance` port with `o` directionality, or to `IOBufferInstance` without `i=` argument), it becomes a top-level `output` port.

Otherwise, it becomes an `inout` port.

Every bit of an `IOPort` can be used (ie. connected to `IOBufferInstance` or `Instance`) at most once.

After a design is elaborated, the list of all `IOPort`s used can be obtained by some mechanism out of scope of this RFC.

Calling `verilog.convert` or `rtlil.convert` without `ports` becomes legal. The full logic of determining top-level ports in `convert` is as follows:

- `ports is not None`: the ports include the specified signals, all `IOPort`s in the design, and clock/reset signals of all autocreated domains
- `ports is None` and top-level is `Component`: the ports include the signature members, all `IOPort`s in the design, and clock/reset signals of all autocreated domains
- `ports is None`, top-level is not `Component`: the ports include all `IOPort`s in the design and clock/reset signals of all autocreated domains

Using `IOPort`s together with other ways of creating top-level ports is not recommended.

`IOPort`s are not supported in any way in simulation.

### Platform interfaces

The interface objects currently returned by `platform.request(dir="-")` are changed to have an empty signature, with the `io`, `p`, `n` attributes becoming `IOPort`s.

## Drawbacks
[drawbacks]: #drawbacks

Arguably we have too many ways of creating top-level ports. However, it's not clear if we can remove any of them.

The empty `Cat()` hack is a minor type system wart.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The change is necessary, as the I/O and platform code is a pile of hacks that needs cleanup.

The core `IOPort` + `IOValue` design codifies existing validity rules for `Instance` `io` ports. The `IOBufferInstance` already exists in current Amaranth as a private API used by the platforms.

The change of allowed arguments for `Instance` `io` ports is done without a deprecation period. It is felt that this will have a minimal impact, as the proposed change to `platform.request` with `dir="-"` in RFC 55 will effectively fix the breakage for most well-formed instances of `io` port usage, and such usage is not common in the first place.

## Prior art
[prior-art]: #prior-art

`IOValue` system is patterned after the current `Value` system.

The port auto-creation essentially offloads parts of the current platform functionality into Amaranth core.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

Should we forbid mixing various ways of port creation?

Traditional name bikeshedding.

## Future possibilities
[future-possibilities]: #future-possibilities

Another RFC is planned that will overhaul `lib.io`, adding generic I/O buffer components.
