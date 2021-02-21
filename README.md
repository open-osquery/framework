# The Audit framework

## Table of Contents
1. [Introduction](docs/intro.md)
2. [Data collection agent](docs/agent.md)
3. [Configuration build and distribution](docs/config.md)
4. [Log aggregation and notification channels](docs/action.md)
5. [Distributed Queries](docs/distributed.md)

## Introduction
The [Introduction](docs/intro.md) describes the use case that the project wants
to solve. This would include the problem statement, existing tools and the
problems with them.

## Data collection agent
The [Data collection agent](docs/agent.md) document describes about what an
agent must have in order to produce the required information for the pipeline to
work.

## Configuration build and distribution
The [Configuration build and distribution](docs/config.md) document describes
how the configuration is structured, built and distributed for usage by the
agent. What are the minimal set of configuration items required in order for the
agent to determine its role and function.

## Log aggregation and notification channels
The [Log aggregation and notification channels](docs/action.md) describes
possible extensions to the log collection pipeline which can then be used for
custom reporting and alerting.
