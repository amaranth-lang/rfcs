- Start Date: 2025-09-21
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Block Coverage

## Summary
[summary]: #summary
Introduce block coverage, which tracks execution of larger HDL regions such as root statement lists and switch/case blocks. A block is a set of statements with a single entry and exit point. These statements are executed as a unit, such as each switch case body or statements per clock domain (sync/ comb) in a fragment. Reports show HIT/MISS status (and counts), with file and line annotations from the Python source that generated the HDL.

## Motivation
[motivation]: #motivation
Block coverage complements statement coverage by focusing on whether whole regions of code are exercised, not just individual statements. It highlights untested case branches or dead blocks, guiding testbench improvements. Built-in support in Amaranth provides a consistent, simulator-agnostic metric that aligns with standard verification practice.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation
Block coverage tracks execution of larger HDL regions like root processes and switch/case branches. Each block gets a unique ID and a readable name (file, line, short summary). A coverage signal is inserted at the block head; when triggered, the BlockCoverageObserver logs an entry. Reports show HIT/MISS status and how many times each block ran.

##### Example: I²C Peripheral with Block Coverage
Below is a minimal setup for an I²C Peripheral. 

```python
from amaranth.sim import Simulator
from amaranth.sim.coverage import BlockCoverageObserver
from tests.test_utils import *
from chipflow_digital_ip.io import I2CPeripheral

dut = _I2CHarness()

# Build a simulator with block-coverage instrumentation
sim, blk_cov, blk_info, fragment = mk_sim_with_blockcov(dut, verbose=True)

sim.add_clock(1e-6)
# sim.add_testbench(testbench)

with sim.write_vcd("i2c_blkcov.vcd", "i2c_blkcov.gtkw"):
    sim.run()

# Merge and emit a JSON summary across tests
merge_blockcov(blk_cov.get_results(), blk_info)
emit_agg_block_summary("i2c_block_cov.json", label="tests/test_i2c.py")
```

When finished, the simulator prints a block coverage report:
```
[Block coverage for tests/test_i2c.py] 41/83 = 49.4%
HIT (3x) | .../csr/action.py:108 | i2c/bridge/divider/val | sync:case('1')
HIT (1x) | .../hdl/_dsl.py:226   | sync:fsm_state = 0
MISS (0x)| .../hdl/_dsl.py:226   | sync:fsm_state = 10
...
```
The report highlights which blocks were entered and how often, such as a case('1') branch hit three times, and others like fsm_state = 10 that were never reached.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

#### API: `BlockCoverageObserver`

```python
class BlockCoverageObserver(Observer):
    def __init__(self, coverage_signal_map, state, blockid_to_info=None, **kwargs): ...
    def update_signal(self, timestamp: int, signal: Signal) -> None: ...
    def update_memory(self, timestamp: int, memory, addr) -> None: ...
    def get_results(self) -> Dict[BlockID, int]: ...
    def close(self, timestamp: int) -> None: ...
```

#### Constructor
- **`coverage_signal_map`**: maps id(signal) → block ID.
- **`state`**: simulation state object, used to track signal values.  
- **`blockid_to_info`**: optional dictionary mapping block IDs → (name, type).

#### Fields
- **`_block_hits: Dict[BlockID, int]`** — hit counts for each block.
- **`coverage_signal_map: Dict[int, BlockID]`** — reverse lookup from signal ID to block ID.  
- **`blockid_to_info: Dict[BlockID, Tuple[str, str]]`** — block metadata (name, type).  

#### Methods
- **`update_signal(timestamp, signal)`**  
  Increments the hit counter when a tagged block’s signal fires.
- **`update_memory(timestamp, memory, addr)`**  
  Currently a placeholder with no effect.  
- **`get_results()`**    
  Returns `{block_id: hit_count}`.
- **`close(timestamp)`** 
  Currently a no-op; results are typically collected via `get_results()` and merged externally.

