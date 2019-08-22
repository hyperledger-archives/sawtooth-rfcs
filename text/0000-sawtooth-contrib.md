- Feature Name: sawtooth_contrib_repo
- Start Date: 2019-08-21
- RFC PR:
- Sawtooth Issue:

# Summary
[summary]: #summary

Creates a new sawtooth-contrib repository to provide a "landing pad" for
Sawtooth-related code that is not officially part of Sawtooth. This repository
will contain a variety of content, including contributed examples, experimental
tools and scripts, unofficial integrations, and experimental new components.

# Motivation
[motivation]: #motivation

Several Sawtooth repositories are valuable to the project but aren't being
actively maintained or released. Consolidating them into a single repository
will reduce overhead for repo maintenance and associated build tasks.
Additionally, a single sawtooth-contrib repository makes it easier for people
to discover Sawtooth and contribute new projects.

In addition, Sawtooth needs a place for contributions that would otherwise not
have a home.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The sawtooth-contrib repository will be a place for unofficial Sawtooth-related
software, including example libraries, data models, immature SDKs, and
reference implementations for smart contracts (also called "transaction
families"). The software in this repository must be designed to work with the
Hyperledger Sawtooth platform.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Existing projects that would be candidates to move into
the sawtooth-contrib repo include:

* [Sawtooth Ansible](https://github.com/hyperledger/sawtooth-ansible)

* [Sawtooth Marketplace](https://github.com/hyperledger/sawtooth-marketplace)

The sawtooth-contrib repo will differ from the existing Sawtooth repositories
in a few important ways:

* No builds

  The repository will not be configured for any CI builds (not included in
  regularly scheduled builds and PR builds).

* Unversioned

  Because the components in sawtooth-contrib aren't being regularly released,
  the repository won't be tagged or versioned.

* Special rules for repository maintainers

  The maintainers list for sawtooth-contrib will be composed of the
  maintainers of all other Sawtooth repositories. Because the sawtooth-contrib
  repo will be the home for many different types of projects, it makes sense to
  have a broad list of maintainers with varied expertise.

* Lower threshold for contribution

  The requirements for contribution will be lower in sawtooth-contrib, so that
  partially implemented ideas and features have a place for collaboration. As a
  result, the maintainers may allow fairly unfinished/unrefined code to be
  committed.

  For example, Chef recipes or Kubernetes Helm charts would require substantial
  infrastructure to test completely. However, a contribution of initial recipes
  or  charts would likely be admitted to provide a starting point for others to
  use and build upon.

There will be no official "graduation" process to move contributions into the
official Sawtooth codebase. However, it is expected that some components will
incubate here and then be integrated into the official Sawtooth repositories,
either as part of existing components or as new components.

# Drawbacks
[drawbacks]: #drawbacks

The repository could become a dumping ground for unmaintained and low-quality
code. This will ultimately require effort to update and purge content from
the sawtooth-contrib repository.

# Prior art
[prior-art]: #prior-art

Other Hyperledger projects are already using this model:

* [grid-contrib](https://github.com/hyperledger/grid-contrib)
* [transact-contrib](https://github.com/hyperledger/transact-contrib)


# Unresolved questions
[unresolved]: #unresolved-questions

None.
