- Feature Name: pbft_consensus_seal
- Start Date: 2018-10-11
- RFC PR: (leave this empty)
- Sawtooth Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC describes a modification to the PBFT implementation to ensure that the
finality of committed blocks can be verified by an external observer after the
fact.

# Motivation
[motivation]: #motivation

One shortcoming of the current PBFT implementation is that it is not possible
to validate after the fact that committed blocks were committed only after
receiving 2f+1 votes to commit the block. This RFC aims to solve this problems
in the interest of making PBFT more production ready.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When a new block is received by a PBFT node, it does an initial validation and,
if the validation is successful, broadcasts a COMMIT message to its peers
indicating that it is ready to commit the new block. Prior to committing the
block, the PBFT node waits to receive 2f+1 COMMIT messages from other nodes.
After receiving these COMMIT messages, the node knows that the block is safe to
commit and will not need to be reverted later. At a later point, the node can
also go back and verify that it received 2f+1 COMMIT messages.

There are two problems with the way this is currently implemented:

1. The node cannot, at a later time, prove that it received the 2f+1 COMMIT
   messages _before_ it committed the block, only that it has them at that
   time.
2. An independent observer cannot verify that the block was committed after
   receiving 2f+1 COMMIT messages without requesting the messages directly from
   the node's log.

For 1, it does not seem like this problem can be solved. The solution would
require including 2f+1 signatures from other nodes that they are ready to
commit a given block id in the block itself. However the block id is
constructed by signing the contents of the block, so the 2f+1 signatures would
need to be available before the block id is created.

For 2, we can provide a partial solution by requiring each new block that is
created contain 2f+1 COMMIT messages for the previous block in its consensus
payload. This creates a public log from which any external observer can
validate that consensus was performed on each block, without depending on a
node being available to respond to queries, or that some node still has the
historic logs available.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The consensus seal for PBFT consists primarily of 2f+1 or more signed COMMIT
messages from other nodes participating in consensus. In order to guarantee
that the seal cannot be reused with another block, we require that it also
contain the new block's summary and the previous block's signature in the seal.

We define the consensus seal for PBFT as the following protobuf message, to be
serialized and included in the block's consensus payload field:

    message PbftSignedCommitVote {
      // Serialized ConsensusPeerMessageHeader
      bytes header = 1;
      bytes header_signature = 2;

      // Serialized ConsensusPeerMessage
      bytes message = 3;
    }

    message PbftSeal {
      bytes previous_id = 1;
      bytes summary = 2;
      repeated PbftSignedCommitVote previous_commit_votes = 3;
    }

Before a new block can be finalized by the leader, a `PbftSeal` must be
constructed with 2f+1 `PbftSignedCommitVote` messages included. Prior to
followers committing a block, they must verify the `PbftSeal`. To verify the
seal, the node must:

1. Verify the `previous_id` field in the seal matches the current block
2. Verify the `summary` field matches the block's summary
3. For each vote:
   1. Verify the `header_signature` of each vote matches the
      `signer_public_key` in the header
   2. Verify the SHA512 digest of the `message` field matches the
      `message_sha512` field in the header


# Drawbacks
[drawbacks]: #drawbacks

There are two drawbacks to this change. First, additional signature and digest
verification for 2f+1 messages at every block creates additional work that
scales up with the size of the network, causing further slowdowns on large
networks. Second, requiring the consensus seal be included in the block before
it is finalized prevents a leader from finalizing a block before the previous
block has been committed, which prevents an aggressive strategy of optimistic
block publishing.

Lastly, while it is possible for an observer to validate that a block commit is
correct by checking the consensus seal of the next block, it is not possible
for an observer to validate that the most recently committed block was
correctly committed.

# Rationale and alternatives
[alternatives]: #alternatives

Other consensus algorithms have attempted to solve this problem by including
the consensus seal in an optional, unsigned field along with the block. (https://github.com/ethereum/EIPs/issues/650)
This provides similar guarantees as above, but there are a number of drawbacks
to this approach:

1. The burden of producing the seal is distributed to all nodes
2. There is more than one valid seal per block
3. Requires adding a new field to the core Block type in Sawtooth
4. Requires adding a new method to the consensus API to append data to an
   existing block, which requires making blocks mutable in the validator
5. Validators replaying the chain must either forego verification of the
   consensus seal, or poll a node for its seals

Another alternative considered was to perform consensus on the block contents
prior to signing and publishing the block.

1. Leader proposes "pre-block" containing (batch list, previous block, leader's
   public key) to followers
2. Followers, vote to commit the pre-block
3. After receiving 2f+1 votes from the followers to commit, leader constructs
   consensus seal described above, inserts into block consensus payload, signs
   block, and publishes block.
4. Followers receive block, are immediately able to verify 2f+1 votes to commit
   because they are in the seal, and commit the block.

At first this seems like an attractive alternative. However, in order for it to
work, substantial changes would need to be made to the validator architecture
to allow consensus engines to submit batches for validation that are not in a
block and to notify consensus engines of batches. Another extension would also
need to be added to recover from a network agreeing to commit a block, but that
block never arriving due to either 1. an invalid signature or 2. the leader
never actually publishing the block.

# Prior art
[prior-art]: #prior-art

See "Rationale and alternatives"

# Unresolved questions
[unresolved]: #unresolved-questions

No unresolved questions
