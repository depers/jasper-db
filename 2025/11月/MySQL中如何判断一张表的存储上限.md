| title                           | tags  | background                                                   | auther | isSlow |
| ------------------------------- | ----- | ------------------------------------------------------------ | ------ | ------ |
| MySQL中如何判断一张表的存储上限 | MySQL | 在日常的需求开发中，数据库表遇到了存储大量数据的情况，必须要保障数据查询的效率。我们如何判断一个数据库表存储数据的上限是多少？什么情况下去做分库分表。通过这篇文件，我们一起来探索一张表的存储上限是如何计算的，他背后的原理是什么 | depers | true   |

假如有这么一张表：

```SQL
drop table if exists `t_email`;
create table `t_email` (
  `id` int auto_increment comment '主键id',
  `attach_file_flag` tinyint not null default '0' comment '是否包含附件，1-是，0-否',
  `subject` varchar(50) not null default '' comment '邮件主题',
  `addresser` varchar(40) not null default '' comment '发件人',
  `receiver` varchar(40) not null default '' comment '收件人',
  `send_time` datetime not null default '1970-01-01 0:0:0' comment '发送时间',
  `content_size` int not null default '0' comment '邮件大小，单位kb',
  `receiver_type` tinyint not null default '1' comment '收件人类型，1-主送，2-抄送，3-密送',
  `multi_receiver_flag` tinyint not null default '0' comment '是否多个收件人，0-否，1-是',
  `unique_identification` varchar(30) not null default '' comment '唯一标识',
  `org_id` varchar(8) not null default '' comment '机构id',
  `department_id` varchar(8) not null default '' comment '部门id',
  `pull_date` date not null default '1970-01-01' comment '数据拉取日期'
) engine=innodb comment '邮件表';
```

分析每个字段的占用空间：

- `id`：int类型，占用4个字节
- `attach_file_flag`：tinyint类型，占用1个字节
- `subject`：varchar类型，字符编码是utf8mb4，存储的是中文，占3个字节，一共是150个字节
- `addresser`：varchar类型，字符编码是utf8mb4，存储的是中文，占3个字节，一共是120个字节
- `receiver`：varchar类型，字符编码是utf8mb4，存储的是中文，占3个字节，一共是120个字节
- `send_time`：datetime类型，占8个字节
- `content_size`：int类型，占4个字节
- `receiver_type`：tinyint类型，占1个字节
- `multi_receiver_flag`：tinyint类型，占1个字节
- `unique_identification`：varchar类型，字符编码是utf8mb4，存储的是英文，占1个字节，一共是30个字节
- `org_id`：varchar类型，字符编码是utf8mb4，存储的是英文，占1个字节，一共是8个字节
- `department_id`：varchar类型，字符编码是utf8mb4，存储的是英文，占1个字节，一共是8个字节
- `pull_date`：占3个字节

综合上面的字段，一行数据一共占用1+150+120+120+8+4+1+1+30+8+8+3=458

加上页的其他信息：

1. 行记录头信息：肯定得有，占用5字节。
2. 事务ID和指针字段：两个都得有，占用13字节。

每个叶子节点可以存放15232/476=32条数据，如果算上页目录，每个槽位占2个字节，每个槽位存放6行数据的位置，算下来也是32条数据。

三层B+树存放的最大数据量就是：

- 主键id是int：32*986049=31553568
- 主键id是bigint：32*619369=19819808

# 参考文章

- [我说MySQL每张表最好不超过2000万数据，面试官让我回去等通知？](https://juejin.cn/post/7165689453124517896#heading-6)