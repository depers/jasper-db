> Redis/缓存

> 之前在日常开发工作中对Redis热key的关注不是很多，这里通过这篇文章对Redis热key的问题做一个反思和总结。

# 热Key的定义

Redis中热键(Hot Key)是指在Redis数据库中被领繁访问的键。比如热门的新闻事件或者是爆款的商品信息都可能会成为热点数据，我们经常为了减轻数据库的压力会将这部分数据缓存到Redis中，极端情况下对热点Key访问可能会超过Redis本身能够承受的OPS（Operations Per Second，每秒执行的操作数量，用于衡量 Redis 系统的性能。），因此热Key监测和解决对于开发和运维人员来说是十分重要的。

# 热key造成的问题

1. 性能瓶颈：大量的访问可能会导致 Redis 服务器的负载增加。
2. 资源竞争：多个线程或进程可能同时尝试访问热键，导致资源竞争。
3. 数据不一致：在高并发情况下，可能会出现数据更新的冲突。

# 如何判断某个Key是热Key呢

解决热Key文件之前，我们首先得监测哪些Key是热Key，在热Key的检测我们可以从以下四个方面来着手：

1. 客户端

    最先想到的方案就是通过客户端工具进行统一的计数统计，这个目前没有找到很好的切入方案，比如我使用Jedis或是spring Data Redis，这两个客户端都没有提供执行命令的拦制器，所以在基础工具上进行统一的封装这个方案不太可行。所以我们只能对这些客户端的每一种命令执行的方法都进行一次封装，也就是在每种操作的代码中都植入我们统计热Key的代码。

    基于固定时间窗口的热key统计算法：

    ```Java
    public class FixedWindowsHotKeyCounter {
    
        private static final AtomicLongMap<String> HOT_KEY_COUNT_MAP = AtomicLongMap.create();
        private long fixedTime;
        private long threshold;
        private long lastTime;
    
    
        public FixedWindowsHotKeyCounter(long fixedTime, long threshold) {
            this.fixedTime = fixedTime;
            this.threshold = threshold;
        }
        
        public void countKey(String key) {
            if (System.currentTimeMillis() - lastTime > fixedTime) {
                HOT_KEY_COUNT_MAP.clear();
                lastTime = System.currentTimeMillis();
            }
            
            if (HOT_KEY_COUNT_MAP.get(key) > threshold) {
                // 记录到数据库或是发送请求到别的系统
            } else {
                HOT_KEY_COUNT_MAP.incrementAndGet(key);
            }
        }
    }
    ```

    这里参考限流算法，我这边分别设计了固定时间窗口的热Key统计方案和滑动时间窗口的热Key统计方案，他们之前的区别和这两种算法的区别其实一致的，固定时间窗口算法无法对突发的流量进行控制，而滑动窗口算法则细化了时间窗口，可以对突发流量进行控制。切换到热Key统计这里，滑动窗口算法可以很好的对一小段时间范围内频繁访问的Key进行准确的统计。

    对客户端进行热点Key的统计，优缺点也很明显:

    * 优点：实现简单
    * 缺点：
        1. 无法预知Key的个数，存在内存泄漏的风险。
        2. 对客户端的代码有侵入，需要针对不同的客户端工具进行二次开发
        3. 只能了解当前客户端的热Key情况，无法实现规模化的统计运维。

2. 代理端

3. 服务端

4. 服务器

# 参考资料

* [《Redis开发与运维》付磊/张益军](https://book.douban.com/subject/26971561/)
