- Feature Name: supply_chain_record_transformation
- Start Date: 2018-04-12
- RFC PR:
- Sawtooth Issue:

# Summary
[summary]: #summary

Allow one or more assets tracked in Sawtooth Supply Chain to be transformed
into one or more other assets, using a new _Transform_ action. A Transform is
effectively a batching together of multiple CreateRecord, UpdateProperties, and
FinalizeRecord actions into a single transaction, with some additional
metadata. These Transforms will then be stored in state, greatly enriching the
historical data available for each Record.

# Motivation
[motivation]: #motivation

In any real world supply chain, assets will be transformed many times between
the initial point of production and finally reaching a consumer. Whether
simple, like taking raw coffee beans and roasting them, or complex, like
constructing a smart phone out of dozens of smaller parts, any robust supply
chain solution will need to account for these transformations, and make them
available for future queries. This is particularly critical in use cases like
food safety, where accurately tracking the lineage of a contaminated item might
be literally life or death.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This proposal adds the concept of a Transform, a new entity in state which
records the details of an event where one or more Records were converted into
one or more other Records. Transforms are created through a transaction which
will simultaneously modify existing Records, create new Records, create a
Transform in state, and link each of those Records to that Transform.

In addition to info directly related to Records, Transforms support robust
metadata. Besides timestamps and the key of the signing Agent, users can
specify arbitrary data using essentially the same sort of list of
PropertyValues that Records use, but in a slightly different from: the
DataValue. DataValues must work on an ad hoc basis without a schema, so they
combine many of the keys from PropertySchema and PropertyValue, and the do not
support the ENUM data type.

Finally, in addition to Record based inputs and outputs, Transforms will
support "untracked" inputs/outputs. These are flexible containers to store
information specific to assets not tracked on the blockchain.

## The Transform Transaction

```protobuf
message TransformRecordsAction {
  message RecordUpdate {
    string record_id = 1;
    repeated PropertyValue properties = 2;
    bool final = 3;
    string record_type = 4;
  }

  message UntrackedComponent {
    string label = 1;
    string description = 2;
    repeated DataValue metadata = 3;
  }

  string transform_id = 1;
  repeated DataValue metadata = 2;

  repeated RecordUpdate inputs = 10;
  repeated RecordUpdate outputs = 11;

  repeated UntrackedComponent untracked_inputs = 12;
  repeated UntrackedComponent untracked_outputs = 13;
}
```

This is the proposed protobuf message for the new TransformRecordsAction.
Instantiating one and sending it to the validator in a payload will cause a
number of changes in blockchain state. For example, if the following Python
payload were submitted:

```python
action = TransformRecordsAction(
    transform_id=uuid.uuid4(),
    metadata=[
        DataValue(
            name='facility',
            data_type=DataValue.DataType.STRING,
            string_value='Sawtooth Processing Plant')],
    inputs=[
        TransformRecordsAction.RecordUpdate(
            record_id='th1s-1s-s0me-f1sh',
            final=true)],
    outputs=[
        TransformRecordsAction.RecordUpdate(
            record_id='t4sty-f1sh-st1cks',
            properties=[
                PropertyValue(
                    name='weight',
                    data_type=PropertySchema.DataType.NUMBER,
                    number_value=5000000)],
            record_type='fish-sticks')],
    untracked_inputs=[
        TransformRecordsAction.UntrackedComponent(
            label='breading'
            description='Artisanal breading from our local farmers',
            metadata=[
                DataValue(
                    name='volume',
                    data_type=DataValue.DataType.NUMBER,
                    number_value=10000000,
                    unit='liters',
                    number_exponent=-3)])],
    untracked_outputs=[
        TransformRecordsAction.UntrackedComponent(
            label='fish slurry',
            description='waste product')])

payload = SCPayload(
    action='TRANSFORM_RECORDS',
    timestamp=time.time(),
    transform_records=action)

payload.SerializeToString()
```

