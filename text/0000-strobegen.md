- Start Date: 2023-09-03
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Strobe Generator

## Summary
[summary]: #summary

Introduce components for generating enable streams for clock division.

## Motivation
[motivation]: #motivation

FPGAs tend to be way happier when the design uses a very low number of
real clock signals, and various clocks are in fact a combination of a
base, high-frequency clock and a regular enable signal that virtually
divides it.  In particular, that avoid all problems linked to
clock-domain-crossing.

This RFC introduces in amaranth.lib three components that generate
such an enable stream for any strobe frequency.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

amaranth.lib.strobegen introduces three components to generate a
slower clock domain from a higher-frequency one through an enable
stream.  They all run on the sync domain.

FixedStrobeGenerator takes the base frequency (of sync) and the strobe
frequency as constructor parameters and has a unique output, strobe,
with the pulses.

VariableStrobeGenerator takes the base frequency and has an input port
for the strobe frequency.  The output is the same.

VariableSimplifiedStrobeGenerator takes the accumulator width (it must
fit the base frequency minus one) and has a port for the strobe
frequency and for the delta value, which is strobe frequency - base
frequency.  The output is the same.  It uses only one adder instead of
two for VariableStrobeGenerator.


## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

FixedStrobeGenerator generates a train of pulses at a mean frequency of
strobe_frequency assuming the sync domain runs as base_frequency.

strobe_frequency must be strictly less than base_frequency.

Output:
- strobe: the generated pulses.

Constructor:
- def \_\_init\_\_(self, base_frequency, strobe_frequency)

Signature:
- strobe: Out(1)



VariableStrobeGenerator generates a train of pulses at a mean frequency of
strobe_frequency assuming the sync domain runs as base_frequency.

Input:
- strobe_frequency: the target frequency (must be stricly less than base_frequency).
        
Output:
- strobe: the generated pulses.

Constructor:
- def \_\_init\_\_(self, base_frequency)

Signature:
- strobe_frequency: In(bits_for(base_frequency - 1)
- strobe: Out(1)



VariableSimplifiedStrobeGenerator generates a train of pulses at a mean frequency of
strobe_frequency assuming the sync domain runs as base_frequency.
base_frequency-1 must fit in width bits.

This variant externalizes the computation of delta to, for
instance, a cpu core in order to remove a barely-used adder.
    
Input:
- strobe_frequency: the target frequency (must be stricly less than base_frequency).
- delta: result of the computation of strobe_frequency - base_frequency.

Output:
- strobe: the generated pulses.

Constructor:
- def \_\_init\_\_(self, width)

Signature:
- strobe_frequency: In(width)
- delta: In(width)
- strobe: Out(1)


## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

It's very, very tight.  Only one adder for FixedStrobeGenerator and
VariableSimplifiedStrobeGenerator, two for VariableStrobeGenerator,
and one mux.  The fixed generator automatically decays to a simple
counter when the strobe frequency is a divider of the base frequency
(e.g. it drops the mux).

It could be in amaranth-soc instead of amaranth.lib, but I think it's
more generic than what -soc gives.


## Drawbacks
[drawbacks]: #drawbacks

It can't generate a continous strobe (when strobe frequency == base
frequency) and that's not fixable without adding some otherwise not
very useful logic (force output to 1 when delta is zero).


## Future possibilities
[future-possibilities]: #future-possibilities

amaranth-soc versions hooked to wishbone CSRs could be rather useful
as clock generators for other wishbone components.
