- Start Date: 2024-02-05
- RFC PR: [amaranth-lang/rfcs#27](https://github.com/amaranth-lang/rfcs/pull/27)
- Amaranth Issue: [amaranth-lang/amaranth#1082](https://github.com/amaranth-lang/amaranth/issues/1082)

# Testbench processes for the simulator

## Summary
[summary]: #summary

The existing `Simulator.add_sync_process` method causes the process function to observe the design in a state before combinational settling, something that is actively unhelpful in testbenches. A new `Simulator.add_testbench` method will only return control to the process function after combinational settling.

## Motivation
[motivation]: #motivation

Consider the following code:

```python
from amaranth import *
from amaranth.sim import Simulator


class DUT(Elaboratable):
    def __init__(self):
        self.out  = Signal()
        self.outn = Signal()

    def elaborate(self, platform):
        m = Module()
        m.d.sync += self.outn.eq(~self.out)
        return m


dut = DUT()
def testbench():
    yield dut.out.eq(1)
    yield
    print((yield dut.out))
    print((yield dut.outn))

sim = Simulator(dut)
sim.add_clock(1e-6)
sim.add_sync_process(testbench)
sim.run()
```

This code prints:

```
1
1
```

While this result is sensible in a behavioral implementation of an elaboratable (where observing the state of the outputs of combinational cells before they transition to the new state is required for such an implementation to function as a drop-in replacement for a register transfer level one), it is not something a testbench should ever print; it clearly contradicts the netlist. Because there are no alternatives to using `add_sync_process`, testbenches (where such result is completely inappropriate) keep using it, and Amaranth designers are left to sprinkle `yield` over the testbenches until the result works.

In addition to the direct impact of this issue, it also prevents building reusable abstractions, including something as simple as `yield from fifo.read()`, since in order to work for back-to-back reads that would first have to `yield Settle()` to observe the updated value of `fifo.r_rdy`, which isn't appropriate for a function in the standard library as it changes the observable behavior (and thus breaks the abstraction).

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The code example above is rewritten as:

```python
dut = DUT()
def testbench():
    yield dut.out.eq(1)
    yield Tick()
    print((yield dut.out))
    print((yield dut.outn))

sim = Simulator(dut)
sim.add_clock(1e-6)
sim.add_testbench(testbench)
sim.run()
```

When run, it prints:

```
1
0
```

Existing testbenches can be ported to use `Simulator.add_testbench` by removing extraneous `yield` or `yield Settle()` calls (and, in some cases, shifting other `yield` calls around).

Reusable abstractions can be built by defining generator functions on interfaces or components.

### Guidance on simulator modalities

There are two main simulator modalities: `add_testbench` and `add_sync_process`. They have completely disjoint purposes:

- `add_testbench` is used for testing logic (asynchronous or synchronous). It is not used for behavioral replacement of synchronous logic.
- `add_sync_process` is used for behavioral replacement of synchronous logic. It is not for testing logic (except for legacy code), and a deprecation warning is shown when `yield Settle()` is executed in such a process.

Example of using `add_testbench` to test combinatorial logic:

```python
m = Module()
m.d.comb += a.eq(~b)

def testbench():
    yield b.eq(1)
    print((yield a)) # => 0

sim = Simulator(m)
# No clock is required
sim.add_testbench(testbench)
sim.run()
```

Example of using `add_testbench` to test synchronous logic:

```python
m = Module()
m.d.sync += a.eq(~b)

def testbench():
    yield b.eq(1)
    yield Tick() # same as Tick("sync")
    print((yield a)) # => 0

sim = Simulator(m)
sim.add_clock(1e-6)
sim.add_testbench(testbench)
sim.run()
```

Example of using `add_sync_process` to replace the flop above, and `add_testbench` to test the flop:

```python
m = Module()

def flop():
    while True:
        yield b.eq(~(yield a))
        yield Tick()

def testbench():
    yield b.eq(1)
    yield Tick() # same as Tick("sync")
    print((yield a)) # => 0

sim = Simulator(m)
sim.add_clock(1e-6)
sim.add_sync_process(flop)
sim.add_testbench(testbench)
sim.run()
```

### Why not replace `add_sync_process` with `add_testbench` entirely?

It is not possible to use `add_testbench` processes that drive signals in a race-free way. Consider this (behaviorally defined) circuit:

```python
x = Signal(reset=1)
y = Signal()

def proc_flop():
    yield Tick()
    yield y.eq(x)

def proc2():
    yield Tick()
    xv = yield x
    yv = yield y
    print(f"proc2 x={xv} y={yv}")

def proc3():
    yield Tick()
    yv = yield y
    xv = yield x
    print(f"proc3 x={xv} y={yv}")
```

If these processes are added using `add_testbench`, the output is:

```
proc3 x=1 y=0
proc2 x=1 y=1
```

If they are added using `add_sync_process`, the output is:

```
proc2 x=1 y=0
proc3 x=1 y=0
```

Clearly, if `proc2` and `proc3` are other flops in the circuit, perhaps performing a computation on `x` and `y`, they must be simulated using `add_sync_process`.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A new `Simulator.add_testbench(process)` is added. This function schedules `process` similarly to `add_process`, except that before returning control to the coroutine `process` it performs the equivalent of `yield Settle()`.

`add_process` and `Settle` are deprecated and removed in a future version.

`yield Tick()` is deprecated within `add_sync_process` and the ability to use it as well as `yield Settle()` is removed in a future version.

## Drawbacks
[drawbacks]: #drawbacks

- Churn.
- Testbench processes can race with each other, and it is not trivial to use multiple testbench processes in a design in a race-free way.
    - Processes using `Settle` can race as well.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The motivating issue has no known alternative resolution besides introducing this (or a very similar) API. The status quo has proved deeply unsatisfactory over many years, and the `add_testbench` process has been trialed in 2020 and found usable.

## Prior art
[prior-art]: #prior-art

Other simulators experience similar challenges with event scheduling. In Verilog, this is one of the reasons for the use of blocking assignment `=`. Where the decision of the scheduling primitive is left to the point of use (rather than the point of declaration, as proposed in this RFC) it leads to complexity in teaching the concept.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

In the standard library, `fifo.read()` and `fifo.write()` functions could be defined that aid in testing designs with FIFOs. Such functions will only work correctly within testbench processes.

As it is, every such helper function would have to take a `domain` argument, which can quickly get out of hand. We have `DomainRenamer` in the RTL sub-language and we may want to have something like that in the simulation sub-language. (@zyp)

A new `add_comb_process` function could be added, to replace combinatorial logic. This function would have to accept a list of all signals driven by the process, so that combinatorial loops could be detected. (The demand for this has not been high; as of right now, this is not possible anyway.)

The existing `add_sync_process` function could accept a list of all signals driven by the process. This could aid in error detection, especially as CXXRTL is integrated into the design, because if a simulator process is driving a signal at the same time as an RTL process, a silent race condition occurs.
