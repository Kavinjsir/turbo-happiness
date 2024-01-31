## Overview

The Broker is the process to route queries to if you want to run a distributed cluster.
It understands the metadata published to ZooKeeper about what segments exist on what processes and routes queries such that they hit the right processes.
This process also merges the result sets from all of the individual processes together.
On start up, Historical processes announce themselves and the segments they are serving in ZooKeeper.

### Running

```
org.apache.druid.cli.Main server broker
```

### Forwarding queries

Most Druid queries contain an interval object that indicates a span of time for which data is requested.
Likewise, Druid Segments are partitioned to contain data for some interval of time and segments are distributed across a cluster.
Consider a simple datasource with 7 segments where each segment contains data for a given day of the week.
Any query issued to the datasource for more than one day of data will hit more than one segment.
These segments will likely be distributed across multiple processes, and hence, they query will likely hit multiple processes.

To determin which processes to forward queries to, the Broker process first builds a view of the world from information in ZooKeeper.
ZooKeeper maintains information about Historical and streaming ingestion Peon processes and the segments they are serving.
For every datasource in ZooKeeper, the Broker process builds a timeline of segments and the process that serve them.
When queries are received for a specific datasource and interval,
The Broker process performs a lookup into the timeline associated with the query datasource for the query interval and retrieves the processes that contain data for the query.
The Broker process then forwards down the query to the selected processes.

### Caching

Broker processes employ a cache with an LRU cache invalidation strategy.
The Broker cache stores per-segment results.
The cache can be local to each Broker process or shared across multiple processes using an external distributed cache such as memcached.
Each time a broker process receives a query, it first maps the query to a set of segments.
A subset of these segment results may already exist in the cache and the results can be directly pulled from the cache.
For any segment results that do no exist in the cache, the broker process will forward the query to the Historical processes.
Once the Historical processes return their results, the Broker will store those results in the cache.
Real-time segments are never cached and hence requests for real-time data will always be forwarded to real-time processes.
Real-time data is perpetually changing and caching the results would be unreliable.
