- Feature Name: supply_chain_property_references
- Start Date: 2018-03-14
- RFC PR:
- Sawtooth Issue:

# Summary
[summary]: #summary

Store PropertySchemas on-chain. These schema can then be referenced by name
when creating a new RecordType. This will copy the schema into the RecordType,
as well as save the name for clients to reference.

# Motivation
[motivation]: #motivation

This will allow multiple RecordTypes to share definitions of a Property. This
could be useful for any sort of measurement. Rather than defining "temperature"
every time you define a type of Record, it is defined once, and those
RecordTypes all share the concept. This will provide a level of consistency and
error protection, but more importantly allows for richer clients to be written
with less hard coded logic. If a client knows that Properties of the type
"temperature" are the same on every Record of every type, it can parse and
display the information in a consistent way.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `PropertySchema` protobuf message is now a little more strictly defined,
referring just to the shape and format of the data itself. Schemas will be
attached to a `PropertyDefinition`, which takes the place schemas used to,
describing the name and other qualities of a Property (like whether or not it
is required). The new more specialized schema will also be attached to a new
concept:  `PropertyType`. These are essentially just named containers for
schema stored in state. A `PropertyDefinition` would then be able to either
name its type, or provide its own a custom schema.

## Transactions

Before a `PropertyType` can be referenced, it must be created. Say we want to
make "temperature" a referenceable concept in our supply chain. We might create
that type like this in Python:

```python
SCPayload(
    action=SCPayload.Action.CREATE_PROPERTY_TYPE,
    timestamp=time.time(),
    create_property_type=CreatePropertyTypeAction(
        name='temperature',
        schema=PropertySchema(
            data_type=PropertySchema.DataType.INT)))
```

Now that we have temperature in state, we can create a RecordType with it. We'll make a
"lightbulb" type which will have both a temperature, and "watts", which is a schema
unique to lightbulbs.
(`SCPayload` omitted for brevity):

```python
CreateRecordTypeAction(
    name='lightbulb',
    properties=[
        PropertyDefinition(
            name='surface_temp',
            property_type='temperature',
            required=True),
        PropertyDefinition(
            name='watts',
            property_schema=PropertySchema(
                data_type=PropertySchema.DataType.INT),
            required=True)
    ]))
```

Upon the creation of this new RecordType, the schema from the "temperature"
type we created earlier will be copied to the RecordType. Any clients will know
this is Property is a "temperature", because it will retain the name under the
`property_type` field. So if a client wanted to do something like use a
thermometer graphic to display the temperature for every Record of every
RecordType, it would be able to make that determination.

Notice that since the "required" flag lives on the PropertyDefinition, not the
schema, it is possible for some RecordTypes to have a temperature which is
required, and others to have a temperature which is optional. The name of the
Property itself is also on a per Property basis. Here the temperature is called
"surface_temp". On other Records it might be called "heat",
"internal_temperature", or just "temperature".

### Errors

Attempting to use _both_ `property_type` and `property_schema` on the same
PropertyDefinition would result in an error:

```json
{
  "error": "InvalidTransaction: PropertyDefinitions cannot have both property_type and property_schema"
}
```

As would attempting to reference a PropertyType that did not exist:

```json
{
  "error": "InvalidTransaction: Named property_type could not be found: temmpurrater"
}
```

PropertyType names must be unique, so attempting create one with an existing
name would also be rejected:

```json
{
  "error": "InvalidTransaction: A PropertyType already exists with name: temperature"
}
```

## API Response

Once the "lightbulb" type is has been created successfully, creating a new
lightbulb would work identically to how it does currently. New PropertyValues
will have to match the schema, regardless of how it was defined. However, on
fetching a lightbulb from the REST API, clients will notice a new field:

```json
{
  "recordId": "5up4-br1t",
  "owner": "03fb457a09cc...",
  "custodian": "03fb457a09cc...",
  "final": false,
  "properties": [
    {
      "name": "surface_temp",
      "type": "INT",
      "propertyType": "temperature",
      "value": 55000000,
      "reporters": [ "03fb457a09cc..." ]
    },
    {
      "name": "watts",
      "type": "INT",
      "value": 80000000,
      "reporters": [ "03fb457a09cc..." ]
    },
  ],
  "updates": { ... },
  "proposals": []
}
```

If set, properties will now have a "propertyType" field with the string name of
the type. This name will be unique on the blockchain, and clients can use it to
create more specific displays for kinds of properties than just from data type
alone (still referred to here as simply "type").

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Schema Split

Building out this idea will mean splitting up the `PropertySchema` message into
two more specific messages:

