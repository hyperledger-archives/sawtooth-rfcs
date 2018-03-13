- Feature Name: supply_chain_expand_data_types
- Start Date: 2018-03-08
- RFC PR:
- Sawtooth Issue:

# Summary
[summary]: #summary

The available "data types" used to describe Properties in Sawtooth Supply Chain
should be expanded to include Booleans, Enums and Structs. In addition, Floats
should be removed and replaced by giving integers an exponent property that
formalize the practice of representing decimals as whole numbers of fractions
(i.e. thousandths or millionths). This new more versatile integer property
would be renamed "Number".

# Motivation
[motivation]: #motivation

The current selection of data types available to Sawtooth Supply Chain
developers is incomplete for most use cases, and leads to "hacks" like storing
complex values as JSON strings, or using an integer to represent a boolean
value. The new data types will not only clarify intent for other developers,
but make it possible to write richer clients able to sensibly parse and display
more complex data.

In addition, floating point calculations introduce the potential for non-
determinism into the transaction processor. To avoid this hazard, some
developers have already been using integers as fractional units of measure.
This should be standardized, clarifying the semantic meaning for clients, and
used to replace floats entirely, preventing developers from unwittingly
introducing bugs into their applications.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Booleans

Previously, Supply Chain had no explicit boolean type as there was little need
and from a technical perspective there is no reason not to just use `BYTES` or
`INT`. The problem of course is in making intent of the original variable
clear, especially to clients that may consume one of these faux booleans. This
is easily fixed by adding `BOOLEAN` as an option to Property schema.

### Examples

The transaction payload to create a simple lightbulb RecordType in Python,
with an `isOn` Property:

```python
SCPayload(
    action='CREAT_RECORD_TYPE',
    timestamp=time.time(),
    create_record_type=CreateRecordTypeAction(
        name='lightbulb',
        properties=[
            PropertySchema(
                name='isOn',
                data_type=PropertySchema.DataType.BOOLEAN,
                required=True)
        ]))
```

Creating a lightbulb Record (SCPayload wrapper omitted for brevity):

```python
CreateRecordAction(
    record_id='5up4-br1t',
    record_type='lightbulb',
    properties=[
        PropertyValue(
            name='isOn',
            data_type=PropertySchema.DataType.BOOLEAN,
            boolean_value=False)
    ])
```

Updating an existing lightbulb:

```python
UpdatePropertiesAction(
    record_id='5up4-br1t',
    properties=[
        PropertyValue(
            name='isOn',
            data_type=PropertySchema.DataType.BOOLEAN,
            boolean_value=True)
    ])
```

The JSON received when requesting this lightbulb from the server:

```json
{
  "recordId": "5up4-br1t",
  "owner": "03fb457a09cc...",
  "custodian": "03fb457a09cc...",
  "final": false,
  "properties": [
    {
      "name": "isOn",
      "type": "BOOLEAN",
      "value": true,
      "reporters": [ "03fb457a09cc..." ]
    }
  ],
  "updates": {
    "owners": [ ... ],
    "custodians": [ ... ],
    "properties": {
      "isOn": [
        {
          "timestamp": 1520880613,
          "value": false
        },
        {
          "timestamp": 1520880674,
          "value": true
        }
      ]
    }
  },
  "proposals": []
}
```

## Enums

`ENUM` is a new data type encompassing Properties for which there is a limited
set of possible values. Their schema would be set with a list of strings names
describing a possible state of the Enum. These names would then be used to set
the Enum value. There are two reasons to use an Enum over just a regular string
value. First, the value can be stored efficiently as an integer index. Second,
the transaction processor can provide a level of error checking, rejecting any
values that do not match a valid name.

### Examples

Creating a lightbulb RecordType in Python with a `color` Property that can be
"white", "red", "green", "blue" or "blacklight":

```python
CreateRecordTypeAction(
    name='lightbulb',
    properties=[
        PropertySchema(
            name='color',
            data_type=PropertySchema.DataType.ENUM,
            enum_options=['white', 'red', 'green', 'blue', 'blacklight'],
            required=True)
    ]))
```

Creating a new white lightbulb:

