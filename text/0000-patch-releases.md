- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)

# Patch releases

## Summary
[summary]: #summary

Change Amaranth versioning from `major.minor` to `major.minor.patch` after version 0.4, and define the backport policy for patch releases.

## Motivation
[motivation]: #motivation

Amaranth 0.3 was released on 2021-12-16; almost two years ago. Several important bugs have been fixed in `main` since, most notably depending on a version of `Jinja2` that is no longer installable. At the moment the policy is to issue only `major.minor` releases, which was OK in the early days but no longer fits the project.

We should change the policy that is used for the next Amaranth release and later ones.

## Explanation
[guide-level-explanation]: #guide-level-explanation

Amaranth feature releases all have the version of `major.minor.0`. The policy for these releases is unchanged and is tied to our two-step process for making breaking changes.

In addition to these releases, Amaranth now has bug-fix releases with the `major.minor.patch` versions. These are intended to address the need of the community to have bugs fixed before a next feature release can be made, and the policy is designed to minimize developer time spent on them.

Bug-fix releases are made when all of the following conditions are satisfied:
- There is an issue that is fixed in the `main` branch.
- A member of the community requests this issue to be fixed in a point release.
- It is possible to fix the issue such that there is a high degree of confidence that the change will not break existing code using Amaranth with a `~=major.minor` version constraint.
- A community member steps up to backport the fix to the release branch.
  - This could be one of the Amaranth maintainers, or anyone else. Amaranth maintainers have no obligation to back-port any fix.

## Drawbacks
[drawbacks]: #drawbacks

This creates additional work for maintainers.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- It would be possible to backport all feasible fixes as a policy.
  - This would significantly increase maintainer workload.
- It is possible to keep the current policy.
  - Because we do not control all of the upstream dependencies (including Python), this seems untenable.

## Prior art
[prior-art]: #prior-art

Rust has an even stricter bug-fix release policy: the Rust project only issues patch releases in cases of security issues, widespread miscompilations, or unintentional breaking changes.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

Should Amaranth SoC adopt the same policy?

## Future possibilities
[future-possibilities]: #future-possibilities

Eventually, Amaranth may gain release engineers who will maintain long-living release branches.
