- Start Date: 2025-09-24
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Assertion Coverage

## Summary
[summary]: #summary
Introduce assertion coverage for Amaranth: during simulation we instrument Assert/Assume/Cover nodes to count outcomes (true, false, fail) per assertion, keyed by hierarchical path, clock domain, and source location. Results aggregate into a summary percentage, plus a JSON report listing each assertion’s name, type, and hit counts.

## Motivation
[motivation]: #motivation
Assertions quickly show whether tests exercised the intended behavior. Assertion coverage reports which checks fired, which were never sensitized, and which failed—surfacing dead code, missing stimulus, and unsafe assumptions. It provides backend-agnostic, consistent metrics and complements other coverage to close verification gaps.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation
Assertion coverage tracks outcomes of formal-style checks in simulation: each Assert/Assume/Cover gets a unique ID and readable name (file, line, short path/domain). Instrumentation inserts tiny signals per assertion to count true, false, and fail. Reports show HIT/MISS plus per-assert outcome counts.

##### Example: I²C Peripheral with Assertion Coverage
Because the I²C peripheral itself contains no Assert/Assume/Cover statements, I added an `I2CChecker` module (with one `Assert` and two `Cover`s) and instantiated it inside `_I2CHarness` so I could exercise and measure assertion coverage.

`I2CChecker` module:
```python
class I2CChecker(Elaboratable):
    def __init__(self, pins):
        self.pins = pins 
    def elaborate(self, platform):
        m = Module()
        sda_prev = Signal(init=1)
        scl_prev = Signal(init=1)
        boot = Signal(init=1)  
        m.d.sync += [
            sda_prev.eq(self.pins.sda.i),
            scl_prev.eq(self.pins.scl.i),
            boot.eq(0),
        ]
        sda_changed = sda_prev ^ self.pins.sda.i
        start = (sda_prev == 1) & (self.pins.sda.i == 0) & self.pins.scl.i
        stop  = (sda_prev == 0) & (self.pins.sda.i == 1) & self.pins.scl.i
        allow = start | stop
        with m.If(~boot):
            m.d.comb += Assert(~(sda_changed & scl_prev) | allow)
        m.d.comb += [
            Cover(start),
            Cover(stop),
        ]
        comb_keep = Signal()
        m.d.comb += comb_keep.eq(comb_keep)
        return m
```

Minimal setup using I²C harness and I²C checker.
```python
from amaranth.sim import Simulator
from amaranth.sim.coverage import AssertionCoverageObserver
from tests.test_utils import *
from chipflow_digital_ip.io import I2CPeripheral

dut = _I2CHarness()

# Build a simulator with assertion-coverage instrumentation
sim, assert_cov, assert_info, fragment = mk_sim_with_assertcov(dut, verbose=True)

sim.add_clock(1e-6)
# sim.add_testbench(testbench)

with sim.write_vcd("i2c_blkcov.vcd", "i2c_blkcov.gtkw"):
    sim.run()

# Merge and emit a JSON summary across tests
merge_assertcov(assert_cov.get_results(), assert_info)
emit_agg_assert_summary("i2c_assertion_cov.json", label="test_i2c.py", print_detail=True)
```

When finished, the simulator prints an assertion coverage report:
```
[Assertion coverage for test_i2c.py] 3/3 = 100.0%
HIT (true=3, false=6, fail=6, total=15) | assert | chipflow-digital-ip/tests/test_i2c.py:31 | i2c_checker | comb:assert((| (~ (& (^ (sig sda_prev) (sig i2c_pins__sda__i)) (sig scl_prev))) (| (& (& (== (sig sda_prev) (const 1'd1)) (== (sig i2c_pins__sda__i) (const 1'd0))) (sig i2c_pins__scl__i)) (& (& (== (sig sda_prev) (const 1'd0)) (== (sig i2c_pins__sda__i) (const 1'd1))) (sig i2c_pins__scl__i)))))
HIT (true=6, false=0, fail=0, total=6) | cover | chipflow-digital-ip/tests/test_i2c.py:33 | i2c_checker | comb:cover((& (& (== (sig sda_prev) (const 1'd1)) (== (sig i2c_pins__sda__i) (const 1'd0))) (sig i2c_pins__scl__i)))
HIT (true=6, false=0, fail=0, total=6) | cover | chipflow-digital-ip/tests/test_i2c.py:34 | i2c_checker | comb:cover((& (& (== (sig sda_prev) (const 1'd0)) (== (sig i2c_pins__sda__i) (const 1'd1))) (sig i2c_pins__scl__i)))
```
The report highlights which assertions were exercised, how often each outcome occurred, and flags vacuous checks (if any) as MISS.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

#### API: `AssertionCoverageObserver`

```python
class AssertionCoverageObserver(Observer):
    def __init__(self, coverage_signal_map, state, assertid_to_info=None, **kwargs): ...
    def update_signal(self, timestamp: int, signal: Signal) -> None: ...
    def update_memory(self, timestamp: int, memory, addr) -> None: ...
    def get_results(self) -> Dict[AssertID, Dict[str, int]]: ...
    def close(self, timestamp: int) -> None: ...
```

#### Constructor
- **`coverage_signal_map`**: maps id(signal) → (assert_id, outcome), where outcome ∈ {"true","false","fail"}.
- **`state`**: simulation state object (used by the engine; stored for completeness).
- **`assertid_to_info`**: optional dict mapping assert_id → (name, type); type ∈ {"assert","assume","cover"}.

