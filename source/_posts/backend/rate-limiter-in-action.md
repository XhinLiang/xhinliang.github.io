title: Rate Limiter In Action
date: 2023-12-29
tags: [Java,Concurrent,Backend,Distributed]
categories: Backend
toc: true
---

Backend systems that handle many requests per second often need a local rate limiter to protect themselves.

Here, "local" means the limiter works only within a single process and is not shared across processes.

Here are some local rate-limiting strategies commonly used in production environments.

- Fixed Window
- Floating Window
- Leaky Bucket
- Token Bucket

And we will discuss them one by one.

## Fixed Window

Fixed Window means that we can split a time to some window, and perform the action for a limit in a window.

The disadvantage is that system throughput may be uneven.
Imagine this scenario: we set a rate limit of 1000 requests per second and receive no requests in the first 999 ms.
Then, 1000 requests arrive within 1 ms, and all of them are granted.
Right after that, another 1000 requests arrive within 1 ms and can also be granted because they fall into a new window.

We can reduce this defect by splitting out the time window smaller.
For example, when we need a rate limiter for 1000 per second, we can split one second as 100 "tenMs" then limit 10 actions per "tenMs".
As we can see, the smaller you split the time window, the smoother the limiter behavior becomes.

After discussing the concept of Fixed Window rate limiting, it's beneficial to provide a code example. Here is a simple implementation in Java:

```java
public class FixedWindowRateLimiter {
    private final long capacity;
    private final long windowSizeInMillis;
    private long windowStart;
    private long usedCapacity;

    public FixedWindowRateLimiter(long capacity, long windowSizeInMillis) {
        this.capacity = capacity;
        this.windowSizeInMillis = windowSizeInMillis;
        this.windowStart = System.currentTimeMillis();
        this.usedCapacity = 0;
    }

    public synchronized boolean tryAcquire() {
        long currentTime = System.currentTimeMillis();
        if (currentTime - windowStart > windowSizeInMillis) {
            windowStart = currentTime;
            usedCapacity = 0;
        }

        if (usedCapacity < capacity) {
            usedCapacity++;
            return true;
        } else {
            return false;
        }
    }
}
```

In this example, the FixedWindowRateLimiter class defines a rate limiter with a fixed capacity and window size. The tryAcquire method checks if the current window has expired and resets it if necessary. If the capacity within the current window hasn't been exceeded, it allows the request.

## Floating Window

Now let's discuss the Floating Window rate limiter.

The Floating Window rate limiter improves upon the Fixed Window by smoothing out the request allowance over time. It avoids the scenario where a burst of requests is allowed at the boundary of the time windows.

A Floating Window rate limiter can be more complex to implement as it requires tracking the timestamps of individual requests.

```java
public class FloatingWindowRateLimiter {
    private final long windowSizeInMillis;
    private final int maxRequests;
    private final Deque<Long> requestTimestamps = new LinkedList<>();

    public FloatingWindowRateLimiter(long windowSizeInMillis, int maxRequests) {
        this.windowSizeInMillis = windowSizeInMillis;
        this.maxRequests = maxRequests;
    }

    public synchronized boolean allowRequest() {
        long currentTime = System.currentTimeMillis();
        while (!requestTimestamps.isEmpty() && currentTime - requestTimestamps.peek() > windowSizeInMillis) {
            requestTimestamps.poll();
        }
        if (requestTimestamps.size() < maxRequests) {
            requestTimestamps.add(currentTime);
            return true;
        }
        return false;
    }
}
```

## Leaky Bucket

The Leaky Bucket algorithm models rate limiting as a bucket that leaks requests at a constant rate. This approach helps in smoothing out bursts of traffic.

```
public class LeakyBucketRateLimiter {
    private final long capacity;
    private final long leakRateInMillis;
    private long availableTokens;
    private long lastLeakTimestamp;

    public LeakyBucketRateLimiter(long capacity, long leakRateInMillis) {
        this.capacity = capacity;
        this.leakRateInMillis = leakRateInMillis;
        this.availableTokens = capacity;
        this.lastLeakTimestamp = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        long currentTime = System.currentTimeMillis();
        long timeSinceLastLeak = currentTime - lastLeakTimestamp;
        long tokensToLeak = timeSinceLastLeak / leakRateInMillis;
        if (tokensToLeak > 0) {
            availableTokens = Math.min(capacity, availableTokens + tokensToLeak);
            lastLeakTimestamp = currentTime;
        }
        if (availableTokens > 0) {
            availableTokens--;
            return true;
        }
        return false;
    }
}
```

## Token Bucket

Token Bucket means we have a bucket that holds tokens, and tokens are replenished at a fixed rate.

The backend system consumes one token per request. When the bucket runs out of tokens, requests are rejected or delayed.

Token Bucket seems like Leaky Bucket but can be more effective, because we can track only the token count and the last refill timestamp instead of running a dedicated refill thread.

Here is a simple implementation.

```java
public class RateLimiter {

    private volatile long latestSupplyTimestamp = 0L;
    private final AtomicLong tokenCount = new AtomicLong();

    private final int performPerSecond;

    public RateLimiter(int performPerSecond) {
        this.performPerSecond = performPerSecond;
    }

    public boolean canDo() {
        long now = TimeUnit.MILLISECONDS.toSeconds(System.currentTimeMillis());
        long diff = now - latestSupplyTimestamp;

        if (diff == 0) {
            return tokenCount.decrementAndGet() > 0;
        }

        synchronized (this) {
            if (diff >= 1000L) {
                latestSupplyTimestamp = now;
                tokenCount.set(performPerSecond - 1L);
            }

            // 0 < diff < 1000
            long shouldSupply = diff * performPerSecond - 1L;
            latestSupplyTimestamp = now;
            return tokenCount.addAndGet(shouldSupply) >= 0L;
        }
    }
}
```
