- Feature Name: consensus-api
- Start Date: 2018-03-15
- RFC PR: (leave this empty)
- Sawtooth Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC describes a new Consensus API for supporting both the existing
_lottery_ consensus algorithms as well as _voting_ algorithms
such as rBFT.

# Motivation
[motivation]: #motivation

The motivation for this RFC is to support integrating new non-lottery-based
consensus mechanisms with Sawtooth, such as rBFT, while continuing to support
PoET/SGX, PoET/Simulator, and devmode consensus modules.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Consensus API was developed after researching a variety of consensus
algorithms, consensus interfaces, and the consensus problem in general. Out of
this research came the following observations:

1. All consensus algorithms describe a state machine.
2. All consensus algorithm state transitions occur as a result of:

    a. Receiving a consensus message from a peer
    b. Receiving a message from a client
    c. An internal interrupt

These observations led to the creation of the _Consensus Engine_ abstraction,
which can be used to implement any consensus algorithm that is either an
_voting_ algorithm or a _lottery_ algorithm.

## Comparison of Algorithm Types

The following describes the differences between what we refer to as _voting_
and _lottery_ algorithms for the purpose of understanding how both are handled
by the consensus interface.

In _voting_ consensus algorithms:

- A single node is authorized to make commits at any given time.
- One or more nodes in the network is required to maintain a "global view" of
  the network.
- Adding and removing nodes from the network is non-trivial and requires
  coordination and network-wide agreement.
- Many "consensus-specific" messages are passed between nodes on the network to
  coordinate consensus.

Examples of _voting_ algorithms include PBFT, rBFT, Tendermint, and
Raft.

In _lottery_ consensus algorithms:

- All nodes are authorized to make commits at any given time.
- Nodes need only be aware of their peers on the network (not all nodes).
- Adding and removing nodes is trivial; the network supports "open-enrollment".
- There are no "consensus-specific" messages; consensus is an emergent property
  of the fork-resolution logic.

Examples of _lottery_ consensus algorithms include Proof-of-Work (PoW)
and Proof-of-Elapsed-Time (PoET).

## Overview of the Consensus Engine

The Consensus Engine lives in a separate process and interacts with the
Validator using protobuf messages.

The Consensus Engine is responsible for:

- Determining messages to send to peers
- Sending commands to progress the blockchain
- Reacting to internal interrupts and a limited set of external messages

The Consensus Engine is not responsible for:

- Validating the integrity of blocks, batches, transactions
- Validating block, batch, transaction, or message signatures
- Gossiping blocks, batches, or transactions
- Block storage, block creation, and (direct) management of the chain head

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Requirements

The following is a list of necessary requirements for the Sawtooth
architecture to be able to support both classes of consensus algorithms:

**R1 - Consensus-Specific Messaging**

At a minimum, an efficient communication channel for sending consensus messages
between network nodes is needed. This channel must support both broadcast and
peer-to-peer messaging.

**R2 - Separate Consensus Process**

Implementing consensus algorithms is expensive. It must be possible to reuse
existing implementations to the extent possible. For this reason, it must be
possible to run consensus in a separate process.

**R3 - Consensus-Driven Block Publishing and Chain Updates**

In order to be efficient, the consensus algorithm must be responsible for
determining when to publish blocks and when to update the chain head.

For example, in voting algorithms, only one node is authorized to
publish blocks at any given time and the authorized node often does not change
until it is deemed faulty. In a stable environment, this could be an indefinite
amount of time. If consensus is not responsible for determining when to publish
and update the chain head, all but one nodes will be polling consensus
for long periods of time.

**R4 - Read Access to Settings**

Consensus engines need to read settings to support on-chain consensus engine
configuration using the Settings Transaction Family. PoET depends on Settings
to configure critical values such as target wait time and enclave measurements
and these values must be consistent across the network.

**R5 - Read/Write Access to Global State**

PoET depends on tracking authorized block publishers in global state and
therefore requires read access to a PoET specific namespace.

**R6 - Reusability**

The Consensus Engine interface and implementations of the interface should be
reusable in other applications. Specifically, the interface should support
non-blockchain applications.

## Data Structures

**Blocks**

In order to support R6, "block" is defined to mean the following data structure
within the context of the consensus engine. When used in a blockchain context,
this block definition is equivalent to those parts of the block that are
relevant to consensus. When used outside the blockchain context, the
interpretation of this data structure is context dependent.

    Block {
        Id: string
        Previous Id: string
        Index: unsigned integer
        Consensus: bytes
    }

**Consensus Messages**

In order to support arbitrary communication between nodes, a generic message
type is defined with a payload that is opaque to the validator. The following
data structure is used for this:

    ConsensusMessage {
        MessageType: string
        Payload: bytes
    }

Payload is the opaque payload to send to other nodes. The interpretation of
MessageType is left to the Consensus Engine implementation.

## API

The Consensus Engine API below is presented as a set of methods in order to
simplify their presentation and explanation. Each method is implemented with a
pair of (Request, Response) messages and the Consensus Proxy is responsible for
handling this transformation within the Validator.

