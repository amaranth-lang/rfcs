- Start Date: 2025-09-27
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Expression Coverage

## Summary
[summary]: #summary
Expression coverage tracks every 1-bit boolean sub-expression in the design (e.g., if conditions, comparisons, switch case matches and the synthesized “default” case), recording how many times each evaluates True and False. A point is considered covered only when both T & F are observed. Results are aggregated across simulations and reported with hierarchical path and Python source file:line, giving a precise map of which predicates were exercised.

## Motivation
[motivation]: #motivation
Many bugs hide in untaken branches and one-sided conditions. Expression coverage exposes those gaps by showing which boolean decisions never flipped, guiding testbenches to add stimulus for missing edges, corner cases, and vacuous truths. It complements statement and block coverage, improves confidence in control logic, and provides a concrete metric that teams can trend and gate in CI to lift verification quality.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation
Expression coverage measures how thoroughly the tests exercise the design’s boolean decisions. During instrumentation, every 1-bit predicate found in assignments and Switch statements (including synthesized “default”) is tagged with a unique ID and a readable name that includes file:line, hierarchy path, and clock domain. The simulator inserts two tiny signals per predicate to count how often it evaluates True and False. Signal is COVERED only when both have been observed; ONE-SIDED means signal means either T or F is seen, and MISS is reported also.

##### Example: I²C Peripheral with Expression Coverage
Here we instrument the I²C harness and collect a JSON + console report.

```python
from amaranth.sim import Simulator
from amaranth.sim.coverage import ExpressionCoverageObserver
from tests.test_utils import *
from chipflow_digital_ip.io import I2CPeripheral

dut = _I2CHarness()

# Build a simulator with expression-coverage instrumentation
sim, expr_cov, expr_info, fragment = mk_sim_with_exprcov(dut, verbose=True)

sim.add_clock(1e-6)
# sim.add_testbench(testbench)

with sim.write_vcd("i2c_exprcov.vcd", "i2c_exprcov.gtkw"):
    sim.run()


# Merge and emit a JSON summary across tests
merge_exprcov(expr_cov.get_results(), expr_info)
emit_expr_summary("i2c_expression_cov.json", label="test_i2c.py", print_detail=True)
```

When finished, the simulator prints an expression coverage report:
```
[Expression coverage for test_i2c.py] 103/204 = 50.5%
MISS         (T=0, F=0, total=0)     | expr | chipflow-digital-ip/.../action.py:108 | sync:switch_case(('1',))
ONE-SIDED F  (T=0, F=3, total=3)     | expr | chipflow-digital-ip/.../action.py:68  | comb:expr(port__w_data)
ONE-SIDED T (T=3, F=0, total=3)      | expr | chipflow-digital-ip/.../_dsl.py:598   | comb:expr(1)
COVERED      (T=20, F=23, total=43)  | expr | chipflow-digital-ip/.../action.py:36  | comb:expr(r_data)
COVERED      (T=57, F=54, total=111) | expr | chipflow-digital-ip/.../cdc.py:107    | comb:expr(stage1)
...
```
The report highlights which predicates never fired, which only fired one way, and which fully toggled, pinpointing untested branches and vacuous conditions. 

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

#### API: `ExpressionCoverageObserver`

```python
class ExpressionCoverageObserver(Observer):
    def __init__(self, coverage_signal_map, state, exprid_to_info=None, **kwargs): ...
    def update_signal(self, timestamp: int, signal) -> None: ...
    def update_memory(self, timestamp: int, memory, addr) -> None: ...
    def get_results(self) -> Dict["ExprID", Dict[str, int]]: ...
    def close(self, timestamp: int) -> None: ...
```

#### Constructor
- **`coverage_signal_map`**: maps id(signal) → (expr_id, outcome), where outcome ∈ {"T","F"}.
- **`state`**: simulation state object (used by the engine; stored for completeness).
- **`exprid_to_info`**: optional dict mapping expr_id → (name, type); here type == "expr".

#### IDs & naming
- **`expr_id`**: Tuple[parent_path: Tuple[str,...], domain: str, ordinal: int] assigned during tagging.
- **`name`**: human-readable "file:line | {domain}:expr(<pretty-expr>)" for boolean sub-expressions found in assignments, or "file:line | {domain}:switch_case(<patterns>)" / "file:line | {domain}:switch_default(default)" for Switch matches.

#### Fields
- **`_expr_hits: Dict[Tuple[ExprID, str], int]`** — counts per (expr_id, "T"/"F").
- **`coverage_signal_map: Dict[int, Tuple[ExprID, str]]`** — reverse lookup from signal object to (expr_id, outcome).
- **`exprid_to_info: Dict[ExprID, Tuple[str, str]]`** — metadata (name, "expr"). 

