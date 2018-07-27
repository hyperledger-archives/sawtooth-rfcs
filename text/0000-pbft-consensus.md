- Feature Name: pbft-consensus
- Start Date: 2018-06-18
- RFC PR: (leave this empty)
- Sawtooth Issue: (leave this empty)

# Summary
[summary]: #summary
This RFC describes a practical Byzantine fault tolerant (PBFT) consensus
algorithm for Hyperledger Sawtooth. The algorithm uses the Consensus API
described in [a pending Sawtooth
RFC](https://github.com/aludvik/sawtooth-rfcs/blob/500b3688acfb0cd4834ea6451a8c5e000f7f5174/text/0000-consensus-api.md),
and is adapted for blockchain use from a 1999 paper by Miguel Castro and
Barbara Liskov [[1]](#references).


# Motivation
[motivation]: #motivation
The motivation for this RFC is to add a new, voting-based consensus mechanism
with Byzantine fault tolerance to the capabilities of Hyperledger Sawtooth.
This PBFT consensus algorithm ensures safety and liveness of a network,
provided at most `floor((n - 1)/3)` nodes are faulty, where `n` is the total
number of nodes in the network [[1]](#references). The PBFT algorithm is also
inherently crash fault tolerant. Another advantage of PBFT is that blocks
committed by nodes are final, so there are no forks in the network. This is
verified by a "Consensus Seal," which is appended to blocks when they are
finalized and checked upon receipt of a new block. In the future, this base
PBFT algorithm could be extended to other PBFT-style algorithms, which
decrease the number of necessary nodes in the network and reduce the total
number of inter-node communication steps required.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The Byzantine Generals Problem
Malicious users and code bugs can cause nodes in a network to exhibit
[arbitrary (Byzantine)
behavior](https://en.wikipedia.org/wiki/Byzantine_fault_tolerance#Byzantine_Generals'_Problem)
[[1]](#references).  Byzantine behavior can be described by nodes that send
conflicting information to other nodes in the network [[2]](#references).

**Example:**
Consider a scenario where there is a binary decision to be made: "to be" or
"not to be." Say that there are nine nodes in a network. Four nodes send a
"to be" message to all the other nodes, and four send "not to be" to all other
nodes. However, the last node is faulty and sends "to be" to half of the
nodes, and "not to be" to the other half. Because of this faulty node, the
network ends up in an existential crisis. The goal of a Byzantine fault
tolerant system is to resolve crises like this.

## Practical Byzantine Fault Tolerance
Algorithms that attempt to conquer the aforementioned Byzantine faults are
called Byzantine fault tolerant algorithms. Such algorithms include: Zyzzyva,
RBFT, MinBFT, and PBFT. This RFC focuses on the base PBFT algorithm.

This consensus algorithm is *voting*-based, meaning [[3]](#references):
+ Only a single node (the primary) can commit blocks to the chain at any given time
+ One or more nodes in the network maintains a global view of the network
+ Adding and removing nodes from the network is difficult
+ There are many peer-to-peer messages passed in between nodes which
  specifically relate to consensus (see [Message Types](#message-types))

PBFT can be thought of in terms of [state machine
replication](https://en.wikipedia.org/wiki/State_machine_replication). The
generic (non-blockchain-specific) algorithm works as follows:
1. A client sends a message (request) to all the nodes in the network
2. A series of messages is sent between nodes to determine if the request is
  valid, or has been tampered with.
3. Once a number of nodes agree that the request is valid, then the
  instructions (operations) in the request are executed, and a result (reply)
  is returned to the client.
4. The client waits for a number of replies that match, then accepts the
  result.

This generic algorithm has been modified and extended for blockchain use; see
[Normal Case Operation](#normal-case-operation) for a more detailed
step-by-step walkthrough of this.

## Terminology

| Term              | Definition                                   |
| ----------------: | :------------------------------------------- |
| Node              | Machine running all the components necessary for a working blockchain (including the Validator, the REST API, at least one transaction processor, and the PBFT algorithm itself). In this RFC, unless otherwise specified, it can be assumed that *Node* refers to the PBFT component of the machine. |
| Server            | Synonym for node. |
| Replica           | Synonym for node. |
| Validator         | Component of a node responsible for interactions with the blockchain. Interactions with the validator are abstracted by the Consensus API. |
| Block             | A part of the [blockchain](https://en.wikipedia.org/wiki/Blockchain), containing some operations and a link to the previous block. |
| Primary           | Node in charge of making the final consensus decisions and committing to the blockchain. Additionally is responsible for publishing the blocks given to it by the Consensus API, and starting the consensus process. |
| Secondary         | Auxiliary node used for consensus. |
| Client            | Machine that sends requests to and receives replies from the network of nodes. PBFT has no direct interaction with clients; the Validator bundles all client requests into blocks and sends them through the Consensus API to the consensus algorithm. |
| Checkpoint        | Point in time when logs can get garbage collected. |
| Checkpoint period | How many client requests in between each checkpoint. |
| Block duration    | How many seconds to wait in between the creation of each block. |
| Message           | Block, with additional information (see [Data Structures](#data-structures)). |
| Working block     | The block that has been initialized but not finalized, and is currently being committed to. |
| Low water mark    | The sequence number of the last stable checkpoint. |
| High water mark   | Low water mark plus the desired maximum size of nodes' message logs. |
| View              | The scope of PBFT when the current primary is in charge. The view changes when the primary is deemed faulty, as described in [View Changes](#view-changes). |
| `n`               | The total number of nodes in the network. |
| `f`               | The maximum number of faulty nodes. |
| `v`               | The current view number. |
| `p`               | The primary server number; `p = v mod n`. |


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation
The PBFT consensus algorithm is written in Rust, and implements the `Engine`
trait described in [the Consensus
API](https://github.com/aludvik/sawtooth-rfcs/blob/500b3688acfb0cd4834ea6451a8c5e000f7f5174/text/0000-consensus-api.md)
[[3]](#references).

## Data Structures

### Consensus Messages
[consensus-messages]: #consensus-messages
These are the messages that will be sent from node to node which specifically
relate to consensus. The content of all consensus-related messages are
serialized using [Protocol
Buffers](https://developers.google.com/protocol-buffers/) (Protobuf).

From the Consensus API [[3]](#references):

```
// A consensus-related message sent between peers
message ConsensusPeerMessage {
  // Interpretation is left to the consensus engine implementation
  string message_type = 1;

  // The opaque payload to send to other nodes
  bytes content = 2;

  // Used to identify the consensus engine that produced this message
  string name = 3;
  string version = 4;
}
```

Consensus messages sent by the PBFT algorithm will have one of the following
types (contained in the `message_type` field of `ConsensusPeerMessage`):

+ `PrePrepare`
+ `Prepare`
+ `Commit`
+ `Checkpoint`
+ `ViewChange`

### Message Types
[message-types]: #message-types
By definition, nodes in a PBFT network need to send a significant number of
messages to each other. Most messages have similar contents, shown by
`PbftMessage`. Auxiliary messages related to view changes are also shown.
Furthermore, PBFT uses some of the message types defined in the Consensus API
(referred to as updates), such as blockchain-related updates like `BlockNew`
and `BlockCommit`, and the system update `Shutdown`.

The following Protobuf-style definitions are used to represent all
consensus-related messages in the PBFT system:

```
// PBFT-specific block information (don't need to keep sending the whole payload
// around the network)
message PbftBlock {
  bytes block_id = 1;

  bytes signer_id = 2;

  uint64 block_num = 3;

  bytes summary = 4;
}

// Represents all common information used in a PBFT message
message PbftMessageInfo {
  // Type of the message
  string msg_type = 1;

  // View number
  uint64 view = 2;

  // Sequence number
  uint64 seq_num = 3;

  // Node who signed the message
  bytes signer_id = 4;
}
```

```
// A generic PBFT message (PrePrepare, Prepare, Commit, Checkpoint)
message PbftMessage {
  // Message information
  PbftMessageInfo info = 1;

  // The actual message
  PbftBlock block = 2;
}
```

```
// View change message, for when a node suspects the primary node is faulty
message PbftViewChange {
  // Message information
  PbftMessageInfo info = 1;

  // Set of `2f + 1` Checkpoint messages, proving correctness of stable
  // Checkpoint mentioned in info's `seq_num`
  repeated PbftMessage checkpoint_messages = 2;
}
```

## Algorithm Operation
The PBFT algorithm will operate within the framework described by the Consensus
API [[3]](#references). The `start` method contains an event loop which
handles all incoming messages, in the form of `Update`s. The most important
form of `Update` to the functionality of PBFT is `Update::PeerMessage`, but
other updates like `BlockNew`, `BlockCommit`, `BlockValid`, `BlockInvalid`,
and `Shutdown` are considered.

### Peer Messages
When a message arrives from a peer, it must be interrogated for its type, and
then the system must create a corresponding language-specific object of that
message type. This is made easier by the fact that all consensus messages are
Protobuf-serialized. Generally, once a message is converted into an
appropriate object, it needs to be checked for content to make sure everything
is legitimate. Some of these checks are performed by the validator (such as
checking that a message's signature is valid), but some (making sure messages
match, and that there are the correct number of them) need to be handled by
the PBFT consensus engine.

### On-chain Settings
The following on-chain settings are configurable, using the [settings
transaction
family](https://sawtooth.hyperledger.org/docs/core/releases/latest/transaction_family_specifications/settings_transaction_family.html),
with their respective defaults:

+ `sawtooth.consensus.pbft.peers` (required): JSON-formatted string of
  `{<public-key1>: <node-id1>, <public-key2>: <node-id2>}` mappings. For instance,
  `{"02f86c2e26cf89117b10db0cfeff3fa26139f0c03ee8d6e3a77a814fca8c6de215":0,
  "03b515f97b2597f82ff0d5004a066d6d4f8052c82540a0b52cafcf5a719c8b5552":1}`
  defines a 2-node network. The public keys refer to the validator's public
  key, usually located in `/etc/sawtooth/keys/validator.pub`

+ `sawtooth.consensus.pbft.block_duration=200`: How long to wait before trying
  to publish a block

+ `sawtooth.consensus.pbft.checkpoint_period=100`: How many `Commit` messages
  in between each checkpoint

+ `sawtooth.consensus.pbft.view_change_timeout=4000`: How long to wait before
  deeming a primary node faulty

+ `sawtooth.consensus.pbft.message_timeout=100`: How long to wait to
  receive a message. If no message is received in this time, the Consensus
  API returns with a `RecvTimeoutError::Timeout`

+ `sawtooth.consensus.pbft.max_log_size=1000`: The maximum number of messages
  that can be in the log


### Node Information Storage
Every node will keep track of the following state information:
+ Its own id

+ Its current sequence number and view number

+ Whether it's a primary or secondary node

+ Which step of the algorithm it's on

+ Mode of operation (`Normal`, `ViewChanging`, `Checkpointing`)

+ The maximum number of faulty nodes allowed in the network

+ The block that it's currently working on

+ Log of every peer message that has been sent to it (used to determine if it
  has received enough matching messages to proceed to the next stage of the
  algorithm; can be [garbage collected](#garbage-collection) every so often).

+ List of its connected peers. This is provided at startup from on-chain
  settings specified by the user. The length of this peer list will be used to
  calculate `f`, the maximum number of faulty nodes this network can tolerate.
  Currently, only static networks will be supported (that is, there will be no
  adding or removal of peers).


### Predicates
In order to keep the algorithm explanation below concise, we'll define some
predicates here.

+ `prepared` is true for the current node if the following messages are
  present in its log:
  + The original `BlockNew` message
  + A `PrePrepare` message matching the original message (in the current view)
  + `2f + 1` matching `Prepare` messages from different nodes that match
    `PrePrepare` message above (including its own)

+ `committed` is true if for the current node:
  + `prepared` is true
  + This node has accepted `2f + 1` `Commit` messages, including its own

### Normal Case Operation
[normal-case-operation]: #normal-case-operation
#### Explanation of Message Types
+ `PrePrepare`: Sent from primary node to all nodes in the network, notifying
them that a new message (`BlockNew`) has been received from the validator.

+ `Prepare`: Broadcast from every node once a `PrePrepare` is received for the
  current working block; used as verification of the `PrePrepare` message, and
  to signify that the block is ready to be checked.

+ `Commit`: Broadcast from every node once a `BlockValid` update is received
  for the current working block; used to determine if there is consensus that
  nodes should indeed commit the block contained in the original message.

+ `Checkpoint`: Sent by any node that has completed `checkpoint_period`
  `Commit` messages.

+ `ViewChange`: Sent by any node that suspects that the primary node is faulty.

#### States
**States:** PBFT follows a state-machine replication pattern, where these
states are defined:

+ `NotStarted`: The algorithm has not been started yet. No `BlockNew` updates
  have been received. In this stage, `Checkpointing` will occur if
  `checkpoint_period` blocks have been committed to the chain. Ready to
  receive a `BlockNew` update for the next block.

+ `PrePreparing`: A `BlockNew` has been received through the Consensus API,
  and its Consensus Seal has been verified. Ready to receive a `PrePrepare`
  message for the block corresponding to the `BlockNew` message just received.

+ `Preparing`: A `PrePrepare` message has been received and is valid. Ready to
  receive `Prepare` messages corresponding to this `PrePrepare`.

+ `Checking`: The predicate `prepared` is true; meaning this node has a
  `BlockNew`, a `PrePrepare`, and `2f + 1` corresponding `Prepare` messages.
  Ready to receive a `BlockValid` update.

+ `Committing`: A `BlockValid` has been received. Ready to receive `Commit`
  messages.

+ `Finished`: The predicate `committed` is true and the block has been
  committed to the chain. Ready to receive a `BlockCommit` update.

These states may be interrupted at any time if the view change timer expires,
forcing the node into `ViewChanging` mode.


**State Transitions:** The following state transitions are defined; listed
with their causes:

+ `NotStarted` &rarr; `PrePreparing`: Receive a `BlockNew` update for the next
  block.

+ `PrePreparing` &rarr; `Preparing`: Receive a `PrePrepare` message
  corresponding to the `BlockNew`.

+ `Preparing` &rarr; `Checking`: `prepared` predicate is true.

+ `Checking` &rarr; `Committing`: Receive a `BlockValid` update corresponding
  to the current working block.

+ `Committing` &rarr; `Finished`: `committed` predicate is true.

+ `Finished` &rarr; `NotStarted`: Receive a `BlockCommit` update for the
  current working block.


The states, state transitions, and actions that the algorithm takes are
represented in the following diagram:

![PBFT states](../images/pbft_states.png)


#### Initialization
At the beginning of the Engine's `start` method, some initial setup is
required:
+ Create the node for processing messages
+ Establish timers and counters for checkpoint periods and block durations,
  which are loaded from the on-chain settings

#### Event Loop
In `Normal` mode (when the primary node is not faulty), the PBFT consensus
algorithm operates as follows, inside the event loop of the `start` method:

1. Receive a `BlockNew` message from the Consensus API, representative of
   several batched client requests. The primary node checks the legitimacy of
   the message and assigns this message a sequence number, then broadcasts a
   `PrePrepare` message to all nodes. Legitimacy is checked by looking at the
   `signer_id` of the block in the `BlockNew` message, and making sure the
   `previous_id` is valid as the current chain head. The Consensus Seal is
   checked here as well, and all nodes tentatively update their working
   blocks. Secondary nodes ignore `BlockNew` messages; only append them to
   their logs.

2. Receive `PrePrepare` messages and check their legitimacy. `PrePrepare`
   messages are legitimate if:
  + `signer_id` and `summary` of block inside `PrePrepare` match the
    corresponding fields of the original `BlockNew` block &&
  + View in `PrePrepare` message corresponds to this server's current view &&
  + This message hasn't been accepted already with a different `summary` &&
  + Sequence number is within the sequential bounds of the log (low and high
    water marks)

    Once the `PrePrepare` is accepted:
      + If primary: double check message matches the `BlockNew`, then broadcast a `Prepare` message to all nodes.
      + If secondary: update its own sequence number from the message, then broadcast a `Prepare` message to all nodes.

    If the `PrePrepare` is determined to be invalid, then start a view change.

3. Receive `Prepare` messages, and check them all against their associated
   `PrePrepare` message in this node's message log.

4. Once the predicate `prepared` is true for this node, then call
   `check_blocks()` on the current working block. If an error occurs
   (`ReceiveError` or `UnknownBlock`), abort (call `ignore_block()`).
   Otherwise, wait for a response (`BlockValid` or `BlockInvalid`) from the
   validator. If `BlockValid`, then broadcast a `Commit` message to all other
   nodes. If `BlockInvalid`, then call `ignore_block()`, and start a view
   change.

5. When the predicate `committed` is true for this node, then it should commit
   the block using `commit_block()`, and advance the chain head.

6. When a `BlockCommit` update is received by the primary node, it will call
   `initialize_block()`.

7. If *block duration* has elapsed, then try to `summarize_block()` with the
   current working block. If the working block is not ready, (`BlockNotReady`
   or `InvalidState` occurs), then nothing happens. Otherwise,
   `finalize_block()` is called with a serialized summary of all the `Commit`
   messages this node has received (which functions as the Consensus Seal).
   This in turn sends out a `BlockNew` update to the network, starting the
   next cycle of the algorithm.

### View Changes
[view-changes]: #view-changes
Sometimes, the node currently in charge (the primary) becomes faulty. This
could mean it is either malicious, or experiencing internal problems. In
either of these cases, a view change is necessary. View changes are triggered
by a timeout: When a secondary node receives a `BlockNew` message, a timer is
started. If the secondary ends up receiving a `Commit` message, the timer is
cancelled, and the algorithm proceeds as normal. If the timer expires, the
primary node is considered faulty and a view change is initiated. This ensures
Byzantine fault tolerance due to the fact that each step of the algorithm will
not proceed to the next unless it receives a certain number of matching
messages, and due to the fact that the Validator does not pass on any messages
that have invalid signatures [[3]](#references).

The view change process is as follows:
1. Any node who discovers the primary as faulty (whose timer timed out) sends
   a `ViewChange` message to all nodes, containing the node's current
   sequence number, its current view, proof of the previous checkpoint, and
   pending messages that have happened since that previous checkpoint. The
   node enters `ViewChanging` mode.

2. Once a server receives `2f + 1` `ViewChange` messages (including its own),
   it changes its own view to `v + 1`, and resumes `Normal` operation. The new
   primary node's ID is `p = v mod n`.

### Garbage Collection
[garbage-collection]: #garbage-collection
After each *checkpoint period* (around 100 successfully completed cycles of
the algorithm), server log messages can possibly be garbage-collected. When
each node reaches a checkpoint, it enters `Checkpointing` mode and sends out a
`Checkpoint` message to all of the other servers, with that node's current
state (described by a `PbftBlock`). When the current node has `2f + 1`
matching `Checkpoint` messages from different servers, the checkpoint is
considered *stable* and the logs can be garbage collected: All log entries
with sequence number less than the one in the `Checkpoint` message are
discarded, and all previous checkpoints are removed. The high and low water
marks are updated to reflect the sequence number of the new stable checkpoint.
Once garbage collection is complete, the node resumes `Normal` operation.


# Drawbacks
[drawbacks]: #drawbacks
One possible downside of PBFT-style consensus algorithms is that in general,
`3f + 1` nodes are needed on the network in order to preserve Byzantine fault
tolerance, unless special considerations are taken to ensure the authenticity
of the peer-to-peer messages and sequence counters [[6]](#references). PBFT
also requires a large number of consensus-specific messages; the general PBFT
algorithm requires five per published block, and this implementation requires
three (because it does not directly interact with clients). Additionally, PBFT
does not prevent nodes from "leaking" information to bad actors.

A drawback of this specific design of PBFT is that the consensus algorithm
operates serially; there is only one working block. Likely in a production
environment, this will be unacceptable. A queue of working blocks will be
necessary, each keeping track of what stage of the algorithm it is at.

Perhaps the most significant drawback of this algorithm stems from a lack of
information provided by the Consensus API. In order to be truly Byzantine
fault tolerant, PBFT must have access to the ordering of batches inside of the
block, not just the blocks themselves. Currently, the Consensus API does not
provide any visibility about batches inside the blocks, so it can't be
confirmed that all blocks indeed have the same batch list ordering. One
possible solution to this would be to package the batch list inside the block.
It is possible that the whole batch list is not necessary, perhaps a hash of
it would be sufficient to ensure ordering of batches inside blocks provided by
the Consensus API.


# Rationale and alternatives
[alternatives]: #alternatives
There are many other voting-type consensus algorithms available to anyone who
wants to use them, such as those mentioned in [Prior art](#prior-art). Many of
these algorithms are based on the methods described in this RFC, and most
strategies employed by this RFC apply to PBFT's derivations as well.


# Prior art
[prior-art]: #prior-art
#### Papers
+ The original PBFT paper [[1]](#references)
+ Improvements on PBFT [[4]](#references)
+ Speculative Byzantine Fault Tolerance [[5]](#references)
+ MinBFT and MinZyzzyva [[6]](#references)

#### Other consensus implementations
+ A PBFT consensus algorithm is [under
  development](https://github.com/hyperledger/fabric/tree/release-1.1/orderer)
  at [Hyperledger Fabric](https://www.hyperledger.org/projects/fabric)

+ There is an open Ethereum Improvement Proposal for Istanbul Byzantine fault
  tolerance in [Ethereum](https://github.com/ethereum/EIPs/issues/650)

+ There is an implementation of Istanbul Byzantine fault tolerance for [J. P.
  Morgan
  Quorum](https://github.com/jpmorganchase/quorum/tree/master/consensus/istanbul)

+ Raft is [being developed](https://github.com/hyperledger/sawtooth-raft) for
  Hyperledger Sawtooth

+ There's an implementation of MinBFT [being
  developed](https://github.com/nec-blockchain/minbft) for NEC blockchain


# Unresolved questions
[unresolved]: #unresolved-questions
+ What should the block consensus payload be comprised of (when calling
  `finalize_block()`)?
  + This is just the Consensus Seal, containing `2f + 1` `Commit` messages and
    proving that the block went through consensus
+ What is the best way to persistently store message logs? Do they need to be
  secure/encrypted?


# References
[references]: #references
[[1] M. Castro and B. Liskov. Practical Byzantine Fault Tolerance, _Operating
Systems Design and Implementation_,
1999.](https://www.usenix.org/legacy/events/osdi99/full_papers/castro/castro_html/castro.html)

[[2] L. Lamport, R. Shostak, and M. Pease. The Byzantine Generals Problem. _ACM
Transactions on Programming Languages and Systems_, 4(3),
1982.](https://inst.eecs.berkeley.edu/~cs162/sp16/static/readings/Original_Byzantine.pdf)

[[3] A. Ludvik. Consensus API. Sawtooth RFCs, March
2018.](https://github.com/aludvik/sawtooth-rfcs/blob/500b3688acfb0cd4834ea6451a8c5e000f7f5174/text/0000-consensus-api.md)

[[4] M. Castro and B. Liskov. Practical Byzantine Fault Tolerance and Proactive
Recovery, _ACM Transactions on Computer Systems_, 20(4),
2002.](http://zoo.cs.yale.edu/classes/cs426/2012/bib/castro02practical.pdf)

[[5] R. Kotla, L. Alvisi et al. Zyzzyva: Speculative Byzantine Fault Tolerance.
_ACM SIGOPS Operating Systems Review_, 41(6),
2007.](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.122.112&rep=rep1&type=pdf)

[[6] G. Veronese et al. Efficient Byzantine Fault Tolerance. _IEEE Transactions
on Computers_, 62(1),
2013.](http://homepages.gsd.inesc-id.pt/~mpc/pubs/Veronese-Efficient%20Byzantine%20Fault%20Tolerance.pdf)
