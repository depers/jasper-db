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
```

# join查询的顺序

先来看一段sql：

```Java
select * from order o 
left join order_items oi on o.id = oi.order_id
where o.status = 1 and oi.quantity = 1;
```

让我们站在MySQL优化器的角度想一下该如何执行这条sql：

1. 逻辑转换：从 LEFT JOIN 到 INNER JOIN

    因为where条件中使用了`oi.quantity = 1`，所以任何 oi 为 NULL 的行都会被`oi.quantity=1`过滤掉，这条sql会变成内连接查询：

    ```sql
    SELECT *
    FROM order o
    JOIN order_items oi ON o.id = oi.order_id
    WHERE o.status = 1
    AND oi.quantity = 1;
    ```

2. 优化阶段：选择驱动表

    因为变成了内循环，所以MySQL优化器需要重新估算扫描行数和索引效率，决定谁是驱动表。这里我们不确定两张表的各自的总行数，没有新建索引的情况下。

    - **场景 A（o 为驱动表）：** 如果 `status` 上有高效索引，或者 `o` 表满足条件的行数很少。
    - **场景 B（oi 为驱动表）：** 如果 `quantity` 上有高效索引，优化器可能选择先扫描 `oi`。

    通常情况下，MySQL 遵循**“小结果集驱动大结果集”**的原则。

3. 执行阶段：嵌套循环关联 (Nested Loop Join)

    假设优化器选择了 `order o` 作为驱动表，执行流程如下：

    1. **扫描驱动表（o）：**
        - 通过索引（如 `status` 的索引）或者全表扫描，找出所有 `status = 1` 的记录。
    2. **逐行匹配（Loop）：** 对于读取到的每一条 `o` 表记录，获取其 `id` 值。
    3. **查询被驱动表（oi）：**
        - 使用 `o.id` 去 `order_items` 表中查找。
        - **理想状态：** `oi.order_id` 上有索引。MySQL 会直接通过索引定位到关联的行，而不是扫描整个 `oi` 表。
    4. **应用右表过滤：**
        - 在关联到的 `oi` 行中，检查 `quantity = 1` 是否成立。
    5. **结果合并：**
        - 如果匹配成功且条件成立，则将 `o` 和 `oi` 的列合并存入结果集。

# join查询优化算法

## 1. Simple Nested-Loop Join(SNJJ，简单嵌套循环连接)

## 2. Index Nested-Loop Join (INLJ, 索引嵌套循环连接)

## 3. Block Nested-Loop Join (BNL, 块嵌套循环连接)

## 4. Batched Key Access (BKA, 批量键访问)

## 5. Hash Join (哈希连接)

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



# join查询的错误点



# 参考文章

* [MySQL 查询优化：JOIN 操作背后的性能代价与更优选择](https://juejin.cn/post/7497886347308466211?searchId=202512112200456F1FAE7D5F5894D8C746#heading-17)