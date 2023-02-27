- Start Date: 2023-02-07
- RFC PR: [amaranth-lang/rfcs#4](https://github.com/amaranth-lang/rfcs/pull/4)
- Amaranth Issue: [amaranth-lang/amaranth#755](https://github.com/amaranth-lang/amaranth/issues/755)

# Const-castable expressions

## Summary
[summary]: #summary

Define a subset of expressions that are "const-castable" and accept them in contexts where currently only integers and enumerations with integer values are accepted.

## Motivation
[motivation]: #motivation

In certain contexts, Amaranth requires a constant to be used. These contexts are: `with m.Case(...):`, `Value.matches(...)`, and the value of an enumeration member that is being cast to a value.

Currently, only integers and enumeration members with integer values are considered constants. However, this is too limiting. For example, when developing a CPU, one might want to define control signals for several functional units and combine them into instructions, or conversely, match an instruction against a combination of control signals:

```python
class Func(Enum):
    ADD = 0
    SUB = 1

class Src(Enum):
    MEM = 0
    REG = 1

class Instr(Enum):
    ADD  = Cat(Func.ADD, Src.MEM)
    ADDI = Cat(Func.ADD, Src.REG)
    ...

with m.Switch(instr):
    with m.Case(Cat(Func.ADD, Src.MEM)):
        ...
```

Currently, all of these cases would produce syntax errors.

There is a private `Value._as_const` method. It is not used internally, however Amaranth developers have started using it due to unmet needs. Removing it without providing a replacement would be disruptive, and will result in downstream codebases defining their own equivalent.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In any context where a constant is accepted (`with m.Case(...):`, `Value.matches(...)`, and the value of an enumeration member), a "const-castable" expression can be used. The following expressions are const-castable:
* `int`;
* `Const`;
* `Cat` where all operands are const-castable;
* instance of a `Enum` subclass where the value is const-castable.

A const-castable expression can be converted to a `Const` using `Const.cast`:

```python
>>> Const.cast(1)
(const 1'd1)
>>> Const.cast(Cat(1, 0, 1))
(const 3'd5)
>>> Const.cast(Cat(Func.ADD, Src.REG))
(const 2'd2)
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `Const.cast` static method casts its argument to a value and examines it. If it is a `Const`, it is returned. If it is a const-castable expression, the operands are recursively cast to constants, and the expression is evaluated.

The list of const-castable expressions is:
* `Cat`

The `m.Case(...)` (through the `Switch()` constructor) and `Value.matches(...)` methods accept two kinds of operands: const-castable expressions, or a string containing a bit pattern (a sequence of `0`, `1`, or `-` meaning a wildcard).

The `Shape.cast` method accepts enumerations where all members have const-castable expressions as their values. The shape of an enumeration is a shape with the smallest width that can represent the value of any enumeration member.

[RFC 3][]: The `EnumMeta.__new__` method accepts enumerations where all members have const-castable expressions as their values. The values of members are the user-provided const-castable expressions, and not the result of casting them to a constant.

[rfc 3]: enumeration-shapes.html

## Drawbacks
[drawbacks]: #drawbacks

- A new language-level concept makes it harder to learn the language.
  - Most developers already have an intuitive understanding of which expressions are const-castable.
- `Const.cast` shadows an existing `Value.cast` method since `Const` inherits from `Value`.
  - No one is calling `Value.cast` through the `Const.cast` binding.
  - `Const.cast` has a compatible interface (it returns a `Value`) and performs a similar function (it calls `Value.cast` first). However, it's not Liskov-compatible.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Alternatives:

1. Do not add this functionality. Developers will define their own const-casting functions, continue to rely on the undocumented and private `._as_const()` method, or use other workarounds.
2. Make `._as_const()` public (i.e. rename it to `.as_const()`).
3. Add a new `Const.cast` method (this option).

Alternatives (2) and (3) both introduce a new language-level concept, the only difference is in the interface that is used to access it.

Alternative (3) fits the language better: `Value.cast` takes something value-castable and returns a `Value`, `Shape.cast` takes something shape-castable and returns a `Shape`, so `Const.cast` is a logical addition in that it takes something const-castable and returns a `Const`.

## Prior art
[prior-art]: #prior-art

Rust and C++ provide functionality (`const fn` and `constexpr` respectively) for performing computation at compile time, restricted to a strict subset of the full language. In particular, it can be used to initialize constants, which makes it similar to the functionality proposed here.

One challenge these languages face is the question of how large the subset should be. Rust in particular started off heavily restricting `const fn`, where it did not have any control flow. The functionality was gradually introduced later as needed.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

Expanding the set of const-castable expressions to include arbitrary arithmetic operations. This RFC limits it to the most requested expression, `Cat`. This simplifies implementation and reduces the likelihood of introducing bugs in the constant evaluation code, most of which would be almost never used.
