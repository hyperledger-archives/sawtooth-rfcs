- Feature Name: read_by_prefix
- Start Date: 2018-12-19
- RFC PR:
- Sawtooth Issue:

# Summary
[summary]: #summary

This RFC proposes an additional API to transaction context,
context.listAddresses(address_prefix), which will get address prefix as input
and return list of all existing addresses that starts with the prefix.

This capability exists when reading state from rest-api but is missing when 
transaction processor reads state.

# Motivation
[motivation]: #motivation

The motivation came when looking for a way to improve processing transaction 
logic by making better use optimization that comes from using the radix tree. 

Since addresses are stored in a radix tree, reading many addresses by 
one prefix is much faster than reading the same amount of addresses one by one.
Further more, read by prefix doesn't require transaction processor to know 
which addresses exist, removing the need to store list of existing addresses 
in a separate address.

Having this capability allows a better and smarter address mapping this a 
better transaction processor logic.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Get state by prefix is a feature available for reading via sawtooth rest-api.
This RFC adds method to expose this capability of reading address prefix 
to transaction processor.

No change is recommended to the existing methods, so the implementation of this 
RFC should not cause any breakage for pre-existing transaction processors.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Example of implementation can be found here: 
Sawtooth-core + Python SDK changes: 
https://github.com/arsulegai/sawtooth-core/tree/jira_1375 
(commits: 4225dc7 and 2c3a223)

Sawtooth-sdk-cxx changes: 
https://github.com/arsulegai/sawtooth-sdk-cxx/tree/jira_1375 (commit: 23697e8)

In the suggested implementation there are two proto changes:

Addition to state_context.proto:


	// A request from a handler/tp for list of addresses with valid data
	message TpStateAddressesListRequest {
		// The context id that references a context in the contextmanager
		string context_id = 1;
		repeated string addresses = 2;
	}

	// A response from the contextmanager/validator with a series of State 
	// entries having valid data
	message TpStateAddressesListResponse {
		enum Status {
			STATUS_UNSET = 0;
			OK = 1;
			AUTHORIZATION_ERROR = 2;
		}

		repeated string addresses = 1;
		Status status = 2;
	}
	
Addition to validator.proto:

	// State list request to get all addresses having non empty data
	TP_STATE_LIST_REQUEST = 17;
	// State list response to get all addresses having non empty data
	TP_STATE_LIST_RESPONSE = 18;
	
	
# Drawbacks
[drawbacks]: #drawbacks

This will need to be implemented in every sawtooth SDK and will increase 
the complexity of the validator slightly.

# Rationale and alternatives
[alternatives]: #alternatives

An alternative could be in the transaction processor logic, store an address
that contains list of existing addresses. When need to manipulate multiple 
addresses transaction processor will first read the special address, then 
read/write/delete each address in the list.
This has a performance downside that each time transaction processor needs to 
write a new address, it will need to modify the special address containing 
list of all addresses. This address could be huge, when the need to actually 
access all the addresses in the list can be rare.

# Prior art
[prior-art]: #prior-art

Read state by address prefix is available from sawtooth-rest-api, and is
one of the key capability that comes from using the radix tree.


# Unresolved questions
[unresolved]: #unresolved-questions
