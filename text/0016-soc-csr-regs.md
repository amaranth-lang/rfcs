- Start Date: 2024-02-02
- RFC PR: [amaranth-lang/rfcs#16](https://github.com/amaranth-lang/rfcs/pull/16)
- Amaranth SoC Issue: [amaranth-lang/amaranth-soc#68](https://github.com/amaranth-lang/amaranth-soc/issues/68)

# CSR register definition RFC

## Summary
[summary]: #summary

Add primitives to define CSR registers.

## Motivation
[motivation]: #motivation

The Amaranth SoC library support for CSRs currently consists of bus primitives behind which multiple registers can be gathered.

Its current notion of a CSR register is limited to the `csr.Element` class, which provides an interface between a register and a CSR bus. The information we have about a register is limited to its width and access mode (necessary to determine the layout of `csr.Element`), in addition to its name and address. This information can then be aggregated by walking through the memory map of a SoC, to generate header files (and documentation, etc) for use by firmware.

However, amaranth-soc lacks the notion of register fields. The CSR bus acts as a transport and isn't concerned about fields. Peripherals often expose their functionality using multiple fields of a register, and initiators (e.g. a CPU running firmware) need to be aware of them.

In addition, users must currently implement their own register primitives, which adds boilerplate.

This RFC aims to add a standard implementation of a CSR register, while building upon the existing infrastructure.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Currently, the implementation of a CSR register is left to the user, as amaranth-soc only requires the use of `csr.Element` as its interface:

```python3
class UARTPeripheral(Elaboratable):
    def __init__(self):
        self._phy = AsyncSerial(divisor=int(100e6//115200), data_width=8)

        self._divisor = csr.Element(self._phy.divisor.width, "rw")
        self._rx_rdy  = csr.Element(1, "r")
        self._rx_err  = csr.Element(1, "r")

        self._csr_mux = csr.Multiplexer(addr_width=4, data_width=32)
        self._csr_mux.add(self._divisor)
        self._csr_mux.add(self._rx_rdy)
        self._csr_mux.add(self._rx_err)

        self.bus = self._csr_mux.bus

    def elaborate(self, platform):
        m = Module()
        m.submodules.phy     = self._phy
        m.submodules.csr_mux = self._csr_mux

        m.d.comb += self._divisor.r_data.eq(self._phy.divisor)

        with m.If(self._divisor.w_stb):
            m.d.sync += self._phy.divisor.eq(self._divisor.w_data)

        # ...

        return m
```

This RFC adds register primitives to amaranth-soc, which are defined by subclassing `csr.Register`:

```python3
class UARTPeripheral(wiring.Component):
    # A register with a parameterized width and reset value:
    class Divisor(csr.Register, access="rw"):
        def __init__(self, *, width, reset):
            super().__init__({
                "divisor": csr.Field(csr.action.RW, width, reset=reset),
            })

    # A simple register, with reserved fields:
    class RxStatus(csr.Register, access="r"):
        rdy : csr.Field(csr.action.R,       unsigned(1))
        _0  : csr.Field(csr.action.ResRAW0, unsigned(3))
        err : csr.Field(csr.action.R,       unsigned(1))
        _1  : csr.Field(csr.action.ResRAW0, unsigned(3))

    class RxData(csr.Register, access="r"):
        data : csr.Field(csr.action.R, unsigned(8))

    def __init__(self, *, divisor):
        self._phy = AsyncSerial(divisor=int(100e6//115200), data_width=8)

        regs = csr.Builder(addr_width=4, data_width=8)

        self._divisor = regs.add("divisor", self.Divisor(width=bits_for(divisor), reset=divisor))

        with regs.Cluster("rx"):
            self._rx_status = regs.add("status", self.RxStatus(), offset=3)
            self._rx_data   = regs.add("data",   self.RxData(),   offset=4)

        self._bridge = csr.Bridge(regs.as_memory_map())

        super().__init__({
            "bus": In(csr.Signature(addr_width=4, data_width=8))
        })
        self.bus.memory_map = self._bridge.bus.memory_map

    def elaborate(self, platform):
        m = Module()

        m.submodules.phy    = self._phy
        m.submodules.bridge = self._bridge

        m.submodules.rx_fifo = rx_fifo = SyncFIFOBuffered(width=8 + 1, depth=16)

        m.d.comb += [
            # Reading a field from the peripheral side:
            self._phy.divisor.eq(self.divisor.f.divisor.data),

            rx_fifo.w_en  .eq(self._phy.rx.rdy),
            rx_fifo.w_data.eq(Cat(self._phy.rx.data, self._phy.rx.err)),
            self._phy.rx.ack.eq(rx_fifo.w_rdy),

            # Writing to a field from the peripheral side:
            self._rx_status.f.rdy.r_data.eq(rx_fifo.r_rdy),
            self._rx_status.f.err.r_data.eq(rx_fifo.r_data[-1]),

            # Consuming data from a FIFO, as a side-effect from a bus read:
            self._rx_data.f.data.r_data.eq(rx_fifo.r_data[:8]),
            rx_fifo.r_en.eq(self._rx_data.f.data.r_stb),
        ]

        return m
```

### Register definitions

The fields of a `Register` instance can be defined in two different ways:
- using [PEP 526](https://peps.python.org/pep-0526/) variable annotations.
- by calling `Register.__init__()` with a non-default `fields` argument.

Variable annotations are suitable for simple use-cases, whereas overriding `Register.__init__()` allows field definitions to be parameterized.

```python3
class UARTPeripheral(Elaboratable):
    class Divisor(csr.Register, access="rw"):
        def __init__(self, *, width, reset):
            super().__init__({
                "divisor": csr.Field(csr.action.RW, width, reset=reset),
            })

    class RxStatus(csr.Register, access="r"):
        rdy : csr.Field(csr.action.R,       unsigned(1))
        _0  : csr.Field(csr.action.ResRAW0, unsigned(3))
        err : csr.Field(csr.action.R,       unsigned(1))
        _1  : csr.Field(csr.action.ResRAW0, unsigned(3))

    class RxData(csr.Register, access="r"):
        data : csr.Field(csr.action.R, unsigned(8))

    ...
```

### Field access and ownership

The `csr.action` class definitions differ by their access mode.

For example, `csr.action.R` describes a field that is:
- *read-only* from the point-of-view of a *bus initiator* (such as a CPU);
- *read/write* from the point-of-view of the peripheral;

Whereas `csr.action.RW` describes a field that is:
- *read/write* from the point-of-view of a CPU
- *read-only* from the point-of-view of the peripheral

In this RFC, write access is defined by ownership. A register field can only be written to by its owner(s). For example:
- a `csr.action.R` field is owned by the peripheral;
- a `csr.action.RW` field is owned by the bus initiator.

In the `UARTPeripheral` example above, each register field has a single owner. This effectively removes the possibility of a write conflict between a CPU and the peripheral.

Otherwise, in case of shared ownership, deciding which transaction has precedence is context-dependent.

### Flag fields

Flag fields may be writable by both the bus initiator and the peripheral. Flag fields are distinct from other kinds of fields, as each bit may be set or cleared independently of others.

This RFC provides two types of flag:
- `csr.action.RW1C` (read/write-one-to-clear) flags may be used when a peripheral needs to notify a CPU of a condition (e.g. an error or a pending interrupt). The CPU clears the flag to acknowledge it. If a write conflict occurs, setting the bit from the peripheral side would have precedence.
- `csr.action.RW1S` (read/write-one-to-set) flags may be used for self-clearing bits, such as the enable bit of a one-shot timer. When the counter reaches its maximum value, it would automatically disable itself by clearing the enable bit. If a write conflict occurs, setting the bit from the CPU side would have precedence.

A use case that involves both `RW1C` and `RW1S` fields would be a register driving an array of GPIO pins. Their values may be set or cleared by a CPU. In a multitasked environment, a read-modify-write transaction would require locking to insure atomicity; whereas having two fields (`RW1S` and `RW1C`) targeting the same flags allows a CPU to set or clear any of them in a single write transaction.

### Reserved fields

Reserved fields may be defined to provide placeholders for past, future or undocumented functions.

This RFC provides four types of reserved fields:

- `csr.action.ResRAW0` (read-any/write-zero)
- `csr.action.ResRAWL` (read-any/write-last)
- `csr.action.ResR0WA` (read-zero/write-any)
- `csr.action.ResR0W0` (read-zero/write-zero)

#### Example use cases for reserved fields

##### One-Time Programmable fuse

- Field type: `ResRAW0` (read-any/write-zero)
- Reads return the fuse state. Writing 1 will blow the fuse.

##### Reserved for future use (as value)

- Field type: `ResRAWL` (read-any/write-last)
- Software drivers need to be aware of such fields, to ensure forward compatibility of software binaries with future silicon versions.
- Software drivers are assumed to access such fields by setting up an atomic read-modify-write transaction.
- The value returned by reads (and written back) must have defined semantics (e.g. a no-op) that can be relied upon in future silicon versions.

##### Reserved for future use (as flag)

- Field type: `ResRAW0` (read-any/write-zero)
- Software drivers need to be aware of such fields, to ensure forward compatibility of software binaries with future silicon versions.
- Software drivers do not need a read-modify-write transaction to write these fields.
- Software drivers should ignore the value returned by reads.
- Writing a value of 0 is a no-op for `RW1C` and `RW1S` flags, if implemented by future silicon versions.

##### Defined, but deprecated

- Field type: `ResR0WA` (read-zero/write-any)
- Such fields may be used as placeholders for phased out fields from previous silicon versions. They are required for backward compatibility with existing software binaries.
- The value of 0 returned by reads (and written back) must have defined semantics (e.g. a no-op).

##### Defined, but unimplemented

- Field type: `ResR0W0` (read-zero/write-zero)
- Such fields may be used to provide variants of a peripheral IP, and facilitate code re-use in software drivers.
- For example on STM32F0x SoCs, the CR1.CKD field (clock divider ratio) is read/write in the "general-purpose" timer TIM14 , but always reads 0 in the "basic" timer TIM6.

### Accessing register fields from peripherals

```python3
class UARTPeripheral(Elaboratable):
    ...

    def elaborate(self, platform):
        ...

        m.d.comb += [
            self._phy.divisor.eq(self._divisor.f.divisor.data),

            ...

            self._rx_status.f.rdy.r_data.eq(rx_fifo.r_rdy),
            self._rx_status.f.err.r_data.eq(rx_fifo.r_data[-1]),

            self._rx_data.f.data.r_data.eq(rx_fifo.r_data[:8]),
            rx_fifo.r_en.eq(self._rx_data.f.data.r_stb),
        ]

        ...
```

From the peripheral side, fields are exposed by the `<reg>.f` attribute of the `csr.Register` they belong to.

##### Access strobes

Peripherals can sample access strobes of `csr.action.R` and `csr.action.W` fields to perform side-effects:

 - `<reg>.f.<field>.r_stb` is asserted when the register is read from the CSR bus (i.e. it is hardwired to `<reg>.element.r_stb`);
 - `<reg>.f.<field>.w_stb` is asserted when the register is written by the CSR bus (i.e. it is hardwired to `<reg>.element.w_stb`).

##### Data

- `<reg>.f.<field>.w_data` is driven by the register if the field is write-only by the bus (i.e. `W`), and hardwired to a slice of `<reg>.element.w_data`.
- `<reg>.f.<field>.r_data` is driven by the peripheral if the field is read-only by the bus (i.e. `R`), and hardwired to a slice of `<reg>.element.r_data`.
- `<reg>.f.<field>.data` is driven by the register (with the value held in its storage), if the field is read-write for the bus (i.e. `RW`, `RW1C`, `RW1S`). It is updated one clock cycle after `<reg>.element.w_stb` is high (on the `sync` clock domain by default).
- `<reg>.f.<field>.set` is driven by the peripheral to set bits of a field that can be cleared by the bus (i.e. `RW1C`).
- `<reg>.f.<field>.clear` is driven by the peripheral to clear bits of a field that can be set by the bus (i.e. `RW1S`).


### Building registers

The `csr.Builder` provides fine-grained control over the address space occupied by the CSR registers of a peripheral. Registers may be organized into a hierarchy of clusters and arrays, which can be composed together (e.g. into an array of clusters, a multi-dimensional array, etc).

For example, an (artificially simplified) interrupt controller could use a 2-dimensional array of registers:

```python3
regs = csr.Builder(addr_width=csr_addr_width, data_width=32, granularity=8):

# For each CPU core and each group of 32 interrupts, add two registers: "IE" and "IP".
for i in range(nb_cpu_cores):
    with regs.Index(i):
        for j in range(ceil(nb_intr_sources / 32)):
            with regs.Index(j):
                self.ie[i][j] = regs.add("IE", self.IE(width=32))
                self.ip[i][j] = regs.add("IP", self.IP(width=32))
```


`regs.as_memory_map()` will create a `MemoryMap` containing those registers, which can be passed to a `csr.Bridge` to expose them over a CSR bus.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Fields

#### `csr.reg.FieldPort.Access`

The `FieldPort.Access` enum defines the supported access modes of a field, with:
- the following values: `R`, `W`, `RW` and `NC` (not connected);
- a `.readable(self)` method, which returns `True` if `self` is `R` or `RW`;
- a `.writable(self)` method, which returns `True` if `self` is `W` or `RW`.

#### `csr.reg.FieldPort`

The `FieldPort` class describes the interface between a register field and the register itself, with:
- a `.__init__(self, shape, access)` constructor, where `shape` is a shape-castable and `access` is a `FieldPort.Access` value;
- a `.shape` property;
- a `.access` property;

#### `csr.reg.FieldPort.Signature`

The `FieldPort.Signature` class describes the signature of a `FieldPort`, with:
- a `.__init__(self, shape, access)` constructor, where `shape` is a shape-castable and `access` is a `FieldPort.Access` value;
- a `.shape` property;
- a `.access` property;
- a `.create(self, path=None, src_loc_at=0)` method that returns a compatible `FieldPort` object;
- a `.__eq__(self, other)` method that returns `True` if both `self` and `other` have the same shape and access mode.

##### Signature members

The members of a `FieldPort.Signature` are defined as follows:

```python3
{
    "r_data": In(self.shape),
    "r_stb":  Out(1),
    "w_data": In(self.shape),
    "w_stb":  Out(1)
}
```

The access mode of a `FieldPort.Signature` has no influence on its members (e.g. `w_data` and `w_stb` are present even if `access.writable()` returns `False`).

#### `csr.reg.FieldAction`

The `FieldAction` class is a `Component` subclass implementing the behavior of a register field, with:
- a `.__init__(self, shape, access, members=()` constructor, where:
    * `shape` is a shape-castable;
    * `access` is a `FieldPort.Access` value;
    * `members` is an iterable of key/value pairs, where keys are strings and values are signature members.
- a `.signature` property, that returns a `Signature` with the following members:

```python3
{
    "port": In(FieldPort.Signature(shape, access)),
    **members
}
```
where `shape`, `access` and `members` are provided in `.__init__()`.

#### `csr.reg.Field`

The `Field` class serves as a factory for builtin or user-defined `FieldAction`s, with:
- a `.__init__(self, action_cls, *args, **kwargs)` constructor, where:
    * `action_cls` is a `FieldAction` subclass;
    * `*args` and `**kwargs` are arguments passed to `action_cls.__init__()`;
- a `.create(self)` method that returns `action_cls(*args, **kwargs)`.

#### `csr.reg.FieldActionMap`

The `FieldActionMap` class describes an immutable mapping of `FieldAction` objects, with:
- a `.__init__(self, fields)` constructor, where `fields` is a dict of strings to either `Field` objects, nested dicts or lists;
- a `.__getitem__(self, key)` method to lookup a field instance by name, without recursion;
- a `.__getattr__(self, name)` method to lookup a field instance by name, without recursion and excluding fields whose name start with `"_"`;
- a `.flatten(self)` method that yields for each field, a tuple containing its path (as a tuple of names or indices) and its instance.

A `FieldActionMap` contains instances of the fields given in `__init__()`:
- `Field` objects are instantiated as `FieldAction` by calling `Field.create()`;
- `dict` objects are instantiated as `FieldActionMap`;
- `list` objects  are instantiated as `FieldActionArray`.

A `FieldActionMap` preserves the iteration order of its fields, from least significant to most significant.

#### `csr.reg.FieldActionArray`

The `FieldActionArray` class describes an immutable sequence of `FieldAction` objects, with:
- a `.__init__(self, fields)` constructor, where `fields` is a list of either `Field` objects, nested dicts or lists;
- a `.__getitem__(self, key)` method to lookup a field instance by index, without recursion;
- a `.flatten(self)` method that yields for each field, a tuple containing its path (as a tuple of names or indices) and its instance.

A `FieldActionArray` contains instances of the fields given in `__init__()`:
- `Field` objects are instantiated as `FieldAction` by calling `Field.create()`;
- `dict` objects are instantiated as `FieldActionMap`;
- `list` objects  are instantiated as `FieldActionArray`.

A `FieldActionArray` preserves the iteration order of its fields, from least significant to most significant.

### Built-in field actions

#### `csr.action.R`

The `csr.action.R` class describes a read-only `FieldAction`, with:
- a `.__init__(self, shape)` constructor, where `shape` is a shape-castable;
- a `.signature` property, that returns a `Signature` with the following members:

```python3
{
    "port":   In(FieldPort.Signature(shape, access="r")),
    "r_data": In(shape),
    "r_stb":  Out(unsigned(1)),
}
```

- a `.elaborate(self, platform)` method, where `self.r_data` and `self.r_stb` are connected to `self.port.r_data` and `self.port.r_stb`.

#### `csr.action.W`

The `csr.action.W` class describes a write-only `FieldAction`, with:
- a `.__init__(self, shape)` constructor, where `shape` is a shape-castable.
- a `.signature` property, that returns a `Signature` with the following members:

```python3
{
    "port":   In(FieldPort.Signature(shape, access="w")),
    "w_data": Out(shape),
    "w_stb":  Out(unsigned(1)),
}
```

- a `.elaborate(self, platform)` method, where `self.port.w_data` and `self.port.w_stb` are connected to `self.w_data` and `self.port.w_stb`.

#### `csr.action.RW`

The `csr.action.RW` class describes a read-write `FieldAction`, with:
- a `.__init__(self, shape, reset=0)` constructor, where `shape` is a shape-castable and `reset` is a const-castable defining the reset value of internal storage;
- a `.reset` property;
- a `.signature` property, that returns a `Signature` with the following members:

```python3
{
    "port": In(FieldPort.Signature(shape, access="rw")),
    "data": Out(shape)
}
```

- a `.elaborate(self, platform)` method, where `self.port.w_data` is used to synchronously write internal storage. Storage output is connected to `self.data` and `self.port.r_data`.

#### `csr.action.RW1C`

The `csr.action.RW1C` class describes a read-write `FieldAction`, with:
- a `.__init__(self, shape, reset=0)` constructor, where `shape` is a shape-castable and `reset` is a const-castable defining the reset value of internal storage.
- a `.reset` property;
- a `.signature` property, that returns a `Signature` with the following members:

```python3
{
    "port": In(FieldPort.Signature(shape, access="rw")),
    "data": Out(shape),
    "set":  In(shape)
}
```

- a `.elaborate(self, platform)` method, where high bits in `self.port.w_data` and `self.port.set` are used to synchronously clear and set internal storage, respectively. Storage output is connected to `self.data` and `self.port.r_data`.

#### `csr.action.RW1S`

The `csr.action.RW1S` class describes a read-write `FieldAction`, with:
- a `.__init__(self, shape, reset=0)` constructor, where `shape` is a shape-castable and `reset` is a const-castable defining the reset value of internal storage;
- a `.reset` property;
- a `.signature` property, that returns a `Signature` with the following members:

```python3
{
    "port":  In(FieldPort.Signature(shape, access="rw")),
    "data":  Out(shape),
    "clear": In(shape)
}
```

- a `.elaborate(self, platform)` method, where high bits in `self.port.w_data` and `self.port.clear` are used to synchronously set and clear internal storage, respectively. Storage output is connected to `self.data` and `self.port.r_data`.

### Built-in reserved field actions

#### `csr.action.ResRAW0`, `csr.action.ResRAWL`, `csr.action.ResR0WA`, `csr.action.ResR0W0`

These classes describe a reserved `FieldAction`, with:
- a `.__init__(self, shape)` constructor, where `shape` is a shape-castable;
- a `.signature` property, that returns a `Signature` with the following members:

```python3
{
    "port":  In(FieldPort.Signature(shape, access="nc")),
}
```

- a `.elaborate(self, platform)` method, that returns an empty `Module()`.

### Registers

#### `csr.reg.Register`

The `csr.reg.Register` class describes a CSR register `Component`, with:
- a `.__init_subclass__(cls, access=None, **kwargs)` class method, where `access` is either a `csr.Element.Access` value, or `None`.
- a `.__init__(self, fields=None, access=None)` constructor, where:
    * `fields` is either:
        - a `dict` that will be instantiated as a `FieldActionMap`;
        - a `list` that will be instantiated as a `FieldActionArray`;
        - `None`; in this case a `FieldActionMap` is instantiated from `Field` objects in variable annotations.
    * `access` is either a `csr.Element.Access` value, or `None`.
- a `.fields` property, returning a `FieldActionMap` or a `FieldActionArray`;
- a `.f` property, as a shorthand to `self.fields`;
- a `.__iter__(self)` method, as a shorthand to `self.fields.flatten()`;
- a `.signature` property, that returns a `Signature` with the following members:

```python3
{
    "element": Out(Element.Signature(width, access))
}
```

where `width` is the total width of the register, i.e. `sum(Shape.cast(f.port.shape).width for _, f in self`.

- a `.elaborate(self, platform)` method, that connects fields to slices of `self.element`, depending on their access mode;

##### Element access mode

The `access` parameter must be provided in `__init_subclass__()` or `__init__()`. A `ValueError` is raised in `__init__()` if:
- `access` is provided in neither method;
- `access` is provided in both methods with different values.

#### `csr.reg.Builder`

The `csr.reg.Builder` class can build a `MemoryMap` from a group of `csr.Register` objects, with:
- a `.__init__(self, *, addr_width, data_width, granularity=8, name=None)` constructor that:
    * raises a `TypeError` if `addr_width`, `data_width` and `granularity` are not positive integers;
    * raises a `ValueError` if `granularity` is not a divisor of `data_width`.

- `.addr_width`, `.data_width`, `.granularity` and `.name` properties;

- a `.freeze(self)` method, which renders the visible state of the `csr.Builder` immutable;

- a `.add(self, name, register, *, offset=None)` method, which:
    * adds `register` to the builder;
    * returns `register`;
    * raises a `ValueError` if `self` is frozen;
    * raises a `TypeError` if `register` is not a `Register` object;
    * raises a `ValueError` if `register` is already present;
    * raises a `TypeError` if `name` is not a non-empty string;
    * raises a `ValueError` if `name` is already assigned to another register or `Cluster`.
    * raises a `TypeError` if `offset` is neither a positive integer or 0;
    * raises a `ValueError` if `offset` is not word-aligned (i.e. a multiple of `self.data_width // self.granularity`);

- a `.Cluster(self, name)` context manager method, which:
    * upon entry, creates a scope where registers added by `self.add()` are assigned to a cluster named `name`;
    * raises a `ValueError` if `self` is frozen;
    * raises a `TypeError` if `name` is not a non-empty string;
    * raises a `ValueError` if `name` is already assigned to another register or `Cluster`;

- a `.Index(self, index)` context manager method, which:
    * upon entry, creates a scope where registers added by `self.add()` are assigned to an array index `index`;
    * raises a `ValueError` if `self` is frozen;
    * raises a `TypeError` if `index` is neither a positive integer or 0;
    * raises a `ValueError` if `index` is already assigned to another `Index`;

- a `.as_memory_map(self)` method, that converts `self` into a `MemoryMap`. `self.freeze()` is implicitly called as a side-effect.

### CSR bus primitives

#### Changes to `memory.MemoryMap`

`MemoryMap.add_resource(self, resource, *, name, size, addr=None, alignment=None)` now requires `resource` to be a `wiring.Component` object.

#### Changes to `csr.bus.Multiplexer`

`Multiplexer` instances are now created from a caller-provided `MemoryMap`, instead of creating and populating one itself.

- replace `.__init__(self, addr_width, data_width, alignment, name)` with `.__init__(memory_map)`, that:
    * raises a `TypeError` if `memory_map` is not a `MemoryMap` object;
    * raises a `ValueError` if `memory_map` has windows.
    * raises a `TypeError` if `memory_map` has resources that are not `wiring.Component` objects with the following signature:

```python3
{
    "element": Out(csr.Element.Signature(...)),
    # additional members are allowed
}
```

- remove the `.align_to(self, alignment)` method;
- remove the `.add(self, elem, name, addr=None, alignment=None)` method.

#### `csr.reg.Bridge`

The `csr.reg.Bridge` class describes a `wiring.Component` that mediates access between a CSR bus and a group of `csr.Register`s, with:
- a `.__init__(self, memory_map)` constructor, that:
    * freezes and assigns `memory_map` to `self.bus.memory_map`;
    * raises a `TypeError` if `memory_map` is not a `MemoryMap` object;
    * raises a `ValueError` if `memory_map` has windows.
    * raises a `TypeError` if `memory_map` has resources that are not `csr.Register` objects;
- a `.signature` property, that returns a `wiring.Signature` with the following members:

```python3
{
    "bus": In(csr.Signature(addr_width=memory_map.addr_width, data_width=memory_map.data_width))
}
```

- a `.elaborate(self, platform)` method, that instantiates a `csr.Multiplexer` submodule and connects its bus interface to `self.bus`. The registers in `self.bus.memory_map` are added as submodules.

## Drawbacks
[drawbacks]: #drawbacks

- While this RFC attempts to provide escape hatches to allow users to circumvent some or all of the proposed API, it is possible that common use-cases may be complicated or impossible to implement, due to the author's oversight.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- The existing CSR infrastructure already guarantees that a CSR access completes atomically. This RFC builds upon it by reasoning in terms of atomic transactions: it identifies scenarios where a write-conflict may happen, and either prevents it (e.g. by restricting a field to a single owner) or defines clear precedence rules.
- The absence of `csr.action.RW0S` and `csr.action.RW0C` is voluntary, to allow a write of 0 to be no-op.
- Alternatively, do nothing. This maximises user freedom, at the cost of boilerplate. A proliferation of downstream CSR register implementations would prevent amaranth-soc's BSP generator from gathering register fields to generate safe accessors and documentation.

## Prior art
[prior-art]: #prior-art

The Rocket Chip Generator has a [register API](https://github.com/chipsalliance/rocket-chip/tree/master/src/main/scala/regmapper) that supports some use-cases of this RFC:

- Each field of a register is a component with its own interface and access mode.
- Reserved fields are neither readable nor writable.
- A field is created as a `RegField` instance with separate `RegWriteFn` and `RegReadFn` functions implementing its behavior, whereas a `csr.FieldAction` in this RFC implements both.
- Its `RegField.rwReg` built-in has the same write latency as `csr.action.RW`, but differs by having users provide the register storage.
- Its `RegField.w1ToClear` built-in has the same behavior as `csr.action.RW1C` (besides the previous point). The peripheral side can only set bits and has precedence in case of set/clear conflicts.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- What conventions should we follow when documenting CSR registers ?

## Future possibilities
[future-possibilities]: #future-possibilities

The notion of ownership in CSR registers can be expanded throughout the entire SoC (interconnect primitives, peripherals, events, etc).

Having an explicit model of ownership across the amaranth-soc library could allow us to provide strong safety guarantees against some concurrency hazards (e.g. two CPU cores writing to the same peripheral).

## Acknowledgements

@whitequark, @zyp, @tpwrules, @galibert and @Fatsie provided valuable feedback while this RFC was being drafted.
