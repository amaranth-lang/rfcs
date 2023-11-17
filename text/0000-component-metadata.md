- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Component metadata RFC

## Summary
[summary]: #summary

Add support for JSON-based introspection of an Amaranth component, describing its interface and properties.

## Motivation
[motivation]: #motivation

Introspection of components is an inherent feature of Amaranth. As Python objects, they make use of attributes to:
- expose the ports that compose their interface.
- communicate other kinds of metadata, such as behavioral properties or safety invariants.

Multiple tools may consume parts of this metadata at different points in time. While the ports of an interface must be known at build time, other properties (such as a bus memory map) may be used afterwards to operate or verify the design.

However, in a mixed HDL design, components implemented in other HDLs require ad-hoc integration:
- their netlist must be consulted in order to know their signature.
- each port must be connected individually (whereas Amaranth components can use `connect()` on compatible interfaces).
- there is no mechanism to pass metadata besides instance parameters and attributes. Any information produced by the instance itself cannot be easily passed to its parent.

This RFC proposes a JSON-based format to describe and exchange component metadata. While building upon the concepts of [RFC 2](https://github.com/amaranth-lang/rfcs/blob/main/text/0002-interfaces.md), this metadata format tries to avoid making assumptions about its consumers (which could be other HDL frontends, block diagram design tools, etc).

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Component metadata

An `amaranth.lib.wiring.Component` can provide metadata about itself, represented as a JSON object. This metadata contains a hierarchical description of every port of its interface:

```python3
from amaranth import *
from amaranth.lib.data import StructLayout
from amaranth.lib.wiring import In, Out, Signature, Component


class AsyncSerialSignature(Signature):
    def __init__(self, divisor_reset, divisor_bits, data_bits, parity):
        self.data_bits = data_bits
        self.parity    = parity

        super().__init__({
            "divisor": In(divisor_bits, reset=divisor_reset),

            "rx_data": Out(data_bits),
            "rx_err":  Out(StructLayout({"overflow": 1, "frame": 1, "parity": 1})),
            "rx_rdy":  Out(1),
            "rx_ack":  In(1),
            "rx_i":    In(1),

            "tx_data": In(data_bits),
            "tx_rdy":  Out(1),
            "tx_ack":  In(1),
            "tx_o":    Out(1),
        })


class AsyncSerial(Component):
    def __init__(self, *, divisor_reset, divisor_bits, data_bits=8, parity="none"):
        self.divisor_reset = divisor_reset
        self.divisor_bits  = divisor_bits
        self.data_bits     = data_bits
        self.parity        = parity

    @property
    def signature(self):
        return AsyncSerialSignature(self.divisor_reset, self.divisor_bits, self.data_bits, self.parity)


if __name__ == "__main__":
    import json
    from amaranth.utils import bits_for

    divisor = int(100e6 // 115200)
    serial = AsyncSerial(divisor_reset=divisor, divisor_bits=bits_for(divisor), data_bits=8, parity="none")

    print(json.dumps(serial.metadata.as_json(), indent=4))
```

The `.metadata` property of a `Component` returns a `ComponentMetadata` instance describing that component. In the above example, ``serial.metadata.as_json()`` converts this metadata into a JSON object, which is then printed:

```json
{
    "interface": {
        "members": {
            "divisor": {
                "type": "port",
                "name": "divisor",
                "dir": "in",
                "width": 10,
                "signed": false,
                "reset": 868
            },
            "rx_ack": {
                "type": "port",
                "name": "rx_ack",
                "dir": "in",
                "width": 1,
                "signed": false,
                "reset": 0
            },
            "rx_data": {
                "type": "port",
                "name": "rx_data",
                "dir": "out",
                "width": 8,
                "signed": false,
                "reset": 0
            },
            "rx_err": {
                "type": "port",
                "name": "rx_err",
                "dir": "out",
                "width": 3,
                "signed": false,
                "reset": 0
            },
            "rx_i": {
                "type": "port",
                "name": "rx_i",
                "dir": "in",
                "width": 1,
                "signed": false,
                "reset": 0
            },
            "rx_rdy": {
                "type": "port",
                "name": "rx_rdy",
                "dir": "out",
                "width": 1,
                "signed": false,
                "reset": 0
            },
            "tx_ack": {
                "type": "port",
                "name": "tx_ack",
                "dir": "in",
                "width": 1,
                "signed": false,
                "reset": 0
            },
            "tx_data": {
                "type": "port",
                "name": "tx_data",
                "dir": "in",
                "width": 8,
                "signed": false,
                "reset": 0
            },
            "tx_o": {
                "type": "port",
                "name": "tx_o",
                "dir": "out",
                "width": 1,
                "signed": false,
                "reset": 0
            },
            "tx_rdy": {
                "type": "port",
                "name": "tx_rdy",
                "dir": "out",
                "width": 1,
                "signed": false,
                "reset": 0
            }
        },
        "annotations": {}
    }
}
```

The `["interface"]["annotations"]` object, which is empty here, is explained in the next section.

### Annotations

Users can attach arbitrary annotations to an `amaranth.lib.wiring.Signature`, which are automatically collected into the metadata of components using this signature.

An `Annotation` class has a name (e.g. `"org.amaranth-lang.soc.memory-map"`) and a [JSON schema](https://json-schema.org) defining the structure of its instances. To continue our previous example, we add an annotation to `AsyncSerialSignature` that will allow us to describe a [8-N-1](https://en.wikipedia.org/wiki/8-N-1) configuration:

```python3
class AsyncSerialAnnotation(Annotation):
    name = "org.example.serial"
    schema = {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "$id": "https://example.org/schema/0.1/serial",
        "type": "object",
        "properties": {
            "data_bits": {
                "type": "integer",
                "minimum": 0,
            },
            "parity": {
                "enum": [ "none", "mark", "space", "even", "odd" ],
            },
        },
        "additionalProperties": False,
        "required": [
            "data_bits",
            "parity",
        ],
    }

    def __init__(self, origin):
        assert isinstance(origin, AsyncSerialSignature)
        self.origin = origin

    def as_json(self):
        instance = {
            "data_bits": self.origin.data_bits,
            "parity": self.origin.parity,
        }
        self.validate(instance)
        return instance
```

We can now override the `.annotations` property of `AsyncSerialSignature` to return an instance of our annotation:

```python3
class AsyncSerialSignature(Signature):
    # ...

    @property
    def annotations(self):
        return (AsyncSerialAnnotation(self),)
```

Note: `Signature.annotations` can return multiple annotations, but they must have different names.

Printing ``json.dumps(serial.metadata.as_json(), indent=4)`` will now output this:

```json
{
    "interface": {
        "members": {
            "divisor": {
                "type": "port",
                "name": "divisor",
                "dir": "in",
                "width": 10,
                "signed": false,
                "reset": 868
            },
            "rx_ack": {
                "type": "port",
                "name": "rx_ack",
                "dir": "in",
                "width": 1,
                "signed": false,
                "reset": 0
            },
            "rx_data": {
                "type": "port",
                "name": "rx_data",
                "dir": "out",
                "width": 8,
                "signed": false,
                "reset": 0
            },
            "rx_err": {
                "type": "port",
                "name": "rx_err",
                "dir": "out",
                "width": 3,
                "signed": false,
                "reset": 0
            },
            "rx_i": {
                "type": "port",
                "name": "rx_i",
                "dir": "in",
                "width": 1,
                "signed": false,
                "reset": 0
            },
            "rx_rdy": {
                "type": "port",
                "name": "rx_rdy",
                "dir": "out",
                "width": 1,
                "signed": false,
                "reset": 0
            },
            "tx_ack": {
                "type": "port",
                "name": "tx_ack",
                "dir": "in",
                "width": 1,
                "signed": false,
                "reset": 0
            },
            "tx_data": {
                "type": "port",
                "name": "tx_data",
                "dir": "in",
                "width": 8,
                "signed": false,
                "reset": 0
            },
            "tx_o": {
                "type": "port",
                "name": "tx_o",
                "dir": "out",
                "width": 1,
                "signed": false,
                "reset": 0
            },
            "tx_rdy": {
                "type": "port",
                "name": "tx_rdy",
                "dir": "out",
                "width": 1,
                "signed": false,
                "reset": 0
            }
        },
        "annotations": {
            "org.example.serial": {
                "data_bits": 8,
                "parity": "none"
            }
        }
    }
}
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Annotations

- add an `Annotation` base class to `amaranth.lib.annotations`, with:
    * a `.name` "abstract" class attribute, which must be a string (e.g. "org.amaranth-lang.soc.memory-map").
    * a `.schema` "abstract" class attribute, which must be a JSON schema, as a dict.
    * a `.validate()` class method, which takes a JSON instance as argument. An exception is raised if the instance does not comply with the schema.
    * a `.as_json()` abstract method, which must return a JSON instance, as a dict. This instance must be compliant with `.schema`, i.e. `self.validate(self.as_json())` must succeed.

The following changes are made to `amaranth.lib.wiring`:
- add a `.annotations` property to `Signature`, which returns an empty tuple. If overriden, it must return an iterable of `Annotation` objects.

### Component metadata

The following changes are made to `amaranth.lib.wiring`:
- add a `ComponentMetadata` class, inheriting from `Annotation`, where:
    - `.name` returns "org.amaranth-lang.component".
    - `.schema` returns a JSON schema describing component metadata. Its definition is detailed below.
    - `.__init__()` takes a `Component` object as parameter.
    -`.origin` returns the component object given in `.__init__()`.
    - `.as_json()` returns a JSON instance of `.origin`, that complies with `.schema`. It is populated by iterating over the component's interface and annotations.
- add a `.metadata` property to `Component`, which returns `ComponentMetadata(self)`.

#### Component metadata schema

```python3
class ComponentMetadata(Annotation):
    name   = "org.amaranth-lang.component"
    schema = {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "$id": "https://amaranth-lang.org/schema/amaranth/0.4/component",
        "type": "object",
        "properties": {
            "interface": {
                "type": "object",
                "properties": {
                    "members": {
                        "type": "object",
                        "patternProperties": {
                            "^[A-Za-z][0-9A-Za-z_]*$": {
                                "oneOf": [
                                    {
                                        "type": "object",
                                        "properties": {
                                            "type": {
                                                "enum": ["port"],
                                            },
                                            "name": {
                                                "type": "string",
                                            },
                                            "dir": {
                                                "enum": ["in", "out"],
                                            },
                                            "width": {
                                                "type": "integer",
                                                "minimum": 0,
                                            },
                                            "signed": {
                                                "type": "boolean",
                                            },
                                            "reset": {
                                                "type": "integer",
                                                "minimum": 0,
                                            },
                                        },
                                        "additionalProperties": False,
                                        "required": [
                                            "type",
                                            "name",
                                            "dir",
                                            "width",
                                            "signed",
                                            "reset",
                                        ],
                                    },
                                    {
                                        "type": "object",
                                        "properties": {
                                            "type": {
                                                "enum": ["interface"],
                                            },
                                            "members": {
                                                "$ref": "#/properties/interface/properties/members",
                                            },
                                            "annotations": {
                                                "type": "object",
                                            },
                                        },
                                        "additionalProperties": False,
                                        "required": [
                                            "type",
                                            "members",
                                            "annotations",
                                        ],
                                    },
                                ],
                            },
                        },
                        "additionalProperties": False,
                    },
                    "annotations": {
                        "type": "object",
                    },
                },
                "additionalProperties": False,
                "required": [
                    "members",
                    "annotations",
                ],
            },
        },
        "additionalProperties": False,
        "required": [
            "interface",
        ]
    }

    # ...
```

Notes:
- Reset values are represented by their two's complement. For example, the reset value of `Member(Out, signed(2), reset=-1)` would be given as `3`.
- Despite not being enforced in this schema, annotations must be uniquely identified by their name. For example, an `"org.example.serial"` annotation may have only one possible schema.

## Drawbacks
[drawbacks]: #drawbacks

- `Annotation` class definitions must be kept in sync with their associated `Signature`. Using `Annotation.validate()` can catch some mismatches, but won't help if one forgets to add a new attribute to the JSON schema.
- it is possible to define multiple `Annotation` classes with the same `.name` attribute.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Usage of this feature is entirely optional. It has a limited impact on the `amaranth.lib.wiring`, by reserving only two attributes: `Signature.annotations` and `Component.metadata`.
- JSON schema is an IETF standard that is well supported across tools and programming languages.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

To do before merging:
- Add support for clock and reset ports of a component.

Out of scope:
- Add support for port annotations (e.g. to describe non-trivial shapes).

## Future possibilities
[future-possibilities]: #future-possibilities

While this RFC can apply to any Amaranth component, one of its motivating use cases is the ability to export the interface and behavioral properties of SoC peripherals.
