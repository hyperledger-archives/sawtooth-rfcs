- Feature Name: supply_chain_subteam
- Start Date: 2018-05-04
- RFC PR: [hyperledger/sawtooth-rfcs#11](https://github.com/hyperledger/sawtooth-rfcs/pull/11)
- Sawtooth Issue:

# Summary
[summary]: #summary

Create a new Supply Chain subteam to shepherd RFCs through the process, and
make governance decisions regarding the design and direction of Sawtooth Supply
Chain.

# Motivation
[motivation]: #motivation

[Sawtooth RFC 0006](https://github.com/hyperledger/sawtooth-rfcs/blob/master/text/0006-sawtooth-governance.md)
introduced an explicit governance model for Hyperledger Sawtooth, including a
framework for creation of subteams. Several potential subteams are mentioned in
that RFC, and although not explicitly mentioned in the RFC, a Supply Chain
subteam was informally discussed at the time.

The Sawtooth Supply Chain application is actively being designed and developed,
and has attracted substantial community interest. Though implemented as a
Sawtooth application, it is being built out into a robust platform in its own
right. As we move forward, it is important that this application be expanded in
a transparent manner which maximizes the benefits of its community.

A subteam is desired to increase community involvement by fully embracing the
Sawtooth RFC process, and providing the necessary shepherding. In addition, the
subteam can provide focus on non-RFC topics which relate specifically to
Sawtooth Supply Chain.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Continued development of Sawtooth Supply Chain should be based on open
technologies and standards, following best practices laid out by Hyperledger
Sawtooth and other open-source platforms. Part of this effort includes using
the established RFC process to invite the larger community to comment on Supply
Chain designs, or propose their own. Using this process necessitates
establishing of a Supply Chain subteam.

The new governing team will be responsible for Sawtooth Supply Chain, as well
as the expansion of Sawtooth into supply chains generally. This will primarily
include decisions about the future to the actual
`hyperledger/sawtooth-supply-chain` repo, but could potentially cover any
exploration of supply chains and provenance tracking the Sawtooth team chooses
to pursue.

As with the other subteams detailed in the governance RFC, the specific
decisions the Supply Chain subteam will be tasked with include:

- Shepherding applicable RFCs
- Coming to consensus on whether to accept or reject RFCs
- Setting policy on what sort of changes require an RFC
- Coming to consensus on new members for the subteam
- Reviewing PRs

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The initial membership of the Supply Chain subteam should include:

- @vaporos, Shawn Amundson, team lead
- @ineffectualproperty, Kelly Olson
- @jsmitchell, James Mitchell
- @delventhalz, Zac Delventhal
- @agunde406, Andrea Gunderson

# Drawbacks
[drawbacks]: #drawbacks

Adding a new subteam will entail additional work, and added complexity to
Sawtooth's governing structure. However, this proposed team has a similar scope
to the already established Seth team, and any Sawtooth application actively
being developed, in a community-driven fashion, and coordinating with the core
team, should be fit into the governance structure.

# Rationale and alternatives
[alternatives]: #alternatives

One alternative is simply not to have a Supply Chain subteam. That is the
current status quo, and it prevents using the RFC process to include the
community in future design. Without a subteam, it follows that
development on Supply Chain will continue to be developed ad hoc with no
governance and limited community involvement in its roadmap.

Another alternative would be to have an umbrella team that covered Seth, Supply
Chain, and any other applications built on Sawtooth. However, more small teams
with narrower concerns seems preferable to the one giant team this would
inevitably become.

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
