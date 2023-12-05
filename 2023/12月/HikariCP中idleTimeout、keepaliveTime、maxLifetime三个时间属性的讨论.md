> Java/连接池/HikariCP

> 之前对连接池这块的参数配置不是很清楚，最近在看HikariCP这个连接池，对它的三个时间设置只看文档还是理解的不够，所以这里写一篇文章来从源码的角度，看看这三个参数的作用。

# HikariCP中idleTimeout、keepaliveTime、maxLifetime三个时间属性的讨论

## idleTimeout

这个值获取是通过com.zaxxer.hikari.HikariConfig#getIdleTimeout这个方法做的，这个方法是被com.zaxxer.hikari.pool.HikariPool.HouseKeeper#run唯一调用的。

### HouseKeeper任务

 关于这个任务，首先我们来看执行这个定时任务的线程池定义和初始化：

```java
private final ScheduledExecutorService houseKeepingExecutorService;

private ScheduledExecutorService initializeHouseKeepingExecutorService() {
    if (config.getScheduledExecutor() == null) {
        // 获取线程工厂
        final var threadFactory = Optional.ofNullable(config.getThreadFactory()).orElseGet(() -> new DefaultThreadFactory(poolName + " housekeeper"));
        // 初始化线程池，拒绝策略DiscardPolicy直接丢弃新提交的任务；
        final var executor = new ScheduledThreadPoolExecutor(1, threadFactory, new ThreadPoolExecutor.DiscardPolicy());
        // 若执行器已关闭，则不再执行有延迟的任务
        executor.setExecuteExistingDelayedTasksAfterShutdownPolicy(false);
        // 设置ScheduledFuture调用cancle()方法后，立即从工作队列中删除已经取消的任务，默认是false
        executor.setRemoveOnCancelPolicy(true);
        return executor;
    } else {
        return config.getScheduledExecutor();
    }
}
```

定时任务的定义：

```java
// 定义一个延迟任务，线程池启动后延迟100ms执行，每30ms执行一次
this.houseKeeperTask = houseKeepingExecutorService.scheduleWithFixedDelay(new HouseKeeper(), 100L, housekeepingPeriodMs, MILLISECONDS);
```

具体的逻辑：

1. 如果时钟发生回退，则对连接池中所有连接进行软驱逐，软驱逐的连接是不用的。驱逐的意思就是关闭。
2. 如果时钟超前，则对连接不做处理。
3. 如果空闲连接的超时时间大于0，且最小空闲线程数小于最大线程数量，则对超过最小线程数的空闲线程进行关闭。
4. 最后将空闲线程数补充到最小空闲线程数。

