- Start Date: (fill in with date at which the RFC is merged, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#77](https://github.com/amaranth-lang/rfcs/pull/77)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Stream port name conventions

## Summary
[summary]: #summary

Settle on and document name conventions for stream components.

## Motivation
[motivation]: #motivation

Having a name convention for stream ports makes stream components easier to work with and allows automating the connection of pipelines like this:

```
pipeline = [component_a, component_b, component_c]
for upstream, downstream in itertools.pairwise(pipeline):
    wiring.connect(m, upstream.output, downstream.input)
```

## Guide- and reference-level explanation
[guide-level-explanation]: #guide-level-explanation

Document the following conventions:

- When a component has a single primary input stream, the port should be named *TBD*.
- When a component has a single primary output stream, the port should be named *TBD*.

Update existing stream components to follow these conventions.

## Drawbacks
[drawbacks]: #drawbacks

None.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Without a common convention, components will be written with different naming schemes, requiring more care when interconnecting them.

## Prior art
[prior-art]: #prior-art

Stream ports in LiteX components are conventionally named `sink` and `source`.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Which exact names should we settle on?
  - `i` / `o`
  - `input` / `output`
  - `w_stream` / `r_stream`
  - `sink` / `source`
  - Others?

## Future possibilities
[future-possibilities]: #future-possibilities

A `connect_pipeline()` function can be added that takes an iterable of components and connects them in sequence like shown above.
