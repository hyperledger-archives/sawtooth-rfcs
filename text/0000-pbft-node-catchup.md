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
new block and confirming that it contains 2f+1 COMMIT messages for the block
that the node is currently trying to commit. If it does, then the node copies
enough COMMIT messages from the consensus seal into its own message log and
commits the block.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

When a new block is received, if there is already a block in progress, the
block's consensus seal will be inspected to determine whether it contains
enough information to commit the block that is in progress. This will be done
by adding the messages included in the seal to the node's message log if they
have not already been added and then checking whether enough messages are
present to commit the in progress block. If this check succeeds, the
in-progress block will be committed and the new block will be made the
in-progress block.

If the in progress block cannot be committed, then the new block message will
be added to the message backlog as usual.

# Drawbacks
[drawbacks]: #drawbacks

There are no known drawbacks to this change.

# Rationale and alternatives
[alternatives]: #alternatives

The first alternative that was considered was to allow nodes to receive
historic messages from other nodes' message logs, either by requesting messages
or requesting that a node catch another node up. However, this solution does
not work if a node is behind far enough that a checkpoint has happened, because
the necessary messages no longer exist anywhere.

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

A big open question is whether pre-prepare or prepare messages are required to
be kept in each node's message log. Currently, the consensus seal contains only
COMMIT messages, so if consensus is short-circuited, the pre-prepare and
prepare messages will be missing from that node's message log. So the
unresolved questions are:

- Do we need all the prepare and pre-prepare messages in the each nodes'
  backlog?
- What depends on prepare and pre-prepare messages?
- Or if we get the next block with the seal, can we skip adding those?

If pre-prepare and prepare messages are required, then the simplest thing to do
is include them in the consensus seal.
