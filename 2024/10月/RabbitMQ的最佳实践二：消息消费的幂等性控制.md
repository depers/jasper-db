| title                                      | tags                     | background                                                   | auther | isSlow |
| ------------------------------------------ | ------------------------ | ------------------------------------------------------------ | ------ | ------ |
| RabbitMQ的最佳实践二：消息消费的幂等性控制 | 消息队列/RabbitMQ/Spring | 最近在项目开发中用到了RabbitMQ，其中涉及到的知识点还是比较多的，想通过这篇文章对RabbitMQ开发中涉及到的点进行一个梳理，方便后续工作中借鉴和扩展。这篇文章是这一系列的第二篇文章，我们来讨论消息消费端的幂等性控制。 | depers | true   |

# 概念明确

在这里我讲三个概念，分别是**幂等性**、**防重放**和**防重复点击**。这三个概念都和**重复**有着有着一定的关系。

# 幂等性

## 1. 概念解释

首先我们来看幂等性控制，这里我们一般指的是Web服务中的接口幂等性，在HTTP/1.1中，对幂等性进行了定义。它描述了一次和多次请求某一个资源对于资源本身应该具有同样的结果（网络超时等问题除外），即第一次请求的时候对资源产生了副作用，但是以后的多次请求都不会再对资源产生副作用。

用人话来说就是有一个扣减库存的接口，会将商品的库存减一，如果订单A第一次请求扣减库存后，后面接着多次请求都不再扣减库存了，因为一个订单只会扣减一次库存。如果在这里我们不做幂等性控制，就可能会造成库存被多次扣减的情况。

值得注意的是幂等性和并发控制是两个概念，千万不要混淆。

## 2. 在Web开发中出现幂等性问题的场景

- **前端重复提交表单**： 在填写一些表格时候，用户填写完成提交，很多时候会因网络波动没有及时对用户做出提交成功响应，致使用户认为没有成功提交，然后一直点提交按钮，这时就会发生重复提交表单请求。
- **用户恶意进行刷单**： 例如在实现用户投票这种功能时，如果用户针对一个用户进行重复提交投票，这样会导致接口接收到用户重复提交的投票信息，这样会使投票结果与事实严重不符。
- **接口超时重复提交**：很多时候 HTTP 客户端工具都默认开启超时重试的机制，尤其是第三方调用接口时候，为了防止网络波动超时等造成的请求失败，都会添加重试机制，导致一个请求提交多次。
- **消息进行重复消费**： 当使用 MQ 消息中间件时候，如果发生消息中间件出现错误未及时提交消费信息，导致发生重复消费。

## 3. RestFul API中接口的幂等性

在Restful 推荐的几种 HTTP 接口方法中，分别存在幂等行与不能保证幂等的方法，如下：

| http方法 | 是否幂等         | 解释                                                         |
| :------- | :--------------- | :----------------------------------------------------------- |
| GET      | 是               | GET方法一般用来执行查询操作，不会对系统资源进行变更，所以是**天然幂等**的。 |
| POST     | 否               | PSOT操作一般用于执行新增操作，所以多次执行就会新增多条数据，**不是幂等**的。 |
| PUT      | 需要根据场景区分 | PUT操作一般用于修改操作，修改一般分为更新某个特定字段或是对某个字段进行累加，前者多次执行是**幂等**的，后者则每次修改后的结果都不同，所以**不是幂等**的。 |
| DELETE   | 需要根据场景区分 | DELETE操作一般用于删除操作，删除一般分为按唯一主键删除或是按条件删除，前者多次执行删除操作是**幂等**的，后者如果第一次删除后，有新增了一批数据，再次删除就会导致误删数据，所以**不是幂等**的。 |

## 4. 数据库操作的幂等性

在执行数据库操作时，与RestFul API一样不同的操作在幂等性方面也有着不同的表现，如下：

| 数据库操作 | 是否幂等         | 解释                                                         |
| ---------- | ---------------- | ------------------------------------------------------------ |
| SELECT     | 是               | 不会对业务数据有影响，**天然幂等**。和RestFul API的GET方法一样。 |
| INSERT     | 否               | 多次执行**不能保证幂等性**。和RestFul API的POST方法一样。    |
| UPDATE     | 需要根据场景区分 | 和RestFul API的PUT方法一样。                                 |
| DELETE     | 需要根据场景区分 | 和RestFul API的DELETE方法一样。                              |

## 5. 解决幂等性问题的方法

