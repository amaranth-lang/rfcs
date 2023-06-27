- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#2](https://github.com/amaranth-lang/rfcs/pull/2)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Interface definition library RFC


## Summary

Add standard ways of declaring that a component of a design conforms to a particular interface and connecting components with complementary interfaces together, to fill the other of the two major use cases of `Record` while avoiding its downsides.

See also [RFC #1](0001-aggregate-data-structures.md).


## Motivation

Digital designs are composed of densely packed components that communicate with each other using well-defined interfaces. Mechanisms to denote the boundary of a component, to ensure that a component complies to a specified interface, and to make reliable connections between components are essential.

Currently, Amaranth provides none of these mechanisms. A component implemented in Amaranth, however well-defined conceptually, has no more external structure than a loose collection of `Signal`s assigned to its attributes; and whether any one of them is a part of the interface or the implementation is up to a guess. Even when an interface is described using `amaranth.hdl.rec.Layout`, such a description cannot be used to verify even the simplest aspects of compliance, such as presence of fields. Although building components by composing smaller components together is ubiquitous, `amaranth.hdl.rec.Layout` is not able to compose their interface with the same ease. Connecting components using `amaranth.hdl.rec.Record.connect` is difficult enough that it sees very little use.

Originally, `Record` was aimed at solving many of these issues. However, it has multiple major drawbacks:

1. `Record` attempts to do too much: it is both a mechanism for _controlling representation_ (including implicitly casting a record to a value) and a mechanism for _defining interfaces_ (specifying signal directions and facilitating connections between records).

   These mechanisms should be defined separately, since the only aspect they have in common is using a container class that consists of multiple named fields. Conflating the two mechanisms constraints the design space, making addressing the other drawbacks impossible, and the ill-defined scope encourages bugs in downstream code.

2. `Record`'s model of signal directions is too complex. Because it attempts to model both aggregates with controlled representation and interfaces with defined directionality, every signal can have one of the three directions, the third option being non-directed. While this can be applied in a robust way--[FIRRTL](https://github.com/chipsalliance/firrtl-spec/releases/latest/download/spec.pdf) has only one aggregate type that it uses for both purposes--this gives rise to a large combination of features and requires handling many edge cases.

3. `Record`'s model of signal directions is too limited. The two static directionalities it has are the confusingly named "fanout" and "fanin", which really mean "from initiator to target" and "from target to initiator". This is insufficient to describe common, straightforward interactions such as two components exchanging streams of data across pairs of identical, complementary endpoints.

4. `Record`s are hard to customize. Records create and hold their signals, only providing the caller with an ability to place caller-created signals into individual fields. Signals often need adjustments: primarily setting a reset value or adding a decoder, but sometimes adding attributes or renaming. These adjustments must be performed at the record creation site, which is burdensome.

5. `Record` fields can be (apart from sub-records) only plain signals. In many cases, an interface between components carries structured data rather than opaque bit vectors. It is not possible to define inner structure for record fields other than through a sub-record, and using a sub-record for this means that an application-specific endpoint that defines such structure cannot be connected to a generic endpoint that does not.

6. `Record`s are hard to compose. The natural way to define a record is to call the `Record` constructor with a layout, but this creates the entire layout hierarchy unless parts of it are replaced; and it requires having the layout of the result in advance.

7. `Record.connect` determines the direction of data flow that it will create by the relative position of the interfaces being connected, with `x.connect(y)` and `y.connect(x)` having the oposite polarity of assignments. However, the direction of data flow is defined by the component that exposes the interface. Thus, every call of `Record.connect` can be done in one of the two very similar ways, one of which is always wrong.

8. `Record.connect` uses wired-OR to gather the "fanin" signals, a feature that exists so that it could be used to connect e.g. Wishbone endpoints together without additional gateware. The assumption that the response signals of inactive endpoints will remain all-zero is, generally, unsound.

9. `Record.connect` manages connections between interfaces with optional signals at the call site using an include/exclude mechanism. However, the semantics of the non-implemented optional signals are a property of the interface, not the connection.

10. `Record` and `rec.Layout` are often used as base classes. The `Record.like` facility, frequently used because of the poor ergonomics of `rec.Layout`, loses this information and returns an instance of the base class; `rec.Layout` does the same when indexed. As a result, there is little value in defining methods and attributes on the subclass, and `Record` subclasses are little more than a callable computing a layout.

11. Due to the limitations of `Record`, one might define a plain Python class that exposes compatible attributes. An instance of such a class cannot be compared to a known `rec.Layout` nor can it be embedded in another `Record`.

12. `Record` is value-castable and implements the `.eq()` protocol. Although useful when all fields are non-directional, using `.eq()` instead of `.connect()` when connecting directional interfaces is, generally, unsound. It also reserves commonly used names such as `any`, `all`, and `matches`, and implements arithmetic operations that are rarely if ever used on field containers.

13. `rec.Layout`'s DSL is very amorphous. It passes around variable length tuples. The second element of these tuples (the shape) can be another `rec.Layout`, which is neither a shape nor a shape-castable object.

Since these drawbacks are entrenched in the public API and make `Record` nearly useless for defining interfaces, a new mechanism must replace it.


## Outline of the design space

Although some HDLs and IRs (Migen, Chisel, FIRRTL, ...) choose to use the same basic aggregate data type to represent _structured data_ and _directional interfaces_, these mechanisms are in direct conflict. Complex forms of structured data, such as unions, are incompatible with associating directionality independently with every leaf member; and the non-directional nature of stored data requires complicated and error-prone rules when it can become a part of a directional connection.

Amaranth, instead, opts to include two superficially similar mechanisms for defining and accessing hierarchical aggregate data: `amaranth.lib.data` (RFC #1) and `amaranth.lib.component` (this RFC). `amaranth.lib.data` provides _data views_ that reinterpret bit containers as complex aggregates, and entirely avoids directionality. `amaranth.lib.component` provides _signatures_ that give a concrete shape to signals at component boundaries, and always treats them as directional.

When connections are made strictly between an output and a correspondingly named input, interfaces gain a dualistic nature: every connection is made between two interfaces whose port directions are the inverse of each other, and which are identical otherwise. To describe interfaces without repeating oneself, then, one has to pick an arbitrarily preferred directionality (and stick with it). Many interfaces are asymmetric, with data flowing from a source to a sink, or transactions issued from an initiator to a target. Amaranth picks the _source_ or _initiator_ perspective; an interface, examined in isolation, defines as outputs the signals that would be outputs of an initiator (and inputs of a target). Then, when an interface with true (non-flipped) directionality describes a component's output, the same interface with inverse (flipped) directionality symmetrically describes an input.

To eliminate the major usability issues with `Record.connect`, the interface connection mechanism assigns no precedence to interfaces and has no effect on signal directionality; whether a signal is an input or an output depends only on the interface itself. A connection is only made from an output to a matching input, and any other combination is rejected with a diagnostic. This way, connecting a pair of interfaces always leads to the same outcome, regardless of their order.

The choice to always treat interface signals as directional and to make their directionality dependent only on the interface itself leaves only one aspect of the design open: when and how interface directionality is flipped. The decisions that determine it affect both ergonomics and soundness. `Record.connect` in effect gives the programmer an option to flip directionality even when it would create an illegal connection. Conversely, `rec.Layout` provides no such affordance, even though it is necessary for composing components.

To facilitate composing components, the interface's directionality is flipped when it is used as an input, whether a top-level input of a component, or as a constituent of a larger interface. This way, the existing mechanism of annotating the directionality of an interface signal or a module port transparently handles interface composition.


## Guide-level explanation

Amaranth designs are made out of components (Python classes implementing `Elaboratable`) whose attributes include signals. These signals have directions: "in" signals are sampled by the component, while "out" signals are driven by it, or left at their reset value, and are provided to be sampled by other components.

At the moment, these directions are completely informal, and described in the documentation and/or in the signal name as an `i_` or `o_` prefix (to make it clearer what the direction is at the point of use, or to disambiguate the ports that would otherwise have identical name):

```python
class SequenceSource(Elaboratable):
    """
    Ports
    -----
    data : Signal(width), out
    ready : Signal(1), in
    valid : Signal(1), out
    """
    def __init__(self, width=16):
        self.data = Signal(width)
        self.ready = Signal()
        self.valid = Signal(reset=1)

    def elaborate(self, platform):
        m = Module()
        with m.If(self.ready):
            m.d.sync += self.data.eq(self.data + 1)
        return m
```

The `SequenceSource` component is implemented as a simple counter producing values for an *output stream* that is connected to some other component consuming them. On each clock tick, if the consumer is *ready*, it samples the *data* (the counter value), and simultaneously with that, the producer advances the data to the next item (the incremented value). Since there is always a next item available and it is ready for consumption on the next clock cycle, the stream always contains *valid* results.

> **Note**
> It is not, in general, possible to infer the directions of the signals from the implementation—here, `ready` and `valid` have different directions and different intended uses, but they look similar to the Amaranth implementation since they are both undriven in the component.

This RFC proposes a way of describing signal directions that can be applied to any Python object. In addition to elaboratables, it includes Python objects that are used to group together signals with a similar purpose, such as those that are parts of a bus.

To describe signal directions, only a single addition is needed: the `signature` property:

```python
from amaranth.lib.component import Signature, In, Out


class SequenceSource(Elaboratable):
    ...

    signature = Signature({
        "data": Out(16),
        "ready": In(1),
        "valid": Out(1)
    })
```

Consider another component that is consuming these values:

```python
class NumberSink(Elaboratable):
    ...

    def elaborate(self):
        m = Module()
        processing = Signal()
        m.d.comb += self.ready.eq(~processing)
        with m.If(self.valid & ~processing):
            m.d.sync += processing.eq(1)
        with m.Elif(processing):
            ... # process it somehow

    signature = Signature({
        "data": In(16),
        "ready": Out(1),
        "valid": In(1)
    })
```

Currently, the only way (given the tools provided by the language and the standard library) to connect the *output stream* of the `SequenceSource` to the *input stream* of the `NumberSink` is to do this signal-wise:

```python
m = Module()
m.submodules.source = source = SequenceSource()
m.submodules.sink = sink = NumberSink()
m.d.comb += [
    sink.data.eq(source.data),
    source.ready.eq(sink.ready),
    sink.valid.eq(source.valid)
]
```

This is tedious, verbose, and error-prone. It is possible to define an application-specific function abstracting this operation, and many applications do, but something this universal should be defined on the language level.

This RFC introduces a way to describe interfaces (collections of directional signals; more on this later) and a single operation: *connecting*. The code above now transforms into:

```python
from amaranth.lib.component import connect


m = Module()
m.submodules.source = source = SequenceSource()
m.submodules.sink = sink = NumberSink()
connect(m, sink, source)
```

The order of arguments to `connect` does not matter as the directionality is defined by the components themselves. It could just as well be written as:

```python
connect(m, source, sink)
```

However, this approach still has flaws. Most importantly, the signature for `SequenceSource` and `NumberSink` is written twice, but their `signature` is exactly the same except that the direction is flipped: `In` members become `Out`, and vice versa. To avoid error-prone repetition here, the signature can be defined once:

```python
Stream16BitSignature = Signature({
    "data": Out(16),
    "ready": In(1),
    "valid": Out(1)
})
```

and then used twice, for both the source and the sink:

```python
class SequenceSource(Elaboratable):
    ...

    signature = Stream16BitSignature

class NumberSink(Elaboratable):
    ...

    signature = Stream16BitSignature.flip()
```

The `Signature.flip()` method returns a _flipped signature object_: a signature object whose members have inverse direction but which is otherwise identical.

Since this approach has reusable signatures defined with a specific direction, it is necessary to make an arbitrary choice: pick the kind of object whose signature will use the non-flipped directions. This RFC picks the object that is the _source of data_ (for stream-like interfaces), the _transaction initiator_ (for bus-like interfaces), and so on to use non-flipped directions by convention.

Although some duplication was eliminated, some more remains: currently, it is necessary to define a stream signature for every kind of stream (a stream of 16-bit values, a stream of RGB colors, and so on). It is possible to define a reusable stream signature by inheriting from the `Signature` class:

```python
class StreamSignature(Signature):
    def __init__(self, payload_shape):
        return super().__init__({
            "payload": Out(payload_shape),
            "ready": In(1),
            "valid": Out(1)
        })
```

The elaboratables above can then be defined as:

```python
class SequenceSource(Elaboratable):
    ...

    signature = StreamSignature(16)

class NumberSink(Elaboratable):
    ...

    signature = StreamSignature(16).flip()
```

Usually, elaboratables have more than one interface. For example, a very simple DSP block could sink a stream of signed numbers, take their absolute value, and source a stream of unsigned numbers. It would then have a pair of `ready`, `valid`, and `payload` signals each: one for the input steam, and another for the output stream.

To handle this case, signature's members can be signatures themselves. These members also have directionality; an `Out` signature leaves the directionality of its members unchanged, while an `In` signature flips it. The signature method of the processing block could be defined as:

```python
class AbsoluteProcessor(Elaboratable):
    ...

    signature = Signature({
        "i": In(StreamSignature(signed(16))),
        "o": Out(StreamSignature(unsigned(16)))
    })
```

To be compatible with this signature, an `AbsoluteProcessor` instance must have an `i` attribute compatible with a `StreamSignature(signed(16)).flip()`, and an `o` attribute compatible with a `StreamSignature(unsigned(16))`. These could be defined manually:

```python
class AbsoluteProcessor(Elaboratable):
    def __init__(self):
        self.i = object()
        self.i.payload = Signal(signed(16))
        self.i.ready = Signal()
        self.i.valid = Signal()
        self.i.signature = StreamSignature(signed(16)).flip()

        self.o = object()
        self.o.payload = Signal(unsigned(16))
        self.o.ready = Signal()
        self.o.valid = Signal()
        self.o.signature = StreamSignature(unsigned(16))

    ...
```

Once more, to reduce error-prone repetition, the `Signature` class offers a way to define objects just like the ones created above, making the complete definition be:

```python
class AbsoluteProcessor(Elaboratable):
    def __init__(self):
        self.i = StreamSignature(signed(16)).flip().create()
        self.o = StreamSignature(unsigned(16)).create()

    signature = Signature({
        "i": In(StreamSignature(signed(16))),
        "o": Out(StreamSignature(unsigned(16)))
    })

    ...
```

`Signature` subclasses can also override the `create` method to add functionality not present in the base class. For example, a signature for a bus such as Wishbone or AXI could return an instance of a class rather than a simple `object()`, and include attributes indicating which optional features of the bus are enabled.

However, since the interface of `AbsoluteProcessor` as a whole can itself be described as a signature, it is possible to further shorten it by deriving from `component.Component` instead of `Elaboratable`, in which case the attributes will be filled in from the signature automatically:

```python
from amaranth.lib.component import Component


class AbsoluteProcessor(Component):
    signature = Signature({
        "i": In(StreamSignature(signed(16))),
        "o": Out(StreamSignature(unsigned(16)))
    })

    def elaborate(self):
        m = Module()
        with m.If(self.i.payload > 0):
            m.d.comb += self.o.payload.eq(self.i.payload)
        with m.Else():
            # Does not overflow, since -(-32768) [least signed(16)] is less
            # than 65536 [greatest unsigned(16)].
            m.d.comb += self.o.payload.eq(-self.i.payload)
        return m
```

Python variable annotations can also be used in cases like the above, where the signature is the same for every instance of the class (i.e. the component is not parameterized during creation):

```python
class AbsoluteProcessor(Component):
    i: In(StreamSignature(signed(16)))
    o: Out(StreamSignature(unsigned(16)))

    def elaborate(self):
        m = Module()
        with m.If(self.i.payload > 0):
            m.d.comb += self.o.payload.eq(self.i.payload)
        with m.Else():
            # Does not overflow, since -(-32768) [least signed(16)] is less
            # than 65536 [greatest unsigned(16)].
            m.d.comb += self.o.payload.eq(-self.i.payload)
        return m
```

All of the import statements in the code examples above can be replaced with:

```python
from amaranth.lib.component import *
```


## Reference-level explanation

This RFC proposes a number of library additions:

* Adding classes that describe a hierarchy of Amaranth objects (an elaboratable object and the objects containing its interface signals) and ease instantiating such hierarchies.
* Adding a function that connects such hierarchies to each other.

It also introduces a number of technical terms:

* A _component_ is an Amaranth elaboratable object.
* An _interface_ (a concept) is a shared boundary across which several Amaranth components exchange data. It is comprised of a set of signals and the invairants that govern their use.
* An _interface object_ (an implementation of the concept) a Python object that includes:
  1. attributes whose value is an Amaranth value-castable, or another interface;
  2. a `signature` attribute whose value is a _signature_ that is _compatible_ with this object;
  3. a description of the invariants applying to its use (in form of documentation, testbenches, formal tests, etc.).
* A _signature_ is a `Signature` instance describing requirements applicable to a hierarchy of interace objects.
* A _signature member_ is a `Member` instance describing requirements applicable to a single attribute of an interface object. Two kinds of signature members exist: port members (requiring the value of the attribute to be a `Signal`), and interface members (requiring the value of the attribute to be another interface object).
* A signature is _compatible_ with an object (therefore making it an interface object) if every member of the signature object corresponds to an attribute of the object whose value fits the requirements.

A single elaboratable object will often have several interfaces; e.g. a peripheral can have a CSR and/or Wishbone bus interface, and a pin interface. However, the elaboratable object itself can be an interface object as well, which makes it easy to convert it to Verilog and use standalone since its signature defines the ports the Verilog module needs to have.


### Interface description

Interfaces are described using an enumeration, `amaranth.lib.component.Flow`, and two classes, `amaranth.lib.component.Member` and `amaranth.lib.component.Signature`:

* `Flow` is an enumeration with two values, `In` and `Out`.

  * `Flow.__call__(arg, **kwargs)` forwards to `Member(self, arg, **kwargs)`.
    * Thus, `Out(unsigned(16), reset=0x1234)` is a shorthand for `Member(Flow.Out, unsigned(16), reset=0x1234)`.
  * `flow.flip()` flips the value from `In` to `Out` and back.

* A `Member(flow, ...)` object describes a part of an interface. It is immutable.

  * A `Member(flow, shape_castable, reset=reset_value)` object describes a port with the given shape and flow direction. The returned `Member` object has:
    * the `.flow` property be `flow`;
    * the `.is_port` property be `True`;
    * the `.is_signature` property be `False`;
    * the `.shape` property be `shape_castable`;
    * the `.reset` property be `reset_value`;
    * the `.signature` property raise `TypeError`;
    * the `.dimensions` property be `()`.
  * A `Member(flow, signature)` object describes a constituent interface with the given flow direction. If `flow` is `Flow.In`, then the actual flow of every port recursively described by `signature` is the reverse of the stated direction. The returned `Member` object has:
    * the `.flow` property be `flow`;
    * the `.is_port` property be `False`;
    * the `.is_signature` property be `True`;
    * the `.shape` property raise `TypeError`;
    * the `.reset` property raise `TypeError`;
    * the `.signature` property return `signature`;
    * the `.dimensions` property be `()`.
  * `member.array(*dimensions)` returns a new `Member` object whose `.dimensions` property is `dimensions`, which is any amount of non-negative numbers, and all other properties are the same as those of `member`.
  * `member.flip()` returns a new `Member` object whose `.flow` property is `~member.flow`, and all other properties are the same as those of `member`.

* A `Signature(...)` object describes an interface comprised of named members: ports and nested interfaces (which themselves are described using signature objects).

  The `Signature` class can be derived from. Instances of `Signature` itself are termed _anonymous signatures_, and instances of derived classes are _named signatures_.

  * A `Signature({"name": Member(...)})` object can be constructed from a name to member mapping.
  * `signature.members` is a mutable mapping that can be used to alter the description of a non-frozen signature.
    * `signature.members += {...}` adds members from the given mapping to `signature.members` if the names being added are not already used. Raises `NameError` otherwise.
  * `signature.freeze()` prevents any further modifications of `signature.members`, enabling the caller to rely on a particular layout. It is applied recursively to constituent interfaces.
  * `signature.__eq__()` compares:
    * anonymous signatures, which are equal when the members and their names compare equal;
    * named signatures, which are equal only to themselves (the same signature object), unless overridden in a derived class.
  * `signature.__iter__()` yields `path` recursively for every member and sub-member. A member's path is a tuple containing every name in the chain of attribute accesses required to reach the member. Members are yielded in an ascending lexicographical order. An interface member's path is yielded before the paths of its sub-members are.
  * `signature.__getitem__(*path)` looks up a member by its path. The flow of the member is flipped as many times as there are `In` signatures between the topmost signature and the signature of the member.
  * `signature.flip()` returns a signature where every member is `member.flip()`ped. The exact object returned is a proxy object that overrides `__iter__` and `__getitem__` to flip the direction of members, and otherwise forwards attribute accesses untouched. That is, `signature.x = <value>` and `signature.flip().x = <value>` both define an attribute on the original `signature` object, and never on the proxy object alone.
  * `signature.compatible(object)` checks whether an arbitrary Python object is compatible with this signature. To be compatible with a signature:
    - for every member of the signature, the object must have a corresponding attribute
    - if the member is a port, the attribute value must be a value-castable such that `Value.cast(object.attr)` method returns a `Signal` or a `Const` that has the same width and signedness, and for signals, is not reset-less and has the same reset value as the member
      - a warning may be emitted if the `.shape` of the member and the `.shape()` of `object.attr` are not equal
    - if the member is an interface, the attribute value must be compatible with the signature of the member
    - if the member's `dimensions` are `(p, q, ...)`, the requirements below hold instead for every result of indexing the attribute value with `[i][j]...` where `i in range(p)`, `j in range(q)`, ...
  * `signature.create_members()` freezes this signature and creates a dictionary of members from it. This is a helper method that is essentially the part of `create()` that subclasses are unlikely to need to override. For every member of the signature, the dictionary contains a value equal to:
    * If the member is a port, `Signal(member.shape, reset=member.reset)`.
    * If the member is a signature, `member.signature.create()` for `Out` members, and `member.signature.flip().create()` for `In` members.
  * `signature.create()` creates an interface object from this signature. To do this, it creates a fresh `object()` and replaces its dictionary with the result of `signature.create_members()`.  This method is expected to be routinely overridden in `Signature` subclasses to perform actions specific to a particular signature.


### Interface connection

Interface objects may be connected to each other using the `amaranth.lib.component.connect(m, first, *rest)` free function.

This function connects interface objects that satisfy the following requirements:
* For every port member with a given `path` and `first_shape` in `first`, there is a port member with the same `path` in each of `rest` with shape `rest_shape`, where `Shape.cast(first_shape) == Shape.cast(rest_shape)`, and there are no other members in any of `rest`. In other words, the width and signedness of all of port members must be equal, but the shape-castable objects specifying the width and signedness do not have to be.
  * It is expected that any mismatch in signatures will be resolved (through wrappers, mutating signature members, or otherwise) before interfaces are being connected.
* Either:
  1. There is exactly one interface object in `rest`, and for every pair of members `first_member` and `rest_member`, `first_member.flow == ~rest_member.flow`.
  2. There is any amount of interfaces in `rest`, and for every pair of members `first_member` and `rest_member`, `first_member.flow == Out` and `rest_member.flow == In`.

In both of the cases (1) and (2), the function, for each pair of an input port and an output of port with the same path, the function connects these as follows:
```python
m.d.comb += output_port.eq(input_port)
```


### Interface forwarding

In some cases, an outer elaboratable object creates an inner elaboratable object and _forwards_ an interface of the inner object like this:

```python
class Outer(Component):
    bus: BusSignature()

    def __init__(self):
        super().__init__()

        self.inner = Inner()

    def elaborate(self, platform):
        m = Module()
        m.d.comb += [
            self.inner.bus.addr.eq(self.bus.addr),
            self.inner.bus.w_data.eq(self.bus.w_data),
            self.bus.r_data.eq(self.inner.bus.r_data),
            # ...
        ]
        return m


class Inner(Component):
    bus: BusSignature()

    ...
```

In this case, `amaranth.lib.component.connect(...)` won't help, since an output needs to be connected to an output, and an input to an input.

An additional function `amaranth.lib.component.forward(obj)` is added to assist in this case. It returns a proxy object `obj_forward` where `obj_forward.signature` equals `obj.signature.flip()`, and everything else is forwarded identically otherwise. So, the `Outer.elaborate` method can be rewritten as:

```python
class Outer(Component):
    bus: BusSignature()

    def __init__(self):
        super().__init__()

        self.inner = Inner()

    def elaborate(self, platform):
        m = Module()
        component.connect(m, component.forward(self.bus), self.inner.bus)
        return m
```


### Component definition

This RFC in effect introduces a particular kind of elaboratable object: one that has a signature. While connecting an elaboratable as a whole (as opposed to its sub-interfaces) will rarely, if ever, happen, it is still convenient to have an elaboratable define its signature, for three reasons:
1. It is a declaration of intent, separating the signals that are purposefully a part of its interface from ones that just happen to be assigned to attributes, and stating their direction;
2. It simplifies and standardizes assignment of the interface attributes, making the `signature` property the single source of truth for the module's interface;
3. It makes it easy to convert a single standalone elaboratable to Verilog.

To this end, a class `amaranth.lib.component.Component` is introduced:
* `Component.__init__` (typically called as `super().__init__()`) updates `self.__dict__` with the result of `self.signature.create_members()`.
  * TBD: what to do in case of a name conflict? abort (then to override, `super().__init__` call will be in the beginning) or ignore (then to override, `super().__init__` call will be in the end)
* `Component.signature` collects PEP 526 variable annotations in the class, if any, and returns a signature object constructed from these, or raises an error otherwise. The signature object is created per-instance, not per-class, so that it can be safely mutated if this is a part of the workflow.


## Alternatives and rationale

- Do nothing. `Record` will continue to be used alongside the continued proliferation of ad-hoc implementations of similar functionality, and continue to impair the use of Amaranth components together.

- Replace the `interface.lib.component.connect` free function with a function `amaranth.hdl.dsl.Module.connect`.
  * It is not a function on `amaranth.hdl.dsl.Module` to avoid privileging the standard interface library over any other library that may be written downstream. At the moment nothing in `amaranth.lib` is special in any way other than its name, and preserving this is valuable to the author.


## Naming questions

- Should `Signature` be called `Interface`?
  - The object returned by `Signature.create()` is an interface, not the signature itself
- Should `Signature.compatible` be named something else, like `Signature.is_implemented`?
- Should `component.forward` be named something else, like `component.evert` or `component.flip`?


## Future work

* One-to-many connections between interfaces are currently provided only with a fan-out topology: a single interface with output members only can be connected with multiple interfaces with input members only. This avoids the question of what to do with an input that must be driven by multiple outputs. The interface library could be enriched by adding a small amount of fixed fan-in topologies, e.g. wired-OR and wired-AND, specified as a `Member()` constructor parameter that must match between all of the respective members.
