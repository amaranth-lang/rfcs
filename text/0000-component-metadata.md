- Start Date: 2023-11-20
- RFC PR: [amaranth-lang/rfcs#30](https://github.com/amaranth-lang/rfcs/pull/30)
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

An `amaranth.lib.wiring.Component` can provide metadata about itself, represented as a JSON object. This metadata contains a hierarchical description of every port of its interface.

The following example defines an `AsyncSerial` component, and outputs its metadata:

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
        super().__init__(AsyncSerialSignature(divisor_reset, divisor_bits, data_bits, parity))


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
                "reset": "868"
            },
            "rx_ack": {
                "type": "port",
                "name": "rx_ack",
                "dir": "in",
                "width": 1,
                "signed": false,
                "reset": "0"
            },
            "rx_data": {
                "type": "port",
                "name": "rx_data",
                "dir": "out",
                "width": 8,
                "signed": false,
                "reset": "0"
            },
            "rx_err": {
                "type": "port",
                "name": "rx_err",
                "dir": "out",
                "width": 3,
                "signed": false,
                "reset": "0"
            },
            "rx_i": {
                "type": "port",
                "name": "rx_i",
                "dir": "in",
                "width": 1,
                "signed": false,
                "reset": "0"
            },
            "rx_rdy": {
                "type": "port",
                "name": "rx_rdy",
                "dir": "out",
                "width": 1,
                "signed": false,
                "reset": "0"
            },
            "tx_ack": {
                "type": "port",
                "name": "tx_ack",
                "dir": "in",
                "width": 1,
                "signed": false,
                "reset": "0"
            },
            "tx_data": {
                "type": "port",
                "name": "tx_data",
                "dir": "in",
                "width": 8,
                "signed": false,
                "reset": "0"
            },
            "tx_o": {
                "type": "port",
                "name": "tx_o",
                "dir": "out",
                "width": 1,
                "signed": false,
                "reset": "0"
            },
            "tx_rdy": {
                "type": "port",
                "name": "tx_rdy",
                "dir": "out",
                "width": 1,
                "signed": false,
                "reset": "0"
            }
        },
        "annotations": {}
    }
}
```

The `["interface"]["annotations"]` object, which is empty here, is explained in the next section.

### Annotations

Users can attach arbitrary annotations to an `amaranth.lib.wiring.Signature`, which are automatically collected into the metadata of components using this signature.

An `Annotation` class has a name (e.g. `"org.amaranth-lang.amaranth-soc.memory-map"`) and a [JSON schema](https://json-schema.org) defining the structure of its instances. To continue our `AsyncSerial` example, we add an annotation to `AsyncSerialSignature` that will allow us to describe a [8-N-1](https://en.wikipedia.org/wiki/8-N-1) configuration:

```python3
class AsyncSerialAnnotation(Annotation):
    name = "com.example.foo.serial"
    schema = {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "$id": "https://example.com/schema/foo/1.0/serial.json",
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

We can attach annotations to a `Signature` subclass by overriding its `.annotations` property:

```python3
class AsyncSerialSignature(Signature):
    # ...

    @property
    def annotations(self):
        return (*super().annotations, AsyncSerialAnnotation(self))
```

The JSON object returned by ``serial.metadata.as_json()`` will now use this annotation:

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
                "reset": "868"
            },
            "rx_ack": {
                "type": "port",
                "name": "rx_ack",
                "dir": "in",
                "width": 1,
                "signed": false,
                "reset": "0"
            },
            "rx_data": {
                "type": "port",
                "name": "rx_data",
                "dir": "out",
                "width": 8,
                "signed": false,
                "reset": "0"
            },
            "rx_err": {
                "type": "port",
                "name": "rx_err",
                "dir": "out",
                "width": 3,
                "signed": false,
                "reset": "0"
            },
            "rx_i": {
                "type": "port",
                "name": "rx_i",
                "dir": "in",
                "width": 1,
                "signed": false,
                "reset": "0"
            },
            "rx_rdy": {
                "type": "port",
                "name": "rx_rdy",
                "dir": "out",
                "width": 1,
                "signed": false,
                "reset": "0"
            },
            "tx_ack": {
                "type": "port",
                "name": "tx_ack",
                "dir": "in",
                "width": 1,
                "signed": false,
                "reset": "0"
            },
            "tx_data": {
                "type": "port",
                "name": "tx_data",
                "dir": "in",
                "width": 8,
                "signed": false,
                "reset": "0"
            },
            "tx_o": {
                "type": "port",
                "name": "tx_o",
                "dir": "out",
                "width": 1,
                "signed": false,
                "reset": "0"
            },
            "tx_rdy": {
                "type": "port",
                "name": "tx_rdy",
                "dir": "out",
                "width": 1,
                "signed": false,
                "reset": "0"
            }
        },
        "annotations": {
            "com.example.foo.serial": {
                "data_bits": 8,
                "parity": "none"
            }
        }
    }
}
```

#### Annotation names and schema URLs

An `Annotation` schema must have a `"$id"` property, which holds an URL that serves as its unique identifier. This URL must have the following format:

`<protocol>://<domain>/schema/<package>/<version>/<path>.json`

