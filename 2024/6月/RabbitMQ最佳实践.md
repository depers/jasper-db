| title                                          | tags                     | background                                                   | auther | isSlow |
| ---------------------------------------------- | ------------------------ | ------------------------------------------------------------ | ------ | ------ |
| RabbitMQ的最佳实践一：如何保证消息的可靠性投递 | 消息队列/RabbitMQ/Spring | 最近在项目开发中用到了RabbitMQ，其中涉及到的知识点还是比较多的，想通过这篇文章对RabbitMQ开发中涉及到的点进行一个梳理，方便后续工作中借鉴和扩展。因为这块涉及到的知识点比较多，将分为多个小节来进行阐述，下面先来让我们看看如何保证消息的可靠性投递，保证消息不会丢失。 | depers | true   |

# 消息队列使用的场景

在系统开发中消息队列主要提供了五大功能，分别是**异步**、**解耦**、**削峰填谷**、**广播和冗余**。

## 异步

所谓异步就是任务不再当前线程中继续执行，而是新启了一个线程去执行。在平时开发中能够实现异步操作的另一个法宝是**线程池**，我们可以将任务丢给线程池异步去做执行，但是线程池使用的当前服务器的线程资源，是无法无限制的增加的。**消息队列**同样也能实现异步功能。

## 解构

消息队列可以实现系统级的解耦，可以将一个流程的上下游拆解开，上游专注于生产消息，下游则专注于处理消息。

## 削峰填谷

在应对突发流量，消息队列扮演了缓冲器的作用，保护下游服务，使其可以根据自身的实际消费能力处理消息。

## 广播

上游生产的消息可以轻松被多个下游服务处理。

## 冗余

保留历史消息，处理失败或当出现异常时可以进行重试或者回溯，防止丢失。

# 如何保证消息的可靠性投递

![](../../assert/rabbitmq消息传递流转图.webp)

上面这幅图是RabbitMQ中消息流转的指向图，从图中我们可以看到一个消息从生产到被消费主要经历了以下三步：

1. 生产者发送消息到Broker的Exchange
2. 接着，Broker中消息根据绑定规则由Exchange传递给Queue
3. 最后，Broker和消费者建立连接后，将消息推送给消费者Consumer进行消费

一条消息完整的走完上面三步，我们才认为这条消息是被可靠投递了。这三步中哪个环节出现问题才会导致消息的丢失呢？消息可能丢失的场景主要有以下四个地方：

1. **生产者将消息发送到 RabbitMQ Server 异常**：可能因为网络问题造成 RabbitMQ 服务端无法收到消息，造成生产者发送消息丢失场景。
2. **RabbitMQ Server 中消息在交换机中无法路由到指定队列**：可能由于代码层面或配置层面错误导致消息路由到指定队列失败，造成生产者发送消息丢失场景。
3. **RabbitMQ Server 中存储的消息丢失**：可能因为 RabbitMQ Server 宕机导致消息未完全持久化或队列丢失导致消息丢失等持久化问题，造成 RabbitMQ Server 存储的消息丢失场景。
4. **消费者消费消息异常**：可能在消费者接收到消息后，还没来得及消费消息，消费者宕机或故障等问题，造成消费者无法消费消息导致消息丢失的场景。

按照上述描述的四个场景，让我们一起来看看如何避免上述情况的发生，从而保证消息的可靠性投递，让我们逐个对其进行攻破。

## 1. 如何保证生产者将消息百分百投递给**RabbitMQ Server**

在这个环节，为了保证消息能够百分百投递到RabbitMQ Server，主要有两种方法：

### 事务机制

1. 配置类中配置事务管理器
2. 配置RabbitTemplate时开启事务实现事务机制
3. 在业务代码中使用添加事务注解

通过上面的配置即可实现事务机制，执行流程为：在生产者发送消息之前，开启事务，而后发送消息，如果消息发送至 RabbitMQ Server 失败后，进行事务回滚，重新发送。如果 RabbitMQ Server 接收到消息，则提交事务。

可以发现事务机制其实是**同步操作**，存在阻塞生产者的情况直到 RabbitMQ Server 应答，这样其实会很大程度上**降低发送消息的性能**，所以**一般不会使用事务机制来保证生产者的消息可靠性**，而是使用发送方确认机制。

