# Amaranth RFCs - [RFC Book](https://amaranth-lang.org/rfcs/)
[Amaranth RFCs]: #amaranth-rfcs

The "RFC" (request for comments) process is intended to provide a consistent and controlled path for changes to Amaranth (such as new features) so that all stakeholders can be confident about the direction of the project.

Many changes, including bug fixes and documentation improvements can be implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put through a bit of a design process and produce a consensus among the Amaranth community.

## Table of Contents
[Table of Contents]: #table-of-contents

  - [Opening](#amaranth-rfcs)
  - [Table of Contents]
  - [When you need to follow this process]
  - [Before creating an RFC]
  - [What the process is]
  - [The RFC life-cycle]
  - [Reviewing RFCs]
  - [Implementing an RFC]
  - [Acknowledgements]
  - [License]

## When you need to follow this process
[When you need to follow this process]: #when-you-need-to-follow-this-process

You need to follow this process if you intend to make "substantial" changes to [core Amaranth language][amaranth]. In the future you will need to follow this process for the [standard I/O library][amaranth-stdio] and the [System-on-Chip library][amaranth-soc] as well, but at the moment they are not covered.

What constitutes a "substantial" change is evolving based on community norms and varies depending on what part of the ecosystem you are proposing to change, but may include the following:

- Any semantic or syntactic change to the language (`amaranth.hdl`) that is not a bugfix.
- Behavioral changes to the standard library (`amaranth.lib`).
- Behavioral changes to the toolchain interface (`amaranth.vendor`).

Some changes do not require an RFC:

- Rephrasing, reorganizing, refactoring, or otherwise "changing shape does not change meaning".
- Additions that strictly improve objective, numerical quality criteria (warning removal, speedup, better platform coverage, handling more errors, etc.)

If you submit a pull request to implement a new feature without going through the RFC process, it may be closed with a polite request to submit an RFC first. When in doubt, please open an issue to discuss the feature first and one of the core maintainers will say if the change requires an RFC or not.

[amaranth]: https://github.com/amaranth-lang/amaranth
[amaranth-stdio]: https://github.com/amaranth-lang/amaranth-stdio
[amaranth-soc]: https://github.com/amaranth-lang/amaranth-soc

## Before creating an RFC
[Before creating an RFC]: #before-creating-an-rfc

A hastily-proposed RFC can hurt its chances of acceptance. Low quality proposals, proposals for previously-rejected features, or those that don't fit into the near-term roadmap, may be quickly rejected, which can be demotivating for the unprepared contributor. Laying some groundwork ahead of the RFC can make the process smoother.

Although there is no single way to prepare for submitting an RFC, it is generally a good idea to pursue feedback from other project developers beforehand, to ascertain that the RFC may be desirable; having a consistent impact on the project requires concerted effort toward consensus-building.

The most common preparations for writing and submitting an RFC include talking the idea over on our IRC channel, [#amaranth-lang at libera.chat](https://web.libera.chat/#amaranth-lang), or opening an issue on the corresponding repository to gather feedback.

## What the process is
[What the process is]: #what-the-process-is

In short, to get a major feature added to Amaranth, one must first get the RFC merged into the RFC repository as a markdown file. At that point the RFC is "active" and may be implemented with the goal of eventual inclusion into Amaranth.

- Fork the RFC repository.
- Copy 0000-template.md to text/0000-my-feature.md (where "my-feature" is descriptive). Don't assign an RFC number yet; This is going to be the PR number and we'll rename the file accordingly if the RFC is accepted.
- Fill in the RFC. Put care into the details: RFCs that do not present convincing motivation, demonstrate lack of understanding of the design's impact, or are disingenuous about the drawbacks or alternatives tend to be poorly-received.
- Submit a pull request. As a pull request the RFC will receive design feedback from the larger community, and the author should be prepared to revise it in response.
- Now that your RFC has an open pull request, use the issue number of the PR to update your 0000- prefix to that number.
- Build consensus and integrate feedback. RFCs that have broad support are much more likely to make progress than those that don't receive any comments. Feel free to reach out to the RFC assignee in particular to get help identifying stakeholders and obstacles.
- RFCs rarely go through this process unchanged, especially as alternatives and drawbacks are shown. You can make edits, big and small, to the RFC to clarify or change the design, but make changes as new commits to the pull request, and leave a comment on the pull request explaining your changes. Specifically, do not squash or rebase commits after they are visible on the pull request.
- At some point, a core maintainer will make a decision on the disposition for the RFC (merge or close).
  - This step is taken when enough of the tradeoffs have been discussed that the core maintainer is in a position to make a decision. That does not require consensus amongst all participants in the RFC thread (which is usually impossible). However, the argument supporting the disposition on the RFC needs to have already been clearly articulated, and there should not be a strong consensus against that position.

## The RFC life-cycle
[The RFC life-cycle]: #the-rfc-life-cycle

Once an RFC becomes "active" then authors may implement it and submit the feature as a pull request to the corresponding repository. Being "active" is not a rubber stamp, and in particular still does not mean the feature will ultimately be merged; it does mean that in principle all the major stakeholders have agreed to the feature and are amenable to merging it.

Furthermore, the fact that a given RFC has been accepted and is "active" implies nothing about what priority is assigned to its implementation, nor does it imply anything about whether a developer has been assigned the task of implementing the feature. While it is not necessary that the author of the RFC also write the implementation, it is by far the most effective way to see an RFC through to completion: authors should not expect that other project developers will take on responsibility for implementing their accepted feature.

Modifications to "active" RFCs can be done in follow-up pull requests. We strive to write each RFC in a manner that it will reflect the final design of the feature; but the nature of the process means that we cannot expect every merged RFC to actually reflect what the end result will be at the time of the next major release.

In general, once accepted, RFCs should not be substantially changed. Only very minor changes should be submitted as amendments. More substantial changes should be new RFCs, with a note added to the original RFC. Exactly what counts as a "very minor change" is up to the core maintainers to decide.

## Reviewing RFCs
[Reviewing RFCs]: #reviewing-rfcs

While the RFC pull request is up, the core maintainers may schedule meetings with the author and/or relevant stakeholders to discuss the issues in greater detail, and the topic may be discussed at weekly meetings. In either case a summary from the meeting will be posted back to the RFC pull request.

A core maintainer makes final decisions about RFCs after the benefits and drawbacks are well understood. These decisions can be made at any time, but the core maintainer will regularly issue decisions. When a decision is made, the RFC pull request will either be merged or closed. In either case, if the reasoning is not clear from the discussion in thread, the core maintainer will add a comment describing the rationale for the decision.

## Implementing an RFC
[Implementing an RFC]: #implementing-an-rfc

Some accepted RFCs represent vital features that need to be implemented right away. Other accepted RFCs can represent features that can wait until some arbitrary developer feels like doing the work. Every accepted RFC has an associated issue tracking its implementation in the corresponding repository.

The author of an RFC is not obligated to implement it. Of course, the RFC author (like any other developer) is welcome to post an implementation for review after the RFC has been accepted.

If you are interested in working on the implementation for an "active" RFC, but cannot determine if someone else is already working on it, feel free to ask (e.g. by leaving a comment on the associated issue).

## Acknowledgements
[Acknowledgements]: #acknowledgements

The process described in this document is based on the [Rust RFC process](https://github.com/rust-lang/rfcs). It has been simplified to match the needs of the much smaller Amaranth community; in particular, policy (including the RFC process itself) is not currently defined through the RFC process.

## License
[License]: #license

This repository is licensed under the [MIT license](LICENSE.txt).
