title: How to Build a Scalable Live Streaming Interactive Service - Part II
date: 2022-03-27
tags: [Backend,LiveStreaming,Live]
categories: Backend
toc: true
---

## Background

Because of the networking improvement and the influent of the COVID-19, live streaming has become the hottest technology in the Internet, again.

![](/uploads/persister-how-to-build-a-scalable-live-streaming-interactive-service--e6c9d24ely1h0obi03jucj20yv0u0dkg.jpg)

In the past article we have discuss about the signal modeling and the interactive modeling, this time we will discuss about the scale methods of interactive service.

## Limitations

As we known, there are simply two kinds of services in the backend, called stateless service and stateful service.

Stateless service means that the service processes requests based only on information relayed with each request and doesn’t rely on information from earlier requests

Stateful service means that the service processes requests based on the information relayed with each request and information stored from earlier requests.

The biggest difference between stateless service and stateful service is that stateless service can route requests to different instances easily and the stateful service can't.

The stateless services are easy to scale and the bottlenecks of entire system are commonly the stateful parts of it.

The interactive service of live streaming is composed by a few parts.

- Storages
  - Relational Databases
- Caches
  - Structured Collections
  - K-V Caches
- Caculating Services
  - Scheduled Tasks
- Paying Service
  - Account Transporting
- HTTP Servers
- Signaling Servers

And today the scaling methods of interactive service are aiming on these parts, too.

## Scaling Methods

Distrubuted system is offen constructed by a number of small instance, we can increase the capacity and performance by scale the instance count. But before scaling it up we have to make our system scalable.

### Relational Databases

As a internet service, we always use a relational database for data storage. We can split our data into different parts so that we can store them in different databases so that we can scale our database up.

We can treat our live streaming databases as this way, too. But the data of live streaming has some special characteristics.

- Timeliness. For now, the longest duration of live streaming is less than 480 hours, the avarege duration is less than 5 minutes. Mostly we only care about the ongoing live streaming rooms, so that we can only storage the recent 4 weeks data for online service.
- Geographicality. People would like to consume the live streaming related to them. So we can simply split the live streaming record by the region of anchor.

First, we generate the live streaming room id by the anchor id, which is genearted by the region they sign up.

And than insert the record into the database the live streaming room id belongs to.

When we are querying the database by ongoing live streaming room id, we can route the request to the actual database by the id.

When we are querying the database by offline live streaming room id, we can just route it into archiving database which is append-only and can  be easily extended by adding nodes.

![](/uploads/persister-scalable-interactive-service-2--e6c9d24ely1h0oertdxcbj216h0u0tc7.jpg)


### Structured Collections

We use Redis to implement the Structured Collections, so the method to scale the Structured Collections is to scale the Redis Servers.

We never use `Redis Cluster` technology as our `Redis Cluster` solution. Conversely we just use use twemproxy to reroute the write and query. So the `Redis Cluster` we talk about below is not as same as the offical `Redis Cluster` but the cluster constructed by several Twemproxy, Redis-Master, Redis-Slave and Redis-Sentinal.

On the other hand, we offen build two same Redis Cluster in different AZ(Availible Zone), and assign one of them as the main cluster by config.

The write operation of main cluster would replica to secondary claster by Kafka Message.

![](/uploads/persister-scalable-interactive-service-2--e6c9d24ely1h0ofdhqbrcj215g0u078o.jpg)

In the image above, We can notice that the `Twemproxy` are the stateless service so it's scalable and will not be the bottleneck of the Redis Cluster.

In fact, the key of Redis storage is offen constructed by the room id, after scale the redis server up, all we have to do is pervent building big-key and hot-key of Redis.

Big-key offen comes from the write operation of sorted set keys and hash keys. In live streaming scenario, there are two effective methods to avoid big-key.

- Trim the collection to a fixed length when the collection is bigger than the threshold.
- Limit the writing speed by rate-limiter or some other method.

Hot-Key offen comse from the read operation, in order to prevent it, we can use these ways below.

- Use another cache to storage the result temporarily. For example, we can build a local-cache in our service and this local-cache will serve all the request before expired
- Storage the key redundantly in different key and read a key randomly. In general Redis Cluster, different key will locate in diffenent Redis-Server, so that all the request will not be routed in a certain Redis-Server, too

![](/uploads/persister-scalable-interactive-service-2--e6c9d24ely1h0olq9yfdqj21aw0u00wc.jpg)

Also, in the social platform, detecting hotspot  and use a seperate poliy to limit writing and reading rate is very common.

