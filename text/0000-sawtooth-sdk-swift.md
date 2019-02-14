- Feature Name: sawtooth_sdk_swift
- Start Date: 2019-02-14
- RFC PR:
- Sawtooth Issue:

# Summary
[summary]: #summary

Develop a Sawtooth SDK for Swift to facilitate development for client side
Cocoa/Cocoa Touch applications. It is modeled after existing SDKs,
implementing a Sawtooth Signing module that provides methods to generate
private/public key pairs and sign transactions. A separate repository should
be created to host the Swift SDK.

# Motivation
[motivation]: #motivation

As the interest in Sawtooth grows, so does the demand for native mobile
applications that can create and submit valid transactions. Additionally, the
introduction of Hyperledger Grid has expanded the possibility for mobile use
cases for the Sawtooth platform. The creation of a Swift SDK will facilitate
the development of native applications for iOS and fulfill a need from the
community.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Swift SDK works similarly to existing SDKs. It provides a SawtoothSigning
module that supports generating private/public key pairs and signing bytes.
This module facilitates signing transactions in a way that is compatible with
Sawtooth requirements.

The Swift SDK proposed here does not currently implement a module to assist
with the creation of Transaction Processors, as the initial purpose is to
support front-end application development.

### Capabilities

- Generate private/public key pairs
- Sign transactions
- Verify signatures

###### Example
```
import SawtoothSigning

let context = Secp256k1Context()
let privateKey = context.newRandomPrivateKey()
let signer = Signer(context: context, privateKey: privateKey)
let signature = signer.sign(data: message_bytes)
context.verify(signature: signature, data: message_bytes,
publicKey: signer.getPublicKey())
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

An initial implementation of the SawtoothSigning framework has already been
written. The code can be found at
[bitwiseio/sawtooth-sdk-swift](https://github.com/bitwiseio/sawtooth-sdk-swift).
The repository also contains documentation explaining how to use the
SawtoothSigning framework and a working example of the framework in use by an
iOS client for the
[xo transaction family](https://sawtooth.hyperledger.org/docs/core/releases/latest/transaction_family_specifications/xo_transaction_family.html).

### Dependencies

#### SawtoothSigning
- [secp256k1](https://github.com/Boilertalk/secp256k1.swift)

#### XO client example
- SawtoothSigning
- [SwiftProtobuf](https://github.com/apple/swift-protobuf/)
- [secp256k1](https://github.com/Boilertalk/secp256k1.swift)

### Modules and Methods

#### SawtoothSigning
- Secp256k1Context
  - sign
  - verify
  - getPublicKey
  - newRandomPrivateKey
- CryptoFactory
  - createContext
- Secp256k1PrivateKey
  - fromHex
  - hex
  - getBytes
- Secp256k1PublicKey
  - fromHex
  - hex
  - getBytes
- Signer
  - sign
  - getPublicKey

### Importing SawtoothSigning framework
The Swift SDK's SawtoothSigning module can be imported to a Cocoa/Cocoa Touch
project via [Carthage](https://github.com/Carthage/Carthage), a dependency
manager for Cocoa applications.

1. Install Carthage using
[these installation instructions](https://github.com/Carthage/Carthage#installing-carthage).

2. Create a Cartfile in the same directory as your `.xcodeproj` or
`.xcworkspace`.

3. In the Cartfile for your project, add:
   ```
   github "bitwiseio/sawtooth-sdk-swift" "master"
   ```

4. Run `carthage update`.

   After the framework is downloaded and built, it can be found at `path/to/your/project/Carthage/Build/<platform>/SawtoothSigning.framework`

5. Add the built .framework binaries to the "Embedded Binaries" and
"LinkedFrameworks and Libraries" sections in your Xcode project.

# Drawbacks
[drawbacks]: #drawbacks

There are no known drawbacks for this addition, besides the maintainence of an
additional SDK. No existing features or repositories will be changed.

# Rationale and alternatives
[alternatives]: #alternatives

The Sawtooth Swift SDK is modeled after the existing SDKs, with similar 
classes and methods. Developers who are already familiar with other Sawtooth
SDKs and Swift will be able to use the Swift SDK with minimal learning
friction.

#### Objective-C SDK

An SDK that supports Cocoa/Cocoa Touch Applications could also be written in
Objective-C. In spite of Objective-C being an older language with many
feature-rich APIs, Swift is a better choice for a couple of reasons:

- Swift is an language that is easier to learn, with simpler syntax which 
  makes code easier to read and write. 
- Using Swift makes is possible to export the SawtoothSigning module as a
  dynamic framework rather than a static library. Dynamic frameworks reduce
  the launch time of apps and their memory footprints when compared to static
  libraries. To use SawtoothSigning as dynamic framework also allows the users
  to use Carthage to download and build the module directly from the GitHub
  repository.

#### Use existing functionality

Without the Swift SDK, developers have two options to write an iOS-compatible 
Sawtooth client:
- Create a web app using the Javascript SDK
- Implement custom methods to create and sign transactions natively

The first solution does not work for a use case that requires a native
application. The second solution raises the barrier of entry for developers
with a background in mobile development who want to use Sawtooth. The creation
of the Sawtooth Swift SDK, including documentation and examples, will make the
Sawtooth platform more accessible to those coming from mobile development.

# Prior art
[prior-art]: #prior-art

The implementation of the SawtoothSigning module for the Swift SDK is based
on existing implementations for other languages, such as 
[Java](https://github.com/hyperledger/sawtooth-sdk-java/tree/master/sawtooth-sdk-signing/src/main/java/sawtooth/sdk/signing) 
and
[Rust](https://github.com/hyperledger/sawtooth-sdk-rust/blob/master/src/signing/secp256k1.rs).

# Unresolved questions
[unresolved]: #unresolved-questions

### To be implemented

- To use Carthage to import the SawtoothSigning framework into an iOS project
  it is necessary to include a Cartfile, which describes the project’s
  dependencies to Carthage. GitHub repositories are specified with the github
  keyword. For example, it is possible to load the framework from a branch:
  ```
  github "bitwiseio/sawtooth-sdk-swift" "master"
  ```

  Or from a release:
  ```
  github "bitwiseio/sawtooth-sdk-swift" ~> 0.1
  ```

  If a new repository within the Hyperledger organization is created to host
  the Sawtooth Swift SDK, the existing Carfile in the XO example needs to be
  updated for the new path of the SDK. The documentation also needs to be
  updated to include instructions on how to import the SawtoothSigning
  framework from the new hyperledger repository.

- The Sawtooth protos are not included as part of the Swift SDK. Before
  stabilization, we may want to include them as part of the SDK. Currently,
  the Sawtooth `.proto` files and Swift protogen scripts are included directly
  in the XO client example application. Further work must be done to
  implement the generation of the protobuf messages as part of the Swift SDK.

### Other issues

- The documentation for the Swift SDK uses the same templates as the other
  SDKs. The templates were copied from the sawtooth-core repository. This is
  not an ideal solution, as there is an extra overhead to maintaining several
  versions of the same documentation across several repositories.

- Ideally, the documentation for the Swift SDK should be published under the
  Application Developer's Guide section in the Sawtooth documentation,
  together with the documentation for the other SDKs. It is not clear what the
  best strategy is to publish the Swift SDK documentation, given that the
  source files and code base are in an different repository than the Sawtooth
  documentation.
