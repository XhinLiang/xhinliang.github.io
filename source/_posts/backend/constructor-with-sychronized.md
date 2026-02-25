title: Java 里的构造函数需要加锁吗
date: 2018-02-20 22:58:29
tags: [Java,Backend]
categories: Backend
toc: true
---

写 Java 代码时，我们经常会用到 `synchronized` 关键字。
`synchronized` 是一种相对重量级的锁，常见用法有两种。

1. 对一个具体的变量加锁。
``` java
        Logger l = LoggerFactory.getLogger(getClass().getName());
        synchronized (l) {
            // do something
        }
        
        synchronized (this) {
            // do something
        }
```

2. 修饰方法，可以是普通方法，也可以是静态方法。这事实上也是对 `this` 加锁。
``` java
    // 普通方法
    private synchronized void doSomething() {
        // something
    }

    // 上面等效
    private void doNothing() {
        synchronized (this) {
            // do something
        }
    }

    private static synchronized void doSomethingStatic() {
        // something
    }
```

`synchronized` 非常常见，但是大家有没有见过在构造函数里用 `synchronized` 修饰的呢？

事实上，如果在构造函数上加上 `synchronized` 修饰符，是无法通过编译的。
编译器会告诉你：**此处不允许使用修饰符 synchronized**。

先明确一点：在构造函数里使用 `synchronized` 修饰符，本质上就是对 `this` 加锁。
但为什么构造函数里不允许使用 `synchronized` 修饰符呢？

我认为有两个原因：
1. 在构造函数中，`this` 还没有完全构造完成，此时不能用作锁。
2. 在构造函数中，`this` 还没有被传递出去，因此不存在并发问题。

第一个原因好理解，第二个需要稍微展开说一下：
在 Java 的并发模型里，最重要的一点是变量的可见性。
如果多个线程同时对同一个对象调用某个方法，就有可能出现并发问题。

但在对一个对象执行 `new` 时，不可能存在多个线程同时 `new` 同一个对象。也就是说，正常情况下并不存在对 `this` 的并发安全性问题，所以在构造函数中对 `this` 加锁完全没有意义。

但有一种例外情况：如果你在构造函数中提前把 `this` 暴露到外部，就可能让构造过程本身也出现并发安全问题。例如：
``` java
    private static Object that;
    
    // 请不要这样做
    public Server() {
        that = this;
    }
```
这是个非常不好的习惯，我们在实际应用中应该尽量避免这样做。

有些朋友可能会问：如果在构造函数中传入某个对象，会不会出现并发安全性问题？
答案当然是会。要规避这些问题，你在构造函数里不应该对 `this` 加锁，而是应该 **在构造函数中对涉及到的、存在并发安全性问题的对象进行加锁**，或者在构造函数之外对该对象进行加锁（不常用）。

记住：构造函数本身是并发安全的；只有在额外引入不安全的参数时，才会导致构造函数变得不安全。