#### IDs & naming
- **`assert_id`**: Tuple[parent_path: Tuple[str,...], domain: str, ordinal: int] assigned during tagging.
- **`name`**: human-readable string "file:line | path | domain:type(cond)", from get_assert_name(...).

#### Fields
- **`_assert_hits: Dict[AssertID, Dict[str,int]]`** — per-assert outcome counters: {"true": x, "false": y, "fail": z}.
- **`coverage_signal_map: Dict[int, Tuple[AssertID,str]]`** — reverse lookup from signal ID to (assert_id, outcome).
- **`assertid_to_info: Dict[AssertID, Tuple[str,str]]`** — metadata (name, type).  

#### Methods
- **`update_signal(timestamp, signal)`**  
  Looks up id(signal) in coverage_signal_map, then increments the bucket for that (assert_id, outcome).
- **`update_memory(timestamp, memory, addr)`**  
  Currently a placeholder with no effect.  
- **`get_results()`**    
  Returns {assert_id: {"true": t, "false": f, "fail": x}}.
- **`close(timestamp)`** 
  Currently a no-op; results are typically collected via `get_results()` and merged externally.

#### Helper functions 
##### `tag_all_asserts(fragment, coverage_id=0, parent_path=(), ...)`
Recursively walks the fragment to find Assert/Assume/Cover nodes and assigns each an assert_id, type, and name. It returns updated coverage_id, a mapping assertid_to_info, and found_nodes for later instrumentation.
##### `insert_assert_coverage_signals(fragment, found_nodes)`
For each found node, inserts small probe signals in the same clock domain that pulse on true/false/fail (or just true for covers). It returns coverage_signal_map mapping id(signal) to (assert_id, outcome) for the observer.
##### `mk_sim_with_assertcov(dut, verbose=False)`
Elaborates the DUT, tags assertions, inserts probes, constructs an AssertionCoverageObserver, and attaches it to the simulator. It returns (sim, assert_cov_observer, assertid_to_info, fragment) ready to run.
##### `merge_assertcov(results, assertid_to_info)`
Merges a single run’s per-assert outcome counts into global aggregators AGG_ASSERT_HITS and AGG_ASSERT_INFO. Use this across multiple tests to accumulate coverage.
##### `emit_agg_assert_summary(json_path, label, print_detail=False)`
Prints a one-line summary and optionally a detailed per-assert table, then writes a JSON report with outcome counts.

#### Example Workflow
Using the provided I²C harness and checker:
1. Build an instrumented simulator: `sim, assert_cov, assert_info, frag = mk_sim_with_assertcov(dut, verbose=True)`. This elaborates your DUT, finds every Assert/Assume/Cover, tags them with IDs/names, inserts tiny probe signals for true/false/fail, and hooks up an observer that will count those probes.
2. Run the simulation with clock and testbench; the inserted probe signals pulse whenever an assertion’s condition is true/false/fail; the observer automatically counts them.
3. After simulation, results are merged and `emit_agg_assert_summary` prints a coverage report.

#### Corner Cases
- **Nested fragments**: handled via recursive tagging (parent_path preserves hierarchy).
- **Covers vs asserts/assumes:**: covers only track true; asserts/assumes track true/false/fail.
- **Nodes without a condition**: skipped (no probes inserted).

#### Interaction with Other Features
- Complements statement/block/expression coverage by showing which checks were exercised, vacuous, or failing.
- Coexists with other observers; you can enable all coverage types simultaneously.
- Opt-in and non-intrusive; existing designs run unchanged if not enabled.

## Drawbacks
[drawbacks]: #drawbacks
Assertion coverage introduces small runtime and memory overhead from probe signals, and high coverage can still be misleading because metrics are gameable. Verbose names/IDs can add log noise, and minor backend differences (e.g., what counts as “fail”) may require adapters.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives
Tag-and-probe with an observer is backend-agnostic, opt-in, and requires no DUT edits, while stable IDs and readable names make reports diff-friendly and CI-ready. Simulator-native coverage would fragment formats; post-processing VCDs is heavy and lossy; formal-only metrics exclude simulation; and macro/wrapper approaches are intrusive and brittle. Without this, vacuous or never-exercised checks remain hidden, weakening regression signals and CI gates. This is a library/simulation feature, not a language change, and it leaves user code clean and unchanged.

## Prior art
[prior-art]: #prior-art
SystemVerilog/PSL and UVM provide assertion coverage and UCIS databases that are proven but vendor-specific and often opaque; portability and uniform naming are limited. Python HDL ecosystems (e.g., cocotb add-ons) offer pieces of coverage, but not an Amaranth-native, backend-agnostic assertion metric integrated with the Amaranth simulator flow.

## Unresolved questions
[unresolved-questions]: #unresolved-questions
We still need to decide if an Assert that is false in simulation should simply count as “false” or also be a “fail” that breaks CI. We must also pick a clear rule for what to do when an Assume is broken in simulation. Reports should keep the same IDs and JSON fields so results stay comparable. We need to check that everything works the same (and fast) on other backends like Verilator/CXXRTL. Finally, we’ll choose sensible default CI checks (e.g., no unhit covers, no fails) and provide waivers for known exceptions.

## Future possibilities
[future-possibilities]: #future-possibilities
Potential extensions include UCIS export, HTML/IDE links to source, filters and waivers, and unified formal+simulation reporting. We can add richer stats like first/last hit timestamps and rates, cross-coverage with FSM state or expression/toggle data, and trend dashboards to track coverage over time across regressions.