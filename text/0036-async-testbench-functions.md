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
async def stream_recv(sim, stream):
    sim.set(stream.ready, 1)
    value = await sim.tick().sample(stream.data).until(stream.valid)
    sim.set(stream.ready, 0)
    return value

async def stream_send(sim, stream, value):
    sim.set(stream.data, value)
    sim.set(stream.valid, 1)
    await sim.tick().until(stream.ready)
    sim.set(stream.valid, 0)
```

`sim.get()` and `sim.set()` replaces the existing operations `yield signal` and `yield signal.eq()` respectively.

`sim.tick()` replaces the existing `Tick()`. It returns a trigger object that either can be awaited directly, or made conditional through `.until()`. Values of signals can be captured using `.sample()`, which is used to sample the interface members at the active edge of the clock. This approach makes these functions robust in presence of combinational feedback or concurrent use in multiple testbench processes.

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

`sim.tick()` return a domain trigger object that can be made conditional through `.until()` or repeated through `.repeat()`. Arbitrary expressions may be sampled at the active edge of the domain clock using `.sample()`.

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
        sim.set(o, a_val + b_val)
sim.add_process(adder)
```

DDR IO buffer as a process:
```python
clk = Signal(); o = Signal(2); pin = Signal()
async def ddr_buffer(sim):
    while True: # could be extended to pre-capture next `o` on posedge
        await sim.negedge(clk)
        sim.set(pin, o[0])
        await sim.posedge(clk)
        sim.set(pin, o[1])
sim.add_process(ddr_buffer)
```

Flop with configurable edge reset and posedge clock as a process:
```python
clk = Signal(); rst = Signal(); d = Signal(); q = Signal()
def dff(rst_edge):
    async def process(sim):
        async for clk_hit, rst_hit in sim.posedge(clk).edge(rst, rst_edge):
            sim.set(q, 0 if rst_hit else d)
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

The usage model of the two kinds of processes are:
- Processes are added with `add_process()` for the sole purpose of simulating a part of the netlist with behavioral Python code.
  - Typically such a process will consist of a top-level `async for values in sim.tick().sample(...):` or `async for values in sim.changed(...)`, but this is not a requirement.
  - Such processes may only wait on signals, via `sim.tick()`, `sim.changed()`, and `sim.edge()`. They cannot advance simulation time via `sim.delay()`.
  - In these processes, `sim.get()` is not available; values of signals may only be obtained by awaiting on triggers.
    `sim.set(x, y)` may be used to propagate the value of `y` without reading it.
  - The function passed to `add_process()` must be idempotent: applying it multiple times to the same simulation state and with same local variable values must produce the same effect each time.
    Provided that, the outcome of running such a process is deterministic regardless of the order of their execution.
- Processes are added with `add_testbench()` for any other purpose, including but not limited to: providing a stimulus, performing I/O, displaying state, asserting outcomes, and so on.
  - Such a process may be a simple linear function, use a top-level loop, or have arbitrarily complex structure.
  - Such processes may wait on signals as well as advance simulation time.
  - In these processes, `sim.get(x)` is available and returns the most current value of `x` (after all pending combinatorial propagation finishes).
  - The function passed to `add_testbench()` may have arbitrary side effects.
    These processes are scheduled in an unspecified order that may not be deterministic, and no mechanisms are provided to recover determinism of outcomes.
  - When waiting on signals, e.g. via `sim.tick()`, the requested expressions are sampled before the processes added with `add_process()` and RTL processes perform combinatorial propagation. However, execution continues only after all pending combinatorial propagation finishes.

The following concurrency guarantees are provided:
- Async processes registered with `add_testbench` may be preempted by:
  - Any other process when calling `await ...`.
  - A process registered with `add_process` (or an RTL process) when calling `sim.set()` or `sim.memory_write()`. In this case, control is returned to the same testbench after combinational settling.
- Async processes registered with `add_process` may be preempted by:
  - Any other process when calling `await ...`.
- Legacy processes follow the same rules as async processes, with the exception of:
  - A legacy process may not be preempted when calling `yield x:ValueLike` or `yield x:Assign`.
- Once running, a process continues to execute until it terminates or is preempted.

The new optional named argument `background` registers the testbench as a background process when true.
Processes created through `add_process` are always registered as background processes (except when registering legacy non-async generator functions).

The simulator context has the following methods:
- `get(expr: Value) -> int`
- `get(expr: ValueCastable) -> any`
  - Returns the value of `expr`.
    When `expr` is a value-castable, and its `shape()` is a `ShapeCastable`, the value will be converted through the shape's `.from_bits()`.
    Otherwise, a plain integer is returned.
    This function is not available in processes created through `add_process`.
- `set(expr: Value, value: ConstLike)`
- `set(expr: ValueCastable, value: any)`
  - Set `expr` to `value`.
    When `expr` is a value-castable, and its `shape()` is a `ShapeCastable`, the value will be converted through the shape's `.const()`.
    Otherwise, it must be a const-castable `ValueLike`.
    When used in a process created through `add_testbench`, it may execute RTL processes and processes created through `add_process`.
- `memory_read(instance: MemoryIdentity, address: int)`
  - Read the value from `address` in `instance`.
    This function is not available in processes created through `add_process`.
- `memory_write(instance: MemoryIdentity, address: int, value: int, mask:int = None)`
  - Write `value` to `address` in `instance`. If `mask` is given, only the corresponding bits are written.
    Like `MemoryInstance`, these two functions are an internal interface that will be usually only used via `lib.Memory`.
    When used in a process created through `add_testbench`, it may execute RTL processes and processes created through `add_process`.
    It comes without a stability guarantee.
- `tick(domain="sync", *, context=None)`
  - Create a domain trigger object for advancing simulation until the next active edge of the `domain` clock.
    When an elaboratable is passed to `context`, `domain` will be resolved from its perspective.
  - If `domain` is asynchronously reset while this is being awaited, `amaranth.sim.AsyncReset` is raised.
- `delay(interval: float)`
  - Create a combinable trigger object for advancing simulation by `interval` seconds.
    This function is not available in processes created through `add_process`.
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
  - Advance simulation and return the value(s) of the sampled expression(s). Values are returned in the same order as the expressions were added.
- `__aiter__()`
  - Return an async generator that is equivalent to repeatedly awaiting the trigger object in an infinite loop.
  - The async generator yields value(s) of the sampled expression(s).
- `sample(*expressions)`
  - Create a new trigger object by copying the current object and appending the expressions to be sampled.
- `until(condition)`
  - Repeat the trigger until `condition` is true.
    `condition` is an arbitrary Amaranth expression.
    The return value is an unspecified awaitable with `await` as the only defined operation.
    It is only awaitable once and returns the value(s) of the sampled expression(s) at the last time the trigger was repeated.
  - Example implementation (without error checking):
    ```python
    async def until(self, condition):
        while True:
            *values, done = await self.sample(condition)
            if done:
              return values
    ```
- `repeat(times: int)`
  - Repeat the trigger `times` times.
    Valid values are `times > 0`.
    The return value is an unspecified awaitable with `await` as the only defined operation.
    It is only awaitable once and returns the value(s) of the sampled expression(s) at the last time the trigger was repeated.
  - Example implementation (without error checking):
    ```python
    async def repeat(self, times):
        values = None
        for _ in range(times):
            values = await self
        return values
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
  - The async generator yields value(s) of the trigger(s).
