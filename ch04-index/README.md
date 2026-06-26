# 第四章：索引体系

> 前面学的是「高速公路怎么修出来的」，现在学的是「车辆到底怎么走、为什么堵车、如何避免堵车」。这部分直接决定你以后能不能设计表、优化 SQL、看懂 EXPLAIN。

---

## 4.1 索引到底是什么？

先拆掉一个错误认知。很多教程说："索引是提高查询效率的数据结构" — 这句话没错，但太抽象。

### 第一性原理

数据库真正的问题是：**如何少读磁盘**，不是**如何少计算**。

索引的本质：**用额外空间换取更少的磁盘 IO**。即：**空间换时间（Space → Trade → Time）**。

### 假设没有索引

表 user，1000 万条数据。查询 `WHERE name='张三'`：

数据库只能：第一页 → 第二页 → 第三页 → ... → 最后一页，全部扫描。

这叫 **Full Table Scan（全表扫描）**，复杂度 O(n)。

### 有索引之后

```sql
CREATE INDEX idx_name ON user(name);
```

数据库创建一棵新的 B+Tree。查询变成：root → middle → leaf，直接定位。复杂度 O(logN)。

---

## 4.2 聚簇索引

这是 InnoDB 最重要的概念。

### 隐含假设

很多人以为：表和索引是分开的。事实上，在 InnoDB 中：**表就是一棵 B+Tree**。

```sql
CREATE TABLE user(
  id BIGINT PRIMARY KEY,
  name VARCHAR(50)
);
```

叶子节点里是**完整数据**。

这棵树既是**数据**，又是**主键索引**。所以叫 **Clustered Index（聚簇索引）**。

### 为什么必须有主键？

