- Feature Name: read_by_prefix
- Start Date: 2018-12-19
- RFC PR:
- Sawtooth Issue:

# Summary
[summary]: #summary

This RFC proposes an additional API to transaction context,
context.listAddresses(address_prefix), which will get address prefix as input
and return list of all existing addresses that starts with the prefix.

This capability exists in the Sawtooth REST API, but is not currently 
available via the transaction processor protocol or SDKs (for reasons covered 
in 'Drawbacks' below).

# Motivation
[motivation]: #motivation

The motivation came when looking for a way to reduce the number of transaction
processor calls to modify large addresses when need to work on a big chunk of 
addresses by making better use optimization that comes from using the radix 
tree. 

Since addresses are stored in a radix tree, reading many addresses by 
one prefix is much faster than reading the same amount of addresses one by one.
Further more, read by prefix doesn't require transaction processor to know 
which addresses exist, removing the need to store list of existing addresses 
in a separate address.

Having this capability allows a better and smarter address mapping and a 
better transaction processor logic.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Get state by prefix is a feature available for reading via sawtooth REST API.
This RFC adds method to expose this capability of reading address prefix 
to transaction processor.

The motivation came when looking for a way to improve transaction processor
performance when deleting chunk of addresses.
For example when need to delete all addresses owned by some user with some
extra conditions based on transaction processor logic.
The problem is that the transaction processor doesn't know which addresses
exists in the merkle tree.
Old approach will be to store a special address with details on all existing
addresses belong to each user. This is not optimal since although the need to 
delete is rare, this special address will need to be updated for almost any
transaction. This address can get huge very fast, having a big performance 
impact for almost every transaction. Bucketing to several addresses can help
improve performance but not significantly.

The approach requested by this RFC is to take advantage of the 'radix' part
of the context radix merkle tree. When storing data use common prefix based
on transaction processor logic, then when need to delete chunk of address, 
get the list of existing addresses under a prefix and request delete for all 
addresses in the list. The returned list of addresses might be huge but with
this approach, only in the rare cases of delete, the transaction processor will
need to handle this huge amount of addresses. 
Can improve performance since need only list of existing addresses under a prefix
and not the address data, keeping the returned value in a relative normal size.

No change is recommended to the existing methods, so the implementation of this 
RFC should not cause any breakage for pre-existing transaction processors.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Add new API to context: list_addresses that returns list of existing addresses
under given address prefix. 
This API should be similar to regular get address data API but can get as input
prefix of address and not just full 35 bytes address and return a list of 
addresses without their data. Since the ability to read by prefix is already
supported from the REST API, this is not a new concept for the validator.
There is still a need to add a path from the transaction processor to the 
validator for this new API and to support this API for each SDK.

In the suggested implementation there are two proto changes:

Addition to state_context.proto:
	
	// Import paging proto as used by REST-API read by prefix request
	// The number of results per page defaults to 100 and maxes out at 1000
	import "client_list_control.proto";
	
	// A request from a handler/tp for list of addresses with valid data
	message TpStateAddressesListRequest {
		// The context id that references a context in the contextmanager
		string context_id = 1;
		repeated string addresses = 2;
		ClientPagingControls paging = 3;
	}

	// A response from the contextmanager/validator with a series of State 
	// entries having valid data
	message TpStateAddressesListResponse {
		enum Status {
			STATUS_UNSET = 0;
			OK = 1;
			AUTHORIZATION_ERROR = 2;
			NO_RESOURCE = 3;
        		INVALID_PAGING = 4;
		}

		repeated string addresses = 1;
		Status status = 2;
		ClientPagingResponse paging = 3;    
	}
	
Addition to validator.proto:

	// State list request to get all addresses having non empty data
	TP_STATE_LIST_REQUEST = 17;
	// State list response to get all addresses having non empty data
	TP_STATE_LIST_RESPONSE = 18;
	

Example of implementation (without paging) can be found here: 
Sawtooth-core + Python SDK changes: 
https://github.com/arsulegai/sawtooth-core/tree/jira_1375 
(commits: 4225dc7 and 2c3a223)
Sawtooth-sdk-cxx changes: 
https://github.com/arsulegai/sawtooth-sdk-cxx/tree/jira_1375 (commit: 23697e8)	
	
# Drawbacks
[drawbacks]: #drawbacks

1. This will need to be implemented in every sawtooth SDK.
2. Once implemented it will be hard/difficult to deprecate because of our 
   commitment to backward compatibility.
3. Will slightly increase the complexity of the validator.
4. Since list of addresses can be very large this could cause a large I/O 
   and CPU impact on validator. 
   This is solved by making the new API return only address list without 
   address data, and by adding paging mechanism like it is done in REST API.

# Rationale and alternatives
[alternatives]: #alternatives

An alternative could be in the transaction processor logic, store an address
that contains list of existing addresses. When need to manipulate multiple 
addresses transaction processor will first read the special address, then 
read/write/delete each address in the list.
This has a performance downside that each time transaction processor needs to 
write a new address, it will need to modify the special address containing 
list of all addresses. This address could be huge and it needs to be changed
for almost every transaction, when the need to actually access all the
addresses in the list is very be rare.

# Prior art
[prior-art]: #prior-art

Read state by address prefix is available at REST API, and is
one of the key capability that comes from using the radix tree.

The relevant protocol defined in client_state.proto is:

	message ClientStateListRequest {
	    string state_root = 1;
	    string address = 3;
	    ClientPagingControls paging = 4;
	    repeated ClientSortControls sorting = 5;
	}

	message ClientStateListResponse {
	    enum Status {
		STATUS_UNSET = 0;
		OK = 1;
		INTERNAL_ERROR = 2;
		NOT_READY = 3;
		NO_ROOT = 4;
		NO_RESOURCE = 5;
		INVALID_PAGING = 6;
		INVALID_SORT = 7;
		INVALID_ADDRESS = 8;
		INVALID_ROOT = 9;
	    }

	    // An entry in the State
	    message Entry {
		string address = 1;
		bytes data = 2;
	    }

	    Status status = 1;
	    repeated Entry entries = 2;
	    string state_root = 3;
	    ClientPagingResponse paging = 4;
	}


# Unresolved questions
[unresolved]: #unresolved-questions