The above payload would create the following changes in state:

- The Record `th1s-1s-s0me-f1sh` is _updated_
    * It becomes `final`
- The Record `t4sty-f1sh-st1cks` is _created_
    * It has a record type of `fish-sticks`
    * It is created with a `weight` of `5000000`
    * Its owner/custodian is the signer of the Transform action
- A Transform is _created_
    * It has a client-specified UUID for reference
    * It has a client-specified timestamp
    * It stores the signer's public key
    * It has metadata with a `facility` of `Sawtooth Processing Plant`
    * It records that a `volume` of 10000 liters of `breading` was consumed
    * It notes that `fish slurry` was produced as a byproduct

## Fetching Transform Data

Full Transform entities will be fetchable from the Supply Chain Server using a
new `/transforms` endpoint. This will return JSON data that closely matches the
protobuf stored in state:

```json
{
  "transformId": "un1que-tr4nsf0rm-1d",
  "agentKey": "03fb457a09cc...",
  "timestamp": 1520880613,
  "metadata": {
    "facility": "Sawtooth Processing Plant"
  },
  "inputs": [
    {
      "recordId": "th1s-1s-s0me-f1sh",
      "final": true
    }
  ],
  "outputs": [
    {
      "recordId": "t4sty-f1sh-st1cks",
      "properties": {
        "weight": 5000
      }
    }
  ],
  "untrackedInputs": [
    {
      "label": "breading",
      "description": "Artisanal breading from our local farmers",
      "metadata": {
        "volume": 10000
      }
    }
  ],
  "untrackedOutputs": [
    {
      "label": "fish slurry",
      "description": "waste product"
    }
  ]
}
```

Note that a `timestamp` and `agentKey` have been pulled from the
payload/transaction, and that `record_type` has been dropped, as it is not
needed after the fish sticks are created. Additionally, the lists of DataValues
have been transformed into JSON objects, similarly to how the REST API handles
other resources.

In addition to fetching the full Transform entities, additional data will be
added to any Records fetched. Specifically, they will gain a history of
Transforms inherited from any parent Records, and any property updates
triggered by a Transform will include its id. The response JSON might look like
this:

```json
{
  "recordId": "t4sty-f1sh-st1cks",
  "owner": "03fb457a09cc...",
  "custodian": "03fb457a09cc...",
  "final": false,
  "properties": [
    {
      "name": "weight",
      "type": "INT",
      "value": 5000,
      "reporters": [ "03fb457a09cc..." ]
    }
  ],
  "updates": {
    "owners": [ ... ],
    "custodians": [ ... ],
    "transforms": [
      {
        "transformId": "un1que-tr4nsf0rm-1d",
        "timestamp": 1520880613
      }
    ]
    "properties": {
      "weight": [
        {
          "timestamp": 1520880613,
          "value": 5000,
          "transformId": "un1que-tr4nsf0rm-1d"
        }
      ]
    }
  },
  "proposals": []
}
```


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This proposal includes a number of new concepts, entities, and actions, as well
as changes to some existing resources and logic. These changes should be as
minimally disruptive as possible, but will inevitably be breaking.

## The Protobufs

### Transforms

```protobuf
message Transform {
  message RecordUpdate {
    string record_id = 1;
    repeated PropertyValue properties = 2;
    bool final = 3;
  }

  message UntrackedComponent {
    string label = 1;
    string description = 2;
    repeated DataValue metadata = 3;
  }

  string transform_id = 1;
  string agent_key = 2;
  uint64 timestamp = 3;
  repeated DataValue metadata = 4;

  repeated RecordUpdate inputs = 10;
  repeated RecordUpdate outputs = 11;

  repeated UntrackedComponent untracked_inputs = 12;
  repeated UntrackedComponent untracked_outputs = 13;
}
```

```protobuf
message TransformContainer {
  repeated Transform entries = 1;
}
```

