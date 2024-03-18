- Start Date: 2024-03-18
- RFC PR: [amaranth-lang/rfcs#54](https://github.com/amaranth-lang/rfcs/pull/54)
- Amaranth Issue: [amaranth-lang/amaranth#1212](https://github.com/amaranth-lang/amaranth/issues/1212)

# Initial and reset values on memory read ports

## Summary
[summary]: #summary

Synchronous memory read ports get initial values, just like `Signal`s.

## Motivation
[motivation]: #motivation

Currently, the initial state of synchronous memory read port's data output is undefined in synthesis, making it one of the very few places in Amaranth where an undefined value can be obtained, and an unexpected one at that. This also cannot be caught with pysim, as it initializes all read ports to 0. This is easily fixable on almost all FPGA targets, as almost all FPGAs have well-defined initial values.

Further, the read ports on almost all FPGA targets also support a reset signal, which is currently not exposed in any way in Amaranth.

The hardware capabilities for yosys-supported targets are as follows:

- Xilinx BRAM: arbitrary initial and reset values, reset can be sync or async (except for very old FPGAs that have sync reset only)
- Lattice, Anlogic, Gowin BRAM; Xilinx URAM, Nexus LRAM: always-zero initial and reset value, reset can be sync or async
- Efinix, iCE40, Gatemate BRAM; iCE40 SPRAM: undefined initial value, no reset
- LUT RAM on any target: full support (uses a normal FF to create a sync read port)

Additionally, on any platform where requested initial value or reset is not natively supported by hardware, yosys will insert a FF and a mux to make it work regardless.

This RFC thus proposes to:

- close the expressiveness hole, making use of the hardware features where supported, using emulation otherwise
- get rid of the undefined behavior


## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Synchronous memory read ports behave in most respects like `Signal`s driven from a synchronous clock domain. As such, they have an initial value that can be set via `init=` on the constructor:

```py
mem = Memory(shape=unsigned(8), depth=8, init=[])
rp = mem.read_port(domain="sync", init=13)
```

The read port's `data` signal will hold the initial value at startup and whenever a domain reset occurs. Additionally, as for `Signal`, `reset_less=True` can be specified to make the port not react to the domain reset.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

`lib.memory.Memory.read_port` and `lib.memory.ReadPort` get two new keyword-only arguments: `init=None` and `reset_less=False`. For synchronous read ports, they effectively have the same behavior as they have on `Signal` when applied to `port.data`. They become introspectable attributes on the port. If the port has `comb` domain, they cannot be changed from their default values and are meaningless.

`hdl.MemoryInstance.read_port` likewise gets two new keyword-only arguments: `init=0` and `reset_less=False`. `init` must be an integer.

## Drawbacks
[drawbacks]: #drawbacks

This is not natively supported by *all* FPGAs. While it can be reasonably cheaply emulated, in the author's experience, any amount of emulation circuitry inserted automatically by the toolchain to ensure well-defined behavior results solely in complaints.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An alternative is to put `init` and `reset_less` on the signature and on the `port.data` signal instead of on the read port. However, `reset_less` is currently not supported by `lib.wiring`, and `is_compliant` will reject any signal with `reset_less=True`. This could be changed by a simple amendment to RFC 2.

## Prior art
[prior-art]: #prior-art

None, or rather obvious enough.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

A way to explicitly request undefined initial value could be added in the future, once undefined values are a well-defined concept in Amaranth.