```java
private final long housekeepingPeriodMs = Long.getLong("com.zaxxer.hikari.housekeeping.periodMs", SECONDS.toMillis(30));

/**
 * housekeeping task是为了清理空闲连接和保持空闲连接的最小连接数
 * The housekeeping task to retire and maintain minimum idle connections.
*/
private final class HouseKeeper implements Runnable {
    // housekeepingPeriodMs为30ms，用当前时间减去30ms为之前的时间
    private volatile long previous = plusMillis(currentTime(), -housekeepingPeriodMs);

    @Override
    public void run() {
        try {

            final var idleTimeout = config.getIdleTimeout();
            final var now = currentTime();

            // Detect retrograde time, allowing +128ms as per NTP spec.
            // 检测逆行时间，允许按NTP规范加128ms
            // 用之前的时间加30s，比当前的时间还大128ms，说明时钟回退了
            if (plusMillis(now, 128) < plusMillis(previous, housekeepingPeriodMs)) {
                // 如果最新的时间+128毫秒，比前面时间还要小，说明时钟超前了，服务器时钟中途被修改了
                // 如果逆行时间大于128毫秒，则标识该连接为软驱逐
                logger.warn("{} - Retrograde clock change detected (housekeeper delta={}), soft-evicting connections from pool.",
                            poolName, elapsedDisplayString(previous, now));
                previous = now;
                softEvictConnections();
                return;
                // 如果当前时间大于之前的时间加45ms，之前的时间其实就已经减过30ms了，也就是说现在的时间比之前获取的时间还要大15ms，也是时钟不对
            } else if (now > plusMillis(previous, (3 * housekeepingPeriodMs) / 2)) {
                // No point evicting for forward clock motion, this merely accelerates connection retirement anyway
                // 没有必要为前进时钟运动去驱逐连接，这只会加速连接退役
                logger.warn("{} - Thread starvation or clock leap detected (housekeeper delta={}).", poolName, elapsedDisplayString(previous, now));
            }

            previous = now;

            // 如果空闲连接的超时时间大于0，且最小空闲连接数小于最大连接数
            if (idleTimeout > 0L && config.getMinimumIdle() < config.getMaximumPoolSize()) {
                logPoolState("Before cleanup ");
                // 获取线程池中空闲连接数组
                final var notInUse = connectionBag.values(STATE_NOT_IN_USE);
                // 需要移除的空闲连接数
                var maxToRemove = notInUse.size() - config.getMinimumIdle();
                for (PoolEntry entry : notInUse) {
                    // 若需要移除的连接数大于0，且上次访问连接的时大于空闲超时时间，且已经标记了该连接为 保留 状态
                    if (maxToRemove > 0 && elapsedMillis(entry.lastAccessed, now) > idleTimeout && connectionBag.reserve(entry)) {
                        closeConnection(entry, "(connection has passed idleTimeout)");
                        maxToRemove--;
                    }
                }
                logPoolState("After cleanup  ");
            } else
                logPoolState("Pool ");

            // 填充空闲连接数量到配置的最小连接数
            fillPool(true); // Try to maintain minimum connections
        } catch (Exception e) {
            logger.error("Unexpected exception in housekeeping task", e);
        }
    }
}
```

## keepaliveTime

这个参数的具体作用在com.zaxxer.hikari.pool.HikariPool#createPoolEntry的这段代码中：

```java
final long keepaliveTime = config.getKeepaliveTime();
if (keepaliveTime > 0) {
    // 心跳时间和保活时间的误差小于保活时间的10%，也就是心跳检查会在保活时间到期时去做
    // variance up to 10% of the heartbeat time
    final var variance = ThreadLocalRandom.current().nextLong(keepaliveTime / 10);
    final var heartbeatTime = keepaliveTime - variance;
    // 这里使用了定时线程池按固定延时去不断的执行保活任务，这是一个延时批量任务
    poolEntry.setKeepalive(houseKeepingExecutorService.scheduleWithFixedDelay(new KeepaliveTask(poolEntry), heartbeatTime, heartbeatTime, MILLISECONDS));
}
```

下面我们去看看KeepaliveTask的具体做了什么

```java
private final class KeepaliveTask implements Runnable {
    private final PoolEntry poolEntry;

    KeepaliveTask(final PoolEntry poolEntry) {
        this.poolEntry = poolEntry;
    }

    public void run() {
        // 设置池对象的状态为 保留
        if (connectionBag.reserve(poolEntry)) {
            // 判断连接是否已经无效，这里面分别使用了JDBC4的isValid()方法或者通过执行connectionTestQuery属性sql判断连接是否已经关闭
            if (isConnectionDead(poolEntry.connection)) {
                // 如果检测连接已经无效，则关闭连接
                softEvictConnection(poolEntry, DEAD_CONNECTION_MESSAGE, true);
                // 新增连接到连接池中
                addBagItem(connectionBag.getWaitingThreadCount());
            } else {
                // 如果连接还有效，
                connectionBag.unreserve(poolEntry);
                logger.debug("{} - keepalive: connection {} is alive", poolName, poolEntry.connection);
            }
        }
    }
}
```

未完待续~



# 参考文章

* [HikariCP探活机制如何保证链接有效](https://blog.csdn.net/m0_46485771/article/details/118583714)
* [一篇文章彻底理解数据库的各种 JDBC 超时参数](https://zhuanlan.zhihu.com/p/582551494)