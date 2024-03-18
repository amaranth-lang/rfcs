- Start Date: 2024-03-18
- RFC PR: [amaranth-lang/rfcs#36](https://github.com/amaranth-lang/rfcs/pull/36)
- Amaranth Issue: [amaranth-lang/amaranth#1213](https://github.com/amaranth-lang/amaranth/issues/1213)

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
We can then implement `stream_send()` and `stream_recv()` functions like this:

```python
@testbench_helper
async def stream_recv(sim, stream):
    await sim.set(stream.ready, 1)
    await sim.tick().until(stream.valid)

    value = await sim.get(stream.data)

    await sim.tick()
    await sim.set(stream.ready, 0)

    return value

@testbench_helper
async def stream_send(sim, stream, value):
    await sim.set(stream.data, value)

    await sim.set(stream.valid, 1)
    await sim.tick().until(stream.ready)

    await sim.tick()
    await sim.set(stream.valid, 0)
```

`await sim.get()` and `await sim.set()` replaces the existing operations `yield signal` and `yield signal.eq()` respectively.

`sim.tick()` replaces the existing `Tick()`. It returns a trigger object that either can be awaited directly, or made conditional through `.until()`.

The `testbench_helper` decorator indicates that this function is only designed to be called from testbench processes and will raise an exception if called elsewhere.

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
    await stream_send(sim, dut.input, rgb)
    yuv = await stream_recv(sim, dut.output)

    print(rgb, yuv)

async def testbench(sim):
    await test_rgb(sim, 0, 0, 0)
    await test_rgb(sim, 255, 0, 0)
    await test_rgb(sim, 0, 255, 0)
    await test_rgb(sim, 0, 0, 255)
    await test_rgb(sim, 255, 255, 255)
```

Since `stream_send()` and `stream_recv()` invokes `sim.get()` and `sim.set()` that in turn will invoke the appropriate value conversions for a value castable (here `data.View`), it is general enough to work for streams with arbitrary shapes.

`Tick()` and `Delay()` are replaced by `sim.tick()` and `sim.delay()` respectively.
In addition, `sim.changed()` and `sim.edge()` is introduced that allows creating triggers from arbitrary signals.

`sim.tick()` return a domain trigger object that can be made conditional through `.until()` or repeated through `.repeat()`.

`sim.delay()`, `sim.changed()` and `sim.edge()` return a combinable trigger object that can be used to add additional triggers.

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

When a combinable trigger object is awaited, it'll return the value(s) of the trigger(s), and it can also be used as an async generator to repeatedly await the same trigger.
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
clk = Signal(); o = Signal(2); pin = Signal()
async def ddr_buffer(sim):
    while True: # could be extended to pre-capture next `o` on posedge
        await sim.negedge(clk)
        await sim.set(pin, o[0])
        await sim.posedge(clk)
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

Both methods are updated to accept an async function passed as `process`.
The async function must accept an argument `sim`, which will be passed a simulator context.
(Argument name is just convention, will be passed positionally.)

The new optional named argument `background` registers the testbench as a background process when true.
Processes created through `add_process` are always registered as background processes (except when registering legacy non-async generator functions).

The simulator context has the following methods:
- `get(expr: Value) -> int`
- `get(expr: ValueCastable) -> any`
  - Returns the value of `expr` when awaited.
    When `expr` is a value-castable, and its `shape()` is a `ShapeCastable`, the value will be converted through the shape's `.from_bits()`.
    Otherwise, a plain integer is returned.
- `set(expr: Value, value: ConstLike)`
- `set(expr: ValueCastable, value: any)`
  - Set `expr` to `value` when awaited.
    When `expr` is a value-castable, and its `shape()` is a `ShapeCastable`, the value will be converted through the shape's `.const()`.
    Otherwise, it must be a const-castable `ValueLike`.
- `memory_read(instance: MemoryIdentity, address: int)`
  - Read the value from `address` in `instance` when awaited.
- `memory_write(instance: MemoryIdentity, address: int, value: int, mask:int = None)`
  - Write `value` to `address` in `instance` when awaited. If `mask` is given, only the corresponding bits are written.
    Like `MemoryInstance`, these two functions are an internal interface that will be usually only used via `lib.Memory`.
    It comes without a stability guarantee.
- `tick(domain="sync", *, context=None)`
  - Create a domain trigger object for advancing simulation until the next active edge of the `domain` clock.
    When an elaboratable is passed to `context`, `domain` will be resolved from its perspective.
  - If `domain` is asynchronously reset while this is being awaited, `amaranth.sim.AsyncReset` is raised.
- `delay(interval: float)`
  - Create a combinable trigger object for advancing simulation by `interval` seconds.
- `changed(*signals)`
  - Create a combinable trigger object for advancing simulation until any signal in `signals` changes.
- `edge(signal, value: int)`
  - Create a combinable trigger object for advancing simulation until `signal` is changed to `value`.
    `signal` must be a 1-bit signal or a 1-bit slice of a signal.
    Valid values for `value` are `1` for rising edge and `0` for falling edge.
- `posedge(signal)`
- `negedge(signal)`
  - Aliases for `edge(signal, 1)` and `edge(signal, 0)` respectively.
- `critical()`
  - Context manager.
    If the current process is a background process, `async with sim.critical():` makes it a non-background process for the duration of the statement.

A domain trigger object is immutable and has the following methods:
- `__await__()`
  - Advance simulation. No value is returned.
- `until(condition)`
  - Repeat the trigger until `condition` is true.
    `condition` is an arbitrary Amaranth expression.
    If `condition` is initially true, `await` will return immediately without advancing simulation.
    The return value is an unspecified awaitable with `await` as the only defined operation.
    It is only awaitable once and awaiting it returns no value.
  - Example implementation:
    ```python
    async def until(self, condition):
      while not await self._sim.get(condition):
        await self
    ```
- `repeat(times: int)`
  - Repeat the trigger `times` times.
    Valid values are `times >= 0`.
    The return value is an unspecified awaitable with `await` as the only defined operation.
    It is only awaitable once and awaiting it returns no value.
  - Example implementation:
    ```python
    async def repeat(self, times):
      for _ in range(times):
        await self
    ```

A combinable trigger object is immutable and has the following methods:
- `__await__()`
  - Advance simulation and return the value(s) of the trigger(s).
    - `delay` and `edge` triggers return `True` when they are hit, otherwise `False`.
    - `changed` triggers return the current value of the signals they are monitoring.
    - At least one of the triggers hit will be reflected in the return value.
      In case of multiple triggers occuring at the same time step, it is unspecified which of these will show up in the return value beyond “at least one”.
- `__aiter__()`
  - Return an async generator that is equivalent to repeatedly awaiting the trigger object in an infinite loop.
- `delay(interval: float)`
- `changed(*signals)`
- `edge(signal, value)`
- `posedge(signal)`
- `negedge(signal)`
  - Create a new trigger object by copying the current object and appending another trigger.
  - Awaiting the returned trigger object pauses the process until the first of the combined triggers hit, i.e. the triggers are combined using OR semantics.

To ensure testbench helper functions are only called from a testbench process, the `amaranth.sim.testbench_helper` decorator is added.
The function wrapper expects the first positional argument (or second, after `self` or `cls` if decorating a method/classmethod) to be a simulator context, and will raise `TypeError` if not.
If the function is called outside a testbench process, an exception will be raised.

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

None.

## Future possibilities
[future-possibilities]: #future-possibilities

- Add simulation helper methods to standard interfaces where it makes sense.
  - This includes `lib.memory.Memory`.
- There is a desire for a `sim.time()` method that returns the current simulation time, but it needs a suitable return type to represent seconds with femtosecond resolution and that is out of the scope for this RFC.