这里我们再次回顾下幂等性的概念，就是多次请求的效果和第一次请求的效果相同，但是不能对系统造成过多副作用，如何保证除第一次外其余请求不能对系统造成副作用呢，首先我们需要对第一次的请求进行**标记**和**存储**，这样在后续的请求中我们就需要去查询标记，判断是否继续执行请求的操作。

上面提到了两个点，**第一个是标记，标记具有唯一性**，所以生成唯一性标记的方法有很多，下面我提几个思路：

- 根据特定的一个或多个业务字段使用md5生成的token
- 特定的业务字段本身就是唯一的，比如订单的id

大家可以看到上面两个思路用到的都是业务相关字段生成的标记，也有人说直接用uuid不行吗？我觉得直接使用uuid这种随机无状态的标记是有问题的，比如新增一个商品，如果使用uuid来做标记，那么同一个商品就有可能会被新增两次，所以**幂等性的标记必须和业务字段相关才行，或者说同一个商品在新增之前就生成好了他的id，多次点击保存id是相同的，但是不能在每次新增的时候都重新生成**。

接着就是**存储，实现存储的方案有很多，最常见的就是数据库和缓存**，比如MySQL和Redis。

明确了上面这两点之后，我们来看具体的应用方案有哪些：

1. **方法一：数据库去重表**

往去重表里插入数据的时候，利用数据库的唯一索引特性，保证唯一的逻辑。唯一序列号可以是一个字段，例如订单的订单号，也可以是多字段的唯一性组合。例如设计如下的数据库去重表。

```SQL
CREATE TABLE `idempotent` (
  `id` int(11) NOT NULL COMMENT 'ID',
  `serial_no` varchar(255)  NOT NULL COMMENT '唯一序列号',
  `source_type` varchar(255)  NOT NULL COMMENT '资源类型',
  `remark` varchar(255)  NOT NULL COMMENT '备注',
  `create_by` bigint(20) DEFAULT NULL COMMENT '创建人',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  PRIMARY KEY (`id`)
  UNIQUE KEY `uk_serial_no_source_type` (`serial_no`,`source_type`)  COMMENT '保证业务唯一性'
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='幂等性校验表';
```

上表中我们需要重点关注的`serial_no`和`source_type`字段，这里我们使用了数据库的唯一索引来对这两个字段进行存储，其中`serial_no`是特定的一个或多个业务字段使用md5生成的token，而`source_type`字段则是不同的标记类型，比如提交订单、短信消息消费等。

落实到具体的开发层面，我们可以声明一个注解，注解可以这么写：

```Java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Idempotent {
    /**
    * 资源类型
    */
    String sourceType();
    
    /**
    * controller接口的参数，支持SPEL表达式
    */
    String key();
}
```

然后在每个Controller的方法上使用该注解，然后通过Spring AOP切面去拦截该注解，在切面逻辑中通过解析`Idempotent.key`和请求参数，从而得到每条业务数据的参数标识，接着通过md5算法进行摘要，将资源类型和摘要入库，如果入库报`DuplicateKeyException`异常，则表示该业务记录已被处理，这里只需正常返回即可，无需继续下面的流程。

如果入库正常则执行具体的接口逻辑，但是如果接口逻辑执行出现异常，这里就涉及到是否允许用户再次调用。如果允许用户再次调用，接口中需要抛出特定的异常，接着被该切面捕获，接着删除去重表中的这条记录；如果不允许，则接口不要抛出特定的异常，也就不允许用户再次调用。

采用数据库去重表的优点：**实现简单**，缺点：**会占用大量的存储空间**，因为随着业务数据的增多，之前的业务数据可能无需再进行去重处理了，比如一个订单支付接口，距离第一次请求已经一个月了，再次被调用，此时订单的状态已经发生了变化，即便是重复请求再业务逻辑层面也会被拦截。如果我们还保留之前的去重表数据，就会浪费数据库的存储空间。

这种方法是通用的幂等性控制方法，也就是上面的post、put和delete的幂等性问题都可以得到解决。

2. **方法二：Redis去重**

常用的数据存储中，除了数据库就是Redis了，和方法一一样，我们在切面逻辑中可以将数据库入库的动作切换成Redis的`SETNX`操作，`SETNX` 是一个原子性命令，用于设置键值对，但只在键不存在的情况下。它的全称是 "SET if not exists"。这个命令通常用于实现分布式锁或者确保某个值的唯一性。如果键 `key` 不存在，那么 `SETNX` 会设置键 `key` 的值，并返回 `1` 表示操作成功。如果键 `key` 已经存在，那么 `SETNX` 命令不做任何操作，并返回 `0` 表示操作失败。