### K-V Caches

We use Memcached as our K-V Caches. Which "K-V" of K-V Caches is always come from the databases.

Different to Redis, we use no proxy in Memcached. All the read is routed by the client.

There are no master and slave in Memacached, so we built some several cluster for the same usage. All the cluster are equal.

![](/uploads/persister-scalable-interactive-service-2--e6c9d24ely1h0ogzrpxaej20xn0u077z.jpg)

When the client try to read a cache from Memcached, it will have a few steps.

1. Choose a cluster of its AZ randomly, and then calculate the hash number and route to a Memcached Node by this hash number;
2. Try to read this Memacached Node, return immediately if success;
3. Choose another cluster of its AZ randomly, and route to another Memcached Node by another hash number;
4. Try to read this Memacached Node. Firstly write back to the Memcached Node in step.2 and then return;
5. Call a service named `DbReader` to read database and then write back to the Memcached Nodes of step.4 and step.2.

One more thing worth mentioning is that the hash salts of different Memcached Cluster are different, so that the same index Memacached Node of different Memacached Cluster will not storage the values of the same keys.

In this way to cache, we will have the highest availablity.

- If one Memcached Node down, no cached will be lost, because clients will read the losted keys from another Memcached Cluster and write back;
- If two Memcached Node of different Memcached Clusters down, only 1/M * 1/N data will be lost. (M, N means the nodes count of the Memcached Clusters)

### Scheduled Tasks

Processing some bussiness periodly is very comonn live streaming backend. Most of the process task do have these properties.

- They should know which rooms are ongoing;
- They would do the logic seperate by the live streaming room.

As we known, the ongoing live streaming room is storage in the online databases, we could only query the ongoing live streaming room ids by scan these tables, it would be a extravagant operation, expecialy for lots of business and process doing these concurrently.

We can use a service named "OngoingQuery" to protect the daatabases.

![](/uploads/persister-scalable-interactive-service-2--e6c9d24ely1h0om5ilpczj21fl0u0dka.jpg)

The OngingQuery service scan the databases periodly for update the entire data, and they subscribe the binlog of databases for instant update.

In this way, the OngoingQuery service will store all the room ids of ongoing live streaming.

When we deploy an amount of shard tasks for processing some business data, the shard tasks will register themselives to ZooKeeper at first, and then run periodly to do these things.

- Query the ZooKeeper to know how many process instances at this moment;
- Find out the index of itself of all the process instances;
- Request Ongoinng Query service to get the part of the partition result;
- Process the business of the part.

With these partition abstraction, we can seperate the process task to several process instances and they can run concurrently.

### Account Transporting

Almost every live streaming platform will support gift feature.

The key content of live streaming gift is account transporting. When a audience pay a gift for the anchor, there will be a account transporting between the account of audience and the account of anchor.

As we known, pltform offen storage the balance of different user in different databases for scalability. So the account transporting in live streaming will always be a distributed transaction, which is very expensive.

We can use a virtual account to improve the transporting performance.

![](/uploads/persister-scalable-interactive-service-2--e6c9d24ely1h0omsn1ul4j21570u0jw6.jpg)

When the live streaming room begin, we will create an virtual account for this live streaming room in each databases of user balance.

The gifting operation during the live streaming will be a transportion between audience account and virtual account.

When the live stream room end, the service named "Settler" will collect all the virtual account and do the distributed transation between virtual account and the anchor account.

### HTTP Service

There a lots of load-balancing strategy for HTTP service, so I will not talk about this topic a lot.

The only thing I want to mention is that we can use a route policy in Nginx, for routing same room id to a same group of service nodes, that would help for us to increate the hit-rate of local-cache.

### Signaling Servers

We have talked this in the [How to Build a Scalable Live Streaming Interactive Service - Part I](https://xhinliang.win/2022/01/backend/livestreaming/scalable-interactive-service-1/).

## Summary

In this article, we have discussed the scale strategies of the stateful services of live streaming interactive service. Some people also called these strategies "sharding strategy".

All of these things have a core concept commonly -- "Split the big thing into small things", oh we have learn it in our university, isn't it?

Next time I will share my thoughts on building a multi-region or cross-region live streaming platform, hope you will like it.

## References

- https://en.wikipedia.org/wiki/Service_statelessness_principle#:~:text=Service%20statelessness%20is%20a%20design,their%20state%20data%20whenever%20possible.
- https://www.proud2becloud.com/stateful-vs-stateless-the-good-the-bad-and-the-ugly/