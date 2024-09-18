| title                        | tags             | background                                                   | auther | isSlow |
| ---------------------------- | ---------------- | ------------------------------------------------------------ | ------ | ------ |
| 如何在多线程中传递日志流水号 | 日志/ThreadLocal | 在日常项目开发中，我们经常会使用线程池处理业务，那么就存在主线程日志流水号无法在线程池子线程执行日志中体现的问题，带这个问题，让我们一起看下，如何来解决这个问题。 | depers | true   |

# 如何在Http请求中传递日志流水号

我们在开发Web应用的时候，希望看到一个请求从进入服务到处理完成，整个处理链路上的日志，实现这个功能我们可以使用OncePerRequestFilter这个类来进行实现，因为在Spring提供的几种拦截机制中，从外到内一次是Filter、Inteceptor、ControllerAdvice、Aspect和Controller，最外面的是Filter，所以我们通过在这一层实现日志MDC的设置，来实现流水号的传递。代码示例如下：

# 如何在Quartz批量中传递日志流水号

在Quartz中，大部分的跑批任务都是基于Cron表达式的任务，这里这里以这种跑批任务的日志流水号传递为例来进行说明。具体的代码实践案例可以参考quartz-log这个项目的代码。

在项目的`cn.bravedawn.quartz`包中首先编写了一个`CronParentJob`的类，在这个类中我们实现了`Job`接口，实现了`execute()`方法，具体的代码逻辑如下：

接着来看`cn.bravedawn.quartz`包下的另一个类`TestJob`，这个类中首先也实现了`Job`接口，实现了`execute()`方法，除此之外，我们将这个类声明为了Spring的一个`Bean`，在类定义上使用了`@Component`注解，具体的代码逻辑如下：

最后启动项目，调用http://localhost:8080/createJob接口看下效果，控制台日志输出如下：

# 如何在线程池中传递日志流水号

在处理一些文件读写、大批量数据库操作、网络请求等耗时操作的时候，我们往往会使用线程池来并发和异步来执行这些任务，从而提升任务执行的效率，缩短等待时间。由于是在另一个线程中执行任务，原来触发线程池的主线程携带过来的日志流水号却无法传递给线程池，导致我们在根据日志流水号查询日志的时候，无法和线程池执行的日志进行串联，导致排查问题的时候，不知道当前线程池线程执行的是那个任务的逻辑，无法准确地定位代码中的问题，造成生产损失。

MDC（Mapped Diagnostic Contexts）映射诊断上下文，主要用在做日志链路跟踪时，动态配置用户自定义的一些信息，比如requestId、sessionId等等。MDC使用的容器支持多线程操作，满足线程安全。Log4j2和LogBack都对MDC进行了自己的实现。

Slf4j中MDC API定义的主要方法：

- `clear()`：移除所有MDC
- `get(String key)`：获取当前线程 MDC 中指定 key 的值
- `getCopyOfContextMap()`：将MDC从内存获取出来，再传给线程
- `put(String key, Object o)`：往当前线程的 MDC 中存入指定的键值对
- `remove(String key)`：删除当前线程 MDC 中指定的键值对
- `setContextMap()`：将父线程的MDC内容传给子线程

介绍完了MDC的基本使用，我们来看下具体的实现，首先是`MdcUtil`，这里主要针对Runnable和Callable进行了适配，在线程池执行业务逻辑之前，将父线程的MDC传递给子线程，在子线程执行结束之后将子线程的MDC进行清理，具体的代码逻辑如下：

接着就是对线程池进行封装，讲MDC的逻辑融入线程池的调用逻辑中，代码如下所示，对其`execute()`和`submit()`方法进行封装：

这样我们在业务逻辑中使用线程池，就可以方便的讲异步线程池的日志与我们原有主线程业务逻辑日志进行融合，方便我们排查问题，具体的使用示例和代码如下所示：

更加详细的代码可以访问multi-thread-log进行获取。

# 如何在Netty中传递日志流水号

在项目中如果涉及TCP协议的交易使用Netty去进行封装是十分便利的，相比Java提供的Socket API，Netty对TCP的API封装和性能提供了更好的方案。这里我以Netty客户端的开发为例说明如何讲主线程的流水号传递到客户端线程中，从而方便我们跟踪客户端的行为。

按照之前的经验，如果要串联一段关联操作的日志，只需要找到**连接建立**和**连接关闭**这两个触点代码，在这里补充添加MDC和清空MDC的逻辑就行，在Netty客户端中，这两个触点在哪呢？

- 建立连接触点

    在`Bootstrap.connect()`的`ChannelFuture`上添加监听器，在监听器逻辑中添加MDC。

    ```Java
    boostrap.connect(host, port).addLister((ChannelFutureListener) future     -> {
    MDC.put("log4j2Id", log4j2Id);
    // ...
    }).sync();
    ```

- 关闭连接触点

    在我们拿到响应之后关闭连接时，我们可以添加一个异步监听器，在连接关闭之后执行监听器的逻辑，在这里清除MDC。
    
    ```Java
    channel.close().addListener((ChannelFutureListener) future -> {
        MDC.clear();
    });
    ```


# 参考文章

- [日志之MDC和异步多线程间传递线程id](https://blog.csdn.net/u012060033/article/details/129718315)
- [Spring Boot + MDC 实现全链路调用日志跟踪，这才叫优雅](https://developer.aliyun.com/article/1002910)
- [lovelife-li/netty-study](https://github.hscsec.cn/lovelife-li/netty-study)