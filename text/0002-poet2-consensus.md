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

This RFC describes an extension to the PoET algorithm for supporting the Intel® 
Software Guard Extensions(SGX) machines without Platform Services Enclave(PSE).

The intended audience of this document is architects, developers and anyone who 
would like to get a better understanding of the details of the PoET 2.0 
algorithm.

# Motivation
[motivation]: #motivation

PoET 1.0 offers a very unique and efficient way to achieve consensus based on 
secure timers set inside an IntelÃ‚Â® Software Guard Extensions (SGX) enclave. The 
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

| Term             | Definition |
| ---------------- | ------------------------------------------- |
|Enclave           | A protected area in an application's address space which provides confidentiality and integrity even in the presence of privileged malware. The term can also be used to refer to a specific enclave that has been initialized with a specific code and data.|
|Basename          | A service provider base name. In our context the service provider entity is the distributed ledger network. Each distinct network should have its own Basename and Service Provider ID (see EPID and IAS specifications).|
|EPID              | An anonymous credential system. See E. Brickell and Jiangtao Li: "Enhanced Privacy ID from Bilinear Pairing for Hardware Authentication and Attestation". IEEE International Conference on Social Computing / IEEE International Conference on Privacy, Security, Risk and Trust. 2010.|
|EPID Pseudonym    | Pseudonym of an SGX platform used in linkable quotes.  It is part of the IAS attestation response according to IAS API specifications. It is computed as a function of the service Basename (validator network in our case) and the device's EPID private key.|
|PPK, PSK          | PoET ECC public and private key created by the PoET enclave.|
|IAS Report Key    | IAS public key used to sign attestation reports as specified in the current IAS API Guide.|
|AEP               | Attestation evidence payload sent to IAS (see IAS API specifications). Contains JSON encodings of the quote and an optional nonce.|
|AVR               | Attestation Verification Report, the response to a quote attestation request from the IAS. It is verified with the IAS Report Key. It contains a copy of the input AEP.|
|`WaitCertId_{n}`  | The `n`-th or most recent WaitCertificate digest. We assume `n >= 0` represents the current number of blocks in the ledger. WaitCertId is a function of the contents of the Wait Certificate. For instance the SHA256 digest of the WaitCertificate ECDSA signature.|
|OPK, OSK          | Originator ECDSA public and private key. These are the higher level ECDSA keys a validator uses to sign messages.|
|OPKhash           | SHA256 digest of OPK|
|blockDigest       | ECDSA signature with OSK of SHA256 digest of transaction block that the validator wants to commit.|
|localMean         | Estimated wait time local mean.|
|PoET\_MRENCLAVE   | Public MRENCLAVE (see SGX SDK documentation) value of valid PoET SGX enclave.|
|`K`               | Number of blocks a validator can commit before having to sign-up with a fresh PPK.|
|`c`               | The "sign-up delay", i.e., number of blocks a validator has to wait after sign-up before starting to participate in elections.|
|MinDuration       | Minimum duration for a WaitTimer.|
|Wall Clock        | The number of seconds elapsed since a synchronization event|
|Chain Clock       | The sum of the wait times of all the blocks in the chain since the synchronization event. Each fork has its own chain clock. The Chain Clock is measured in seconds |
|BaseTime	   | The timestamp of the synchronization event. BaseTime is used to subsequently compute Wall Clock by performing Wall Clock = CurrentTime - BaseTime|
|Duration          | A 256 bit random number generated by the enclave and recorded in the WaitCertificate. The Duration influences the WaitTime of a block |
|WaitTime	   | Time in seconds for which the validator should wait before broadcasting the block. Also used in calculating the Chain Clock.
|BlockNumber       | A number indicative of the position of a block in the chain. It is also be known as the 'Block Depth'|
|Proposed Block    | A block that is currently under construction and for whom a WaitCertificate has not yet been generated|
|Claim Block       | A block whose construction is complete and for whom a WaitCertificate has been generated. A Claim Block announces a validator's candidature for the PoET Leader Election|

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Proof of Elapsed Time (PoET) Consensus method offers a solution to the
Byzantine Generals Problem that utilizes a â€œtrusted execution environmentâ€? to
improve on the efficiency of present solutions such as Proof-of-Work. This
specification defines a concrete implementation for SGX. The following
presentation assumes the use of Intel SGX as the trusted execution environment. 

