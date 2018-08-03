- Feature Name: send_serialize_txn
- Start Date: 2018-08-03
- RFC PR:
- Sawtooth Issue:

# Summary
[summary]: #summary

Requesting to add API to the sdk - txn.get_serialized_header(), that will return the header bytes.

# Motivation
[motivation]: #motivation

For security reasons transaction processor would like to verify the incoming transaction header, and signature.
For example, in the private ledger TP (C++) that runs inside SGX enclave, we can't trust the signature verrification done by sawtooth since it happens outside of SGX.
In order to verify the transaction signature, TP needs to get the serialized transaction header and not just all the fields.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Sawtooth is using protobuf to serialize and de-serialize transaction, sdk provide the de-serialized transaction to TP with access to transaction fields.
This will provide TP developers to access the transaction header bytes.

For implementation-oriented RFCs (e.g. for validator internals), this section
should focus on how contributors should think about the change, and give
examples of its concrete impact. For policy RFCs, this section should provide
an example-driven introduction to the policy, and explain its impact in
concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

there are several ways to implement this api.
first and most simple, txn object that arrives to TP in 'apply' method will have an api to re-serialized transaction header.
this solution could be problematic since it requires trust in protobuf that de-serializing and re-serializing back will produce same bytes.
the other solution is for validator to forward the serialized header bytes to the transaction object, now calling the new get_serialized_header() will provide the bytes without the need to re-serialize.
for the second solution, there is also an option to add flag to TP registration in order to let TP decide if it requires this API.

The section should return to the examples given in the previous section, and
explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For consensus, global state, transaction processors, and smart contracts
  implementation proposals: Does this feature exists in other distributed
  ledgers and what experience have their communities had?
- For community proposals: Is this done by some other community and what were
  their experiences with it?
- For other teams: What lessons can we learn from what other communities have
  done here?
- Papers: Are there any published papers or great posts that discuss this? If
  you have some relevant papers to refer to, this can serve as a more detailed
  theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other distributed ledgers, provide readers of your RFC with
a fuller picture.  If there is no prior art, that is fine - your ideas are
interesting to us whether they are brand new or if it is an adaptation.

Note that while precedent set by other distributed ledgers is some motivation,
it does not on its own motivate an RFC.  Please also take into consideration
that Sawtooth sometimes intentionally diverges from common distributed
ledger/blockchain features.

# Unresolved questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?
