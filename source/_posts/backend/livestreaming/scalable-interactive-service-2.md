title: How to Build a Scalable Live Streaming Interactive Service - Part II
date: 2022-03-27
tags: [Backend,LiveStreaming,Live]
categories: Backend
toc: true
---

## Background

With faster networks and the push from COVID-era remote life, live streaming has seen a resurgence.

![](/uploads/persister-how-to-build-a-scalable-live-streaming-interactive-service--e6c9d24ely1h0obi03jucj20yv0u0dkg.jpg)

In Part I, we discussed signal types and interaction/connection modeling. In Part II, we’ll focus on scaling methods for interactive services.

## Limitations

In backend systems, services are often described as either stateless or stateful.

A stateless service can process a request using only the information carried with that request and does not rely on state from earlier requests.

A stateful service relies on state accumulated from earlier requests (in memory or in storage) in addition to the data carried with the current request.

This distinction matters because stateless workloads can usually be load-balanced freely, while stateful workloads often require affinity (sticky sessions, consistent hashing, sharding) and are harder to scale out.

As a result, bottlenecks often come from stateful components.

The interactive service of live streaming is composed of several parts:

- Storage
  - Relational databases
- Caches
  - Structured collections
  - Key-value caches
- Compute Services
  - Scheduled tasks
- Payment Service
  - Account transfers
- HTTP Servers
- Signaling Servers

The scaling methods discussed below largely target these components.

## Scaling Methods

Most distributed systems scale by adding instances, but that only works when each component can actually scale out.

### Relational Databases

As an internet service, relational databases are usually the source of truth. The typical scaling approach is sharding: split data into partitions and store them in different databases.

Live streaming data has a few useful characteristics that make sharding more approachable:

- Timeliness: live streams have a bounded lifetime. The longest sessions are under 480 hours, and the average duration is under 5 minutes. For online traffic, we mostly care about ongoing rooms and recent history, so we can keep only the most recent ~4 weeks in the “hot” dataset.
- Geography: users often consume streams that are geographically close or at least within the same region. That makes it natural to shard records by the host’s region.

One practical routing scheme is:

1. Generate the live streaming room ID based on the host (anchor) ID, and generate the host ID with a regional prefix (based on where the account was registered).
2. Insert the room record into the database shard that the room ID maps to.
3. When querying an ongoing room by room ID, route to the owning shard directly.
4. When querying an offline (ended) room by room ID, route to an append-only archive database that can be scaled independently by adding nodes.

![](/uploads/persister-scalable-interactive-service-2--e6c9d24ely1h0oertdxcbj216h0u0tc7.jpg)


### Structured Collections

We use Redis to implement structured collections (lists, sets, sorted sets, hashes). Scaling this layer mostly means scaling Redis itself: add more shards and spread keys across them.

In our setup, we don’t use the official `Redis Cluster` protocol. Instead, we use Twemproxy to shard reads/writes across a fleet of Redis master/replica nodes managed by Redis Sentinel. So the “Redis cluster” below means “Twemproxy + Redis + Sentinel”, not the built-in Redis Cluster feature.

We also build two identical Redis clusters in different AZs (Availability Zones) and designate one as the primary via configuration.

Writes to the primary cluster are replicated to the secondary cluster through Kafka.

![](/uploads/persister-scalable-interactive-service-2--e6c9d24ely1h0ofdhqbrcj215g0u078o.jpg)

In the diagram above, notice that `Twemproxy` is stateless. You can scale it horizontally, and it shouldn’t be the bottleneck of the Redis layer.

In practice, Redis keys are often derived from the room ID, which distributes load naturally once you have enough shards. The remaining work is to avoid creating big keys and hot keys.

Big keys often come from writes to large sorted sets and hashes. In live streaming, two practical ways to avoid big keys are:

- Trim collections to a fixed size once they exceed a threshold.
- Limit write rate with a rate limiter (or other backpressure mechanism).

Hot keys usually come from reads. To mitigate hot keys, you can:

- Add another cache layer for derived results (for example, a short-lived per-process local cache).
- Store the same logical value under multiple physical keys and choose a key randomly when reading. In most sharding schemes, those keys land on different Redis nodes, so the read load spreads out.

![](/uploads/persister-scalable-interactive-service-2--e6c9d24ely1h0olq9yfdqj21aw0u00wc.jpg)

On large social-style platforms, hotspot detection plus special-case policies (rate limits, circuit breakers, pre-warming) are also common tools.