At a high-level, PoET stochastically elects individual peers to execute requests
at a given target rate. Individual peers sample an exponentially distributed
random variable and wait for an amount of time dictated by the sample. The peer
with the smallest sample wins the election. Cheating is prevented through the
use of a trusted execution environment, identity verification and blacklisting
based on asymmetric key cryptography, and an additional set of election
policies.

For the purpose of achieving distributed consensus efficiently,
a good lottery function has several characteristics:

+ Fairness: The function should distribute leader election across the broadest 
possible population of participants.

+ Investment: The cost of controlling the leader election process should be 
proportional to the value gained from it.

+ Verification: It should be relatively simple for all participants to verify 
that the leader was legitimately selected.

PoET is designed to achieve these goals using new secure CPU instructions
which are becoming widely available in consumer and enterprise processors.
PoET uses these features to ensure the safety and randomness of the leader
election process without requiring the costly investment of power and
specialized hardware inherent in most â€œproofâ€? algorithms.

Sawtooth includes an implementation which simulates the secure instructions.
This should make it easier for the community to work with the software but
also forgoes Byzantine fault tolerance.

PoET 2.0 essentially works as follows:

+ Each validator requests a `WaitCertificate` from an enclave (a trusted 
	function), corresponding to each block it wishes to publish. 

+ The `WaitCertificate` contains a `Duration` as well as a related `WaitTime`.
	The `Duration` is a 256-bit random number generated using the secure
	RNG available within the SGX. The `WaitTime` is derived from the
	`Duration`. See 'CreateWaitCertificate' for details on WaitTime computation.

+ Every validator maintains two clocks, a WallClock (`WC`) - the
	time elapsed since an initial synchronization event and a ChainClock
        (`CC`) - a sum of of the wait times for all blocks in the
        chain since the synchronization event
	   
+ On the originating validator, the `WaitTime` is used to throttle broadcast of
	claim blocks. Upon creating the `WaitCertificate`, the validator waits
	until `WaitTime` seconds have elapsed before broadcasting the block over
	the gossip network
	
+ On peer validators, a block is eligible for consensus if the `CC` for its fork
        does not exceed the validator's `WC`. If it does (e.g. for an
        early-arriving block), the block is held back by `CC-WC` seconds until
        it becomes eligible for consensus and broadcast over the gossip network
  
+ The validator publishing the block with the lowest CC and extending the longest
		valid chain for a particular transaction block is elected the leader. The 
		"Leader Election" section describes this in greater detail.

+ One function, such as "CreateWaitCertificate", takes a transaction block
	 and creates a `WaitCertificate` that is guaranteed to have been created 
	 by the enclave

+ Another function, such as "CheckWaitCertificate", verifies
	that the `WaitCertificate` was created by the enclave

The PoET leader election algorithm meets the criteria for a good lottery
algorithm. It randomly distributes leadership election across the entire
population of validators with distribution that is similar to what is
provided by other lottery algorithms. The probability of election
is proportional to the resources contributed (in this case, resources
are general purpose processors with a trusted execution environment).
An attestation of execution provides information for verifying that the
certificate was created within the enclave. Further, the low cost of participation
increases the likelihood that the population of validators will be large, increasing
the robustness of the consensus algorithm.

In PoET 1.0, the PoET SGX enclave set a timer internally and would only release
a WaitCertificate, once the timer elapsed. To be considered a candidate for 
consensus, a block (a.k.a Claim Block) would have to be accompanied by the 
corresponding `WaitCertificate`. This meant that peers receiving a Claim Block 
accompanied by the `WaitCertificate`, were assured that the block had not been 
forwarded prematurely, hence the phrase 'Proof of Elapsed Time'.
PoET 2.0, however, removes the reliance on internal timers and monotonic 
counters within SGX. This necessitates a different approach for enforcing wait 
times. Rather than set a timer inside SGX, PoET 2.0 relies on SGX to provide it 
with a random 'duration', which is then used to set a timer OUTSIDE the SGX. 
A Claim Block is forwarded only after the timer has expired. The block is now 
accompanied by a `WaitCertificate` from SGX containing the 'duration' time that 
the block was expected to wait for.

