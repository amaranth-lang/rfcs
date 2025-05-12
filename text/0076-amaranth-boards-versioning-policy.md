- Start Date: (fill in with date at which the RFC is merged, 2025-05-12)
- RFC PR: [amaranth-lang/rfcs#76](https://github.com/amaranth-lang/rfcs/pull/76)
- Amaranth Issue: [amaranth-lang/amaranth#1595](https://github.com/amaranth-lang/amaranth/issues/1595)

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

A breaking change is anything that requires manual intervention for code that previously worked fine.
I.e. a change to a board file which previously had two pins incorrectly swapped would not be a breaking change; a change to names of resources across multiple boards would be.

This should be implemented by automatically releasing every commit to `main` whose HEAD message doesn't contain `[breaking change]` or `[breaking-change]` with a version like `v0.x.y` where `y` is the number of commits since the last breaking change, and automatically tagging every commit that does with a tag `v0.(x+1).0`.

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
