- Feature Name: send_serialize_txn
- Start Date: 2018-08-03
- RFC PR:
- Sawtooth Issue:

# Summary
[summary]: #summary

Requesting to add API to the sdk - txn.get_serialized_header(), that will return
the transaction header bytes.

# Motivation 
[motivation]: #motivation

For security reasons transaction processor would like to verify the incoming
transaction header, and signature.  For example, in the private ledger TP (C++)
that runs inside SGX enclave, we can't trust the signature verification done by
sawtooth since it happens outside of SGX.  In order to verify the transaction
signature, TP needs to get the serialized transaction header and not just all
the fields.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC adds a new method to directly access the transaction header bytes.
Transaction Processor developers will use this API when they choose to re-verify
the signature inside of a transaction processor.

No change is recommended to the existing methods, so implementation of this RFC
should not cause any breakage for pre-existing transaction processors.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

There are several ways to implement this api. First and most simple, txn object
that arrives to TP in 'apply' method will have an api to re-serialize the
transaction header. This solution could be problematic since it requires trust
in protobuf that de-serializing and re-serializing back will produce same bytes.

A more reliable soultion is for the validator to forward the serialized header
bytes of the transaction object, now calling the new get_serialized_header()
will provide the bytes without the need to re-serialize. For this recommended
solution, there is also an option to add flag to TP registration in order to let
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
    TpProcessRequestHeaderStyle process_request_header_style;
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

# Drawbacks
[drawbacks]: #drawbacks

Most transaction processors do not need the raw header bytes.
Enabling this feature may mean carrying extra data through the sawtooth-core
internal pipeline.

# Rationale and alternatives
[alternatives]: #alternatives

An alternative would be to include a signature within the payload itself. That
has the downside of bloating the transactions with an additional redundant
field.

# Prior art [prior-art]: #prior-art

This problem appears unique to the conjunction of the sawtooth architecture and
the use of a trusted execution environment for the transaction handling logic.

# Unresolved questions [unresolved]: #unresolved-questions

Internal sawtooth-core management of the header is not prescribed in this RFC.
