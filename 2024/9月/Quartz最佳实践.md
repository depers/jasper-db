| title          | tags                  | background                                                   | auther | isSlow |
| -------------- | --------------------- | ------------------------------------------------------------ | ------ | ------ |
| Quartz最佳实践 | 分布式任务调度/Quartz | 当前市面上常见的分布式任务调度框架中，Quartz、ElasticJob和XXL-JOB是广受欢迎的开源解决方案，阿里的SchedulerX是闭源产品。在普通项目中使用Quartz就可以满足需求，本篇文章就Quartz框架来讨论项目中如何更好的去使用它。 | depers | true   |

当前市面上常见的分布式任务调度框架中，Quartz、ElasticJob和XXL-JOB是广受欢迎的开源解决方案，阿里的SchedulerX是闭源产品。在普通项目中使用Quartz就可以满足需求，本篇文章就Quartz框架来讨论项目中如何更好的去使用它。

# 三个核心类

- `JobDetail`（任务）

    - 作用：具体要执行的业务逻辑。

- `Trigger`（触发器）

    -  用来定义Job（任务）触发条件、触发时间，触发间隔，终止时间等。

    - `SimpleTrigger`

        在具体的时间点执行一次，或者在具体的时间点执行，并且以指定的间隔重复执行若干次。

    - `CronTrigger`

        可以定义Cron表达式来定义任务执行的规则。

- `Scheduler`（调度器）

    -  Scheduler启动Trigger去执行Job。

- 其他类 

    - `JobDataMap`

        负责存储批量执行的附加信息，在`JobDataMap`里面如果存储对象，这个对象一定要实现`Serializable`，否则会报错`JobPersistenceException`。

# 关键配置

这里我先给出一份配置，然后对照配合来进行分析：

```YAML
spring:
    datasource:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://localhost:3306/quartz
        username: root
        password: root
    quartz:
        jdbc:
            initialize-schema: never # 这个配置真的很坑人，记得第一次初始化是always，之后要改成never
        job-store-type: jdbc
        properties:
            org:
                quartz:
                    # 作业存储配置
                    jobStore:
                        driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
                        tablePrefix: QRTZ_
                        isClustered: true
                        clusterCheckinInterval: 10000
                        misfireThreshold: 10000
                    # 调度器配置
                    scheduler:
                        instanceName: Quartz
                        instanceId: AUTO
                    # 线程池配置
                    threadPool:
                        class: org.quartz.simpl.SimpleThreadPool
                        threadCount: 3
```

- `initialize-schema`：在项目启动的时候是否需要初始化表结构，因为Quartz框架的数据是保存在数据库中的，所以有一些表需要提前建出来。值得注意的是第一次运行项目我们需要将该配置设置为`always`，之后的配置设置成`never`就行，否则每次运行项目都会重新建表。

- `job-store-type`：用来指定任务存储类型的配置项。这个配置项决定了Quartz的调度器（Scheduler）将如何存储任务（Job）和触发器（Trigger）的信息。`job-store-type`有两个主要的选项：`memory`和`jdbc`。

    - `memory`：这是默认的存储类型，它使用`RAMJobStore`来存储任务和触发器的信息。这种方式的优点是简单且速度快，因为它将所有数据保持在内存中。但是，当应用程序停止或崩溃时，所有的调度信息将会丢失，这意味着`RAMJobStore`不支持持久化存储。这种方式适用于单节点环境，且不需要持久化存储的场景。
    - `jdbc`：这个选项使用`JDBCJobStore`来存储任务和触发器的信息，通过数据库来实现。这种方式的优点是支持集群和持久化存储，即使应用程序停止或崩溃，任务信息也不会丢失。此外，`JDBCJobStore`可以更好地支持事务和并发。这种方式适用于多节点环境，或者需要持久化存储任务信息的场景。

- `driverDelegateClass`：用于指定`JobStore`要使用的特定数据库的代理类。不同的数据库可能需要不同的JDBC工作，而这些代理类就是用来处理这些特定工作的。

    -  `driverDelegateClass`的值应该是一个实现了`DriverDelegate`接口的类的全限定名。Quartz提供了几个标准的代理类，例如：
    - `org.quartz.impl.jdbcjobstore.StdJDBCDelegate`：这是一个标准的JDBC代理，适用于大多数数据库。
    - `org.quartz.impl.jdbcjobstore.WebLogicDelegate`：用于与WebLogic服务器一起使用。
    - `org.quartz.impl.jdbcjobstore.oracle.OracleDelegate`：专门用于Oracle数据库。

- `tablePrefix`：JDBCJobStore 的 “table prefix” 属性是一个字符串，等于为数据库中创建的 Quartz 表提供的前缀。如果 Quartz 的表使用不同的表前缀，则可以在同一个数据库中拥有多组 Quartz 表。

