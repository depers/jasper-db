| title       | tags          | background                                                   | auther | isSlow |
| ----------- | ------------- | ------------------------------------------------------------ | ------ | ------ |
| MySQL锁探究 | MySQL/锁/事务 | 最近在研究基于数据库实现分布式序列号生成方案时发现，自己对MySQL这块锁的理解还停留在理论和简单使用的层面，并没与在实践中充分的掌握这块知识点，所以这里专门开一篇文章来讨论这块内容。 | depers | true   |

# 表锁

* **共享锁**
    * 加锁命令：`LOCK TABLES t READ`
* **排他锁**
    * 加锁命令：`LOCK TABLES t WRITE`
* **意向锁**
    * **意向共享锁**
    * **意向排他锁**

# 行锁

## 记录锁

* 也叫**Record Lock**。
* 仅仅是将一条记录锁上，所以记录锁是**查询条件是添加了唯一非空索引或是主键索引的列**。

### 具体实践

1. 创建表`user`

    ```sql
    CREATE TABLE `user` (
      `id` int NOT NULL AUTO_INCREMENT COMMENT '主键',
      `name` varchar(20) NOT NULL DEFAULT '' COMMENT '姓名',
      PRIMARY KEY (`id`),
      UNIQUE KEY `uni_name` (`name`)
    ) ENGINE=InnoDB COMMENT='用户表';
    ```

    `user`表有两个字段，一个是id，添加了主键索引。一个是name，添加了非空唯一索引。值得注意的是如果一个列添加了唯一索引，但是这个列可以为null，这种情况下这个列首先可以插入多个null值，其次针对这个列的是不会添加记录锁的。

## 间隙锁

* 排他锁的一种，也叫**Gap Lock**。
* **不允许其他事务往这条记录前面的间隙插入新记录，锁定一个范围但是不包含记录本身**。
* 仅在可重复读（Repeatable Read）隔离级别下有效。
* 作用是解决幻读问题。
* 基于**非唯一索引**。
* 案例说明：
    * `SELECT * FROM user WHERE age BETWEEN 100 AND 200 FOR UPDATE;`：索引字段`age`在`(100，200)`范围内添加了间隙锁。
    * `SELECT * FROM user where age = 100;`：索引字段`age`在` (-∞,1)`范围内添加了间隙锁。

## 临键锁

* 也叫**Next-Key Lock**。
* 结合了记录锁（Record Lock）和间隙锁（Gap Lock）的一种锁机制。**锁定的也是一个范围，包含记录本身**。
* 作用是解决幻读问题。
* 临键锁只与**非唯一索引列**有关，**在唯一索引列（包括主键列）上不存在**临键锁。
* 案例说明：`SELECT * FROM users WHERE age = 25 FOR UPDATE;`

## 插入意向锁

# 参考文章

* 刘遵庆、凡新雷、邹勇 《MySQL DBA精英实战课》
* [记录锁、间隙锁  Next-Key Lock](https://mp.weixin.qq.com/s/Zh7GSzXJg_zt2ug3X5TwEQ)
* [什么是MySQL插入意向锁？](https://juejin.cn/post/7178321966024097829)
* [详解 MySql InnoDB 中的三种行锁（记录锁、间隙锁与临键锁）](https://juejin.cn/post/6844903666420285454#heading-7)