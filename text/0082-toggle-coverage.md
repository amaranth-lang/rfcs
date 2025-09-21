- Start Date: 2025-08-27
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Toggle Coverage

## Summary
[summary]: #summary

Introduce toggle coverage, a feature that tracks the number of individual signal transitions from 0→1 and 1→0. It generates human-readable reports that list each signal's bitwise toggle by its full hierarchical path.

## Motivation
[motivation]: #motivation

Toggle coverage is one of the fundamental coverage metrics in hardware verification. It helps identify signals that remain constant during tests, revealing insufficient stimulus, missing corner cases, or unreachable logic. It is also used in activity/power analysis and is expected in most coverage flows. 
Today, Amaranth users who need toggle coverage must either post-process VCDs or rely on simulator-specific options (e.g., Verilator’s --coverage-toggle) with no consistent integration. Adding toggle coverage to Amaranth provides a unified, easy-to-use API across simulators and improves test quality by highlighting dead or under-exercised parts of designs.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation
Toggle coverage is currently implemented for Amaranth’s built-in Python simulator, which is most useful for lightweight unit tests and quick functional checks. In this environment, users can attach an observer to collect per-bit toggle counts without modifying their testbench code.

##### Example: UART Peripheral with Toggle Coverage

A user writes a testbench for a simple `UARTPeripheral`. By attaching a `ToggleCoverageObserver` to the simulator, they can collect per-bit transition counts for all relevant signals.

```python
from amaranth.sim import Simulator
from amaranth.sim.coverage import ToggleCoverageObserver
from tests.test_utils import get_signal_full_paths
from chipflow_digital_ip.io import UARTPeripheral

dut = UARTPeripheral(init_divisor=8, addr_width=5)
sim = Simulator(dut)

# Attach toggle coverage
design = sim._engine._design
signal_path_map = get_signal_full_paths(design)
toggle_cov = ToggleCoverageObserver(sim._engine.state, signal_path_map=signal_path_map)
sim._engine.add_observer(toggle_cov)

# Add clock, processes, and testbench (omitted for brevity)
sim.add_clock(1e-6)
# sim.add_process(uart_rx_proc)
# sim.add_process(uart_tx_proc)
# sim.add_testbench(testbench)

with sim.write_vcd("uart.vcd"):
    sim.run()
toggle_cov.close(0)
```

When the simulation completes, Amaranth prints a toggle coverage report:

```
=== Toggle Coverage Report ===
bench/top/_uart/rx/bridge/mux/bus__r_stb:
Bit 0: 0→1=2, 1→0=3
bench/top/_uart/rx/bridge/Status/element__r_stb:
Bit 0: 0→1=1, 1→0=2
bench/top/_phy/rx/lower/fsm_state:
Bit 0: 0→1=0, 1→0=1
Bit 1: 0→1=1, 1→0=1
bench/top/_phy/rx/lower/timer:
Bit 0: 0→1=39, 1→0=38
Bit 1: 0→1=20, 1→0=19
Bit 2: 0→1=10, 1→0=10
Bit 3: 0→1=0, 1→0=0
Bit 4: 0→1=0, 1→0=0
Bit 5: 0→1=0, 1→0=0
...
```
Each signal is listed with its full hierarchical path (e.g. `bench/top/_uart/rx/bridge/Status/element__r_stb`). Each bit records how many times it transitioned from `0→1` and from `1→0`.  

From a programmer’s perspective, toggle coverage should be thought of as a completeness check: it does not prove correctness, but it highlights which parts of the design have remained idle across the test suite. This makes it easier to spot insufficient test stimulus or unreachable conditions.  

For new users, the feature provides an approachable structural coverage metric that requires no special setup beyond enabling the collector. Reports identify under-exercised signals by their hierarchical paths, making it straightforward for developers to locate the relevant logic in source code. As testbenches evolve, coverage output offers immediate feedback on whether changes have reduced or improved test completeness.  

Finally, no deprecations or migration steps are required: toggle coverage is additive, opt-in, and does not affect existing simulation behavior.


## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation
Toggle coverage is implemented as an observer attached to the simulation engine. For each signal, it stores the previous value, then on each tick it compares the old and new values bit by bit. If a bit changes from 0→1 or 1→0, it increments the corresponding counter. Each counter is stored in a dictionary keyed by signal identifier and bit index. Signal names are resolved into full hierarchical paths, ensuring results can be traced directly back to source code.

#### API: `ToggleCoverageObserver`

```python
class ToggleCoverageObserver(Observer):
    def __init__(self, state, signal_path_map: Optional[Dict[int, str]] = None, **kwargs): ...
    def update_signal(self, timestamp: int, signal: Signal) -> None: ...
    def update_memory(self, timestamp: int, memory, addr) -> None: ...
    def get_results(self) -> Dict[str, Dict[int, Dict[ToggleDirection, int]]]: ...
    def close(self, timestamp: int) -> None: ...
```
#### `ToggleDirection` Enum
Represents the direction of a signal toggle:  
- **`ZERO_TO_ONE`** — transition from logic 0 → 1  
- **`ONE_TO_ZERO`** — transition from logic 1 → 0  

