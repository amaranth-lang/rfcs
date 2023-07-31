- Start Date: 2024-08-04
- RFC PR: [amaranth-lang/rfcs#72](https://github.com/amaranth-lang/rfcs/pull/72)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# LFSR generator

## Summary
[summary]: #summary

Add a linear feedback shift register (LFSR) generator to the Amaranth standard
library, with pre-defined generators for standard PRBS sequences.

## Motivation
[motivation]: #motivation

Many digital designs require the generation of pseudo-random bit sequences,
for purposes such as link error detection, data whitening, generating training
signals, or low-overhead generation of random noise signals. One common and
efficient way to generate maximum-length pseudo-random bit sequences is a
linear feedback shift register (LFSR).

Because an LFSR is fully characterised by its generating polynomial (or,
equivalently, by the set of feedback taps), it is straightforward to support
a wide range of use cases in an optimal implementation that can be used by
most Amaranth users. In particular, the well-known PRBS sequences can be
provided as pre-defined LFSRs.

See the [Wikipedia page on LFSRs] for more background and use cases.

[Wikipedia page on LFSRs]: https://en.wikipedia.org/wiki/Linear-feedback_shift_register

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Amaranth standard library includes a generator for a linear feedback shift
register (LFSR) module, which can be used to compute many common bit sequences.

An LFSR is characterised by two properties, stored in the `lfsr.Algorithm`
class:

* `width`: the bit-width of the feedback register
* `polynominal`: the polynomial that defines which bits are used for feedback,
  excluding an implicit leading 1 for the "x^n" term
* `reverse`: whether to generate the time-reversed output sequence, equivalent
  to mirroring the polynominal

Common PRBS sequences are generated using LFSRs and are provided as pre-defined
`lfsr.Algorithm` objects in `lfsr.catalog`, including PRBS7, PRBS9, PRBS11,
PRBS13, PRBS15, PRBS20, PRBS23, and PRBS31.

Settings specific to a particular instantiation of an LFSR are contained in the
`lfsr.Parameters` class, which is constructed by calling an `lfsr.Algorithm`
instance and providing the additional settings:

* `output_width`: by default 1, but may be increased to generate multiple
  output bits in parallel per clock cycle
* `structure`: either `"galois"` (the default) or `"fibonacci"` to select the
  implementation technique; both generate the same sequence but with a
  different time offset and sequence of internal states, and one or the
  other may be preferable on some architectures
* `init`: by default 1, but may be set to any non-zero initial state matching
  the width of the LFSR register.

The `lfsr.Parameters` class can generate output bit sequences in software using
its `compute()` method, which returns an infinite iterator.

To generate a hardware LFSR module, either call `create()` on `lfsr.Parameters`
or manually construct an `lfsr.Processor`:

```python
from amaranth.lib import lfsr
algo = lfsr.Algorithm(width=7, polynomial=0x20)
params = algo(output_width=4)
m.submodules.prbs7 = lfsr.Processor(params)
m.submodules.prbs9 = lfsr.catalog.PRBS9().create()
```

The LFSR module generates a new output on its `out` output every clock cycle.
Use `ResetInserter` and `EnableInserter` if additional sequence control is
required. The internal state is available as the `state` output.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The proposed new library items are:

* The `lfsr.Algorithm` class to hold the generic parameters, which are all passed
  to the constructor by name:
    * `width`: positive integer width of LFSR state register
    * `polynominal`: integer constant indicating which feedback taps to use,
       excluding implicit 1 in the most-significant bit position
    * `reverse`: default-false boolean indicating whether the output sequence
      should be time-reversed, which can be achieved by mirroring the
      polynomial before logic generation
* `lfsr.Algorithm` implements `__call__()` which is used to create `lfsr.Parameters` instances
* The `lfsr.Parameters` class which holds an `lfsr.Algorithm` alongside
  implementation-specific settings, provided as named arguments to the
  `__call__()` method on `lfsr.Algorithm`:
    * `output_width`: positive integer number of output bits to generate per clock cycle, default 1
    * `structure`: either `"galois"` or `"fibonacci"` (default), which type of
        implementation to generate
    * `init`: positive integer initial state, default 1
* `lfsr.Parameters` has the following methods:
    * `compute()` to compute output words in software, returning an infinite iterator
    * `create()` to generate an `lfsr.Processor`
    * `algorithm()` returns the `lfsr.Algorithm` used to create this instance
* The `lfsr.Processor` class which is a `lib.wiring.Component` and implements
  the hardware generator, with the following signature:
    * `out: Out(parameters.output_width)`
    * `state: Out(algorithm.width)`
* An `lfsr.catalog` module which contains instances of `lfsr.Algorithm`
  corresponding to commonly used PRBS sequences and the 64b66b and 128b130b
  training sequences.

Further technical detail will be added as the draft implementation is developed.

## Drawbacks
[drawbacks]: #drawbacks

* LFSR generation could exist outside of the Amaranth stdlib; implementing it
  here may discourage experimentation in alternative interfaces.
* Some hardware or design patterns may still require other implementation
  techniques, although all options I'm aware of are covered in the proposed
  design or the unresolved questions and future possibilities below.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design implements LFSRs (and thus PRBS sequence generation) in a standard
way that should be useful to almost all users of LFSRs. There is a fairly small
design space for the actual hardware implementation and so it is unlikely an
alternative design would be significantly more efficient.

Some design alternatives include:

* We could simplify the implementation and API by only supporting Galois or
  only Fibonacci implementations, with the downside of potentially less
  efficient implementations on some platforms.
  I haven't researched how serious an impact this would be.
* We could not support multi-bit generation, but this is a useful feature for
  many applications.
* The split between Algorithm and Parameters mirrors that in `lib.crc`,
  but with fewer parameters we could consider merging them or making the
  Parameters settings be part of the Processor creation.

## Prior art
[prior-art]: #prior-art

LFSR generation is well established, some references are:

* Wikipedia articles:
    * https://en.wikipedia.org/wiki/Linear-feedback_shift_register
    * https://en.wikipedia.org/wiki/Pseudorandom_binary_sequence
* "Efficient Shift Registers, LFSR Counters, and Long Pseudo- Random Sequence Generators", application note by AMD:
    * https://docs.amd.com/v/u/en-US/xapp052
* CCITT O.151 and O.152 define some PRBS sequences

Implementations in Verilog and VHDL are widespread, including:

* https://github.com/alexforencich/verilog-lfsr/blob/master/rtl/lfsr.v
* https://opencores.org/projects/lfsrcountergenerator

The Glasgow project includes an implementation in Amaranth:

* https://github.com/GlasgowEmbedded/glasgow/blob/main/software/glasgow/gateware/lfsr.py
    * Uses `degree` instead of `width`, and a list of tap positions instead of
      `polynominal`, and `reset` instead of `init`
    * Also uses `EnableInserter` and `ResetInserter`
    * Does not support generation of multi-bit outputs, but the entire internal
      state might be used instead
    * Only supports Fibonacci implementations

## Unresolved questions
[unresolved-questions]: #unresolved-questions

Before this RFC is ready to merge, we should resolve:

* It might be useful to add logic for LFSR matching/error detection, which is
  a small addition and often useful
* Not sure if we should add explicit ready/valid signals or a reset signal,
  or leave those to EnableInserter/ResetInserter.
* Should we support internally-inverted LFSRs using XNOR operations, permitting
  a valid all-0s internal state to make initialisation easier e.g. on ASICs?
* Parallel generation of bits might change the internal state representation,
  so we may need to only guarantee it matching expected values with single-bit
  outputs.

The specific details of the parallel LFSR generation will be resolved during
implementation - there are a couple of possible ways to implement this, either
a similar matrix-based approach to the CRC module or by a virtual extension of
the shift register length.

## Future possibilities
[future-possibilities]: #future-possibilities

Some unresolved questions above may become a future possibility instead.
Beyond those, I don't foresee other future extensions.