```protobuf
message PropertyDefinition {
  string name = 1;
  string property_type = 2;
  PropertySchema property_schema = 3;

  bool required = 10;
}

message PropertySchema {
  enum DataType {
    BYTES = 0;
    STRING = 1;
    INT = 2;
    FLOAT = 3;
    LOCATION = 4;
  }

  DataType data_type = 1;
}
```

`PropertySchema` now describes only the data itself. Property specific
qualities, like its name or whether or not it is required, is described by the
`PropertyDefinition`. The definition is also where you would reference the
`PropertyType` if any.

## Transaction Payloads

The messages in `payload.proto` will need to be expanded to allow for a new
transaction to create property types:

```protobuf
message SCPayload {
  enum Action {
    CREATE_AGENT = 0;
    CREATE_RECORD = 1;
    FINALIZE_RECORD = 2;
    CREATE_RECORD_TYPE = 3;
    UPDATE_PROPERTIES = 4;
    CREATE_PROPOSAL = 5;
    ANSWER_PROPOSAL = 6;
    REVOKE_REPORTER = 7;
    CREATE_PROPERTY_TYPE = 8;
  }

  Action action = 1;
  uint64 timestamp = 2;

  CreateAgentAction create_agent = 3;
  CreateRecordAction create_record = 4;
  FinalizeRecordAction finalize_record = 5;
  CreateRecordTypeAction create_record_type = 6;
  UpdatePropertiesAction update_properties = 7;
  CreateProposalAction create_proposal = 8;
  AnswerProposalAction answer_proposal = 9;
  RevokeReporterAction revoke_reporter = 10;
  CreatePropertyTypeAction create_property_type = 11;
}

message CreatePropertyTypeAction {
  string name = 1;
  PropertySchema property_schema = 2;
}
```

## State

First, the `RecordType` message will change slightly to use
`PropertyDefinition` instead of `PropertySchema`:

```protobuf
message RecordType {
  string name = 1;

  repeated PropertyDefinition properties = 2;
}
```

Since property types will be stored in state, they would need a message as well
as a container. Like other Supply Chain resources, this container is to help
handle hash collisions.

```protobuf
message PropertyType {
  string name = 1;
  PropertySchema property_schema = 2;
}

message PropertyTypeContainer {
  repeated PropertyType entries = 1;
}
```

Like existing state entities, these containers would be stored under the
Sawtooth namespace (`3400de`), and then under their own single byte type
prefix. No existing type is using `00`, so that will become PropertyType's
prefix. The remaining 62 characters of the address would be the first 62
characters of a SHA-512 hash of the type's name.

```
3400de 00 f08bfeb8fd09b963f81f9dfe92ddfe1de9a52d1a4750891cbb3e04a38f42c4
  -> { name: 'temperature', ... }
```

# Drawbacks
[drawbacks]: #drawbacks

This design adds a fair amount of mental overhead. There is a lot of semantic
overlap between the ideas of "schema", "definition", and "type", which might
cause confusion. Not to mention PropertyType vs DataType vs RecordType. There
might be better terminology that could help solve the problem, but
fundamentally the differences between this structures is pretty narrow and
esoteric.

It's not exactly a lightweight solution. Since the schema is copied to the
RecordType there will be no additional state fetches after the RecordType is
created, so it shouldn't slow down the TP much, but it is yet another data
structure to keep track of.

There may not be the use case to justify the extra mental overhead. In order to
be useful a client must be tracking multiple resource types which share some
data schemas. At the very least, the universal client proposed elsewhere
_would_ use this.

It's also worth mentioning that this proposal will make some breaking protobuf
changes. It could probably be reworked a little bit to avoid breaking changes.
There could be no PropertyDefinition, with PropertySchema being used both in
RecordType and PropertyType. That idea was rejected because many of the fields
that are appropriate for use in a RecordType are not appropriate for use in a
PropertyType (required for example). This implementation would be more
confusing and error prone.

# Rationale and alternatives
[alternatives]: #alternatives

A way to get similar utility with a simpler implementation, would be to just
have the `property_type` name, with no reference to an additional structure.
Clients could then just parse the name. This might be fine for a simple integer
like the temperature example above, but if some of the more complex expanded
data types are implemented as detailed in a separate RFC, this will quickly
become unwieldy. More importantly, without this additional structure there is
no way to guarantee that the structure of the named type is consistent across
different RecordTypes.

If not implemented, clients would be stuck parsing only the data_type, or the
specific name of the specific Property. While this might be fine for some uses,
it has already led to some hacky workarounds in FishNet and AssetTrack, and
would make it impossible to do any sophisticated data parsing in the proposed
new universal UI.

# Prior art
[prior-art]: #prior-art

No prior art.

# Unresolved questions
[unresolved]: #unresolved-questions

No particular unresolved questions. Some validation issues will no doubt be
discovered and resolved during implementation.