This is very similar to the TransformRecordsAction detailed earlier, though it
gains an `agent_key` and `timestamp`, and loses `record_type` from its
RecordUpdate. Like other state entities, it will be stored within a container
protobuf to handle hash collisions.

### DataValue

```protobuf
message DataValue {
  enum DataType {
    TYPE_UNSET = 0;
    BYTES = 1;
    BOOLEAN = 2;
    NUMBER = 3;
    STRING = 4;
    STRUCT = 6;
    LOCATION = 7;
  }

  string name = 1;
  DataType data_type = 2;

  bytes bytes_value = 11;
  bool boolean_value = 12;
  sint64 number_value = 13;
  string string_value = 14;
  repeated DataValue struct_values = 16;
  Location location_value = 17;

  sint32 number_exponent = 20;
  string unit = 21;
}
```

A new protobuf message will be added combining some of the keys from
PropertyValue and PropertySchema: the DataValue. The purpose of this message is
to contain arbitrary data which has no schema defined ahead of time, so it will
need to be able to use some schema-type fields on an ad hoc basis, specifically
`number_exponent` and `unit`. Additionally, there is no support for ENUMs, as
those only add value when they can be updated.


### Records

```protobuf
message Record {
  message Association {
    string resource_id = 1;
    uint64 timestamp = 2;
  }

  string record_id = 1;
  string record_type = 2;
  repeated Association owners = 3;
  repeated Association custodians = 4;
  bool final = 5;
  repeated Association transforms = 6;
}
```

Records gain a new repeated field: `transforms`. This is simply a timestamped
list of ids for every Transform that modified this Record, or other Records up
the input chain. When a Transform occurs, the `transforms` of any `inputs`
would be copied to every new Record created in `outputs`. So for example, there
is a Record A, and through Transforms we also create Records B, C, D, E like
this:

```
      B -> D -> E
     /
A ->
     \
      C
```

The `transforms` of Record E would include the transforms of A, B, and D, but
not C. In addition, if D had been transformed three times, and E was created
with the first such Transform, E would only have a record of D's first
Transform, not the subsequent two. Finally, transforms would be filtered for
uniqueness, so in the case that two inputs shared a transform in their history,
the resulting output would only have one copy of it.

In addition to the new field, the AssociatedAgent message has been renamed
Association, as it now is used for both Agents and Transforms.

### Property Pages

```protobuf
message PropertyPage {
  message ReportedValue {
    uint32 reporter_index = 1;
    uint64 timestamp = 2;
    string transform_id = 3;

    bytes bytes_value = 11;
    string string_value = 12;
    sint64 int_value = 13;
    float float_value = 14;
    Location location_value = 15;
  }

  string name = 1;
  string record_id = 2;
  repeated ReportedValue reported_values = 3;
}
```

PropertyPages gain a `transform_id` field, to link any Property update to the
Transform update that caused it. Note that this is an optional field, and for
most updates (such as those submitted by reporters), it will not be used.

## Transaction Validation

A Transform action is considerably more complex than any existing transaction
payloads. It will need to run a considerable number of validation rules.
However, as it is essentially a composition of multiple CreateRecord,
UpdateRecord, and FinalizeRecord actions, so there is an opportunity to reuse that
logic. Aside from saving development time, it is important to keep the API
consistent by mapping the behavior of those three actions directly to the
corresponding parts of the Transform API. One validation rule unique to
Transforms: the `transform_id` must be unique.

Note that a consequence of mirroring the permissioning of the UpdateProperties
action is that in order to update a Property with a Transform, the signer must
be authorized a Reporter.

One existing validation rule must be changed in order to accommodate
Transforms, the permissioning around FinalizeRecord actions. Currently it is
restricted to signers who are _both_ the owner and custodian of a Record. This
made sense when Records were whole and intact all the way to the consumer, but
now they might be broken down and converted at any point in the supply chain.
The new validation rule will be that a Record may be finalized by _either_ the
owner or the custodian.

