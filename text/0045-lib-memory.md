- Start Date: 2024-02-05
- RFC PR: [amaranth-lang/rfcs#45](https://github.com/amaranth-lang/rfcs/pull/45)
- Amaranth Issue: [amaranth-lang/amaranth#1083](https://github.com/amaranth-lang/amaranth/issues/1083)

# Move `hdl.Memory` to `lib.Memory`

> **Obsoletes**
> This RFC is functionally a superset of, and obsoletes, [RFC 40](0040-arbitrary-memory-shape.md).

## Summary
[summary]: #summary

Move `amaranth.hdl.Memory` to `amaranth.lib.Memory`, enabling it to interoperate with the rest of standard library; while we're at it, fix transparency handling.

## Motivation
[motivation]: #motivation

In most ways, a `Memory` is a normal elaboratable: it has ports, it lowers to a kind of fragment, it can be added as a submodule[^1], it has behavior that can be completely expressed in terms of an array of `Signal`s (which is also how its simulation is internally implemented). However, because it exists in `amaranth.hdl` and not `amaranth.lib`, its ports cannot be interface objects, it cannot have a signature and be a component, its lowering cannot be customized, and its behavior cannot include recognizing `lib.data` shapes. Each of these facts has concrete repercussions:
* Amaranth SoC would like to have a `Memory` as a resource in `MemoryMap`, but this is not possible as it's not a component.
* `DummyPort` should morally be a `PureInterface` object created from a `PortSignature`, but is its own thing. (It predates [RFC 2](0002-interfaces.html), md would not be able to use it even if it didn't.)
* One has to trust that (a) Yosys memory handling core (between RTLIL and Verilog) is correct and (b) the backend toolchain memory technology mapping code (from Verilog) is correct. This is not a given and both open source and proprietary toolchain components have had bugs related to memory mapping, working around which at the time requires very invasive changes rather than using a custom lowering or a replacement component. Vivado in particular has longstanding issues around LUTRAM read port inference.
* ASIC flows often support SRAM only in the form of pre-compiled black boxes, without any support for inference. Similarly, handling this case currently requires invasive changes to the code, and is especially difficult in case of standard library FIFOs that are included in third party code, where there is no option besides patching or forking that third party code.
    * For a long time, SPRAM on iCE40 was only supported in the same way when Yosys was used for synthesis, and the workaround for that was universally considered burdensome.[^2]
* Memories with large amounts of write ports are useful in out-of-order CPUs, yet aren't supported by any of the toolchains Amaranth can work with. [Techniques exist](https://tomverbeure.github.io/2019/08/03/Multiport-Memories.html) to translate memories with any amounts of read and write ports to ones that can be technology mapped by most toolchains, but at the moment it is burdensome to switch between the simulation primitive (which doesn't limit the amount of write ports) and the synthesis lowering.
* [RFC 40](0040-arbitrary-memory-shape.md) had to back out a part of the proposal that was accepted during the vote (special treatment for `amaranth.lib.data.ArrayLayout`) after it was discovered that it would require a forbidden dependency.

[^1]: Due to historical reasons related to the memory representation in RTLIL, Amaranth had instructed designers to add memory ports as submodules, but at the time of writing adding either the `Memory` itself or its ports works.

[^2]: SPRAM inference still does not work for Amaranth code, due to Amaranth not supporting uninitialized memories. That will be addressed by another RFC.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

`amaranth.lib.memory.Memory` is a subclass of `amaranth.lib.wiring.Component`.

`amaranth.lib.memory.ReadPort` and `.WritePort` are interface objects; their signature classes are `amaranth.lib.memory.ReadPort.Signature` and `.WritePort.Signature` respectively, following Amaranth SoC conventions. The read and write port `domain` is a parameter of the interface object. The read port `transparent_for` is a parameter of the interface object, and contains a list of `WritePort` objects. The write port `granularity` is a parameter of the signature.

`amaranth.hdl.Memory`, `.ReadPort`, `.WritePort`, and `.DummyPort` are deprecated and removed. During the deprecation period they lower to `amaranth.lib.memory` primitives.

 The `transparent=` parameter of `Memory.read_port()` is deprecated and removed. During the deprecation period, `transparent=True` preserves existing behavior.

A new `amaranth.hdl.MemoryInstance` primitive is added that is lowered by the backend to an appropriate two-dimensional array construct. The interface of this primitive is public but can be changed without notice (although reasonable care will be taken to not break code that relies on it). This primitive exists to fulfill the contract of `amaranth.lib` (which cannot depend on private Amaranth interfaces) and should not be used whenever it is possible to use `amaranth.lib.Memory` instead.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A new module `amaranth.lib.memory` is added.

### Memory ports
[memory-ports]: #memory-ports

Two new signatures are added:
- `amaranth.lib.memory.ReadPort.Signature(addr_width, shape)`, with members:
    - `addr: In(addr_width)`
    - `data: Out(shape)`
    - `en: In(1, reset=1)`
- `amaranth.lib.memory.WritePort.Signature(addr_width, shape, granularity=data_width)`, with members:
    - `addr: In(addr_width)`
    - `data: In(shape)`
    - `en: In(en_width, reset=~0)`

The constructor `WritePort.Signature` ensures that at least one of the following requirements holds:
- `granularity is None`, in which case `en_width = 1`, or
- `shape == unsigned(data_width)`, in which case `en_width` is chosen such that `granularity * en_width == data_width`, or
- `shape == amaranth.lib.data.ArrayLayout(_, elem_count)`, in which case `en_width` is chosen such that `granularity * en_width == elem_count`.

These signatures create two new interface objects:
- `amaranth.lib.memory.ReadPort(signature, *, memory, domain, transparent_for)`, where `transparent_for` is an iterable of write ports (which is converted to `tuple` and stored).
- `amaranth.lib.memory.WritePort(signature, *, memory, domain)`

These signatures and interface objects are fully introspectable: all of the construction parameters are available as read-only attributes. The interface objects may be created with `memory=None` to make a stub that can be used in simulation in place of an actual read port.

### Memory component
[memory-component]: #memory-component

A new component is added:
- `amaranth.lib.memory.Memory(*, shape, depth, init, attrs=None)`, with methods:
    - `.read_port(*, domain="sync", transparent_for=())`, which creates a `ReadPort` instance with a signature whose `addr_width=ceil_log2(self.depth)`. If `domain == "comb"`, the `.en` of the returned port is tied off to a constant 1, so as to avoid creating a latch.
    - `.write_port(*, domain="sync", granularity=None)`, which creates a `WritePort` instance with a signature whose `addr_width=ceil_log2(self.depth)`. `domain` cannot be `"comb"`, so as to avoid creating an array of latches.

This component is introspectable: the construction parameters `shape` and `depth` are available as read-only attributes; `attrs` is exposed in the usual way, and the ports added by the `read_port` and `write_port` methods are available via the `.r_ports` and `.w_ports` read-only attributes.

The `init` parameter is mandatory; if it is acceptable to initialize the memory with the default value for `shape` (typically zero) then `init=[]` should be used. The provided sequence is used to fill in the first `len(init)` elements of the initializer, and the default value for `shape` is used for the rest. When support for indeterminate values is added to Amaranth at a later point, the `init` argument will become optional, and not providing `init` or providing `init=None` will leave the memory filled with indeterminate values.[^3]

[^3]: Currently, Amaranth requires every memory to be initialized. This usually works on the FPGA (not in case of iCE40 SPRAM mentioned above), but only at first boot. It does not work (results in a memory filled with indeterminate values, if it has a write port) after an FPGA design is reset, and it does not work at all on ASIC SRAMs. So this is really a special case that became the general one through historical accident. It is highly desirable to lift this restriction but it cannot be done until Amaranth has a concept of an indeterminate value, and its simulator (as well as, likely, CXXRTL) can compute propagation of such values.

The value of the `init` parameter, cast to a `Memory.Init(*, shape, depth)` (name TBD) container is available as the `init` read-write attribute. This container implements the `MutableSequence` protocol. The length of this container is always `depth`. The values that are retrieved by `__getitem__` are the same as those stored by `__setitem__` or via the constructor parameter, or `None` if they were not stored. When a value that is not castable to `shape` is stored, the appropriate exception is propagated. Assigning to out of bounds indices raises `IndexError`.

The existing `amaranth.hdl.Memory.__getitem__` interface is provided on `amaranth.lib.memory.Memory` with exactly the same semantics as before, which is not defined in this RFC. It is expected that an upcoming RFC addressing async function interface for the simulator will deprecate it.

The signature of `amaranth.lib.memory.Memory` is always empty because the ports are added using an imperative API. If desired to use with `verilog.convert`, ports may be manually enumerated, or a wrapper component may be used that has a declarative API.

When `amaranth.lib.memory.Memory` is elaborated, and the platform has a `get_memory` function, the `Memory.elaborate` function returns `platform.get_memory(self)`, as is customary for standard library components.

Inheriting from `amaranth.lib.memory.Memory` is allowed, with the restriction that only `elaborate` can be overridden. Combinations of ports that are illegal for a given target can be rejected during elaboration, but not earlier.

### Memory instance

A new class is added as a backend for `amaranth.lib.memory.Memory`:

- `amaranth.hdl.MemoryInstance(*, width, depth, init=None, attrs=None, src_loc=None)`, with methods:
  - `.read_port(*, domain, addr, data, en, transparent_for)`, which adds a read port
    - `addr` and `en` must be value-like
    - `data` must be an lvalue value-like (with the same validity rules as `Instance` output ports)
    - `transparent_for` must be a list of write port indices that this read port should be transparent with
  - `.write_port(*, domain, addr, data, en)`, which adds a write port and returns a port index, which is an opaque integer
    - `addr`, `data`, `en` must be value-like

A `MemoryInstance` is essentially a special low-level HDL construct that will be lowered to native memory representation in the target language. It serves as the default backend for `Memory` if the platform doesn't supply its own lowering. Due to its nature as a low-level primitive, it provides no introspection support. It can be used as a submodule in the same way as `Elaboratable` and `Instance`.

### Notes

The usual `src_loc_at=` parameters are added where appropriate.

## Drawbacks
[drawbacks]: #drawbacks

- Churn.
- An internal interface `amaranth.hdl.MemoryInstance` becomes perma-unstable public, increasing support burden and potential fragility.
- Having the signature of `amaranth.lib.memory.Memory` be always empty is pretty weird.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could simply not do this and continue to suffer the consequences.

Either more (see "Future possibilities" below) or fewer (see the changes to transparency) features could be integrated into this particular proposal.

The signature of `amaranth.lib.memory.Memory` could contain all of its ports members. This would violate the current (as of 2024-01) explicitly stated contract of `wiring.Component` given the builder-style interface. The contract could perhaps be relaxed to say that `.signature` should only be idempotent, rather than fixed after construction.
- This isn't possible to solve by making the `w_ports` member have dimensionality because write ports are non-homogeneous (they can have different granularity).
    - If `ratio=` is added (see "Future possibilities" below) then this will also apply to `r_ports` too.
- A solution not involving dimensionality would be to have a `__getattr__` handler matching on `r"[rw]_port_\d+"` and forwarding to `r_ports` or `w_ports`.

A much better solution to the same problem is to have a wrapper `amaranth.lib.memory.DeclarativeMemory(*, shape, depth, read_ports=1, write_ports=1, ...)` (naming TBD) that has a non-empty signature instantiated in the constructor (without any violation of the `wiring.Component` contract). This wrapper will also be much more valuable in all the likely use cases of `verilog.convert(Memory(...))`, as the declarative port count based interface is compatible with including it in a Verilog design (via `connect_rpc`, generated by Amaranth CLI, integrated via FuseSoC, etc), while the imperative builder based interface is not.
- The declarative interface could allow configuring granularity only for all write ports at once, solving the issue with non-homogeneous ports.
    - There is probably no way to provide `ratio=` through such an interface.

Regarding naming:
- Having `XPort` and `XPort.Signature` follows the current Amaranth SoC convention. An alternative would be to have `XPortSignature`, but this doesn't really have any benefits and requires importing more names in the relatively rare case where ports and their signatures are used explicitly.
- `r_ports` and `w_ports` follow the naming scheme of `lib.fifo`.

## Prior art
[prior-art]: #prior-art

This proposal mostly builds on the original `amaranth.hdl.mem` design, which itself builds on the old Yosys memory cell design. This redesign incorporates lessons learned from the Yosys memory cell redesign, and should handle (with features from "Future possibilities" incorporated) nearly any practical memory configuration that can be synthesized to a synchronous SRAM array.

## Resolved questions
[resolved-questions]: #resolved-questions

- Should the signature be empty, or should it contain ports?
    - If it should contain ports: Should `lib.wiring` be amended to support non-homogeneous arrays, or should a workaround be applied for this specific case?
        - A workaround is easy to do in a reasonably backwards-compatible way here, but this might come up more generally in things like AXI interconnect.
    - We decided that the signature should be empty.
- Should the granularity for `ArrayLayout` shapes work the way it's described, or some other way?
    - We decided that this behavior is OK.
- How exactly should this proposal work in simulation? Right now `Memory.__getitem__(index: int)` returns an unspecified assignable value that can only be used in simulation. It's been a frequent source of issues when someone tried to synthesize it.
    - The `amaranth.hdl.MemoryInstance` class could provide a `__getitem__` implementation returning an unspecified non-value-like proxy class that is a valid simulator command. In this case, `Memory` subclasses that lower to something else in simulation would have to provide their own `__getitem__` override matching their lowering.
    - We decided to leave simulation behavior unspecified in this RFC.

## Future possibilities
[future-possibilities]: #future-possibilities

The following work is expected to be done very shortly after this RFC is accepted:
- The error-prone `amaranth.lib.memory.Memory.__getitem__` interface is deprecated and replaced with one that mirrors the interface used in async testbenches for signals.

Yosys and the new IR support several new features, the support for which can be added in a backwards-compatible way:
- Memory ports can be "wide", i.e. have their data width be a multiple of the memory width. This allows defining asymmetric memories, which are especially useful for lane count converting async FIFOs. This feature can be supported by adding a `ratio=` parameter to `Memory.read_port()` and `.write_port()`, which causes the returned signature to have `ratio` fewer address bits and a factor of `1 << ratio` more data bits.
- The output register of a synchronous read port can have its initial value set and a reset input connected. This feature can be supported by adding a `reset_less=True` parameter to `ReadPort.Signature` and a `reset=` (or, after RFC 43, `init=`) parameter to `ReadPort`. If `reset_less=False`, the read port output register is reset by the domain reset signal. (At the moment it is not reset at all.)

The following other options can be explored:
- At the time, memories have an empty signature, to combine the imperative builder based interface with the requirements placed by `wiring.Component`. This means that passing an `amaranth.lib.memory.Memory` to `verilog.convert` produces a useless module with no ports (unless `ports=` is used explicitly). Although niche, this becomes much more useful when memories can have custom lowering.
- Alternatively to the previous item, new component with a (slightly more limited) declarative port count based interface could be added instead, to address a need for integrating Amaranth's custom lowering with Verilog designs.
- At the time there is no way to pass a memory to `VCDWriter(traces=)`. This limitation may be lifted.
- To aid simulation access to memories, `amaranth.hdl.MemoryInstance`'s interface may be extended to convey the identity of any particular memory in presence of fragment transformers.
    - Eventually, fragment rewriting itself should go, at which point the identity of the `MemoryInstance` itself will become usable.
- Support for uninitialized memories will be added in a future RFC.

## Acknowledgements
[acknowledgements]: #acknowledgements

@wanda-phi and @jfng provided valuable feedback during drafting of this proposal.