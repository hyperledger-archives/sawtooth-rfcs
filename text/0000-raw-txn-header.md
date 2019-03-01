- Feature Name: raw_txn_header
- Start Date: 2018-08-03
- RFC PR:
- Sawtooth Issue:

# Summary
[summary]: #summary

This RFC proposes an additional API call, txn.get_raw_header(), which will
return the transaction's header bytes.

# Motivation
[motivation]: #motivation

For security reasons transaction processor would like to verify the incoming
transaction header, and signature, if TP that runs inside SGX enclave, we
can't trust the signature verification done by sawtooth since it happens
outside of SGX (TEE).
In order to verify the transaction signature, TP needs to get the serialized
transaction header and not just all the fields.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC adds a new method to directly access the transaction header bytes.
Transaction Processor developers will use this API when they choose to re-verify
the signature inside of a transaction processor.

No change is recommended to the existing methods, so the implementation of this RFC
should not cause any breakage for pre-existing transaction processors.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Calling get_raw_header() will return the serialized bytes of the transaction
object without the need to re-serialize.
There is also an option to add a flag to the TP registration in order to let the
TP decide if it requires this API.

A possible implementation approach would be to modify TpRegisterRequest to
indicate that the transaction processor desires header bytes.

    message TpRegisterRequest {
        message TpProcessRequestHeaderStyle {
            STYLE_UNSET,
            EXPANDED,
            RAW
        }

        string family = 1;
        string version = 2;
        repeated string namespaces = 4;
        uint32 max_occupancy = 5;
        TpProcessRequestHeaderStyle process_request_header_style = 6;
    }

With a new field within TpRegisterRequest:

    message TpProcessRequest {
        TransactionHeader header = 1;  // The transaction header
        bytes payload = 2;  // The transaction payload
        string signature = 3;  // The transaction header_signature
        string context_id = 4; // The context_id for state requests.
        bytes header_raw = 5;
    }

For EXPANDED, 'header' field would be filled in, for RAW, 'header_raw' would
be filled in.
For backwards compatibility, EXPANDED is the default if RAW is not set.

# Drawbacks
[drawbacks]: #drawbacks

Adding this feature will increase the stable API surface for the benefit of a
single transaction processor implementation. (Though in theory it could be
useful to other transaction processors in the future, we do not know of any
plans.) The stable API is important because we have made a commitment of
backward support, so this feature, if added, will need to be supported into
the future.

This will also increase the complexity of the validator slightly.

# Rationale and alternatives
[alternatives]: #alternatives

An alternative would be that transaction object which is provided to the TP's
'apply' method will have an API to re-serialize the transaction header.
This solution could be problematic since it requires placing trust in protobuf
that de-serializing and re-serializing back will produce the same bytes.

An alternative would be to sign the payload itself. That has the downside of
bloating the transactions with an additional redundant field from the header.

# Prior art [prior-art]: #prior-art

The transaction processor API in Hyperledger Sawtooth v0.8 sent the raw header
bytes, however this API was changed during the 1.0 stabilization period to
its current state. The requirement that raw header bytes be included in the
transaction processor API seems to be unique to processors that use a trusted
execution environment.

# Unresolved questions [unresolved]: #unresolved-questions

Internal validator management of the header is not prescribed in this RFC.
