- Feature Name: pbft_node_catchup
- Start Date: 2018-11-2
- RFC PR: (leave this empty)
- Sawtooth Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC describes a modification to the PBFT implementation to allow new nodes
or nodes that have fallen behind to quickly catch up with the rest of the
network. It builds on the consensus seal concept described in an earlier RFC.

# Motivation
[motivation]: #motivation

A major flaw in the current PBFT implementation is that a node that has fallen
behind or a node that has just joined the network has no way to catch up with
the rest of the network. This is because, in order to commit a block, a node
must receive 2f+1 messages from other nodes agreeing that the block should be
committed. However, nodes do not have a way to request historic messages from
each other and nodes only send out commit messages once when they are ready to
commit. If a node is not running when this happens, it will never receive the
messages and can never commit a block.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In order for new nodes and nodes that have fallen behind to quickly catch up
with the rest of the network, Sawtooth PBFT provides a mechanism by which the
normal consensus process can be short-circuited without weakening the consensus
protocol's guarantees.

Under normal operation, when a new block is received by a PBFT node, it does an
initial validation and, if the validation is successful, broadcasts a COMMIT
message to its peers indicating that it is ready to commit the new block. Prior
to committing the block, the PBFT node waits to receive 2f+1 COMMIT messages
from other nodes. After receiving these COMMIT messages, the node knows that
the block is safe to commit and will not need to be reverted later.

If, while waiting to receive 2f+1 COMMIT messages from its peers, a node
receives the next block in the chain, the node has an opportunity to
short-circuit consensus. This is done by validating the consensus seal in the
new block and confirming that it is a valid proof for the block that the node is
currently trying to commit. If it is, then the node copies the COMMIT messages
from the consensus seal into its own message log and commits the block.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

When a new block is received by a node that has not yet committed the previous
block, the new block's consensus seal will be inspected to determine whether it
contains a valid seal to commit the previous block. This will be done by adding
the messages included in the seal to the node's message log and committing the
previous block.

# Drawbacks
[drawbacks]: #drawbacks

There are no known drawbacks to this change.

# Rationale and alternatives
[alternatives]: #alternatives

The first alternative that was considered was to allow nodes to receive
historic messages from other nodes' message logs, either by requesting messages
or requesting that a node catch another node up. However, this solution does
not work if a node is behind far enough that the other nodes have already
cleaned their logs and no longer have the relevant messages.

Another alternative that was considered is putting nodes into a special
"catchup" mode when they startup up if they are behind. The problem with this
alternative is there is no clear point where the node should transition from
"catchup" to regular mode and it does not solve the problem where a node misses
a bunch of messages because of a networking problem.

# Prior art
[prior-art]: #prior-art

The original PBFT paper describes this problem briefly. From Section 4.3:

> For the safety condition to hold, messages must be kept in a replica’s log
> until it knows that the requests they concern have been executed by at least
> 1 non-faulty replicas and it can prove this to others in view changes. In
> addition, if some replica misses messages that were discarded by all
> non-faulty replicas, it will need to be brought up to date by transferring
> all or a portion of the service state. Therefore, replicas also need some
> proof that the state is correct.

The "proof" that the paper states is required is provided by the consensus
seal, which is signed and contains the necessary messages (state).

# Unresolved questions
[unresolved]: #unresolved-questions

No unresolved questions