### TransformTypes

One aspect of Transforms not addressed so far is giving the Transforms types.
Similar to RecordTypes, these would be entities stored on state that provide
extra validation rules that Transforms could inherit. For example, a
TransformType might specify that when creating fish sticks, only `fish` may be
the input, and only `fish-sticks` may be the output. A Transform would then
reference this TransformType by name, and the transaction processor would
ensure it matched the specified rules.

Ultimately TransformTypes are outside of the scope of this RFC, but it has been
designed with them in mind for possible future implementation. Such an
expansion should be fairly straightforward and non-breaking. Simply add a new
`transform_type` field to Transforms, which will reference a new TransformType
entity in state, which specifies some extra validation rules for the processor
to run when validating the transaction.

## Addressing

Transforms will be stored under the supply chain namespace (`3400de`), followed
by their own resource prefix (likely `01` or `02`, depending on what other RFCs
are incorporated). In order to leave room for future TransactionTypes, the next
eight character should be a hash of the name of the TransactionType. For
Transforms without a type, this would be `0000000`. The remaining 54 characters
of the address will be the first 54 characters of a SHA-512 hash of the
`transform_id`. For example:

```
3400de 02 00000000 9842b36d6b07876d262676130f919de68c5cd4af2d5acdb5360a72
  -> { transform_id: 'un1que-tr4nsf0rm-1d' ... }
```

# Drawbacks
[drawbacks]: #drawbacks

This design includes a number of breaking changes. However, as Supply Chain
matures from a simple demo to a robust platform, it cannot realistically be
expected to maintain existing APIs.

The TransformRecords action is fairly complex, and has a lot of moving parts.
Manually creating one would be daunting and error-prone. This is probably a
problem to fix with good client-side tools, not a simpler transaction payload,
but any functionally equivalent yet simpler designs would certainly be
considered.

If Transforms turn out to be quite common, happening many thousands of times
between production and consumption, storing their full ids on each RecordUpdate
might become a storage concern, as would storing each id on the Record itself.
If Transforms are actually that common, it might be preferable to store an
index in PropertyUpdates (similar to Reporters now), and add a new
TransformPage entity would be needed, mirroring the use of PropertyPages.

# Rationale and alternatives
[alternatives]: #alternatives

The guiding principle for this RFC was flexibility. Any Transform design must
be applicable in a number of wildly different use cases. In the real world,
some ingredients in a process might be tracked as Records, but others may not
be. The quantities and yields might be known or unknown. What information is
useful, or even available, might vary dramatically as well. Perhaps the exact
machine a process happened in could be recorded, perhaps just the facility.
Maybe the location doesn't matter at all. This design can accommodate all of
these use cases, and many more not yet conceived of.

# Prior art
[prior-art]: #prior-art

This design is built on previous Supply Chain concepts, as well as a number of
discussions about real world supply chain needs, but there is no specific prior
art.

# Unresolved questions
[unresolved]: #unresolved-questions

One unresolved question is how to handle situations where the leftovers from
one process are used in the next batch. For example, Records A, B, and C might
be poured into a vat, and Record X comes out. But there is also a little left
in the vat, and for the next batch, Records D, E, F are poured on top, and Y
comes out. If you are interested in something like food safety, and it turns
out that B was contaminated, then you will need to recall both X and Y, and
probably Z too when that comes down the line. This design will probably get
fairly unwieldy if tracked Records are reused in this way, though properly
handling this use case might require a more fundamental Supply Chain redesign.

Another unanswered question is how to handle scenarios where there is a time-
lag in acquiring the information for a transform. What happens if 90% of the
data is ready to go at transform time, but 10% needs to be calculated after the
fact? An UpdateTransform action could perhaps be added, or perhaps Transaforms
could become multi-signature transactions that are implemented in stages. Any
solution is likely to add a not insignificant complexity, and should be
carefully considered for its trade offs.
