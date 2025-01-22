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

# 参考文章

* [MySQL 是怎么加锁的？](https://xiaolincoding.com/mysql/lock/how_to_lock.html)