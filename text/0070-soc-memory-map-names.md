- Start Date: 2024-06-28
- RFC PR: [amaranth-lang/rfcs#70](https://github.com/amaranth-lang/rfcs/pull/70)
- Amaranth SoC Issue: [amaranth-lang/amaranth-soc#93](https://github.com/amaranth-lang/amaranth-soc/issues/93)

# Unify the naming of `MemoryMap` resources and windows

## Summary
[summary]: #summary

- Use a `MemoryMap.Name` class to represent resource and window names.
- Assign window names in `MemoryMap.add_window()` instead of `MemoryMap.__init__()`.

## Motivation
[motivation]: #motivation

In a `MemoryMap`, resources (e.g. registers of a peripheral) and windows (nested memory maps) are added with similar methods: `MemoryMap.add_resource()` and `.add_window()`.

However, their names are handled differently:
- a resource name is a tuple of strings, assigned as a parameter to `MemoryMap.add_resource()`.
- a window name is a string, assigned at the creation of the window itself (as a parameter to `MemoryMap.__init__()`).

These differences add needless complexity to the API:
- the name of a window is only relevant from the context of the memory map to which it is added.
- window names may also benefit from being split into tuples, in order to let consumers of the memory map (such as a BSP generator) decide their format.

Additionally, support for integers in resource and window names is needed to represent indices. A BSP generator may choose to format them in a specific way (e.g. `"foo[1]"`).

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Resource and window names are instances of `MemoryMap.Name`, a subclass of `tuple` which can only contain non-empty strings and non-negative integers.

Some examples:

```python3
>>> MemoryMap.Name(("rx", "status"))
Name('rx', 'status')
>>> MemoryMap.Name(("uart", 0))
Name('uart', 0)
>>> MemoryMap.Name(MemoryMap.Name(("uart", 0))
Name('uart', 0)
>>> MemoryMap.Name("foo")
Name('foo',)
```

### Assigning resource names

The name of a resource is given by the `name` parameter of `MemoryMap.add_resource()`. For the sake of brevity, users can pass a tuple which is implicitly cast to a `MemoryMap.Name`.

This example shows how names are assigned to the registers of an UART peripheral:

```python3
from amaranth import *
from amaranth.lib import wiring
from amaranth.lib.wiring import In, Out

from amaranth_soc import csr
from amaranth_soc.memory import MemoryMap


class UART(wiring.Component):
    class RxConfig(csr.Register, access="rw"):
        enable: csr.Field(csr.action.RW, 1)

    class RxStatus(csr.Register, access="rw"):
        ready: csr.Field(csr.action.R,    1)
        error: csr.Field(csr.action.RW1C, 1)

    class RxData(csr.Register, access="r"):
        def __init__(self):
            super().__init__(csr.Field(csr.action.R, 8))

    csr_bus: In(csr.Signature(addr_width=10, data_width=32))

    def __init__(self):
        super().__init__()

        memory_map = MemoryMap(addr_width=10, data_width=32)
        memory_map.add_resource(self.RxConfig(), size=1, name=("rx", "config"))
        memory_map.add_resource(self.RxStatus(), size=1, name=MemoryMap.Name(("rx", "status")))
        memory_map.add_resource(self.RxData(),   size=1, name=("rx", "data"))
        memory_map.freeze()

        self.csr_bus.memory_map = memory_map

    def elaborate(self, platform):
        m = Module()
        # ...
        return m
```

### Assigning window names

Similarly, the name of a window is given by the `name` parameter of `MemoryMap.add_window()`. Unlike resource names, window names are optional.

This example shows how names are assigned to two UART peripherals, as their memory maps are added as windows to a bus decoder memory map:

```python3
class Decoder(wiring.Component):
    csr_bus: In(csr.Signature(addr_width=20, data_width=32))

    def __init__(self):
        super().__init__()
        self._subs = dict()
        self.csr_bus.memory_map = MemoryMap(addr_width=20, data_width=32)

    def add(self, sub_bus, *, name=None, addr=None):
        self._subs[sub_bus.memory_map] = sub_bus
        return self.csr_bus.memory_map.add_window(sub_bus.memory_map, name=name, addr=addr)

    def elaborate(self, platform):
        m = Module()

        with m.Switch(self.csr_bus.addr):
            for window, name, (pattern, ratio) in self.csr_bus.memory_map.window_patterns():
                sub_bus = self._subs[window]
                with m.Case(pattern):
                    pass # ... drive the subordinate bus interface

        return m


uart_0 = UART()
uart_1 = UART()

decoder = Decoder()
decoder.add(uart_0.csr_bus, name=("uart", 0))
decoder.add(uart_1.csr_bus, name=MemoryMap.Name(("uart", 1)))
```

In a `MemoryMap` hierarchy, each resource is identified by a path. The path of a resource is a tuple ending with its name, preceded by the name of each window that contains it:

```python3
>>> for res_info in decoder.csr_bus.memory_map.all_resources():
...     print(res_info.path)
(Name('uart', 0), Name('rx', 'config'))
(Name('uart', 0), Name('rx', 'status'))
(Name('uart', 0), Name('rx', 'data'))
(Name('uart', 1), Name('rx', 'config'))
(Name('uart', 1), Name('rx', 'status'))
(Name('uart', 1), Name('rx', 'data'))
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The following changes to the `amaranth_soc.memory` module are made:

- Add a `MemoryMap.Name(name, /)` class. It is a subclass of `tuple`, where:

  * `name` must either be a string or a tuple of strings and non-negative integers. If `name` is a string, it is implicitly converted to `(name,)`.

- The following changes to `MemoryMap` are made:

  * The optional `name` parameter of `MemoryMap()` is removed.
  * The `MemoryMap.name` property is removed.
  * The `name` parameter of `MemoryMap.add_resource()` must be a `MemoryMap.Name`.
  * An optional `name` parameter is added to `MemoryMap.add_window()`, which must be a `MemoryMap.Name`.
  * The yield values of `MemoryMap.windows()` are changed to `window, name, (start, end, ratio)`.
  * The yield values of `MemoryMap.window_patterns()` are changed to `window, name, (pattern, ratio)`.

- The following changes to `ResourceInfo` are made:

  * The `path` parameter of `ResourceInfo()` must be a tuple of `MemoryMap.Name` objects.

As a consequence of this proposal, the following changes are made to other modules:

- `amaranth_soc.csr.bus` and `amaranth_soc.wishbone.bus`:
  * The optional `name` parameter of `Decoder()` is moved to `Decoder.add()`.

- `amaranth_soc.csr.reg`:
  * The optional `name` parameter of `Builder()` is removed.

- `amaranth_soc.csr.wishbone`:
  * The optional `name` parameter of `WishboneCSRBridge()` is assigned to `csr_bus.memory_map` (instead of `wb_bus.memory_map`).

## Drawbacks
[drawbacks]: #drawbacks

- This will break codebases that make use of window names.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Providing a `MemoryMap.Name` class for resource and window names facilitates their validation and documentation.
  * ~~Alternative #1: do not add a class, and use standard tuples instead. Names will have to be validated by other means.~~
  * ~~Alternative #2: use `MemoryMap.Name` for resource names only. Window names remain limited to strings.~~

## Prior art
[prior-art]: #prior-art

- Resource names became tuples of strings as a consequence of [RFC 16](https://amaranth-lang.org/rfcs/0016-soc-csr-regs.html). However, array indices defined with `csr.Builder.Index()` were [cast to strings](https://github.com/amaranth-lang/amaranth-soc/issues/69) when `.as_memory_map()` was called.

## Resolved questions
[unresolved-questions]: #unresolved-questions

- Should we require the presence of at least one string in `MemoryMap.Name` ?
  * Empty names are forbidden and transparent windows (i.e. without names) must use `None` instead. Further validation is deferred to consumers of the memory map (e.g. a BSP generator).
- Should we specify the order between strings and integers ? In `csr.Builder`, array indices precede cluster and register names (e.g. `('bar', 0, 'foo')` could be formatted as `"bar.foo[0]"`).
  * No, this decision is left out to consumers of the memory map. They may interpret a name differently depending on what it is assigned to.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