#### Constructor

- **`state`**: simulation state object, used to query signal values.  
- **`signal_path_map`**: optional dict mapping `id(signal)` → hierarchical path string.  
- **`**kwargs`**: forwarded to the base `Observer`.  

#### Fields

- **`_prev_values: Dict[int, int]`** — last observed values, keyed by signal ID.  
- **`_toggles: Dict[int, Dict[int, Dict[ToggleDirection, int]]]`** — per-signal, per-bit toggle counters.  
- **`_signal_names: Dict[int, str]`** — human-readable names for reporting.  
- **`_signal_path_map: Dict[int, str]`** — mapping of signal IDs to full paths.  

#### Methods

- **`update_signal(timestamp, signal)`**  
  Updates toggle counters by comparing current vs. previous values, bit by bit.  

- **`update_memory(timestamp, memory, addr)`**  
  Currently a placeholder with no effect.  

- **`get_results()`**    Returns structured results:  
  ```python
  {
      "signal_name": {
          bit_index: {
              ToggleDirection.ZERO_TO_ONE: count,
              ToggleDirection.ONE_TO_ZERO: count
          }
      }
  }
- **`close(timestamp)`** Prints a formatted toggle coverage report to stdout.
#### Helper functions 
##### `collect_all_signals(obj)` Helper
Recursively traverses a design object to collect all `Signal` instances.  
- Skips private attributes (those starting with `_`).  
- Follows `submodules` if present (supports both dicts and iterables).  
- Returns: a list of discovered `Signal` objects.  

##### `get_signal_full_paths(design)` Helper
Builds hierarchical names for all signals in a design.  
- Iterates through `design.fragments` and their signal names.  
- Returns: a dictionary mapping `id(signal)` → full hierarchical path string.  

#### Example Workflow

Using the `UARTPeripheral` testbench (see Guide-level explanation):

1. The user attaches a `ToggleCoverageObserver` to the simulator.  
2. During execution, each clock tick updates the observer with new signal values.  
3. On a transition (e.g. `prev=0, curr=1`), the appropriate counter is incremented.  
4. At the end of simulation, `toggle_cov.close()` produces a report.

#### Corner Cases
- **First sample of a signal**: when a signal is seen for the first time, its current value is stored but no toggle is recorded, since there is no previous value to compare against.  
- **Bitwise handling**: multi-bit signals are tracked per bit; if only some bits change, only those bit counters are incremented.  
- **No toggles**: signals that never change during simulation remain in the report with zero counts for both directions.   

#### Interaction with Other Features

- Works alongside other coverage observers (e.g. statement, block coverage).  
- The observer is read-only and has no side effects on simulation behavior.
- Backward compatible: the feature is strictly opt-in. Existing designs and testbenches require no modification and continue to run unchanged.

## Drawbacks
[drawbacks]: #drawbacks
- **Increased complexity**: Introducing another coverage type adds new APIs, configuration options, and reporting paths that need to be maintained in Amaranth.  
- **Simulation overhead**: Recording per-bit transitions adds bookkeeping at each tick, which may slow down large simulations if many signals are tracked.  
- **Performance trade-offs**: Even with filtering, toggle coverage introduces extra work each cycle. Users who do not need it may see slower simulations if they enable it unnecessarily.  

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design is the best choice because it builds on Amaranth’s existing simulator observer interface, keeping coverage separate from design and testbench code. By counting bit-level 0→1 and 1→0 transitions, it provides exactly the information verification engineers expect from toggle coverage, while staying lightweight and easy to extend. Unlike external-only solutions (e.g. relying solely on Verilator flags or post-processing VCDs), this approach gives Amaranth users a consistent API across simulators, immediate feedback during testing, and reports tied directly to hierarchical signal paths in their designs. Having toggle coverage in the Python simulator helps new users understand the concept without needing Verilator or extra tools.

#### Alternatives considered
- **Post-processing waveforms (VCD/FSDB) to compute toggles**  
  *Pros:* simulator-agnostic; no runtime overhead in the simulator.  
  *Cons:* slow and memory-heavy on large traces; direction counts must be inferred; requires extra tooling and a second pass.  
  *Why not:* increases friction and resource usage; less immediate feedback.

- **Simulator-specific coverage only (e.g., “just use Verilator flags”)**  
  *Pros:* leverages mature backend features; high performance for big designs.  
  *Cons:* fractured user experience; different flags, formats, and reports per backend; difficult to teach and document.  
  *Why not:* Amaranth aims for consistent tooling and ease of learning, but today there is no unified API across simulators. This makes toggle coverage harder to use for both new learners and advanced users, since it requires backend-specific commands and extra tooling.

- **Functional coverage libraries**
  *Pros:* expressive, user-defined coverage points; great for protocols, corner cases, and cross-conditions.
  *Cons:* they don’t automatically tell you if a particular signal or bit ever changed state. Writing functional coverage requires extra effort to define bins and events - higher authoring cost. 
  *Why not:* functional coverage complements, but cannot substitute toggle coverage. Toggle coverage ensures structural activity (bits move at least once), while functional coverage ensures behavioral activity (scenarios were exercised).

- **Inline instrumentation (manually adding counters into testbench, using macros/ library utilities)**  
  *Pros:* no simulator support required; explicit in tests.  
  *Cons:* pollutes design/testbench with counters and boilerplate; easy to miss signals; harder to maintain.  
  *Why not:* observer does nto change the DUT or clutter test code. This separation is cleaner, less error-prone, and easier to maintain.

### Impact of not doing this
- **Fragmented workflows**: Without a unified toggle coverage feature, users are stuck with a mix of ad-hoc methods (e.g. post-process VCD files, Verilator flags). This results in inconsistent behavior across simulators, more effort for users, and more support questions for maintainers.
- **Lower test effectiveness**: Missing an essential structural metric makes it harder to detect unexercised logic early.  

Toggle coverage could technically be implemented as a standalone library, providing an observer and reporting scripts outside of Amaranth itself. However, this approach is suboptimal: without upstream support, such a library would rely on unstable simulator hooks, invent its own naming schemes and formats, and remain disconnected from Amaranth’s documentation and tutorials. By integrating toggle coverage directly, Amaranth can guarantee consistent APIs across backends, stable hierarchical naming, and a unified reporting surface. 

## Prior art
[prior-art]: #prior-art

Toggle coverage is a long-established feature in hardware simulators. Commercial tools such as Questa/ModelSim, Xcelium, and VCS provide it alongside line, branch, and FSM coverage. They record results into UCIS databases, which makes it easy to merge results from different runs and display them in coverage dashboards. These tools are mature and polished, with graphical interfaces and robust reporting, but they come with downsides: they are proprietary, require expensive licenses, and each vendor uses its own command lines and formats. In the open-source space, Verilator includes toggle coverage counters and utilities for post-processing reports. This is fast and accessible, but the options and report formats are tool-specific, and users need to learn Verilator’s workflow separately.

Other open-source flows compute toggle counts by post-processing waveforms such as VCD files. This has the advantage of working with any simulator, but it is often slow and memory-intensive on large designs, and it does not provide a unified API. Functional coverage libraries, such as SystemVerilog covergroups or Python packages, are widely used for protocol scenarios. These are powerful and flexible, but they answer different questions: they check if a protocol behaved as expected, not whether every bit in the design ever toggled. They complement toggle coverage but do not replace it.

There is no direct equivalent of toggle coverage in software ecosystems. Languages like C, Java, or Python rely on statement, branch, or path coverage (via tools like gcov, LCOV, or JaCoCo). Software does not have per-bit signal activity, so toggle coverage is a hardware-specific structural metric.

Amaranth takes a slightly different approach. Its goal is to provide consistent tooling and an approachable learning experience. By integrating toggle coverage directly, Amaranth can avoid backend-specific flags or offline scripts and instead present a single, unified API. Reports would be tied to hierarchical signal paths and work consistently across simulators, while remaining optional and non-intrusive to user RTL or testbenches.

## Unresolved questions
[unresolved-questions]: #unresolved-questions
- **Output formats.** Is a plain text report sufficient for the first version, or should JSON export be included? Should UCIS/LCOV output be in scope for this RFC or deferred to later proposals?  
- **Signal filtering.** Should include/exclude lists and a maximum bit-width cap be defined in this RFC, or added as extensions after the basic observer is merged?  
- **Interaction with other coverage types.** Since work is also underway on statement and block coverage, should toggle coverage eventually share a common reporting interface with these metrics?
- **Scope of implementation.** Should this RFC cover only the Python simulator (current implementation) or also include Verilator integration? If not, how should future work on Verilator support be tracked?  


## Future possibilities
[future-possibilities]: #future-possibilities

The most natural next step is to integrate toggle coverage with other coverage metrics currently under development, such as statement, block, assertion, unreachable code, and expression coverage. Toggle coverage will evenually share a common reporting interface with these other metrics. This would allow users to collect, merge, and visualize different coverage types through a single entry point, making Amaranth’s coverage ecosystem more cohesive.

Toggle coverage could also participate in hierarchical aggregation, where reports summarize coverage per module, instance, or subsystem rather than just per signal bit.

Over time, toggle coverage might evolve into a more configurable feature. Examples include filtering (include/exclude lists, maximum bus width), or selective depth of hierarchy in reports.

Ultimately, toggle coverage can serve as a foundation for more advanced metrics. While the initial implementation focuses on simple per-bit transitions, future work can unify it with broader coverage reporting, add richer configuration options, and extend support across backends. This evolution would position toggle coverage not only as a basic completeness check but also as an integral part of a comprehensive verificatiosn framework.