The methods presented below are only meant to describe roughly what the
high-level interactions between the Consensus Engine and the Validator will be.
The names, parameters, and exact behavior of these methods are subject to
change based on additional constraints and requirements discovered during
implementation.

The methods defined by the Consensus Engine interface are split into the
following groups:

1. P2P Messaging
2. Event Handlers
3. Block Creation
4. Block Management
5. Queries
6. Engine

### P2P Messaging Methods
The following methods are provided to Consensus Engines for sending consensus
messages to other nodes on the network. These methods support R1.

| Method | Description |
| --- | --- |
| SendTo(peer, message) | Send a consensus message to a specific, connected peer |
| Broadcast(message) | Broadcast a message to all peers |

### Event Handlers
The following methods will be implemented by Consensus Engines. They are used
to update the Consensus Engine on external events. These methods support R1-R3.

| Method | Description |
| --- | --- |
| OnMessageReceived(message) | Called when a new consensus message is received |
| OnNewBlockReceived(block) | Called when a new block is received and validated |
| OnAddPeer(peer) | Called when a new peer is added |
| OnDropPeer(peer) | Called when a peer is dropped |

### Block Creation Methods
The following methods will be provided to Consensus Engines for controlling
block creation. These methods support R3.

| Method | Description |
| --- | --- |
| InitializeBlock() |  Initialize a new block based on the current chain head and start adding batches to it. |
| FinalizeBlock(data) -> block_id | Stop adding batches to the current block and finalize it. Include the given consensus data in the block. If this call is successful, OnNewBlockReceived() will be called with the block. |
| CancelBlock() |  Stop adding batches to the current block and abandon it. |

### Block Management Methods
The following methods will be provided to Consensus Engines for controlling
chain updates. These methods support R3.

| Method | Description |
| --- | --- |
| CommitBlock(block_id) | Set the chain head to the given block id. The Chain Controller would handle the details here. |
| DropBlock(block_id) | Remove the given block from the system. The main purpose of this method is to allow the Consensus Engine to notify the rest of the system that a block is invalid from the perspective of consensus. |

### Query Methods
The following methods will be provided to Consensus Engines for getting
information from the validator.

| Method | Description |
| --- | --- |
| GetSetting(setting) -> data | Read the current value of the setting. Supports R4. |
| GetState(address) -> data | This is needed to read values from arbitrary addresses used by consensus (eg., validator registry). Supports R5. |
| GetBlock(block_id) -> block | Retrieve consensus-related information about a block. |

While the consensus engine should maintain a cache of blocks internally, this method is provided to:
1. Allow cache entries to expire without losing the block forever
2. Allow the engine to handle getting a block before its parent

### Engine Methods
The following methods will be implemented by Consensus Engines. They are used
to initialize and shutdown the engine, when a new validator starts up, or when
the consensus engine changes.

| Method | Description |
| --- | --- |
| Startup() | Startup and synchronize with any peers if necessary. |
| Shutdown() | Shutdown and notify peers if necessary. |

## Consensus Engine Architecture Integration
The following describes the interactions between the consensus engine and the
other major architectural components of the validator.

![consensus engine architecture](../images/consensus-engine-architecture.svg "Consensus Engine Architecture")

**Consensus Engine**

The Consensus Engine is a separate process that implements a consensus
algorithm and communicates with the Consensus Proxy through the Dispatcher. It
can also communicate with peers using consensus messages. Its role is to direct
block creation and chain management. It must implement the interface described
above.

**Consensus Proxy**

The Consensus Proxy mediates interactions between the Consensus Engine and the
rest of the Validator. This includes marshaling and unmarshaling messages from
the Dispatcher, passing block notifications to the Consensus Engine, and
passing commands from the Consensus Engine to the Chain Controller and Block
Publisher.

**Block Validator**

The Block Validator validates blocks except from the perspective of consensus.

**Chain Controller**

The Chain Controller manages both committed and uncommitted chain heads. The
Chain Controller is driven by the Consensus Engine and does not know about
block validity or consensus.

**Block Publisher**

The Block Publisher handles the creation of blocks. It is directed by the
Consensus Engine on when to do this. When a block is finalized, it is forwarded
to the Block Validator.

# Drawbacks
[drawbacks]: #drawbacks

1. Requires additional work to maintain existing consensus modules.
2. Requires work to change existing architecture.
3. Moving consensus to a separate process and requiring additional
   (de)serialization may degrade performance.

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

This design borrows some ideas from the [Tendermint ABCI][tendermint] and the
[Parity Consensus Engine][parity] interface.

[tendermint]: https://tendermint.readthedocs.io/en/master/app-development.html#abci-design)
[parity]: https://github.com/paritytech/parity/blob/e95b09348386d01b71901365785c5fa3aa2f7a6d/ethcore/src/engines/mod.rs#L176

# Unresolved questions

[unresolved]: #unresolved-questions

- Should the consensus engine be able to read arbitrary locations in state?
