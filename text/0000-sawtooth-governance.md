- Feature Name: governance
- Start Date: 2018-03-16
- RFC PR: (leave this empty)
- Sawtooth Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes an explicit governance structure for the Hyperledger Sawtooth
project and an initial list of teams that make up this structure. It also
defines several levels of contributor status for component repositories.

# Motivation
[motivation]: #motivation

Many important aspects of the Sawtooth governance model have been based on
implicit norms and expectations. As the project continues to grow, it is
important that this governance model be made explicit with respect to decision
making authority, repository ownership, and the establishment of the project's
direction and vision. To this end, this RFC seeks to make explicit existing
norms that can be carried forward as the project grows and to establish new
policies and procedures where additional structure is needed.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

At a high-level, the Sawtooth governance model consists of two arms:

1. A hierarchy of teams responsible for the continued growth and success of the
   project
2. A set of contributor levels used to manage component repository permissions

The team hierarchy contains a high-level root team and a set of more focused
subteams, each of which is led by a member of the root team. Having a member of
the root team lead each of the subteams is designed to promote communication
and alignment of vision across the project.

The contributor levels consist of three levels: reviewer, committer and
maintainers. These levels define what actions a contributor can perform for a
particular repository.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Project Team Hierarchy

The Sawtooth governance model consists of a root team and a number of subteams.

### Root Team

The root team serves as leadership for the Sawtooth project as a whole. In
particular, it:

* Sets the overall direction and vision for the project. This means setting the
  core values that are used when making decisions about technical tradeoffs. It
  means steering the project toward specific use cases where Sawtooth can have
  a major impact. It means leading the discussion, and writing RFCs for, major
  initiatives in the project.

* Sets the priorities and release schedule. Design bandwidth is limited, and
  it's dangerous to try to grow the project too quickly; the root team makes
  some difficult decisions about which areas to prioritize for new design,
  based on the core values and target use cases.

* Focuses on broad, cross-cutting concerns. The root team is specifically
  responsible for taking a global view of the project, to make sure the pieces
  are fitting together in a coherent way.

* Spins up or shuts down subteams. Over time, it may make sense to expand the
  set of subteams or it may make sense to have temporary "strike teams" that
  focus on a particular, limited task.

The root team includes stakeholders who are actively involved in the Sawtooth
community and have expertise within the project. Subteam leaders are chosen
from the root team.

Members are added to the root team by unanimous vote of the existing root team
members. Members are removed by unanimous vote of the existing root team,
excluding the member being removed.

### Subteams

Each of the subteams has responsibility over a specific domain of the Sawtooth
project, which is established at its inception. Specifically, subteams are
responsible for:

* Shepherding RFCs that are within the purview of the subteams domain. That
  means (1) ensuring that stakeholders are aware of the RFC, (2) working to
  tease out various design tradeoffs and alternatives, and (3) helping build
  consensus.

* Accepting or rejecting RFCs within the subteam's domain.

* Setting policy on what changes in the subteam area require RFCs, and
  reviewing direct PRs for changes that do not require an RFC.

Subteams make it possible to involve a larger, more diverse group in the
decision-making process. In particular, they should involve a mix of:

* Sawtooth project leadership, in the form of at least one root team member
  (the leader of the subteam).

* Area experts: people who have a lot of interest and expertise in the subteam
  area, but who may be far less engaged with other areas of the project.

* Stakeholders: people who are strongly affected by decisions in the subteam
  area, but who may not be experts in the design or implementation of that
  area. Whenever possible, it is crucial that users of Sawtooth have a seat at
  the table, to make sure we are actually addressing real-world needs.

Members should have demonstrated a good sense for design and dealing with
tradeoffs, an ability to work within a framework of consensus, and of course
sufficient knowledge about or experience with the subteam area. Leaders should
in addition have demonstrated exceptional communication, design, and people
skills. They must be able to work with a diverse group of people and help lead
it toward consensus and execution.

Each subteam is led by a member of the root team. The leader is responsible
for:

* Setting up the subteam:

    * Deciding on the initial membership of the subteam (in consultation with
      the root team).

    * Working with subteam members to determine and publish subteam policies
      and mechanics, including the way that subteam members join or leave the
      team (which should be based on subteam consensus).

* Communicating root team vision downward to the subteam.

* Alerting the root team to subteam RFCs that need global, cross-cutting
  attention, and to RFCs that have entered the "final comment period".

* Ensuring that RFCs and PRs are progressing at a reasonable rate and
  re-assigning shepherds/reviewers as needed.

* Making final decisions in cases of contentious RFCs that are unable to reach
  consensus otherwise (should be rare).

The way that subteams communicate internally and externally is left to each
subteam to decide, but:

* Technical discussion should take place as much as possible in public,
  ideally on the RFC PRs and on the chat server.

* Subteams should actively seek out discussion and input from stakeholders who
  are not members of the team.

* Subteams should have some kind of regular meeting or other way of making
  decisions. The content of this meeting should be summarized with the
  rationale for each decision -- and, as explained below, decisions should
  generally be about weighting a set of already-known tradeoffs, not discussing
  or discovering new rationale.

* Subteams should regularly publish the status of RFCs, PRs, and other news
  related to their area. Ideally, this should be done to the mailing list or
  chat server.

### Decision-making

#### Consensus

Sawtooth uses a form of [consensus decision-making][consensus]. In a nutshell
the premise is that a successful outcome is not where one side of a debate has
"won", but rather where concerns from *all* sides have been addressed in some
way. **This emphatically does not entail design by committee, nor compromised
design**. Rather, it's a recognition that

> ... every design or implementation choice carries a trade-off and numerous
> costs. There is seldom a right answer.

Breakthrough designs sometimes end up changing the playing field by eliminating
tradeoffs altogether, but more often difficult decisions have to be made. **The
key is to have a clear vision and set of values and priorities**, which is the
root team's responsibility to set and communicate, and the subteam's
responsibility to act upon.

Whenever possible, we seek to reach consensus through discussion and design
revision. Concretely, the steps are:

* Initial RFC proposed, with initial analysis of tradeoffs.
* Comments reveal additional drawbacks, problems, or tradeoffs.
* RFC revised to address comments, often by improving the design.
* Repeat above until "major objections" are fully addressed, or it's clear that
  there is a fundamental choice to be made.

Consensus is reached when most people are left with only "minor" objections,
i.e., while they might choose the tradeoffs slightly differently they do not
feel a strong need to *actively block* the RFC from progressing.

One important question is: consensus among which people, exactly? Of course, the
broader the consensus, the better. But at the very least, **consensus within the
members of the subteam should be the norm for most decisions.** If the root team
has done its job of communicating the values and priorities, it should be
possible to fit the debate about the RFC into that framework and reach a fairly
clear outcome.

[consensus]: http://en.wikipedia.org/wiki/Consensus_decision-making

#### Lack of consensus

In some cases, though, consensus cannot be reached. These cases tend to split
into two very different camps:

* "Trivial" reasons, e.g., there is not widespread agreement about naming, but
  there is consensus about the substance.

* "Deep" reasons, e.g., the design fundamentally improves one set of concerns
  at the expense of another, and people on both sides feel strongly about it.

In either case, an alternative form of decision-making is needed.

* For the "trivial" case, usually either the RFC shepherd or subteam leader
  will make an executive decision.

* For the "deep" case, the subteam leader is empowered to make a final
  decision, but should consult with the rest of the root team before doing so.

#### Final Comment Period (FCP)

Each RFC has a shepherd drawn from the relevant subteam. The shepherd is
responsible for driving the consensus process -- working with both the RFC
author and the broader community to dig out problems, alternatives, and
improved design, always working to reach broader consensus.

At some point, the RFC comments will reach a kind of "steady state", where no
new tradeoffs are being discovered, and either objections have been addressed,
or it's clear that the design has fundamental downsides that need to be
weighed.

At that point, the shepherd will announce that the RFC is in a "final comment
period" (which lasts for one week). This is a kind of "last call" for strong
objections to the RFC. **The announcement of the final comment period for an
RFC should be very visible**; it should be included in the subteam's periodic
communications.