值得注意的是在Redis中存储数据需要为键值对设置有效时间 ，从而防止内存占用过高，浪费内存的情况。所以这里建议如果针对幂等性控制的键值对应该根据具体的情况来进行设置。

采用Redis去重的优点：**一实现简单；二不会占用大量的空间**，因为每条去重数据都会被加上有效时间。缺点：**引入了新的组件，给系统的可用性增加复杂度**。

这种方法也是通用的幂等性控制方法，也就是上面的post、put和delete的幂等性问题都可以得到解决。

3. **方法三：乐观锁机制**

乐观锁机制一般用于update更新操作，在乐观锁的设计中，系统假设多个事务或操作可以并发地执行，而不会发生冲突。只有在数据提交更新时，乐观锁才会正式检测数据是否发生了冲突。如果检测到冲突，系统会返回错误信息，让用户决定如何处理这种情况。

乐观锁的实现通常基于两种机制：数据版本（Version）记录机制和CAS（Compare And Swap）算法。这里我们以数据版本记录机制来展开讨论。假设我们有一张这样的数据表：

| id   | name     | price |
| :--- | :------- | :---- |
| 1    | 小米手机 | 1000  |
| 2    | 苹果手机 | 3000  |
| 3    | 华为手机 | 2000  |

为了每次执行更新时防止重复更新，通常都会添加一个 `version` 字段记录当前的记录版本，这样在更新时候将该值带上，那么只要执行更新操作就能确定一定更新的是某个对应版本下的信息。

| id   | name     | price | version |
| :--- | :------- | :---- | :------ |
| 1    | 小米手机 | 1000  | 2       |
| 2    | 苹果手机 | 3000  | 3       |
| 3    | 华为手机 | 2000  | 4       |

假设我们的更新的sql如下，我们需要将小米手机的价格增加50块钱：

```SQL
update my_table set price = price + 50, version = version + 1 where id = 1 and version = 2;
```

上面 `WHERE` 后面跟着条件 `id=1 AND version=5` 被执行后，`id=1` 的 `version` 被更新为 `6`，所以如果重复执行该条 SQL 语句将不生效，因为 `id=1 AND version=5` 的数据已经不存在，这样就能保住更新的幂等性，多次更新对结果不会产生影响。

在使用这种方法的时候，我们需要将id和version都作为条件传进来，比如在web管理端的查询页面我们就需要将version返回给前端，这样在前端修改数据的时候，就需要前端将version也传回后端。**相比较于前两种办法，实现层面相对复杂且只适用update操作，所以一般不建议使用这种方式。**

4. **方法四：状态机**

状态机这种方式是和业务代码强绑定的，**一般在具体流转状态的业务模型中才会使用**。比如订单模型，它有固定的流转状态，未提交、已提交、待付款、付款成功、付款失败等等状态。

在每一次业务流转的代码逻辑中，我们都先要判断对象的状态，如果状态不符合状态判断，则中断业务流程。比如订单支付接口，我们首先就要判断订单是否已经支付，也就是判断订单的状态是不是待支付，如果不是则任务不能继续支付。

这种方式一般要结合方法一或方法二来配合使用。

# 防重放

## 1. 概念解释

防重放攻击是一种网络安全策略，旨在保护数据传输过程中的安全，防止攻击者通过截获并重新发送数据包来欺骗系统。主要用于身份认证过程。

重放攻击是二次请求，黑客通过抓包获取到了请求的HTTP报文，然后黑客自己编写了一个类似的HTTP请求，发送给服务器。也就是说服务器处理了两次相同的请求，先处理了正常的HTTP请求，然后又处理了黑客发送的篡改过的HTTP请求。

## 2. 应用场景

防重放攻击的应用场景主要有以下场景：

1. 身份认证：比如在用户登录场景下，攻击者截取用户登录的报文进行重复登录。
2. 电子支付：在移动支付、网上银行转账等场景中，防重放攻击技术可以防止攻击者重复提交支付请求或转账请求，确保交易的安全性和准确性。例如，用户在使用手机支付时，支付请求需要经过防重放攻击技术的验证，防止攻击者重复提交支付请求来骗取资金。
3. 在线投票：比如在投票或竞猜等活动中，攻击者通过重复投票或作弊来影响结果。

诸如此类的场景还有很多，落地到我们现在的开发场景中，由于目前我们很多对外接口都是RestFul API风格的，所以在设计`POST`、`PUT`和`DELETE`交易的时候，我们就要关注是否要为该交易增加防重放的设计。

