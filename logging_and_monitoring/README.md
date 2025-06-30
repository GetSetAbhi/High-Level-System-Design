# Logging and Monitoring System

Only a couple of things will be discussed.

## Metrics Collections

There are two approaches to metrics collection, push and pull.

### Push Model

![Push Model](push_model.svg)

We have a cluster of services, which have a metrics collection agent software installed on every cluster.
Which periodically aggregate metrics on their respective clusters and forward that information to metrics collection
service. The metrics collections service, aggregates this information and stores it into timeseries DB like influx

### Pull Model

![Pull Model](pull_model.svg)

In pull model, the metrics collector pulls metric data via pre defined HTTP endpoint */metrics*
Since servers, are added/removed frequently, we need to make sure that our metrics collector is aware of these developments,
and does not miss out on any information. To do that we use zookeeper service discovery.
Every cluster from which we seek information, registers itself with zookeeper. The metrics collector service queries zookeeper
to fetch all available services and using */metrics* https endpoint, it fetches metric information from them.  