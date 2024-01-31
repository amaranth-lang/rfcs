- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#44](https://github.com/amaranth-lang/rfcs/pull/44)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Syntax for initial state property assertions

## Summary
[summary]: #summary

Add dedicated syntax for describing assertions or assumptions for the initial simulation state.

## Motivation
[motivation]: #motivation

At the moment, to e.g. assume that an input has a certain value at the initial step of formal verification, the following syntax is used:

```python
from amaranth.asserts import Initial, Assume

with m.If(Initial()):
    with m.If(x):
        m.d.comb += Assume(y == 0)
```

There are several issues with this approach:
- `Initial()`, as it stands (being a `Value`) can be used in expressions, in which case it's translated to the Yosys-specific `$initstate()` cell that's not supported anywhere else. No validation is performed that `Initial()` is only used in well-defined positions.
- The semantics of `Initial()` depend on the Yosys passes that are used (`multiclock on` / `async2sync`) and cannot be expressed without relying on the concept of a global clock, which is another Yosys extension. (Consider what `with m.If(~Initial()):` should be lowered to in plain SystemVerilog.)
- No validation is performed that statements are only added with `m.d.comb +=`. Adding statements with `m.d.sync +=` is meaningless.
- No validation is performed that only formal properties are added. Adding assignments is meaningless.
- `Initial()` is an extra name in the crowded global namespace.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

To define a formal property at the initial simulation step, use:

```python
from amaranth.asserts import Assume

with m.Initial():
    with m.If(x):
        m.d.comb += Assume(y == 0)
```

This syntax can also be used to print values of signals at the initial simulation step. Only the `comb` domain may be used, and no assignments can be made.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The module builder is extended with the following syntax:

```
with m.Initial():
    ...
```

The only constructs valid within a `with m.Initial():` block are:

- Control flow. (These become property enables.)
- `m.d.comb += Property(...)`. (This covers `Assert`, `Assume`, `Cover`.)
- `m.d.comb += Print(...)`. (When there is a `Print` statement.)

This syntax is always translated to the Verilog construct:

```
initial begin
    ...
end
```

The exact details of how this is done are not specified and it depends on the interaction with the Verilog backend. Currently (2024-01) this should be translated to async `$check` or `$print` Yosys cells with enabled trigger and empty trigger list, which is then reliably lowered into the appropriate Verilog construct. No `$initstate` cell is ever used.

## Drawbacks
[drawbacks]: #drawbacks

None. The current implementation is untenable.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

If we are emitting generic SVA (not Yosys-specific) then something, somewhere, must emit the `initial` Verilog block. It may be possible to add an additional level of indirection so that no new control flow construct is required. However, that would not provide the validation against assinging values at the initial time, which isn't something we should allow in Amaranth. Considering the special properties of doing things at the initial time step, a new control flow construct seems reasonable.

Instead of `with m.Initial():`, we could potentially have `m.initial +=`. For example:

```python
from amaranth.asserts import Assume

with m.If(x):
    m.initial += Assume(y == 0)
    m.d.comb += ...
```

This is exactly equivalent to `with m.Initial():` in terms of expressivity, as you can still nest `with m.Initial():` in `with m.If():` if you want. It looks more domain-like than control-flow-like, which may be a benefit or a drawback depending on your viewpoint. It also avoids the somewhat awkward prohibition of normal constructs like `m.d.sync +=` and `m.d.comb += x.eq(y)` in exactly one control flow context.

It's unclear if we should call this `initial`. The inventor of Verilog [regrets](https://archive.computerhistory.org/resources/access/text/2013/11/102746653-05-01-acc.pdf#page=54) the choice of `initial`, and wishes he could have used `once`:

> **Moorby:** First of all, Scott Sandler came up with probably the right alternative to what “initial” should have been, and that was “once.” He said, “Phil, you shouldn’t have called it ‘initial.’ It should’ve been ‘once,’” because “initial” is sometimes used with a timing control within it, so it isn’t necessarily initial. So, you can have “initial” this, that and then do “forever,” right, so it will go on forever. So the keyword “initial” probably was quite correctly the wrong word, so “once” would’ve been good. And I think between “always” and “forever” there’s one of them should have gone and maybe both. So, you can always write “initial” and once you get into it to write “forever”—I mean, there’s many alternate—maybe out of those three keywords you could’ve only just—try and work through—tried to have it so there’s only one, maybe two.

## Prior art
[prior-art]: #prior-art

This feature heavily draws from the Verilog `initial` construct.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should this be a `with m.X():` control flow style construct or a `m.x +=` statement style construct?
- Should this be named "initial" or something else?

## Future possibilities
[future-possibilities]: #future-possibilities

Whenever `Print` is added, printing values at the initial time is possible and sometimes useful, which is a straightforward addition at that time.