## 3. 解决防重放攻击的方法

1. **加随机数**：**该方法优点是认证双方不需要时间同步**，双方记住使用过的随机数，如发现报文中有以前使用过的随机数，就认为是重放攻击。**缺点是需要额外保存使用过的随机数**, 若记录的时间段较长, 则保存和查询的开销较大.
2. **加时间戳**：**该方法优点是不用额外保存其他信息。缺点是认证双方需要准确的时间同步**, 同步越好， 受攻击的可能性就越小。但当系统很庞大，跨越的区域较广时，要做到精确的时间同步并不是很容易。
3. **加流水号**：就是双方在报文中添加一个逐步递增的整数, 只要接收到一个不连续的流水号报文(太大或太小)，就认定有重放威胁。该方法优点是不需要时间同步，保存的信息量比随机数方式小。缺点是一旦攻击者对报文解密成功，就可以获得流水号，从而每次将流水号递增欺骗认证端。

**在实际使用中，常将1和2结合使用，时间戳有效期内判断随机数是否已存在，有效期外则直接丢弃。**

## 4. 具体实践

在具体的项目开发中，我们一般会使用`时间戳`+`随机数`的方式来实现一个简单的重放攻击拦截器。时间戳和随机数互补，既能在时间有效范围内通过校验缓存中的随机数是否存在来分辨是否为重放请求，也能够减小大量重放数据的存储。

**防重放逻辑的实现主要有五部分，第一是参数校验，第二是验证时间戳，第三步判断随机数nonce是否已经存在，第四步是生成签名进行比较，第五步是将随机数nonce保存到redis中** 。具体的代码实现参考下面的代码：

```Java
package cn.bravedawn.aspect;

import cn.bravedawn.constant.RedisKey;
import cn.bravedawn.exception.BusinessException;
import cn.bravedawn.exception.ExceptionEnum;
import cn.bravedawn.util.RedissonUtils;
import cn.bravedawn.util.SpringContextUtil;
import cn.hutool.crypto.digest.DigestUtil;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;

import java.util.Map;
import java.util.Objects;
import java.util.TreeMap;

/**
 * @author : depers
 * @program : replay-protect-demo
 * @date : Created in 2024/10/29 21:12
 */

@Slf4j
public class ReplayProtection {

    private volatile static ReplayProtection instance = null;
    private final String secret = "";

    private ReplayProtection() {
        log.info("防重放组件初始化");
    }

    public static ReplayProtection getInstance() {
        if (instance == null) {
            synchronized (ReplayProtection.class) {
                if (instance == null) {
                    instance = new ReplayProtection();
                }
            }
        }
        return instance;
    }

    private String generatesign(TreeMap<String, String> param) {
        StringBuilder stringBuilder = new StringBuilder();
        param.forEach((key, value) -> stringBuilder.append(key).append("=").append(value).append("&"));
        stringBuilder.append("secret=").append(secret);
        log.info("未哈希的字符中{}", stringBuilder);
        return DigestUtil.md5Hex16(stringBuilder.toString());
    }

    public void validateSign(String timestamp, String nonce, String sign, Map<String, Object> requestParams) {
        if (StringUtils.isBlank(sign)) {
            log.error("签名不能为空");
            throw new BusinessException(ExceptionEnum.REPLAY_PROTECT_PARAM_ERROR);
        }
        if (StringUtils.isBlank(nonce)) {
            log.error("随机数不能为空");
            throw new BusinessException(ExceptionEnum.REPLAY_PROTECT_PARAM_ERROR);
        }
        if (StringUtils.isBlank(timestamp)) {
            log.error("时间戳不能为空");
            throw new BusinessException(ExceptionEnum.REPLAY_PROTECT_PARAM_ERROR);
        }
        Long timestampLong = Long.parseLong(timestamp);
        if (timestampLong - System.currentTimeMillis() > 60000) {
            log.info("时间戳已过期, timestamp={}", timestamp);
            throw new BusinessException(ExceptionEnum.REPLAY_PROTECT_SIGN_OVERDUE_ERROR);
        }
        String key = String.format(RedisKey.REPLAY_PROTECT_NONCE.getKey(), nonce);
        RedissonUtils redissonutils = (RedissonUtils) SpringContextUtil.getBean(RedissonUtils.class);
        if (redissonutils.get(key) != null) {
            log.error("随机字符串有Redis中已存在，nonce={}", nonce);
            throw new BusinessException(ExceptionEnum.REPLAY_PROTECT_REPEAT_ERROR);
        }
        TreeMap<String, String> sortedMap = new TreeMap<>();
        requestParams.entrySet().stream().filter(entry -> Objects.nonNull(entry.getValue()) && Objects.nonNull(entry.getKey())).forEach(entry ->
                sortedMap.put(entry.getKey(), String.valueOf(entry.getValue())));


        // 将头信息也放进去
        sortedMap.put("timestamp", timestamp);
        sortedMap.put("nonce", nonce);
        sortedMap.put("sign", sign);

        String hashsign = generatesign(sortedMap);
        if (!StringUtils.equals(sign, hashsign)) {
            log.error("签名验证不通过 sign={},hashsign={}", sign, hashsign);
            throw new BusinessException(ExceptionEnum.REPLAY_PROTECT_SIGN_VERIFY_ERROR);
        }
        redissonutils.set(key, "1", RedisKey.REPLAY_PROTECT_NONCE.getTtl());

    }

}
```

