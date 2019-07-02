- Event Processor SDK    
- Start Date: 2019-07-02
- RFC PR: (leave this empty)
- Sawtooth Issue: (leave this empty)

# Summary
[summary]: #summary

The Event Processor SDK is an extension to the Sawtooth SDK that provides a
simplified abstraction to consume events generated on block commits by the
Sawtooth validator.

# Motivation
[motivation]: #motivation

The events emitted by the validator are an incredibly useful tool for pulling
data off-chain, but the current SDK does not provide a straight-forward way to
subscribe and receive events.  This API provides an analogous framework to the
transaction processors by removing the low-level transport and messaging
details.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Event Processor SDK provides APIs for generating subscriptions, a handler
interface for events, and an event processor for managing messaging and
dispatching events to handlers.

Subscriptions are generated through a subscription builder.  The protocol
requirements and format are isolated behind this abstraction.  The builder is
used to specify the event types and their filters.

The event handler interface handles a subset of events received.  It specifies
the names of events of interest.  It is provided the events of interest by the
event processor.  Each handler should manage its own state, such as connections
to databases or log files.

The event processor takes a validator endpoint, a set of subscriptions, and a
set of event handlers.  The event processor manages connecting to the validator,
submitting the subscriptions, and receiving the event messages.  As events are
received, the event processor finds the appropriate handlers for the given set
of events, and provides the events of interest to each handler.

Unlike transaction handlers, there may be multiple event handlers that are
interested in specific event types.  For example, there may be a subscription
for `sawtooth/block-commit`, `sawtooth/state-delta`, and `my-app/custom-event`.
A handler may be created that handles the `block-commit` and `state-delta`
events, and writes the state values to a
[slowly-changing-dimensions](https://en.wikipedia.org/wiki/Slowly_changing_dimension#Type_2:_add_new_row)
database.  A second handler may handle the `block-commit` and `custom-event` and
forward the values to an external queue system, such as Apache Kafka.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Subscription Builders
[subscription-builders]: #subscription-builders

The subscription builder API provides a clean abstraction for specifying
Sawtooth event subscriptions.  

The following API is presented in Rust, but is applicable to all of the SDKs.
The builders are not traits, in this case, but the implementation of the structs
and the functions are omitted here.

```rust
impl SubscriptionBuilder {
    fn with_last_known_block_ids(&mut self, block_ids: Vec<String>)
        -> Self
    { ... }

    fn with_event(&mut self, event_filter: EventFilter)
        -> Self 
    { ... }

    fn build(self)
        -> Result<Subscription, SubscriptionBuildError>
    { ... }
}

enum FilterConstraint {
    Any,
    All,
}

impl EventFilterBuilder {
    fn with_event_type(&mut self, event_type: String)
        -> Self 
    { ... }
    
    fn with_regex_filter(
        &mut self, 
        key: String, 
        regex: String, 
        constraint: FilterConstraint
    ) -> Self 
    {...}

    fn with_simple_filter(
        &mut self, 
        key: String, 
        value: String, 
        constraint: FilterConstraint
    ) -> Self 
    {...}

    fn build(self) -> Result<EventFilter, EventFilterBuildError> 
    { ... }
}
```

Subscriptions (`Subscription` instances) are made up of a series of event
filters (`EventFilter` instances).  These filters match on specific key/value
pairs, either through an exact match or a regular expressions.  The filters may
apply to all match keys in an event or any, as specified by the
`FilterConstraint` set when the filter is created.

## Event Handlers
[event-handlers]: #event-handler

The event handler interface is the core piece that SDK consumers use to
implement their business logic.  It is analogous to the transaction handler API.
The main difference between them is that it doesn't necessarily have to be
deterministic. 

Events are handled as a group.  Many events provide only a slice of the
information that occurs at block commit time. For example, `block-commit` and
`state-delta` events are often handled in pairs, when connecting to a validator
network that is using a lottery-style consensus.

The events passed to the handler are guaranteed to have occurred within a single
block.

The following API is presented in Rust, but is applicable to all of the SDKs. 

```rust
trait EventHandler {
    fn accepts_event_types(&self) -> &[&str];
    fn handle(&self, events: &[Event]) -> Result<(), HandleEventError>;
}

enum HandleEventError {
    Internal(Box<dyn std::error::Error>),
}
```

The `accepts_event_types` function returns the list of events that the handler
will accept.  The events provided to the handler are merely references, as
multiple handlers may read the data contained in the same event.

## Event Processor
[event-processor]: #event-processor

The event processor uses the subscriptions, the handlers and a validator
component endpoint to connect, subscribe and process event messages. It is
analogous to the transaction processor. Its main difference is that it may
dispatch the same event to multiple handlers. 

The following API is presented in Rust, but is applicable to all of the SDKs.
The event processor is not a trait in this case, but the implementation of the
struct and the functions are omitted here.

```rust
struct EventProcessor 
{ ... }

impl EventProcessor {
    pub fn new(validator_endpoint: String) -> Self
    

    pub fn add_subscription(&mut self, subscription: Subscription)
        -> Result<(), EventProcessorError>
    { ... }
    

    pub fn add_event_handler(&mut self, event_handler: Box<dyn EventHandler>)
        -> Result<(), EventProcessorError>
    { ... }
    

    pub fn start(self)
        -> Result<ShutdownHandle, EventProcessorError> 
    { ... }
}

struct ShutdownHandle
{ ... }

impl ShutdownHandle {
    pub fn shutdown(mut self) -> Result<(), EventProcessorError>
    { ... }
}
```
 
A `EventProcessorError` indicates errors when adding a service or starting the
processor.  The `ShutdownHandle` returned from start blocks until any background
resources have been terminated

# Drawbacks
[drawbacks]: #drawbacks

The only drawback is the fact this extension to the SDKs should be implemented
in all the current first-tier SDKs.  It has increased maintenance costs.

# Rationale and alternatives
[alternatives]: #alternatives

The benefit of providing as simple API for event processing as is provided for
transaction execution is obvious. Much of the positive feedback from the
community is based around the ease of creating applications at the state and
transaction level. This adds a third leg in the platform by providing an easy
way to offload state to external databases with richer query capabilities. 

The alternative is continuing to maintain the current stream and protobuf
message method for subscribing to and receiving events. Improvements to
documentation could improve adoption of the event system. However, this current
method still would be clumsy. 

# Prior art
[prior-art]: #prior-art

Much of the concepts described here are inspired by the current Sawtooth
transaction processor API in the SDK.  

A similar, though not complete implementation of this API was implemented as
part of Hyperledger Grid's state delta export capabilities.  Similarly, the
builder patterns used for protocol objects is found in much of Grid.

# Unresolved questions
[unresolved]: #unresolved-questions

None.
