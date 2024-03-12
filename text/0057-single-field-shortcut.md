- Start Date: 2024-03-15
- RFC PR: [amaranth-lang/rfcs#57](https://github.com/amaranth-lang/rfcs/pull/57)
- Amaranth SoC Issue: [amaranth-lang/amaranth-soc#78](https://github.com/amaranth-lang/amaranth-soc/issues/78)

# Single-Field Register Definition and Usage Shortcut

## Summary
[summary]: #summary

Add a shortcut to `csr.Register` to enable easier and cleaner definition and use
of registers which contain exactly one `csr.Field`.

## Motivation
[motivation]: #motivation

Currently, use of a `csr.Register` in an amaranth-soc peripheral requires creation of a map (or list) of `csr.Field`s, which actually represent the register data and control access to each part. This is commonly done by creating a `Register` subclass. Each `Field` also needs a name (or index) in the `Register`. However, many registers in peripherals only contain one field, so the extra name and subclass are unnecessary and redundant. This RFC introduces a shortcut for constructing and using a `Register` containing exactly one un-named `Field`.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Consider the following simple peripheral which generates a toggling signal on bits of an output corresponding to bits set in a register:

```python
from amaranth import *
from amaranth.lib import wiring
from amaranth.lib.wiring import In, Out
from amaranth_soc import csr

class Toggle(wiring.Component):
    bus: In(csr.Signature(addr_width=1, data_width=32))
    toggle: Out(32)

    class Toggle(csr.Register, access="rw"):
        toggle: csr.Field(csr.action.RW, 32)

    def __init__(self):
        self._toggle = Toggle()

        # ... bridge setup ...

    def elaborate(self, platform):
        m = Module()

        # ... bridge setup ...

        for i in range(32):
            with m.If(self._toggle.f.toggle.data[i]):
                m.d.sync += self.toggle[i].eq(~self.toggle[i])
            with m.Else():
                m.d.sync += self.toggle[i].eq(0)

        return m
```

The `toggle` name is used, among other ways, to name the register class, to name
the field within, and to access that field on the register instance during
elaboration.

This can be simplified by passing the `csr.Field` directly to the `csr.Register`
without enclosing it in a dict/list and naming/indexing it, or subclassing
`csr.Register`. This also causes the `.f` member to access that Field directly
instead of needing to provide a name/index.

This simplifies the design as follows, reducing clutter and redundancy:

```python
from amaranth import *
from amaranth.lib import wiring
from amaranth.lib.wiring import In, Out
from amaranth_soc import csr

class Toggle(wiring.Component):
    bus: In(csr.Signature(addr_width=1, data_width=32))
    toggle: Out(32)

    def __init__(self):
        # Register can take the Field directly, eliminating the subclass
        self._toggle = csr.Register(csr.Field(csr.action.RW, 32), access="rw")

        # ... bridge setup ...

    def elaborate(self, platform):
        m = Module()

        # ... bridge setup ...

        for i in range(32):
            # FieldAction accessed directly, no need to specify `toggle` again
            with m.If(self._toggle.f.data[i]):
                m.d.sync += self.toggle[i].eq(~self.toggle[i])
            with m.Else():
                m.d.sync += self.toggle[i].eq(0)

        return m
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

(with reference to [RFC #16](https://github.com/amaranth-lang/rfcs/blob/main/text/0016-soc-csr-regs.md))

#### `csr.reg.Register`

The `csr.reg.Register` class describes a CSR register `Component`, with:
- a `.__init__(self, fields=None, access=None)` constructor, where:
    * `fields` is either:
        - a `dict` that will be instantiated as a `FieldActionMap`;
        - a `list` that will be instantiated as a `FieldActionArray`;
        - a `Field` that will be instantiated as a `FieldAction`;
        - `None`; in this case a `FieldActionMap` is instantiated from `Field` objects in variable annotations.
    * `access` is either a `csr.Element.Access` value, or `None`.
- a `.field` property, returning the instantiated `FieldActionMap`/`FieldActionArray`/`FieldAction`;
- a `.f` property, as a shorthand to `self.field`;
- a `.__iter__(self)` method that yields, for each contained field, a tuple containing its path (as a tuple of names or indices) and its instantiated `FieldAction`. If only a single `Field` was supplied, its path is always `tuple()`.

## Drawbacks
[drawbacks]: #drawbacks

* A novice might not understand that it's possible to create a `Register` with multiple `Fields` if exposed only to code which uses single-`Field` `Register`s.
* There is a difference between a `Register` with one named/indexed `Field` and with one unnamed `Field`.
* Library code will be made more confusing by having `fields` refer to possibly only one field.
* Downstream code will have to deal with a third type of object returned by the
  `.f` property which behaves unlike the current two.
* BSP/doc generators will have to intelligently render a `Field` with an empty name and path (likely by omitting them, possibly by reusing the register's name).
    * An empty name is prohibited by `FieldActionMap`, so empty names are a novel
      problem.
    * The register does not know its own name independently of its container, so this may be difficult in practice.
* Adding a second `Field` to a register created this way will break all accesses to the `.f` property and likely all BSP users (though the actual bus transactions to the first `Field` will not change).
    * This could be especially entertaining if the new `Field` names happen to
      be the same as properties of the first `Field`'s `FieldAction`.
    * BSP users will have to change all constants/functions named after just the
      register to instead use the register + the field name.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

* Simple and easily explainable change, easy to migrate code to, though no
  migration is even necessary
* Minimal change to the library's existing code
* Nicely reduces redundant names, types, and properties
* Not strictly necessary, not having it will just make peripheral code a little
  more cluttered
* Users could instead be taught to eliminate the subclass by passing a dict to
  the `Register` constructor and assign the `.f` property of interest to a
  local variable to de-clutter code

## Prior art
[prior-art]: #prior-art

AVR microcontrollers have many registers where the field name is the same as the register name, though this is not always the case for single-field registers. Application code usually uses the register name in this case and ignores the field name.

STM32 microcontrollers are similar, but the field name is usually a suffix of the register name, as the field name is not expected to be globally unique. However, the generated BSP files do appear to always contain the field name at least somewhere.

Field arrays are already in some sense anonymous, being named by an integer instead of a string.

More experienced input is welcomed here.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

* ~~Should we change `fields` to `field` in the constructor and properties? This could break existing code.~~
    * The property will be `.field` and the constructor will be `fields=`.
* ~~Should we add `field` too? This would access the same object/s as `fields` and so would make available the option to use the right plurality, though it would be more code and wouldn't force it.~~
    * The property will be `.field` and the constructor will be `fields=`.
* ~~Should we force a `Register` with a single `Field` to always have that one be un-named? Would create backwards compatibility problems, but reduce ambiguity.~~
    * No, would cause backwards compatibility problems, which we don't want.
* ~~Should we go further? `Register` could take a single `FieldAction` and automatically wrap it in a `Field` (and inherit its access mode).~~
    * No, unnecessarily convenient and not yet proved necessary by experience.

## Future possibilities
[future-possibilities]: #future-possibilities

None obvious, this is a pretty small and self-contained idea.