where:

* `<domain>` is a domain name registered to the person or entity defining the annotation;
* `<package>` is the name of the Python package providing the `Annotation` subclass;
* `<version>` is the version of the aforementioned package;
* `<path>` is a non-empty string.

An `Annotation` name should be retrievable from the `"$id"` property of its schema. Its name is the concatenation of the following parts, separated by `"."`:

* `<domain>`, reversed (e.g. "com.example");
* `<package>`;
* `<path>`, split using `"/"` as separator.

Some examples of valid schema URLs:

- "https://example.github.io/schema/foo/1.0/serial.json" for the "io.github.example.foo.serial" annotation;
- "https://amaranth-lang.org/schema/amaranth/0.4/fifo.json" for "org.amaranth-lang.amaranth.fifo";
- "https://amaranth-lang.org/schema/amaranth-soc/0.1/memory-map.json" for "org.amaranth-lang.amaranth-soc.memory-map".

Changes to schema definitions hosted at https://amaranth-lang.org should follow the [RFC process](https://github.com/amaranth-lang/rfcs).

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Annotations

- add an `Annotation` base class to `amaranth.lib.meta`, with:
    * a `.name` "abstract" class attribute, which must be a string (e.g. "org.amaranth-lang.amaranth-soc.memory-map").
    * a `.schema` "abstract" class attribute, which must be a JSON schema, as a dict.
    * a `.__init_subclass__()` class method, which raises an exception if:
        - `.schema` does not comply with the [2020-12 draft](https://json-schema.org/specification-links#2020-12) of the JSON Schema specification.
        - `.name` cannot be extracted from the `.schema["$id"]` URL (as explained [here](#annotation-names-and-schema-urls)).
    - a `.origin` attribute, which returns the Python object described by an annotation instance.
    * a `.validate()` class method, which takes a JSON instance as argument. An exception is raised if the instance does not comply with the schema.
    * a `.as_json()` abstract method, which must return a JSON instance, as a dict. This instance must be compliant with `.schema`, i.e. `self.validate(self.as_json())` must succeed.

The following changes are made to `amaranth.lib.wiring`:
- add a `.annotations` property to `Signature`, which returns an empty tuple. If overriden, it must return an iterable of `Annotation` objects.

### Component metadata

The following changes are made to `amaranth.lib.wiring`:
- add a `ComponentMetadata` class, with:
    - a `.name` class attribute, which returns `"org.amaranth-lang.amaranth.component"`.
    - a `.schema` class attribute, which returns a JSON schema of component metadata. Its definition is detailed [below](#component-metadata-schema).
    - a `.validate()` class method, which takes a JSON instance as argument. An exception is raised if the instance does not comply with the schema.
    - `.__init__()` takes a `Component` object as parameter.
    - a `.origin` attribute, which returns the component object given in `.__init__()`.
    - a `.as_json()` method, which returns a JSON instance of `.origin` that complies with `.schema`. It is populated by iterating over the component's interface and annotations.
- add a `.metadata` property to `Component`, which returns `ComponentMetadata(self)`.

#### Component metadata schema

```python3
class ComponentMetadata(Annotation):
    name   = "org.amaranth-lang.amaranth.component"
    schema = {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "$id": "https://amaranth-lang.org/schema/amaranth/0.5/component.json",
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
                                                "type": "string",
                                                "pattern": "^[+-]?[0-9]+$",
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

Reset values are serialized to strings (e.g. "-1"), because JSON can only represent integers up to 2^53.

## Drawbacks
[drawbacks]: #drawbacks

- Developers need to learn the JSON Schema language to define annotations.
- An annotation schema URL may point to a non-existent domain, despite being well formatted.
- Handling backward-incompatible changes in new versions of an annotation is left to its consumers.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- As an alternative, do nothing; let tools and downstream libraries provide non-interoperable mechanisms to introspect components to and from Amaranth designs.
- Usage of this feature is entirely optional. It has a limited impact on the `amaranth.lib.wiring`, by reserving only two attributes: `Signature.annotations` and `Component.metadata`.
- JSON schema is an IETF standard that is well supported across tools and programming languages.
- This metadata format can be translated into other formats, such as [IP-XACT](https://www.accellera.org/downloads/standards/ip-xact).

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- The clock and reset ports of a component are omitted from this metadata format. Currently, the clock domains of an Amaranth component are only known at elaboration, whereas this RFC requires metadata to be accessible at any time. While this is a significant limitation for multi-clock components, single-clock components may be assumed to have a positive edge clock `"clk"` and a synchronous reset `"rst"`. Support for arbitrary clock domains should be introduced in later RFCs.
- Annotating individual ports of an interface is out of the scope of this RFC. Port annotations may be useful to describe non-trivial signal shapes, and introduced in a later RFC.

## Future possibilities
[future-possibilities]: #future-possibilities

While this RFC can apply to any Amaranth component, one of its motivating use cases is the ability to export the interface and behavioral properties of SoC peripherals in various formats, such as SVD.
