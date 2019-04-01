- Feature Name: pbft_consensus_seal
- Start Date: 2018-10-11
- RFC PR: https://github.com/hyperledger/sawtooth-rfcs/pull/30
- Sawtooth Issue: N/A

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

The consensus seal for PBFT consists primarily of 2f or more signed COMMIT
messages from other nodes participating in consensus. The minimum number of
messages in the seal is actually 2f rather than 2f + 1 because the seal itself
counts as the COMMIT vote from the node that creates the seal. This is done out
of necessity: a vote that is created by the PBFT consensus engine and added to
its own log is not actually signed, since signing is done by the validator and
the consensus engine does not have access to the private key for signing the
message.

In order to guarantee that the seal cannot be reused for another block, we
require that it contain the ID of the block it verifies. Additionally, the seal
contains an information message that identifies the creator of the seal, as well
as the view and sequence number that the included votes must match.

We define the consensus seal for PBFT using the following protobuf messages:

    message PbftSignedVote {
      // Serialized ConsensusPeerMessage header
      bytes header_bytes = 1;

      // Signature of the serialized ConsensusPeerMessageHeader
      bytes header_signature = 2;

      // Serialized PBFT message
      bytes message_bytes = 3;
    }

    message PbftSeal {
      // Message information
      PbftMessageInfo info = 1;

      // ID of the block this seal verifies
      bytes block_id = 2;

      // A list of Commit votes to prove the block commit (must contain at least
      // 2f votes)
      repeated PbftSignedVote commit_votes = 3;
    }

The `PbftSeal` protobuf will be serialized and included in a block's consensus
payload field.

Before a new block can be finalized by the leader, a `PbftSeal` must be
constructed with 2f `PbftSignedVote` messages for the previous block. Prior to
followers committing a block, they must verify the `PbftSeal`. To verify the
seal, the node must:

1. Parse the `PbftSeal` from the bytes in the block's `payload` field
2. Check that the `block_id` field in the seal matches the `previous_id` field
   in the block that the node is trying to commit
3. For each vote, verify that:
   1. It is a `Commit` message
   2. Its `block_id`, `seq_num`, and `view` fields match those of the `PbftSeal`
   3. Its `header_signature` is a valid signature over `header` using the
      private key associated with the public key in the header's `signer_id`
      field.
   4. The header's `content_sha512` is a valid SHA-512 hash of the vote's
      `message_bytes`.
   5. The signer was a member of the network at the time the block was voted on
      (the block was voted on before it was committed, so the list of peers
      should be taken from the block before the one the seal verifies)
   6. The signer is not the same as the consensus seal's signer (this would be a
      double-vote)
4. Check that the consensus seal's signer was a member of the network at the
   time the block was voted on (the block was voted on before it was committed,
   so the signer's ID should be in the list of peers that is taken from the
   block before the one the seal verifies)
5. Check that all votes are from unique peers
6. Check that there are a total of 2f votes

# Drawbacks
[drawbacks]: #drawbacks

There are two drawbacks to this change. First, additional signature and digest
verification for 2f messages at every block creates additional work that
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
