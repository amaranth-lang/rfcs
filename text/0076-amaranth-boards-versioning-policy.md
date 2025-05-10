- Start Date: (fill in with date at which the RFC is merged, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#76](https://github.com/amaranth-lang/rfcs/pull/76)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# `amaranth-boards` versioning policy

## Summary
[summary]: #summary

Add a versioning policy for `amaranth-boards`.

## Motivation
[motivation]: #motivation

Packages with direct (e.g. git) dependencies aren't allowed on PyPI, so by not publishing releases on PyPI, we're blocking downstream projects depending on `amaranth-boards` from publishing their releases on PyPi without having to do workarounds.

## Explanation
[guide-level-explanation]: #guide-level-explanation

- Use a `0.x.y` versioning scheme.
- Increment `y` and do a new release any time there's been done significant additions.
- Increment `x`, reset `y` and do a new release any time there are breaking changes.

## Drawbacks
[drawbacks]: #drawbacks

None.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

`amaranth-boards` is mainly a collection of independent board definitions where most changes consists of adding or updating definitions for one specific target.

Most changes will have none to limited impact on existing code, and a simple policy reducing friction for new additions and fixes is more useful than having a strict deprecation policy with regular major releases.

## Prior art
[prior-art]: #prior-art

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

None.
