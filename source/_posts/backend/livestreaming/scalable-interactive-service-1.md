title: How to Build a Scalable Live Streaming Interactive Service - Part I
date: 2022-1-15
tags: [Backend,LiveStreaming,Live]
categories: Backend
toc: true
---

## Background

With faster networks and the push from COVID-era remote life, live streaming has seen a resurgence.

![](/uploads/persister-how-to-build-a-scalable-live-streaming-interactive-service--e6c9d24ely1h0obi03jucj20yv0u0dkg.jpg)

I’ve been involved in building a live streaming platform, and the part I’ve spent the most time on is the “Interactive Service”. This series is a write-up of how I think about that component.

First, let’s define what “Interactive Service” means. A useful mental model is to split a live streaming platform into two subsystems: “Video Streaming” and “Interactive Service”.

![](/uploads/persister-how-to-build-a-scalable-live-streaming-interactive-service--e6c9d24ely1h0oblby6l5j20z60u0tbv.jpg)

“Video Streaming” is the audio/video pipeline that viewers watch and listen to in (near) real time. It’s the foundation of live streaming—without it, there’s nothing to watch, and the room degenerates into a plain group chat.

“Interactive Service” is everything else: comments, gifts, e-commerce notifications, moderation actions, presence, and so on. It turns a stream from a one-way broadcast into a shared room where the host and audience can interact.

## Interactive Service

I treat the “Interactive Service” as an event distribution layer plus a set of business workflows.

Both hosts and viewers generate signals (events/messages) to it, and it delivers those signals to other participants in the room. For example, a viewer `foo` can send a comment like “You look good”, and the interactive service will broadcast that comment to everyone else shortly after.

The interactive service can also generate events on its own. For example, it might periodically broadcast the current online user count so everyone can see how many viewers are in the room.

The core topic of this article is how to build an interactive service that delivers signals safely, quickly, and cost-effectively:

- Safe
  - The interactive service should authenticate and authorize requests/connections.
  - Clients should not receive any signals they are not allowed to see.
- Quick
  - The interactive service should deliver signals within a bounded, low latency—regardless of room size.
- Economic
  - The interactive service should support very large rooms with huge audiences.
  - It should use resources efficiently: CPU, memory, disk, and network bandwidth.

## Signal Types

Before we discuss different ways to model an interactive service, it helps to classify the signals you need to deliver. In my experience, there are three types, and each tends to want a different implementation.

### State Sync Signal

Right after entering a room, clients need an initial snapshot of state (room metadata, configuration, pinned content, and so on). When that state changes, clients should be notified and update their local view.

A common implementation uses a “total + partial versions” scheme to describe business state changes.

In this scheme, both servers and clients track a total version plus multiple partial versions. Clients poll the total version periodically; if it differs from the local total version, the client fetches only the partial versions it is missing.

![](/uploads/persister-how-to-build-a-scalable-live-streaming-interactive-service--e6c9d24ely1h0oc8bkeruj21d90u078y.jpg)

We call this model `SS Signal` for short.

### Time Series Signal

A Time Series Signal is a temporary event in a room. Events that happened before a client joined are usually not replayed, and missing a few time-series events typically should not break the experience.

There are multiple ways to implement time-series signals, but two common approaches are:
- Use a messaging system, such as Redis Pub/Sub or Kafka.
- Store events in a time-series database and have clients fetch them periodically.

![](/uploads/persister-how-to-build-a-scalable-live-streaming-interactive-service--e6c9d24ely1h0oc8tju01j219g0u0djh.jpg)

We call this model `TS Signal` for short.

### Peer Delivery Signal

SS signals and TS signals are designed to reach everyone in the same live room. When state changes or an action happens, these signals are delivered to all clients in that room. Peer delivery signals, in contrast, target only a subset of participants.

You can implement peer delivery by broadcasting and letting clients filter, but it tends to create performance issues (especially bandwidth waste) as rooms grow.

A more efficient approach is to treat peer delivery like instant messaging. With a typical IM system, you can locate which server the target client is connected to, so sending peer signals is both direct and efficient.

## Connection Modeling

The interactive service also needs a way for servers and clients to communicate. There are several common connection models; I’ll briefly outline a few.

### C/S Modeling

Traditional request/response is easy to scale and optimize, so many systems start with an HTTP-based client/server (C/S) model.

Can we use this model to implement an interactive service? Sure—especially for pull-based delivery.

A familiar example is WeChat: each account has a message list, and clients request unread messages via a cursor. After pulling messages, the client stores the latest cursor locally and uses it for the next request.

We can apply the same idea to an interactive service, with a few differences:

- Split servers into groups, classify clients by the live room they are watching, and route them to the correct server group.
- Handle large rooms differently from small ones. Some rooms can have huge audiences, and a single group may become a bottleneck; those rooms often need extra fan-out capacity across multiple groups.

### CDN Modeling

Once you have a pull-based C/S interactive service, it starts to resemble a CDN: lots of stateless nodes accepting requests, caching aggressively, and serving responses close to users.

CDN-style architectures can improve latency and throughput, so it can be useful to borrow the same ideas (or even reuse CDN infrastructure) to accept requests and deliver some classes of signals.

### Server-Push Modeling

When a room updates rapidly, server-push is often a better fit than repeated polling.

In this model, you might choose WebSocket for compatibility, QUIC for efficiency, or plain TCP for simplicity in a controlled environment.

The transport is relatively straightforward; the difficult part is load balancing stateful connections.

With classic HTTP request/response, each request is mostly independent, which makes load balancing simpler.

The “server grouping” idea from the C/S model also applies to server-push. Common approaches include:

- Split servers into groups and give each group its own endpoint. Clients choose the endpoint based on the live room ID.
- Use (or implement) an application-level load balancing algorithm to assign and migrate connections.

![](/uploads/persister-how-to-build-a-scalable-live-streaming-interactive-service--e6c9d24ely1h0oc72dn9yj21fc0u0afn.jpg)

## Summary

Building a live streaming platform is a large project, so I can’t capture all of my thoughts in a single post.

In Part I, we discussed a few core concepts that will be useful later:

- The boundary between the Interactive Service and the Video Streaming service;
- How live streaming signals differ from instant messages;
- Different types of live streaming signals;
- Different connection models for an interactive service.

In Part II, I’ll share my thoughts on scaling methods for interactive services.
