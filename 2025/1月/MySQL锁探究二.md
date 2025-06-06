| title         | tags          | background                                                   | auther | isSlow |
| ------------- | ------------- | ------------------------------------------------------------ | ------ | ------ |
| MySQL锁探究二 | MySQL/锁/事务 | 接着上篇关于MySQL锁探究一，这里我们主要讨论MySQL是如何加锁的 | depers | true   |

# 一、哪些SQL语句可以添加行级锁？

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

# 二、有什么命令可以分析加了什么锁？

可以通过 `select * from performance_schema.data_locks;` 这条语句，查看事务执行 SQL 过程中加了什么锁。

其中`LOCK_TYPE`表示是表级锁还是行级锁：

* `TABLE`：表示表级锁。
* `RECORD`：表示行级锁。

其中LOCk_MODE字段的含义：

| lock_mode 值       | 锁类型                                | 说明                                         |
| ------------------ | ------------------------------------- | -------------------------------------------- |
| `S`                | 共享锁 (Shared Lock)                  | 允许多个事务同时读取同一资源                 |
| `X`                | 排他锁 (Exclusive Lock)               | 只允许一个事务读写资源，其他事务不能加任何锁 |
| `IS`               | 意向共享锁 (Intention Shared Lock)    | 表示事务打算在表的某些行上设置共享锁         |
| `IX`               | 意向排他锁 (Intention Exclusive Lock) | 表示事务打算在表的某些行上设置排他锁         |
| `GAP`              | 间隙锁 (Gap Lock)                     | 锁定索引记录之间的间隙                       |
| `REC_NOT_GAP`      | 记录锁 (Record Lock)                  | 只锁定索引记录本身，不锁定间隙               |
| `INSERT_INTENTION` | 插入意向锁 (Insert Intention Lock)    | 表示准备在间隙中插入新记录的意向             |
| `AUTO_INC`         | 自增锁 (Auto-Increment Lock)          | 用于自增列插入操作的特殊表级锁               |

通过 `LOCK_MODE`可以确认是 next-key 锁，还是间隙锁，还是记录锁：

- `X`，说明是 next-key 锁
- `X, REC_NOT_GAP`，说明是记录锁
- `X, GAP`，说明是间隙锁
- `IS`：意向共享锁（表级）
- `IX`：意向排他锁（表级）

其中LOCK_STATUS字段的含义，通常有以下几种可能的值：

* **GRANTED** - 表示锁已被成功获取，当前持有该锁
* **WAITING** - 表示会话正在等待获取该锁（可能被其他会话持有的锁阻塞）
* **TIMEOUT** - 表示锁请求已超时（在某些情况下可能出现）
* **KILLED** - 表示锁请求已被终止（如会话被kill）

# 三、行级锁的基本特性

* **锁的对象是索引，加锁的基本单位是 next-key lock**，它是由记录锁和间隙锁组合而成的，**next-key lock 是前开后闭区间，而间隙锁是前开后开区间**。
* **在能使用记录锁或者间隙锁就能避免幻读现象的场景下， next-key lock 就会退化成记录锁或间隙锁**。
* 如果是**二级索引的唯一索引**，除了对二级索引的加锁规则之外，还会对查询到的记录的主键索引项加「**记录锁**」。

# 四、MySQL是如何添加行级锁的

![](../../assert/MySQL如何添加行级锁.png)

# 参考文章

* [MySQL 是怎么加锁的？](https://xiaolincoding.com/mysql/lock/how_to_lock.html)
* [第25章 工作面试老大难-锁](https://relph1119.github.io/mysql-learning-notes/#/mysql/25-%E5%B7%A5%E4%BD%9C%E9%9D%A2%E8%AF%95%E8%80%81%E5%A4%A7%E9%9A%BE-%E9%94%81)
* [MySQL锁探究一](http://www.bravedawn.cn/details.html?aid=8499)