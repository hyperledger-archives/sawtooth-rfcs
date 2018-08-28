- Feature Name: seth\_rpc\_extensions
- Start Date: 2018-07-30
- RFC PR:
- Sawtooth Issue:

# Summary
[summary]: #summary

The Seth JSON-RPC API is an implementation of the Ethereum JSON-RPC API, used
for cross-compatibility with existing Ethereum clients. The Ethereum JSON-RPC
API lacks several areas of functionality that require the existing Seth client
to talk directly to the Sawtooth REST API. This RFC proposes several JSON-RPC
API extensions that will allow the CLI to become a thin client that builds on
top of the `web3` library.

# Motivation
[motivation]: #motivation

Effecting this RFC proposal will bring us in line with existing extensions to
the Ethereum JSON-RPC API, allowing current Ethereum clients to work with Seth
without modification.

Moving the CLI to becoming a thin client will allow a simplified architecture
that stores private keys in one place, and will drastically reduce code
complexity. It will also eliminate the need for clients to understand how to
talk to the Sawtooth REST API.

An additional benefit will be that dogfooding our JSON-RPC API will ensure that
the implementation is tested and up-to-date with other implementations.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Managing Accounts

The [parity] project has faced the same issue of allowing account management
via the [JSON-RPC API][personal-api] that it represents. This RFC proposes
implementing a compatible API that presents these endpoints:

 - personal_listAccounts
 - personal_newAccount
 - personal_unlockAccount
 
Additionally, in order to facilitate import existing accounts, the API will
present this method that [geth][import-raw-key] has implemented:

 - personal_importRawKey

[parity]: https://www.parity.io/
[import-raw-key]: https://github.com/ethereum/go-ethereum/wiki/Management-APIs#personal_importrawkey
[personal-api]: https://wiki.parity.io/JSONRPC-personal-module.html

## Managing Permissions

Two new endpoints will be added for general permissions management, and one
existing endpoint will be extended to handle permissions. The new endpoints
will be `seth_getPermissions` and `seth_setPermissions` and will allow general
retrieval and setting of permissions.

The existing `eth_sendTransaction` endpoint will be extended to allow a
`permissions` argument.

Finally, the proposed `personal_importRawKey` and `personal_newAccount`
endpoints will accept a `permissions` argument as well.

## Contract Chaining

Sawtooth has the concept of a limited set of input and output address that a
transaction is limited to reading from/writing to. That concept doesn't exist
in Ethereum, and makes the concept of contract chaining (e.g. creating a
contract from another contract) difficult. The `eth_sendTransaction` endpoint
will be extended to allow an argument that sets the list of inputs and outputs
to be the entire Sawtooth Seth address space, enabling general contract
chaining.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Newly-proposed JSON-RPC methods