在上面的实现中`validateSign()`方法的`timestamp`、`nonce`和`sign`字段都是从http请求头中获取的，签名的验证就为了校验请求参数是否被人篡改，所以除了请求头的信息，我们还需要报文体的内容。在Spring Web中读取请求的body并不是一件容易的事，因为`HttpServletRequest`的字节流只能被读取一次，如果我们在防重放这里读取了`HttpServletRequest`的内容，那么在后续的业务逻辑的`Controller`就会出现问题。Spring也提供了一个类`ContentCachingRequestWrapper`允许我们重复的读取`HttpServletRequest`中的内容，但是我这边试验过，这个类兼容性不是很好，所以针对预先读取`HttpServletRequest`中的内容，这里**建议通过切面进行获取**。

# RabbitMQ中消息消费的幂等性控制

## 1. 消息消费需要幂等性控制的原因

为了防止同一条消息被多次消费，我们需要在消费端为消息消费添加幂等性控制的逻辑。首先我们来思考下，什么情况下消息会出现多次被消费的情况？这里我列举几种比较常见的情况：

1. **网络问题导致的消息重传**

    当消费者从消息队列获取消息后，在向消息队列发送确认（ACK）消息的过程中，如果网络出现抖动或短暂中断，消息队列可能无法收到 ACK。例如，在基于 AMQP（高级消息队列协议）的 RabbitMQ 系统中，消费者在执行完业务逻辑后向 RabbitMQ 发送 ACK，但由于网络波动，ACK 丢失。消息队列会认为该消息未被消费，从而在超期后重新将消息发送给消费者。

2. **消费者业务逻辑处理失败触发重试**

    当消费者在处理消息时，如果业务逻辑出现错误（如数据库插入操作失败、调用外部服务超时等），消息队列通常会有重试机制。以 RabbitMQ 为例，如果消费者在消费消息时抛出异常，RabbitMQ 会根据配置的重试策略（如重试次数、重试间隔等）将消息重新发送给消费者。如果在重试过程中，没有对消息的重复性进行有效判断，就可能导致同一条消息被多次处理。

3. **消息生产端的重复发送**

    消息生产端在发送消息时，为了确保消息发送成功，往往会有重试机制。以RabbitMQ为例，为了保证消息在生产端的可靠性传递，我们会采用confirm机制+重发批量的逻辑去保证消息的可靠性投递，如果超过一定的时间没有收到Broker的确认，批量就会对超时未确认的消息进行重发，但是如果因为网络问题导致生产者没有收到Broker的确认消息，就会导致生产者再次重发该消息，从而导致消息被消费端多次消费。

## 2. 在消费端如何保证消息消费的幂等性

保证消息消费的幂等性顾名思义就是要保证消息有且只能被消费一次，如何保证消息只能被消费一次呢？大体的思路就是为每条消息添加唯一标识符（如UUID），在消费时检查该标识符是否已经处理过。这里消息的id生成我们采用雪花算法，存储我们使用Redis，通过Redis去重的功能来实现幂等性的控制。具体的代码实现如下：