这块内容的具体代码演示可以参考depers/rabbitmq-spring-reliable-delivery中transaction包下的内容。

### confirm机制

1. 首先需要在application.yml添加配置：

   ```yml
    spring:
      rabbitmq:
        publisher-confirm-type: correlated  # 开启发送方确认机制
   ```

2. 在RabbitTemplate中设置异步确认的回调逻辑

   从日志输出中我们可以看到，http请求、生产者、broker的异步回调和消费者分别是4个线程在执行逻辑：

   上面我们提到采用事务来保证生产者发送消息到broker这段链路的可靠性，是同步的，所以性能一般。而采用confirm机制后，确保了消息从生产者可以成功送达RabbitMQ服务器。当生产者发送消息到RabbitMQ后，如果服务器成功接收消息，它会发送一个确认响应给生产者。这样，生产者就知道消息已经安全到达，就会执行确认的回调方法。**在项目的实际开发中推荐使用confirm机制**。

## 2. 如何在Broker中保证消息能够从交换机投递到队列

 为了保证将消息从交换机顺利地投递到队列中，RabbitMQ提供了mandatory机制，用于控制一个消息到达交换机器时，**交换器根据自身类型和路由键找不到一个符合条件的队列**，若该参数设置为`true`就将该消息返回给生产者，如果为`false`，就将消息丢弃。我们来看具体代码：

1. 先看配置：

    ```yml
    spring:
        rabbitmq:
            # 开启消息返回
            publisher-returns: true
            template:
                # 消息投递失败返回客户端，该参数与publisher-returns同时配合使用
                mandatory: true
    ```

2. 设置RabbitTemplate的`setReturnCallback()`方法设置路由失败后的回调方法：

使用建议：**消息发送时没有必要设置mandotory**，因为Spring Boot和RabbitMQ的集成后，我发现即便是exchage找不到对应的queue后，生产端不仅在`rabbitTemplate`的`ReturnsCallback`中做了返回，而且还在`ConfirmCallback`中也做了`ack`的确认，两个地方同时做了返回，所以我们并不能区分到底成功与否，所以这里其实作用不大，这就需要我们在项目启动的时候做好`exchange`、`queue`和`binding`的创建。

## 3. 如何保证消息在 RabbitMQ Server 中不丢失

 对于消息在Broker的持久化主要有三个要点：

1. 交换机持久化
    1.   在声明交换机时将durable置为`true`实现。
    2.   丢失的信息：保存交换机元数据不会丢失，对于一个长期使用的交换机建议设置为true。
2. 队列持久化
    1.   在声明队列时将durable置为`true`实现。
    2.   丢失的信息：保存队列的元数据和存储在队列中的消息。
    3.   值得注意的是队列的持久化并不能保证消息的持久化。
3. 消息持久化
   
    在通过RabbitTemplate发送消息时，在`convertAndSend()`方法参数`MessagePostProcessor`中设置消息的持久化即可，代码如下：

通过确保消息、交换机、队列的持久化操作可以保证消息的在 RabbitMQ Server 中不丢失，从而保证可靠性。将所有消息设置持久化，需要将消息写入磁盘，会降低吞吐量。对于可靠性不高的消息，可以不做持久化。其实除了持久化之外还需要保证 RabbitMQ 的高可用性，否则 MQ 都宕机或磁盘受损都无法确保消息的可靠性，关键业务队列要设置为镜像队列

## 4. 如何保证消费者消费的消息不丢失

首先我们来看下消息消费的关键配置：**消费模式**，消费者主要有三种消费模式：

- `MANUAL`：手动确认，必须要收到ack或者nack。**如果消息消费不需要重试且出现异常又要保证消息不能丢失，建议使用该模式。**
- `AUTO`：程序在执行中，消息会保存，直到程序正常执行完成或者抛出异常才删除消息。**如果消息消费出现异常需要重试，就需要使用该模式。**
- `NONE`：发送到消费端后就自动确认，消息被删除。**不建议配置该模式，如果消息消费异常，就会导致消息丢失。**

下面我们通过代码来简单说明`MANUAL`模式和`AUTO`模式的使用。

### Manual模式

