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
    - [Configuration tree](#configuration-tree)
    - [Configuration lookup](#configuration-lookup)
4. [Configuration distribution](#configuration-distribution)
5. [Configuration bundling for a static file
   server](#configuration-bundling-for-a-static-file-server)
    - [Configuration packaging to a single bundle](#configuration-packaging-to-a-single-bundle)
    - [Note on the X.509 Certificate](#note-on-the-x509-certificate)
6. [Configuration bundling for an API
   server](#configuration-bundling-for-an-api-server)
   - [Use of query parameters](#use-of-query-parameters)
   - [API server response](#api-server-response)
7. [Configuration transport and trust](#configuration-transport-and-trust)
8. [Appendix](#appendix)
   - [A Sample API Response](#a-sample-api-response)

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

The configuration can be seen as a forest where each root is a broad category,
like OS platform, and the subsequent children are sub-categories with the leaves
of the tree being actual configuration files.

### Namespace
A *namespace* is a string which represents the path between a root in the forest
to a specific leaf (configuration file(s)) excluding the leaf itself. The path
can be represented as an ordered list and the namespace can be formed out of it
by joining all the nodes with a `/` character, making it compatible with a unix
style file system and a URL path. These node identifiers should be alphanumeric
characters only, i.e. the charset `[a-zA-Z0-9]`. For compatibility with other
formats, it should be kept under `256 Bytes` of total size.

The nodes can logically represent details about the path, for example,
```
["linux", "centos", "platform", "messaging"]
```
The first 2 terms represent the OS platform and the next 2 represent
organization and team identifiers.

### Configuration tree
A single tree from the forest would look something like this.
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
The forest is then rooted virtually on a node called `config` from where all the
configuration paths would originate. The namespace can be derived visually from
the path as follows. For the file,
```sh
config/linux/debian/platform/audit.conf
```
It would be part of the `linux/debian/platform` namespace. Notice it does not
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

## Configuration distribution
The configuration can be distributed in 2 ways,
1. As a bundle from a static file server
2. As an API response. The API could be built over http or rpc.

## Configuration bundling for a static file server
This section deals with bundling the configuration for serving from a static
file server.

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
GET /config/{namespace}
```

The response from the server will be a JWS body and the content type of the
response will be [`application/jwt`](https://tools.ietf.org/html/rfc7519#section-10.3.1). The configuration data will be present in the JWS Payload section. The JWS body
can have standard claims and body.

The actual configuration will be present in a custom claim type `cfg` which
will also be a JSON object. Every top level key identifies the file name and
the corresponding JSON object, which could be a string, array or another JSON
object, identifies the content of the file. A base64 encoding may be
used to write raw data.

A sample JWS Body (base64 decoded)
```json
{
  "sub": "audit_config",
  "aud": "linux/debian/platform/messaging",
  "iat": 1516239022,
  "exp": 1516239322,
  "qry": ["owner=foo", "org=messaging", "env=prod"],
  "iss": "https://config-server.example.com",
  "cfg": {
    "osquery.conf": {
      "foo": "bar"
    },
    "audit.conf": {
      "baz": "qux"
    }
  }
}
```

### Custom claims
There are certain custom claims that will be part of the response body.
* `qry`: The query parameters that were send in the request.
* `cfg`: The JSON body that contains the contents for configuration files.

### Use of query parameters

Any additional values can be supplied using the query parameters. For example,
the configuration files can actually be templates which require certain fields
to be populated before being returned. For example, the owner information can be
supplied at fetch time and the templates can use the variables to build a
configuration.
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

The reserved query parameters are:
* `org`: The logical organization for the resource
* `project`: The project under which the resource is assigned
* `owner`: The individual owner for the resource
* `owner_dl`: The owner group for the resource
* `env` : The environment classification

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
be re-processed at the endpoint or not.

## Configuration transport and trust

The configuration can be fetched over regular HTTP or rpc provided all the
required values are supplied in the request.

The configuration fetch can be done over and insecure transport as the
configuration integrity is taken care in the payload itself via signatures.

However, there are certain validations that are strongly recommended, in some
cases necessary, before trusting the configuration.

**Precondition** - All security measures depend on the user agent trusting a
Root CA since standard PKI is being used to protect payloads. This requires the
endpoint to trust the issuing CA or the configuration should be rejected.

### Checks that must be performed before trusting the configuration
The above checks are assuming the channel is insecure.

* The Signature of the JWS body must be validated before use.
* The JWS claims must be valid with respect to `iat`, `exp` and `nbf` if
  present.
* The requested namespace must match or be a *child* of the returned
  configuration. The returned namespace is part of the `sub` claim in the JWS
  payload.
* The query parameters sent must match in the `qry` claim in the JWS body.
* The leaf certificate in the `x5c` header attribute must have required
  constraints which can include, Extended Key Usage attributes, issuers,
  Distinguished Name, validity etc.
* The certificate chain in `x5c` must be validated against the system trust.
* The `iss` attribute must match the host from which the configuration was
  requested.

If any of the checks fail, the configuration must be rejected by the user agent.

## Appendix
### A Sample API Response

For the static file server, the bundling format with the verification is quite
straight forward. For the API server, it provides much more flexibility in terms
of how the configuration is sent and how it's processed.

A sample JWS response that may be expected is as follows (newlines for clarity)
```
eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCIsImp3ayI6eyJrdHkiOiJFQyIsImNydiI6IlAtMjU2IiwieCI6InBBaWJYckE0cXRKKzhkallIemEzV25aeDBIcGFlNGRnTGNicXF4SGo5L289IiwieSI6ImsvQUZBWG5vZEVUODY5elNKSVRuRmZaVm92TnQwWTFQbjhZcVpWK3NlUFE9Iiwia2lkIjoiMSJ9LCJ4NWMiOlsiTUlJRVFqQ0NBaXFnQXdJQkFnSVVOTVFzYW9JUlpXdzVBNDMvUkRORkUrcnlzbGN3RFFZSktvWklodmNOQVFFTkJRQXdlREVMTUFrR0ExVUVCaE1DU1U0eEVqQVFCZ05WQkFnVENVSmhibWRoYkc5eVpURUxNQWtHQTFVRUJ4TUNTMEV4RlRBVEJnTlZCQW9UREc5d1pXNHRiM054ZFdWeWVURU9NQXdHQTFVRUN4TUZRWFZrYVhReElUQWZCZ05WQkFNVEdHOXdaVzR0YjNOeGRXVnllUzVsZUdGdGNHeGxMbU52YlRBZUZ3MHlNVEF4TWpBeE1EUXdNREJhRncweU1qQXhNakF4TURRd01EQmFNR294Q3pBSkJnTlZCQVlUQWxWVE1STXdFUVlEVlFRSUV3cERZV3hwWm05eWJtbGhNUll3RkFZRFZRUUhFdzFUWVc0Z1JuSmhibU5wYzJOdk1SUXdFZ1lEVlFRS0V3dGxlR0Z0Y0d4bExtTnZiVEVZTUJZR0ExVUVBeE1QZDNkM0xtVjRZVzF3YkdVdVkyOXRNRmt3RXdZSEtvWkl6ajBDQVFZSUtvWkl6ajBEQVFjRFFnQUVwQWliWHJBNHF0Sis4ZGpZSHphM1duWngwSHBhZTRkZ0xjYnFxeEhqOS9xVDhBVUJlZWgwUlB6cjNOSWtoT2NWOWxXaTgyM1JqVStmeGlwbFg2eDQ5S09CbkRDQm1UQU9CZ05WSFE4QkFmOEVCQU1DQmFBd0hRWURWUjBsQkJZd0ZBWUlLd1lCQlFVSEF3RUdDQ3NHQVFVRkJ3TUNNQXdHQTFVZEV3RUIvd1FDTUFBd0hRWURWUjBPQkJZRUZINXlkakx2QW1tNnJ4UDhEOGdZeWpPcE0wNytNQjhHQTFVZEl3UVlNQmFBRkpEMms1aURTbWFhNnBDWDRPelNnZ1dBZzdhQk1Cb0dBMVVkRVFRVE1CR0NEM2QzZHk1bGVHRnRjR3hsTG1OdmJUQU5CZ2txaGtpRzl3MEJBUTBGQUFPQ0FnRUFmYUo3TEZBZk02MFA0OXZoOHJSNURsMVdIRTlzUTVPQS9oK2xTbHZsNHI0RWN1bkFySytrRmMxdTR0OHpFTy9GcnpQbUx3RmtaMG42K0FCM00ySFJEaFJpaHloWkNNTFQ4LzhTMkJGWGp2V3VtSDJhVXJuK1R3ZHc0NXh5OGl3aVdVRWxwaVo4NGdBSzZLQnhudGtWUjd3N283OWdjd09ZNHV0TTRhVkhielpPcU9lMUJrS2wrOWwreElHeEx2TUtFVitDOHBDK3g1SjB3MGNWWUl5clhJZWtMbGsvUkIrZm0zMFNKd243TnRRUkhjL2FxMlhZMlpqbnNaZ1ZaRUliNGNaRC9JWFZodkdKcHBSR09RQnp3dURlK1hzVVY3RjMyaVRscTk2SWRsbWhhdzdrRGE5MlF0RlJ5TUhGZlJBNnBIUWhMMmhQMlBseXR0Skc0S0dpN0xmWFd0YTNtS1pzaUZvUDJCZm96aWpLT0g2TVBrRUpSYzVmRWo0Rzc0dGE3SlprZ0FtVVZzSVBXYTlmcHI2VTJhYlBUNVJMVHd0WVJsQmgwZEpZR3hVN1lyNDUzWHlCTHRNdUdZOXlha29pdDNzSnlnb0xKQjlVaHh5Mk9OWDVHVDNzV2FUWkhjc0s3YXZjSHZZZjlBZ1NwbGlQK0sxZFFMdTg5dDJvMnlwSU5HaEdEekdyeXdZL1VDZTdzL1RTd1JXMldqRHBNSHJGK2taUW1WazhSemFsVi9RMjhEbTdXY2Y2YmN4MmJHMUZRSzlpaXpwTmh3T3pKT0lUU0xEUHczVWc5YUxkcWZYYmRiY2FkNTZOM09zU1J5dVBHdEF2cDQ1SHg1TlNvV0dGbjNxV21xOE8vdEo5U1h6N2NsQ3lGQVk5YUxFdzc0TldJeGpXTjZFLzB6ND0iXX0
.
eyJzdWIiOiJhdWRpdF9jb25maWciLCJhdWQiOiJsaW51eC9kZWJpYW4vcGxhdGZvcm0iLCJpYXQiOjE1MTYyMzkwMjIsImlzcyI6Imh0dHBzOi8vY29uZmlnLXNlcnZlci5leGFtcGxlLmNvbSIsImNmZyI6eyJvc3F1ZXJ5LmNvbmYiOnsib3B0aW9ucyI6eyJsb2dnZXJfcGx1Z2luIjoiZmlsZXN5c3RlbSwgc3lzbG9nIiwic2NoZWR1bGVfc3BsYXlfcGVyY2VudCI6MTB9LCJwbGF0Zm9ybSI6ImxpbnV4Iiwic2NoZWR1bGUiOnsicHJvY2Vzc19ldmVudHMiOnsicXVlcnkiOiJTRUxFQ1QgYXVpZCwgY21kbGluZSwgY3RpbWUsIGN3ZCwgZWdpZCwgZXVpZCwgZ2lkLCBwYXJlbnQsIHBhdGgsIHBpZCwgdGltZSwgdWlkIEZST00gcHJvY2Vzc19ldmVudHM7IiwiaW50ZXJ2YWwiOjEwLCJkZXNjcmlwdGlvbiI6IlByb2Nlc3MgZXZlbnRzIGNvbGxlY3RlZCBmcm9tIHRoZSBhdWRpdCBmcmFtZXdvcmsifSwiZmlsZV9ldmVudHMiOnsicXVlcnkiOiJTRUxFQ1QgKiBGUk9NIGZpbGVfZXZlbnRzOyIsImludGVydmFsIjoxMCwicmVtb3ZlZCI6ZmFsc2V9fSwiZmlsZV9wYXRocyI6eyJjb25maWd1cmF0aW9uIjpbIi9ldGMvcGFzc3dkIiwiL2V0Yy9zaGFkb3ciLCIvZXRjL2xkLnNvLnByZWxvYWQiLCIvZXRjL2xkLnNvLmNvbmYiLCIvZXRjL2xkLnNvLmNvbmYuZC8lJSIsIi9ldGMvcGFtLmQvJSUiLCIvZXRjL3Jlc29sdi5jb25mIiwiL2V0Yy9yYyUvJSUiLCIvZXRjL215LmNuZiIsIi9ldGMvbW9kdWxlcyIsIi9ldGMvaG9zdHMiLCIvZXRjL2hvc3RuYW1lIiwiL2V0Yy9mc3RhYiIsIi9ldGMvY3JvbnRhYiIsIi9ldGMvY3JvbiUvJSUiLCIvZXRjL2luaXQvJSUiLCIvZXRjL3JzeXNsb2cuY29uZiJdLCJiaW5hcmllcyI6WyIvdXNyL2Jpbi8lJSIsIi91c3Ivc2Jpbi8lJSIsIi9iaW4vJSUiLCIvc2Jpbi8lJSIsIi91c3IvbG9jYWwvYmluLyUlIiwiL3Vzci9sb2NhbC9zYmluLyUlIl19fX19
.
vF4OF6k4-ojbS5UpFfiuG44DI0yiS8Gv3ENWTZX-g7_9cK2WaY5Q6efMDOijdZu2Cmjg8EqbZvvxqZQXdFUZhA
```

The payload decodes into
#### The JOSE header
```json
{
  "alg": "ES256",
  "typ": "JWT",
  "jwk": {
    "kty": "EC",
    "crv": "P-256",
    "x": "pAibXrA4qtJ+8djYHza3WnZx0Hpae4dgLcbqqxHj9/o=",
    "y": "k/AFAXnodET869zSJITnFfZVovNt0Y1Pn8YqZV+sePQ=",
    "kid": "1"
  },
  "x5c": ["MIIEQjCCAiqgAwIBAgIUNMQsaoIRZWw5A43/RDNFE+ryslcwDQYJKoZIhvcNAQENBQAweDELMAkGA1UEBhMCSU4xEjAQBgNVBAgTCUJhbmdhbG9yZTELMAkGA1UEBxMCS0ExFTATBgNVBAoTDG9wZW4tb3NxdWVyeTEOMAwGA1UECxMFQXVkaXQxITAfBgNVBAMTGG9wZW4tb3NxdWVyeS5leGFtcGxlLmNvbTAeFw0yMTAxMjAxMDQwMDBaFw0yMjAxMjAxMDQwMDBaMGoxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQHEw1TYW4gRnJhbmNpc2NvMRQwEgYDVQQKEwtleGFtcGxlLmNvbTEYMBYGA1UEAxMPd3d3LmV4YW1wbGUuY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEpAibXrA4qtJ+8djYHza3WnZx0Hpae4dgLcbqqxHj9/qT8AUBeeh0RPzr3NIkhOcV9lWi823RjU+fxiplX6x49KOBnDCBmTAOBgNVHQ8BAf8EBAMCBaAwHQYDVR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMAwGA1UdEwEB/wQCMAAwHQYDVR0OBBYEFH5ydjLvAmm6rxP8D8gYyjOpM07+MB8GA1UdIwQYMBaAFJD2k5iDSmaa6pCX4OzSggWAg7aBMBoGA1UdEQQTMBGCD3d3dy5leGFtcGxlLmNvbTANBgkqhkiG9w0BAQ0FAAOCAgEAfaJ7LFAfM60P49vh8rR5Dl1WHE9sQ5OA/h+lSlvl4r4EcunArK+kFc1u4t8zEO/FrzPmLwFkZ0n6+AB3M2HRDhRihyhZCMLT8/8S2BFXjvWumH2aUrn+Twdw45xy8iwiWUElpiZ84gAK6KBxntkVR7w7o79gcwOY4utM4aVHbzZOqOe1BkKl+9l+xIGxLvMKEV+C8pC+x5J0w0cVYIyrXIekLlk/RB+fm30SJwn7NtQRHc/aq2XY2ZjnsZgVZEIb4cZD/IXVhvGJppRGOQBzwuDe+XsUV7F32iTlq96Idlmhaw7kDa92QtFRyMHFfRA6pHQhL2hP2PlyttJG4KGi7LfXWta3mKZsiFoP2BfozijKOH6MPkEJRc5fEj4G74ta7JZkgAmUVsIPWa9fpr6U2abPT5RLTwtYRlBh0dJYGxU7Yr453XyBLtMuGY9yakoit3sJygoLJB9Uhxy2ONX5GT3sWaTZHcsK7avcHvYf9AgSpliP+K1dQLu89t2o2ypINGhGDzGrywY/UCe7s/TSwRW2WjDpMHrF+kZQmVk8RzalV/Q28Dm7Wcf6bcx2bG1FQK9iizpNhwOzJOITSLDPw3Ug9aLdqfXbdbcad56N3OsSRyuPGtAvp45Hx5NSoWGFn3qWmq8O/tJ9SXz7clCyFAY9aLEw74NWIxjWN6E/0z4="]
}
```

#### The JWS Payload
```json
{
  "sub": "audit_config",
  "aud": "linux/debian/platform",
  "iat": 1516239022,
  "iss": "https://config-server.example.com",
  "cfg": {
    "osquery.conf": {
      "options": {
        "logger_plugin": "filesystem, syslog",
        "schedule_splay_percent": 10
      },
      "platform": "linux",
      "schedule": {
        "process_events": {
          "query": "SELECT auid, cmdline, ctime, cwd, egid, euid, gid, parent, path, pid, time, uid FROM process_events;",
          "interval": 10,
          "description": "Process events collected from the audit framework"
        },
        "file_events": {
          "query": "SELECT * FROM file_events;",
          "interval": 10,
          "removed": false
        }
      },
      "file_paths": {
        "configuration": [
          "/etc/passwd",
          "/etc/shadow",
          "/etc/ld.so.preload",
          "/etc/ld.so.conf",
          "/etc/ld.so.conf.d/%%",
          "/etc/pam.d/%%",
          "/etc/resolv.conf",
          "/etc/rc%/%%",
          "/etc/my.cnf",
          "/etc/modules",
          "/etc/hosts",
          "/etc/hostname",
          "/etc/fstab",
          "/etc/crontab",
          "/etc/cron%/%%",
          "/etc/init/%%",
          "/etc/rsyslog.conf"
        ],
        "binaries": [
          "/usr/bin/%%",
          "/usr/sbin/%%",
          "/bin/%%",
          "/sbin/%%",
          "/usr/local/bin/%%",
          "/usr/local/sbin/%%"
        ]
      }
    }
  }
}
```

### The Signature
```
bgBs5rdsi8Pi0qpjZ2cQA55Tf4tOZKIywfZkkZg3PsgMp-IVi1dKNX3Du0iSu7UJf9NsiivJt57ZHTErikBhiw
```
This can be verified at [https://jwt.io](https://jwt.io) using the following,
```
-----BEGIN EC PRIVATE KEY-----
MHcCAQEEIAJhT4Kq2Q+9Q6zVNoRkhQVa6EbxqYh+FS/TpBGO6/JtoAoGCCqGSM49
AwEHoUQDQgAEpAibXrA4qtJ+8djYHza3WnZx0Hpae4dgLcbqqxHj9/qT8AUBeeh0
RPzr3NIkhOcV9lWi823RjU+fxiplX6x49A==
-----END EC PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
MIIEQjCCAiqgAwIBAgIUNMQsaoIRZWw5A43/RDNFE+ryslcwDQYJKoZIhvcNAQEN
BQAweDELMAkGA1UEBhMCSU4xEjAQBgNVBAgTCUJhbmdhbG9yZTELMAkGA1UEBxMC
S0ExFTATBgNVBAoTDG9wZW4tb3NxdWVyeTEOMAwGA1UECxMFQXVkaXQxITAfBgNV
BAMTGG9wZW4tb3NxdWVyeS5leGFtcGxlLmNvbTAeFw0yMTAxMjAxMDQwMDBaFw0y
MjAxMjAxMDQwMDBaMGoxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlh
MRYwFAYDVQQHEw1TYW4gRnJhbmNpc2NvMRQwEgYDVQQKEwtleGFtcGxlLmNvbTEY
MBYGA1UEAxMPd3d3LmV4YW1wbGUuY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcD
QgAEpAibXrA4qtJ+8djYHza3WnZx0Hpae4dgLcbqqxHj9/qT8AUBeeh0RPzr3NIk
hOcV9lWi823RjU+fxiplX6x49KOBnDCBmTAOBgNVHQ8BAf8EBAMCBaAwHQYDVR0l
BBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMAwGA1UdEwEB/wQCMAAwHQYDVR0OBBYE
FH5ydjLvAmm6rxP8D8gYyjOpM07+MB8GA1UdIwQYMBaAFJD2k5iDSmaa6pCX4OzS
ggWAg7aBMBoGA1UdEQQTMBGCD3d3dy5leGFtcGxlLmNvbTANBgkqhkiG9w0BAQ0F
AAOCAgEAfaJ7LFAfM60P49vh8rR5Dl1WHE9sQ5OA/h+lSlvl4r4EcunArK+kFc1u
4t8zEO/FrzPmLwFkZ0n6+AB3M2HRDhRihyhZCMLT8/8S2BFXjvWumH2aUrn+Twdw
45xy8iwiWUElpiZ84gAK6KBxntkVR7w7o79gcwOY4utM4aVHbzZOqOe1BkKl+9l+
xIGxLvMKEV+C8pC+x5J0w0cVYIyrXIekLlk/RB+fm30SJwn7NtQRHc/aq2XY2Zjn
sZgVZEIb4cZD/IXVhvGJppRGOQBzwuDe+XsUV7F32iTlq96Idlmhaw7kDa92QtFR
yMHFfRA6pHQhL2hP2PlyttJG4KGi7LfXWta3mKZsiFoP2BfozijKOH6MPkEJRc5f
Ej4G74ta7JZkgAmUVsIPWa9fpr6U2abPT5RLTwtYRlBh0dJYGxU7Yr453XyBLtMu
GY9yakoit3sJygoLJB9Uhxy2ONX5GT3sWaTZHcsK7avcHvYf9AgSpliP+K1dQLu8
9t2o2ypINGhGDzGrywY/UCe7s/TSwRW2WjDpMHrF+kZQmVk8RzalV/Q28Dm7Wcf6
bcx2bG1FQK9iizpNhwOzJOITSLDPw3Ug9aLdqfXbdbcad56N3OsSRyuPGtAvp45H
x5NSoWGFn3qWmq8O/tJ9SXz7clCyFAY9aLEw74NWIxjWN6E/0z4=
-----END CERTIFICATE-----
```
