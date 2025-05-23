- Start Date: 2025-05-12
- RFC PR: [amaranth-lang/rfcs#0074](https://github.com/amaranth-lang/rfcs/pull/0074)
- Amaranth Issue: [amaranth-lang/amaranth#1599](https://github.com/amaranth-lang/amaranth/issues/1599)

# Structured VCD Output

## Summary
[summary]: #summary

When generating VCD files, use the `scope` keyword to distinguish aggregate signals.

## Motivation
[motivation]: #motivation

These changes are intended to make it easier for waveform viewers to recover information about aggregate signals from VCD files.
This is partially motivated by an issue with VCD parsing that occurs in Surfer (see [ekiwi/wellen#36](https://github.com/ekiwi/wellen/issues/36)).
Ideally, this leads to a better user experience when inspecting simulated Amaranth designs with a waveform viewer.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC describes changes to the way Amaranth generates a VCD file.
When reading a VCD file, waveform viewers like [Surfer](https://surfer-project.org/) and [GTKWave](https://gtkwave.sourceforge.net/) support use of the `scope` keyword for organizing variables into groups.
Currently, when writing a VCD file, Amaranth uses the `module` scope to distinguish between signals belonging to different modules.
For instance:

```
$scope module top $end          # top
  $var wire 1 <id> clk $end     # top.clk
  $var wire 1 <id> rst $end     # top.rst
  $var wire 32 <id> in $end     # top.in
  $var wire 32 <id> out $end    # top.out
  $scope module submodule $end  # top.submodule
    $var wire 1 <id> clk $end   # top.submodule.clk
    $var wire 1 <id> rst $end   # top.submodule.rst
    $var wire 5 <id> in $end    # top.submodule.in
    $var wire 5 <id> out $end   # top.submodule.out
  $upscope $end
$upscope $end
```

However, it does *not* attempt to use scopes for organizing the members of aggregate signals with array-like or struct-like datatypes.
Instead, when creating VCD variables for aggregate signals, members are distinguished only by appending to the name of the parent signal.
For instance, a signal `top.submodule.my_array` with the type `ArrayLayout(unsigned(32), 4)` is currently represented as:

```
$scope module top $end                    # top
  ...
  $scope module submodule $end            # top.submodule
    ...
    $var wire 128 <id> my_array $end      # top.submodule.my_array
    $var wire 32 <id> my_array[0] $end    # top.submodule.my_array[0]
    $var wire 32 <id> my_array[1] $end    # top.submodule.my_array[1]
    $var wire 32 <id> my_array[2] $end    # top.submodule.my_array[2]
    $var wire 32 <id> my_array[3] $end    # top.submodule.my_array[3]
  $upscope $end
$upscope $end
```

This is not sufficient for explicitly conveying the relationship between aggregate signals and their members to a waveform viewer.
In this case, `my_array` does not explicitly *contain* its members, and waveform viewers may only *infer* this relationship by attempting to recover it from the names.

This RFC proposes the use of `scope vhdl_array` and `scope vhdl_record` to pass information about these relationships to a waveform viewer.
After this change, the VCD output above would become:

```
$scope module top $end                    # top
  ...
  $scope module submodule $end            # top.submodule
    ...
    $comment Flattened representation of 'my_array' $end
    $var wire 128 <id> my_array $end      # top.submodule.my_array

    $comment Hierarchical representation of 'my_array' members $end
    $scope vhdl_array my_array $end       # top.submodule.my_array
      $var wire 32 <id> 0 $end            # top.submodule.my_array.0
      $var wire 32 <id> 1 $end            # top.submodule.my_array.1
      $var wire 32 <id> 2 $end            # top.submodule.my_array.2
      $var wire 32 <id> 3 $end            # top.submodule.my_array.3
    $upscope $end

  $upscope $end
$upscope $end
```

These images demonstrate the difference in GTKWave output:

<figure>
  <img align="center" src="./0074-structured-vcd/gtkwave_pre_rfc.png" alt="a screenshot of array-like signals in GTKWave (current behavior)">
  <figcaption><i>Figure 1. A screenshot of array-like signals in GTKWave (current behavior)</i></figcaption>
</figure>

<figure>
<img align="center" src="./0074-structured-vcd/gtkwave_post_rfc.png" alt="a screenshot of array-like signals in GTKWave (proposed behavior)">
  <figcaption><i>Figure 2. A screenshot of array-like signals in GTKWave (proposed behavior)</i></figcaption>
</figure>

... and the difference in Surfer output:

<figure>
  <img align="center" src="./0074-structured-vcd/surfer_pre_rfc.png" alt="a screenshot of array-like signals in Surfer (current behavior)">
  <figcaption><i>Figure 3. A screenshot of array-like signals in Surfer (current behavior)</i></figcaption>
</figure>

<figure>
<img align="center" src="./0074-structured-vcd/surfer_post_rfc.png" alt="a screenshot of array-like signals in Surfer (proposed behavior)">
  <figcaption><i>Figure 4. A screenshot of array-like signals in Surfer (proposed behavior)</i></figcaption>
</figure>

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Currently, Amaranth represents aggregate signals in the VCD by appending the names of elements/members to the name of the signal.
When simulator output is written to a VCD file, we intend that signals with aggregate datatypes are explicitly given their own scope and split into multiple VCD variables.

### Changes to VCD Writing

We propose the following conventions:

- The `module` scope defines a group of signals belonging to the same `Module`
- The `vhdl_array` scope defines a group of signals belonging to an array (such as a signal with `ArrayLayout`)
- The `vhdl_record` scope defines a structured group of signals (such as a signal with `StructLayout`, `UnionLayout`, or `FlexibleLayout`)

After these changes, this functionality will be enabled by default in the VCD writer.
These changes will **not** be backported to Amaranth 0.5.x.
Some users may require compatibility with output that is closer to the actual VCD specification.
In this case, users that need the pre-RFC behavior are expected to either:

- Opt-out on a case-by-case basis by passing arguments to `write_vcd()` (ie. `pure=True`)
- Opt-out globally by setting an environment variable (ie. `AMARANTH_PURE_VCD=1`)

### Changes to `pyvcd`

The VCD writer in Amaranth depends on `pyvcd`, which represents VCD scopes with a `ScopeType` enum.
`vhdl_record` and `vhdl_array` variants are required to implement this RFC.

### Changes to Amaranth Playground

The Amaranth Playground has a waveform viewer that depends on the [`d3-wave`](https://github.com/Nic30/d3-wave) library.
`d3-wave` has an exported/extensible `RowRendererBits` class that can be used to implement renderers for particular datatypes.
Custom renderers for aggregate datatypes are required for rendering grouped signals in the Amaranth Playground.

### Expected VCD Output: Array-like

```python
from amaranth import *
from amaranth.lib.data import *

foo = Signal(ArrayLayout(unsigned(32), 4))
```

For `ArrayLayout`, each element in the array should become a VCD variable whose name is integer array index.
In this case, the signal `foo` is split into `foo.0`, `foo.1`, `foo.2`, and `foo.3`:

```
$scope vhdl_array foo $end
  $var wire 32 <id> 0 $end
  $var wire 32 <id> 1 $end
  $var wire 32 <id> 2 $end
  $var wire 32 <id> 3 $end
$upscope $end
```

### Expected VCD Output: Struct-like

```python
from amaranth import *
from amaranth.lib.data import *

foo = Signal(StructLayout({
  "a": unsigned(1),
  "b": unsigned(4),
  "c": unsigned(32),
}))
```

For `StructLayout`, each member should become a VCD variable with the member name.
In this case, the signal `bar` is split into `bar.a`, `bar.b`, and `bar.c`:

```
$scope vhdl_record bar $end
  $var wire 1 id a $end
  $var wire 4 id b $end
  $var wire 32 id c $end
$upscope $end
```

### Expected VCD Output: Nested Aggregates

```python
from amaranth import *
from amaranth.lib.data import *

class MyStruct(Struct):
    a: unsigned(1)
    b: unsigned(4)
    c: ArrayLayout(unsigned(32), 4),

# A struct-like signal
foo = Signal(MyStruct)

```

For signals with nested aggregate types, the scopes are nested in the same way that `scope module` is already used for nested modules.
In this case, the signal `foo` is split like this:

```
$scope vhdl_record bar $end
  $var wire 1 id a $end
  $var wire 4 id b $end
  $scope vhdl_array c $end
    $var wire 32 id 0 $end
    $var wire 32 id 1 $end
    $var wire 32 id 2 $end
    $var wire 32 id 3 $end
  $upscope $end
$upscope $end
```

## Drawbacks
[drawbacks]: #drawbacks

- VCD is not a particularly efficient format, and this adds even more bytes to generated VCD files.

- This breaks any existing compatibility with waveform viewers that do not handle the `vhdl_record` and `vhdl_array` scope types.
  Since these are not part of the VCD specification, some waveform viewers may fail to handle this.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- The choice of `scope vhdl_array` and `scope vhdl_record` are suggested due to known compatibility with Surfer and GTKWave.
  However, note that these are not part of the VCD specification (which is somewhat old and under-defined).

### Alternatives

- Expose an environment variable (or a parameter in `write_vcd()`) allowing users to opt-in/opt-out of structured VCD output
- Include support for a different waveform format that has better-defined support for variables with composite datatypes.

## Prior art
[prior-art]: #prior-art

- As far as I can tell, use of the `vhdl_record` and `vhdl_array` scope types comes from the [ghdl/ghdl](https://github.com/ghdl/ghdl) simulator.
- [verilator/verilator](https://github.com/verilator/verilator) makes no attempt to use these scope types
- [steveicarus/iverilog](https://github.com/steveicarus/iverilog) makes no attempt to use these scopes types

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

Since this adds more overhead to VCD output, it's worth mentioning that VCD may be unsuitable for testing very large designs.
This RFC can also serve as a stepping stone for supporting alternative waveform formats in the future.

