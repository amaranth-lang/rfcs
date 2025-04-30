- Start Date: (fill in with date at which the RFC is merged, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#0075](https://github.com/amaranth-lang/rfcs/pull/75)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Property labels

## Summary
[summary]: #summary

Add a `label` argument to the `Assume`, `Assert` and `Cover` statements, that gets converted to a statement label in the verilog backend.

## Motivation
[motivation]: #motivation

Formal verification tools like [sby](https://github.com/YosysHQ/sby) or [Jasper](https://www.cadence.com/en_US/home/tools/system-design-and-verification/formal-and-static-verification.html) use the statement labels of `assert`, `assume` and `cover` statements in verilog sources to produce more human friendly messages and names. 
For example `sby` prints a message like 
```
Assert failed in $MODULE_NAME: $LABEL
```
instead of
```
Assert failed in top: $FILENAME:$LINENO 
```
if a label is present. By choosing a meaningful label it can be easier to identify the failing assertion. 

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The optional `label` argument of the `Assert`, `Assume` and `Cover` statements assigns a string label to the respective statement. For example
```python
m = Module()
a = Signal()
b = Signal()
m.d.comb += Assert(a + b == 3, label="one_plus_two_is_three")
```
The verilog backend will convert this label into a statement label for the corresponding `assert` statement. This label will then usually be used by formal verification tooling to refer to this statement. The previous example would generate verilog akin to
```verilog
always @* 
    one_plus_two_is_three: assert(a + b == 3);
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Add a optional `label` argument to `Assume`, `Assert` and `Cover`. This argument can either be `None` or a string containing no whitespaces. If not `None`, the label is used in the `rtlil` backend as the name of the corresponding `$check` cell. Since https://github.com/YosysHQ/yosys/pull/4877 this name will then be converted to a statement label in the `verilog` backend.

## Drawbacks
[drawbacks]: #drawbacks

- To generate statement labels in the verilog backend https://github.com/YosysHQ/yosys/pull/4877 is required, which raises the minimum required yosys version.
- Using `str` as the type of labels does not make it clear that no whitespace characters are allowed from the type alone.
- For `sby` this stops the filename and linenumber of the source of the property from being printed. This might make it harder find the correct file / line.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Using statement labels for `assert`, `assume` and `cover` seems to be the de facto standard for formal verification in `systemverilog`. For Jasper it is, to my knowledge, the only source used for the name of a property. In particular it also does not use the `src` attribute like `sby` does to attribute the property to the filename / linenumber specified therein. 

## Prior art
[prior-art]: #prior-art

Amaranth supported a `name` argument for properties that was lowered the same way until [RFC 50](https://amaranth-lang.org/rfcs/0050-print.html) was implemented in https://github.com/amaranth-lang/amaranth/pull/1190 where it was replaced with the message argument.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- What should happen with duplicate labels? Should this be a error, or should amaranth generate unique labels and potentially warn on duplicates?
- Should the argument be called `label` or `name`? `label` would be closer to the corresponding verilog feature of statement labels, but amaranth used to call this `name`.
- Should labels that have to be escaped in the verilog backend be supported? `write_verilog` supports this, however the generated verilog cannot be parsed by yosys, as it only supports a `simple_identifier` as statement label (See [here](https://github.com/YosysHQ/yosys/blob/18a7c00382cd64d46b61ff6cafe80851ca29cb77/frontends/verilog/verilog_lexer.l#L251-L253)). 
- How should `pysim` handle these labels?

## Future possibilities
[future-possibilities]: #future-possibilities

None
