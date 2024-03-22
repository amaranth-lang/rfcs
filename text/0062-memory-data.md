- Start Date: 2024-03-25
- RFC PR: [amaranth-lang/rfcs#62](https://github.com/amaranth-lang/rfcs/pull/62)
- Amaranth Issue: [amaranth-lang/amaranth#1241](https://github.com/amaranth-lang/amaranth/issues/1241)

# The `MemoryData` class

## Summary
[summary]: #summary

A new class, `amaranth.hdl.MemoryData`, is added to represent the data and identity of a memory. It is used to reference the memory in simulation.

## Motivation
[motivation]: #motivation

It is commonly useful to access a memory in a simulation testbench without having to create a special port for it. This requires storing some kind of a reference to the memory on the elaboratables.

Currently, the object used for this is of a private `MemoryIdentity` class. It is implicitly created by `lib.memory.Memory` constructor, and passed to the private `_MemorySim{Read|Write}` objects when `__getitem__` is called.  This has a few problems:

- the `Memory` needs to be instantiated in the constructor of the containing elaboratable; combined with its mutability and the occasional need to defer memory port creation to `elaborate`, this results in elaboratables that break when elaborated more than once
- `amaranth.sim` currently requires a gross hack to recognize `Memory` objects in `traces`
- occasionally, it is useful to have `Signal`s that are not included in the design proper, but are used to communicate between simulator processes; it could be likewise useful with memories, but it cannot work with the current code (since the `MemoryIdentity` that the simulator gets doesn't have enough information to actually create backing storage for the memory)
- `MemoryIdentity` nor `MemoryInstance` don't contain information about the element shape, which would require further wrappers on `lib.Memory` to perform shape conversion in post-RFC 36 world

The proposed `MemoryData` class:

- replaces `MemoryIdentity` and serves as the reference point for the simulator
- encapsulates the memory's shape, depth, and initial value (so the simulator can use it to create backing storage)
- in the common scenario, is created by the user in elaboratable constructor, stored as an attribute on the elaboratable, then passed to `Memory` constructor in `elaborate`

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

If the memory is to be accessible in simulation, the code to create a memory changes from:

```py
m.submodules.memory = memory = Memory(shape=..., depth=..., init=...)
port = memory.read_port(...)
```

to:

```py
# in __init__
self.mem_data = MemoryData(shape=..., depth=..., init=...)
# in elaborate
m.submodules.memory = memory = Memory(self.mem_data)
port = memory.read_port(...)
```

The `my_component.mem_data` object can then be used in simulation to read and write memory:

```py
addr = 0x1234
row = sim.get(mem_data[addr])
row += 1
sim.set(mem_data[addr], row)
```

The old way of creating memories is still supported, though somewhat less flexible.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Two new classes are added:

- `amaranth.hdl.MemoryData(*, shape: ShapeLike, depth: int, init: Iterable[int | Any], name=None)`: represents a memory's data storage. `name`, if not specified, defaults to the variable name used to store the `MemoryData`, like it does for `Signal`.
  - `__getitem__(self, addr: int) -> MemoryData._Row | ValueCastable`: creates a `MemoryData._Row` object; if `self.shape` is a `ShapeCastable`, the `MemoryData._Row` object constructed is immediately wrapped via `ShapeCastable.__call__`
- `amaranth.hdl.MemoryData._Row` (subclass of `Value`): represents a single row of `MemoryData`, has no public constructor nor operations (other than ones derived from `Value`), can only be used in simulator processes and testbenches

The `MemoryData` class allows access to its constructor arguments via read-only properties.

The `lib.memory.Memory.Init` class is moved to `amaranth.hdl.MemoryData.Init`. It is used for the `init` property of `MemoryData`.

The `Memory` constructor is changed to:

- `amaranth.lib.memory.Memory(data: MemoryData = None, *, shape=None, depth=None, init=None, name=None)`

  - either `data`, or all three of `shape`, `depth`, `init` need to be provided, but not both
  - if `data` is provided, it is used directly, and stored
  - if `shape`, `depth`, `init` (and possibly `name`) are provided, they are used to create a `MemoryData`, which is then stored

The `MemoryData` object is accessible via a new read-only `data` property on `Memory`. The existing `shape`, `depth`, `init` properties become aliases for `data.shape`, `data.depth`, `data.init`.

`MemoryInstance` constructor is likewise changed to:

- `amaranth.hdl.MemoryInstance(data: MemoryData, *, attrs={})`

The `sim.memory_read` and `sim.memory_write` methods proposed by RFC 36 are removed. Instead, the new `Memory._Row` simulation-only value is introduced, which can be passed to `get`/`set`/`changed` like any other value. Masked writes can be implemented by `set` with a slice of `MemoryData._Row`, just like for signals.

`MemoryData` and `MemoryData._Row` instances (possibly wrapped in `ShapeCastable` for the latter) can be added to the `traces` argument when writing a VCD file with the simulator.

Using `MemoryData._Row` within an elaboratable results in an immediate error, even if the design is only to be used in simulation. The only place where `MemoryData._Row` is valid is within an argument to `sim.get`/`sim.set`/`sim.changed` and similar functions.

`sim.edge` remains restricted to plain `Signal` and single-bit slices thereof. `MemoryData._Row` is not supported.

## Drawbacks
[drawbacks]: #drawbacks

None.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

`MemoryData` having `shape`, `depth`, and `init` is necessary to allow the simulator to create the underlying storage if the memory is not included in the design hierarchy, but is used to communicate between simulator processes.

## Prior art
[prior-art]: #prior-art

`MemoryData` is conceptually equivalent to a 2D `Signal`, for simulation purposes. It thus follows similar rules.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

A `MemoryData.Slice` class could be added, allowing a whole range of memory addresses to be `get`/`set` at once, monitored for changes with `changes`, or added to `traces`.

Support for `__getitem__(Value)` could be added (currently it would be blocked on CXXRTL capabilities).
