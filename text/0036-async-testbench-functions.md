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

~~Having `.get()` and `.set()` methods provides a convenient way for value castables to implement these in a type-specific manner.~~

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
In addition, `sim.changed()` is introduced that allows creating triggers from arbitrary signals.
These all return a trigger object that can be made conditional through `.until()`.

`Active()` and `Passive()` are replaced by an `passive=False` keyword argument to `.add_process()` and `.add_testbench()`.
To mark a passive testbench temporarily active, `sim.active()` is introduced, which is used as a context manager:

```python
async def packet_reader(sim, stream):
    while True
        # Wait until stream has valid data.
        await sim.tick().until(stream.valid)

        # Go active to ensure simulation doesn't end in the middle of a packet.
        async with sim.active():
            packet = await stream.read_packet()
            print('Received packet:', packet.hex(' '))
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The following `Simulator` methods have their signatures updated:

* `add_process(process, *, passive=False)`
* `add_testbench(process, *, passive=False)`

The new optional named argument `passive` registers the testbench as passive when true.

Both methods are updated to accept an async function passed as `process`.
The async function must accept a named argument `sim`, which will be passed a simulator context.

The simulator context have the following methods:
- `get(signal)`
  - Returns the value of `signal` when awaited.
    When `signal` is a value-castable, the value will be converted through `.from_bits()`. (Pending RFC #51)
- `set(signal, value)`
  - Set `signal` to `value` when awaited.
    When `signal` is a value-castable, the value will be converted through `.const()`.
- `delay(interval)`
  - Return a trigger object for advancing simulation by `interval` seconds.
- `tick(domain="sync", *, context=None)`
  - Return a trigger object for advancing simulation by one tick of `domain`.
    When an elaboratable is passed to `context`, `domain` will be resolved from its perspective.
- `changed(signal, value=None)`
  - Return a trigger object for advancing simulation until `signal` is changed to `value`. `None` is a wildcard and will trigger on any change.
- `active()`
  - Return a context manager that temporarily marks the testbench as active for the duration.
- `time()`
  - Return the current simulation time.

A trigger object has the following methods:
- `until(condition)`
  - Repeat the trigger until `condition` is true.
    `condition` is an arbitrary Amaranth expression.
    If `condition` is initially true, `await` will return immediately without advancing simulation.

~~`Value`, `data.View` and `enum.EnumView` have `.get()` and `.set()` methods added.~~

`Tick()`, `Delay()`, `Active()` and `Passive()` as well as the ability to pass generator coroutines as `process` are deprecated and removed in a future version.

## Drawbacks
[drawbacks]: #drawbacks

-  ~~Reserves two new names on `Value` and value castables~~
- Increase in API surface area and complexity.
- Churn.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Do nothing. Keep the existing interface, add `Changed()` alongside `Delay()` and `Tick()`, use `yield from` when calling functions.

- ~~Don't introduce `.get()` and `.set()`. Instead require a value castable and the return value of its `.eq()` to be awaitable so `await value` and `await value.eq(foo)` is possible.~~

## Prior art
[prior-art]: #prior-art

Other python libraries like [cocotb](https://docs.cocotb.org/en/stable/coroutines.html) that originally used generator based coroutines have also moved to `async`/`await` style coroutines.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- It should be possible to combine triggers, e.g. when we have a set of signals and are waiting for either of them to change.
  Simulating combinational logic with `add_process` would be one use case for this.
  Simulating sync logic with async reset could be another.
  What would be a good syntax to combine triggers?
- Is there any other functionality that's natural to have on the simulator context?
  - (@wanda-phi) `sim.memory_read(memory, address)`, `sim.memory_write(memory, address, value[, mask])`?
- Is there any other functionality that's natural to have on the trigger object?
  - Maybe a way to skip a given number of triggers? We still lack a way to say «advance by n cycles».
- Bikeshed all the names.
  - (@whitequark) We should consider different naming for `active`/`passive`.

## Future possibilities
[future-possibilities]: #future-possibilities

Add simulation helpers in the manner of `.send()` and `.recv()` to standard interfaces where it makes sense.
