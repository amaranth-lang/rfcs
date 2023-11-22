- Start Date: 2023-02-14
- RFC PR: [amaranth-lang/rfcs#1](https://github.com/amaranth-lang/rfcs/pull/1)
- Amaranth Issue: [amaranth-lang/amaranth#748](https://github.com/amaranth-lang/amaranth/issues/748)

# Aggregate data structure library

> **Amendments**
> The behavior described in this RFC was updated by [RFC #8](0008-aggregate-extensibility.md), [RFC #9](0009-const-init-shape-castable.md), and [RFC #15](0015-lifting-shape-castables.md).

## Summary

Add a rich set of standard library classes for accessing hierarchical aggregate data an idiomatic way, to fill one of the two major use cases of `Record` while avoiding its downsides.

See also [RFC #2](0002-interfaces.html).


## Motivation

Amaranth has a single data storage primitive: `Signal`. A `Signal` can be referenced on the left hand and the right hand side of an assignment, and so can be its slices. Although a signal serves the role of a numeric and bit container type fine, designs often include signals used as bit containers whose individual bits are named and have unique meanings. A mechanism that allows referring to such bit fields by name is essential.

Currently, the role of this mechanism is served by `Record`. However, `Record` has multiple major drawbacks:

  1. `Record` attempts to do too much: it is both a mechanism for _controlling representation_ (including implicitly casting a record to a value) and a mechanism for _defining interfaces_ (specifying signal directions and facilitating connections between records).

     These mechanisms should be defined separately, since the only aspect they have in common is using a container class that consists of multiple named fields. Conflating the two mechanisms constraints the design space, making addressing the other drawbacks impossible, and the ill-defined scope encourages bugs in downstream code.

  2. `Record` has limited composability: records can only be nested within each other. Practical designs (in particular, implementations of protocols) use data with complex representation beyond nested sequences of fields: these include overlaid sequences of fields (where interpretation alternates based on some discriminant) and arrays of fields (where a field can be indexed by a non-constant), where any individual field can have a complex representation itself.

     `Record` is structured as a sequence of `Signal`s, which is a part of its API. As such, it cannot support overlaid fields, and implementing support for arrays of fields is challenging.

  3. `Record` has limited introspectability: while its `layout` member can be accessed to enumerate its fields, the results do not include field boundaries, and the types of the returned shape-castable objects are preserved only as an implementation detail. `Layout` objects themselves are also not shape-castable.

     `Record` and `Layout` are structured as a sequence of `Signal`s rather than a view into an underlying bit container, which is reflected in its API. Thus, `Layout` does not fit into Amaranth's data model, which concerns individual values.

  4. `Record` comes with its own storage: while its `fields` argument can be used to substitute the signals that individual fields are pointing to (in an awkward and error-prone way), it is still a collection of `Signal`s. Using `Record` to impose structure on an existing `Value` requires a `Module` and a combinatorial assignment. This is an unnecessary complication, especially in helper functions.

  5. `Record` does not play well with Python's type annotations. Amaranth developers often inherit from `Record` as well as `Layout`, but in both cases the class definition syntax is usually little more than a way to define a callable returning a `Record` with a specific layout, and provides no benefits for IDE users.

  6. `Record` reserves a lot of names, including commonly used names like `connect`, `any`, `all`, and `matches`. Conversely, it defines a lot of arithmetic methods that are rarely if ever used on field containers.

  7. `Layout`'s DSL is very amorphous. It passes around variable length tuples. The second element of these tuples (the shape) can be another `Layout`, which is neither a shape nor a shape-castable object.

  8. Neither `Record` nor `Layout` allow defining fields whose shapes are arbitrary `ShapeCastable` classes.

Since these drawbacks are entrenched in the public API and heavily restrict usefulness of `Record` as a mechanism for specifying data representation, a new mechanism must replace it.


## Overview and examples

This section shows a bird's eye view of the new syntax and behavior proposed in this RFC. The detailed design is described afterwards.

```python
from amaranth import *
from amaranth.lib import data


# Declaring a simple structure:
class Float32(data.Struct):
    fraction: unsigned(23)
    exponent: unsigned(8)
    sign: unsigned(1)


# Defining a signal with the structure above:
flt_a = Float32()

# Reinterpreting an existing value with the same structure:
flt_b = Float32(Const(0b00111110001000000000000000000000, 32))

# Referencing and updating structure fields by name:
with m.If(flt_b.fraction > 0):
    m.d.comb += [
        flt_a.sign.eq(1),
        flt_a.exponent.eq(127)
    ]


# Declaring a simple union, referencing an existing structure:
class FloatOrInt32(data.Union):
    float: Float32
    int: signed(32)


# Using the union to bitcast an IEEE754 value from an integer:
f_or_i = FloatOrInt32()
is_sub_1 = Signal()
m.d.comb += [
    f_or_i.int.eq(0x41C80000),
    is_sub_1.eq(f_or_i.float.exponent < 127) # => 1
]


class Op(enum.Enum):
  ADD = 0
  SUB = 1


# Programmatically declaring a structure layout:
adder_op_layout = data.StructLayout({
    "op": Op,
    "a": Float32,
    "b": Float32
})

# Using the layout defined above to define appropriately sized storage...
adder_op_storage = Signal(adder_op_layout)
len(adder_op_storage) # => 65

# ... and wrap it for the fields to be accessible.
adder_op = data.View(adder_op_layout, adder_op_storage)
m.d.comb += [
    adder_op.op.eq(Op.SUB),
    adder_op.a.eq(flt_a),
    adder_op.b.eq(flt_b)
]
```


## Detailed design

This RFC proposes a number of language and library additions:

  * Adding a `ShapeCastable` interface, similar to `ValueCastable`;
  * Adding classes that hierarchically describe representation of aggregate values: named field containers with non-overlapping fields (structs), named field containers with overlapping fields (unions), and indexed field containers (arrays);
  * Adding a wrapper class that accepts a `Value` (or a `ValueCastable` object) and provides accessors that slice it according to the corresponding aggregate representation;
  * Adding an ergonomic and IDE-compatible interface for building descriptions of non-parametric layouts of aggregate values.


### User-defined shape-castable objects

`ShapeCastable` is an interface for defining `Shape`-like values outside of the core Amaranth language. It is functionally identical to `ValueCastable`, and could be used like:

```python
from amaranth import *


class Layout(ShapeCastable):
    def __init__(self, fields):
        self.fields = fields

    def as_shape(self):
        return unsigned(sum(len(field) for field in self.fields))
```


### Value layout descriptions

Aggregate value layouts are represented using two classes: `amaranth.lib.data.Field` and `amaranth.lib.data.Layout`:

  * A `Field(shape_castable, offset=0)` object describes a field of the given shape starting at bit number `offset` of the aggregate value.
  * A `Layout()` object describes an abstract aggregate value. It can be iterated, returning `(name, field)` or `(index, field)` pairs; or indexed (`__getitem__`) by the name or index of the field. It has a `.size` in bits, determined by the type of the layout, and is shape-castable, being converted to `unsigned(layout.size())`.
    * A `StructLayout(members={"name": shape_castable})` object describes an aggregate value with non-overlapping named fields (struct). The fields are placed at offsets such that they immediately follow one another, from least significant to most significant.
    * A `UnionLayout(members={"name": shape_castable})` object describes an aggregate value with overlapping named fields (union). The fields are placed at offset 0.
    * An `ArrayLayout(element=shape_castable, length=1)` object describes an aggregate value with indexed fields (array). The fields all have identical shape and are placed at offsets such that they immediately follow one another, from least significant to most significant.
    * A `FlexibleLayout(fields={"name": field, 0: field}, size=16)` object describes a aggregate value with fields arbitrarily placed within its bounds.

The representation of a discriminated union could be programmatically constructed as follows:

```python
import enum
from amaranth.lib import data


class Kind(enum.Enum):
    ONE_SIGNED = 0
    TWO_UNSIGNED = 1


layout = data.StructLayout({
    "kind": Kind,
    "value": data.UnionLayout({
        "one_signed": signed(2),
        "two_unsigned": data.ArrayLayout(unsigned(1), 2)
    })
})
```


### Aggregate value access

Aggregate values are manipulated through the `amaranth.lib.data.View` class. A `View(layout, value_castable)` object wraps a value-castable object (which may be a valid assignment target) and provides access to fields according to the layout. A view is itself value-castable, being converted to the object it's wrapping. If the view is wrapping a valid assignment target, then the accessors also return a valid assignment target.

Fields can be accessed using either `__getitem__` (for both named and indexed fields) or `__getattr__` (for named fields). To avoid name collisions when using `__getattr__` to access fields, views do not define any non-reserved attributes of their own except for the `.as_value()` casting method. Field names starting with `_` are reserved as attribute names and and can only be accessed using the `view["name"]` indexing syntax.

When a view is used to access a field whose shape is an ordinary `Shape` object, the accessor returns a `Value` of the corresponding shape that slices the viewed object.

When a view is used to access a field whose shape is an aggregate value layout, the accessor returns another `View` with this layout, wrapping the slice of the viewed object. For fields that have any other shape-castable object set as their shape, the behavior is the same as for the `Shape` case.

Views that have an `ArrayLayout` as their layout can be indexed with a `Value`. In this case, the viewed object is sliced with `Value.word_select`.

A signal can be manipulated with its structure viewed as the discriminated union defined above as follows:

```python
# creates an unsigned(3) signal by shape-casting `layout`
sig = Signal(layout)
view = data.View(layout, sig)

# if the second argument is omitted, a signal with the right shape is created internally;
# the line below is equivlent to the two lines above
view = data.View(layout)

m = Module()
m.d.comb += [
    view.kind.eq(Kind.TWO_UNSIGNED),
    view.value.two_unsigned[0].eq(1),
]
```


### Ergonomic layout definition

Rather than using the underlying `StructLayout` and `UnionLayout` classes, struct and union layouts can be defined using the Python class definition syntax, with the shapes of the members specified using the [PEP 526](https://peps.python.org/pep-0526/) variable annotations:

```python
class SomeVariant(data.Struct):
    class Value(data.Union):
        one_signed: signed(2)
        two_unsigned: data.ArrayLayout(unsigned(1), 2)

    kind: Kind
    value: Value


# this class can be used in the same way as a `data.View` without needing to specify the layout:
view2 = SomeVariant()
m.d.comb += [
    view2.kind.eq(Kind.ONE_SIGNED),
    view2.value.eq(view.value)
]
```

When they refer to other structures or unions defined in the same way, the variable annotations are also valid [PEP 484](https://peps.python.org/pep-0484/) type hints, and will be used by IDEs to derive types of properties and expressions. Otherwise, the annotations will be opaque to IDEs or type checkers, but are still useful for a human reader.

The classes defined in this way are shape-castable and can be used anywhere a shape or a aggregate value layout is accepted:

```python
sig2 = Signal(SomeVariant)
layout2 = data.StructLayout({
    "ready": unsigned(1),
    "payload": SomeVariant
})
```

**Implementation note:** This can be achieved by using a custom metaclass for `Struct` and `Union` that inherits from `ShapeCastable`.

If an explicit `Layout` object is nevertheless needed (e.g. for introspection), it can be extracted from the class using `Layout.cast`:

```python
layout == data.Layout.cast(SomeVariant) # => True
```

Conversely, the shape-castable object defining the layout of a `View` (which might be a `Layout` subclass or a `Struct`/`Union` subclass) can be extracted from the view using `Layout.of`:

```python
SomeVariant is data.Layout.of(view2) # => True
```


### Advanced usage: Parametric layouts

The ergonomic definitions using the `Struct` and `Union` base classes are concise and integrated with Python type annotations. However, they cannot be used if the layout of an aggregate value is parameterized. In this case, a class with similar functionality can be defined in a more explicit way:

```python
class Stream8b10b(data.View):
    data: Signal
    ctrl: Signal

    def __init__(self, value=None, *, width: int):
        super().__init__(data.StructLayout({
            "data": unsigned(8 * width),
            "ctrl": unsigned(width)
        }), value)


len(Stream8b10b(width=1).data) # => 8
len(Stream8b10b(width=4).data) # => 32
```

Since the parametric class name itself does not have a fixed layout, it cannot be used with `Layout.cast`. Similarly, the type annotations cannot include specific field widths; they are included only to indicate the presence of a corresponding attribute to IDEs and type checkers.


### Structure field ordering

The fields of a structure layout object are ordered from least significant to most significant:

```python
float32_layout = data.StructLayout({
    "fraction": unsigned(23),   # bits  0..22
    "exponent": unsigned(8),    # bits 23..30
    "sign": unsigned(1)         # bit  31
})

class Float32(data.Struct):
    fraction: unsigned(23)      # bits  0..22
    exponent: unsigned(8)       # bits 23..30
    sign: unsigned(1)           # bit  31
```

In other words, the following identity holds:

```python
float32_storage = Signal(float32_layout)
float32 = data.View(float32_layout, float32_storage)

float32_storage == Cat(float32.fraction, float32_layout.exponent, float32.sign)
```


### Customizing the automatically created `Signal`

When a view is instantiated without an explicit view target, it creates a `Signal` with a shape matching the view layout. The `View` constructor accepts all of the `Signal` constructor keyword arguments and passes them along; the `reset=` argument accepts a struct or an array (according to the type of the layout):

```python
flt_neg_reset = data.View(float32_layout, reset={"sign": 1})

flt_reset_less = Float32(reset_less=True)
```


## Drawbacks

This feature introduces a language-level concept, shape-castable objects, increasing language complexity.

This feature introduces a finely grained hierarchy of 5 similar and related classes for describing layouts.


## Alternatives

Do nothing. `Record` will continue to be used alongside the continued proliferation of ad-hoc implementations of similar functionality.

Remove `ArrayLayout` from this proposal. The array functionality is niche and introduces the complexity of handling by-index accessors alongside by-name ones.

Remove `ArrayLayout`, `UnionLayout`, and `FlexibleLayout` from this proposal. Their functionality is less commonly used than that of `StructLayout` and introduces the substantial complexity of handling fields at arbitrary offsets. (This would make `amaranth.lib.data` useless for slicing CSRs in Amaranth SoC.) This change would bring this proposal close to the original `PackedStruct` proposal discussed in https://github.com/amaranth-lang/amaranth/issues/342.

Combine the `Layout` and all of its derivative classes into a single `Layout(fields={"name": Field(...), 0: Field(...)})` class that provides a superset of the functionality. This simplifies the API, but makes introspection of aggregate layouts very difficult and can be inefficient if large arrays are used. In this case, factory methods of the `Layout` class would be provided for more convenient construction of regular struct, union, and array layouts.

Remove `Struct` and `Union` annotation-driven definition syntax. This makes the API simpler, less redundant, and with fewer corner cases, also avoiding the use of variable annotations that are not valid PEP 484 type hints, at the cost of a continued jarring experience for IDE users.

Include a more concise and less visually noisy way to build `StructLayout` and `UnionLayout` objects (or their equivalents) using a builder pattern. This may make the syntax slightly nicer, though the RFC author could not come up with anything that would actually be such.


## Bikeshedding

The names of the `Field`, `*Layout`, and `View` classes could be changed to something better.

  * `IrregularLayout` was renamed to `FlexibleLayout`.


## Future work

This feature could be improved in several ways that are not in scope of this RFC:

  * `StructLayout`, `UnionLayout`, and `ArrayLayout` could be extended to generate layouts with padding at the end (for structs and unions) or between elements (for arrays). Currently this requires the use of a `FlexibleLayout`.
  * `StructLayout` could be extended to accept a list of fields in addition to a map of field names to values. In this case, it would represent an aggregate value with non-overlapping indexed fields (tuple).
  * `Struct` and/or `StructLayout` could be extended to treat certain reserved field names (e.g. `"_1"`, `"_2"`, ...) as designating padding bits. In this case, the offset of the following fields would be adjusted, and the fields with such names would not appear in the layout.
  * `Struct` and/or `StructLayout` could be extended to treat certain reserved field names (e.g. `"_"` for `Struct` and `None` for `StructLayout`) as designating an anonymous inner aggregate. In this case, the members of the anonymous inner aggregate would be possible to access as if they were the members of the outer aggregate.
  * The automatic wrapping of accessed aggregate fields done by `View` could be extended to call a user-specified cast function rather than hard-coding a check for whether the shape is a `Layout`. This would allow seamless inclusion of user-defined value-castable types in aggregates.
  * The [PEP 484](https://peps.python.org/pep-0484/) generics could be used to define layouts parametric over field shapes, using type annotations alone. Since Python does not have type-level integers, layouts parametric over field sizes would still need to be defined explicitly.
  * The struct, union, and enum support could be used as the building blocks to implement first-class discriminated unions. Discriminated unions will also benefit from tuples, described above. ([Suggestion](https://github.com/amaranth-lang/amaranth/issues/693#issuecomment-1089322514) by @lachlansneff.)


## Acknowledgements

[@modwizcode], [@Kaucasus], and [@lachlansneff] provided valuable feedback while this RFC was being drafted.

[@modwizcode]: https://github.com/modwizcode
[@Kaucasus]: https://github.com/Kaucasus
[@lachlansneff]: https://github.com/lachlansneff
