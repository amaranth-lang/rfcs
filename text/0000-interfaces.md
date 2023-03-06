- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Interface definition library RFC


## Summary

Add standard ways of declaring that a component of a design conforms to a particular interface and connecting components with complementary interfaces together, to fill the other of the two major use cases of `Record` while avoiding its downsides.

See also #1.


## Motivation

Digital designs are composed of densely packed components that communicate with each other using well-defined interfaces. Mechanisms to denote the boundary of a component, to ensure that a component complies to a specified interface, and to make reliable connections between components are essential.

Currently, Amaranth provides none of these mechanisms. A component implemented in Amaranth, however well-defined conceptually, has no more external structure than a loose collection of `Signal`s assigned to its attributes; and whether any one of them is a part of the interface or the implementation is up to a guess. Even when an interface is described using `amaranth.hdl.rec.Layout`, such a description cannot be used to verify even the simplest aspects of compliance, such as presence of fields. Although building components by composing smaller components together is ubiquitous, `amaranth.hdl.rec.Layout` is not able to compose their interface with the same ease. Connecting components using `amaranth.hdl.rec.Record.connect` is difficult enough that it sees very little use.

Originally, `Record` was aimed at solving many of these issues. However, it has multiple major drawbacks:

  1. `Record` attempts to do too much: it is both a mechanism for _controlling representation_ (including implicitly casting a record to a value) and a mechanism for _defining interfaces_ (specifying signal directions and facilitating connections between records).

     These mechanisms should be defined separately, since the only aspect they have in common is using a container class that consists of multiple named fields. Conflating the two mechanisms constraints the design space, making addressing the other drawbacks impossible, and the ill-defined scope encourages bugs in downstream code.

  2. `Record`'s model of signal directions is too complex. Because it attempts to model both aggregates with controlled representation and interfaces with defined directionality, every signal can have one of the three directions, the third option being non-directed. While this can be applied in a robust way--[FIRRTL](https://github.com/chipsalliance/firrtl-spec/releases/latest/download/spec.pdf) has only one aggregate type that it uses for both purposes--this gives rise to a large combination of features and requires handling many edge cases.

  3. `Record`'s model of signal directions is too limited. The two static directionalities it has are the confusingly named "fanout" and "fanin", which really mean "from initiator to target" and "from target to initiator". This is insufficient to describe common, straightforward interactions such as two components exchanging streams of data across pairs of identical, complementary endpoints.

  4. `Record`s are hard to customize. Records create and hold their signals, only providing the caller with an ability to place caller-created signals into individual fields. Signals often need adjustments: changing name, adding a decoder, attaching attributes, setting a reset value or making them reset-less. These adjustments are very burdensome, prohibitively so for nested records.

  5. `Record` fields can be (apart from sub-records) only plain signals. In many cases, an interface between components carries structured data rather than opaque bit vectors. It is not possible to define inner structure for record fields other than through a sub-record, and using a sub-record for this means that an application-specific endpoint that defines such structure cannot be connected to a generic endpoint that does not.

  6. `Record`s are hard to compose. The natural way to define a record is to call the `Record` constructor with a layout, but this creates the entire layout hierarchy unless parts of it are replaced; and it requires having the layout of the result in advance.

  7. `Record.connect` determines the direction of data flow that it will create by the relative position of the interfaces being connected, with `x.connect(y)` and `y.connect(x)` having the oposite polarity of assignments. However, the direction of data flow is defined by the component that exposes the interface. Thus, every call of `Record.connect` can be done in one of the two very similar ways, one of which is always wrong.

  7. `Record.connect` uses wired-OR to gather the "fanin" signals, a feature that exists so that it could be used to connect e.g. Wishbone endpoints together without additional gateware. The assumption that the response signals of inactive endpoints will remain all-zero is, generally, unsound.

  8. `Record.connect` manages connections between interfaces with optional signals at the call site using an include/exclude mechanism. However, the semantics of the non-implemented optional signals are a property of the interface, not the connection.

  9. `Record` and `rec.Layout` are often used as base classes. The `Record.like` facility, frequently used because of the poor ergonomics of `rec.Layout`, loses this information and returns an instance of the base class; `rec.Layout` does the same when indexed. As a result, there is little value in defining methods and attributes on the subclass, and `Record` subclasses are little more than a callable computing a layout.

  10. Due to the limitations of `Record`, one might define a plain Python class that exposes compatible attributes. An instance of such a class cannot be compared to a known `rec.Layout` nor can it be embedded in another `Record`.

  11. `Record` is value-castable and implements the `.eq()` protocol. Although useful when all fields are non-directional, using `.eq()` instead of `.connect()` when connecting directional interfaces is, generally, unsound. It also reserves commonly used names such as `any`, `all`, and `matches`, and implements arithmetic operations that are rarely if ever used on field containers.

  12. `rec.Layout`'s DSL is very amorphous. It passes around variable length tuples. The second element of these tuples (the shape) can be another `rec.Layout`, which is neither a shape nor a shape-castable object.

Since these drawbacks are entrenched in the public API and make `Record` nearly useless for defining interfaces, a new mechanism must replace it.


