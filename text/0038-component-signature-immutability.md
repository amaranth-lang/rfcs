- Start Date: 2023-12-11
- RFC PR: [amaranth-lang/rfcs#38](https://github.com/amaranth-lang/rfcs/pull/38)
- Amaranth Issue: [amaranth-lang/amaranth#996](https://github.com/amaranth-lang/amaranth/issues/996)

# `Component.signature` immutability

## Summary
[summary]: #summary

Clearly define the contract for `amaranth.lib.wiring.Component.signature`: an `amaranth.lib.wiring.Signature` object is assigned in the constructor to the read-only property `.signature`.

## Motivation
[motivation]: #motivation

It is important that the signature of an interface object never change. `connect` relies on the signature to check that the connection being made is valid; if the signature changes after the connection is made (intentionally or accidentally), the design would silently break in a difficult to debug ways. For an ASIC this may mean a respin.

Right now, the guidance for the case where the default behavior of `Component.signature` is insufficient (i.e. for parametric components) and the generation of the signature must be customized, is to override the `signature` property, or to assign `self.signature =` in the constructor. This is clearly wrong: both of these approaches can easily result in the signature changing after construction of the component.

Moreover, at the moment, the implementation of the `Component.signature` property creates a new `Signature` object every time it is called. If the identity of the `Signature` object is important (i.e. if it has interior mutability, which is currently the case in Amaranth SoC), this is unsound. (It is unlikely, though not impossible, that this implementation would return an object with different members.)

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

To define a simple component whose signature does not change, use variable annotations:

```python
class SimpleComponent(wiring.Component):
    en: In(1)
    data: Out(8)
```

To define a component whose signature is parameterized by constructor arguments, call the superclass constructor with the signature that should be applied to the component:

```python
class ParametricComponent(wiring.Component):
    def __init__(self, data_width):
        super().__init__({
            "en": In(1),
            "data": Out(data_width)
        })
```

Do not override the `signature` property, as both Amaranth and third-party code relies on the fact that it is assigned in the constructor and never changes.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The constructor of `Component` is updated to take one argument: `def __init__(self, signature=None)`.
- If the `signature` argument is not provided, the signature is derived from class variable annotations, as in RFC 2, and assigned to an internal attribute.
- If the `signature` argument is provided, it is type-checked/type-cast and assigned to the internal attribute.
    - If a `Signature` is provided as a value, it is used as-is.
    - If a `dict` is provided, it is cast by calling `wiring.Signature(signature)`.
    - No other types are accepted.

If both the `signature` argument is provided and variable annotations are present, the constructor raises a `TypeError`. This is to guard against accidentally passing a `Signature` as an argument when constructing a component that is not parametric. (If this behavior is undesirable for some reason, it is always possible to implement a constructor that does not pass superclasses' constructor, and redefine the `signature` property, but this should be done as last resort only. We should cleary document that the `signature` property should return the exact same value at all times for any given `Component` instance.)

The `signature` property is redefined to return the value of this internal attribute.

No access to this attribute is provided besides the means above, and the name of the attribute is not defined in the documentation.

After assigning the internal attribute, the constructor creates the members as in RFC 2.

## Drawbacks
[drawbacks]: #drawbacks

None. This properly enforces an invariant that is already relied upon.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Some other behaviors were considered for `Component`: those which would made `signature` a class attribute, assigned in `__init_subclass__`. However, this option would leave the `signature` attribute mutable (as class-level properties are not reasonably supported in Python), which is undesirable, and significantly complicated the case of signatures with interior mutability.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
