- Feature Name: PoET-2.0
- Start Date: 2018-05-21
- RFC PR: (leave this empty)
- Sawtooth Issue: (leave this empty)

# Summary
[summary]: #summary

As blockchain technology matures, there is a pressing need for more efficient 
consensus mechanisms that will scale to large networks without putting 
potentially crippling demands on the infrastructure required to support it. 
Proof of Elapsed Time (PoET) is an attempt at providing an innovative algorithm 
that not only scales very efficiently, but also places very few demands on the 
infrastructure.

This document details a new mechanism for the PoET algorithm that overcomes 
some of the challenges with the original algorithm.

The intended audience of this document is architects, developers and anyone who 
would like to get a better understanding of the details of the PoET 2.0 
algorithm.

This RFC describes a new PoET 2.0 Consensus for supporting the Intel® Software 
Guard Extensions(SGX) machines without Platform Services Enclave(PSE).

# Motivation
[motivation]: #motivation

PoET 1.0 offers a very unique and efficient way to achieve consensus based on 
secure timers set inside an Intel® Software Guard Extensions (SGX) enclave. The 
secure timer and the monotonic counters used in PoET 1.0 rely on 
_SGX Platform Services_ to access these services from the system hardware. 
_SGX Platform Services_, unfortunately, are not yet available on all SGX 
enabled machines. While plans are in place for introducing these services 
universally across the processor lineup, it will take some time to release the 
services and for them to become ubiquitous. PoET 2.0 addresses these challenges 
by making the PoET algorithm independent of the _SGX Platform Services_.

# Definitions
The following terms are used throughout the PoET spec and are defined here for 
reference.

*Enclave*

A protected area in an application’s address space which provides 
confidentiality and integrity even in the presence of privileged malware.

The term can also be used to refer to a specific enclave that has been 
initialized with a specific code and data.

*EPID*

An anonymous credential system. 

*EPID Pseudonym*

Pseudonym of an SGX platform used in linkable quotes. 
It is part of the IAS attestation response according to IAS API 
specifications. It is computed as a function of the service Basename 
(validator network in our case) and the device’s EPID private key.

*PPK, PSK*

PoET ECC public and private key created by the PoET enclave.

*IAS Report Key*

IAS public key used to sign attestation reports as specified in the current 
IAS API Guide.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In PoET 1.0, the PoET SGX enclave set a timer internally and would only release a WaitCertificate, once the timer elapsed. To be considered a candidate for consensus, a block (a.k.a Claim Block) would have to be accompanied by the corresponding WaitCertificate. This meant that peers receiving a Claim Block accompanied by the WaitCertificate, were assured that the block had not been forwarded prematurely, hence the phrase ‘Proof of Elapsed Time’.
PoET 2.0, however, removes the reliance on internal timers and monotonic counters within SGX. This necessitates a different approach for enforcing wait times. Rather than set a timer inside SGX, PoET 2.0 relies on SGX to provide it with a duration to wait and sets the timer OUTSIDE SGX. A Claim Block is forwarded only after the timer has expired. The block is now accompanied by a WaitCertificate from SGX containing the duration that the block was expected to wait for.
While this opens up the possibility of misuse by a malicious actor, the community of validators is engaged to enforce the wait on the block. Blocks arriving at peers earlier than suggested by their WaitCertificate are held back until the time is right to forward them. The larger the size of the network, the more resistant the protocol is to attack by a malicious actor or a cabal.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Here we describe the PoET 2.0 algorithm in greater detail.

## Initialization
When the PoET 2.0 module is initialized, it performs the following actions:
1. Generate an ECDSA key pair. This pair is held in SGX memory and NEVER committed to disk for reuse subsequently
2. Create a table in SGX memory to map the Block Number to WaitCertificates. Also store the block number of the chain head in SGX
3. Generate a Quote containing the PPK and perform Quote verification with the IAS
4. Check if there was a previous PPK registration. If there was, verify if ‘K’ blocks have been added to the chain since the key’s registration. If K blocks have not passed, wait until K blocks have been added. If the process restarts, signup at process startup and resume the K-test since this new signup
5. Register the PPK and the IAS response with the ledger
6. Wait for ‘C’ blocks to be added to the chain before participating in consensus(C-test)