## Outline of the design space

Although some HDLs and IRs (Migen, Chisel, FIRRTL, ...) choose to use the same basic aggregate data type to represent _structured data_ and _directional interfaces_, these mechanisms are in direct conflict. Complex forms of structured data, such as unions, are incompatible with associating directionality independently with every leaf member; and the non-directional nature of stored data requires complicated and error-prone rules when it can become a part of a directional connection.

Amaranth, instead, opts to include two superficially similar mechanisms for defining and accessing hierarchical aggregate data: `amaranth.lib.data` (RFC issue #693) and `amaranth.lib.intf` (this RFC). `amaranth.lib.data` provides _data views_ that reinterpret bit containers as complex aggregates, and entirely avoids directionality. `amaranth.lib.intf` provides _signatures_ that give a concrete shape to signals at component boundaries, and always treats them as directional.

When connections are made strictly between an output and a correspondingly named input, interfaces gain a dualistic nature: every connection is made between two interfaces whose port directions are the inverse of each other, and which are identical otherwise. To describe interfaces without repeating oneself, then, one has to pick an arbitrarily preferred directionality (and stick with it). Many interfaces are asymmetric, with data flowing from a source to a sink, or transactions issued from an initiator to a target. Amaranth picks the _source_ or _initiator_ perspective; an interface, examined in isolation, defines as outputs the signals that would be outputs of an initiator (and inputs of a target). Then, when an interface with true (non-flipped) directionality describes a component's output, the same interface with inverse (flipped) directionality symmetrically describes an input.

To eliminate the major usability issues with `Record.connect`, the interface connection mechanism assigns no precedence to interfaces and has no effect on signal directionality; whether a signal is an input or an output depends only on the interface itself. A connection is only made from an output to a matching input, and any other combination is rejected with a diagnostic. This way, connecting a pair of interfaces always leads to the same outcome, regardless of their order.

The choice to always treat interface signals as directional and to make their directionality dependent only on the interface itself leaves only one aspect of the design open: when and how interface directionality is flipped. The decisions that determine it affect both ergonomics and soundness. `Record.connect` in effect gives the programmer an option to flip directionality even when it would create an illegal connection. Conversely, `rec.Layout` provides no such affordance, even though it is necessary for composing components.

To facilitate composing components, the interface's directionality is flipped when it is used as an input, whether a top-level input of a component, or as a constituent of a larger interface. This way, the existing mechanism of annotating the directionality of an interface signal or a module port transparently handles interface composition.


## Detailed design

This RFC proposes a number of library additions:

  * ?????


### Interface descriptions

Interfaces are described using an enumeration, `amaranth.lib.intf.Flow`, and two classes, `amaranth.lib.intf.Member` and `amaranth.lib.intf.Signature`:

  * `Flow` is an enumeration with two values, `In` and `Out`.

    * `Flow.__getitem__(arg)` forwards to `Member(self, arg)`; thus, `Out[foo]` is a shorthand for `Member(Flow.Out, foo)`.
    * `~flow` flips the value from `In` to `Out` and back.

  * A `Member(flow, ...)` object describes a part of an interface. ~It is immutable.~

    * A `Member(flow, shape_castable)` object describes a port with the given shape and flow direction.
    * A `Member(flow, signature)` object describes a constituent interface with the given flow direction. If `flow` is `Flow.In`, then the actual flow of every port recursively described by `signature` is the reverse of the stated direction.

  * A `Signature(...)` object describes an interface comprised of named members: ports and nested interfaces (which themselves are described using signature objects).

    The `Signature` class can be derived from. Instances of `Signature` itself are termed _anonymous signatures_, and instances of derived classes are _named signatures_.

    * A `Signature({"name": Member(...)})` object can be constructed from a name to member mapping.
    * `signature.members` is a mutable mapping that can be used to alter the description of a ~non-frozen~ signature.
      * `signature.members += {...}` adds members from the given mapping to `signature.members` if the names being added are not bound. Raises `NameError` otherwise.
    * ~`signature.freeze()` prevents any further modifications of `signature.members`, enabling the caller to rely on a particular layout. It is applied recursively to constituent interfaces.~
    * `signature.__eq__()` compares:
      * anonymous signatures, which are equal when the members and their names compare equal;
      * named signatures, which are equal only to themselves, unless overridden in a derived class.
    * `signature.__iter__()` yields `(path, member)` pairs recursively for every member. A member's path is a tuple containing every name required to access the member. The flow of the member is flipped as appropriate. Members are yielded in an order sorted by their name. An interface member is yielded before its sub-members are.
    * `signature.__getitem__(path)` looks up a member by its path. The flow of the member is flipped as appropriate.

Interfaces may be instantiated using the `amaranth.lib.intf.Interface` class:

  * An `Interface(...)` object implements an interface described by a signature and comprised of named members: ports and nested interfaces.

    * An `Interface(signature)` object can be constructed from a signature.
    * An `Interface({"name": Member(...)})` object can be constructed from a name to member mapping, which will be converted to an anonymous signature first.
    * `Interface.__init__(self, ...)` instantiates signature members and assigns them to correspondingly named attributes on `self`.
