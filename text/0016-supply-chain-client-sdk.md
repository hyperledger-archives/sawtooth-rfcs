- Feature Name: supply_chain_client_sdk
- Start Date: 2018-03-26
- RFC PR: [hyperledger/sawtooth-rfcs#16](https://github.com/hyperledger/sawtooth-rfcs/pull/16)
- Sawtooth Issue: N/A

# Summary
[summary]: #summary

Develop a JS SDK to abstract common aspects of building Sawtooth Supply Chain
front-ends, easing future UI development and minimizing code duplication. This
would mean pulling out the majority of the existing code in the `src/services/`
and `src/components/` directories with some refactoring to remove references to
specific commodities, and improve abstractions as needed. In addition, a set of
customizable page templates would be created, which reflect the commonalities
between different `src/views/` in different clients.

# Motivation
[motivation]: #motivation

Though the universal client specified elsewhere will make it easy to deploy
quick demos with no coding at all, that will still leave a gap for more
specific demos with custom pages. Currently, developing these demos involves a
lot of copying and pasting commonly used code, which is awkward and error
prone. It also isn't particularly friendly to new developers who might wish to
develop their own clients. Creating an internal JS SDK would explicitly
encourage reusing this code more, and make future development easier.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The SDK would not be published to npm, rather it would occupy a root directory
in the Supply Chain repo, and could be imported using Node's file system syntax
or by creating a link with `npm link` (the examples below use a link). It would
be broken up into three modules: services, components, and templates, which
would be further broken down into sub-modules specific to various tasks.
Imports could be by the parent module, or individually by sub-module. In either
case, all methods and classes will be made available at the root level of the
import.

For example, the `makePrivateKey` function could be imported three ways:

```javascript
const supplyChain = require('supply-chain');
supplyChain.makePrivateKey();
```

```javascript
const services = require('supply-chain/services');
services.makePrivateKey();
```

```javascript
const transactions = require('supply-chain/services/transactions');
transactions.makePrivateKey();
```

Using the most specific import possible will ensure small bundle sizes thanks
to webpack tree shaking.

## Services

Services will contain most of code from the existing `src/services/` modules,
but care will be taken to remove any references to the display layer of the
client and the Mithril.js library specifically. It would be suitable to import
into any front-end even if not built on Mithril, which will underpin the
components and template libraries.

### Capabilities

- Generate private/public key pairs
- Encrypt and store private/public keys
- Generate Sawtooth transactions
- Generate Supply Chain payloads
- Authorize users with the API
- Wrap API requests in methods that fill in auth and other header info
- Resource parsing (i.e. NUMBER/INT to display strings, fetching Property
  values by name)

### Examples

Creating a new user account with a new private key:

```javascript
const api = require('supply-chain/services/api')

...

api.createUser({ email, password, name })
  .then( user => {
      state.setUser(user);
      m.route.set('/');
    })
  .catch( err => alert(err.error));
```

Submitting a createRecord transaction:

```javascript
const { payloads, transactions, parsing } = require('supply-chain/services');

const payload = payloads.createRecord({
  recordId,
  recordType,
  properties: {
    species,
    length: parsing.toNumber(length),
    weight: parsing.toNumber(weight),
    location: { latitude, longitude }
  }
});

transactions.submit(payload)
  .then(() => m.route.set('/'))
  .catch(err => alert(err.error));
```

## Components

The components module contains Mithril/Bootstrap components which can be used
to construct pages. Most are small building blocks like headers or input
fields, which would be combined to construct larger pages. Some larger
components like map widgets and paginated lists will also be provided by the
module. Any complete pages would be in the templates module.

### Capabilities

  - Provide small reusable building blocks like headers and input fields
  - Data widgets like maps and graphs
  - Modals which return form data in a promise
  - Tables which display paginated lists of resources

### Examples

Creating a labeled form field to get a user's email:

```javascript
const forms = require('supply-chain/components/forms')

. . .

let email = null;

forms.emailInput(val => { email = val; }, 'Enter your email');
```

Launching a form in a FieldModal:

```javascript
const modals = require('supply-chain/components/modals');

. . .

const state = {};

modals.showModal(modals.FieldModal, {
  title: 'Update Destination',
  label: 'Enter new destination',
  submitText: 'Update',
  submitFn: dest => { state.destination = dest; }
});
```

## Templates

While the other two modules are heavily based on existing code, the templates
module would be brand new, at least in implementation if not concept. It will
consist of fully fleshed out page templates that mirror the existing client
pages, but customizable by passing app specific data in at instantiation (for
example, the names of headers). Built with Mithril/Bootstrap, every existing
page will be replicable with a template with no apparent change from a user
perspective.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Specific implementation will closely mirror the existing code base:

- `services` -> [src/services](https://github.com/hyperledger/sawtooth-supply-chain/tree/master/fish_client/src/services)
- `components` -> [src/components](https://github.com/hyperledger/sawtooth-supply-chain/tree/master/fish_client/src/components)
- `templates` -> [src/views](https://github.com/hyperledger/sawtooth-supply-chain/tree/master/fish_client/src/views)

The `templates` module will have the most changed, as the pages will need to be
fairly heavily modified to go from hard coded to templates that can be modified
with a simple API. None the less, all three modules will closely follow the
design of the existing code base.

## Dependencies

### services

- Lodash
- Stanford Javascript Crypto Library
- Node's crypto module
- ProtobufJS
- Sawtooth SDK

### components

  - the services module
  - Mithril.js
  - Bootstrap
  - Lodash
  - Octicons
  - Chart.js
  - Google Maps SDK

### templates

  - the services and component modules
  - Mithril.js
  - Bootstrap
  - Lodash

## Modules and Methods

Note that these list is not final, and will likely be revised during
implementation.

### services

- account
  - newPrivateKey
  - setPrivateKey
  - getPrivateKey
  - clearPrivateKey
  - updatePassword
  - getPublicKey
- api
  - request
  - get
  - post
  - patch
  - authorize
  - createUser
  - getAuth
  - setAuth
  - clearAuth
- parsing
  - stringify _(returns a display string for a specified data type)_
  - stringifyBool
  - stringifyNumber
  - stringifyEnum
  - stringifyStruct
  - stringifyTimestamp
  - toNumber _(converts a float into a NUMBER/INT with a scaling factor)_
  - getPropertyValue _(returns a Property value by name)_
  - isReporter _(checks if a public key is a reporter on a Property)_
  - isOwner
  - isCustodian
- payloads
  - createAgent
  - createRecord
  - finalizeRecord
  - createRecordType
  - updateProperties
  - createProposal
  - answerProposal
  - revokeReporterType
- transactions
  - create
  - submit

### components

  - layout
    - title
    - heading
    - row
    - rowsOf
    - icon
    - labeledField
  - forms
    - input
    - labeledInput
    - textInput
    - passwordInput
    - numberInput
    - emailInput
    - dateInput
    - select
    - dropdown
    - multiselect
    - clickableIcon
    - triggerDownload
    - stateSetter
    - validityChecker
  - data
    - Map
    - Graph
  - lists
    - PagingControls
    - FilterControls
    - List
  - modals
    - modalBase,
    - modalHeader,
    - modalBody,
    - modalFooter,
    - modalButton,
    - BasicModal,
    - FieldModal,
    - showModal
  - core
    - navbar
    - router


### templates

Unlike other modules, templates have no sub-methods. Instead, importing the
module would return a Mithril component directly, similar to how "views" are
structured in the current client code. Unlike views, these components would be
highly customizable, with no hard-coded content.

  - main_page
  - login_form
  - signup_form
  - agent_list
  - agent_detail
  - record_list
  - record_detail
  - create_record
  - property_detail

# Drawbacks
[drawbacks]: #drawbacks

In some ways this is a lateral move, time invested into turning existing code
into a library when copying and pasting works _okay_. However, down the road
this will speed up development of new custom clients, and might be especially
useful for new developers looking to use Supply Chain for their first PoCs.

The `components` and `templates` modules will be tightly bound to the Mithril
and Bootstrap libraries. They are versatile tools, and Bootstrap in particular
can be highly customized. However, there will be no simple way to use custom
CSS or a new MVC library with either module. As a result, some highly
customized clients may not find them useful. The `services` module, having no
display logic, should be useful for any Supply Chain client.


# Rationale and alternatives
[alternatives]: #alternatives

SDKs, modules, and libraries are common ways to package code for reuse. Short
of building a Wordpress style CMS or a full WYSIWYG editor, a module with a
sensible API is the most practical way to share useful client code and
standardize useful concepts like client-side encrypting.

Of course building out a WYSIWYG is far more of an investment than could
possibly pay off at this point. The only real alternative is to leave things as
they are, with the client code available for reference and to be copied and
pasted. This is probably acceptable for the most part, but will cost some
development time each time a new client is built, and will somewhat raise the bar to
entry for new adopters of the platform.

# Prior art
[prior-art]: #prior-art

As detailed earlier, the existing client code and design will heavily inform
the implementation of this SDK.

# Unresolved questions
[unresolved]: #unresolved-questions

The API for templates is not fully fleshed out. They would utilize Mithril's
`vnode.attrs` concept to pass an arbitrary dictionary of information into the
template. However the exact keys used will vary from template to template, and
will likely be hashed out during implementation.

Currently most components are simple _functions_ which take a few parameters
and return instantiated Mithril components. This was favored for its
simplicity, and is likely the approach this SDK will follow. However, for
consistency's sake, it may be worth using the more heavyweight _object_ format
used by more complex components in the client code. If they continue to be
mixed, clear dividing lines should be drawn between when one approach or
another is used to define Mithril components.
