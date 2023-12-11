- Start Date: 2023-12-11
- RFC PR: [amaranth-lang/rfcs#37](https://github.com/amaranth-lang/rfcs/pull/37)
- Amaranth Issue: [amaranth-lang/amaranth#995](https://github.com/amaranth-lang/amaranth/issues/995)

# Make `Signature` immutable

## Summary
[summary]: #summary

Remove mutability from `amaranth.lib.wiring.Signature`.

## Motivation
[motivation]: #motivation

At the time of writing, `Signature` allows limited mutability: members can be added, but not removed or changed; and a signature may be frozen, preventing further mutation.

The intent behind this feature was unfortunately never explicitly described. I (Catherine) came up with it, to cover the following case: suppose an interface without certain features needs to be connected to an interface with such features. For example, a Wishbone initiator that does not handle errors needs to be connected to a Wishbone decoder that does. In this case, a method on the Wishbone initiator's interface could be used to add a dummy `err` output, which will always remain at its reset value, zero.

This intent was never realized as the feature was never actually used by Amaranth SoC. In addition, it turned out to be problematic when combined with variable annotations. Consider this class definition:

```py
class SomeInitiator(wiring.Component):
    bus: Out(wishbone.Interface(...))
```

In this case, only one instance of `wishbone.Interface` is created. Mutating this instance would be unsound, since it would affect every instance of the component and not just the one for that particular one, and when this causes issues this would be very difficult to debug.

The presence of this feature encouraged adding other mutable objects to `Signature` subclasses, such as memory maps. That is also unsound, for similar reasons. Because memory maps in Amaranth SoC can also be frozen, considerable additional complexity was introduced since piecewise freezing was now possible.

Because freezing was defined as a property of the signature members dictionary, overloading `Signature.freeze` was meaningless: it would be possible, in rare but legal cases, to have a signature frozen without `freeze` being called for that signature, leaving any additional objects that should have been frozen mutable.

The `wiring.connect` function also supports constants as both inputs and outputs, where a constant input can be connected to a constant output provided their values match. This behavior is also suited for implementing optional features (where the absence of an optional feature means the corresponding ports are fixed at a constant value), and does not pose any hazards.

## Guide-level and reference-level explanation
[guide-level-explanation]: #guide-level-explanation

The `SignatureMembers.freeze`, `SignatureMembers.frozen`, `Signature.freeze`, `Signature.frozen`, `FlippedSignatureMembers.freeze`, `FlippedSignatureMembers.frozen`, `FlippedSignature.freeze`, `FlippedSignature.frozen`, `SignatureMembers.__iadd__`, `SignatureMembers.__setitem__`, `FlippedSignatureMembers.__iadd__`, `FlippedSignatureMembers.__setitem__` methods and properties are removed.

`SignatureMembers` becomes an immutable mapping.

`Signature` becomes immutable. Subclasses of `Signature` are required to ensure any additional methods or properties do not allow mutation.

## Drawbacks
[drawbacks]: #drawbacks

The author is not aware of anyone actually using this feature.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An alternative to this proposal would be to automatically freeze any signature that is used in `wiring.Component` variable annotations. This does not address similar hazards, such as the case of a user-defined constant `SomeSignature = Signature({...})`, which is also a legitimate way to define a signature that does not require parameterization. It also would not address hazards associated with interior mutability.

## Prior art
[prior-art]: #prior-art

None seems applicable. Mutation of interface definitions is uncommon in first place (excluding languages where everything is mutable, like Python).

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should *all* interior mutability be prohibited? At the moment it is not completely clear where memory maps should be attached, and requiring no interior mutability would mean it cannot be `Signature` no matter what. Interior mutability could be left to specific `Signature` subclasses instead on an experimental basis, and prohibited later if it turns out to be a bad idea.

## Future possibilities
[future-possibilities]: #future-possibilities

Reintroducing mutability of `Signature` after it has been removed will be unfeasible due to expectation of immutability being baked in widely in downstream code. Once we commit to this RFC we will have to commit to it effectively forever.