After the final comment period, the subteam can make a decision on the RFC. The
role of the subteam at that point is *not* to reveal any new technical issues
or arguments; if these come up during discussion, they should be added as
comments to the RFC, and it should undergo another final comment period.

Instead, the subteam decision is based on **weighing the already-revealed
tradeoffs against the project's priorities and values** (which the root team is
responsible for setting, globally). In the end, these decisions are about how
to weight tradeoffs. The decision should be communicated in these terms,
pointing out the tradeoffs that were raised and explaining how they were
weighted, and **never introducing new arguments**.

### Initial Subteams

The following is a proposed list of initial subteams which is subject to change
at the discretion of the root team, once it has formed.

**Core** - Responsible for the design, architecture, performance, stability,
 and security of the core Sawtooth platform

**Application SDKs** - Responsible for the design and consistency of
user-facing SDKs for application development

**Consensus** - Responsible for the supported consensus algorithms and
the consensus interface

**Seth** - Responsible for the Sawtooth integration with Ethereum

**Community Outreach** - Responsible for writing and hosting documentation,
managing the main Sawtooth website, promoting Sawtooth to the public, managing
demos, and providing the community with training and support

**Release Management** - Responsible for release related issues such as
dependency management, license compliance, version control, and upgrade and
backwards compatibility

**Continuous Integration** - Responsible for test and build environments and
deployment artifacts

## Contributor Permission Levels

The Sawtooth community encourages contributions to the project from all
interested individuals. The project defines three levels of permissions that
determine what actions a contributor can perform on a particular repository.

A **reviewer** has "read" permission, meaning they can review pull requests.

A **committer** has "write" permission, meaning they can merge pull requests
once they have been approved.

Any community member can be promoted to a reviewer or committer by a
maintainer. Reviewers and committers lose their status when it is removed by a
maintainer. This is usually due to inactivity, but can also be used for
moderation.

A **maintainer** has permission to approve changes. All pull-requests must be
approved by at least 2 maintainers before a committer may merge it.
Maintainers are expected to:

- Carefully review pull requests and leave thoughtful, constructive feedback
- Ensure the component is up-to-date with patches and security updates
- Participate in community discussions related to their component

A committer can be promoted to a maintainer by unanimous vote by the existing
maintainers. Maintainers lose their status due to:

- Inactivity of 12 Months
- Unanimous vote by all other Maintainers

## Code of Conduct

Members of the Sawtooth community are expected to abide by the [Hyperledger Code of Conduct][hyperledger-coc].
Violations of this code of conduct by any community member may result in
disciplinary action at the discretion of the root team (by unanimous vote,
excluding the offending member if applicable).

[hyperledger-coc]: https://wiki.hyperledger.org/community/hyperledger-project-code-of-conduct

# Drawbacks
[drawbacks]: #drawbacks

This RFC makes explicit a governance model. Since this was previously implicit,
some existing community members may feel offended by not being given a
permission level that is consistent with their expectations or by not being
placed on a team whose domain they feel responsible for.

# Rationale and alternatives
[alternatives]: #alternatives

The [Mozilla governance model](https://www.mozilla.org/en-US/about/governance/policies/module-ownership/ )
was also considered, however a hierarchical model was focused on ensuring the
project maintained a consistent vision was preferred.

# Prior art
[prior-art]: #prior-art

This RFC is directly derived from the [Rust Governance RFC][rust-gov] and
copyright of the text can be determined in that repository. The repository is
dual-licensed, and we are using the RFC under Apache 2.0. The LICENSE-APACHE
file from the original repository is included as LICENSE in this repository.

[rust-gov]: https://github.com/rust-lang/rfcs/blob/master/text/1068-rust-governance.md

The contributor statuses proposed in this RFC are loosely based on ideas from the [Hyperledger Fabric Contributor Guide](https://hyperledger-fabric.readthedocs.io/en/release-1.0/CONTRIBUTING.html)
and preexisting community norms within the Sawtooth project.

# Unresolved questions
[unresolved]: #unresolved-questions

- Who should be on the initial root team roster?
