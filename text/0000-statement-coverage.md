- Start Date: 2025-09-17
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Statement Coverage

## Summary
[summary]: #summary
Introduce statement coverage, a feature that tracks execution of individual HDL statements: assignments and conditional branches (switch / switch_case). The report indicates whether each statement was HIT or MISS (and how many times), along with the file name and line number of the Python source that generated the HDL.

## Motivation
[motivation]: #motivation
Statement coverage is a fundamental metric in hardware verification. It ensures that every line of code in the design has been exercised at least once during simulation. Uncovered statements highlight dead code, incomplete testbenches, or untested logic branches, guiding engineers to strengthen their stimulus and improve test completeness.
Without built-in support, Amaranth users must rely on simulator-specific options or manual inspection of waveforms/logs to approximate this metric. Integrating statement coverage directly into Amaranth provides a unified solution across simulators — aligning with standard verification practices and improving overall design quality.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation
Statement coverage in Amaranth’s Python simulator works by tagging each HDL statement with its type (assignment or conditional), a unique ID (hierarchical path, clock domain, numeric counter), and a human-readable name (file and line number plus a short summary such as `comb:sda = rhs or sync:switch_case(pattern)`), making results both machine-traceable and easy to interpret in reports. A coverage signal is inserted so that when the statement executes, the signal goes high for one cycle. The StatementCoverageObserver records these hits, and at the end results are merged into a report showing the amount of times it was HIT or MISS. 

##### Example: I²C Peripheral with Statement Coverage
Below is a minimal example showing how to enable statement coverage for an I²C Peripheral. Some code in the original testbench omitted for brevity. 

```python
from amaranth.sim import Simulator
from amaranth.sim.coverage import StatementCoverageObserver
from tests.test_utils import *
from chipflow_digital_ip.io import I2CPeripheral

dut = _I2CHarness()

# Build a simulator with statement-coverage instrumentation
sim, stmt_cov, stmt_info, fragment = mk_sim_with_stmtcov(dut, verbose=True)

# Add clock, processes, and testbench (omitted for brevity)
sim.add_clock(1e-6)
# sim.add_testbench(testbench)

with sim.write_vcd("i2c_stmtcov.vcd", "i2c_stmtcov.gtkw"):
    sim.run()

# Merge and emit a JSON summary across tests
merge_stmtcov(results, stmt_info)
emit_agg_summary("i2c_statement_cov.json", label="tests/test_i2c.py")
```

When the simulation finishes, Amaranth prints a statement coverage report:

```
[Statement coverage for tests/test_i2c.py] 315/378 = 83.3%

HIT (3x) | chipflow-digital-ip/tests/test_i2c.py:51 | comb:sda = (& (~ (sig i2c_pins__sda__oe)) (sig sda_i))
HIT (3x) | chipflow-digital-ip/tests/test_i2c.py:55 | comb:i2c_pins__scl__i = scl
HIT (8x) | .../amaranth_soc/csr/bus.py:564          | i2c/bridge/mux | comb:switch_case(('10000',))
HIT (3x) | .../amaranth_soc/csr/bus.py:564          | i2c/bridge/mux | sync:switch_case(('00001',))
...
```
HIT means that a statement executed at least once during the run. The observer not only marks it as covered but also keeps a counter, so you can see how many times that statement was executed in total (ex. (3x) means three times). MISS indicates code that never executed. This highlights untested branches, dead code, or insufficient stimulus in the testbench.
Each entry in the report includes the hierarchical path (e.g. i2c/bridge/mux) and the source location (src_loc), allowing you to quickly find the exact statement in the source code. For conditional logic, entries such as switch_case(('00001',)) show which case patterns were taken, while default cases are explicitly labeled.
There are some important trade-offs to keep in mind. Instrumentation adds a small amount of extra logic and can modestly increase simulation time, though this is typically negligible for unit tests. Hit counts represent statement execution events, not per-cycle residency; a signal toggling frequently does not increase the count unless the statement itself re-executes. Statement coverage only shows which code was exercised, not whether behavior was correct. MISS items should be treated as guidance to improve stimulus or revisit design intent. Finally, there is zero migration cost: statement coverage is opt-in and additive; enabling it does not affect existing simulation behavior.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation
Similar to toggle coverage, statement coverage also has an observer attached to the simulation engine. Each HDL statement is tagged during elaboration with a unique ID, a type (assignment or conditional), and a human-readable name. A coverage signal is injected so that when the statement executes, the signal goes high for one cycle. The StatementCoverageObserver records these pulses, and each execution increments that statement’s entry in the hit counter dictionary `AGG_STMT_HITS`. Finally, the unique IDs and names are mapped back to their file name and line number, and the report prints both the source location and the number of times each statement executed.