- [personal_listAccounts](#personal_listaccounts)
- [personal_newAccount](#personal_newaccount)
- [personal_unlockAccount](#personal_unlockaccount)
- [personal_importRawKey](#personal_importrawkey)
- [seth_getPermissions](#seth_getpermissions)
- [seth_setPermissions](#seth_setpermissions)

### personal_listAccounts

Lists all stored accounts.

#### Parameters

None

#### Returns

- `Array` - A list of 20 byte account identifiers.

#### Example

Request
```bash
curl --data '{"method":"personal_listAccounts","params":[],"id":1,"jsonrpc":"2.0"}' -H "Content-Type: application/json" -X POST localhost:3030
```

Response
```js
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": [
    "0x531af7d0d177b7568fa4350ef07b5218e2976175",
    "0x701d1862d9371510c7397e8d01a534a6db7bce04"
  ]
}
```

***

### personal_newAccount

Creates new account.

**Note:** it becomes the new current unlocked account. There can only be one unlocked account at a time.

#### Parameters

0. `String` - (optional) Password for the new account.

```js
params: ["*******"]
```

#### Returns

- `Address` - 20 Bytes - The identifier of the new account.

#### Example

Request
```bash
curl --data '{"method":"personal_newAccount","params":["*******"],"id":1,"jsonrpc":"2.0"}' -H "Content-Type: application/json" -X POST localhost:3030
```

Response
```js
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": "0xd971e64810ebae561cf94c46850e3589b3e89ddd"
}
```

***

### personal_unlockAccount

Unlocks specified account for use.

If permanent unlocking is disabled (the default) then the duration argument will be ignored, and the account will be unlocked for a single signing. With permanent locking enabled, the duration sets the number of seconds to hold the account open for. It will default to 300 seconds. Passing 0 unlocks the account indefinitely.

There can only be one unlocked account at a time.

#### Parameters

0. `Address` - 20 Bytes - The address of the account to unlock.
0. `String` - Passphrase to unlock the account.
0. `Quantity` - (default: `300`) Integer or `null` - Duration in seconds how long the account should remain unlocked for.

```js
params: [
  "0x81a09c6ccb1ef629385b5775aeb204466c30619e",
  "*******",
  null
]
```

#### Returns

- `Boolean` - whether the call was successful

#### Example

Request
```bash
curl --data '{"method":"personal_unlockAccount","params":["0x81a09c6ccb1ef629385b5775aeb204466c30619e","*******",null],"id":1,"jsonrpc":"2.0"}' -H "Content-Type: application/json" -X POST localhost:3030
```

Response
```js
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": true
}
```

***

### personal_importRawKey

Imports the given unencrypted private key, optionally encrypting it

#### Parameters

0. `String` - The private key to import
0. `String` - (optional) The password to encrypt the key with 

```js
params: [
  "...",
  "*******"
]
```

#### Returns

- `Address` - 20 Bytes - The identifier of the new account.

#### Example

Request
```bash
curl --data '{"method":"personal_importRawKey","params":["*******"],"id":1,"jsonrpc":"2.0"}' -H "Content-Type: application/json" -X POST localhost:3030
```

Response
```js
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": "0x556e0f5279ea0a40e711efd14552ef62eec444f5"
}
```

***

### seth_getPermissions

Returns the permissions of the given address.

#### Parameters

0. `Address` - 20 Bytes - The address of the account to retrieve permissions for.

```js
params: [
  "0xf509aebd62164a8842333669c0f2dbfc3aea9d05",
]
```

#### Returns

- `Data` - The current permissions of the account

#### Example

Request
```bash
curl --data '{"method":"seth_getPermissions","params":["0xf509aebd62164a8842333669c0f2dbfc3aea9d05"],"id":1,"jsonrpc":"2.0"}' -H "Content-Type: application/json" -X POST localhost:3030
```

Response
```js
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": "-root,+send,+call,+contract,+account"
}
```

***

### seth_setPermissions

Sets permissions for the given address.

#### Parameters

0. `Address` - 20 Bytes - The address of the account to set permissions for.
0. `String` - Permissions to set.

```js
params: [
  "0xb9dbd06c0436c227eeb912cf2fa872403acd1a89",
  "-root,+send,+call,+contract,+account",
]
```

#### Returns

- `Boolean` - Whether the call was successful

#### Example

Request
```bash
curl --data '{"method":"seth_setPermissions","params":["0xb9dbd06c0436c227eeb912cf2fa872403acd1a89","-root,+send,+call,+contract,+account"],"id":1,"jsonrpc":"2.0"}' -H "Content-Type: application/json" -X POST localhost:3030
```

Response
```js
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": true
}
```

***

# Drawbacks
[drawbacks]: #drawbacks

There should be no general drawbacks to extending the existing API code beyond
the work involved, as any extra functionality will be ignored by clients that
are not aware of it. However, there may be slight drawbacks if the Seth CLI
sends arguments that are not understood to non-Seth JSON-RPC implementations.
This can be mitigated by having the Seth CLI be capable of not sending
additional flags when talking to non-Seth JSON-RPC APIs implementations.

# Rationale and alternatives
[alternatives]: #alternatives

This RFC proposes a step forward to aligns Seth with existing extensions to
the Ethereum JSON-RPC API. Not going forward with this RFC will leave a wide
gap between the Seth and other implementations of the Ethereum JSON-RPC API and
its extensions.

One alternative considered is setting up an additional service that handles the
functionality not present in the standard Ethereum JSON-RPC API. This option
was rejected due to the extra complexity involved in managing that service.

# Prior art
[prior-art]: #prior-art

This RFC conforms to many elements from these two JSON-RPC API extensions:

https://wiki.parity.io/JSONRPC

https://github.com/ethereum/go-ethereum/wiki/Management-APIs#list-of-management-apis

# Unresolved questions
[unresolved]: #unresolved-questions

None at present
