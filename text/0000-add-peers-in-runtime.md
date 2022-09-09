- Feature Name: `add_peers_in_runtime`
- Start Date: 2018-11-12
- RFC PR: (leave this empty)
- Sawtooth Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes functionality to add and remove static
peer connections while a validator node is running in the `static` peering mode.
This RFC also adds the corresponding extensions to the off-chain permssioning
model and adds the `admin` endpoint.

# Motivation
[motivation]: #motivation

When an administrator adds a new node to an existing Sawtooth network, he has to
restart a node with new peering settings. Restarting the process adds
substantial complexity to infrastructure automation, and incurs system downtime.
To resolve this problem, our team proposes to add a method to add new peers to a
running node and to remove peers from a running node.

The `admin` endpoint is added to separate the administrative functions from
client requests and to facilitate better security (for instance, using firewall
rules to restrict the access to the `admin` endpoint).

An example use case is using Sawtooth along with a service discovery system like
[Consul](https://www.consul.io):

- A Consul node is set up along with the Sawtooth node;
- A middleware continuously fetches the changes of the peer list from the Consul
  node;
- The middleware adds peers to the Sawtooth node via the validator admin
  endpoint.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When an administrator adds new peers to the network, he likely wants to connect
to them without needing to restart existing validators with an updated `--peers`
parameter value. To add a new peer, the administrator can send send the
`AdminAddPeersRequest` to the `admin` endpoint of the Sawtooth validator.

The Protocol Buffers definition for this message is the following:

```protobuf
message AdminAddPeersRequest {
    repeated string peers = 1;
}
```

Simple Python example using `sawtooth_sdk.messaging.stream`:

```python
new_peer = 'tcp://192.168.0.100:8008'
add_peer_request = AdminAddPeerRequest(peer_uri=new_peers)
future = stream.send(Message.ADMIN_PEERS_ADD_REQUEST,
                     add_peer_request.SerializeToString())
response_serialized = future.result().content
response = AdminAddPeerResponse()
response.ParseFromString(response_serialized)
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Admin endpoint

The `admin` is the separate validator endpoint that separates administrative
requests from client requests.

All request passed to this endpoint are signed and checked against the `admin`
off-chain permissioning policy.

The structure of a request to the `admin` endpoint is the following:

```protobuf
message AdminMessage {
    message AdminMessageHeader {
        Message.MessageType message_type = 1;
        string payload_sha512 = 2;
        string signer_public_key = 3;
    }

    AdminMessageHeader header = 1;
    string header_signature = 2;
    bytes payload = 3;
}
```

Administrative messages are passed as specified in `protos/validator.proto` and
the special type is introduced for administrative messages:

```protobuf
message Message {
    enum MessageType {
        // ...
        ADMIN_REQUEST = 400;
        // ...
    }
}
```

## Protocol Buffers definitions
[protobuf]: #protobuf

The definitions proposed to be added to `protos/admin_peers.proto`:

```protobuf
message AdminAddPeerRequest {
    repeated string peer_uri = 1;
}

message AdminAddPeerResponse {
    enum Status {
        STATUS_UNSET = 0;
        OK = 1;
        INTERNAL_ERROR = 2;
        ADMIN_AUTHORIZATION_ERROR = 3;
        // One or more of peer URIs were malformed. List of malformed URIs is in
        // the `invalid_uris` field of this response.
        INVALID_PEER_URI = 4;
        MAXIMUM_PEERS_CONNECTIVITY_REACHED = 5;
        AUTHORIZATION_VIOLATION = 6;
        CONNECTION_DROPPED = 7;
    }
    Status status = 1;
}

message AdminRemovePeerRequest {
    repeated string peer_uri = 1;
}

message AdminRemovePeerResponse {
    enum Status {
        STATUS_UNSET = 0;
        OK = 1;
        INTERNAL_ERROR = 2;
        ADMIN_AUTHORIZATION_ERROR = 3;
        // The requested peer do not exist
        PEER_NOT_FOUND = 4;
    }
    Status status = 1;
}
```

The rationale behind the `invalid_uris` is to be more precise about what is
wrong and to ease the debugging process for developers.

New message types should also be added to the `admin` endpoint:

```protobuf
message AdminMessage {

    enum AdminMessageType {
        // ...
        ADMIN_PEERS_ADD_REQUEST = 411;
        ADMIN_PEERS_ADD_RESPONSE = 412;
        ADMIN_PEERS_REMOVE_REQUEST = 413;
        ADMIN_PEERS_REMOVE_RESPONSE = 414;
        // ...
    }
    // ...
}
```

### Adding peers

When the validator receives a new request for adding peers it:

- Validates the format of peer URI which has to be `tcp://ADDRESS:PORT_NUMBER`;
- If the validation was successful, then the validator tries to connect to a
  provided peer. If the connection was successful, it returns the `OK` status.
  Otherwise, the corresponding error status is returned.

Edge cases:

- The peer address format was wrong in one or more of the provided peers. If
  that happens, then the request fails without adding any new peers to the peers
  list and returns the `INVALID_PEER_URI` status along with the list of faulty
  peer URIs.
- If the `--maximum-peer-connectivity` parameter was provided to the validator,
  then the validator checks if it has reached the maximum peer connectivity, and
  fails with an error if so. The validator also fails if it cannot add _all_ of
  the peers provided in a request without breaking the provided
  `maximum-peer-connectivity`.

### Removing peers

When the validator receives a new request for removing peers it does the
following:

- Validates the format of peer URI which has to be `tcp://ADDRESS:PORT_NUMBER`;
- If a peer is connected, the validator removes it. Otherwise, the
  `PEER_NOT_FOUND` error is thrown.

# Drawbacks
[drawbacks]: #drawbacks

- The proposed solution does not specify any connection retry policies leaving
  it to the existing peering implementation.

# Rationale and alternatives
[alternatives]: #alternatives

Alternatives were not considered and judging from multiple examples this is the
state-of-the-art solution.

# Prior art
[prior-art]: #prior-art

- [`addnode` in Bitcoin JSON RPC][btcrpc]
- [`admin_addPeer` in Ethereum management API][ethrpc]

Those two allow adding new peers in their platforms. Interesting points:

- Ethereum can return the connection status;
- Bitcoin allows specifying the connection retry policy.

# Unresolved questions
[unresolved]: #unresolved-questions

- During the pre-RFC discussion on the #sawtooth-core-dev channel, there was no
  final solution on "should it be included to the REST API or no?"
- In the same discussion, there was a proposition to resolve security issues by
  using this feature along with the permissioning module. If we do not add this
  feature to the REST API and hence to the administrative utilities, then I do
  not see any point in permissioning because the described feature remains an
  internal interface of the application. Even if we do, then we can restrict the
  access to that feature by using a proxy as suggested in the documentation.
- Should our team include the integration example of this solution for Consul?
- Should the permissioning be left as it is or generalized to a structure like
  `(admin_public_key, signature, request)`?

[btcrpc]: https://bitcoincore.org/en/doc/0.16.0/rpc/network/addnode/
[ethrpc]: https://github.com/ethereum/go-ethereum/wiki/Management-APIs#admin_addpeer