#### API: `StatementCoverageObserver`

```python
class StatementCoverageObserver(Observer):
    def __init__(self, coverage_signal_map, state, stmtid_to_info=None, **kwargs): ...
    def update_signal(self, timestamp: int, signal: Signal) -> None: ...
    def update_memory(self, timestamp: int, memory, addr) -> None: ...
    def get_results(self) -> Dict[StmtID, int]: ...
    def close(self, timestamp: int) -> None: ...

```

#### Constructor
- **`coverage_signal_map`**: maps id(signal) → statement ID.
- **`state`**: simulation state object, used to track signal values.  
- **`stmtid_to_info`**: optional dictionary mapping statement IDs → (name, type).

#### Fields
- **`_statement_hits: Dict[StmtID, int]`** — counters of how many times each statement executed..  
- **`coverage_signal_map: Dict[int, StmtID]`** — reverse lookup from signal ID to statement ID.  
- **`stmtid_to_info: Dict[StmtID, Tuple[str, str]]`** — statement metadata (name, type).  

#### Methods
- **`update_signal(timestamp, signal)`**  
  Checks whether a signal corresponds to a tagged statement; if so, increments its hit counter.
- **`update_memory(timestamp, memory, addr)`**  
  Currently a placeholder with no effect.  
- **`get_results()`**    
  Returns a dictionary of `{stmt_id: hit_count}`.
- **`close(timestamp)`** 
  Currently a no-op; results are typically collected via `get_results()` and merged externally.

#### Helper functions 
##### `tag_all_statements(fragment)`
Recursively traverses an Amaranth fragment, assigning each `Assign` and `Switch` statement a unique coverage ID, name, and type. The header of each switch statement is tagged (type = "switch") and each case inside switch is also tagged separately (type = "switch_case"). Every branch (e.g. case 0:, case 1:) gets its own unique ID and name, allowing coverage to distinguish which branches of the conditional were executed.
##### `insert_coverage_signals(fragment)`
Walks through every HDL statement in the design and inserts a special coverage signal assignment just before it. These coverage signals don’t affect the design logic; instead, they act as breadcrumbs. Each time a statement executes, its breadcrumb signal is written for one cycle, and the `StatementCoverageObserver` detects this write, increments the statement’s hit counter, and records the execution. This way, every Assign, Switch, and switch_case can be tracked precisely during simulation.
##### `mk_sim_with_stmtcov(dut, verbose=False)`
Elaborates the DUT into an Amaranth IR fragment, then calls `tag_all_statements(fragment)` and `insert_coverage_signals(fragment)`. Next, it builds a simulator from the instrumented fragment and attaches a `StatementCoverageObserver`, which maps those coverage signals back to statement IDs and increments hit counters whenever they fire. Function returns `(sim, stmt_cov, stmtid_to_info, fragment)`.
##### `merge_stmtcov(results, stmtid_to_info)`
Merges results from multiple simulations into global aggregators `AGG_STMT_HITS` and `AGG_STMT_INFO`. It first registers all statements, then accumulates statement hits. After merging all tests, the globals contain everything needed to compute coverage percentages and to print a final report with HIT (Nx) or MISS (0x) for every statement.
##### `emit_agg_summary(json_path, label)`
Emits a console report and a JSON file showing each statement’s source location, type, and hit count. It calculates the overall coverage percentage by dividing the number of statements executed at least once by the total number of instrumented statements.