- `isClustered`：用于指定调度器是否以集群模式运行。集群模式下的Quartz调度器可以跨越多个实例（节点）共享作业和触发器，这些实例通常连接到同一个数据库。这种模式非常适合于需要高可用性和负载均衡的场景。

    值得注意的是：

    -  当`isClustered`设置为`true`时，Quartz会使用数据库中的锁机制来确保同一时间只有一个节点执行特定的作业。
    -  集群模式需要所有节点连接到同一个数据库，并且共享相同的Quartz表结构。
    -  在集群环境中，通常需要配置`instanceId`以确保每个节点都有一个唯一标识。
    -  集群模式下，作业的持久化（`storeDurably`）和故障恢复（`requestRecovery`）设置非常重要，以确保作业在节点故障后能够被其他节点接管。

- `clusterCheckinInterval`：在Quartz中，`clusterCheckinInterval` 是一个重要的配置属性，它定义了调度器实例（Scheduler instance）在集群模式下向数据库中记录其状态的频率。这个频率是以毫秒为单位的，调度器会在这个时间间隔内更新它在 `qrtz_scheduler_state` 表中的最后检入时间（`last_checkin_time`）。这个属性对于集群模式下的故障检测和故障恢复至关重要。

    -  当集群中的一个节点发生故障时，其他节点可以通过检查 `qrtz_scheduler_state` 表中记录的最后检入时间来检测到这一点。如果一个节点在预定的 `clusterCheckinInterval` 时间内没有更新其检入时间，那么其他节点就会认为它已经失败，并可能接管其作业。
    -  默认情况下，`clusterCheckinInterval` 的值是15000毫秒，即15秒。但是，这个值可以根据实际需要进行调整。例如，如果你需要更快地检测到故障，你可以减小这个值；如果你希望减少数据库的负载，你可以增加这个值。
    -  需要注意的是，`clusterCheckinInterval` 的设置会影响到集群的故障检测速度和数据库的负载。设置得太短可能会导致数据库负载过高，而设置得太长可能会延迟故障检测。因此，应该根据实际的集群规模和业务需求来选择合适的值。
    -  在实际部署时，还应该考虑到时钟同步的问题，因为节点之间的时间偏差可能会影响故障检测的准确性。通常建议使用网络时间协议（NTP）来同步集群中所有节点的时钟。

- `misfireThreshold`：用来设定触发器超时的临界值。当触发器错过了预定的触发时间，如果超出了这个设定的时间阈值，调度器就会认为触发器发生了"misfire"（超时）。这个属性对于调度器如何处理延迟执行的任务非常重要。

    -  `misfireThreshold` 的默认值通常是60000毫秒（即60秒）。如果触发器的执行延迟时间超过了这个阈值，调度器就会根据触发器的类型（如SimpleTrigger或CronTrigger）和它们的**misfire指令**来决定如何补偿。关于**misfire指令**在后续章节会重点讨论。

- `instanceName`：用于为调度器实例命名。这个名称对于区分同一个程序中运行的不同调度器实例或集群环境中逻辑上相同的调度器实例非常有用。在集群模式下，所有调度器实例都应该使用相同的`instanceName`，这样它们才能正确地协同工作。

- `instanceId`：用于在集群环境中标识不同的调度器实例。每个调度器实例都应该有一个唯一的 `instanceId`，以确保在集群中的多个节点可以协同工作，而不会相互干扰。

    -  `instanceId` 可以设置为以下值之一：

    - `NON_CLUSTERED`：表示调度器不参与集群。

    - `AUTO`：Quartz将自动生成一个唯一的 `instanceId`。

    - `SYS_PROP`：从系统属性 `org.quartz.scheduler.instanceId` 中获取值。

    -  此外，可以通过实现 `InstanceIdGenerator` 接口来提供自定义的 `instanceId` 生成策略。Quartz提供了几种内置的 `InstanceIdGenerator` 实现，例如：

    - `SimpleInstanceIdGenerator`：基于主机名和时间戳生成 `instanceId`。

    - `SystemPropertyInstanceIdGenerator`：从系统属性 `org.quartz.scheduler.instanceId` 获取 `instanceId`。

    - `HostnameInstanceIdGenerator`：使用本地主机名生成 `instanceId`。

    -  在配置文件中，可以通过以下方式设置 `instanceId` 和相关的生成器类：

       ```YAML
        org.quartz.scheduler.instanceId = AUTO
        org.quartz.scheduler.instanceIdGenerator.class = org.quartz.simpl.SimpleInstanceIdGenerator
       ```

