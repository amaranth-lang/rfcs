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

The standard library includes a generator for a cyclic redundancy check (CRC)
module, which can be used to compute and/or validate most common CRCs used by
transmission and storage protocols.

There are many variations on CRCs in use, but they can almost all be described
by the following parameters:

* The bit width of the checksum, commonly (but not always) a power of 2
* The generator polynomial
* The initial value of the CRC register
* Whether to process input words least- or most-significant-bit first
* Whether to flip the final output so that its least-significant-bit becomes
    the most-significant bit
* What value, if any, to XOR the output bits with before using the CRC value

This set of parameters is commonly known as the Williams or Rocksoft model. For
more information, refer to ["A Painless Guide to CRC Error Detection Algorithms"].

["A Painless Guide to CRC Error Detection Algorithms"]: http://www.ross.net/crc/download/crc_v3.txt

The generator in the Amaranth standard library computes CRCs from an input
data stream of configurable width, and can therefore be use for both validating
existing CRCs and creating new ones. It can handle any CRC parameterised by
the Williams model above.

For a list of parameters to use for standard CRCs, refer to:

* [reveng]'s catalogue, which uses the same parameterisation
* [crcmod]'s predefined list, but removing the leading `1` from the
    polynomials, and where `Reversed` is `True`, set both `ref_in` **and**
    `ref_out` to `True`.
* [CRC Zoo], which only lists polynomials; use the "explicit +1" form but
  remove the leading `1`.

[reveng]: https://reveng.sourceforge.io/crc-catalogue/all.htm
[crcmod]: http://crcmod.sourceforge.net/crcmod.predefined.html
[CRC Zoo]: https://users.ece.cmu.edu/~koopman/crc/

The computed CRC value is updated on any clock cycle where the `valid` input
is asserted, with the updated value available on the `crc` output on the
subsequent clock cycle. The latency is one cycle, and the throughput is
one data word per clock cycle.

With the data width set to 1, a traditional bit-serial CRC is implemented
for the given polynomial in a Galois-type shift register. For larger values
of data width, a similar architecture computes every new bit of the CRC in
parallel.

The `match` output signal may be used to validate data that contains a trailing
CRC. If the most recently processed word(s) form a valid CRC for all the data
processed since reset, the CRC register will always contain a fixed value which
can be computed in advance, and the `match` output indicates whether the CRC
register currently contains this value.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The proposed CRC generator is already implemented and available in [PR 681].
The docstrings and comments in it should explain its operation to a suitably
technical level of detail.

[PR 681]: https://github.com/amaranth-lang/amaranth/pull/681

The proposed implementation uses the fact that CRCs are a linear calculation,
and so the new value of any bit of the CRC register can be found as a linear
combination of the current state and all the input bits. By expressing the CRC
computation as a linear system like this, we can then determine the logic
equation used to update each bit of the CRC in parallel. A software CRC
calculator is implemented in Python in order to find these logic equations.

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

- There is a `reset` signal to reset the CRC state to its initial value.
  In practice, users could replace this with a `ResetInserter` wrapper.
  Since all users will need this signal, it seems easier to include it
  directly in the module interface.

- We could include a number of standard CRC parameters directly in the standard
  library, to make using them more convenient. I don't know what the preferred
  interface for this would be, perhaps just a Python module with some
  constants?

# Future possibilities
[future-possibilities]: #future-possibilities

- The data interface uses a `data` and `valid` signal. Eventually, this should
  be replaced with a Stream, once they are finalised.

- Currently the entire input data word must be valid together; there is no
  support for masking some bits off. In particular, such a feature could be
  useful for wide data paths where the underlying CRC computation is byte-wise,
  for example a 128-bit-wide data stream from a 10GbE MAC where the Ethernet
  FCS must be computed over individual bytes. However, the implementation
  complexity is high, the use cases seem more niche, and such a feature could
  be added in a backwards-compatible later revision.