## Wall Clock and Chain Clock
The PoET 2.0 algorithm maintains two clocks, the _Wall Clock_ and the _Chain Clock_.
The Wall Clock (WC) measures the time elapsed since the arrival of the ‘Sync Block’, i.e. the first block on the chain.
The Chain Clock (CC) measures the cumulative wait duration of all the blocks in the chain (each fork will have its own CC).
Upon receiving the Sync Block, WC and CC are initialized to 0. In practice, the WC may be calculated by recording the system time at the moment of the arrival of the Sync Block and subsequently subtracting this timestamp from the current time.
To handle clock drifts, it may become necessary to synchronize with a standard network clock. Details are out of scope for this version of the document.

## Block Creation
Once the algorithm is in a state to publish blocks (i.e. the C-test passes), it starts the process of creating Claim Blocks with the aim of participating in the Consensus mechanism.
The Claim/Candidate Block is assembled as usual with no special processing required for PoET 2.0

## Wait Certificate Creation
Once a Claim Block is created, the PoET algorithm requests the PoET enclave to create the WaitCertificate.

The contents of the WaitCertificate for PoET 2.0 are:

* Duration: A random 256 bit number generated by SGX
* WaitTime: The number of seconds to wait, determined by the LocalMean
* BlockID: The BlockID passed in to the Enclave
* PrevBlockID: The BlockID of the previous block, as stored in the previousWaitCertificate
* TxnHash: The hash of the transactions in the block
* BlockNumber: The length of the chain
* ValidatorID: The ID of the current Validator
* Sign: The signature of the WaitCertificate computed over all the fields using the PSK 

Once the WaitCertificate is created, the enclave stores the Last Block # and the WaitCertificate in the table created previously. The validator will only create a single wait certificate per block. This implies that if a fork containing fewer blocks becomes active, the validator will NOT be able to publish any more blocks until the new fork has added enough blocks to catch up to the previous chain.

## Block Publishing
With the Claim Block ready and the WaitCertificate created, the validator will determine the duration for which to set the timer. The duration of the wait is determined by the ‘Duration’ field in the WaitCertificate.
Ideally, any selected algorithm should be capable of converting the duration to a wait time distributed uniformly between 0 and LocalMean seconds.

## Block Verification
Upon receiving a Claim Block and a WaitCertificate, the receiving validator does the following:
### Sanity checks
1.	Ensure the Block and WaitCertificate signatures are valid
2.	The Block’s ID matches the WaitCertificate’s blockID
3.	If the block extends the current chain head, proceed to the Block Validity check. If it is a ‘future’ block, cache it for processing later.
4.	If the block claims a previous block as the parent, proceed to ‘Fork Resolution’
### Validity check & block publishing
5.	Compute the new CC. The block is valid if CC + E < WC. Here E is a network latency coefficient added to simulate the average network delay seen by the validator.
6.	Send the block to the validator for processing the transactions
7.	If valid, commit the block, store the blockID and WaitCertificate in SGX
8.	Forward the block to the gossip network
9.	Drop the existing block proposal and restart from the new chain head.
### Early Arriving blocks
10.	If the block is an early-arriving block, compare the duration with the duration of the Claim Block. If the block duration is smaller, drop the Claim Block. Wait for the WC to catch up with the CC. If the Claim Block duration is less than the incoming block duration, discard the incoming block.
11.	Proceed to step 6 once the block becomes valid.
### Fork Resolution
12.	If the new block claims an earlier block as a parent, check for any cached blocks claiming the new block as a parent, retrieve them. Compute the new CC and check if the chain is valid (CC + E < WC)
13.	If valid, compare existing chain length with the new chain length. If existing chain length is more, discard new chain. If the new chain is longer, switch to the new chain (commit the new blocks)
14.	If the two chains are equal, compare Chain Clocks. Select the chain with the smaller CC.
15.	If the chain is switched, drop the block proposal & restart (Step 9).
16.	Forward the new block over the gossip network.

# Drawbacks
[drawbacks]: #drawbacks

# Rationale and alternatives
[alternatives]: #alternatives

Poet 2.0 improves on PoET 1.0 adding support for SGX devices without PSE. Current PoET 1.0 limits the availability of PoET consensus only to SGX devices with PSE.

# Prior art
[prior-art]: #prior-art

PoET 2.0 is based on mostly PoET 1.0 and newer solutions for overcoming challenges thereof. 

# Unresolved questions
[unresolved]: #unresolved-questions