### Key-Value Caches

We use Memcached as our key-value cache. Most values cached in Memcached are derived from the relational databases.

Unlike Redis, we don’t use a proxy in front of Memcached. Reads are routed by the client library (typically via consistent hashing).

Memcached has no built-in primary/replica role. To improve availability, we run multiple equivalent clusters for the same purpose, and treat them as peers.

![](/uploads/persister-scalable-interactive-service-2--e6c9d24ely1h0ogzrpxaej20xn0u077z.jpg)

When a client tries to read a value from Memcached, it follows a flow like this:

1. Randomly pick one cluster in the same AZ, then hash the key and route to a Memcached node.
2. Read from that node; if it’s a hit, return immediately.
3. On a miss, randomly pick another cluster in the same AZ and route to its node.
4. Read again; if it’s a hit, write the value back to the node from step 2 (to heal the miss) and then return.
5. If it’s still a miss, call a service like `DbReader` to read from the database, then write back to both nodes (step 4 and step 2).

One more detail: different clusters use different hash salts. That way, even if two clusters have the same number of nodes, “node 0” in cluster A does not tend to store the same set of keys as “node 0” in cluster B.

With this approach, availability is high:

- If one Memcached node is down, clients can still read most keys from another cluster and heal the missing copy via write-back.
- If two nodes in different clusters are down, the expected lost portion is roughly `1/M * 1/N` (where `M` and `N` are the node counts of the two clusters).

### Scheduled Tasks

Running periodic business jobs is common in live streaming backends. Many of these jobs share two properties:

- They should know which rooms are ongoing;
- Their logic can be partitioned by live room.

Ongoing live rooms are stored in the online databases, but the naïve way to find them is scanning tables. That is expensive—especially when multiple jobs and services scan concurrently.

We can use a service named “OngoingQuery” to protect the databases.

![](/uploads/persister-scalable-interactive-service-2--e6c9d24ely1h0om5ilpczj21fl0u0dka.jpg)

The OngoingQuery service stores the room IDs of ongoing live streams.

It periodically scans the database for a full refresh, and tails database binlogs for near-real-time updates. With this setup, its cache eventually converges to the database state.

When we deploy a sharded job (multiple worker processes), workers register themselves in ZooKeeper and then run periodically:

- Query ZooKeeper to learn how many worker instances are currently running.
- Determine the worker’s own index among the instances.
- Request OngoingQuery for the worker’s partition of the ongoing room list.
- Process only that partition.

With this partitioning abstraction, one logical job can be split across multiple processes and run concurrently.

### Gift Settlement

Almost every live streaming platform will support gift features.

The core of gifting is money movement: when a viewer sends a gift, balance is transferred from the viewer’s account to the host (anchor)’s account.

For scalability, user balances are often sharded across databases. A direct viewer → host transfer therefore tends to become a distributed transaction, which is expensive.

One way to reduce the cost is to introduce a virtual “room account”.

![](/uploads/persister-scalable-interactive-service-2--e6c9d24ely1h0omsn1ul4j21570u0jw6.jpg)

When a live room begins, create a virtual room account in each balance database shard.

During the live stream, gifts become transfers between the viewer’s account and the virtual room account in the same shard (a local transaction).

When the live room ends, a service like “Settler” collects all virtual room accounts and performs settlement to the host’s account (the cross-shard part happens once, in batch).

### HTTP Service

There are many load-balancing strategies for HTTP services, so I won’t spend much time on this topic.

One practical trick is to use an Nginx routing policy that sends the same room ID to the same service group, which can improve local-cache hit rate.

### Signaling Servers

We have talked about this in the [How to Build a Scalable Live Streaming Interactive Service - Part I](https://xhinliang.win/2022/01/backend/livestreaming/scalable-interactive-service-1/).

## Summary

In this article, we discussed scaling strategies for the stateful parts of a live streaming interactive service. Many people refer to these ideas simply as “sharding strategies”.

The common theme is simple: split a big thing into smaller, independent pieces.

If there is interest, I can write a follow-up on building a multi-region (or cross-region) live streaming platform.

## References

- https://en.wikipedia.org/wiki/Service_statelessness_principle#:~:text=Service%20statelessness%20is%20a%20design,their%20state%20data%20whenever%20possible.
- https://www.proud2becloud.com/stateful-vs-stateless-the-good-the-bad-and-the-ugly/