While a malicious actor may send the claim block earlier than the assigned time, 
the community of validators is engaged to enforce the wait on the block. Blocks 
arriving at peers earlier than suggested by their `WaitCertificate` are held back
until the time is right to become eligible for consensus. The larger the size of
the network, the more resistant the protocol is to attack by a malicious actor 
or a cabal.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Here we describe the PoET 2.0 algorithm in greater detail. We will also call out
sections that are unchanged from PoET 1.0.

## PoET enclave Data Structures
The PoET enclave uses the following data structures:

### WaitCertificate
```
WaitCertificate {
    byte[32] Duration    # A random 256 bit number generated by SGX
    double WaitTime      # The number of seconds to wait, as a function of the
                         # Duration and the LocalMean
    double LocalMean     # The computed local mean
    byte[32] BlockID     # The BlockID passed in to the Enclave
    byte[32] PrevBlockID # The BlockID of the previous block, as stored in the
                         # previousWaitCertificate
    uint32 BlockNumber   # The length of the chain
    byte[32] TxnHash     # The hash of the transactions in the block
    byte[] ValidatorID   # The ID of the current Validator
    byte[64] Sign        # The signature of WaitCertificate computed over all
                         # the fields using the PSK 
  }
```
### Global state
```
uint32 LastBlockNumber              # The last blocknumber that the enclave has
                                    # generated
map<BlockNumber, WaitCertificate>   # Contains each wait certificate generated 
                                    # by the enclave. Only one waitCertificate 
                                    # will be allowed per block position across 
                                    # all forks
```
## PoET enclave functions
It exports the following functions:

### generateSignUpData(OPKhash)

**Returns**

```
    byte[64]  PPK
    byte[432] report # SGX Report Data Structure
```    
**Parameters**

```
    byte[32] OPKhash # SHA256 digest of OPK
```
**Description**

1. Generate fresh ECC key pair (PPK, PSK)
2. Create SGX enclave report, store ``SHA256(OPKhash|PPK)`` in 
``report_data`` field.
3. Return (PPK, report).

### createWaitCertificate(PreviousWaitCert, BlockDigest, ValidatorID)

**Returns**

```
    WaitCertificate waitCertificate
    byte[64] signature # ECDSA PSK signature of waitCertificate
```
**Parameters**

```
    WaitCertificate PreviousWaitCert #The wait certificate of the current chain 
                                     #head
    byte[] blockDigest # ECDSA signature with originator private key of SHA256
                       # digest of transaction block that the validator wants
                       # to commit
    byte[] ValidatorID
```
**Description**

1. `if(PreviousWaitCert.BlockNumber < LastBlockNumber) then throw
    StaleRequestException()`
2. Generate 256 bit random `Duration`.
3. Convert lowest 64-bits of `Duration` into double precision number 
	in `[0, 1]` represented by `duration'`
4. Compute `WaitTime = minimumDuration - localMean * log(duration')`.
5. Create WaitCertificate object `waitCertificate =
   WaitCertificate(WaitTimer, Duration, blockDigest)`
6. Compute ECDSA signature of waitCertificate using PSK: `signature =
   ECDSA_{PSK} (waitCertificate)`
7. Return `(waitCertificate, signature)`
    
### checkWaitCertificate(WaitCertificate)
**Returns**

```
    boolean isValid # The WaitCertificate passed all validity checks in Enclave
```
**Parameters**

```
    WaitCertificate waitCertificate
    byte[64] signature
    byte[]   block
    byte[32] OPK
    byte[32] PPK
```
## Initialization and Sign-up

A new validator joins an existing network by downloading the PoET SGX enclave 
and a SPID certificate for the blockchain. The validator then runs the following
sign-up procedure:

1. Start PoET SGX enclave: ENC.
2. Generate sign-up data: `(PPK, report) =
   {ENC.generateSignUpData(OPKhash)}` The ``report_data`` (512 bits)
   field in the report body includes the SHA256 digest of (OPKhash | PPK).
