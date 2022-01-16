title: Rate Limiter in Action
date: 2022-1-16
tags: [Backend]
categories: Backend
toc: true
---

## Overview

The backend systems which have lots of requests per second always need a local rate limiter to protect themself. Which "local" means that this rate limiter worked in only this process and was not shared with another.

Here are some local rate limiters used in the production environment.

- Fixed Window
- Floating Window
- Leaky Bucket
- Token Bucket

And we will discuss them one by one.

## Fixed Window

Fixed Window means that we can split a time to some window, and perform the action for a limit in a window.

The disadvantage is that the throughput of this system can be not such steady.
Image this scenario, we have a rate limiter for 1000 per second and perform not one request in the first 999ms.
After this, we perform 1000 requests in 1ms, all of these requests will be granted.
And then we perform 1000 requests in 1ms too, as we can see, the latest 1000 requests will be granted too because these two parts of the request match the different time windows.

We can reduce this defect by splitting out the time window smaller. For example, when we need a rate limiter for 1000 per second, we can split one second as 100 "ten milliseconds" then limit 10 actions per "ten milliseconds". As we can see, the smaller you slit the time window, the smoother it rate limiter can be.

## Floating Window

The ideology of Floating Window is just like the combination of a big amount of little Fixed Window. We limit the total quota with the range of little Fixed Window, and calculate the floating consumption and floating allowance dynamically.

## Leaky Bucket

We could use a queue to store the request temporarily, and use another thread to accept the request asynchronously, this thought is called Leaky Bucket.

Actually, people would not use a queue and a thread to do this, because it's commonly a waste of thread resources.

## Token Bucket

Token Bucket means that we have a bucket that contains lots of buckets, and there is another thread that will supply the token in a fixed-rate at the same time.

The backend system will take a token from the bucket per request. When the bucket is run out of tokens, the request will be blocked.

Token Bucket seems like Leaky Bucket but can be more effective, because we can just record the number of the bucket and the latest time when we supply the bucket, instead of using a token-supplier thread.

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
