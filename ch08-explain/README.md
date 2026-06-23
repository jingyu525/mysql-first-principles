# 第八章：EXPLAIN（执行计划）

> 你会索引原理，会事务原理。可是面对一条 SQL，你仍然不知道：MySQL 到底是怎么执行的？为什么有时候快？为什么有时候慢？为什么明明建了索引却没用？

---

## 8.1 EXPLAIN 到底是什么？

很多教程教你背 `type`、`key`、`rows`、`extra` — 这是错误学习方式。

先理解：你写的 SQL 不会直接执行。

```text
SQL → Parser → Optimizer → Executor → InnoDB
```

这里最关键的是 **Optimizer（优化器）**。

---

## 8.2 优化器为什么存在？

从北京到上海有很多路线：高铁、飞机、开车。目标：成本最低或时间最短。

数据库也一样。`SELECT * FROM user WHERE age=18` — 可能走索引，也可能全表扫描。**数据库必须选择一个**。

决策者：**Optimizer**。

---

## 8.3 EXPLAIN 本质

一句话：**查看优化器的决策结果**。

类似 Go 的 `go tool pprof`：看 CPU 到底耗在哪里。EXPLAIN：看 SQL 到底怎么执行。

### 一条 SQL 的生命周期

```text
步骤1: Parser → 解析 SQL 是否合法，生成语法树
步骤2: Optimizer → 走索引还是全表扫描？
步骤3: Executor → 按照选好的路线执行
步骤4: InnoDB → 访问 Page / B+Tree / Buffer Pool → 返回结果
```

---

## 8.4 type（访问方式）

表示**怎么找数据**。从好到坏：

| type | 含义 | 性能 |
|------|------|------|
| `const` | 主键定位 | 最好 |
| `eq_ref` | 关联查询唯一匹配 | 很好 |
| `ref` | 索引查找 | 好 |
| `range` | 索引范围扫描 | 一般 |
| `index` | 索引全扫描 | 差 |
| `ALL` | 全表扫描 | 最差 |

### ALL

`WHERE age+1=18` → 索引失效 → 只能第一页、第二页...全部扫描。

### const

`WHERE id=1` → Root → Middle → Leaf，直接找到。

---

## 8.5 key（实际使用的索引）

- `key=idx_age` → 索引命中了 ✅
- `key=NULL` → 没用索引 ❌

---

## 8.6 rows（预估扫描行数）

- `rows=1` → 几乎瞬间 ✅
- `rows=10000000` → **危险**，扫描太多 Page ❌

---

## 8.7 Extra（关键附加信息）

### Using index

覆盖索引，不需要回表 → **好事** ✅

### Using where

MySQL 还要继续过滤。如 `WHERE age=18 AND city='北京'`，索引只有 age → 先找 age=18，再逐行判断 city。

### Using filesort

经典性能问题。`ORDER BY age` 没有对应索引 → MySQL 只能内存排序或磁盘排序。**看到这个要警觉** ⚠️

### Using temporary

创建临时表。常见于 `GROUP BY`。说明额外开销 ⚠️

---

## 8.8 为什么建了索引还是慢？

线上最常见问题。

### 例1：函数运算

```sql
WHERE age+1=18  -- 索引存 age，不是 age+1 → 优化器放弃索引
```

### 例2：前缀模糊

```sql
WHERE name LIKE '%张'  -- 不知道从哪里开始找 → 放弃索引
```

### 例3：数据分布问题

```sql
WHERE gender='男'
-- 1000 万数据：男 500 万，女 500 万
-- 即使用索引仍扫描 500 万 → 优化器判断：全表扫描更快
```

---

## 8.9 执行计划背后的本质（CBO）

优化器不关心"有没有索引"，只关心**哪个成本最低**。

**Cost Based Optimizer（成本优化器）**。

```text
总代价 = IO 成本 + CPU 成本
IO 成本 >> CPU 成本
最终目标：减少 Page 访问
```

---

## 从 EXPLAIN 回到知识树

| 看到 | 想到 |
|------|------|
| `Using index` | 覆盖索引 |
| 回表 | 二级索引 → 聚簇索引 |
| `rows=1000000` | 大量 Page 访问 |
| `ALL` | 全表扫描 |
| Lock Wait | 事务与锁 |

---

> 上一章：[第七章：锁](../ch07-lock/README.md)
> 下一章：[第九章：InnoDB 内核](../ch09-innodb/README.md)
