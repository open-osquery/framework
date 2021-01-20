# Data Collection Agents

## Table of contents
1. [Introduction](#introduction)
2. [Basic feature set](#basic-feature-set)
3. [Notes on the ownership identity](#notes-on-the-ownership-identiy)
4. [Reporting Capabilities](#reporting-capabilities)
5. [Mandatory events to ensure integrity](#mandatory-events-to-ensure-integrity)
6. [Appendix](#appendix)

## Introduction

The framework allows multiple agents that can be used to collect and or
structure the information. There aren't a lot of constraints for the agent as
the information required is the primary driver. Instead of that, what this
document will focus on is features that would allow an agent or a group of
agents to fulfill other requirements of the framework, some of them being,
tamper detection and coherent log formating and filtering.

## Basic feature set

The agent(s) should be able to implement the following features

1. Ability to collected static information about the compute. This would include
   the state of the hardware, OS, network, packages etc.
2. Ability to listen to events emitted by the OS and hardware. This would
   include audit events, sensor events, file events, user events, process events
   etc.
3. Ability to read configuration from a local or remote endpoint, i.e. should
   include http capabilities.
4. Ability to configure the kernel to emit tailored events. This would include
   and require additional capabilities to be able to listen and write to privileged
   sockets, configure kernel to install callbacks on different kinds of events
   etc.
5. Ability to cryptographically verify a piece of data.
6. Ability to annotate data collected with a set of constants that may be
   fetched at initialization or runtime.
7. Ability to query the orchestrator to get ownership information and identity.

## Notes on the ownership identity

One of the main requirements for the audit framework is to be able to annotate
every event with sufficient information for identity and source such that an
event can always be traced back to an owner and a hardware (which could be
ephemeral).

The fields that are mandatory to be present in the annotated event are

* **owner**: An _owner_ describes an identification for an individual who is
  responsible for the resource and should be assigned any responsibility for
  triaging.
* **owner_group**: An _owner_group_ defines a group of people who should be
  informed as a fallback to the _owner_ not be reachable.
* **org**: An _org_ describes an indentification for the highest division in an
  organization under which the resource lies.
* **project**: A _project_ describes an indentification for an initiative or
  project under which the resource lies.
* **hostname**: The _hostname_ defines the dynamic identity of the resource, for
  example, it's entry in the DNS server or a friendly name.
* **ip**: The _ip_ defines the IP addresses of the exposed interfaces which can
  be used to reach a resource.
* **Platform**: The _platform_ fields describes metadata about the resource, for
  example, OS flavor and version, kernel version and patch.

## Reporting capabilities

The information collected by the agent must be filtered based on the
configuration and then reported periodically. The framework should be able to
detect events and changes. This may not exclusively be implemented on the agent
but anywhere in the pipeline.

The agents should implement a push mechanism which would make sure the data
collected by the agent is emitted without any external input.

## Mandatory events to ensure integrity

The agents must be utilizing a configuration. The agents must have a mechanism
to report the following information periodically.

* **configuration version**: The configuration version defines a unique
  identifer which can be used to determine the configuration that was running at
  a point in time accurately.

* **agent version**: The agent must have a mechanism to report what version it's
  running to detect the state of the agent a point in time.

* **agent hashes**: The agent must have a mechanism to report any change in the
  integrity of any artefact present as part of the audit framework
  implementation. A standard hashing could be used, eg SHA256 as the goal is to
  only detect breach in integrity.

* **configuration**: The agent can have
  [multiple layers of configuration](config.md#the-layers-of-configuration).
  Any configuration that is established at runtime and is not part of the
  bootstrap shall be reported periodically.

* **heartbeats**: The agent(s) must have a way to determine if the are running.
  This shall be achieved by sending the uptime of the agent. Any reset in the
  agent uptime shall mark an event and should be triaged.

## Appendix

One of the possible and existing implementations of the above spec is
[osquery](https://github.com/osquery/osquery) paired with an extension that can
take care of getting the configuraton as described in the [Configuration
bundling and distribution](config.md). A sample event from osquery would look
like.
```json
{
  "name": "process_events",
  "hostIdentifier": "devserver",
  "calendarTime": "Mon Jan 11 19:46:40 2021 UTC",
  "unixTime": 1610394400,
  "epoch": 0,
  "counter": 0,
  "numerics": false,
  "decorations": {
    "config_version": "20201217074124-108254d",
    "host_uuid": "984BADFA-E708-A54F-8A88-E9B7BDDC9B2B",
    "hostname": "devserver",
    "ip": ["192.168.56.102", "10.0.3.15"],
    "namespace": "linux/centos/test",
    "org": "open-osquery",
    "os": "CentOS Linux CentOS Linux release 7.9.2009 (Core)",
    "owner": "foo@example.com",
    "owner_ad": "prateek",
    "project": "framework",
    "version": "4.5.1"
  },
  "columns": {
    "atime": "1610021140",
    "auid": "1000",
    "btime": "0",
    "cmdline": "sudo auditctl -l",
    "ctime": "1608658510",
    "cwd": "\"/home/prateek\"",
    "egid": "1000",
    "euid": "0",
    "fsgid": "1000",
    "fsuid": "0",
    "gid": "1000",
    "mode": "0104111",
    "mtime": "1601487748",
    "owner_gid": "0",
    "owner_uid": "0",
    "parent": "1429",
    "path": "/usr/bin/sudo",
    "pid": "1721",
    "sgid": "1000",
    "suid": "0",
    "syscall": "execve",
    "time": "1610026268",
    "uid": "1000",
    "uptime": "5207"
  },
  "action": "added"
}
```
The above event has information about identity of the resource as well as the
ownership of the resource.
