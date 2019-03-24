---
title: redis实现分布式锁
date: 2019-03-03 21:12:34
categories: 分布式
---

在分布式环境下，数据一致性问题一直是一个比较重要的话题，而又不同于单进程的情况。分布式于单机情况下最大的不同在于其不是多线程而是多进程。多线程可以通过共享堆内存，因此可以简单的才去内存作为标记存储位置。而进程之间甚至都不在同一台物理机器上，因此需要将标记存储在一个所有进程都能看得到的地方。要确保分布式锁可用，我们至少要保证锁的实现同时满足以下四个条件：

- 互斥性：在任意时刻，只有一个客户端能持有锁
- 不会发生死锁：即使有一个客户端在持有锁期间崩溃而没有主动解锁，也能保证后续其他客户端能加锁
- 解锁和加锁需要是同一个线程：客户端自己不能吧别人给加的锁解了

常见的实现分布式锁的方案如下：

- 基于数据库实现分布式锁
- 基于缓存实现分布式锁，例如redis
- 基于zk实现分布式锁

本篇博客简单介绍下第二种，基于单节点redis实现的分布式锁，下面是简单的代码实现

#### 加锁的正确姿势

```java
private boolean getLock(String lockKey) {
    String requestId = UUID.randomUUID().toString();
    String result = jedis.set(lockKey, requestId, "NX", "EX", 60);
    return "OK".equals(result);
}
```

加锁的代码看起来非常简单，就一行，这个set方法一共有五个参数：

- 第一个为key，全局唯一的key
- 第二个为value，之所以需要这个value是因为我们需要知道这把锁是哪个请求加的，在解锁的时候就有依据
- 第三个为NX，意思是SET IF NOTEXIST，即当key不存在的时候，我们进行set操作。若key已经存在了，则不做任何操作。
- 第四个为EX，意思是给key加一个过期时间
- 第五个为具体的过期时间

上面这一行代码，便满足了我们可靠性保证中的三个条件。首先set()方法加入了NX参数，可以保证如果key已经存在，则函数不会调用成功，也就是只有一个客户端能持有锁，满足互斥性。其次我们给锁设置了过期时间，即使锁的持有者后续发生崩溃儿没有解锁，锁也会因为到了过期时间而自动解锁。最后我们传入requestId，保证客户端解锁的时候就可以校验是否是同一个客户端。

#### 解锁的正确姿势

```java
private void releaseLock(String lockKey, String requestId) {
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));
    }
```

同样，代码非常简单。第一行一个Lua脚本，第二行将lua脚本传到jedis.eval()方法里，并使参数keys[1]赋值为lockkey，argv[1]赋值为requestId。eval方法将Lua这段脚本交给redis服务端执行。其实，解锁的一个最重要的特征是要保证操作的原子性，至于为什么eval方法能保证原子性，下面是官网对eval命令的部分解释：

<img src="redis.png">

简单来说，就是在eval命令执行Lua代码的时候，Lua代码将被当成一个命令去执行，并且直到eval命令执行完成，redis才会执行其他命令。

之所以会有这篇文章，是当时做提现功能的时候，需要对提现主流程加锁，确保一个用户在同一个时刻只能做同一个操作。