#### Helper functions 
##### `tag_all_blocks(fragment)`
Recursively assigns IDs and names to root statement lists and switch/case branches.
##### `insert_block_coverage_signals(fragment, blockid_to_info)`
Inserts coverage signals at the start of each block so execution can be logged.
##### `mk_sim_with_blockcov(dut, verbose=False)`
Builds an instrumented simulator, attaches a `BlockCoverageObserver`, and returns `(sim, blk_cov, blockid_to_info, fragment)`.
##### `merge_blockcov(results, blockid_to_info)`
Merges results into global aggregators `AGG_BLOCK_HITS` and `AGG_BLOCK_INFO`.
##### `emit_agg_block_summary(json_path, label)`
Prints a console summary and writes a JSON file with block hit/miss info and overall coverage %.

#### Example Workflow
Using the `I²C peripheral` testbench (see Guide-level explanation):

1. Build a simulator with `mk_sim_with_blockcov(dut)`.
2. Helper tags all blocks and inserts coverage signals.
3. During simulation, when a block executes, its signal pulses high. The observer records the hit.
4. After simulation, results are merged and `emit_agg_block_summary` prints a coverage report.

#### Corner Cases
- **Nested fragments**: recursion ensures nested blocks are covered.
- **Default switch cases:**: labeled as `case:default`. 
- **FSM states**: each `Case` in a `Switch(fsm_state)` corresponds to a state. Block coverage tags these case bodies as blocks, so it records which states were entered (HIT) and which were not (MISS). This provides basic state reachability, but not state transition coverage.

#### Interaction with Other Features
- Complements statement coverage by showing whether entire regions of code (blocks or case branches) were exercised.
- Works with other observers (assertion, expression, toggle etc).
- Opt-in and non-intrusive; existing designs run unchanged if not enabled.

## Drawbacks
[drawbacks]: #drawbacks
Block coverage gives a coarse view of exercised regions but lacks detail: it only shows whether a block was entered, not which individual statements inside ran or whether transitions between blocks were taken. Nested logic may inflate block counts, making percentages harder to interpret, and overlapping with statement coverage can cause redundancy. It also does not capture correctness—only activity—so a HIT does not guarantee the block behaved as intended.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives
This design reuses Amaranth’s fragment traversal and observer infrastructure, making it lightweight, simulator-agnostic, and consistent with other coverage metrics. Alternatives like external waveform inspection or simulator-specific options were rejected as they are harder to automate, less portable, and inconsistent across tools. Without block coverage, users miss a key metric for verifying control paths and FSM state reachability, leading to weaker testbenches. 

## Prior art
[prior-art]: #prior-art
Block coverage is a standard feature in hardware verification, widely supported in commercial simulators (e.g. ModelSim, VCS, Questa) and defined in methodologies like SystemVerilog’s UCIS. These tools show that block-level metrics are useful for catching untested branches and FSM states, but they are often tied to proprietary flows and not portable. In contrast, Amaranth currently lacks built-in support, so adding block coverage aligns it with established practice while keeping the solution open, lightweight, and consistent with its Python-based ecosystem.

## Unresolved questions
[unresolved-questions]: #unresolved-questions
Some open points remain to be clarified during the RFC process, such as the exact naming format for blocks (e.g. whether to shorten hierarchical paths or include domain tags) and how much metadata should be exposed in reports. Implementation will confirm performance impact and ensure block tagging integrates cleanly with other observers (statement, assertion, expression). Out of scope for this RFC are more advanced metrics like FSM transition coverage, or cross-coverage between blocks and signals, which could be proposed separately in future extensions.

## Future possibilities
[future-possibilities]: #future-possibilities
Future extensions of block coverage could include FSM transition coverage (tracking arcs between states, not just state entry), hierarchical coverage summaries (roll-ups per submodule or domain), and integration into a unified UCIS-like coverage database for cross-tool compatibility. Another direction is combining block coverage with assertion and expression coverage to highlight untested behaviors more precisely.