```Java
import cn.bravedawn.constant.MQRedisKey;
import cn.bravedawn.exception.BusinessException;
import cn.bravedawn.model.bo.MQMessage;
import cn.bravedawn.util.RedissonUtils;
import cn.bravedawn.util.SnowflakeUtil;
import cn.hutool.json.JSONUtil;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.redisson.api.RedissonClient;
import org.slf4j.MDC;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * @author : depers
 * @program : replay-protect-demo
 * @date : Created in 2024/11/2 22:13
 */
@Aspect
@Component
@Slf4j
public class RabbitMQListenerAspect {

    @Autowired
    private RedissonUtils redissonUtils;


    @Autowired
    private RedissonClient redissonClient;

    // 定义切点
    @Pointcut("@annotation(org.springframework.amqp.rabbit.annotation.RabbitListener)")
    public void pointcut() {

    }


    @Around("pointcut()")
    public void around(ProceedingJoinPoint joinPoint) throws Throwable {
        MDC.put("log4i2Id", String.valueOf(SnowflakeUtil.getId()));
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Object[] args = joinPoint.getArgs();
        MQMessage message = (MQMessage) args[0];
        String targetMethod = signature.getMethod().getClass().getName() + "." + signature.getMethod().getName();
        log.info("对消息进行幂等性控制,message={}, method={}", JSONUtil.toJsonStr(message), targetMethod);
        String idempotenceKey = String.format(MQRedisKey.MQ_CONSUME_FLAG.getKey(), message.getMessageId());
        boolean res = redissonUtils.setNx(idempotenceKey, "1", MQRedisKey.MQ_CONSUME_FLAG.getTtl());
        if (res) {
            log.info("通过幂等性校验，messageId-{}", message.getMessageId());
        } else {
            log.warn("幂等性校验不通过，messageId={}", message.getMessageId());
            return;
        }
        Throwable cause = null;
        try {
            String timeKey = String.format(MQRedisKey.REDIS_MQ_CONSUME_TIME.getKey(), message.getMessageId());
            long time = redissonUtils.incr(timeKey, MQRedisKey.REDIS_MQ_CONSUME_TIME.getTtl());
            log.info("第{}次消费消息，messageId={}", time, message.getMessageId());
            long startTime = System.currentTimeMillis();
            joinPoint.proceed();
            log.info("消息消费完成, messageId={}, 耗时={}ms", message.getMessageId(), System.currentTimeMillis() - startTime);
        } catch (Throwable e) {
            log.error("消息消费出现异常", e);
            redissonUtils.del(idempotenceKey);

            if (e instanceof BusinessException) {
                cause = e;
            } else {
                cause = new BusinessException("消息消费异常，请联系管理员");
            }
        } finally {
            if (cause != null) {
                throw cause;
            }
            MDC.clear();
        }
    }
}
```

# 防重复点击

在上面幂等性一节的描述中，我们提到了在Web开发中常见的幂等性问题，其中主要是用户重复点击前端菜单按钮导致的。防重复点击是幂等性控制的一部分，是我们具体开发中的实现落地，这里大家不要混淆，之所以拿出来单独来说是因为在日常Web开发中这块逻辑是比较常用的。

因为这块逻辑和前端是强绑定的，只需要那些为前端点击提供服务的接口去做适配。首先我们来看下具体的实现思路：

1. 声明幂等性控制的注解

    可以看到这个注解一共有五个属性，其中常用的属性是前三个`prefix`、`key`和`duration`，剩余的两个属性，保持默认就行。

    **值得一提的是这里的duration字段**，这里我默认设置了45s，原因是前端请求后端接口的超时时间是40s，假如这里我将允许重复点击的时间设置为20s，一个接口请求后端超过了20s还没有拿到后端的响应，在21s的时候又可以点击并请求后端接口，很有可能之前一个请求还没有处理完。所以这里将重复点击的时间间隔设置为了45s，也就是说**要大于前端请求的超时时间**。

    ```Java
    public @interface Idempotent {
    
        /**
         * 前缀
         */
        String prefix();
    
        /**
         * 唯一性标识，支持SPEL表达式
         */
        String key();
    
        /**
         * 幂等控制时长，默认45秒
         */
        int duration() default 45;
    
        /**
         * 在具体逻辑执行结束之后是否删除Key
         */
        boolean removeKeyWhenFinished() default false;
    
        /**
         * 在具体逻辑执行出现异常之后是否移除Key
         */
        boolean removeKeyWhenError() default false;
    }
    ```

