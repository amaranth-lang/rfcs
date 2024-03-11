- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#0036](https://github.com/amaranth-lang/rfcs/pull/0036)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Async testbench functions

## Summary
[summary]: #summary

Introduce an improved simulator testbench interface using `async`/`await` style coroutines.

## Motivation
[motivation]: #motivation

For the purpose of writing a testbench, an `async` function will read more naturally than a generator function, especially when calling subfunctions/methods.

A more expressive way to specify trigger/wait conditions allows the condition checking to be offloaded to the simulator engine, only returning control to the testbench process when it has work to do.

Passing a simulator context to the testbench function provides a convenient place to gather all simulator operations.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

As an example, let's consider a simple stream interface with `valid`, `ready` and `data` members.
On the interface class, we can then implement `.send()` and `.recv()` methods like this:

```python
class StreamInterface(PureInterface):
    async def recv(self, sim):
        await sim.set(self.ready, 1)
        await sim.tick().until(self.valid)

        value = await sim.get(self.data)

        await sim.tick()
        await sim.set(self.ready, 0)

        return value

    async def send(self, sim, value):
        await sim.set(self.data, value)

        await sim.set(self.valid, 1)
        await sim.tick().until(self.ready)

        await sim.tick()
        await sim.set(self.valid, 0)
```

`await sim.get()` and `await sim.set()` replaces the existing operations `yield signal` and `yield signal.eq()` respectively.

`sim.tick()` replaces the existing `Tick()`. It returns a trigger object that either can be awaited directly, or made conditional through `.until()`.

> **Note**
> This simplified example does not include any way of specifying the clock domain of the interface and as such is only directly applicable to single domain simulations.
> A way to attach clock domain information to interfaces is desireable, but out of scope for this RFC.

Using this stream interface, let's consider a colorspace converter accepting a stream of RGB values and outputting a stream of YUV values:

```python
class RGBToYUVConverter(Component):
    input: In(StreamSignature(RGB888))
    output: Out(StreamSignature(YUV888))
```

A testbench could then look like this:

```python
async def test_rgb(sim, r, g, b):
    rgb = {'r': r, 'g': g, 'b': b}
    await dut.input.send(sim, rgb)
    yuv = await dut.output.recv(sim)

    print(rgb, yuv)

async def testbench(sim):
    await test_rgb(sim, 0, 0, 0)
    await test_rgb(sim, 255, 0, 0)
    await test_rgb(sim, 0, 255, 0)
    await test_rgb(sim, 0, 0, 255)
    await test_rgb(sim, 255, 255, 255)
```

Since `.send()` and `.recv()` invokes `sim.get()` and `sim.set()` that in turn will invoke the appropriate value conversions for a value castable (here `data.View`), it is general enough to work for streams with arbitrary shapes.

`Tick()` and `Delay()` are replaced by `sim.tick()` and `sim.delay()` respectively.
In addition, `sim.changed()` and `sim.edge()` is introduced that allows creating triggers from arbitrary signals.
These all return a trigger object that can be made conditional through `.until()`.

`Active()` and `Passive()` are replaced by an `background=False` keyword argument to `.add_testbench()`.
Processes created through `.add_process()` are always created as background processes.
To allow a background process to ensure an operation is finished before end of simulation, `sim.critical()` is introduced, which is used as a context manager:

```python
async def packet_reader(sim, stream):
    while True:
        # Wait until stream has valid data.
        await sim.tick().until(stream.valid)

        # Ensure simulation doesn't end in the middle of a packet.
        async with sim.critical():
            packet = await stream.read_packet()
            print('Received packet:', packet.hex(' '))
```

When a trigger object is awaited, it'll return the value(s) of the trigger(s), and it can also be used as an async generator to repeatedly await the same trigger.
Multiple triggers can be combined.
Consider the following examples:

Combinational adder as a process:
```python
a = Signal(); b = Signal(); o = Signal()
async def adder(sim):
    async for a_val, b_val in sim.changed(a, b):
        await sim.set(o, a_val + b_val)
sim.add_process(adder)
```

DDR IO buffer as a process:
```python
o = Signal(2); pin = Signal()
async def ddr_buffer(sim):
    while True: # could be extended to pre-capture next `o` on posedge
        await sim.negedge()
        await sim.set(pin, o[0])
        await sim.posedge()
        await sim.set(pin, o[1])
sim.add_process(ddr_buffer)
```