在该模式下如果消息在消费的过程中出现异常，我们可以选择将消息重新入队，方便后续再次对其进行消费，从而可以保证消息不被丢失。

如果该队列配置了死信队列，在遇到异常的时候，我们可以通过`channel.basicNack(index, false, false)`将该条消息传递到死信队列中。切记这里的`requeue`参数为`false`。

1. 首先我们来看配置文件：

    ```yaml
    spring:
      rabbitmq:
        listener:
          simple:
            acknowledge-mode: manual  # 全局开启手动确认消费机制
    ```

2. 消费端消费代码：

    ```Java
    /**
     * 监听消费队列的消息
     */
    @RabbitListener(queues = RabbitMQConfig.MESSAGE_QUEUE)
    public void onMessage(Message message, Channel channel) {
        // 获取消息索引
        long index = message.getMessageProperties().getDeliveryTag();
        // 解析消息
        byte[] body = message.getBody();
        ...
        try {
            // 业务处理
            ...
                // 业务执行成功则手动确认
                channel.basicAck(index, false);
        }catch (Exception e) {
            // 记录日志
            log.info("出现异常：{}", e.getMessage());
            try {
                // 手动丢弃信息
                channel.basicNack(index, false, true);
            } catch (IOException ex) {
                log.info("丢弃消息异常");
            }
        }
    }
    ```

    大家可以看到上面步骤中channel.basicAck()和channel.basicNack()两个方法分别代表了消息确认和消息拒绝两个应答操作，让我们详细来看下他们的具体参数：

    `void basicAck(long deliveryTag, boolean multiple)`方法（会抛异常）： 
    
    - deliveryTag：该消息的index
    - multiple：是否批量处理（true 表示将一次性ack所有小于deliveryTag的消息）
    
    `void basicNack(long deliveryTag, boolean multiple, boolean requeue)`方法（会抛异常）： 
    
    - deliveryTag：该消息的index
    - multiple：是否批量处理（true 表示将一次性ack所有小于deliveryTag的消息）
    - requeue：被拒绝的是否重新入队列（true 表示添加在队列的末端；false 表示丢弃）

**问题：什么情况下`channel.basicAck()`方法的第二个参数`multiple`在什么场景下应该设置为true？**

当你想要一次性确认多条消息时，可以将 `multiple` 设置为 `true`。这在消费者能够快速且可靠地处理消息时特别有用，因为它可以减少与RabbitMQ服务器的网络往返次数，从而提高吞吐量。示例代码如下：

```Java
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;
import com.rabbitmq.client.Channel;

@Component
@RabbitListener(queues = "yourQueueName")
public class YourMessageHandler {

    // 这里消费者现成应该只有一个，如果有多个应该使用AtomicLong或是LongAddr
    private long lastAckedDeliveryTag = 0;

    @RabbitHandler
    public void handleMessage(Message message, Channel channel) throws IOException {
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        try {
            // 处理消息的业务逻辑
            // ...

            // 更新最后一个已确认的deliveryTag
            lastAckedDeliveryTag = deliveryTag;

            // 如果累积了足够多的消息，或者达到了某个逻辑点，执行批量确认
            if (shouldBatchAck()) {
                channel.basicAck(lastAckedDeliveryTag, true);
                // 重置最后一个已确认的deliveryTag
                lastAckedDeliveryTag = 0;
            }
        } catch (Exception e) {
            // 处理失败，拒绝消息
            channel.basicNack(deliveryTag, false, true);
        }
    }

    private boolean shouldBatchAck() {
        // 这里可以是你的逻辑，例如：
        // - 达到了某个消息数量
        // - 处理时间超过了某个阈值
        // - 处理的消息数量是某个批次大小的整数倍
        return lastAckedDeliveryTag % 100 == 0; // 例如每100条消息确认一次
    }
}
```

### Auto模式

**如果我们想使用Spring AMQP提供的自动重试功能**，那么我们就应该使用Auto模式，在这种模式下，如果消息消费出现异常，这条消息会按照我们配置的次数和频率进行多次重试，重试是在消费者当前服务器进行的，如果消息重试到达最大重试次数，此时消息就会被丢失。

**如果不需要重试**，如果消息被正常处理，则消息在处理完之后就会被自动确认；如果消息出现异常，在抛出异常之后，消息也会被自动确认，在这种情况下消息会因为消费异常而出现消息丢失的情况。

