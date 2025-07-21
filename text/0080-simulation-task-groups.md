- Start Date: (fill in with date at which the RFC is merged, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#80](https://github.com/amaranth-lang/rfcs/pull/80)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Simulation task groups

## Summary
[summary]: #summary

Add task groups to the simulator to allow parallel execution in a testbench.

## Motivation
[motivation]: #motivation

When testing a component, it's common to need to interact with multiple interfaces in parallel.
For instance, when testing a stream component, the output stream must typically be read in parallel with writing the input stream to avoid backpressure from the output stream to propagate back to the input stream and deadlock the simulation.
Currently this must be done by adding independent testbenches, which can be awkward to synchronize.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Typical testbenches for a stream component can currently look like this:
```python
test_vectors = [...]

async def input_testbench(ctx: SimulatorContext):
    for input in test_vectors:
        await send_packet(ctx, dut.i, input)

async def output_testbench(ctx: SimulatorContext):
    for input in test_vectors:
        output = await recv_packet(ctx, dut.o)
        assert output == expected_output(input)
```

With task groups, this can instead be written like:
```python
test_vectors = [...]

async def testbench(ctx: SimulatorContext):
    for input in test_vectors:
        async with ctx.group() as group:
            group.start(send_packet(ctx, dut.i, input))
            output = await recv_packet(ctx, dut.o)
            assert output == expected_output(input)
```

In a similar manner to background testbenches, it is also possible to add background tasks, for tasks that are not intended to run to completion.
This allows code like this:
```python
@asynccontextmanager
async def timeout(ctx, ticks, domain = 'sync'):
    async def task():
        await ctx.tick(domain).repeat(ticks) # Never returns if the task group ends before `ticks` have elapsed.
        raise TimeoutError()

    async with ctx.group() as group:
        group.start(task(ctx, ticks, domain), background = True)
        yield

async def testbench(ctx):
    async with timeout(ctx, 100):
        ... # Some operation expected to take less than 100 ticks

    async with timeout(ctx, 200):
        ... # Some other operation expected to take less than 200 ticks
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

`SimulatorContext` have the following methods added:
- `group() -> TaskGroup`
  - Create a new task group.
- `async gather(coros*) -> tuple`
  - Shorthand for creating a task group, starting all `coros`, letting the group run to completion and collecting the return values.
  - Example implementation:
    ```python
        async def gather(self, *coros):
            async with self.group() as group:
                tasks = [group.start(coro) for coro in coros]
            return tuple(task.result() for task in tasks)
    ```

`TaskGroup` is added with the following methods:
- `start(coro, *, background = False) -> Task`
  - Create and start a new task.
  - A background task can be temporarily made non-background with `ctx.critical()` like a regular testbench.
  - Raise an exception if called before `__aenter__()` or after `__aexit__()`.
- `async __aenter__() -> Self`
  - Return `self`.
  - Raise an exception if called multiple times.
- `async __aexit__(...)`
  - Wait for all non-background tasks to run to completion.
  - Once all non-background tasks are done, any remaining background tasks are dropped without returning or unwinding from their pending awaits.
  - Raise an exception if called multiple times or if called without calling `__aenter__()` first.

`Task` is added with the following methods:
- `result() -> Any`
  - Get the return value of a completed task.
  - Raise an exception if the task has not completed.

Exception propagation and task cancellation (beyond dropping background tasks when a group is done) are out of scope for this RFC.
If a task raises any unhandled exceptions, this immediately terminates the simulation and propagates out of the simulator as if raised from an independent testbench.

## Drawbacks
[drawbacks]: #drawbacks

- Increased simulator complexity.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Exception propagation and task cancellation was omitted from the scope of this RFC because it would significantly increase complexity, for limited benefit.
It is expected that the desired outcome of an unhandled exception in a task in most cases would be to terminate simulation and therefore don't need the ability for the parent testbench to catch it.

## Prior art
[prior-art]: #prior-art

The proposed API is modelled after `asyncio.TaskGroup` and `asyncio.gather()`.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- The usual bikeshedding of names.

## Future possibilities
[future-possibilities]: #future-possibilities

- A future RFC could add exception propagation and task cancellation.
- Context managers like the timeout example above could be added for common cases.