- `delay(interval: float)`
- `changed(*signals)`
- `edge(signal, value)`
- `posedge(signal)`
- `negedge(signal)`
  - Create a new trigger object by copying the current object and appending another trigger.
  - Awaiting the returned trigger object pauses the process until the first of the combined triggers hit, i.e. the triggers are combined using OR semantics.

`Tick()`, `Delay()`, `Active()` and `Passive()` as well as the ability to pass generator coroutines as `process` are deprecated and removed in a future version.

## Drawbacks
[drawbacks]: #drawbacks

- Increase in API surface area and complexity.
- Churn.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

`sim.get()` is not available in processes created with `add_process()` to simplify the user interface and eliminate the possibility of misusing a helper function by calling it from the wrong type of process.
- Most helper functions will be implemented using `await sim.tick().sample(...)`, mirroring the structure of the gateware they are driving. These functions may be safely called from either processes added with `add_testbench()` or with `add_process()` since the semantics of `await sim.tick()` is the same between them.
- Some helper functions will be using `sim.get(val)`, and will only be callable from processes added with `add_testbench()`, raising an error otherwise. In the legacy interface, the semantics of `yield val` changes depending on the type of the process, potentially leading to extremely confusing behavior. This is not possible in the async interface.

Alternatives:
- Do nothing. Keep the existing interface, add `Changed()` alongside `Delay()` and `Tick()`, expand `Tick()` to add sampling, use `yield from` when calling functions.

## Prior art
[prior-art]: #prior-art

Other python libraries like [cocotb](https://docs.cocotb.org/en/stable/coroutines.html) that originally used generator based coroutines have also moved to `async`/`await` style coroutines.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is there really a need to ban `sim.delay()` from processes added with `add_process()`?
    - The value of `add_process()` is in ensuring that multiple processes waiting on the same trigger will modify simulation state deterministically no matter which order they run. Multiple processes waiting on a specific point in time using `sim.delay()` does not appear a common case.
    - `sim.delay()` in processes added with `add_process()` may unduly complicate implementation, since timeline advancement then can raise readiness of two kinds of processes instead of one. It is also likely to cause issues with CXXRTL integration.
    - `sim.delay()` in processes added with `add_process()` is useful to implement delay and phase shift blocks. However, these can be implemented in processes added with `add_testbench()` with no loss of functionality, as such blocks do not need delta cycle accurate synchronization with others on the same trigger.

## Future possibilities
[future-possibilities]: #future-possibilities

- Add simulation helper methods to standard interfaces where it makes sense.
  - This includes `lib.memory.Memory`.
- There is a desire for a `sim.time()` method that returns the current simulation time, but it needs a suitable return type to represent seconds with femtosecond resolution and that is out of the scope for this RFC.
