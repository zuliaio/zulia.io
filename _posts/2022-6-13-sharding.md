---
layout: post
title: Sharding and Replication
description: How Zulia shards and replicates indexes across the cluster
date: 2022-06-13 12:00:00
---

# Sharding and Replication

## Sharding

Each Zulia index is split into a fixed number of shards (set with `numberOfShards` at index creation). Shards are distributed across the nodes in the cluster, allowing a single index to scale beyond one machine. Writes are routed to the shard that owns a document's unique id, and queries are federated across all shards and merged before being returned to the client.

### Faceting with Sharding

When sharding is enabled and when a facet contains more values then are returned, counts will represent a lower bound.  Each facet will also contains a max error for the count.  The count + max_error equals the upper bound for the count.  To reduce the error, set shard facets (setTopNShard in Java Client) higher than its default of 10 x topN or to -1 to request all for an exact count. This will request more results from each shard but not return them to the client.


## Replication

Set `numberOfReplicas` greater than `0` to keep additional copies of each shard on other nodes. Primary shards stream their committed Lucene segments to the configured replicas using a per-shard generation gate, a virtual-thread executor, a per-replica circuit breaker, and a shared bandwidth rate limiter (default 60 MiB/s). A 30-second watchdog recovers from missed checkpoints.

Replicas serve queries from their local segment copies and can take over if a primary node fails. Replicas advertise their Lucene version, and a primary refuses to ship segments to a replica whose Lucene version is older than its own (it requires `replicaVersion >= primaryVersion`), so operators can roll Lucene upgrades replica-first.

### Changing the replica count on a live index

The number of replicas is no longer fixed at index creation. It can be increased or decreased on an existing index with `UpdateIndex` (Java client) or with `zuliaadmin updateIndex --index <name> --numberOfReplicas <n>`, without recreating the index. Previously this operation threw "Cannot change replication factor for existing index yet".

### Tuning replication

Two node-level settings in `zulia.yaml` control replication transfers (see [Install]({% post_url 2022-6-14-install %}#replication)):

* `replicaResponseTimeout` - minutes allowed for a single segment-file stream to a replica (default 10).
* `replicationMaxBytesPerSec` - aggregate replication bandwidth cap leaving the node, in bytes/sec (default 60 MiB/s; `0` disables the cap).

### Monitoring replication

The `GET /replication/{indexName}` REST endpoint reports per-shard and per-replica progress: last attempted generation, replica lag, last-success timestamp, consecutive failures, and circuit-breaker state. See [Rest-Service]({% post_url 2022-6-12-rest %}#replication-rest).