```python
CreateRecordAction(
    record_id='5up4-br1t',
    record_type='lightbulb',
    properties=[
        PropertyValue(
            name='color',
            data_type=PropertySchema.DataType.ENUM,
            enum_value='white')
    ])
```

Updating that lightbulb to now be a blacklight:

```python
UpdatePropertiesAction(
    record_id='5up4-br1t',
    properties=[
        PropertyValue(
            name='color',
            data_type=PropertySchema.DataType.ENUM,
            enum_value='blacklight')
    ])
```

The JSON received when requesting this lightbulb from the server:

```json
{
  "recordId": "5up4-br1t",
  "owner": "03fb457a09cc...",
  "custodian": "03fb457a09cc...",
  "final": false,
  "properties": [
    {
      "name": "color",
      "type": "ENUM",
      "value": "blacklight",
      "reporters": [ "03fb457a09cc..." ]
    }
  ],
  "updates": {
    "owners": [ ... ],
    "custodians": [ ... ],
    "properties": {
      "color": [
        {
          "timestamp": 1520880613,
          "value": "white"
        },
        {
          "timestamp": 1520880674,
          "value": "blacklight"
        }
      ]
    }
  },
  "proposals": []
}
```

The JSON received when attempting to change the color to mauve:

```json
{
  "error": "InvalidTransaction: attempted value for \"color\" was not valid: mauve"
}
```

## Whole and Fractional Numbers

Although a `FLOAT` data type is currently available, it has been generally
considered bad practice to actually use it because of the potential determinism
bugs it can introduce. Best practices should be enforced by removing the
option, but it needs a replacement that is equally easily understood by clients
which consume the supply chain. This can be accomplished by utilizing the
existing `INT` type and adding an exponent property. Setting this
property turns a value from a regular integer to one that might be expressed
though scientific notation. So for example:

```
(value: 24, exponent: 3)  -> 24 * 10^3  -> 24000
(value: 24, exponent: -3) -> 24 * 10^-3 -> 0.024
(value: 24, exponent: 0)  -> 24 * 10^0  -> 24
```

Importantly, this exponent will be set on a Property's _schema_, not when the
value is actually input. It will affect the semantic meaning of integers stored
under a Property, not any of the actual operations done with them. Properties
with an exponent of `3` or `-3` will always be expressed in some whole integer
number of thousands or thousandths. For this reason, the exponent should be
thought of more as a unit of measure than as true scientific notation.

Because this new usage will now represent both whole and fractional numbers,
the designation `INT` no longer adequately describes it, and so the data type
will be renamed `NUMBER`.

### Examples

Creating a lightbulb RecordType in Python with a `brightness` Property
measured in millionths of a lumen:

```python
CreateRecordTypeAction(
    name='lightbulb',
    properties=[
        PropertySchema(
            name='brightness',
            data_type=PropertySchema.DataType.NUMBER,
            number_exponent=-6,
            required=True)
    ]))
```

Creating a new lightbulb of 800 lumens:

```python
CreateRecordAction(
    record_id='5up4-br1t',
    record_type='lightbulb',
    properties=[
        PropertyValue(
            name='brightness',
            data_type=PropertySchema.DataType.NUMBER,
            number_value=800000000)
    ])
```

Updating that lightbulb to now have 791.234 lumens:

```python
UpdatePropertiesAction(
    record_id='5up4-br1t',
    properties=[
        PropertyValue(
            name='brightness',
            data_type=PropertySchema.DataType.NUMBER,
            int_value=791234000)
    ])
```

The JSON received when requesting this lightbulb from the server:

```json
{
  "recordId": "5up4-br1t",
  "owner": "03fb457a09cc...",
  "custodian": "03fb457a09cc...",
  "final": false,
  "properties": [
    {
      "name": "brightness",
      "type": "NUMBER",
      "value": 791234000,
      "reporters": [ "03fb457a09cc..." ]
    }
  ],
  "updates": {
    "owners": [ ... ],
    "custodians": [ ... ],
    "properties": {
      "brightness": [
        {
          "timestamp": 1520880613,
          "value": 800000000
        },
        {
          "timestamp": 1520880674,
          "value": 791234000
        }
      ]
    }
  },
  "proposals": []
}
```