这种模式的具体使用如下：

1. 首先我们来看配置，这里我们通过Java Bean的方式对消费者进行配置，这样做的好处是我们可以为每一个消费者配置特定的消费者配置，从而个性化定制每个消费者的消费逻辑，不同于上面Manual模式的配置文件配置，配置文件配置是全局的配置。

    ```Java
    @Configuration
    @Slf4j
    public class RabbitmqConfig {
    
        /**
         * 声明消息的重试机制
         * @return
         */
        @Bean
        public RetryOperationsInterceptor retryInterceptor() {
            return RetryInterceptorBuilder.stateless()
                .maxAttempts(3)
                .backOffOptions(1000, 3.0, 10000)
                .recoverer(messageRecover())
                .build();
        }
    
        /**
         * 消息到达最大重试次数的处理逻辑
         * @return
         */
        public MessageRecoverer messageRecover() {
            return new MessageRecoverer() {
                @Override
                public void recover(Message message, Throwable cause) {
                    log.warn("消息重试执行失败, msg={}, cause={}", message.getBody(), cause);
                }
            };
        }
    
        /**
         * 使用这个工厂类来生成并发容器
         * @param connectionFactory
         * @param onceRetryInterceptor
         * @return
         */
        @Bean("simpleContainerFactory")
        public SimpleRabbitListenerContainerFactory simpleRabbitListenerContainerFactory(ConnectionFactory connectionFactory,
                                                                                         RetryOperationsInterceptor onceRetryInterceptor) {
            SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
            // 设置连接
            factory.setConnectionFactory(connectionFactory);
            // 开启自动确认模式
            factory.setAcknowledgeMode(AcknowledgeMode.AUTO);
            // 在simple线程模式下，多个消费者共享一个channel，该配置是指prefetchCount应全局应用于channel还是应用于channel上的每个消费者，这里设置为false，作用于每个消费者
            factory.setGlobalQos(false);
            // 设置消费者初始数量
            factory.setConcurrentConsumers(2);
            // 设置消费者最大并发数，也就是消费者的最大数量
            factory.setMaxConcurrentConsumers(2);
            // 设置每一个消费者阻塞队列的大小
            factory.setPrefetchCount(10);
            // 设置消费者标签
            factory.setConsumerTagStrategy(queue -> "consumer" + (index.incrementAndGet()));
            // 设置重试逻辑
            factory.setAdviceChain(onceRetryInterceptor);
            return factory;
        }
    }
    ```
    
2. 给消费者配置并发容器，代码如下：

    ```Java
    @Component
    @Slf4j
    public class MessageConsumer {
    
        @RabbitListener(queues = RabbitmqConfig.QUEUE, containerFactory = "simpleContainerFactory")
        public void receive(String msg, Message message, Channel channel) throws IOException {
            Thread thread = Thread.currentThread();
            log.info("Channel:" + channel.getChannelNumber()
                     + "  ThreadId is:" + thread.getId()
                     + "  ConsumerTag:" + message.getMessageProperties().getConsumerTag()
                     + "  Queue:" + message.getMessageProperties().getConsumerQueue()
                     + "  msg:" + msg);
        }
    }
    ```

# 参考项目

- depers/rabbitmq-spring-reliable-delivery
- depers/rabbitmq-spring-boot-manual-consumer
- [depers/rabbitmq-spring-boot-thread-model-simple-consumer](https://github.com/depers/JavaMall/tree/master/mq/rabbitmq-spring-boot-thread-model-simple-consumer)

# 参考文章

* [如何保证 RabbitMQ 的消息可靠性](https://juejin.cn/post/7228864364744507450)
* [为什么使用消息队列？引入MQ会为我们带来哪些优势？](https://juejin.cn/post/6966041461863219236)
* [RabbitMQ 可靠性、重复消费、顺序性、消息积压解决方案](https://juejin.cn/post/6977981645475282958)
* [一篇文搞定消息队列选型](https://mp.weixin.qq.com/s/hn6VWEmuTiDvxHQ9EmbjlA)
* [消息队列知识点汇总](http://www.bravedawn.cn/details.html?aid=8495)