2. 幂等性切面的具体逻辑

    在下面的代码中可以看到，我们使用了Spring提供的SPEL表达式，使用SPEL表达式就可以让我们方便接解析出接口参数中的内容，从而方便我们可以针对每一条业务数据进行去重。具体的去重逻辑如下。这个切面的大体实现思路是：第一通过解析接口参数从而获得Redis存储的key值；第二使用Redis的setnx命令保存去重信息，如果保存成功则继续业务逻辑，如果保存失败则报错。具体的代码逻辑如下：

    ```Java
    import cn.bravedawn.annotation.Idempotent;
    import cn.bravedawn.constant.RedisKey;
    import cn.bravedawn.exception.BusinessException;
    import cn.bravedawn.exception.ExceptionEnum;
    import cn.bravedawn.util.RedissonUtils;
    import jakarta.servlet.http.HttpServletRequest;
    import jakarta.servlet.http.HttpServletResponse;
    import lombok.extern.slf4j.Slf4j;
    import org.apache.commons.lang3.StringUtils;
    import org.aspectj.lang.ProceedingJoinPoint;
    import org.aspectj.lang.annotation.Around;
    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.Pointcut;
    import org.aspectj.lang.reflect.MethodSignature;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.core.DefaultParameterNameDiscoverer;
    import org.springframework.expression.EvaluationContext;
    import org.springframework.expression.Expression;
    import org.springframework.expression.common.TemplateParserContext;
    import org.springframework.expression.spel.standard.SpelExpressionParser;
    import org.springframework.expression.spel.support.StandardEvaluationContext;
    import org.springframework.stereotype.Component;
    import org.springframework.web.context.request.RequestContextHolder;
    import org.springframework.web.context.request.ServletRequestAttributes;
    
    import java.util.List;
    
    /**
     * @author : depers
     * @program : idempotence-demo
     * @date : Created in 2024/11/21 21:56
     */
    @Aspect
    @Component
    @Slf4j
    public class IdempotentAspect {
    
        @Autowired
        private RedissonUtils redissonUtils;
    
        private final SpelExpressionParser spelExpressionParser = new SpelExpressionParser();
    
    
        @Pointcut("@annotation(cn.bravedawn.annotation.Idempotent)")
        public void idempotent() {
        }
    
    
        @Around("idempotent()")
        public Object around(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
            HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
            HttpServletResponse response = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getResponse();
            // 获取注解
            MethodSignature signature = (MethodSignature) proceedingJoinPoint.getSignature();
            Idempotent annotation = signature.getMethod().getAnnotation(Idempotent.class);
            log.info("开始进行幂等性校验,url={}, method={}, annotation={}", request.getRequestURI(), signature.getMethod().getName(), annotation);
            String key = "";
            try {
                // 获取lockKey
                key = getKey(annotation, signature, proceedingJoinPoint);
                key = String.format(RedisKey.IDEMPOTENT_CONTROL.getKey(), key);
                log.info("最终解析出来的key={}", key);
                boolean b = redissonUtils.setNx(key, "1", annotation.duration());
                if (!b) {
                    log.error("幂等性控制失败，url={}，key={}", request.getRequestURI(), key);
                    throw new BusinessException(ExceptionEnum.SYSTEM_BUSY);
                }
            } catch (Throwable e) {
                log.error("幂等性控制具体逻辑执行出错", e);
                if (annotation.removeKeyWhenError()) {
                    redissonUtils.del(key);
                }
                if (e instanceof BusinessException) {
    
                    throw e;
                } else {
                    throw new BusinessException(ExceptionEnum.SYSTEM_ERROR);
                }
            } finally {
                if (annotation.removeKeyWhenFinished()) {
                    redissonUtils.del(key);
                }
                log.info("赛等件控验涌过 url={}, method={}, annotation={}", request.getRequestURI(), signature.getMethod().getName(), annotation);
                return proceedingJoinPoint.proceed();
    
            }
        }
    
    
        private String getKey(Idempotent annotation, MethodSignature signature, ProceedingJoinPoint pjp) {
            if (StringUtils.isBlank(annotation.key()) || StringUtils.isBlank(annotation.prefix())) {
                log.error("幂等性控制中，prefix={}或key={}不能为空", annotation.prefix(), annotation.key());
                throw new BusinessException(ExceptionEnum.SYSTEM_ERROR);
            }
            if (annotation.key().contains("#")) {
                boolean checkRes = checkSpel(annotation.key());
                if (!checkRes) {
                    throw new BusinessException(ExceptionEnum.SYSTEM_ERROR);
                }
                String realKey = getKeyVal(annotation.key(), signature, pjp);
                if (StringUtils.isBlank(realKey)) {
                    log.error("获取幂等性的key错误");
                    throw new BusinessException(ExceptionEnum.SYSTEM_ERROR);
                }
                return annotation.prefix() + "_" + realKey;
            } else {
                log.error("幂等性配置错误，key={}", annotation.key());
                throw new BusinessException(ExceptionEnum.SYSTEM_ERROR);
    
            }
        }
    
        private String getKeyVal(String key, MethodSignature signature, ProceedingJoinPoint pjp) {
            Object[] params = pjp.getArgs();
    
            DefaultParameterNameDiscoverer nameDiscoverer = new DefaultParameterNameDiscoverer();
            String[] parameterNames = nameDiscoverer.getParameterNames(signature.getMethod());
            if (parameterNames == null || parameterNames.length == 0) {
                log.error("请求方法参数为空，无法解析参数");
                throw new BusinessException(ExceptionEnum.SYSTEM_ERROR);
            }
            Expression expression = spelExpressionParser.parseExpression(key);
            EvaluationContext context = new StandardEvaluationContext();
            for (int i = 0; i < params.length; i++) {
                context.setVariable(parameterNames[i], params[i]);
            }
            Object value = expression.getValue(context);
            if (value == null) {
                log.error("幂等性校验解析为nul1");
                throw new BusinessException(ExceptionEnum.SYSTEM_ERROR);
            }
            if (value instanceof List<?>) {
                StringBuilder str = new StringBuilder();
                for (Object item : (List<?>) value) {
                    str.append(item).append("");
                }
                return str.toString();
            } else if (checkTypePrimitiveAndPackage(value.getClass())) {
                return String.valueOf(value);
            }
            return null;
        }
    
    
        private boolean checkSpel(String key) {
            try {
                spelExpressionParser.parseExpression(key, new TemplateParserContext());
                return true;
            } catch (Throwable e) {
                log.error("无效的spel表达式", e);
                return false;
            }
        }
    
        private boolean checkTypePrimitiveAndPackage(Class<?> type) {
            if (type.isPrimitive()) {
                return true;
            } else if (type == Integer.class || type == Double.class || type == Float.class || type == Long.class || type == Short.class || type == Byte.class || type == Character.class || type == Boolean.class) {
                return true;
            } else if (type == String.class) {
                return true;
            }
            return false;
        }
    
        private boolean checkTypePrimitiveAndPackageByName(String typeName) {
            if (typeName.equals("Integer") || typeName.equals("Double") || typeName.equals("Float") || typeName.equals("lona") || typeName.equals("short") || typeName.equals("Byte") || typeName.equals("character") || typeName.equals("Boolean")) {
                return true;
            } else if (typeName.equals("String")) {
                return true;
            } else {
                return false;
            }
    
        }
    }
    ```

