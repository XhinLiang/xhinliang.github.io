# Rate Limiter In Action
## Overview

The backend systems which have lots of request per second always need a local rate limiter to protect themself.

Which "local" means that this rate limiter worked in only this process and not shared with another.

Here are some local rate limiter used in prodction evironment.

- Fixed Window
- Floating Window
- Leaky Bucket
- Token Bucket

And we will discuss them one by one.

## Fixed Window

Fixed Window means that we can split a time to some window, and perform the action for a limit in a window.

The disatvantage is that the throughput of this system can be not such steady.
Image this scenario, we have a rate limiter for 1000 per second, and perform not one request in first 999ms.
After this, we perform 1000 request in 1ms, all of these request will be granted.
And then we perform 1000 request in 1ms too, as we can see, the latest 1000 requests will be granted too, because there two part of request matches the different time window.

We can reduce this defect by splitting out the time window smaller.
For example, when we need a rate limiter for 1000 per second, we can split one second as 100 "tenMs" then limit 10 actions per "tenMs".
As we can see, the smaller you slit the time window, the smoother it rate limiter can be.

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

Token Bucket means that we have a bucket which contains lots of bucket, and there is another thread will supply the token in a fixed rate at the same time.

Backend system will take a token from the bucket for per request. When the bucket is run out of token, the request will be blocked.

Token Bucket seems like Leaky Bucket but can be more effective, because we can just record the number of bucket and the latest time when we supply the bucket, instead of using a token-supplier thread.

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