## Structs

The final and most complex new data type is `STRUCT`. Structs are a recursively
defined collection of other named properties, representing two or more values
that are intrinsically linked, like X/Y coordinates or RGB colors. These values
can be of any Supply Chain data type including `STRUCT`, allowing nesting to an
arbitrary depth. Although versatile and powerful, Structs are heavyweight and
should be used somewhat conservatively, restricted only to linking values that
must always be updated together. The transaction processor will enforce this
usage, rejecting any transactions that do not have a value for _every_ property
in a Struct.

Note that although Structs are built using a list of PropertySchemas, any
nested use of the `required` property is meaningless and will be rejected by
the transaction processor. As Properties are set in their entirety, either the
entire Property is required, or none of it is.

### Examples

Creating a lightbulb RecordType in Python with a `shock` Property that measures
any jostling it might receive during shipping using a combination of change in
velocity and duration:

```python
CreateRecordTypeAction(
    name='lightbulb',
    properties=[
        PropertySchema(
            name='shock',
            data_type=PropertySchema.DataType.STRUCT,
            struct_properties=[
                PropertySchema(
                    name='speed',
                    data_type=PropertySchema.DataType.NUMBER,
                    number_exponent=-6),
                PropertySchema(
                    name='duration',
                    data_type=PropertySchema.DataType.NUMBER,
                    number_exponent=-6),
            ],
            required=True)
    ]))
```

Creating a new lightbulb with a zeroed out initial shock:

```python
CreateRecordAction(
    record_id='5up4-br1t',
    record_type='lightbulb',
    properties=[
        PropertyValue(
            name='shock',
            data_type=PropertySchema.DataType.STRUCT,
            struct_values=[
              PropertyValue(
                  name='speed',
                  data_type=PropertySchema.DataType.NUMBER,
                  number_value=0),
              PropertyValue(
                  name='duration',
                  data_type=PropertySchema.DataType.NUMBER,
                  number_value=0)
            ])
    ])
```

And updating that lightbulb after it accelerated to 0.5m/s over 0.01 seconds:

```python
UpdatePropertiesAction(
    record_id='5up4-br1t',
    properties=[
        PropertyValue(
            name='shock',
            data_type=PropertySchema.DataType.STRUCT,
            struct_values=[
              PropertyValue(
                  name='speed',
                  data_type=PropertySchema.DataType.NUMBER,
                  number_value=500000),
              PropertyValue(
                  name='duration',
                  data_type=PropertySchema.DataType.NUMBER,
                  number_value=10000)
            ])
    ])
```

The JSON received when requesting this lightbulb from the server:

```json
{
  "recordId": "5up4-br1t",
  "owner": "03fb457a09cc...",
  "custodian": "03fb457a09cc...",
  "final": false,
  "properties": [
    {
      "name": "shock",
      "type": "STRUCT",
      "value": {
        "speed": 500000,
        "duration": 10000
      },
      "reporters": [ "03fb457a09cc..." ]
    },
  ],
  "updates": {
    "owners": [ ... ],
    "custodians": [ ... ],
    "properties": {
      "shock": [
        {
          "timestamp": 1520880613,
          "value": {
            "speed": 0,
            "duration": 0
          }
        },
        {
          "timestamp": 1520880674,
          "value": {
            "speed": 500000,
            "duration": 10000
          }
        }
      ]
    }
  },
  "proposals": []
}
```

## Type Summary

If all data type proposals suggested in this RFC are accepted, the new set of possible data types would be:

- `BYTES`: opaque byte code, decoding would be up to the clients
- `BOOLEAN`: true or false
- `NUMBER`: a number stored as an integer, with an exponent to designate
  fractional values
- `STRING`: a string of text
- `ENUM`: a limited set of named values, designated by a zero-based index
- `STRUCT`: a recursive structure with named values of any type
- `LOCATION`: latitude and longitude in millionths of a degree

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Protobuf Changes

The new data type will necessitate changes to the `PropertySchema` and
`PropertyValue` protobufs defined in `protos/property.proto`:

```protobuf
message PropertySchema {
  enum DataType {
    BYTES = 0;
    BOOLEAN = 1;
    NUMBER = 2;
    STRING = 3;
    ENUM = 4;
    STRUCT = 5;
    LOCATION = 6;
  }

  string name = 1;
  DataType data_type = 2;
  bool required = 3;

  sint32 number_exponent = 10;
  repeated string enum_options = 11;
  repeated PropertySchema struct_properties = 12;
}


message PropertyValue {
  string name = 1;
  PropertySchema.DataType data_type = 2;

  bytes bytes_value = 10;
  bool boolean_value = 11;
  sint64 number_value = 12;
  string string_value = 13;
  uint32 enum_value = 14;
  repeated PropertyValue struct_values = 15;
  Location location_value = 16;
}
```

These changes will not be backwards compatible. In addition to the elimination
of `FLOAT` and `INT`, the protobuf messages are being rearranged slightly.
Existing clients would have to be updated in order to take advantage of the new
data structures.

## New Validation Rules

While booleans, Numbers, and even Enums can be handled fairly trivially by the
transaction processor, Structs will require some extra validation. Care should
be taken to ensure that every nested value is set and properly named.

# Drawbacks
[drawbacks]: #drawbacks

One concern might be backwards compatibility. All of these changes except
Numbers could be structured to be non-breaking. However, Sawtooth Supply Chain
has made no claims of reaching 1.0 status, and today has a fairly small user
base. Its continued development should not be constrained by backwards
compatibility concerns until it is somewhat more mature and complete.

The Struct data type will be non-trivial to implement, and the ability to nest
them infinitely might introduce new concerns. If this is a worry, Structs could
be limited to a single layer deep (no Structs in Structs), and still satisfy
the vast majority of their use cases.

Overall, there are probably limited reasons _not_ to expand the data types
available to Sawtooth Supply Chain. Ultimately its just different ways of
structuring the bytes the blockchain will store. These new types will be
efficient and bring more semantic meaning.

# Rationale and alternatives
[alternatives]: #alternatives

For the most part these are fairly straightforward extensions of the current
data types available. There are certainly others which might have been
implemented, but these four all have past or future use cases, and address
specific challenges to the usability of Supply Chain. It is very difficult to
write a client that is not tightly coupled to a particular RecordType if you
have no way of telling the difference between a string and a JSON object.

These changes also enforce writing better applications. Structs will be much
more efficient than JSON. Floating point values should pretty much always be
avoided in Sawtooth, and the new Number type provide a viable alternative.
Without these changes, future Sawtooth Supply Chain developers will be stuck
using hacky error prone workarounds

Specific to Enums now, those might be updated by index rather than name.
This is simpler from the transaction processor's perspective, and in some ways
is less prone to typos. However, a typo in a name will be caught and rejected
immediately, while an incorrect index would likely go unnoticed and committed
permanently to the blockchain.

Setting Struct values is a somewhat cumbersome affair, and could be simplified
by having the transaction processor use the index of values and the RecordType
as a lookup. However, this introduces more opportunities for uncaught errors,
as two values of the same type could have their positions swapped, and make it
onto the blockchain.

There are a number of well documented ways to handle fractional numbers that
might be considered. The current design is favored for its simplicity. It is
essentially just an imposition to use integers, along with a clue as to how
that integer can be translated into decimal point notation.

# Prior art
[prior-art]: #prior-art

While these designs were influenced by some existing math and CS conventions
around fixed point numbers, enums, and dictionaries/structs, there is no
outright prior art.

# Unresolved questions
[unresolved]: #unresolved-questions

There are no unresolved questions at this time, however there are other
improvements that could be proposed in future RFCs, specifically giving
Properties referenceable types. The could be used for Properties with any data
type, but pairs particularly well with Structs. The current `LOCATION` data
type is essentially a hard coded Struct, and was included to give clients the
semantic meaning they needed to know when to display a map. Instead a
PropertyType named "location" could be created and stored on the blockchain.
This would allow the removal of an overly specific hard coded type, but more
importantly open the door for application developers to design and parse
anything they need.

However, making referenceable PropertyTypes is outside of the scope of this
RFC, and will be detailed in its own separate document.