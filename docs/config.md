# Configuration build and distribution

The configuration, in the framework pipeline, is a bundle of file that instructs
the event collection agents on how to run and what identity they assume.

## Table of contents
1. [The contents of the configuration](#the-contents-of-the-configuration)
    - [The agent configuration](#the-agent-configuration)
    - [Cryptographic trust evidence](#cryptographic-trust-evidence)
2. [The layers of configuration](#the-layers-of-configuration)
3. [Configuration bundling](#configuration-bundling)
    - [Namespace](#namespace)
4. [Configuration bundling for a static file
   server](#configuration-bundling-for-a-static-file-server)
    - [Configuration tree](#configuration-tree)
    - [Configuration lookup](#configuration-lookup)
    - [Configuration packaging to a singlebundle](#configuration-packaging-to-a-single-bundle)
    - [Note on the X.509 Certificate](#note-on-the-x509-certificate)
5. [Configuration bundling for an API
   server](#configuration-bundling-for-an-api-server)
   - [Use of query parameters](#use-of-query-parameters)
   - [API server response](#api-server-response)
6. [Configuration transport and trust](#configuration-transport-and-trust)
7. [A Sample JWS Response](#a-sample-jws-response)

## The contents of the configuration

Depending on the number and complexity of the event collection agents, there can
be multiple configuration file required for the audit agents to run. This
may be fetched by the agents themselves or be ensured by the orchestrator if
it's generic enough.

### The agent configuration

The configuration that will be utilized by the event collection agents, must
have all the required event types they required to collect and forward. It may
optionally contain flags that can be used to tune the agents or enable or
disable features since these are endpoint agents.

For example, considering there are multiple agents with the following
responsibilities,

#### The configuration bootstrap
A bootstrap configuration file that is not dynamic is managed by the
orchestrator and contains the smallest subset of items required to bootstrap the
whole process.
* Where should it fetch the agents and required extension and configurations
  from.
* Fallback settings
* Ownership information in case a dynamic endpoint is not present for getting
  the info.

#### Event collector
* What event types are supposed to be collected.
* What data, in the context of the computes, should be present in each event.
* At what interval should the snapshots be collected
* Where should the logs be emitted to.

#### Event annotator
The event annotator will accept the raw events and annotate it with context
information. A configuration file should contain at least the following
information.
* A direct point to point contact of a person, usually user id of the owner
* A distribution list for the team that owns the application
* The identity and source of the configuration. Each version should be versioned
  controlled in order to determine what configuration was running at on a
  particular instance.
* Organization information, e.g. pillar, teams, divisions etc.
* Provision to add additional tags to the events for custom filtering.

#### Log forwarder
The log forwarder is something which shall be controlled by the team responsible
for the audit infrastructure hence shall be managed by the orchestrator. The
orchestrator should be responsible for the setting the log forwarding agent and
the required configurations.

This could include,
* The destination for the event logs
* The fall back buffer configuration
* The tuning parameters

### Cryptographic trust evidence
The configurations can be distributed as plain text files or API responses.
Either of them should allow free read access to anyone with sufficient
information to verify the source of the configuration and it's validity. All
parameters that are used to verify the configuration should also be sent out as
events. Refer the event collector agent section for more details.

The cryptographic trust evidence consists of a Digital signature using an
Asymmetric key which is signed and certified by the CA the organization uses.
More on this in [Configuration bundling](#configuration-bundling).

## The layers of configuration

There are multiple parties involved in setting up the pipeline. For a simple
case, these are the following parties.
* The security team
* The orchestration team
* The application team

Each configuration can include configuration relevant to their use case. For
example, the security team can bundle a configuration required to complete the
bare minimum of a compliance requirement. This could include proofs for access
controls, audit logging.

The orchestration team can bundle a configuration that may be required for them
to prove their responsibilities for compliance from an infrastructure
standpoint. This could include proofs for regular patches, user password policy
enforcement, network settings, package version.

The application teams can bundle configuration to implement their share of
controls, for example evidence on File integrity monitoring on sensitive files
for their application, documented production changes, continuous monitoring.

## Configuration bundling

The configuration can be distributed in 2 ways,
1. As a bundle from a static file server
2. As an API response. The API could be built over http or rpc.

### Namespace
Every configuration has 2 types of categorization, one for the OS platform and
one for a logical grouping in an organization. Both of them are ordered sets of
tokens and are connected by filepath separator `/`. For example, a set of
configurations can be created for the linux platform which can then be
categorized into flavors and the other could be organization and team name.

```
Platform: {linux, centos} -> linux/centos
Logical : {platform, messaging} -> platform/messaging
```
The above pair of ordered sets together, again joined by the unix style file
separators, form a `Namespace` the configuration.

The above example would for `linux/centos/platform/messaging`. The namespace can
be seen as a path in the filesystem tree and should not be deeper than what a
filesystem would support. For compatibility with other formats, it should be
kept under `256 Bytes` of total size.

## Configuration bundling for a static file server
This section deals with bundling the configuration for serving from a static
file server.

### Configuration tree
The configuration bundle can be visualized as a
[tree](https://en.wikipedia.org/wiki/Tree_(data_structure)), the leaf node
should always contain the configuration files that are required by the agents
and all the paths from the root should only consist of folders.  There can be
as many paths as needed. For example, this could be one of the subtrees.
```
├── linux
│  ├── centos
│  │  ├── audit.conf
│  │  └── osquery.conf
│  ├── debian
│  │  ├── audit.conf
│  │  ├── osquery.conf
│  │  └── platform
│  │     ├── audit.conf
│  │     └── osquery.conf
│  └── osquery.conf
...
```
Once this tree is constructed, all the folders are then rooted at folder named
`config`. So for the configuration file,
```sh
config/linux/debian/platform/audit.conf
```
it would be part of the `linux/debian/platform` namespace. Notice it does not
contain the root of the tree, i.e. `config`.

### Configuration lookup
For configuration selection or lookup, if, the above subtree, someone tries to
lookup, `linux/centos/platform` (which does not exist), the configuration files
at the closest path should be used, or the last node while traversing from the
tree root to the end.
A sample computation,
```py
nodes = ['linux', 'centos', 'platform']
namespace = ''

for node in nodes:
    p = namespace + '/' + node
    if not exists(p) or not isdir(p):
        return namespace
    namespace = namespace + '/' + node

# final namespace to use
print (namespace)
```

### Configuration packaging to a single bundle
The configuration once rooted at `config`, needs to be bundled into a
single blob using methods like
[tar](https://en.wikipedia.org/wiki/Tar_(computing)),
[zip](https://en.wikipedia.org/wiki/ZIP_(file_format)) or other similar methods.
Let's name the blob as `config.ext`.

This blob, `config.ext`, is then cryptographically signed using a key-pair. Any
standard signing method can be used. eg RSA with SHA256 or ECDSA with SHA256.
The signature can then be written as it is into a file named `config.sign`.

The key that is used to sign must be valid and trusted. The trust can be
expressed using the standard [X.509 PKI Certificates](
https://tools.ietf.org/html/rfc5280). The X509 Certificates must now be bundled
with the package for any party to verify the configuration.

The leaf certificate must be written into a file called `cert.pem` in the
[PEM](https://tools.ietf.org/html/rfc7468) format. This file should have exactly
one PEM block.

The intermediate certificate, **excluding the Root CA** must be written into a
file called `intermediate.pem` in the PEM format.

The order of the PEM blocks must be such that, the issuer of the leaf
certificate `leaf`, `Iss01` must be a certificate in the `intermediate.pem`.
The issuer of the certificate `Iss01`, `Iss02` must be in the `intermediate.pem`
strictly after `Iss01`. This continues for the chain. The `intermediate.pem` file
must not contain any self-signed certificate, i.e. it should not contain a
Root CA certificate.

An optional metadata file `manifest` can be created that contains the
information about
* Signing algorithm
* The format in which `config.ext` has been bundled.
* Any other metadata about the bundle.

The files, `cert.pem`, `intermediate.pem`, `manifest`, `config.sign` and
`config.ext` are then written into a top level folder, say `bundle` structured
as,
```
bundle
├── certs
│  ├── cert.pem
│  └── intermediate.pem
├── config.ext
├── config.sign
└── manifest
```
This bundle can then be distributed as any arbitrary format.

### Note on the X.509 Certificate
The certificate must have the following attributes
* A distinguished name which has access controls over it
* Extended key usage attributes should have `digitalSignature` and
  `nonRepudiation` as defined in
  [rfc3280](https://www.ietf.org/rfc/rfc3280.txt).
* Standard key size and digest algorithm can be applied to the certificate.

Any additional constraints can be added to the certificate, which must be
verified when extracting the `config.ext` bundle at the endpoint.

## Configuration bundling for an API server

The API server must expose an endpoint `/config` which at least supports a `GET`
request to fetch a configuration bundle.

The Namespace as described in [Configuration bundle#Namespace](#namespace)
section shall be used as a resource path on an HTTP url.

A `GET` request can be used to fetch the configuration in the
[JSON Web Signature](https://tools.ietf.org/html/rfc7515) format.
```
GET /config/linux/debian/platform
```

The configuration data will be present in the JWS Payload section of the JWS
response. The configuration files can be encoded in a format and specified in
the
[`cty`](https://tools.ietf.org/html/rfc7515#section-4.1.10) field of the JWS header.

Some examples of the content types.
* `application/json`: The files are present as part of the top level key and the contents of
  the file are send as JSON object.
```json
{
    "osquery.conf": {},
    "audit.conf": {}
}
```
* `application/tar`: The files can be tared at the leaf folder as the parent and
  then encoded in base64url.
* `application/zip`: Similar construction as tar.
* `application/gzip`: Similar construction as tar.

### Use of query parameters

Any additional values can be supplied using the query parameters. For example,
the configuration files can actually be templates which require certain fields
to be populated before being returned. For example, the owner information can be
supplied at fetch time and the templates can use the variables to build a
config.
```
GET /config/linux/debian/platform?owner=foo&org=messaging&env=prod
```
The server may use the query parameters,
```json
{
    "owner": "foo",
    "org": "messaging",
    "env": "prod"
}
```
to add the required metadata in the configuration. These may be mandated to
force the configuration to have ownership information if already not present.

### API server response
The API server must respond to the configuration request with appropriate http
response codes.
* 200: If the namespace is found and all required values are populated in the
  configuration, if a template.
* 400: If one of the required template values are not provided. The server can
  optionally return the missing template key.

The server must return the `Last-Modified` header for every request to the
`/config` endpoint which denotes when was the content on the URL last changed.
The user agent shall use this header to determine if the configuration needs to
be re-processed or not.

## A Sample JWS Response

For the static file server, the bundling format with the verification is quite
straight forward. For the API server, it provides much more flexibility in terms
of how the config is sent and how it's processed.

A sample JWS response that may be expected is as follows.
```
// TODO
```
