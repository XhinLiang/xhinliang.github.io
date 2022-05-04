title: How to Build a Scalable Live Streaming Interactive Service - Part I
date: 2022-1-15
tags: [Backend,LiveStreaming,Live]
categories: Backend
toc: true
---

## Background

Because of the networking improvement and the influent of the COVID-19, live streaming has become the hottest technology on the Internet, again.

![](/uploads/persister-how-to-build-a-scalable-live-streaming-interactive-service--e6c9d24ely1h0obi03jucj20yv0u0dkg.jpg)

Fortunately, I have been participating in building a live streaming platform. In other words, I also have some experience in this domain. Today I am gonna talk about the "Interactive Service" of live streaming, my most familiar part of the platform.

At first, I have to define the "Interactive Service". Commonly, people always split the live streaming platform into two parts: "Video Steaming" and "Interactive Service".

![](/uploads/persister-how-to-build-a-scalable-live-streaming-interactive-service--e6c9d24ely1h0oblby6l5j20z60u0tbv.jpg)

"Video Streaming" means the audio and video that people can instantly watch and hear. "Video Streaming" is the bastion of live streaming, we can't see anything and hear any word from the anchor without it, the live streaming totally becomes a boring group chat.

"Interactive Service" means all the other parts without "Video Streaming". Almost everything you can join to the live streaming is the result of "Interactive Service". For example, comments, gifts, e-commerce notifications, etc. "Interactive Service" gives live streaming a soul so that live streaming is never a monologue of the anchor, it's a party between the anchor and all of the audiences.

## Interactive Service

In my opinion, "Interactive Service" is a full-feature eco-system.

Anchors and audiences can generate signals to it, and it would deliver the signals to the other people in the live room. For example, the audience foo can comment on a signal "You look good" to the "Interactive Service", and "Interactive Service" will broadcast this signal to the other people after a while.

On the other hand, "Interactive Service" can generate some event by itself. For example, "Interactive Service" will broadcast the exact number of users online of every live room by period, so that people can see how many people do the live room has now.

How to build an "Interactive Service" to deliver such signals in a safe, quick, and economic way, are the core parts we discuss in this article.

- Safe
  - "Interactive Service" should check the authority of the request/connection
  - Client should not receive any signal not belonging to them
- Quick
  - "Interactive Service" should deliver any signal in a certain and short duration no matter how many people are watching this live streaming
- Economic
  - "Interactive Service" should have the ability to serve a big live room that has a huge number of audiences
  - "Interactive Service" should use fewer resources, less CPU, RAM, disk, networking bandwidth, etc.

## Signal Types

Before we discuss the different modelings of "Interactive Service", we should talk about 3 types of signals whiches should implement in different ways.

### State Sync Signal

The clients should init some state just after entering the live room. When the states have been changed, clients should be noticed and do something about these changes.

A modal implementation is using "Total and Partial Version" to describe state changes of the business state updating of live streaming.

In this implementation, servers and clients should store the total version and partial versions. Clients should fetch the total version by interval and then fetch the outdated partial versions if the fetched total version is different from the local total version.

![](/uploads/persister-how-to-build-a-scalable-live-streaming-interactive-service--e6c9d24ely1h0oc8bkeruj21d90u078y.jpg)

We call this model `SS Signal` for a shortcut.

### Time Series Signal

Time Series Signal is a temporary signal in this live room. The time-series signals that happened before the client entered would not be sent to this client, because the missing of some time-series signals should not affect the experience.

We can use several ways to implement time-series signals. But there are two different routes to them.
- Using a Messaging System, such as Redis Pub/Sub or Kafka Messaging
- Use a Time Series Database, and fetch them periodically

![](/uploads/persister-how-to-build-a-scalable-live-streaming-interactive-service--e6c9d24ely1h0oc8tju01j219g0u0djh.jpg)

We call this model `TS Signal` for a shortcut.

### Peer Delivery Signal

SS signal and Ts signal are designed for all of the people in the same live room. When one of the state change or some action happen, these signals should be delivered to all of the clients in these live room. But peer signals are designed for just some of the people in the live room.

We could use signals with some filters to implement peer signals, but it will have lots of performance issues.

A more effective way to treat the peer signal of live streaming as a normal signal on the Instant Messaging System. With a typical Instant Messaging System, we can find which server is the target client connecting, so that sending it signals is efficient.

## Connection Modeling

Servers and the clients should have a way to communicate, called connection modeling. There are different ideas for implementing it, I will explain them briefly.

### C/S Modeling

As we all know, it's easy to optimize a C/S model, so people usually choose HTTP as a communication protocol between server and client.

So can we use this model to implement the "Interactive Service"? Sure we can!

In the most famous IM software Wechat, we saw that every account has its signal list. Clients can request the unread signals via the cursor, after pulling the signals, clients will save the newest cursor to the local storage, and use it for the next request.

We can use this theory to "Interactive Service" too, but we should make some differences.

- We should split the server into several groups, classify the clients from the live room they attempt to request, and then redirect them to the correct server group.
- We should process the big live rooms and the little live rooms in different ways because some live rooms would have lots of audiences. When the big room is created, we should use all the groups of the server to receive the response.

### CDN Modeling

When we finished the C/S Modeling "Interactive Service", we could find that the C/S Modeling "Interactive Service" is very much like the CDN Service.

As we all know, CDN services can achieve higher performance and less latency. So we could use CDN services to help us to accept requests and deliver the signals.

### Server-Push Modeling

When the live room is updated fastly, using a custom application layer based on a proper transport layer is considerable.

In this way, we could choose WebSocket to reach more compatibility, or use QUIC to reach more efficiency, or just use TCP typically.

The protocol is easy, but the difficult part is load balancing. 

In HTTP Server, each server is stateless so it's easier to implement the load balancing. 

In the C/S Modeling part, we talked about the grouping of HTTP Server, the conception works in Server-Push Modeling, too. There are several models of this conception.

- Split up the server into some groups, and each group has its endpoint, the clients choose the targe endpoint by the id of the live room
- Using or implementing an application-level load balancing algorithm.

![](/uploads/persister-how-to-build-a-scalable-live-streaming-interactive-service--e6c9d24ely1h0oc72dn9yj21fc0u0afn.jpg)

## Summary

Buiding a live streaming platform is such a large project, so I cannot write down all of my thoughts in one article.

But in this part, we have discussed some core concepts of live streaming.

- Differences between Interactive Service and Video Streaming Service;
- Differences between Instant Messages and Live Streaming Signals;
- Different types of Live Streaming signals;
- Different types of Interactive Service connection modelings.

Next time I will share my thoughts on the scaling method of interactive service, hope you will like it.
