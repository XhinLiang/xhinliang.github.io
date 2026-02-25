title: Rate Limiter In Action
date: 2023-12-29
tags: [Java,Concurrent,Backend,Distributed]
categories: Backend
toc: true
---

Backend systems that handle high QPS often need a local rate limiter to protect themselves.

Here, “local” means the limiter runs within a single process and does not coordinate state across instances.

Below are a few classic local rate-limiting strategies that show up in production code:

- Fixed Window
- Floating Window
- Leaky Bucket
- Token Bucket

We’ll go through them one by one.

## Fixed Window

Fixed Window rate limiting splits time into discrete windows and allows up to `N` actions per window.

The downside is that throughput can be uneven around window boundaries. Imagine we set a limit of 1000 requests per second:

- No requests arrive in the first 999 ms.
- Then 1000 requests arrive within 1 ms and all are allowed.
- Immediately after that, another 1000 requests arrive within 1 ms and are also allowed because they land in the next window.

One common mitigation is to make the window smaller (at the cost of a little more bookkeeping). For example, for “1000 per second”, you can split one second into 100 × 10 ms windows and limit 10 actions per 10 ms. The smaller the window, the smoother the limiter behaves.

Here is a minimal Java implementation:

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

This implementation tracks a window start time plus a counter. When the window expires it resets the counter; otherwise it allows the request until the capacity is reached.

## Floating Window

The Floating Window rate limiter smooths out Fixed Window’s boundary bursts by evaluating the request count over a moving time range.

It’s more expensive to implement because it typically requires tracking individual request timestamps (or maintaining buckets that approximate them).

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

The Leaky Bucket algorithm models rate limiting as a bucket that “leaks” at a constant rate. Bursts can be absorbed up to the bucket capacity, but the outflow rate stays stable.

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

Token Bucket uses a bucket of tokens. Tokens are replenished at a fixed rate.

The backend system consumes one token per request. When the bucket runs out of tokens, requests are rejected or delayed.

Compared with Leaky Bucket, Token Bucket is often more flexible because it allows short bursts (up to the bucket size) while still enforcing an average rate. It’s also convenient to implement by tracking only the token count and the last refill timestamp, without a dedicated refill thread.

Here is a simple implementation:

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
