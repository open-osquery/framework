# Distributed queries

So far the framework has focussed on providing a mechanism to distribute
configuration for passive monitoring, i.e. rules are published independently and
it's up to the agent to pull the configuration and act upon it.

The configuration is intended for the regular agent and it's subcomponents to
run and not for dynamic queries which may be required only during triaging.

## Table of contents
1. [What are distributed queries](#what-are-distributed-queries)
2. [Implementing distributed queries](#implementing-distributed-queries)
    - [Components for implemention](#components-for-implemention)
3. [Additional constraints for configuration](#additional-constraints-for-configuration)
    - [Avoid replay using nonce](#avoid-replay-using-nonce)
    - [Selecting what config to send](#selecting-what-config-to-send)
## What are distributed queries
The term is directly picked up from
[osquery distributed queries](https://osquery.readthedocs.io/en/stable/deployment/remote/#distributed-queries)
which allows an operator to perform arbitrary queries and the queries are
immediately returned instead of the cron-like query mechanism for the scheduled
queries.

## Implementing distributed queries
The distributed query mechanism can be implemented in the framework by
leveraging the configuration distribution mechanism plus some additional
constraints and a custom logging pipeline.

### Components for implementation
To be able to run near-real-time queries, explicit queries must be sent at
runtime instead of being part of the regular config plugin response. This
configuration can be fetched from a different endpoint `/distributedquery` with
the identity attached to the endpoint in the same way as the `/config` endpoint.
```
GET /distributedquery/{namespace}?{query}
```
example,
```
GET /distributedquery/linux/centos?org=foo&project=bar&host=hostname&nonce=1234
```

The configuration will need to be verified in the same way as the regular config
is verified. Once the configuration is verified, The queries will be run and
results are generated. The event entries created must follow the spec for
ensuring [ownership identity](docs/agent.md#notes-on-the-ownership-identiy)

Once annotated events are generated and ready to be written, they must be
transported to the relevant / configured log collector.

## Additional constraints for configuration
The distributed queries are supposed to run once which is initiated by the
agent. This involves the agent to periodically query the config server to see if
a new config is available to run.

*A Note on the pull vs push*, one alternate way to get the configuration could
be implemented by adding a server to the extension listening for config which
would give real time query execution. There are some caveats with this approach.
* The framework does not rely or use discovery, so it would not know what
  endpoints to send the queries to
* This would require all the nodes running a webserver which would complicate
  the setup since it would require to have authentication and authorization,
  consume a port, firewall rules and other configuration for any nodes coming
  up. Opening up a port on all IPs is risky as compared to allowing connections
  from all nodes to a single destination.

The configuration sent on the distributed query endpoint may have the following
parameters,

* `sub`: The subject for the config would be `distributed_query`.
* `qry`: The query url which was send to fetch the config. More on this later in
  the document.

The distributed query must be run only once. To ensure the agent knows about
the staleness of the query without keeping track of it, a nonce may be
introduced in the configuration metadata.

### Avoid replay using nonce
To make sure the query is run only once, a nonce shall be attached with the
config request. For example,
```
GET /distributedquery/linux/centos?org=foo&project=bar&nonce=12345678
```
The nonce will be agent generated and need not be cryptographically secure as
it's only used to determine if the config is not stale.

The `nonce` sent in the query is optional and the plugin may choose to use other
attributed to detect the staleness of the query. In case the `nonce` is
supplied, the config returned should contain the same nonce as part of the `qry`
claim in the JWS. The agent, if relying on the `nonce` should compare the nonce
value as well when verifying the JWS body for config replay on top of the time
and signature based validity.

However, nonce will not allow a webserver to cache the response as the request
string changes everytime. Depending on priority for performance vs correctness,
the implementation may decide on the `nonce` usage.

### Selecting what config to send
It is possible to have a distributed query to run on a broad set of nodes which
is above the `org` or `project` classification. For a query on a resource, all
queries that belong to a super set of that resource shall be returned in the
response.

For example, let the following queries be present,
```json
{
    "linux/centos": [
        "SELECT foo FROM bar",
        "SELECT * FROM bar"
    ],
    "linux/centos?org=foo": [
        "SELECT * FROM baz"
    ]
}
```

When a request for existing query comes in for say
```
linux/centos?org=foo&project=bar&host=app1.bar.foo.com
```
all the above queries must be returned as this node belongs to both categories
or resource classifications.
