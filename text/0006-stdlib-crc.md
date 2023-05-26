- Start Date: 2023-02-21
- RFC PR: [amaranth-lang/rfcs#0006](https://github.com/amaranth-lang/rfcs/pull/0006)
- Amaranth Issue: [amaranth-lang/amaranth#681](https://github.com/amaranth-lang/amaranth/issues/681)

# Summary
[summary]: #summary

Add a cyclic redundancy check (CRC) generator to the Amaranth standard library.

# Motivation
[motivation]: #motivation

Computing CRCs is a common requirement in hardware designs as they are used
by a range of communication and storage protocols to detect errors and thereby
ensure data integrity. Because of the standard structure and typical set of
variations used by CRCs, it is readily possible to provide a general-purpose
CRC generator in the standard library which should cover the majority of use
cases efficiently.

See the [Wikipedia page on CRCs] for more background and use cases.

[Wikipedia page on CRCs]: https://en.wikipedia.org/wiki/Cyclic_redundancy_check

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Amaranth standard library includes a generator for a cyclic redundancy
check (CRC) module, which can be used to compute and/or validate most common
CRCs used by transmission and storage protocols.

There are many different CRC algorithms in use, but they can almost all be
described by the following parameters:

* The bit width of the CRC, commonly (but not always) a power of 2,
* The generator polynomial, represented as an integer where each bit is a 0 or
  1 term in a binary-valued polynomial with as many terms as the CRC width,
* The initial value of the CRC register, commonly non-zero to allow detection
  of additional 0-valued data words at the start of a message,
* Whether to process input words least- or most-significant-bit first,
  allowing the CRC processing order to match the transmission or storage order
  of the data bits,
* Whether to flip the final output so that its least-significant-bit becomes
  the most-significant bit, set for the same reason as reflecting the input
  when the CRC value will also be transmitted or stored with a certain bit
  order,
* What value, if any, to XOR the output bits with before using the CRC value,
  often used to invert all bits of the output.

This set of parameters is commonly known as the Williams or Rocksoft model. For
more information, refer to ["A Painless Guide to CRC Error Detection Algorithms"].

["A Painless Guide to CRC Error Detection Algorithms"]: http://www.ross.net/crc/download/crc_v3.txt

For a list of parameters to use for standard CRCs, refer to:

* [reveng]'s catalogue, which uses the same parameterisation
* [crcmod]'s predefined list, but remove the leading `1` from the
    polynomials, XOR the "Init-value" with "XOR-out" to obtain `initial_crc`,
    and where `Reversed` is `True`, set both `ref_in` **and**
    `ref_out` to `True`.
* [CRC Zoo], which only lists polynomials; use the "explicit +1" form but
  remove the leading `1`.

The CRC algorithms described in the [reveng] catalogue are also available
in the Amaranth standard library in the `crc.Catalog` class.

[reveng]: https://reveng.sourceforge.io/crc-catalogue/all.htm
[crcmod]: http://crcmod.sourceforge.net/crcmod.predefined.html
[CRC Zoo]: https://users.ece.cmu.edu/~koopman/crc/

In Amaranth, the `crc.Parameters` class holds the parameters that describe a
CRC algorithm:

* `crc_width`: the bit width of the CRC
* `data_width`: the width of the input data words the CRC is computed over;
  commonly 8 for processing byte-wise data but can be any length greater than 0
* `polynomial`: the generator polynomial of the CRC, excluding an implicit
  leading 1 for the "x^n" term
* `initial_crc`: the initial value of the CRC, loaded when computation of a
  new CRC begins
* `reflect_input`: if True, input words are bit-reversed so that the least
  significant bit is processed first
* `reflect_output`: if True, the final output is bit-reversed so that its
    least-significant bit becomes the most-significant bit of output
* `xor_output`: a value to XOR the output with

The `crc.Parameters` class may be constructed manually, or via a
`crc.Predefined` entry in `crc.Catalog`, which contains many commonly
used CRC algorithms. The data width is not part of the `crc.Predefined`
class, so it must be specified to create a `crc.Parameters`, for example:

```python
from amaranth.lib import crc
params1 = crc.Parameters(8, 8, 0x2f, 0xff, False, False, 0xff)
params2 = crc.Catalog.CRC8_AUTOSAR(data_width=8)
```

If not specified, the data width defaults to 8 bits.

The `crc.Parameters` class can be used to compute CRCs in software using its
`compute()` method, which is passed an iterable of integer data words and
returns the resulting CRC value.

```python
from amaranth.lib import crc
params = crc.Parameters(8, 8, 0x2f, 0xff, False, False, 0xff)
assert params.compute(b"123456789") == 0xdf
```

To generate a hardware CRC module, either call `create()` on `crc.Parameters`
or manually construct a `crc.Processor`:

```python
from amaranth.lib import crc
params = crc.Parameters(8, 8, 0x2f, 0xff, False, False, 0xff)
crc1 = m.submodules.crc1 = crc.Processor(params)
crc2 = m.submodules.crc2 = crc.Catalog.CRC8_AUTOSAR().create()
```

The `crc.Processor` module begins computation of a new CRC whenever its `first`
input is asserted. Input on `data` is processed whenever `valid` is asserted,
which may occur in the same clock cycle as `first`. The updated CRC value is
available on the `crc` output on the clock cycle after `valid`.

With the data width set to 1, a traditional bit-serial CRC is implemented
for the given polynomial in a Galois-type shift register. For larger values
of data width, a similar architecture computes every new bit of the CRC in
parallel.

The `match_detected` output signal may be used to validate data that contains a
trailing CRC. If the most recently processed word(s) form a valid CRC for all
the data processed since reset, the CRC register will always contain a fixed
value which can be computed in advance, and the `match_detected` output
indicates whether the CRC register currently contains this value.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The proposed new interface is:

* A `crc.Parameters` class which holds the parameters for a CRC algorithm,
  all of which are passed to the constructor:
    * `crc_width`: bit width of the CRC register
    * `data_width`: bit width of input data words
    * `polynomial`: generator polynomial for CRC algorithm
    * `initial_crc`: initial value of CRC at start of computation
    * `reflect_input`: if True, input words are bit-reversed
    * `reflect_output`: if True, output values are bit-reversed
    * `xor_output`: value to XOR the CRC value with at output
* The class has the following methods:
    * `compute(data)` performs a software CRC computation on `data`, an
      iterable of input data words, and returns the CRC value
    * `create()` returns a `crc.Processor` instance preconfigured to use
      these parameters
    * `residue()` returns the residue value for these parameters, which is
      the value left in the CRC register after processing an entire valid
      codeword (data followed by its own valid CRC)
    * `check()` returns the CRC computation of the ASCII string "123456789",
      a defacto standard for validating CRC correctness
* A `crc.Predefined` class which is constructed using all the parameters of
  `crc.Parameters` except `data_width`, which is application-specific. This
  class implements `__call__(data_width=8)` which is used to create a
  `crc.Parameters` with the specified data width
* A `crc.Catalog` class which contains instances of `crc.Predefined` as
  class attributes
* A `crc.Processor` class which inherits from `Elaboratable` and implements
  the hardware generator

The hardware implementation uses the property that CRCs are linear, and so the
new value of any bit of the CRC register can be found as a linear combination
of the current state and all the input bits. By expressing the CRC computation
as a linear system like this, we can then determine the boolean equation used
to update each bit of the CRC in parallel. A software CRC calculator is
implemented in Python in order to find these logic equations.

The proposed CRC generator is already implemented and available in [PR 681].
The docstrings and comments in it should explain its operation to a suitably
technical level of detail.

[PR 681]: https://github.com/amaranth-lang/amaranth/pull/681

# Drawbacks
[drawbacks]: #drawbacks

Users could always write their own CRC or use an external library; Amaranth
does not need to provide one for them. However, since it's a very common
requirement that we can satisfy efficiently for a lot of users, it seems
reasonable to include in the standard library.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

As far as I'm aware, the method here is the optimal technique for generating
the logic equations required for this combinatorial CRC generation.

One alternative to the combinatorial logic equations is to store intermediate
values in a lookup table; the table needs to contain a CRC-sized value for
every possible input value, and then the computation required is reduced to a
table lookup, an XOR, and some bit shifts. For single-byte words this approach
may be practical, but it is unlikely to be worthwhile with 16- or 32-bit words.
Additionally, the table approach generally requires a latency of 2 cycles (one
extra to perform the table lookup). It's possible this would give better timing
in some circumstances, but at the cost of block RAM resources and latency.

# Prior art
[prior-art]: #prior-art

The specification chosen for the CRC parameters is a popular de-facto standard,
and importantly the [reveng] catalogue lists suitable parameters for a wide
range of standard CRCs.

This particular implementation was written in 2020 and is extracted (with
permission) from a proprietary codebase, where it is used to generate a
variety of CRCs on FPGAs.

One early public example of using Amaranth to generate CRCs is from
[Harmon Instruments], also in 2020, which has a similar construction
but does not support the full set of CRC parameters.

[Harmon Instruments]: https://gitlab.com/harmoninstruments/harmon-instruments-open-hdl/-/blob/master/Ethernet/CRC.py

In general, I found many examples of implementations of _specific_ CRCs in
other HDLs, but few for generic generators. There are many software libraries
for generating CRCs in most programming languages, but as they are not
generating hardware their implementation details are not as relevant - small
table lookups are popular as the tradeoffs there tend to favour word-at-a-time
computations.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Currently the design uses a `first` signal to signal the beginning of a new
  CRC computation (previously called `reset`). Requiring that it may be
  asserted simultaneously with the first valid data word of the new CRC adds
  muxes to the design that would not be needed if `first` and `valid` cannot
  be asserted together. Is this a worthwhile tradeoff, or is there a way to
  avoid the extra muxes?

# Future possibilities
[future-possibilities]: #future-possibilities

- The data interface uses `first`, `data`, and `valid` signals.
  Eventually, this should be replaced with a Stream, once they are finalised.

- Currently the entire input data word must be valid together; there is no
  support for masking some bits off. In particular, such a feature could be
  useful for wide data paths where the underlying CRC computation is byte-wise,
  for example a 128-bit-wide data stream from a 10GbE MAC where the Ethernet
  FCS must be computed over individual bytes. However, the implementation
  complexity is high, the use cases seem more niche, and such a feature could
  be added in a backwards-compatible later revision.
