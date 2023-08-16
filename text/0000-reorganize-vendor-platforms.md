- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#18](https://github.com/amaranth-lang/rfcs/pull/18)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Reorganize vendor platforms

## Summary
[summary]: #summary

Update `amaranth.vendor` namespace so that instead of:

```python
from amaranth.vendor.lattice_ecp5 import LatticeECP5Platform
```

you would write:

```python
from amaranth.vendor import LatticeECP5Platform
```


## Motivation
[motivation]: #motivation

Vendor names are ever-changing. Xilinx was bought by AMD and the brand has been phased out. Altera was bought by Intel and the brand has been phased out. SiliconBlue has been bought by Lattice (a long time ago) and the brand has *long* been phased out but still remains as "SB" in `SB_LUT` primitive name.

In addition, we attempt to group FPGA families into a single file, like `vendor.lattice_machxo2_3l` that has been renamed from `vendor.lattice_machxo2`. This will likely include another FPGA family as soon as it becomes available.

By tying module (and file) names to brand names we create churn. Every Amaranth release so far has included renaming of both platform class names and module names. This causes additional downstream breakage and annoys designers using Amaranth.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

To target your FPGA-based project for a particular FPGA family, import the platform class corresponding to the FPGA family from `amaranth.vendor`, e.g.:

```python
from amaranth.vendor import LatticeECP5Platform
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

All of the `amaranth.vendor.name` modules are renamed to `amaranth.vendor._internal_name`.

Python allows `__getattr__` to be present in modules:

    $ cat >x.py
    def __getattr__(self, name):
        return f"__getattr__({name!r})"
    $ python
    >>> from x import abc
    >>> abc
    "__getattr__('abc')"

This allows us to make all the platform classes be present as-if they were defined in the `amaranth.vendor` modules, while retaining all of the benefits of having them in their own `amaranth.vendor._internal_name` module, such as lazy loading.

## Drawbacks
[drawbacks]: #drawbacks

- Churn.
- A somewhat unusual loading mechanism could cause confusion.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Decoupling marketing/brand names from technical names is increasingly important as Amaranth evolves and supports more FPGA families. It allows us to maintain any internal hierarchy we want without it having any impact on downstream code, which solely operates on names imported from `amaranth.vendor`.

## Prior art
[prior-art]: #prior-art

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
