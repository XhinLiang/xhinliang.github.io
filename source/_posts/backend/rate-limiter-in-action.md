title: 本地限流器实战：四种经典算法与实现思路
date: 2023-12-29
tags: [Java,Concurrent,Backend,RateLimiter]
categories: Backend
toc: true
---

高 QPS 的后端服务，几乎一定需要“限流”（Rate Limiting）。

这里讨论的是**本地限流器（local rate limiter）**：限流状态只存在于**单个进程/单个实例**中，不和其他实例共享。

它的目标很朴素：

- 在流量突增时保护服务自身（CPU/线程池/DB 连接/下游依赖）
- 让系统吞吐更可控，避免被瞬时尖峰打穿

本地限流最常见的四种算法：

1. 固定窗口（Fixed Window）
2. 滑动窗口（Sliding / Floating Window）
3. 漏桶（Leaky Bucket）
4. 令牌桶（Token Bucket）

下面逐个讲它们的“直觉、优缺点、以及一个简化实现”。

> 提醒：示例代码为了讲清楚思路，刻意简化了很多工程细节（时间单位、溢出、并发性能、时钟回拨、分布式场景等）。

## 1. 固定窗口（Fixed Window）

### 思路
把时间切成一个个**固定长度的窗口**（比如 1 秒一个窗口），每个窗口允许最多 N 次请求。

### 典型问题：窗口边界突刺
固定窗口的最大缺点是**边界效应**。

举个极端例子：限制是 **1000/s**。

- 前 999ms 一次请求都没有
- 在最后 1ms 突然来了 1000 个请求（全放行）
- 下一秒的第 1ms 又来了 1000 个请求（也全放行）

结果就是：在非常短的时间内，系统可能承受接近 2000 的瞬时突刺。

### 缓解方式：把窗口切得更细
比如你要实现 1000/s，可以把 1 秒切成 100 个 10ms 小窗口，每个 10ms 允许 10 次。

窗口越细，越平滑，但实现也更复杂、开销也更高。

### Java 简化实现

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
        long now = System.currentTimeMillis();
        if (now - windowStart > windowSizeInMillis) {
            windowStart = now;
            usedCapacity = 0;
        }

        if (usedCapacity < capacity) {
            usedCapacity++;
            return true;
        }
        return false;
    }
}
```

## 2. 滑动窗口（Sliding / Floating Window）

### 思路
滑动窗口的目标是解决固定窗口的“边界突刺”。

做法之一是：记录最近一个窗口大小（比如最近 1 秒）内的请求时间戳；每次请求进来时，把超过窗口的记录丢掉，然后看当前窗口内数量是否超过阈值。

### 优缺点
- 优点：更平滑，更符合“过去 1 秒最多 N 次”的直觉
- 缺点：需要存储时间戳（内存/CPU 开销），高并发下需要更好的数据结构与锁策略

### Java 简化实现

```java
import java.util.Deque;
import java.util.LinkedList;

public class SlidingWindowRateLimiter {
    private final long windowSizeInMillis;
    private final int maxRequests;
    private final Deque<Long> timestamps = new LinkedList<>();

    public SlidingWindowRateLimiter(long windowSizeInMillis, int maxRequests) {
        this.windowSizeInMillis = windowSizeInMillis;
        this.maxRequests = maxRequests;
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();

        while (!timestamps.isEmpty() && now - timestamps.peekFirst() > windowSizeInMillis) {
            timestamps.pollFirst();
        }

        if (timestamps.size() < maxRequests) {
            timestamps.addLast(now);
            return true;
        }
        return false;
    }
}
```

## 3. 漏桶（Leaky Bucket）

### 思路
把请求想象成倒进桶里的水。

- 桶有容量上限（满了就拒绝）
- 桶底部以一个**固定速率**漏水（对应“以恒定速率处理请求”）

这会把突发流量“削峰填谷”，让输出更平滑。

### 直觉
漏桶更像是：你限制的是“处理速率”，而不是“允许速率”。

### Java 简化实现（以 token/水量表示）

```java
public class LeakyBucketRateLimiter {
    private final long capacity;
    private final long leakRateInMillis;

    private long available;
    private long lastLeakTs;

    public LeakyBucketRateLimiter(long capacity, long leakRateInMillis) {
        this.capacity = capacity;
        this.leakRateInMillis = leakRateInMillis;
        this.available = capacity;
        this.lastLeakTs = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();
        long elapsed = now - lastLeakTs;

        long leaked = elapsed / leakRateInMillis;
        if (leaked > 0) {
            available = Math.min(capacity, available + leaked);
            lastLeakTs = now;
        }

        if (available > 0) {
            available--;
            return true;
        }
        return false;
    }
}
```

> 注意：这里的“available”实现只是为了方便理解。生产实现里你可能会用更精确的方式表示速率与时间。

## 4. 令牌桶（Token Bucket）

### 思路
令牌桶是工程上最常用的一种：

- 桶里有令牌，最大容量为 `burst`
- 系统按固定速率往桶里“补令牌”
- 每个请求消耗一个令牌
- 没令牌就拒绝（或排队等待）

它和漏桶的区别在于：**令牌桶天然允许一定程度的突发（burst）**：只要桶里之前攒了令牌，就可以瞬间放行一批。

### 常见实现技巧
很多人一开始会用一个“补令牌线程”。但实际上可以不用线程：

- 记录上次补给时间 `lastSupply`
- 每次请求到来时，按 `now - lastSupply` 计算应该补多少令牌

这样更简单，也更省资源。

### Java 简化实现（接近原思路）

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

public class TokenBucketRateLimiter {

    private volatile long latestSupplySecond = 0L;
    private final AtomicLong tokenCount = new AtomicLong();

    private final int permitsPerSecond;

    public TokenBucketRateLimiter(int permitsPerSecond) {
        this.permitsPerSecond = permitsPerSecond;
    }

    public boolean canDo() {
        long nowSec = TimeUnit.MILLISECONDS.toSeconds(System.currentTimeMillis());
        long diff = nowSec - latestSupplySecond;

        if (diff == 0) {
            return tokenCount.decrementAndGet() >= 0;
        }

        synchronized (this) {
            long nowSec2 = TimeUnit.MILLISECONDS.toSeconds(System.currentTimeMillis());
            long diff2 = nowSec2 - latestSupplySecond;

            if (diff2 <= 0) {
                return tokenCount.decrementAndGet() >= 0;
            }

            // 距离上次补给过了 diff2 秒
            long shouldSupply = diff2 * (long) permitsPerSecond;
            latestSupplySecond = nowSec2;

            // 这里没做“桶上限”（burst）限制，是为了示例简化
            return tokenCount.addAndGet(shouldSupply - 1) >= 0;
        }
    }
}
```

## 什么时候该用哪种？（快速建议）

- 想要实现简单、开销低：固定窗口（但要接受边界突刺）
- 想要更严格的“过去 1 秒最多 N 次”：滑动窗口
- 想要输出速率尽量平滑：漏桶
- 既要限速，又要允许一定突发（最常见）：令牌桶

如果你后面要做“分布式限流”（多实例共享额度），那就需要把状态放到 Redis/网关/Sidecar 等地方，算法本身也需要考虑一致性与性能。