3. Ask SGX Quoting Enclave (QE) for linkable quote on the report (using the
   validator network's Basename).
4. If Self Attestation is enabled in IAS API: request attestation of linkable
   quote to IAS. The AEP sent to IAS must contain:

   * isvEnclaveQuote: base64 encoded quote
   * nonce: `WaitCertId_{n}`

   The IAS sends back a signed AVR containing a copy of the input AEP and the
   EPID Pseudonym.

5. If Self Attestation is enabled in IAS API: broadcast self-attested join
   request, (OPK, PPK, AEP, AVR) to known participants.

6. If Self Attestation is NOT enabled in IAS API: broadcast join request, (OPK,
   PPK, quote) to known participants.

A validator has to wait for `c` blocks to be published on the distributed
ledger before participating in an election.

>Note: 
>An important difference from PoET 1.0 is that PoET 2.0 does not store any 
>persistent data in a sealed blob. Part of the reason for moving away from the 
>sealed blob creation is the unavailability of the Monotonic counter (MTC) which 
>provides replay protection guarantees for sealed data stored to disk.

Upon receiving a join request, validators already on the network run the following
sign-up procedure:

1. Wait for a join request.
2. Upon arrival of a join request do the verification:

   If the join request is self attested (Self Attestation is enabled in IAS
   API): (OPK, PPK, AEP, AVR)
	* Verify AVR legitimacy using IAS Report Key and therefore quote
          legitimacy.
	* Verify the ``report_data`` field within the quote contains the SHA256
          digest of (OPKhash | PPK).
	* Verify the nonce in the AVR is equal to `WaitCertId_{n}`, namely the
          digest of the most recently committed block. It may be that the sender
          has not seen `WaitCertId_{n}` yet and could be sending
          `WaitCertId_{n'}` where `n'<n`.
          In this case the sender should be urged to updated his/her
          view of the ledger by appending the new blocks and retry. It could
          also happen that the receiving validator has not seen
          `WaitCertId_{n}` in which case he/she should try to update his/her
          view of the ledger and verify again.
	* Verify MRENCLAVE value within quote is equal to PoET\_MRENCLAVE (there
      could be more than one allowed value).
	* Verify basename in the quote is equal to distributed ledger Basename.
	* Verify attributes field in the quote has the allowed value (normally
          the enclave must be in initialized state and not be a debug enclave).
 
   If the join request is not self attested (Self Attestation is NOT enabled in
   IAS API): (OPK, PPK, quote)

   * Create AEP with quote:

      * isvEnclaveQuote: base64 encoded quote

 3. Send AEP to IAS. The IAS sends back a signed AVR.
 4. Verify received AVR attests to validity of both quote and
    save EPID Pseudonym.
 5. Verify ``report_data`` field within the quote contains the SHA256 digest of
	(OPKhash | PPK).
 6. Verify MRENCLAVE value within quote is equal to PoET\_MRENCLAVE (there could
	be more than one allowed value).
 7. Verify basename in the quote is equal to distributed ledger Basename.
 8. Verify attributes field in the quote has the allowed value (normally the
    enclave must be in initialized state and not be a debug enclave).
   
   If the verification fails, exit.

   If the verification succeeds but the SGX platform identified by the EPID
   Pseudonym in the quote has already signed up within the last 'K' blocks, ignore 
   the join request and exit.

   If the verification succeeds:

   * Pass sign-up certificate of new participant (OPK, EPID Pseudonym, PPK,
     current `WaitCertId_{n}` to upper layers for registration in
     EndPoint registry.
   * Goto 1
     
# Leader Election
### Wall Clock and Chain Clock
The PoET 2.0 algorithm maintains two clocks, the _Wall Clock_ and the
_Chain Clock_.

The Wall Clock (`WC`) measures the time elapsed since the synchronization event.
For the first validator, the synchronization event occurs when the genesis block
is added as the first block in the chain.  For validators subsequently added to
the network, the synchronization event occurs when the Synchronization block arrives 
at the validator. See "Bootstrapping new validator nodes" for details. 
The Wall Clock is maintained by the validator on the 'untrusted' portion of the
PoET code. 

The Chain Clock (`CC`) measures the cumulative `WaitTime` of all the blocks in 
the chain (each fork will have its own `CC`).

Upon receiving the Sync Block, `WC` and `CC` are initialized to 0. 

>Note 1: In practice, the WC may be calculated by recording the system time
>(`BaseTime`) at the moment of the synchronization event  and subsequently 
>subtracting this timestamp from the current time (`WC = CurrentTime - BaseTime`). 

>Note 2: Notice that the CC is a function of the WaitTime, which is computed within
>the enclave.

### Block Publishing

To publish a block in the election phase a validator runs the following procedure: 

1. Start the PoET SGX enclave: ENC
2. Assemble a Proposed Block (outside the enclave)
3. Call `(waitCertificate, signature) = ENC.createWaitCertificate(blockDigest)`
4. Wait `WaitCertificate.WaitTime` seconds
5. Broadcast `(waitCertificate, signature, block, OPK, PPK)` over the
   Gossip Network.

   Here `block` is the transaction block identified by `blockDigest`.

>Once a `WaitCertificate` is created, the enclave stores the `BlockNumber` and 
>`WaitCertificate` in a table. The enclave only allows one `WaitCertificate` per
>`BlockNumber`. Further, it uses the `LastBlockNumber` to ensure a `WaitCertificate` 
> is created only for future blocks. This prevents an attacker from 'going back 
>in time' to request a `WaitCertificate` for a past block.

### Block Verification
Upon receiving a Block from a peer over the Gossip Network, a validator first 
checks the block for eligibility before it can be allowed to participate in 
consensus.

#### Block Eligibility Checks
1. Verify the PPK and OPK belong to a registered validator by checking the
   EndPoint registry.

2. Verify the signature is valid using sender's PPK.

3. Verify the PPK was used by sender to commit less than `K` blocks by checking 
   EndPoint registry (otherwise sender needs to re-sign).

4. Verify the `WaitCertificate.LocalMean` is correct by comparing
   against `LocalMean` computed locally.

5. Verify the waitCertificate.blockDigest is a valid ECDSA signature of the
   SHA256 hash of block using OPK.

6. Verify the sender has been winning elections according to the expected
   distribution (see z-test documentation).

7. Verify the sender signed up at least `c` committed blocks ago, i.e.,
   respected the `c` block start-up delay.

8. Verify that the block is not an early-arriving block by computing 
`CC' = CC + WaitCertificate.WaitTime`, then doing `WC >= CC'`, where `CC` is the 
existing Chain Clock of the fork being extended by the new block.

>Implementation Note1: While checking for early arriving blocks, implementations
>may choose to add a constant `E`, representing average delay in seconds due to 
>network latency. The check will then be `WC >= CC' + E`

>Implementation Note2: There may be cases where a block arrives earlier than 
>its predecessors. In such cases, the block will need to be cached until its
>predecessors arrive before it can be checked for Eligibility.

A block passing all checks is considered 'Eligible' and proceeds to participate
in consensus. It is also then broadcast over the Gossip network to the 
validator's peers.

An early arriving block (where `WC < CC'`) is considered 'Ineligible'. The block
is cached for `CC' - WC` seconds until it becomes 'Eligible'. It is then
broadcast to validator's peers over the Gossip Network.

>In the absence of an enclave enforced block publishing delay as in PoET 1.0,
>the Wall Clock (`WC`) acts as a community enforced block publishing delay.
>Validators will only forward blocks when they can independently verify that
>sufficient time has elapsed. 

>Similar to PoET 1.0, the primary aim of this mechanism is to improve efficiency
>of leader election. It aims at having a single claim outsanding at any given
>time in the network.

### Block Commit and Fork Resolution
Once a validator has determined that an incoming block is Eligible, it performs
the following steps:

1. If the incoming block extends the current chain head:
   * If the validator is still working on a Proposed Block (i.e. no Claim
     Block has been created), commit the incoming block as the Chain Head, drop
     the current proposed block. 
   * If the validator has published a Claim Block, compare the 
     `WaitCertificate.Duration` of both blocks. The block having the lower
     Duration is the winner. If the incoming block is the winner, commit the 
     incoming block and update the Chain Head.
   * Check the Block Cache for any blocks that may claim the new Chain Head
     as a parent (i.e. blocks that may have arrived out of order). If such 
     blocks are identified, start the Block Eligibility checks on those blocks.
	
2. If the incoming block claims an earlier block as a parent, walk the chain back 
   until a common parent is reached. This may involve pulling previously
   discarded blocks out of the block cache. Check the chain length of each
   fork.
   * If the current (active) fork is longer, discard the incoming block
   (implementations may choose to cache discarded blocks for some time)
   * If the new fork is longer, switch to the new fork.
   * If the two forks are of equal length
     * compare the Chain Clock(`CC`) of each fork. The fork with the lower `CC`
       is the winner. 
     * If the Chain Clocks are equal, compare the `WaitCertificate.Duration` 
       of the topmost blocks in the forks. The block with the lower Duration is
       the winner.

3. If the Chain Head has been updated, discard any existing Proposed Block and
   start building a new Proposed Block.

## Bootstrapping new validator nodes

When a new validator joins the network, it submits a registration transaction.
It then starts synchronizing its ledger and state with its neighbors by requesting
blocks. 

It then performs the following steps to bootstrap its Wall Clock:

1. Start accepting incoming blocks and requesting missing blocks from neighbors 
	to build the chain until the genesis block is received.
2. When a block containing the validator's registration transaction is received,
	start the 'C' block counter. Let this block be known as the 'registration' 
	block.
3. Pick a random block number between the 'registration' block and the C'th block.
	We call this block the 'Synchronization' block, since it will be used to 
	initialize and synchronize the two clocks. 
4. To be considered a Synchronization block, the block must extend the current 
	chain head.
5. Once the Synchronization block is received, the WallClock & ChainClock are 
	initialized. This block is henceforth used for ChainClock computation.
6. Any incoming blocks before the C'th block are considered Eligible.
7. The start of the validator's participation in consensus (after the C-block
	delay) is also when the validator starts enforcing the Eligibility
	checks on incoming blocks.

This mechanism bootstraps the Wall Clock to a reasonable value and allows
the validator to effectively participate in consensus. 

## Clock Drifts and Network Latency

PoET 2.0 relies on the system clock in two instances: for setting a timer to
wait for `WaitCertificate.WaitTime` seconds and for maintaining the Wall Clock.

Clock drifts on a validator may cause it to broadcast a block earlier or later
than intended. On the peer validator handling incoming blocks, clock drifts will
impact the Wall Clock being used to determine block eligibility. Depending on 
the drift, blocks may become eligible earlier or later than ideal.

The Leader Election mechanism itself is independent of the clock and will not be
impacted by drifts.

For forward clock drifts, the primary impact on the network will be of blocks
being broadcast earlier. This could result in situations where multiple Claim Blocks
are active at any given time, resulting in decreased efficiency of the consensus 
mechanism. Negative clock drifts will cause blocks to be broadcast later, potentially
impacting SLAs for block publishing intervals.

Because Leader Election is isolated from clock drifts, the the clocks need only 
be reasonably accurate to keep the network operating at the desired level of 
efficiency. Existing system clock synchronization mechanisms like NTP
etc. may be sufficient for PoET 2.0 requirements.

Network latencies may be exploited by malicious nodes to broadcast blocks
earlier than required. Again, this behavior doesn't impact leader election. The
community of validators may choose to take observed network latencies into
account while determining block eligibility, thereby regulating the frequency of
blocks published by malicious or compromised nodes.

# Drawbacks
[drawbacks]: #drawbacks

# Rationale and alternatives
[alternatives]: #alternatives

Poet 2.0 improves on PoET 1.0 adding support for SGX devices without PSE. 
Current PoET 1.0 limits the availability of PoET consensus only to SGX devices 
with PSE.

# Prior art
[prior-art]: #prior-art

PoET 2.0 is based on PoET 1.0 and newer solutions for overcoming 
challenges thereof. 

# Unresolved questions
[unresolved]: #unresolved-questions

