- Start Date: 2025-06-09
- RFC PR: [amaranth-lang/rfcs#77](https://github.com/amaranth-lang/rfcs/pull/77)
- Amaranth Issue: [amaranth-lang/amaranth#1615](https://github.com/amaranth-lang/amaranth/issues/1615)

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
    wiring.connect(m, upstream.o, downstream.i)
```

## Guide- and reference-level explanation
[guide-level-explanation]: #guide-level-explanation

Document the following conventions:

- Whenever the name of a stream member of the component interface includes an indication of its direction, the indication must match one of the following forms:
  - For `In` members, either the complete name is `i` (for components that have one primary input), or the name must start with `i_` (for more complex components).
  - For `Out` members, either the complete name is `o` (for components that have one primary input), or the name must start with `o_` (for more complex components).

- It is also acceptable to have a name that doesn't indicate directionality explicitly, e.g. `requests` or `packets`.

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

None.

## Future possibilities
[future-possibilities]: #future-possibilities

A `connect_pipeline()` function can be added that takes an iterable of components and connects them in sequence like shown above.