#### Example Workflow
Using the `I²C peripheral` testbench (see Guide-level explanation):

1. The user builds a simulator with `mk_sim_with_stmtcov(dut)`.
2. The helper injects coverage signals into all `Assign` and `Switch` statements.
3. During simulation, when a statement executes, its signal pulses high for one cycle. The `StatementCoverageObserver` records the hit for that statement ID.  
4. After simulation, results are merged, and emit_agg_summary prints a HIT/MISS report and writes a JSON file.

#### Corner Cases
- **Default switch cases:**: labeled as `switch_case(default)` when patterns is `None` (catch-all case).
- **Unreachable code**: statements that never execute appear as MISS (0x) in the report.
- **Nested fragments**: recursion ensures statements inside submodules or cases are covered.   
- **Hit count granularity**: counter is incremented only when a statement actually executes, not for every cycle it remains active.

#### Interaction with Other Features

- Designed to work alongside other coverage observers (toggle, block, expression, assertion). Especially unreachable code coverage ... 
- The observer is read-only and does not affect simulation behavior.
- Backward compatible: opt-in and additive; existing designs and testbenches run unchanged if not enabled.

## Drawbacks
[drawbacks]: #drawbacks
- **Increased complexity**: Adding statement coverage introduces new helper functions, observers, and reporting logic, which increases maintenance cost in Amaranth.
- **Instrumentation overhead**: Each statement gets an extra coverage signal and assignment, slightly expanding the design and simulation workload.
- **Simulation slowdown**: The observer must check and count every coverage signal write. For designs with thousands of statements, this can noticeably impact simulation speed.
- **Limited granularity**: Coverage only records whether a statement executed and how many times; it does not capture path conditions, correctness of values, or deeper functional coverage.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives
This design builds on Amaranth’s existing simulator observer interface and works entirely at the IR (Fragment) level, so coverage stays separate from the DUT and testbench code. By tagging each Assign/Switch (and each switch_case) with a unique ID and injecting a one-cycle coverage pulse, it captures exactly what engineers expect from statement coverage: whether each statement executed and how many times. Results are mapped back to source locations, giving immediate, actionable feedback during testing with a consistent API across simulators. Keeping it in the Python simulator makes the concept approachable for new users without requiring external tools or backend-specific flags.

#### Alternatives considered
- **Post-processing waveforms (VCD/FSDB) to infer statement hits**  
  *Pros:* simulator-agnostic; no runtime overhead in the simulator.  
  *Cons:* inferring which statement executed from signals is error-prone; slow on large traces; needs extra tooling/passes.  
  *Why not:* fragile inference and higher friction; less immediate feedback.

- **Simulator-specific statement coverage (e.g., backend flags only)**  
  *Pros:* can leverage mature features in some tools; good performance on large designs.
  *Cons:* fragmented UX (different flags/formats), inconsistent reports/naming, harder to teach and document.
  *Why not:* Amaranth aims for a unified, portable coverage surface; backend-only solutions don’t provide that.

- **Functional coverage libraries**
  *Pros:* expressive, user-defined coverage points; great for protocols, corner cases, and cross-conditions.
  *Cons:* do not automatically show whether each HDL statement executed; authoring bins/events adds effort.
  *Why not:* functional coverage complements but does not replace statement coverage. We still need a structural baseline to expose dead/untouched code.

- **Inline instrumentation (manually adding counters into testbench, using macros/ library utilities)**  
  *Pros:* no simulator hooks required; explicit control.
  *Cons:* clutters code, easy to miss statements, harder to maintain, not standardized across projects.
  *Why not:* the observer approach keeps coverage out of the DUT, minimizes boilerplate, and standardizes reporting.