- `threadPool.class`：线程池（ThreadPool）提供一组线程供Quartz在执行作业（Job）时使用。线程池中的线程数量决定了可以同时执行的作业数量。Quartz默认提供的线程池实现是 `org.quartz.simpl.SimpleThreadPool`，它对于大多数用户来说已经足够使用。

    -  SimpleThreadPool 是一个简单但非常可靠的线程池实现，它维护了一个固定数量的线程，这些线程的生命周期与调度器（Scheduler）的生命周期相同。它提供了一个固定大小的线程池，适用于大多数情况，并且经过了很好的测试。Quartz还支持创建自己的线程池实现，这里不多做叙述。
    -  此外，还可以设置其他与线程池相关的属性，如 threadCount（线程数量）和 threadPriority（线程优先级）。值得注意的是在Java中，线程的默认优先级是`Thread.NORM_PRIORITY`，其值是5。Java线程优先级范围是从1（`Thread.MIN_PRIORITY`）到10（`Thread.MAX_PRIORITY`），其中5代表正常优先级。

# 数据库表结构和核心字段

按照上面的配置，我们需要提前为Quartz项目新建相关表结构，Quartz涉及的表结构一共有11张，我们一张一张来看。

## qrtz_job_details 

负责存储每一个已配置的 jobDetail 的详细信息，如下表所示。

| 表字段            | 含义                                                         |
| :---------------- | :----------------------------------------------------------- |
| is_durable        | 是否持久化，把该属性设置为1，quartz会把job持久化到数据库中。可以在创建`JobDetail`的时设置`storeDurably()`为`true`或`false`。 |
| is_nonconcurrent  | 是否非并发执行，0-并发执行，1-非并发执行。该字段用于标识一个作业是否允许并发执行。如果一个作业的 is_nonconcurrent 字段设置为 0 或 false，则表示该作业允许并发执行；如果设置为 1 或 true，则表示不允许并发执行，即当该作业正在执行时，即使到了下一个触发时间，作业也不会再次启动，直到当前执行完成。可以在实现Job接口的类上添加`@DisallowConcurrentExecution`注解表示该任务是非并发执行的。 |
| is_update_data    | 是否更新数据。用于指示在执行作业时是否更新作业的数据。这一字段的设置影响到作业在执行过程中是否可以修改其数据。如果 `is_update_data` 设置为 `1` 或 `true`，表示在作业执行时，Quartz会更新作业的数据。这通常用于那些需要在执行过程中动态修改数据的作业。如果设置为 `0` 或 `false`，则表示作业在执行时不会更新其数据，保持原有状态。可以在实现Job接口的类上添加`@PersistJobDataAfterExecution`注解表示该任务是是可以修改任务的附加信息的。 |
| requests_recovery | 在Quartz调度器中，`requests_recovery` 是 `qrtz_job_details` 表的一个字段，用于指示当作业执行过程中调度器实例失败时，作业是否应该被其他实例重新执行。这个字段对于确保作业在出现故障时能够被恢复执行非常重要。如果 `requests_recovery` 设置为 `1` 或 `true`，则表示如果作业在执行过程中调度器实例失败，作业应该被其他实例重新执行。如果设置为 `0` 或 `false`，则表示作业在调度器实例失败时不会被重新执行，作业将丢失，直到下次预定的触发时间。这个字段通常用于关键任务，确保即使在调度器实例出现故障的情况下（**指的是例如调度节点宕机这种情况，并不是任务执行过程中出现异常**），作业也能够被恢复执行。在配置作业时，可以通过编程方式设置这个属性，例如在创建 `JobDetail` 对象时，可以设置 `requestsRecovery()` 方法为 `true` 或 `false`。 |

## qrtz_triggers

用来保存触发器的基本信息。