因为 B+Tree 必须排序（见[第二章 §2.6：B+Tree 必须排序——这是它的"基因"](../ch02-data-structure/README.md#b+tree-必须排序--这是它的基因)），总得有个排序依据。如果没定义主键，MySQL 会偷偷创建隐藏主键。

**经验原则：每张表都要有主键。**

---

## 4.3 二级索引

`WHERE name='张三'` — 聚簇索引按 id 排序，name 没法快速找。

于是 `CREATE INDEX idx_name ON user(name)` 再生成一棵树。

**叶子节点存什么？** 很多新人以为存完整数据 → **错**。实际存 `(name, id)`，只保存**主键值**。

这叫 **Secondary Index（二级索引）**。

### 为什么不存完整数据？

如果存完整数据：聚簇索引存一份、name 索引存一份、email 索引存一份 → 数据重复三份 → 空间爆炸。

---

## 4.4 回表

这是慢 SQL 的重要来源。

```sql
SELECT * FROM user WHERE name='张三';
```

流程：
1. 走 `idx_name` 找到 `张三 → id=1`
2. 再去**聚簇索引**找到完整数据 `(1, 张三, 18, 北京)`

两次查找，这叫**回表**。

```text
        idx_name
 张三 → 1
          ↓
      PRIMARY
 1 → 完整数据
```

---

## 4.5 覆盖索引

既然回表慢，能不能不回？

```sql
SELECT id FROM user WHERE name='张三';
```

需要的是 `id`，而 `idx_name` 叶子节点已经有 `(name, id)`。直接返回，不用访问主键树。

这叫 **Covering Index（覆盖索引）**，性能极高。

---

## 4.6 联合索引

现实业务：`WHERE age=18 AND city='北京'`。

建立两个索引 `idx_age` 和 `idx_city` — 未必最好，数据库可能查两棵树再求交集。

更好：`(age, city)` → 形成排序：

```text
18 北京
18 上海
19 北京
19 上海
```

称 **Composite Index（联合索引）**。

---

## 4.7 最左匹配原则

面试高频，但不要背，理解即可。

联合索引 `(age, city, gender)`，排序规则：

```text
先 age → 再 city → 再 gender
```

类似 Excel 排序：年龄 → 城市 → 性别。

- ✅ `WHERE age=18`
- ✅ `WHERE age=18 AND city='北京'`
- ✅ `WHERE age=18 AND city='北京' AND gender='男'`
- ❌ `WHERE city='北京'` — 树首先按 age 排序

这就是**最左匹配原则**。

---

## 4.8 索引失效

这是线上事故来源之一。

### 函数运算

```sql
WHERE age+1=18  -- 索引存的是 age，不是 age+1 → 全表扫描
```

### 前缀模糊

```sql
WHERE name LIKE '%三'  -- B+Tree 无法定位开头 → 全表扫描
```

---

## 4.9 索引设计第一原则

索引不是越多越好。

INSERT 时：**写数据 + 更新所有索引**。索引越多，写越慢。

```text
原则：索引服务查询，但增加写成本。这是一个权衡。
```

---

## 4.10 WHERE/GROUP BY/ORDER BY：索引联动的完整流水线

前面各节讲的是索引本身的能力，这一节把它们串起来：

> **一条 SQL 中的 WHERE、GROUP BY、ORDER BY 并不是三个独立的优化点，而是 MySQL 按照固定顺序依次利用索引。**

如果你继续用第一性原理看，会发现它们都在利用 B+Tree 的两个能力：

1. **快速定位（Seek）**
2. **天然有序（Ordered Scan）**

---

### SQL 的语义执行顺序

例如：

```sql
SELECT city, COUNT(*)
FROM user
WHERE age >= 18
GROUP BY city
ORDER BY city;
```

逻辑执行顺序（SQL 语义）：

```text
FROM
    ↓
WHERE
    ↓
GROUP BY
    ↓
HAVING
    ↓
SELECT
    ↓
ORDER BY
    ↓
LIMIT
```

而 **MySQL 利用索引的顺序** 基本围绕这个流程展开：

```text
① WHERE —— 先利用索引缩小数据
       ↓
② GROUP BY —— 再利用索引完成分组
       ↓
③ ORDER BY —— 再利用索引完成排序
       ↓
④ LIMIT —— 提前停止扫描
```

一句话：

> **先过滤，再聚合，再排序。前面的步骤如果没有利用索引建立正确的扫描顺序，后面的 GROUP BY 和 ORDER BY 往往也就失去了利用索引有序性的机会。**

---

### 第一阶段：WHERE —— 找数据（Seek）

```sql
WHERE age = 20
```

索引 `(age)` 对应的 B+Tree：

```text
10
15
20
20
20
21
22
```

数据库：**定位到 20 → 一直扫描**。

WHERE 用的是：**索引的定位能力（Seek）**。

---

### 第二阶段：GROUP BY —— 连续的数据天然是一组

GROUP BY 最怕什么？

```text
北京
上海
北京
广州
上海
```

数据库需要 Hash 或者临时表，不断统计。

如果数据已经变成：

```text
北京
北京
北京
上海
上海
广州
```

数据库只需要：**北京结束 → 输出 → 上海结束 → 输出**。

根本不用 Hash。

所以 GROUP BY 最喜欢：**数据已经按 GROUP BY 字段排好序**。

例如索引 `(city)`，数据在 B+Tree 里自然就是：

```text
北京
北京
北京
上海
上海
广州
```

`GROUP BY city` 只需扫描即可。这叫 **Loose Index Scan（松散索引扫描）**，EXPLAIN 中显示 `Using index for group-by`。

---

### 第三阶段：ORDER BY —— 利用已有顺序

如果 GROUP BY 后数据已经是：

```text
北京
上海
广州
```

而 SQL 是 `ORDER BY city` — 是不是已经有序？直接输出，不用 Filesort。

**但如果排序列和 GROUP BY 不同呢？**

```sql
GROUP BY city
ORDER BY COUNT(*)   -- count 是聚合后才算出来的，索引里没有
```

那就不行了。必须在统计完成之后重新排序。这时 EXPLAIN 就会看到 `Using filesort`。

---

### 一个完整例子

索引：`(city, age)`

```sql
SELECT city, COUNT(*)
FROM user
WHERE city = '北京'
GROUP BY city
ORDER BY city;
```

数据库的执行过程：

```text
第一步：Seek → 定位到"北京"
第二步：Ordered Scan → 北京/北京/北京/北京 天然就是一组
第三步：GROUP BY city → 直接聚合输出
第四步：ORDER BY city → 已经有序，直接输出
```

整个过程：**Seek → Scan → Output**。

没有临时表，没有 Filesort。

---

### 为什么 GROUP BY 有时不能利用索引？

索引：`(city, age)`

```sql
GROUP BY age
```

索引里的数据：

```text
北京 18
北京 20
北京 25
上海 18
上海 20
```

注意 age 列：

```text
18
20
25
18
20
```

age 并不是连续的。数据库无法"扫描→发现一组结束"，必须 Hash 或临时表。

**所以不能利用索引。**

---

### 联合索引案例

索引：`(city, age, name)`

```sql
WHERE city = '北京'
GROUP BY age
ORDER BY age
```

执行：

```text
第一步：Seek → 定位"北京"
得到：
    北京 18
    北京 18
    北京 20
    北京 20
    北京 30
```

对于"北京"内部，age 已经是连续的。

- GROUP BY age → 不用 Hash
- ORDER BY age → 不用 Filesort

整个 SQL：**Seek → Group Scan → Output**。

---

### 一个容易踩坑的例子

索引：`(city, age)`

```sql
SELECT city, age, COUNT(*)
FROM user
WHERE age > 20
GROUP BY city
ORDER BY city;
```

很多人以为：GROUP BY city，ORDER BY city，应该能利用索引。

**实际上不能。**

为什么？因为索引第一列是 `city`，而 WHERE 却是 `age > 20`。

数据库无法利用 `(city, age)` 索引快速定位满足 `age > 20` 的记录，只能扫描大量索引甚至回表，再进行过滤。这样进入 GROUP BY 时，数据已经不是按 city 连续读取的效果，通常就需要临时表和额外排序。

**关键原则：前面的步骤如果没有利用索引建立正确的扫描顺序，后面的 GROUP BY 和 ORDER BY 往往也就失去了利用索引有序性的机会。**

---

### 一张完整的流程图

```text
                SQL

                 │
                 ▼
        WHERE（利用索引定位）
        Seek / Range Scan
                 │
                 ▼
      GROUP BY（利用连续有序）
     顺序聚合 / Hash / 临时表
                 │
                 ▼
      ORDER BY（利用已有顺序）
    Index Order / Filesort
                 │
                 ▼
          LIMIT（提前停止扫描）
```

可以记成一句话：

> **WHERE 决定"从哪里开始读"；GROUP BY 决定"是否可以边读边聚合"；ORDER BY 决定"是否可以按读出的顺序直接输出"；LIMIT 决定"什么时候可以提前停止"。**

这也是 MySQL 优化器在设计索引时最核心的思考顺序：**先过滤（WHERE），再聚合（GROUP BY），最后排序（ORDER BY）**。如果一个联合索引能够同时满足这三个阶段的需求，就可能做到一次索引扫描完成整个查询，避免临时表、额外排序和大量回表。

---

## 完整知识树

```text
索引
│
├── 聚簇索引 → 数据本身
│
├── 二级索引 → 主键引用
│
├── 回表 → 二次查找
│
├── 覆盖索引 → 避免回表
│
├── 联合索引 → 多列排序
│
├── 最左匹配 → 排序规则
│
├── 索引失效 → 全表扫描
│
└── WHERE/GROUP BY/ORDER BY 联动 → 索引流水线
    ├── WHERE：索引定位（Seek）
    ├── GROUP BY：连续有序 → 顺序聚合
    └── ORDER BY：利用已有顺序 → 避免 Filesort
```

---

> 上一章：[第三章：存储引擎](../ch03-storage-engine/README.md)
> 下一章：[第五章：事务与 ACID](../ch05-transaction/README.md)
