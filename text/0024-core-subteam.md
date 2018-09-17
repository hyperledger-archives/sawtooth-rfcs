- Feature Name: core_subteam
- Start Date: 2018-08-30
- RFC PR: [hyperledger/sawtooth-rfcs#24](https://github.com/hyperledger/sawtooth-rfcs/pull/24)
- Sawtooth Issue: N/A

# Summary
[summary]: #summary

Create a new Core subteam to shepherd technical RFCs through the RFC process,
and make technical governance decisions; technical topics include those related
to the design and architecture of Sawtooth. This team is responsible for
ensuring long-term architectural consistency with the existing Sawtooth code
and overall project direction adopted by the Root team.

The Core subteam shall be the default subteam for technical decisions in
absence of another appropriate subteam.

# Motivation
[motivation]: #motivation

[Sawtooth RFC 0006](https://github.com/hyperledger/sawtooth-rfcs/blob/master/text/0006-sawtooth-governance.md)
introduced an explicit governance model for Hyperledger Sawtooth, including
a framework for creation of subteams. Several potential subteams are mentioned
in that RFC, including a Core subteam. The RFC process has generated many
technical RFCs which are currently pending, waiting for the Core subteam to be
formed so that shepherding and voting can proceed.

In order to ensure that we always have a functional technical decision making
process, the Core subteam will serve as the default subteam for technical
decisions.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

[Sawtooth RFC 0006](https://github.com/hyperledger/sawtooth-rfcs/blob/master/text/0006-sawtooth-governance.md)
describes the Core subteam as being "Responsible for the design, architecture,
performance, stability, and security of the core Sawtooth platform".

The Core subteam exists for the purpose of technical review and decision
making, in particular those which are in the form of an RFC.

Non-technical decisions should be left to the Root subteam or other subteam as
appropriate.  For example, as new components are added to Sawtooth, the Core
subteam may act in a capacity of technical review of the potentially incoming
components. However, the decision on whether to add the component may be left
with the Root subteam as a wider project decision. Thus it is possible that
RFCs may require approval from both the Root and Core subteams, if that is
deemed appropriate.

As with the other subteams detailed in the governance RFC, the specific
decisions the Core subteam will be tasked with include:

- Shepherding applicable RFCs
- Coming to consensus on whether to accept or reject RFCs
- Setting policy on what sort of changes require an RFC
- Coming to consensus on new members for the subteam
- Reviewing PRs

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

In order to achieve the long-term architectural consistency, membership in the
Core team should consist of individuals who have a fairly long history with the
project. For the subteam to be able to properly evaluate RFCs or decisions with
the proper context, the subteam membership should consist of members who have
a working knowledge of the existing code (especially "core" components such as
the validator) and/or overall existing architecture.

Therefore, the initial membership of the Core subteam is made up of Root team
members who are also Sawtooth core maintainers:

- @vaporos, Shawn Amundson, team lead
- @jsmitchell, James Mitchell
- @agunde406, Andrea Gunderson
- @aludvik, Adam Ludvik
- @peterschwarz, Peter Schwarz
- @dcmiddle, Dan Middleton

While members of this subteam have decision making capability, it is critical
to the health of the project that the wider community is involved in
discussions and that the subteam members represent the wider community to
the extent possible.

# Drawbacks
[drawbacks]: #drawbacks

None.

# Rationale and alternatives
[alternatives]: #alternatives

No alternatives have been discussed.

# Prior art
[prior-art]: #prior-art

This is based on the governance model laid out by
[Sawtooth RFC 0006](https://github.com/hyperledger/sawtooth-rfcs/blob/master/text/0006-sawtooth-governance.md),
as well as the work done by the
[Rust team](https://github.com/rust-lang/rfcs/blob/master/text/1683-docs-team.md)
to expand their own roster of subteams.

# Unresolved questions
[unresolved]: #unresolved-questions

None.
