- Feature Name: docker_compose_builds
- Start Date:  2018-03-27
- RFC PR:
- Sawtooth Issue:

# Summary
[summary]: #summary
Redesign the build process to move away from a monolithic procedure to one
where individual component builds are possible using Docker multi-stage builds.

# Motivation
[motivation]: #motivation

The current build system is a collection of bash scripts and Dockerfiles.
The build is performed by building and running Docker images in a specific
order. As the project has grown and more components and dependencies have been
added, this process has become more complex and opaque.

Ideally, developers would have the ability to build only the components needed
for their specific task.  However,  builds are presently tightly coupled by
language. For example, it is difficult to build a subset of components which
are needed only for Smallbank or Seth. After this change the "unit of build"
will be at the component level, which will allow building only the desired
components.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Each buildable component in the sawtooth-core repository will have two
associated Dockerfiles, (Dockerfile and Dockerfile-installed) that reside
alongside the component code. These are referenced by two Compose (version 3.5)
files (docker-compose.yaml and docker-compose-installed.yaml) in the root of
the repository.

Running `docker-compose -f docker-compose.yaml up --build` will
produce a set of docker images with external dependencies installed. These
images will be run with the local repository mounted in via Docker volume.
This is suitable for development.

Running `docker-compose -f docker-compose-installed.yaml up --build` will
produce a set of docker images with sawtooth installed from .deb files created
with the code present in the repository. The .deb files will be copied out of
the new images to sawtooth-core/build/debs. These images have no dependencies on
anything in the repository and are able to be published to a docker registry.

Individual components are buildable with either `docker-compose` or `docker`.

Docker:
```
cd validator
docker build -t sawtooth-validator -f Dockerfile-installed .
```

Compose:
```
docker-compose -f docker-compose-installed.yaml build validator
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The proposed build solution aims to replace all build scripts currently in the
repository with individual Dockerfiles that contain all dependencies and steps
required to build each component. This will be done by using Docker's
multi-stage builds in order to reduce the final image size.

The various non-installed Dockerfiles will look similar to the existing "dev"
Dockerfiles like `sawtooth-dev-python`.

Each "installed" Dockerfile will have multiple named build stages that build any
sawtooth packages that the component depends upon. These artifacts ultimately
are copied into a final build stage which will install only the .deb packages
and any run-time dependencies from the public "ci" apt repository.

A mock Dockerfile-installed for reference:

```
# -------------=== dependency build ===-------------
FROM ubuntu:xenial as dependency-builder

ENV VERSION=AUTO_STRICT

RUN echo "deb http://repo.sawtooth.me/ubuntu/ci xenial universe" >> /etc/apt/sources.list \
 && apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 8AA7AF1F1091A5FD \
 && apt-get update \
 && apt-get install -y -q
    python3 \
    python3-protobuf \

COPY . /project

RUN cd /project/dependency \
 && python3 setup.py clean --all \
 && python3 setup.py --command-packages=stdeb.command debianize \
 && dpkg-buildpackage -b -rfakeroot -us -uc


# -------------=== component build ===-------------

FROM ubuntu:xenial as component-builder

ENV VERSION=AUTO_STRICT

RUN echo "deb http://repo.sawtooth.me/ubuntu/ci xenial universe" >> /etc/apt/sources.list \
 && apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 8AA7AF1F1091A5FD \
 && apt-get update \
 && apt-get install -y -q
    python3 \
    python3-grpcio \

COPY --from=dependency-builder /project/sawtooth-dependency*.deb /tmp
RUN dpkg -i /tmp/sawtooth-*.deb

COPY . /project

RUN cd /project/component \
 && python3 setup.py clean --all \
 && python3 setup.py --command-packages=stdeb.command debianize \
 && dpkg-buildpackage -b -rfakeroot -us -uc

# -------------=== final build ===-------------
FROM ubuntu:xenial

COPY --from=dependency-builder /project/sawtooth-dependency*.deb /tmp
COPY --from=component-builder /project/sawtooth-component*.deb /tmp

RUN dpkg -i /tmp/sawtooth-*.deb

```

Nothing other than the Jenkinsfile depends on the build scripts so this change
can be made without affecting anything else in the repository.

# Drawbacks
[drawbacks]: #drawbacks

By choosing to build a full set of dependencies within each component’s
Dockerfiles, certain build systems that do not keep the interim build
layers cached throughout the entire build (such as Drone) may experience
longer build times.

Another side effect of building dependencies within each component’s
Dockerfiles is that there will be significant code duplication. Any time
changes are made to the method or requirements for building a component,
updates will be required in the Dockerfiles of everything that depends on it.
For instance, if the signing library is changed, the Dockerfiles for cli, PoET
core, PoET cli, PoET simulator, settings, rest api, python SDK, xo python,
intkey python, and validator will also need updating.

# Rationale and alternatives
[alternatives]: #alternatives

By removing dependence on a central build script, adding support for a new
platform is as simple as creating a new set of Dockerfiles with an associated
Compose file.

An alternate approach that relied on pre-building images using the
docker-compose feature `depends_on` option rather than using Docker multi-stage
builds was considered, but was ultimately not chosen because builds would only
be possible using the Compose files. The proposed method allows for individual
components to be built by navigating to the directory and running `docker build`.

Also considered was the idea of having separate Compose files for each
component. This would allow customization of the environment by specifying each
desired component as an argument to docker-compose. This idea was rejected
because the functionality was not considered to be useful and the resulting
commands were very long and complex. With the proposed approach, the
environment can be customized by modifying the Docker Compose files.

# Unresolved questions
[unresolved]: #unresolved-questions

- The proposed changes call for copying the repository into each docker image
and generating artifacts during docker build. Docker does not presently support
mounting VOLUMES or copying files from an image to the host during build time.
The mechanism for extracting build artifacts from the completed images is still
to be determined.

- Because of the significant code duplication, managing the build system may be
unwieldy. One solution may be to generate the Dockerfiles from templates.

 - All protos are currently created by the bin/protogen script, this
functionality will need to be replicated in the setup.py file or equivalent for
each component.
