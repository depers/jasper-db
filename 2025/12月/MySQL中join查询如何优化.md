| title                   | tags         | background                                                   | auther | isSlow |
| ----------------------- | ------------ | ------------------------------------------------------------ | ------ | ------ |
| MySQL中join查询如何优化 | MySQL/数据库 | 最近在项目开发中遇到了一个和复杂的sql，这个sql中使用了大量的左外连接查询，所以就想着把这句sql做下优化，通过这篇文章和大家一起探讨下join查询的优化如何来做。 | depers | true   |

# 创建实验表

为了方便演示，这里首先创建两张表：

```sql
-- 创建订单表
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT COMMENT '订单ID，主键',
    order_no VARCHAR(32) NOT NULL COMMENT '订单编号，唯一',
    user_id INT NOT NULL COMMENT '用户ID',
    total_amount DECIMAL(10, 2) NOT NULL DEFAULT 0.00 COMMENT '订单总金额',
    status TINYINT NOT NULL DEFAULT 1 COMMENT '订单状态：1-待支付 2-已支付 3-已发货 4-已完成 5-已取消',
    payment_method TINYINT COMMENT '支付方式：1-支付宝 2-微信 3-银行卡',
    payment_time DATETIME COMMENT '支付时间',
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单表';

-- 创建订单详情表
CREATE TABLE order_items (
    id INT PRIMARY KEY AUTO_INCREMENT COMMENT '详情ID，主键',
    order_id INT NOT NULL COMMENT '订单ID，关联orders.id',
    product_id INT NOT NULL COMMENT '商品ID',
    product_name VARCHAR(100) NOT NULL COMMENT '商品名称',
    unit_price DECIMAL(10, 2) NOT NULL COMMENT '商品单价',
    quantity INT NOT NULL DEFAULT 1 COMMENT '购买数量',
    subtotal DECIMAL(10, 2) NOT NULL COMMENT '小计金额 = unit_price * quantity',
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单详情表';

CREATE TABLE order_items (
  id BIGINT PRIMARY KEY,
  order_id BIGINT,
  status TINYINT,
  create_time DATETIME,
  amount DECIMAL(10,2)
);

```

# join查询的优化要点

## 1. JOIN 字段必须有索引

查询sql：

```sql
SELECT *
FROM orders o
JOIN order_items i ON o.id = i.order_id;
```

索引设计：

```Java
orders(id)           -- 主键
order_items(order_id) -- 外键索引
```

## 2. 小表驱动大表（连接顺序）

原则：**行数少、过滤条件多的表 → 放在前面**，尽量让结果集最小的表作为**驱动表（Driving Table）**。

**对于 `INNER JOIN`：** MySQL 通常会自动选择小表驱动大表，但如果使用了 `STRAIGHT_JOIN` 关键字，MySQL 会强制按照语句中表的顺序进行 JOIN。

**对于 `LEFT JOIN`：** 左表是驱动表，右表是被驱动表。优化器不会改变这个顺序。确保左表的数据量相对较小，或者在左表的 WHERE 条件能显著过滤数据。

MySQL 通常会自动优化，但以下情况会出问题：

- 统计信息不准
- 使用 `STRAIGHT_JOIN`
- 复杂子查询

可以强制指定顺序：

```sql
SELECT *
FROM small_table
STRAIGHT_JOIN big_table ON ...
```

## 3. 用 ON 过滤，而不是 JOIN 后过滤

查询sql：

```sql
SELECT *
FROM orders o
JOIN order_items oi ON o.id = oi.order_id AND oi.quantity = 1;
```

创建索引：

```Java
orders(id)
order_items(order_id, quantity)
```

值得注意的是这是一个内连接查询，在INNER JOIN中，on条件和where条件其实是等价的，为什么INNER JOIN中 `ON` 里可以“过滤 order_items 表”，执行逻辑如下：

```Java
1. 从 orders 取一行
2. 到 order_items 表中找：
   - oi.order_id = o.id
   - 且 oi.quantity = 1
3. 找不到 → 整行丢弃
```

如果将上面内连接改为左外连接呢？

```Java
-- 看起来是 LEFT JOIN
SELECT *
FROM orders o
LEFT JOIN users u ON o.user_id = u.id
WHERE u.status = 1;
```

实际的效果：

```Java
u.status = 1 为 NULL 的行被 WHERE 过滤掉
→ LEFT JOIN 退化成 INNER JOIN
```

上面这句sql的查询结果和使用INNER JOIN查询到的结果是一致的，如果你真实的目的是LEFT JOIN，但是执行结果却是INNER JOIN的效果，那代码的逻辑就错了。

**ON 是 JOIN 时的匹配条件，WHERE 的真正职责是：过滤“已经 JOIN 完成后的结果集”。**

所以，如果你是内连接查询where条件写到on或是where后都可以，如果是外连接查询，就要注意了。

## 4. 控制返回列，避免 `SELECT *`

查询sql：

```sql
SELECT o.id, o.create_time, u.name
FROM orders o
JOIN users u ON ...
```

好处：

- 减少 IO
- 覆盖索引更容易生效

## 5. 合理使用联合索引（顺序非常关键）



# join查询的错误点



# 参考文章

* [MySQL 查询优化：JOIN 操作背后的性能代价与更优选择](https://juejin.cn/post/7497886347308466211?searchId=202512112200456F1FAE7D5F5894D8C746#heading-17)