| 表字段        | 含义                                                         |
| :------------ | :----------------------------------------------------------- |
| priority      | 在Quartz中，`qrtz_triggers` 表的 `priority` 字段用于设置触发器的优先级。当多个触发器同时触发时，线程池中的线程不足以处理所有触发器，此时会根据触发器的优先级来决定哪个触发器先执行。优先级数值越大，优先级越高。如果没有显式设置触发器的优先级，那么它将使用默认优先级，通常这个默认值是5。在配置触发器时，可以通过编程方式设置优先级。例如，在Java代码中，可以使用 `TriggerBuilder.withPriority(10)` 来构建触发器并设置其优先级。 |
| trigger_state | `qrtz_triggers`表的`trigger_state`字段用于表示触发器的状态。常见的状态值有：WAITING：触发器正在等待触发条件满足。ACQUIRED：触发器已被调度器获取，准备触发。EXECUTING：触发器正在执行相关的任务。COMPLETE：触发器执行完成。BLOCKED：触发器被阻塞，可能由于资源竞争或其他原因。PAUSED：触发器被暂停，不会触发任务，直到恢复。ERROR：触发器处于错误状态。 |
| trigger_type  | 在Quartz中，`qrtz_triggers` 表的 `trigger_type` 字段用于指示触发器的类型。这个字段通常用于区分不同类型的触发器，例如 `CRON`、`SIMPLE` 或者 `BLOB` 等。每种触发器类型都有其特定的属性和行为。`CRON`：表示触发器使用 cron 表达式来定义触发规则。`SIMPLE`：表示触发器使用简单的重复间隔来定义触发规则。`BLOB`：表示触发器使用二进制大型对象（blob）来存储其触发规则，这通常用于自定义的触发器类型。 |
| misfire_instr | 在Quartz中，`qrtz_triggers `表的 `misfire_instr`字段是一个关键的属性，它定义了当触发器错过了预定的触发时间（即发生了“misfire”）时，Quartz应该如何处理这种情况。这个字段的设置在后续章节再来详细说明。 |

下面是一段声明触发器的代码，方便大家参考：
```Java
Trigger trigger = TriggerBuilder.newTrigger()
    .withIdentity("myTrigger","group1")
    .startNow()
    .withSchedule(SimpleScheduleBuilder
                  .simpleSchedule()
                  .withIntervalInSeconds(40)
                  .repeatForever())
    .withPriority(10) // 设置触发器的优先级
    .build();
```

## qrtz_cron_triggers 

负责存储触发器的cron表达式表。

## qrtz_simple_triggers 

负责存储简单的 Trigger，包括重复次数，间隔，以及已触发的次数。

## qrtz_blob_triggers

负责存储以Blob类型存储的触发器信息的表。在实际应用中，如果开发者定义了自己的触发器类并需要持久化一些特定的状态，那么可能会用到qrtz_blob_triggers表来存储这些状态信息。这样，当调度器重新启动时，可以从数据库中恢复这些自定义触发器的状态。

## qrtz_fired_triggers 

用于存储已经触发的触发器的状态信息，以及与触发的作业相关的执行信息。当一个作业被触发并且开始执行时，Quartz会在这个表中创建一条记录。

## qrtz_paused_trigger_grps

`qrtz_paused_trigger_grps` 表在Quartz数据库中用于存储已暂停的触发器组的信息。当一个触发器组被暂停时，Quartz会在这张表中插入一条记录，以表示该组中的所有触发器都处于暂停状态。

## qrtz_scheduler_state 

负责存储集群中note实例信息，quartz会定时读取该表的信息判断集群中每个实例的当前状态。

## qrtz_calendars 

`qrtz_calendars` 表在Quartz数据库中用于存储日历信息，这些日历可以定义特定的时间范围，以便控制作业触发器在这些时间范围内是否应该触发。Quartz提供了多种类型的日历，例如：

- `AnnualCalendar`：排除一年中特定的日期。
- `BaseCalendar`：作为更复杂日历的基础类。
- `CronCalendar`：根据 `CronExpression` 排除时间。
- `DailyCalendar`：每天排除特定的时间范围。
- `HolidayCalendar`：存储一系列假日（全天排除调度）。
- `MonthlyCalendar`：排除一个月中的特定日期。
- `WeeklyCalendar`：排除一周中的特定日期。

在Quartz中，日历可以与触发器关联，以修改触发器的触发行为。例如，可以创建一个 `HolidayCalendar` 来排除公共假日，然后将其与一个触发器关联，这样触发器在假日就不会触发作业。

## qrtz_locks

`qrtz_locks` 表在Quartz数据库中扮演着至关重要的角色，尤其是在集群模式下。这个表用于实现Quartz集群中的同步机制，通过数据库锁来保证多个节点在处理作业时的一致性和防止数据的竞态条件。表结构简单，通常包含一个名为 `LOCK_NAME` 的字段，用于标识不同类型的锁。

# 动态添加批量任务

## 动态新增

## 暂停批量

## 删除批量

# 错过执行策略

# 中断执行策略

# 执行线程不足的情况下，批量任务该如何执行

# 参考项目

- depers/quartz-dynamic-job

# 参考文字

- [阿里SchedulerX和开源产品对比](https://help.aliyun.com/zh/schedulerx/schedulerx-serverless/product-overview/comparison-between-schedulerx-and-open-source-solutions?spm=a2c4g.11186623.0.0.33de2104EG3Bzo)
- [quartz-scheduler/quartz](https://github.com/quartz-scheduler/quartz/tree/main)
- [Quartz中表及其表字段的意义](https://www.cnblogs.com/zyulike/p/13671130.html)