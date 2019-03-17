---
title: Redis锁
tags: 
  - redis
categories:
  - redis
---

为了对数据进行排他性的访问，Redis可以通过WATCH, MULTI, EXEC组成的事务来实现。另外一种方法就是锁。

`SETNX key value`

将 `key` 的值设为 `value` ，当且仅当 `key` 不存在。若给定的 `key` 已经存在，则不做任何动作。

一个简单的锁获取和释放demo：

```python
def acquire_lock(conn, lock_name, acquire_timeout=10):
    identifier = str(uuid.uuid1())
    end_time = time.time() + acquire_timeout
    while time.time() < end_time:
        if conn.setnx('lock:' + lock_name, identifier):
            return identifier
        time.sleep(0.001)
    return False
```

```python
def release_lock(conn, lock_name, identifier):
    pipe = conn.pipeline(True)
    lock_name = 'lock:' + lock_name
    while True:
        try:
            pipe.watch(lock_name)
            if pipe.get(lock_name) == identifier:
                pipe.multi()
                pipe.delete(lock_name)
                pipe.execute()
                return True
            pipe.unwatch()
            break
        except redis.exceptions.WatchError:
            pass
    return False
```

当锁在持有者崩溃的时候，上例中的锁是不会自动释放的，因此为了解决这个问题，需要为锁加上超时功能。

```python
def acquire_lock_with_timeout(conn, lock_name, acquire_timeout=10, lock_timeout=10):
    identifier = str(uuid.uuid1())
    lock_name = "lock:" + lock_name
    end_time = time.time() + acquire_timeout
    while time.time() < end_time:
        if conn.setnx(lock_name, identifier):
            conn.expire(lock_name, lock_timeout)
            return identifier
        elif not conn.ttl(lock_name):
            # ttl
            # 当 key 不存在时，返回 -2 。 当 key 存在但没有设置剩余生存时间时，返回 -1 。
            # 否则，以秒为单位，返回 key 的剩余生存时间。
            conn.expire(lock_name, lock_timeout)
        time.sleep(0.001)
    return False
```

如果锁的持有者在`setnx`和`expire`中间崩溃了，可以考虑用`set`命令

`SET key value [EX seconds][PX milliseconds] [NX|XX]`

- `EX second` ：设置键的过期时间为 `second` 秒。 `SET key value EX second` 效果等同于 `SETEX key second value` 。
- `PX millisecond` ：设置键的过期时间为 `millisecond` 毫秒。 `SET key value PX millisecond` 效果等同于 `PSETEX key millisecond value` 。
- `NX` ：只在键不存在时，才对键进行设置操作。 `SET key value NX` 效果等同于 `SETNX key value` 。
- `XX` ：只在键已经存在时，才对键进行设置操作。

计数信号量也是一种锁，可以限制一项资源能够同时被多少个进程访问，demo如下：

```python
def acquire_semaphore(conn, sem_name, limit=3, timeout=10):
    """
        信号量
    """
    identifier = str(uuid.uuid1())
    czset = sem_name + ':owner'
    ctr = sem_name + ':counter'    # 计数器
    now = time.time()
    pipe = conn.pipeline(True)
    # 删除超时信号量
    pipe.zremrangebyscore(sem_name, '-inf', now-timeout)
    # ZINTERSTORE 计算给定的一个或多个有序集的交集, 0,1是乘法因子WEIGHTS
    pipe.zinterstore(czset, {czset: 1, sem_name: 0})
    # 获取计数器的自增值
    pipe.incr(ctr)
    counter = pipe.execute()[-1]
    pipe.zadd(sem_name, identifier, now)
    pipe.zadd(czset, identifier, counter)
    pipe.zrank(czset, identifier)
    rank = pipe.execute()[-1]   # 返回从小到大的排名
    if rank < limit:
        return identifier
    pipe.zrem(sem_name, identifier)
    pipe.zrem(czset, identifier)
    pipe.execute()
    return None
```