- Feature Name: pbft_regular_view_changes
- Start Date: 2018-10-11
- RFC PR: (leave this empty)
- Sawtooth Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC describes an extension to the PBFT implementation to mitigate the
effect of compromised leader nodes. The RFC takes into account leaders unfairly
ordering batches within blocks (the "unfair ordering" problem) and leaders
intentionally not producing new blocks to stall the network (the "silent
leader" problem).

# Motivation
[motivation]: #motivation

This RFC proposes a solution to the "unfair ordering" problem for Sawtooth
PBFT. In general, fair ordering means that a Byzantine node can not order
transactions in a way that unfairly benefits that node. This problem is
important in voting-type algorithms with a long-term leader, such as many PBFT
variants. A malicious leader can, depending on the protocol, participate in the
protocol perfectly but manipulate the ordering to its benefit by:

1. Manipulating the order of a given set of transactions
2. Generating and inserting new transactions into the ordering
3. Withholding submitted transactions from the ordering

One thing that makes this problem hard is that, without application-specific
knowledge from the transactions, it is difficult to tell if an ordering
actually benefits the leader or if "random noise" caused the ordering to be
substantially different than some expected or fair ordering.

The current implementation of PBFT does not prevent a leader from unfairly
ordering transactions. This is a problem when a leader node has an incentive to
do so. This RFC aims to mitigate this problem in the interest of making PBFT
more resilient to bad actors.

This RFC also proposes a mitigating solution to the "silent leader" problem.
Sawtooth PBFT checks for leader liveness by starting a timer when a leader
proposes a new block. If the timer expires before the new block is committed,
the leader is suspected of being faulty and a view change is initiated.
However, if a leader in the NotStarted state never proposes a block, no timer
is started. This means that a faulty leader can "remain silent" and stall the
network without the other nodes being able to determine whether the silence is
because no new batches are arriving, or the leader is intentionally ignoring
them.

                NotStarted  <----------- Finished
    (start timer) |                           ^
                  v                           | (stop timer)
                PrePreparing -> ... -> Committing

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In order to mitigate the effects of the "unfair ordering" and "silent leader"
problems, Sawtooth PBFT will force regular view changes.

Forcing regular view changes measured by the number of blocks committed
mitigates the "unfair ordering" problem by giving every node a chance to be
unfair for a period of blocks. This is inspired by how the Tendermint consensus
algorithm handles the problem and how lottery-style algorithms handle the
problem, where a new leader can be elected for every block.

Forcing regular view changes while the network is idle mitigates the "silent
orderer" problem by limiting the amount of time a faulty leader can stall the
network.

The setting `sawtooth.consensus.pbft.forced_view_change_period` determines how
often, measured in blocks, a view change should be forced. After transitioning
from the Finished to NotStarted state, a node will check whether a view change
should be forced and, if so, vote for one.

The setting `sawtooth.consensus.pbft.idle_timeout` is set to an integer and
represents how long to wait in between committing a block and starting a round
of consensus on a new block before starting a view change. After transitioning
to the NotStarted state from any other state, if a node has not already started
a view, the node will start a timer based on this setting and, if it expires
before a new block is proposed, will vote for a view change.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Let n_force := the configured period in blocks between view changes as read
from `sawtooth.consensus.pbft.forced_view_change_period` and t_idle := the time
to wait between a block being committed and a new block being proposed before
forcing a view change as read from `sawtooth.consensus.pbft.idle_timeout`.

Regular, forced view changes occur only when the validator is in the NotStarted
state. Let n_seq := the sequence number after transitioning to NotStarted.
(Note that the sequence number is incremented at the same time as the block
height and are identical except when PBFT is not the consensus engine used at
genesis.) Immediately after transitioning to NotStarted, check if

    `n_seq % n_force == 0`

and if true, force a view change.

If a view change is not started by the above check, a timer is started which
will expire after t_idle has passed. If this timer expires without a BlockNew
update arriving, force a view change. A separate timer is used here instead of
moving where the existing timer is started for two reasons. First, an idle
network should not affect how long a network has to commit a block after
proposing one. Second, it is desirable that the time a network has to commit a
block and the time a network can remain idle before forcing a view change
be independently configurable.

# Drawbacks
[drawbacks]: #drawbacks

There are two drawbacks to these changes:

1. They do not completely solve the unfair ordering and silent leader problems
2. They rely on false positives being okay, which means more view changes than
   necessary are performed

# Rationale and alternatives
[alternatives]: #alternatives

A couple alternatives to the unfair ordering problem were considered. The first
was to have followers compute a heuristic or "probability that the ordering is
fair" and to start a new election if they determine the leader appears to be
manipulating the order. The second was inspired by the proposed method for
handling non-determinism in the original PBFT paper by Castro and Liskov.

One example of a heuristic that prevents excluding batches indefinitely here:
https://gist.github.com/aludvik/164a9c0a7419b758e54190c4a0dfa72b This solution
is somewhat inspired by the PoET z-test.

For the second alternative, the relevant portion of the PBFT paper is section
4.6 on non-determinism. (Paper found here:
http://pmg.csail.mit.edu/papers/osdi99.pdf) If we treat the batch ordering as
non-deterministic (because of network latency, dictionary iteration order,
etc.), then we can use some application of what is described in 4.6. An example
would be to have all followers vote on the batch ordering for the next block
before it is published and to require that the leader decide which batches to
include by some deterministic computation over 2f+1 of the votes. This creates
a smaller new problem to deal with, which is that malicious followers can vote
for bogus batch ids that must be solved.

It was determined that a complete solution to the problems described was beyond
the scope of this implementation of PBFT and a complete solution should come in
the form of an implementation of a new algorithm based on PBFT.

# Prior art
[prior-art]: #prior-art

The Tendermint consensus protocol also solves the unfair ordering problem by
electing a new leader for every block. This can be viewed as the same solution
as described above, with the number of blocks between view changes configured
to 1.

# Unresolved questions
[unresolved]: #unresolved-questions

- What should the default values for the new settings be?
- Should forced view changes be enabled by default? Should it be possible to
  turn them off?
- To what extent do forced view changes affect performance?
- How to guarantee all nodes have finished committing the in progress block
  before participating in the forced view change?