#### Methods
- **`update_signal(timestamp, signal)`**  
  Looks up id(signal) in coverage_signal_map and increments the (expr_id, outcome) counter.
- **`update_memory(timestamp, memory, addr)`**  
  Currently a placeholder with no effect.  
- **`get_results()`**    
  Returns {expr_id: {"T": t_hits, "F": f_hits}} aggregated from the internal counters.
- **`close(timestamp)`** 
  Currently a no-op; results are typically collected via `get_results()` and merged externally.

#### Helper functions 
##### `tag_all_expressions(fragment, coverage_id=0, parent_path=(), exprid_to_info=None)`
Recursively walks the elaborated fragment, finds every 1-bit predicate in assignments and Switches, and assigns each a unique (parent_path, domain, ordinal) plus a readable name. Returns the next coverage_id and exprid_to_info map.
##### `insert_expression_coverage_signals(fragment)`
Injects two tiny probes per tagged predicate (t_sig := e, f_sig := ~e) in the correct domain, including inside Switch case bodies. Returns a dict mapping (expr_id, "T"/"F") → Signal.
##### `mk_sim_with_exprcov(dut, verbose=False)`
Elaborates the DUT, tags predicates, inserts probes, builds the coverage_signal_map, and attaches an ExpressionCoverageObserver to a Simulator. Returns (sim, expr_cov_observer, exprid_to_info, fragment).
##### `merge_exprcov(results, exprid_to_info)`
Folds a run’s {expr_id: {"T": t, "F": f}} into global accumulators AGG_EXPR_HITS and AGG_EXPR_INFO. Use across tests to aggregate coverage.
##### `emit_expr_summary(json_path, label, print_detail=False)`
Prints a one-line coverage summary (covered = both T and F seen) and optional per-expression detail. Writes a JSON report with counts and coverage status for each predicate.

#### Example Workflow
1. Build an instrumented simulator that tags predicates and inserts probes.
2. Run the simulation with clock and testbench; the inserted probe signals pulse whenever an expression’s condition is true/false; the observer automatically counts them.
3. After simulation, results are merged and `emit_expr_summary` prints a coverage report.

#### Corner Cases
- **Deduping**: A predicate node seen multiple times is tagged once (checked via _expr_coverage_id).
- **Default in Switch:**: Emitted as ~any_match and reported as "switch_default(default)".
- **Pattern strings**: '0'/'1'/'-' masks must match test width; unsupported/mismatched patterns raise ValueError during tagging.

#### Interaction with Other Features
Expression coverage complements statement/block coverage by showing whether each decision toggled both ways, and it coexists cleanly with assertion coverage to expose vacuous truths and missing stimuli. It shares the same IDs/names and JSON schema, so reports line up, aggregation is unified, and CI gates are straightforward. Instrumentation is simulation-only and coverage nets are prefixed (e.g., cov/...), keeping synthesis clean and allowing easy waveform filtering.

## Drawbacks
[drawbacks]: #drawbacks
Expression coverage adds simulation overhead and larger VCDs (two 1-bit nets per predicate). It can give a false sense of completeness: a predicate toggling both ways does not prove correct behavior. Reports can be noisy without filters (e.g., third-party IP or trivial predicates).

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives
This design tags every 1-bit predicate and counts True/False directly in the simulator, giving precise, portable results with clear names (file:line, path, domain). Alternatives considered: (1) statement/branch-only coverage—misses one-sided conditions; (2) simulator-specific coverage—locks users to one tool; (3) post-processing waveforms—slow and fragile; (4) formal-only coverage—powerful but not always available. If we don’t do this, many untaken branches remain hidden and testbenches won’t target missing edges. It fits as a library/tooling feature (observer + instrumentation), so Amaranth code stays readable and unchanged.

## Prior art
[prior-art]: #prior-art
Commercial HDL tools report condition/branch coverage and MC/DC; these are standard metrics used to expose one-sided decisions and improve safety (e.g., DO-178C). Open tools like Verilator also provide condition coverage; software tools (gcov/llvm-cov) track branch polarity similarly. Experience shows these metrics catch corner cases early, though MC/DC is stricter and more costly.

## Unresolved questions
[unresolved-questions]: #unresolved-questions
What defaults should we use (e.g., include/exclude external IP, minimum hit thresholds)? How do we balance performance vs. detail (e.g., counting transitions vs. cycle samples, filtering trivial predicates)? Do we stabilize the naming/ID schema now, and how will we map this to stricter metrics like MC/DC later?

## Future possibilities
[future-possibilities]: #future-possibilities
Add MC/DC and pairwise predicate coverage, plus excludes and goal thresholds. Generate HTML/UCIS reports, source annotations, and CI gates. Integrate with unreachable-code analysis, link predicates to failing assertions, and surface guidance to auto-target uncovered cases.