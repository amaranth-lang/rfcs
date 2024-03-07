- Start Date: 2024-03-18
- RFC PR: [amaranth-lang/rfcs#55](https://github.com/amaranth-lang/rfcs/pull/55)
- Amaranth Issue: [amaranth-lang/amaranth#1210](https://github.com/amaranth-lang/amaranth/issues/1210)

# New `lib.io` components

## Summary
[summary]: #summary

Building on RFC 2 and RFC 53, a new set of components is added to `lib.io`. The current contents of `lib.io` (`Pin` and its signature) become deprecated.

## Motivation
[motivation]: #motivation

Currently, all IO buffer and register logic is instantiated in one place by `platform.request`. Per [amaranth-lang/amaranth#458](https://github.com/amaranth-lang/amaranth/issues/458), this has caused problems. Ideally, the act of requesting a specific I/O (user's responsibility) would be decoupled from instantiating the I/O buffer (peripherial library's responsibility).

Further, we currently have no standard I/O buffer components, other than the low-level `IOBufferInstance`.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

`IOPort`, introduced in RFC 53, is Amaranth's low-level IO port primitive. In `lib.io`, `IOPort`s
are wrapped in two higher-level objects: `SingleEndedPort` and `DifferentialPort`. These objects contain the raw `IOValue`s together with per-bit inversion flags. They are obtained from `platform.request`. If no platform is used, they can be constructed by the user directly, like this:

```py
a = SingleEndedPort(IOPort(8, name="a")) # simple 8-bit IO port
b = SingleEndedPort(IOPort(8, name="b"), invert=True) # 8-bit IO port, all bits inverted
c = SingleEndedPort(IOPort(4, name="c"), invert=[False, True, False, True]) # 4-bit IO port, varying per-bit inversions
d = DifferentialPort(p=IOPort(4, name="dp"), n=IOPort(4, name="dn")) # differential 4-bit IO port
```

Once a `*Port` object is obtained, whether from `platform.request` or by direct creation, it most likely needs to be passed to an I/O buffer. Amaranth provides a set of cross-platform I/O buffer components in `lib.io`.

For a non-registered port, the `lib.io.Buffer` can be used:

```py
port = platform.request(...) # or = SingleEndedPort(,,,)
m.submodules.iob = iob = lib.io.Buffer(lib.io.Direction.Bidir, port)
m.d.comb += [
    iob.o.eq(...),
    iob.oe.eq(...),
    (...).eq(iob.i),
]
```

For an SDR registered port, the `lib.io.FFBuffer` can be used:

```py
m.submodules.iob = iob = lib.io.FFBuffer(lib.io.Direction.Bidir, port, i_domain="sync", o_domain="sync")
m.d.comb += [
    iob.o.eq(...),
    iob.oe.eq(...),
    (...).eq(iob.i),
]
```

For a DDR registered port (given a supported platform), the `lib.io.DDRBuffer` can be used:

```py
m.submodules.iob = iob = lib.io.DDRBuffer(lib.io.Direction.Bidir, port, i_domain="sync", o_domain="sync")
m.d.comb += [
    iob.o[0].eq(...),
    iob.o[1].eq(...),
    iob.oe.eq(...),
    (...).eq(iob.i[0]),
    (...).eq(iob.i[1]),
]
```

All of the above primitives are components with corresponding signature types. When elaborated, the primitives call a platform hook, allowing it to provide a custom implementation using vendor-specific cells. If no special support is provided by the platform, `Buffer` and `FFBuffer` provide a simple vendor-agnostic default implementation, while `DDRBuffer` raises an error when elaborated.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The following classes are added to `lib.io`:

- ```py
  class Direction(enum.Enum):
      Input  = "i"
      Output = "o"
      Bidir  = "io"
  ```

  Represents a port or buffer direction.

- `SingleEndedPort(io: IOValue, *, invert: bool | Iterable[bool]=False, direction: Direction=Direction.Bidir)`: represents a single ended port; the `invert` parameter is normalized to a tuple of `bool` before being stored as an attribute
  - `__len__(self)`: returns `len(io)`
  - `__getitem__(self, index: slice | int)`: allows slicing the object, returning another `SingleEndedPort`; requesting a single index is equivalent to requesting a one-element slice
  - `__add__(self, other: SingleEndedPort)`: concatenates two ports together into a bigger `SingleEndedPort`
  - `__invert__(self)`: returns a new `SingleEndedPort` derived from this one by having the opposite (every element of) `invert`
- `DifferentialPort(p: IOValue, n: IOValue, *, invert: bool | Iterable[bool]=False, direction: Direction=Direction.Bidir)`: represents a differential pair; both `IOValue`s given as arguments must have equal width
  - `__len__(self)`: returns `len(p)` (which is equal to `len(n)`)
  - `__getitem__(self, index: slice | int)`: allows slicing the object, returning another `DifferentialPort`
  - `__add__(self, other: DifferentialPort)`: concatenates two ports together into a bigger `DifferentialPort`
  - `__invert__(self)`: returns a new `DifferentialPort` derived from this one by having the opposite (every element of) `invert`
- `Buffer.Signature(direction: Direction | str, width: int)`: a signature for the `Buffer`; if `direction` is a string, it is converted to `Direction`
  - `i: Out(width)` if `direction in (Direction.Input, Direction.Bidir)`
  - `o: In(width)` if `direction in (Direction.Output, Direction.Bidir)`
  - `oe: In(1, init=1)` if `direction is Direction.Output`
  - `oe: In(1, init=0)` if `direction is Direction.Bidir`
- `Buffer(direction: Direction | str, port: SingleEndedPort | DifferentialPort | ...)`: non-registered buffer, derives from `Component`
  - when elaborated, tries to return `platform.get_io_buffer(self)`; if such a function doesn't exist, lowers to `IOBufferInstance` plus optional inverters
- `FFBuffer.Signature(direction: Direction | str, width: int)`: a signature for the `FFBuffer`
  - `i: Out(width)` if `direction in (Direction.Input, Direction.Bidir)`
  - `o: In(width)` if `direction in (Direction.Output, Direction.Bidir)`
  - `oe: In(1, init=1)` if `direction is Direction.Output`
  - `oe: In(1, init=0)` if `direction is Direction.Bidir`
- `FFBuffer(direction: Direction | str, port: SingleEndedPort | DifferentialPort | ..., *, i_domain="sync", o_domain="sync")`: SDR registered buffer, derives from `Component`
  - when elaborated, tries to return `platform.get_io_buffer(self)`; if such a function doesn't exist, lowers to `IOBufferInstance`, plus reset-less FFs realized by `m.d[*_domain]` assignment, plus optional inverters
- `DDRBuffer.Signature(direction: Direction | str, width: int)`: a signature for the `DDRBuffer`
  - `i: Out(ArrayLayout(width, 2))` if `direction in (Direction.Input, Direction.Bidir)`
  - `o: In(ArrayLayout(width, 2))` if `direction in (Direction.Output, Direction.Bidir)`
  - `oe: In(1, init=1)` if `direction is Direction.Output`
  - `oe: In(1, init=0)` if `direction is Direction.Bidir`
- `DDRBuffer(direction: Direction | str, port: SingleEndedPort | DifferentialPort | ..., *, i_domain="sync", o_domain="sync")`: DDR registered buffer, derives from `Component`
  - when elaborated, tries to return `platform.get_io_buffer(self)`; if such a function doesn't exist, raises an error

All of the above classes are fully introspectable, and the constructor arguments are accessible as read-only attributes.

If a platform is not used, the `port` argument must be a `SingleEndedPort` or `DifferentialPort`. If a platform is used, the platform may define support for additional types. Such types must implement the same interface as `*Port` objects, that is:

- `__len__` must provide length in bits (so that `*Buffer` can know the proper signature)
- `__getitem__` which supports slices, and where plain indices return single-bit slices
- `__invert__` that returns another port-like
- `direction` attribute that must be a `Direction`

If a platform is not used, and a `DifferentialPort` is used, a pseudo-differential port is effectively created.

The `direction` argument on `*Port` can be used to restrict valid allowed buffer directions as follows:

- an `Input` buffer will not accept an `Output` port
- an `Output` buffer will not accept an `Input` port
- a `Bidir` buffer will only accept a `Bidir` port

This is validated by the `*Buffer` constructors. Custom buffer-like elaboratables that take `*Port` are likewise encouraged to perform similar checking.

The `platform.request` function with `dir="-"` returns `SingleEndedPort` when called on single-ended ports, `DifferentialPort` when called on differential pairs. Using `platform.request` with any other `dir` becomes deprecated, in favor of having the user (or peripherial library) code explicitly instantiate `*Buffer`s. The `lib.io.Pin` interface and its signature likewise become deprecated.

## Drawbacks
[drawbacks]: #drawbacks

The proposed `FFBuffer` and `DDRBuffer` interfaces have a minor problem of not actually being currently implementable in many cases, as there is no way to obtain clock signal polarity at that stage of elaboration. A solution for that needs to be proposed, whether as a private hack for the current platforms, or as an RFC.

Using plain domains for `DDRBuffer` has the unprecedented property of triggering logic on the opposite of active edge of the domain.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The buffers have minimal functionality on purpose, to allow them to be widely supported. In particular:

- clock enables are not supported
- reset is not supported
- initial values are not supported
- `xdr > 2` is not supported

Such functionality can be provided by vendor-specific primitives.

## Prior art
[prior-art]: #prior-art

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

Vendor-specific versions of the proposed buffers can be added to the `vendor` module, allowing access to the full range of hardware functionality.