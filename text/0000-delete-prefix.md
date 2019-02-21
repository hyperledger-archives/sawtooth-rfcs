- Feature Name: delete_by_prefix
- Start Date: 2019-2-20
- RFC PR:
- Sawtooth Issue:

# Summary
[summary]: #summary

This RFC proposes an additional API to transaction context,
context.deletePrefix(address_prefix) which will get an address prefix as input
parameter and delete all the addresses in the prefix subtree.

# Motivation
[motivation]: #motivation

The motivation came when looking for an efficient way to delete chunk of 
addresses, for example all addresses that are allocated for a certain user.
If each member has some unique address prefix, deleting all the addresses 
belong to a member can become more efficient with delete by prefix.

Since addresses are stored in a radix tree, deleting many addresses by one 
prefix is much faster than deleting the same amount of addresses one by one.
Furthermore, delete by prefix doesn't require transaction processor to know 
which addresses exist. This will remove the need to store huge list of 
existing addresses in a special address that will need to be modified by every
transaction that writes a new address.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A context API to delete address already exist for transaction processors.
The existing delete API accepts only full 35 bytes addresses.
This RFC exposes to transaction processor a new capability of deleting by 
address prefix. When delete by prefix is called, all addresses under the 
prefix subtree will be deleted.

Main usage for this feature is the need to support some 'clean transaction'
that will remove all addresses with some communality, for example all 
addresses that belong to some member.
The proposed solution is to have address mapping logic that allocates some
unique pre-defined address prefix per member. The transaction processor logic
will know to calculate the address suffix based on transaction payload data
and execute any transaction.
In order to execute the clean transaction, the transaction processor will
call context.deletePrefix with the member prefix.

The existing solution is to keep track of all used addresses, this will
require read-modify-write of a special address for every new address that is
used. This solution could work but will slow down executing of every regular 
transactions while the clean transaction is rare.

No change is recommended to the existing methods, so the implementation of this
RFC should not cause any breakage for pre-existing transaction processors.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Add new API to context: deletePrefix that delete a subtree of addresses 
starting from a given address prefix. 
This API should be similar to regular delete address API but can get as input
prefix of address and not just full 35 bytes address.
Calling this API with full address should behave exactly like calling existing
delete API.

In the suggested implementation there are two proto changes:

Addition to state_context.proto:
	
	// A request from the handler/tp to delete state prefix entries at an 
	// collection of addresses prefix subtree
	message TpStateDeletePrefixRequest {
		string context_id = 1;
		repeated string addresses_prefix = 2;
	}

	// A response form the contextmanager/validator with the addresses that 
	// were deleted
	message TpStateDeleteResponse {
		enum Status {
			STATUS_UNSET = 0;
			OK = 1;
			AUTHORIZATION_ERROR = 2;
		}

		repeated string addresses = 1;
		Status status = 2;
	}

Addition to validator.proto:

	// State delete prefix request from the transaction processor to the 
	// validator/context_manager
	TP_STATE_DELETE_PREFIX_REQUEST = 17;
	// State delete prefix response from the validator/context_manager to the
	// transaction processor
	TP_STATE_DELETE_PREFIX_RESPONSE = 18;
	
	
# Drawbacks
[drawbacks]: #drawbacks

1. This will need to be implemented in every sawtooth SDK.
2. Once implemented it will be difficult to deprecate because of our 
   commitment to backward compatibility.
3. This requires transaction processor developers to understand the risk of
   calling delete by prefix and accidently deleting unintended addresses.
   Delete by prefix should be used only when developer is sure prefix is 
   unique like when having a pre-defined prefix and not in case there could
   be hash collisions for the prefix.

# Rationale and alternatives
[alternatives]: #alternatives

An alternative could be in the transaction processor logic, store an address
that contains list of existing addresses. When need to delete multiple 
addresses transaction processor will first read the special address, then 
delete each address in the list.
This has a performance downside that each time transaction processor needs to 
write a new address, it will need to modify the special address containing 
list of all addresses. This address could be huge and it needs to be changed
for almost every transaction, while the need to delete all the addresses in 
the list is very rare.

# Prior art
[prior-art]: #prior-art

Access address by prefix is a key capability of radix tree.
This RFC intends to take advantage of sawtooth design to use a radix tree for
storing the global state.

# Unresolved questions
[unresolved]: #unresolved-questions

Implementation of the validator changes for the new context API are not
prescribed in this RFC.