# 防重放、幂等性和防重复点击的区别

防重放主要是指在网络通信中，防止攻击者截获并重新发送有效的消息或请求，以达到欺骗系统的目的。**是从系统网络安全范畴来做的防护手段。**

幂等性则是为了保证一个接口能够被重复调用多次而不会有业务逻辑的副作用，是从系统功能角度出发，**关注功和业务的正确性和可靠性。**

防重复点击则是指在用户界面中防止用户在短时间内多次点击同一个按钮或链接，从而导致重复的请求或操作。**主要关注用户体验，防止用户因误操作导致的重复请求。**

从上面来看三个概念的关注点和范围都各不相同，三者也并不矛盾，三者在系统设计中可以相辅相成，确保系统的安全性、可靠性和用户体验。

# 参考文章

- [简单聊一聊幂等和防重](https://juejin.cn/post/7302805039450292233?searchId=2024100816365224874698A0E01F636BF2)
- [一口气说出四种幂等性解决方案，面试官露出了姨母笑~](https://juejin.cn/post/6906290538761158670?searchId=2024100816365224874698A0E01F636BF2)
- [如何保证 RabbitMQ 的消息可靠性](https://juejin.cn/post/7228864364744507450?searchId=2024100816365224874698A0E01F636BF2)
- [RabbitMQ 可靠性、重复消费、顺序性、消息积压解决方案](https://juejin.cn/post/6977981645475282958?searchId=2024100816365224874698A0E01F636BF2)
- [分布式系统中接口的幂等性](https://testerhome.com/articles/29399)
- [基于timestamp和nonce的防止重放攻击方案](https://blog.csdn.net/koastal/article/details/53456696)
- [开放API网关实践(二) —— 重放攻击及防御](https://juejin.cn/post/6844903910516195341?searchId=2024101811440668EB97F9736DA8E34C8C)
- [API接口防止参数篡改和重放攻击](https://juejin.cn/post/6890798533473992717?searchId=202410251153373BBEE9832B42586DA034)
- [API接口签名(防重放攻击)](https://juejin.cn/post/6983864029550739463)