Flop with configurable edge reset and posedge clock as a process:
```python
clk = Signal(); rst = Signal(); d = Signal(); q = Signal()
def dff(rst_edge):
    async def process(sim):
        async for clk_hit, rst_hit in sim.posedge(clk).edge(rst, rst_edge):
            await sim.set(q, 0 if rst_hit else await sim.get(d))
    return process
sim.add_process(dff(rst_edge=0))
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The following `Simulator` methods have their signatures updated:

* `add_process(process)`
* `add_testbench(process, *, background=False)`

The new optional named argument `background` registers the testbench as a background process when true.

Both methods are updated to accept an async function passed as `process`.
The async function must accept an argument `sim`, which will be passed a simulator context.
(Argument name is just convention, will be passed positionally.)

The simulator context have the following methods:
- `get(expr: Value) -> int`
- `get(expr: ValueCastable) -> any`
  - Returns the value of `expr` when awaited.
    When `expr` is a value-castable, the value will be converted through `.from_bits()`.
- `set(expr: Value, value: ConstLike)`
- `set(expr: ValueCastable, value: any)`
  - Set `expr` to `value` when awaited.
    When `expr` is a value-castable, the value will be converted through `.const()`.
- `memory_read(instance: MemoryInstance, address: int)`
  - Read the value from `address` in `instance` when awaited.
- `memory_write(instance: MemoryInstance, address: int, value: int, mask:int = None)`
  - Write `value` to `address` in `instance` when awaited. If `mask` is given, only the corresponding bits are written.
- `delay(interval: float)`
  - Create a trigger object for advancing simulation by `interval` seconds.
- `tick(domain="sync", *, context=None)`
  - Create a trigger object for advancing simulation until the next active edge of the `domain` clock.
    When an elaboratable is passed to `context`, `domain` will be resolved from its perspective.
  - If `domain` is asynchronously reset while this is being awaited, `amaranth.sim.AsyncReset` is raised.
- `changed(*signals)`
  - Create a trigger object for advancing simulation until any signal in `signals` changes.
- `edge(signal, value)`
  - Create a trigger object for advancing simulation until `signal` is changed to `value`.
    `signal` must be a 1-bit signal or a 1-bit slice of a signal.
- `posedge(signal)`
- `negedge(signal)`
  - Aliases for `edge(signal, 1)` and `edge(signal, 0)` respectively.
- `critical()`
  - Context manager.
    If the current process is a background process, `async with sim.critical():` makes it a non-background process for the duration of the statement.

A trigger object has the following methods:
- `__await__()`
  - Advance simulation and return the value(s) of the trigger(s).
    - `delay`, `tick` and `edge` triggers return `True` when they are hit, otherwise `False`.
    - `changed` triggers return the current value of the signals they are monitoring.
- `__aiter__()`
  - Return an async generator that is equivalent to repeatedly awaiting the trigger object in an infinite loop.
- `delay(interval: float)`
- `tick(domain="sync", *, context=None)`
- `changed(*signals)`
- `edge(signal, value)`
- `posedge(signal)`
- `negedge(signal)`
  - Create a new trigger object by copying the current object and appending another trigger.
- `until(condition)`
  - Repeat the trigger until `condition` is true.
    `condition` is an arbitrary Amaranth expression.
    If `condition` is initially true, `await` will return immediately without advancing simulation.
    The return value is an unspecified awaitable with `await` as the only defined operation.

`Tick()`, `Delay()`, `Active()` and `Passive()` as well as the ability to pass generator coroutines as `process` are deprecated and removed in a future version.

## Drawbacks
[drawbacks]: #drawbacks

- Increase in API surface area and complexity.
- Churn.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Do nothing. Keep the existing interface, add `Changed()` alongside `Delay()` and `Tick()`, use `yield from` when calling functions.

## Prior art
[prior-art]: #prior-art

Other python libraries like [cocotb](https://docs.cocotb.org/en/stable/coroutines.html) that originally used generator based coroutines have also moved to `async`/`await` style coroutines.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Bikeshed all the names.
  - (@whitequark) Should we go for `posedge` (Verilog convention) or `pos_edge` (Python convention)?

## Future possibilities
[future-possibilities]: #future-possibilities

- Add simulation helpers in the manner of `.send()` and `.recv()` to standard interfaces where it makes sense.
- There is a desire for a `sim.time()` method that returns the current simulation time, but it needs a suitable return type to represent seconds with femtosecond resolution and that is out of the scope for this RFC.
- We ought to have a way to skip a given number of triggers, so that we can tell the simulation engine to e.g. «advance by n cycles».
