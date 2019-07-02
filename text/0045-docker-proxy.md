- Feature Name: docker_proxy
- Start Date: 2019-07-01
- RFC PR: https://github.com/hyperledger/sawtooth-rfcs/pull/45
- Sawtooth Issue: N/A

# Summary
[summary]: #summary

Sawtooth has containerized build and test processes for core and all
components. Running Docker containers behind a network proxy requires
special handling so that the containers can find external routes to download
packages and build dependencies, for example. There are a number of
ways to configure these environment for proxy support. It is desirable that we
adopt a single consistent approach across the large body of Docker and
Docker Compose files in the Sawtooth project.

This RFC defines the canonical approach for supporting proxies for
Sawtooth containers using Docker 18.x and Docker Compose 1.24.x with
Ubuntu 16.04 and 18.04 and serves as guidelines to resolve these issues.

# Motivation
[motivation]: #motivation

Sawtooth repositories make extensive use of Docker containers to build and
test the software as well as provide turn key examples. Those containers
typically need to fetch external packages or make other outbound network
calls. The containers need special configuration to be executed behind
network proxies. There are multiple ways to provide that proxy configuration
and it is desirable that the Sawtooth repositories standardize on a single
approach. The challenge is further complicated by different abilities and
constraints of different versions of Docker and Ubuntu.

Starting with Ubuntu 18 (Bionic), `gpg` , invoked through `apt-key adv` ,
now ignores `http_proxy` and `https_proxy` environment variables, and ignores
the command line options
`apt-key adv --keyserver-options http-proxy=<proxy server url>`
and `https-proxy=<proxy server url>` . The only method remaining is with
cumbersome apt command-specific configuration file settings in
`/etc/apt/apt.conf.d/proxy` .


# Goals
[goals]: #goals

* Have Docker and Docker Compose files and shell scripts that work in explicit
  proxy, transparent proxy, and non-proxy environments
* Have a uniform method to access proxy servers across Hyperledger Sawtooth
  projects * Minimize Docker and script clutter with unnecessary tests and
  settings

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In order to use the proposed containers, developers and users in proxied
environments must set these proxy variables in their
`$HOME/.docker/config.json` configuration file. The `config.json` file
resides on the Docker host system, not in the Docker container filesystem.
The relevant settings are:

* `httpProxy` sets the proxy and port for unsecure web (HTTP) requests.
  The setting is site-specific for your corporate proxy server. For example,
  `"httpProxy": "http://proxy.foocorp.com:8080",`
* `httpsProxy` sets the proxy and port for HTTPS (TLS) requests. For example,
  `"httpsProxy": "http://proxy.foocorp.com:4443",`
* `ftpProxy` sets the proxy for FTP file transfer (this is not common)
* `noProxy` sets a comma-separated list of networks and hosts that are in
  the internal network. For example,
  `"noProxy": "192.168.0.0/16,172.16.0.0/12,10.0.0.0,localhost,127.0.0.0/8,::1,foocorp.com",`

  Make sure you include Docker Compose internal network(s), which by default
   are within `172.16.0.0/12` (use `netstat -rn` to list your networks)
* Do not set `http_proxy` , `https_proxy` , and `no_proxy` or their upper
  case equivalents. This method has been replaced with the `config.json`
  settings.
* Proxy support should be briefly mentioned in the README.md file or similar
  file, along with the proxy variable(s) needed, such as `httpProxy`

The following is a generic example of `$HOME/.docker/config.json` . The
actual contents are site specific:

```
{
 "proxies":
 {
   "default":
   {
     "httpProxy": "http://proxy.foocorp.com:8080",
     "httpsProxy": "http://proxy.foocorp.com:4443",
     "noProxy": "192.168.0.0/16,172.16.0.0/12,10.0.0.0,localhost,127.0.0.0/8,::1,foocorp.com"
   }
 }
}
````

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

All Docker files must be updated to conform as follows:
* Commands `apt, apt-get, cargo, deb, go, npm, pip3, wget` and others require
  no changes
* `curl` requires no changes, but it is not usually installed. It needs to be
  installed for containers that use it. For example,
  `RUN apt-get update && apt-get install -yq curl `

* `apt-key` : Instead of using `apt-key adv` to download an apt signing key,
  use `curl` and `apt-key add -` . For example, instead of the following in a
  `Dockerfile` (which no longer works):

```
RUN \
 apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 \
 --recv-keys 8AA7AF1F1091A5FD
```

Use the following, which works in proxy (with http_proxy) and non-proxy environments:

```
RUN \
 apt-get update && apt-get install -yq curl && curl -sSL \
 'http://keyserver.ubuntu.com/pks/lookup?op=get&search=0x8AA7AF1F1091A5FD’\
 | apt-key add -
```


# Conclusion
[conclusion]: #conclusion

These recommendations allow Docker and Docker compose use with Sawtooth
software in a standard, concise manner that work in both a corporate proxy
and non-proxy (or transparent proxy) environments.

# Drawbacks
[drawbacks]: #drawbacks

None

# Rationale and alternatives
[alternatives]: #alternatives

The current alternative used is hand-modifying Docker scripts each time a
proxy is required. This is time consuming, expensive, and unnecessary.

An earlier alternative of using `apt-key adv` and passing proxy variables on
the gpg command line (discussed above) worked in Ubuntu Xenial, but not in
Bionic. This alternative has the disadvantage of making Docker scripts messy
with a lot of proxy configuration setup.

The alternative of setting `https_proxy` , `http_proxy` , and `no_proxy`
variables has the disadvantage of requiring these variables to be explicitly
passed through the `args:` and `environment:` sections for each container in a `docker-compose.yaml` file.

A forth alternative, setting configuration for newer versions of gpg (in
Bionic, but not in Xenial) is more difficult as it does not work in Xenial
and requires adding GPG-specific configuration files containing proxy
configuration (instead of just passing an environment variable. This will
also require messy proxy configuration setup in Docker.


# Prior art
[prior-art]: #prior-art

Not applicable.

# Unresolved questions
[unresolved]: #unresolved-questions

None.
