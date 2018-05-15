<!--
  Copyright 2018 Cargill Incorporated

  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->

- Feature Name: wasm\_smart\_contracts
- Start Date: 2018-04-16
- RFC PR: [hyperledger/sawtooth-rfcs#7](https://github.com/hyperledger/sawtooth-rfcs/pull/7)
- Sawtooth Issue: N/A

# Summary
[summary]: #summary

Proposes Sawtooth Sabre, a transaction family which implements on-chain smart
contracts executed in a WebAssembly virtual machine.

WebAssembly (Wasm) is a stack-based virtual machine newly implemented in major
browsers. It is well-suited for the purposes of smart contract execution due to
its sandboxed design (executed in an environment independent from the host),
growing popularity, and tool support.

# Motivation
[motivation]: #motivation

On-chain smart contracts have excellent characteristics for ad-hoc distributed
deployment. The code of the smart contract is deployed (or loaded) into global
state via a transaction.  Other transactions cause that on-chain code to be
loaded into a virtual machine and executed. The most commonly used on-chain
smart contract system is Ethereum which uses the Ethereum Virtual Machine (EVM)
to execute smart contracts.

This differs from Sawtooth's transaction processors, which are not deployed
on-chain; transaction processors are distributed as software in a manner
similar to Sawtooth itself. This has various drawbacks from a distributed
deployment perspective, since some level of network-wide coordination is
necessary to modify transaction processors.  However, conceptually transaction
processors can be thought of as a layer below on-chain smart contracts.  In
fact, implementing on-chain smart contracts using transaction processors was
one of the initial motivators behind Sawtooth's modular transaction execution
platform design.

An existing transaction family, Sawtooth Seth, implements an EVM on-chain smart
contract solution for Sawtooth.  Seth's strength is compatibility with the
Ethereum tool chain and existing smart contracts written for Ethereum (for
example, using Solidity). Despite the awesome capability it provides, Seth is
restricted by the Ethereum design and does not expose the entire feature set of
Sawtooth transaction processors.

Sawtooth Sabre fills this gap by providing a permissioned WebAssembly-based
smart contract platform which provides full feature-parity with the Sawtooth
SDK's transaction processor API. It further attempts to provide these
capabilities in the most source-compatible way possible to make it easy to
transition existing transaction processors to run as a Sabre smart contract.

The goal of Sawtooth Sabre is to provide a Sawtooth-native on-chain smart
contract capability to Sawtooth. The goal of the design and implementation
described in this RFC is to jumpstart Sabre with a minimal but usable
implementation, not describe every eventual feature.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Sawtooth Sabre is an on-chain smart contract platform which executes smart
contracts in a WebAssembly virtual machine. It consists of three primary
components:

- Sabre transaction processor (sabre-tp)
- CLI command (sabre)
- Sabre Smart Contract SDK

The design of Sabre includes three fundamental data structures:

- Namespace Registry
- Contract Registry
- Contract

## Namespace Registry

Generally speaking, transaction processors can read and write arbitrary global
state addresses. While restrictions can be placed upon the transaction
processor via a couple of different existing mechanisms, they are not granular
enough to be used in the context of Sabre, since all Sabre contracts would be
subject to the same restrictions.

Smart contracts and namespaces of global state are not necessarily a one-to-one
relationship. For example, we could create separate Sabre smart contracts
which provide multiplication and addition operations on the intkey namespace.
Both smart contracts would operate on the same intkey namespace. Since
this is the case, for a fully permissioned system we desire the ability to
control access to the namespace independently from the permission to deploy and
maintain the smart contracts.

In Sabre, a namespace registry is used to control permissions related to
a namespace. The entry in the namespace registry for a namespace also contains
a list of owners, which can be updated through transactions. An owner of the
namespace can add and remove permissions associated with specific contracts.
Thus, the owner of the intkey namespace can add permissions which allow the
multiplication and addition smart contracts read and write access. This
delegates some authority to the smart contract owners; so while the owner of
the namespace and owner of the contract are not necessarily the same, there is
an implied degree of trust and coordination between them.

## Contract Registry

Sabre contracts are referenced by name. In the example of smart contracts which
multiply and add values in the intkey namespace, the names of the contracts
might be intkey-multiply and intkey-add.

The contracts are also versioned. Following our example, intkey-multiply v1.0.0
may have only allowed multiplying two values, while a more recent
intkey-multiply v1.1.0 may allow any number of values to be multiplied
together.  The intent is to allow developers to follow Semantic Versioning best
practices.

A contract registry stores information related to each version of the contract
and the list of owners. Owners are allowed to load new versions of contracts.

## Contract

A Sabre contract consists of the compiled Wasm code and related metadata (name,
version, input/output addresses, etc.).

Writing Sabre contracts is similar in nature to writing Sawtooth transaction
processors, and the Sabre SDK is intended to be a drop-in replacement for the
Sawtooth SDK.  Thus, porting existing Sawtooth transaction processor code to
Sawtooth Sabre is relatively easy - just change a few import statements to
refer to the Sabre SDK instead of the Sawtooth SDK. The details of running
within a Wasm interpeter are hidden from the smart contract author.

It is also possible to maintain dual-target smart contracts, where the smart
contract can be run either as a native transaction processor or as a Sabre
contract.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## State

All Sabre objects are serialized using Protocol Buffers before being stored in
state. Theses objects include namespace registries, contract registries, and
contracts. All objects are stored in a list to handle hash collisions.

### NamespaceRegistry

A namespace is a state address prefix used to identify a portion of state.
The NamespaceRegistry stores the namespace, the owners of the namespace and the
permissions given to that namespace. NamespaceRegistry is uniquely identified
by its namespace.

Permissions are used to control read and/or write access to the namespace. It
includes a contract name that correlates to the name of a Sabre contract and
whether that contract is allowed to read and/or write to that namespace. The
permission is uniquely identified by the contract\_name. If the contract is
executed but does not have the needed permission to read or write to state,
the transaction is considered invalid.

```protobuf
  message NamespaceRegistry {
    message Permission {
      string contract_name = 1;
      bool read = 2;
      bool write = 3;
    }

    string namespace = 1;
    repeated string owners = 2;

    repeated Permission permissions = 3;
  }
```

When the same address is computed for different namespace registries, a
collision occurs; all colliding namespace registries are stored in at the
address in a NamespaceRegistryList

```protobuf
  message NamespaceRegistryList {
    repeated NamespaceRegistry registries = 1;
  }
```

### ContractRegistry

ContractRegistry keeps track of versions of the Sabre contract and the list of
owners. A ContractRegistry is uniquely identified by the name of its contract.

Versions represent the contract version and include the sha512 hash of the
contract and the public key of the creator. The hash can be used by a client to
verify this is the correct version of the contract that should be executed.

```protobuf
  message ContractRegistry {
    message Version {
      string version = 1;

      // used to verify a contract is same as the one the client intended to
      // invoke
      string contract_sha512 = 2;

      // for client information purposes only - the key that created this
      // contract on the chain
      string creator = 3;
    }

    string name = 1;
    repeated Version versions = 2;
    repeated string owners = 3;
  }
```

ContractRegistry whose addresses collide are stored in a ContractRegsitryList.

```protobuf
  message ContractRegistryList {
    repeated ContractRegistry registries = 1;
  }
```

### Contract

A Contract represents the Sabre smart contract. It is uniquely
identified by its name and version number. The contract also contains the
expected inputs and outputs used when executing the contract, the public
key of the creator and the compiled wasm code of the contract.

```protobuf
    message Contract {
      string name = 1;
      string version = 2;
      repeated string inputs = 3;
      repeated string outputs = 4;
      string creator = 5;
      bytes contract = 6;
    }
```

Contracts whose addresses collide are stored in a ContractList.

```protobuf
    message ContractList {
      repeated Contract contracts = 1;
    }
```

### Addressing

Sabre objects are stored under 3 namespaces:

  - ``00ec00``: Namespace for NamespaceRegistry
  - ``00ec01``: Namespace for ContractRegistry
  - ``00ec02``: Namespace for Contracts

The remaining 64 characters of the objects address is the following:
  - NamespaceRegistry: the first 64 characters of the hash of the first 6
    characters of the namespaces.
  - ContractRegistry: the first 64 characters of the hash of the name.
  - Contract: the first 64 characters of the hash of "name,version"

For example, the address for a contract with name "example" and version "1.0"
address would be:

```pycon
  >>> '00ec02' + get_hash("example,1.0")
  '00ec0248a8e00e3fbca83815668ec5eee730023e6eb61b03b54e8cae1729bf5a0bec64'
```


## Transaction Payload and Execution

Below, the different payload actions are defined along with the inputs and
outputs that are required in the transaction header.

### SabrePayload

A SabrePayload contains an action enum and the associated action payload. This
allows for the action payload to be dispatched to the appropriate logic.

Only the defined actions are available and only one action payload should be
defined in the SabrePayload.

```protobuf
  message SabrePayload {
    enum Action {
      ACTION_UNSET = 0;
      CREATE_CONTRACT = 1;
      DELETE_CONTRACT = 2;
      EXECUTE_CONTRACT = 3;
      CREATE_CONTRACT_REGISTRY = 4;
      DELETE_CONTRACT_REGISTRY = 5;
      UPDATE_CONTRACT_REGISTRY_OWNERS = 6;
      CREATE_NAMESPACE_REGISTRY = 7;
      DELETE_NAMESPACE_REGISTRY = 8;
      UPDATE_NAMESPACE_REGISTRY_OWNERS = 9;
      CREATE_NAMESPACE_REGISTRY_PERMISSION = 10;
      DELETE_NAMESPACE_REGISTRY_PERMISSION = 11;
    }

    Action action = 1;

    CreateContractAction create_contract = 2;
    DeleteContractAction delete_contract = 3;
    ExecuteContractAction execute_contract = 4;

    CreateContractRegistryAction create_contract_registry = 5;
    DeleteContractRegistryAction delete_contract_registry = 6;
    UpdateContractRegistryOwnersAction update_contract_registry_owners = 7;

    CreateNamespaceRegistryAction create_namespace_registry = 8;
    DeleteNamespaceRegistryAction delete_namespace_registry = 9;
    UpdateNamespaceRegistryOwnersAction update_namespace_registry_owners = 10;
    CreateNamespaceRegistryPermissionAction create_namespace_registry_permission = 11;
    DeleteNamespaceRegistryPermissionAction delete_namespace_registry_permission = 12;
  }
```

### CreateContractAction

Creates a contract and updates the associated contract registry.

```protobuf
  message CreateContractAction {
    string name = 1;
    string version = 2;
    repeated string inputs = 3;
    repeated string outputs = 4;
    bytes contract = 5;
  }
```

If a contract with the name and version already exists the transaction is
considered invalid.

The contract registry is fetched from state and the transaction signer is
checked against the owners. If the signer is not an owner, the transaction is
considered invalid.

If the contract registry for the contract name does not exist, the transaction
is invalid.

Both the new contract and the updated contract registry are set in state.

The inputs for CreateContractAction must include:

* the address for the new contract
* the address for the contract registry

The outputs for CreateContractAction must include:

* the address for the new contract
* the address for the contract registry

### DeleteContractAction

Delete a contract and remove its entry from the associated contract registry.

```protobuf
  message DeleteContractAction {
    string name = 1;
    string version = 2;
  }
```

If the contract does not already exist or does not have an entry in the
contract registry the transactions is invalid.

If the transaction signer is not an owner, they cannot delete the contract and
the transaction is invalid.

The contract is deleted and version entry is removed from the
contract entry.

The inputs for DeleteContractAction must include:

* the address for the contract
* the address for the contract registry

The outputs for DeleteContractAction must include:

* the address for the contract
* the address for the contract registry

### ExecuteContractAction

Execute the contract.

```protobuf
  message ExecuteContractAction {
    string name = 1;
    string version = 2;
    repeated string inputs = 3;
    repeated string outputs = 4;
    bytes payload = 5;
  }
```

The contract is fetched from state. If the contract does not exist, the
transaction is invalid.

The inputs and outputs are then checked against the namespace registry
associated with the first 6 characters of each input or output. If the input
or output is less then 6 characters the transaction is invalid. For every
input, the namespace registry must have a read permission for the contract and
for every output the namespace registry must have a write permission for the
contract. If either are missing or the namespace registry does not exist,
the transaction is invalid.

The contract is than loaded into the wasm interpreter and run against the
provided payload. A result is returned. If the result is ok the transaction
is valid and the contract data is stored in state. If the result is invalid or
another error occurs the transaction is invalid. If the result is an internal
error, an internal error is raised.

The inputs for ExecuteContractAction must include:

* the address for the contract
* the address for the contract registry
* any inputs that are required for executing the contract
* the addresses for every namespace registry required to check the provided
  contract inputs

The outputs for ExecuteContractAction must include:

* the address for the contract
* the address for the contract registry
* any outputs that are required for executing the contract
* the addresses for every namespace registry required to check the provided
  contract outputs

### CreateContractRegistryAction

Create a contract registry with no version.

```protobuf
  message CreateContractRegistryAction {
    string name = 1;
    repeated string owners = 2;
  }
```

If the contract registry for the provided contract name already exists, the
transaction is invalid.

Only those whose public keys are stored in ``sawtooth.swa.administrators`` are
allowed to create new contract registries. If the transaction signer is an
administrator, the new contract registry is set in state. Otherwise, the
transaction is invalid.

The new contract registry is created for the name and provided owners. The
owners should be a list of public keys of users that are allowed to add new
contract versions, delete old versions, and delete the registry.

The new contract registry is set in state.

The inputs for CreateContractRegistryAction must include:

* the address for the contract registry
* the settings address for ``sawtooth.swa.administrators``

The outputs for CreateContractRegistryAction must include:

* the address for the contract registry

### DeleteContractRegistryAction

Deletes a contract registry if there are no versions.

```protobuf
  message DeleteContractRegistryAction {
    string name = 1;
  }
```

If the contract registry does not exist, the transaction is invalid. If the
transaction signer is not an owner or does not have their public key in
``sawtooth.swa.administrators`` or the contract registry has any number of
versions, the transaction is invalid.

The contract registry is deleted.

The inputs for DeleteContractRegistryAction must include:

* the address for the contract registry
* the settings address for ``sawtooth.swa.administrators``


The outputs for DeleteContractRegistryAction must include:

* the address for the contract registry

### UpdateContractRegistryOwnersAction

Update the contract registry's owners list.

```protobuf
  message UpdateContractRegistryOwnersAction {
    string name = 1;
    repeated string owners = 2;
  }
```

If the contract registry does not exist or the transaction signer is not an
owner or does not have their public key in ``sawtooth.swa.administrators``, the
transaction is invalid.

The new owner list will replace the current owner list and the updated contract
registry is set in state.

The inputs for UpdateContractRegistryOwnersAction must include:

* the address for the contract registry
* the settings address for ``sawtooth.swa.administrators``


The outputs for UpdateContractRegistryOwnersAction must include:

* the address for the contract registry

### CreateNamespaceRegistryAction

Creates a namespace registry with no permissions.

```protobuf
  message CreateNamespaceRegistryAction {
    string namespace = 1;
    repeated string owners = 2;
  }
```

The namespace must be at least 6 characters long. If the namespace registry
already exists, the transaction is invalid.

Only those whose public keys are stored in ``sawtooth.swa.administrators`` are
allowed to create new namespace registries. If the transaction signer is an
administrator, the new namespace registry is set in state. Otherwise, the
transaction is invalid.

The inputs for CreateNamespaceRegistryAction must include:

* the address for the namespace registry
* the settings address for ``sawtooth.swa.administrators``

The outputs for CreateNamespaceRegistryAction must include:

* the address for the namespace registry

### DeleteNamespaceRegistryAction

Deletes a namespace registry if it does not contains any permissions.

```protobuf
  message DeleteNamespaceRegistryAction {
    string namespace = 1;
  }
```

If the namespace registry does not exist or contain permissions, the
transaction is invalid.

If the transaction signer is either an owner in the namespace registry or has
their public key in ``sawtooth.swa.administrators``, the namespace registry
is deleted. Otherwise, the transaction is invalid.

The inputs for DeleteNamespaceRegistryAction must include:

* the address for the namespace registry
* the settings address for ``sawtooth.swa.administrators``

The outputs for DeleteNamespaceRegistryAction must include:

* the address for the namespace registry

### UpdateNamespaceRegistryOwnersAction

Update the namespace registry's owners list.

```protobuf
  message UpdateNamespaceRegistryOwnersAction {
    string namespace = 1;
    repeated string owners = 2;
  }
```

If the namespace registry does not exist, the transaction is invalid.

If the transaction signer is either an owner in the namespace registry or has
their public key in ``sawtooth.swa.administrators``, the namespace registry's
owners are updated. Otherwise, the transaction is invalid.

The updated namespace registry is set in state.

The inputs for UpdateNamespaceRegistryOwnersAction must include:

* the address for the namespace registry
* the settings address for ``sawtooth.swa.administrators``

The outputs for UpdateNamespaceRegistryOwnersAction must include:

* the address for the namespace registry

### CreateNamespaceRegistryPermissionAction

Adds a permission entry into a namespace registry for the associated namespace.

```protobuf
  message CreateNamespaceRegistryPermissionAction {
    string namespace = 1;
    string contract_name = 2;
    bool read = 3;
    bool write = 4;
  }
```

If the namespace registry does not exist, the transaction is invalid.

If the transaction signer is either an owner in the namespace registry or has
their public key in ``sawtooth.swa.administrators``, a new permission is
added for the provided contract\_name. Otherwise, the transaction is invalid.

If there is already a permission for the contract\_name in the namespace
registry, the old permission is removed and replaced with the new
permission.

The updated namespace registry is set in state.

The inputs for CreateNamespaceRegistryPermissionAction must include:

* the address for the namespace registry
* the settings address for ``sawtooth.swa.administrators``

The outputs for CreateNamespaceRegistryPermissionAction must include:

* the address for the namespace registry

### DeleteNamespaceRegistryPermissionAction

Delete a permission entry in a namespace registry for the associated
namespace.

```protobuf
  message DeleteNamespaceRegistryPermissionAction {
    string namespace = 1;
    string contract_name = 2;
  }
```

If the namespace registry does not exist, the transaction is invalid. If the
transaction signer is either an owner in the namespace registry or has their
public key in ``sawtooth.swa.administrators``, the permission for the provided
contract name is removed. Otherwise, the transaction is invalid.

The inputs for DeleteNamespaceRegistryPermissionAction must include:

* the address for the namespace registry
* the settings address for ``sawtooth.swa.administrators``

The outputs for DeleteNamespaceRegistryPermissionAction must include:

* the address for the namespace registry

## Transaction Header

### Inputs and Outputs

The required inputs and outputs are defined for each action payload above.

### Dependencies

No dependencies.

### Family

- family_name: "sabre"
- family_version: "0.1"

# Sub-team Creation
[subteam]: #subteam

A Sawtooth Sabre sub-team shall be formed; this sub-team shall be responsible
for the following related to Sabre: shepharding RFCs, accepting or rejecting
RFCs, policy decisions (including when RFCs are required), and making decisions
on topics which do not require an RFC.

## Discussion

A #sawtooth-sabre channel shall be created in Hyperledger's RocketChat for the
purposes of discussion related to Sabre.

## Membership

- Shawn Amundson, team lead
- Andi Gunderson
- Ryan Banks
- James Mitchell

There are no strict rules for how to become a member of the Sabre sub-team, but
in general, the following areas of participation are important:

- Submit high-quality PRs to Sabre and/or RFCs concerning Sabre
- Participate in Sabre code reviews and/or Sabre RFC discussion
- Write and enhance Sabre documentation
- Actively support Sabre users on RocketChat

# Drawbacks
[drawbacks]: #drawbacks

Only a Sabre SDK for Rust will be initially implemented.

Though the goal is compatibility with the transaction processor API, it is not
always trivial to compile commonly used Rust dependencies into Wasm. This may
improve over time as Wasm popularity grows, or it may persist into the future.

The use of named contracts is substantially different than other smart contract
systems such as Ethereum, which instead use a hash of the contract as the
unique identifier. The design was chosen because it provides a friendlier
method of versioning smart contracts for permissioned networks. However, it
would not be a suitable approach for a public non-permissioned network; thus,
supporting such networks will require substantial design changes in the future.

# Rationale and alternatives
[alternatives]: #alternatives

The design presented is intended to maximize compatibility with the existing
Sawtooth transaction processor approach. It is intended to extend the full
capability of transaction processors into the realm of deployable on-chain
smart contracts. To achieve this, it was clear that a permissioning system
would be required to constrain deployments.

The only existing on-chain smart contract system for Sawtooth is Seth, which
implements Ethereum compatibility via a JSON-RPC interface and execution of
smart contracts in the EVM. Seth does not currently have access to all of the
features present in Sawtooth transaction processors, nor can Seth smart
contracts access the entire global state. Is is therefore impossible today, for
example, to write a Seth contract which interacts with the Intkey namespace.
It would be possible to extend Seth with a set of built-in smart contracts
which provided native access to Sawtooth features and/or non-Seth global state.
However, the inherent complexity, both from an implementation and a user
experience perspective, makes this solution less desirable. It would also
provide no natural transition path from Sawtooth transaction processors.

The versioning system in Sabre provides upgrade functionality in a more
traditional software deployment pattern than that used in the Ethereum
approach. In the Ethereum approach, smart contracts are immutible and
independent, which are good attributes for a public network; however, these
constraints require a smart contract developer to understand novel
Ethereum-specific patterns and perform significant pre-planning to ensure the
ability to smoothly upgrade to a newer smart contract. In Sabre, we have
selected an upgrade pattern suitable for permissioned networks which is
both simple and conceptually closer to traditional software upgrade patterns.

# Prior art
[prior-art]: #prior-art

Sawtooth currently relies on Transaction Processors that run in a separate
process. The Sabre API for the smart contracts will closely match that of the
current transaction processor API.

Sawtooth Sabre follows the intent of Sawtooth [Seth][seth], being able to run
smart contracts on Sawtooth, but removes the dependencies on an Ethereum
Virtual Machine.

This design borrows some ideas from [Ethereum][ethereum] smart contracts run on
the Ethereum Virtual Machine as well as [Parity][parity] and
[Parity Wasm][wasmi].

[seth]: https://github.com/hyperledger/sawtooth-seth
[ethereum]:https://www.ethereum.org/greeter
[parity]:https://github.com/paritytech/parity/tree/27c32d3629e4b70b63ee37736147318035f48792/ethcore/wasm
[wasmi]:https://github.com/paritytech/wasmi

# Unresolved questions
[unresolved]: #unresolved-questions

Any Turing-complete system that supports arbitrary code execution necessarily
is subject to the undecidability of the Halting Problem. Because Sawtooth
transaction processors do not afford any protection against the Halting
Problem, it is up to developers and reviewers to ensure transaction processing
halts for all input. Because the "traditional" deployment model for transaction
processors requires cross-network coordination, there is ample opportunity for
review by the entire network because all participants have to agree to install
the new processor.

The Sabre transaction processor supplements this traditional deployment model
with a more flexible model that allows new smart contracts to be deployed
without full consensus across the network. By delegating responsibility to
fewer individuals, there are fewer opportunities to confirm that processing
always halts. As a result, it is desired that some form of execution limiting
be applied to smart contracts run by the Sabre transaction processor.

This feature is outside the scope of the initial implementation and will be
addressed in a future RFC. Currently, instruction counting is being considered
as one possible solution to this problem, inspired by the concept of "gas" used
by the Ethereum Virtual Machine, however more work needs to be done before a
complete design is proposed.