### Impact of not doing this
- **Fragmented workflows**: Users would bounce between ad-hoc scripts, backend flags, or waveform post-processing to approximate statement hits, leading to inconsistent results and more support burden.
- **Lower test effectiveness**: Without a built-in structural metric, unexecuted assignments/branches remain hidden longer, reducing confidence in test completeness.

Statement coverage could technically live as a standalone library, but that would rely on unstable hooks, ad-hoc naming, and disconnected documentation. Integrating directly with Amaranth guarantees stable IDs/names, consistent APIs, and a unified reporting surface—making coverage both reliable and easy to adopt.

## Prior art
[prior-art]: #prior-art
Statement coverage is one of the most established structural metrics in hardware verification. Commercial simulators such as Questa/ModelSim, Xcelium, and VCS provide statement, branch, and FSM coverage alongside toggle coverage. Results are usually stored in UCIS databases, which makes it easy to merge multiple runs and display progress in polished dashboards. These tools are mature, with graphical browsers and sophisticated reporting, but they are proprietary, expensive, and vendor-specific: each simulator has its own switches, report formats, and workflows.

In the open-source space, Verilator supports line and toggle coverage and can emit LCOV-compatible reports. This is fast and integrates well with existing software coverage tooling, but it is still simulator-specific, and users must learn Verilator’s flags and reporting pipeline separately. Other open flows attempt to infer statement coverage by post-processing waveforms (VCD/FSDB), but this is slow and memory-hungry for large designs, and results are fragile since waveforms only show signal values, not which HDL statement executed.

From the software side, statement coverage is a direct parallel to tools like gcov, LCOV, or JaCoCo, which measure whether each line or branch in C, C++, Java, or Python has executed. This makes the concept familiar to many engineers, but unlike software, hardware IRs often lower high-level if/else into switch/case, so structural coverage must map back carefully to source constructs.

Amaranth’s approach is to integrate statement coverage directly into its Python simulator observer framework, avoiding reliance on backend-specific flags or heavyweight post-processing. This provides a consistent API, immediate feedback during testing, and reports tied to hierarchical paths and source locations — all while remaining optional and keeping DUT/testbench code uncluttered.

## Unresolved questions
[unresolved-questions]: #unresolved-questions
- **Statement types** At present only Assign and Switch/Case are tagged. Should additional constructs (e.g. assertions, covers, or more granular branch/path coverage) be included, or left for follow-up work?
- **Scope of implementation** Should this RFC cover only the Python simulator (current implementation) or also include Verilator integration? If not, how should future work on Verilator support be tracked?  
- **Cross-run merging** Is the current global AGG_STMT_HITS sufficient, or should there be a more formal UCIS/LCOV-like database for merging across regressions?
- **Branch vs statement coverage** Should coverage distinguish between branches of conditionals (like if / else if / else) separately from generic switch_case hits, or is statement-level enough?

## Future possibilities
[future-possibilities]: #future-possibilities
The next natural step is to integrate statement coverage with other coverage metrics such as block, toggle, assertion, unreachable code, and expression coverage. Statement coverage could eventually share a common reporting interface with these metrics, allowing users to collect, merge, and visualize different coverage types through a single entry point. This would make Amaranth’s coverage ecosystem more unified and easier to adopt.

Another possibility is hierarchical aggregation, where reports summarize coverage not only per statement but also per module, instance, or subsystem. This would let users spot which parts of a design are poorly exercised without sifting through every individual assignment or case.

Over time, statement coverage could also become more configurable. For example, reports might filter by domain (sync vs comb), include/exclude certain files or submodules, or allow adjustable verbosity (summary-only vs. full detail with hit counts).

Ultimately, statement coverage could serve as a foundation for higher-level metrics. While the initial implementation focuses on recording executions of assignments and conditionals, future work could extend to branch/path coverage, standardized output formats (UCIS/LCOV), and integration with Verilator or other backends. This evolution would make statement coverage not just a basic completeness check, but a key part of a comprehensive verification framework in Amaranth.