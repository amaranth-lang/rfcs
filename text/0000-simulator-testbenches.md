- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

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
    yield
    print((yield dut.out))
    print((yield dut.outn))

sim = Simulator(dut)
sim.add_clock(1e-6)
sim.add_testbench(testbench, domain="sync")
sim.run()
```

When run, it prints:

```
1
0
```

Existing testbenches can be ported to use `Simulator.add_testbench` by removing extraneous `yield` or `yield Settle()` calls.

Reusable abstractions can be built by defining generator functions on interfaces or components.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A new `Simulator.add_testbench(process, *, domain=None)` is added. This function schedules `process` similarly to `add_process`, except that before returning control to the coroutine `process` it performs the equivalent of `yield Settle()`. If `domain` is not `None`, then calling `yield` within `add_testbench` performs the equivalent of `yield Tick(domain)`.

`Settle` is deprecated and removed in a future version.

## Drawbacks
[drawbacks]: #drawbacks

Increase in API surface area and complexity. Churn.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The motivating issue has no known alternative resolution besides introducing this (or a very similar) API. The status quo has proved deeply unsatisfactory over many years, and the `add_testbench` process has been trialed in 2020 and found usable.

The `domain` argument of `add_testbench` could default to "sync", as for `add_sync_process`. Since a testbench does not inherently have a "default" domain (unlike a behavioral replacement for a register transfer level module, where `sync` is the default), this does not appear appropriate.

## Prior art
[prior-art]: #prior-art

Other simulators experience similar challenges with event scheduling. In Verilog, this is one of the reasons for the use of blocking assignment `=`. Where the decision of the scheduling primitive is left to the point of use (rather than the point of declaration, as proposed in this RFC) it leads to complexity in teaching the concept.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

In the standard library, `fifo.read()` and `fifo.write()` functions could be defined that aid in testing designs with FIFOs. Such functions will only work correctly within testbench processes.