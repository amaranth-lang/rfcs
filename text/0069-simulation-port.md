- Start Date: 2024-07-01
- RFC PR: [amaranth-lang/rfcs#69](https://github.com/amaranth-lang/rfcs/pull/69)
- Amaranth Issue: [amaranth-lang/amaranth#1446](https://github.com/amaranth-lang/amaranth/issues/1446)

# Add a `lib.io.PortLike` object usable in simulation

## Summary
[summary]: #summary

A new library I/O port object `lib.io.SimulationPort` is introduced that can be used to simulate `lib.io.Buffer` and `lib.io.FFBuffer` components.

## Motivation
[motivation]: #motivation

End-to-end verification of I/O components requires simulating `lib.io.Buffer`, `lib.io.FFBuffer`, or `lib.io.DDRBuffer` objects, which is currently not possible because all defined library I/O ports (`lib.io.SingleEndedPort`, `lib.io.DifferentialPort`) require the use of unsimulatable core I/O values. Simulating these buffers can be done by adding a new library I/O port object with separate `i`, `o`, `oe` parts represented by normal `Signal`s, and modifying the lowering of `lib.io` buffer components to drive the individual parts. Simulation models could then be written using code similar to (for SPI):

```python
await ctx.set(cipo_port.o, cipo_value)
_, copi_value = await ctx.posedge(sck_port.i).sample(copi_port.i)
```

This functionality is one of the last pieces required to build a library of reusable I/O interfaces, and the need for it became apparent while writing reusable I/O blocks for the Glasgow project.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Let's consider an I/O component, such as below, which takes several ports as its constructor argument:

```python
class SPIController(wiring.Component):
    o_data: stream.Signature(8)
    i_data: stream.Signature(8)

    def __init__(self, sck_port, copi_port, cipo_port):
        self._sck_port  = sck_port
        self._copi_port = copi_port
        self._cipo_port = cipo_port

        super().__init__()

    def elaborate(self, platform):
        m = Module()

        m.submodules.sck  = sck_buffer  = io.FFBuffer("o", self._sck_port)
        m.submodules.copi = copi_buffer = io.FFBuffer("o", self._copi_port)
        m.submodules.cipo = cipo_buffer = io.FFBuffer("i", self._cipo_port)

        # ... wire it up to self.[io]_data...

        return m
```

To simulate such a component, instantiate `lib.io.SimulationPort` objects for each of the ports, and use their `.o`, `.oe`, and `.i` signals to verify the functionality:

```python
sck_port  = io.SimulationPort("o", 1)
copi_port = io.SimulationPort("o", 1)
cipo_port = io.SimulationPort("i", 1)

dut = SPIController(sck_port, copi_port, cipo_port)

async def testbench_peripheral(ctx):
    """Simulates behavior of an SPI peripheral. Simplified for clarity: ignores chip select."""
    for copi_fixture, cipo_fixture in zip([0,1,0,1,0,1,0,1], [1,0,1,0,1,0,1,0]):
        await ctx.negedge(sck_port.o)
        ctx.set(cipo_port.i, cipo_fixture)
        _, copi_driven, copi_value = await ctx.posedge(sck_port.o).sample(copi_port.oe, copi_port.o)
        assert copi_driven == 0b1 and copi_value == copi_fixture

async def testbench_controller(ctx):
    """Issues commands to the controller."""
    await stream_put(ctx, dut.o_data, 0x55)
    assert (await stream_get(ctx, dut.i_data)) == 0xAA

sim = Simulator(dut)
sim.add_clock(1e-6)
sim.add_testbench(testbench_peripheral)
sim.add_testbench(testbench_controller)
sim.run()
```

Note that the peripheral testbench not only checks that the right values are being output by the I/O component, but that the port is driven (not tristated) at the time when values are sampled.

Note also that e.g. `copi_port.oe` is a multi-bit signal, with the same width as the port itself. This is unlike the `lib.io` buffers, whose `oe` port member is always 1-bit wide. This means that for multi-wire ports that are always fully driven or fully undriven, it is necessary to compare them with `0b00..0` or `0b11..1`, according to the width of the port.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `amaranth.lib.io` module is expanded with a new subclass of `PortLike`, `SimulationPort(direction, width, *, invert=False)`:
- `i`, `o`, `oe` are read-only properties each containing a new `Signal(width)`;
- `invert` is a property equal to the `invert` parameter normalized to a tuple of `width` elements;
- `direction` is a property equal to the `direction` parameter normalized to `Direction` enum;
- `len(port)` returns `width`;
- `port[...]` returns a new `SimulationPort` whose `i`, `o`, `oe` properties are slices of `port.i`, `port.o`, `port.oe` respectively;
- `~port` returns a new `SimulationPort` whose `i`, `o`, `oe` properties are the same as for `port` and the `invert` property has each value inverted;
- `port + other`, where `other` is a `SimulationPort`, returns a new `SimulationPort` whose `i`, `o`, `oe` properties are concatenations of the respective properties of `port` and `other`.

Since this is a third `PortLike` with a compatible `__add__` implementation, and it is weird to have slicing but not concatenation, the `__add__` method is added to the `PortLike` signature, with a deprecation warning in 0.5.1 and a hard requirement in 0.6.0.

The `amaranth.lib.io.Buffer` component is changed to accept `SimulationPort`s and performs the following connections (with the direction checks omitted for clarity):

```python
m.d.comb += [
    buffer.i.eq(Cat(Mux(oe, o, i) for oe, o, i in zip(port.oe, port.o, port.i))),
    port.o.eq(buffer.o),
    port.oe.eq(buffer.oe.replicate(len(port))),
]
```

The `amaranth.lib.io.FFBuffer` component is changed to accept `SimulationPort`s and performs the following connections (with the direction checks omitted for clarity):

```python
m.d[buffer.i_domain] += [
    buffer.i.eq(Cat(Mux(oe, o, i) for oe, o, i in zip(port.oe, port.o, port.i))),
]
m.d[buffer.o_domain] += [
    port.o.eq(buffer.o),
    port.oe.eq(buffer.oe.replicate(len(port))),
]
```

The `amaranth.lib.io.DDRBuffer` component is not changed.

None of the `get_io_buffer` functions in the vendor platforms are changed. Thus, `SimulationPort` is not usable with any vendor platform that implements `get_io_buffer`.

To improve the health of the Amaranth standard I/O library (which will make heavy use of simulation port objects), the changes are back-ported to the 0.5.x release branch.

## Drawbacks
[drawbacks]: #drawbacks

* Bigger API surface.
* Another concept to keep track of.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Alternatives:

* The signature of the `SimulationPort()` constructor is designed to closely resemble that of `SingleEndedPort()` and `DifferentialPort()`. However, `SimulationPort()` is intended to be only constructed by the designer, and the latter two almost never are. Is it a good idea to make direction optional, in this case? Specifying the direction provides additional checks that are valuable for verification.
    * Alternative #0: `SimulationPort(width, *, invert=False, direction=Direction.Bidir)`. This alternative is a near-exact match to the other port-like objects, with the exception of `io` replaced by `width`.
    * Alternative #1: `SimulationPort(width, *, invert=False, direction)`. This alternative minimally changes the signature, but requires lengthy `direction=` at every call site. **Currently in the RFC text.**
    * Alternative #2: `SimulationPort(width, direction, *, invert=False)`. This alternative is a slightly less minimal change, but does not match the ordering of arguments of signature of `Buffer(direction, port)`.
    * Alternative #3: `SimulationPort(direction, width, *, invert=False)`. This alternative differs significantly from `SingleEndedPort()` etc, but matches `Buffer()` most closely.
* It would be possible to make `SimulationPort` an interface object. Although the opportunities for connecting it are limited (since the signals are only synchronized to each other and not to any clock domains in the design), one can foresee the addition of a component that implements end-to-end or multi-drop buses using simulation-compatible logic, for example, to simulate multiple Amaranth components accessing a common SPI or I2C bus. In this context `connect` would be valuable.
    * This functionality could be prototyped out-of-tree (by wrapping `SimulationPort` or simply setting its `signature` property) and added upstream later if it proves to be useful.
* It would be possible to add `o_init`, `i_init`, `oe_init` to the signature for a minor usability improvement in cases where the port is never driven in simulation, but the benefit seems low and the signature becomes much more cluttered.
    * It is no longer possible since Amaranth 0.5 to directly set `.init` attribute of signals, so without these arguments the initial value will always be 0.

Rejected alternatives:
* ~~Multi-bit `oe` signal is a radical departure from the rest of Amaranth, where `oe` is always single-bit. However, note the proposed RFC #68, which suggests making `oe` optionally multi-bit for `io.Buffer` etc.~~
    * ~~It would be possible to make `oe` a single-bit signal and to forbid slicing of `SimulationPort` objects. However, the `PortLike` interface does not currently allow this, and it would have to be changed, with generic code having to be aware of this new error.~~
    * ~~It would also be possible to make `oe` a choice of multi-bit or single-bit signal during construction, similar to RFC #68.~~
    * Without multibit `oe` it is not possible to concatenate ports.

## Prior art
[prior-art]: #prior-art

We have previously had, and deprecated, the `amaranth.lib.io.Pin` object, which is essentially the same triple of `i`, `o`, `oe`. However, in terms of its intended use (notwithstanding how it was actually used), the `Pin` object was closest to the interface created by `io.Buffer.Signature.create()`. The newly introduced `SimulationPort` is intended to represent data on the other side of the buffer than `Pin` does.

## Resolved questions
[resolved-questions]: #resolved-questions

- ~~What should the signature of `SimulationPort()` be?~~
    - Alternative #3 (matching `io.Buffer`, etc) was decided.

## Future possibilities
[future-possibilities]: #future-possibilities

- Add `DDRBuffer` support by using `port.o.eq(Mux(ClockSignal(), self.o[1], self.o[0]))`.
    - It is not yet clear if this can be safely used, or should be used, in simulations.
- Make `SimulationPort` an interface object.
- Add a `EndToEndBus` (naming TBD) class connecting pairs of ports to each other and ensuring that no wire is driven twice.
- Add a `OpenDrainBus` (naming TBD) class connecting a set of ports to each other and providing a "pull-up resistor" equivalent functionality.
