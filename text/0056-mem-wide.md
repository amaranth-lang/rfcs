- Start Date: 2024-03-18
- RFC PR: [amaranth-lang/rfcs#56](https://github.com/amaranth-lang/rfcs/pull/56)
- Amaranth Issue: [amaranth-lang/amaranth#1211](https://github.com/amaranth-lang/amaranth/issues/1211)

# Asymmetric memory port width

## Summary
[summary]: #summary

Memory read and write ports can have varying width, allowing eg. for memories with 8-bit read path and 32-bit write path.

## Motivation
[motivation]: #motivation

This is a common hardware feature. It allows for eg. having a slow but wide port in one domain, and fast but narrow port in another domain. On platforms lacking dedicated hardware support, it can often be emulated almost for free.


## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Memories can have asymmetric port width. To use that feature, instantiate the memory with the shape of the narrowest desired port, then pass the `aggregate` argument on ports that should be wider than that:

```py
m.submodules.mem = mem = Memory(shape=unsigned(8), depth=4096, init=[])
# 8-bit write port
wp = mem.write_port()
# 32-bit read port
rp = mem.read_port(aggregate=4)
# Address 0x123 on rp is equivalent to addresses (0x123 * 4, 0x123 * 4 + 1, 0x123 * 4 + 2, 0x123 + 3) on wp.
# Shape of rp.data is ArrayLayout(unsigned(8), 4)
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Both `lib.memory.Memory.read_port` and `lib.memory.Memory.write_port` have a new `aggregate=None` keyword-only argument. If `aggregate` is not `None`, the behavior is as follows:

- `aggregate` has to be a power of two
- `mem.depth` must be divisible by `aggregate`
- the `shape` passed to the `*Port.Signature` constructor becomes `ArrayLayout(memory.shape, aggregate)`
- implied by the previous point, `granularity` on wide write ports is counted in terms of single memory row
- the `addr_width` passed to `*Port.Signature` constructor becomes `ceil_log2(memory.depth // aggregate)`

The behavior of wide ports is defined by expanding them to `aggregate` narrow ports:

- the `data` of subport `i` is connected to `data[i]` of wide port
- the `addr` of subport `i` is connected to `addr * aggregate + i` of wide port
- for read ports and write ports without granularity, `en` is broadcast
- for write ports with granularity, `en` of subport `i` is connected to `en[i // granularity]` of wide port

No change is made to signature types or port types. Wide ports are recognized solely by their relation to `memory.shape`.

The rules for `MemoryInstance.read_port` and `MemoryInstance.write_port` change as follows:

- define `aggregate_log2 = ceil_log2(depth) - len(addr)`, `aggregate = 1 << aggregate_log2`
- `aggregate_log2` must be non-negative
- `depth` must be divisible by `aggregate`
- `len(data)` must be equal to `width * aggregate`
- for write ports, one of the following must hold:
  - `aggregate` is divisible by `len(en)`
  - `len(en)` is divisible by `aggregate` and `len(data)` is divisible by `len(en)`

## Drawbacks
[drawbacks]: #drawbacks

More complexity.

Wide write ports with sub-row write granularity cannot be expressed. However, there is no hardware that would actually natively support such a combination.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The design is straightforward enough.

An alternative is not doing this. Yosys already has an optimization pass that recognizes wide ports from a collection of narrow ports, so this is not necessarily an expressiveness hole. However, platforms with non-yosys toolchain could still benefit from custom lowering for this case.

## Prior art
[prior-art]: #prior-art

This proposal is directly based on yosys memory model.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

Similar functionality could potentially be added to `lib.fifo`.
