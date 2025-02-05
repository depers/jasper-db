| title         | tags          | background                                                   | auther | isSlow |
| ------------- | ------------- | ------------------------------------------------------------ | ------ | ------ |
| MySQL锁探究二 | MySQL/锁/事务 | 接着上篇关于MySQL锁探究一，这里我们主要讨论MySQL是如何加锁的 | depers | true   |

创建一张表：

```sql
CREATE TABLE `user` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `name` varchar(30) NOT NULL,
  `age` int NOT NULL,
  PRIMARY KEY (`id`),
  KEY `index_age` (`age`) USING BTREE
) ENGINE=InnoDB comment '用户表';
```

插入几条数据：

```sql
insert into `user` (id, name, age) values
(1, '路飞', 19),
(5, '索隆', 21),
(10, '山治', 22),
(15, '乌索普', 20),
(20, '香克斯', 39);
```

# 一、哪些SQL语句可以添加行级锁

InnoDB 引擎是支持行级锁的，而 MyISAM 引擎并不支持行级锁，所以后面的内容都是基于 InnoDB 引擎 的。

如果要在查询时对记录加行级锁，可以使用下面这两个方式，这两种查询会加锁的语句称为**锁定读**。

```sql
//对读取的记录加共享锁(S型锁)
select ... lock in share mode;

//对读取的记录加独占锁(X型锁)
select ... for update;
```

上面这两条语句必须在一个事务中，**因为当事务提交了，锁就会被释放**，所以在使用这两条语句的时候，要加上`begin`或者`start transaction`开启事务的语句，提交事务使用`commit`，回滚事务使用`rollback`。

**除了上面这两条锁定读语句会加行级锁之外，update 和 delete 操作都会加行级锁，且锁的类型都是独占锁(X型锁)**。

```sql
//对操作的记录加独占锁(X型锁)
update table .... where id = 1;

//对操作的记录加独占锁(X型锁)
delete from table where id = 1;
```

共享锁（S锁）满足读读共享，读写互斥。独占锁（X锁）满足写写互斥、读写互斥。

![](../../assert/x锁和s锁.webp)

# 二、行级锁的基本特性

* **锁的对象是索引，加锁的基本单位是 next-key lock**，它是由记录锁和间隙锁组合而成的，**next-key lock 是前开后闭区间，而间隙锁是前开后开区间**。
* **在能使用记录锁或者间隙锁就能避免幻读现象的场景下， next-key lock 就会退化成记录锁或间隙锁**。



# 参考文章

* [MySQL 是怎么加锁的？](https://xiaolincoding.com/mysql/lock/how_to_lock.html)