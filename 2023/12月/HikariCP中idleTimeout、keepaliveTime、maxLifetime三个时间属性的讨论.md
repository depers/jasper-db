> Java/连接池/HikariCP

> 之前对连接池这块的参数配置不是很清楚，最近在看HikariCP这个连接池，对它的三个时间设置只看文档还是理解的不够，所以这里写一篇文章来从源码的角度，看看这三个参数的作用。

# idleTimeout

## 源码解析

HouseKeeper任务

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

Housekeeper的具体逻辑如下：

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
                // 进行连接驱逐
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

## 总结

设置了idleTimeout后，每30ms会去清理超过空闲时间的连接。值得注意的是如果设置这个参数需要满足几个限制：

1. idleTimeout必须大于0且小于maxLifetime，这个HikariCP会做参数校验；
2. 最小空闲连接数必须小于最大连接数，但是官方建议将这两个值设置为一样，所以这个参数**不建议设置**；

# keepaliveTime

## 源码解析

这个参数的启动程序作用在com.zaxxer.hikari.pool.HikariPool#createPoolEntry的这段代码中声明的：

```java
final long keepaliveTime = config.getKeepaliveTime();
// 切记这里的前提是你设置了keepaliveTime
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
        // 设置池对象的状态为“不可借”
        if (connectionBag.reserve(poolEntry)) {
            // 判断连接是否已经无效，这里面分别使用了JDBC4的isValid()方法或者通过执行connectionTestQuery属性sql判断连接是否已经关闭
            if (isConnectionDead(poolEntry.connection)) {
                // 如果检测连接已经无效，则关闭连接
                softEvictConnection(poolEntry, DEAD_CONNECTION_MESSAGE, true);
                // 新增连接到连接池中
                addBagItem(connectionBag.getWaitingThreadCount());
            } else {
                // 设置连接可以被再次借用
                connectionBag.unreserve(poolEntry);
                logger.debug("{} - keepalive: connection {} is alive", poolName, poolEntry.connection);
            }
        }
    }
}
```

这里需要特别讲一下的是isConnectionDead()方法：

```java
/**
  * 判断连接是否已经关闭
  *
  * @param connection
  * @return
  */
boolean isConnectionDead(final Connection connection) {
    try {
        // 设置连接到数据库的超时时间
        setNetworkTimeout(connection, validationTimeout);
        try {
            final var validationSeconds = (int) Math.max(1000L, validationTimeout) / 1000;

            // 判断是否使用JDBC4的API验证连接是否有效，如果配置了connectionTestQuery这个值，isUseJdbc4Validation为false
            if (isUseJdbc4Validation) {
                return !connection.isValid(validationSeconds);
            }

            try (var statement = connection.createStatement()) {
                if (isNetworkTimeoutSupported != TRUE) {
                    // 设置单个语句的执行超时时间
                    setQueryTimeout(statement, validationSeconds);
                }
                // 执行配置的连接检测查询语句
                statement.execute(config.getConnectionTestQuery());
            }
        } finally {
            // 设置连接到数据库的超时时间
            setNetworkTimeout(connection, networkTimeout);

            // isIsolateInternalQueries：此属性确定 HikariCP 是否将内部池查询（例如连接活动测试）隔离在自己的事务中。由于这些查询通常是只读查询，因此很少需要将它们封装在自己的事务中。仅当禁用时 autoCommit ，此属性才适用。默认是false
            // isAutoCommit：是否自动提交
            // 如果你配置了不自动提交，且这些sql的执行都是连接池内部自己的逻辑
            if (isIsolateInternalQueries && !isAutoCommit) {
                // 撤销当前事务中所做的所有更改，并释放当前由此Connection对象持有的所有数据库锁。此方法应仅在禁用自动提交模式时使用。
                connection.rollback();
            }
        }

        return false;
    } catch (Exception e) {
        // 出现异常，说明数据库连接已经失效
        lastConnectionFailure.set(e);
        logger.warn("{} - Failed to validate connection {} ({}). Possibly consider using a shorter maxLifetime value.",
                    poolName, connection, e.getMessage());
        return true;
    }
}
```

## 总结

keepaliveTime这个参数的作用是保活空闲连接的，或是说检测空闲连接是否可用的。如果数据库驱动不支持JDBC4，这个参数需要和connectionTestQuery一起使用。而基本上java MySQL驱动包5以上的版本都支持JDBC4，所以`connectionTestQuery`
配置项，官方建议如果驱动支持JDBC4，不要设置此属性。

这个参数默认是0，如果设置了，值不能大于等于maxLifetime或者是小于30ms，否则会被重置为0。这个参数我觉得是**需要设置的**，这样做也从一方面避免获取到被数据库已经关闭的连接，从而避免了业务出错。

# maxLifetime

## 源码解析

这个参数的启动程序作用在com.zaxxer.hikari.pool.HikariPool#createPoolEntry的这段代码中声明的：

```java
final var maxLifetime = config.getMaxLifetime();
if (maxLifetime > 0) {
    // variance up to 2.5% of the maxlifetime
    // 任务执行的延迟时间和最大生存时间的误差在2.5%，也就是说这个任务会提前2.5%的时间关闭这个连接
    final var variance = maxLifetime > 10_000 ? ThreadLocalRandom.current().nextLong(maxLifetime / 40) : 0;
    final var lifetime = maxLifetime - variance;
    poolEntry.setFutureEol(houseKeepingExecutorService.schedule(new MaxLifetimeTask(poolEntry), lifetime, MILLISECONDS));
}
```

具体来看下MaxLifetimeTask做了什么，这块的代码逻辑比较简单，

```java
private final class MaxLifetimeTask implements Runnable {
    private final PoolEntry poolEntry;

    MaxLifetimeTask(final PoolEntry poolEntry) {
        this.poolEntry = poolEntry;
    }

    public void run() {
        // 驱逐关闭连接
        if (softEvictConnection(poolEntry, "(connection has passed maxLifetime)", false /* not owner */)) {
            // 添加一个新的连接
            addBagItem(connectionBag.getWaitingThreadCount());
        }
    }
}
```

## 总结

这个值是**必须设置**的，值的大小应该比数据库设置的连接存活时间短一些。例如在MySQL中，参数`wait_time`，mysql 为了防止空闲连接浪费，占用资源，在超过wait_timeout 时间后，会主动关闭该连接，清理资源。我们可以通过`show variable like 'wait_time%'`来查看MySQL具体配置的这个时间，默认是28800s，也就是8小时。也就是说，MySQL发现某个连接超过8小时还有没有任何请求，就会自动断开，但是HikariCP如何知道我池子里维护的一把连接，有没有被mysql回收呢？所以就有了`maxLifetime`这个配置，官方也强烈建议必须按需设置此值！自然这个值也应该小于MySQL的`wait_timeout`。

在不考虑idleTimeout参数的情况下，那hikaricp在空闲连接超过maxLifetime，就会从连接池中剔除，防止业务进程取到了已关闭的连接，导致业务受损。

在考虑idleTimeout参数的情况下，若idleTimeout参数为0，直到到达maxLifetime后，才会被移除。这个时间必须小于maxLifetime。

# 参考文章

* [HikariCP探活机制如何保证链接有效](https://blog.csdn.net/m0_46485771/article/details/118583714)
* [一篇文章彻底理解数据库的各种 JDBC 超时参数](https://zhuanlan.zhihu.com/p/582551494)
* [HikariCP - 理解并正确使用配置](https://www.modb.pro/db/215198)