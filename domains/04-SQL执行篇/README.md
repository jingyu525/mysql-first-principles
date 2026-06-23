# 领域视角：SQL执行篇

> 你写的 SQL 不会直接执行。它经过 Parser、Optimizer、Executor，最终由 InnoDB 完成实际的磁盘读写。EXPLAIN 就是观察这条流水线的窗口。

---

## 核心问题

| 问题 | 答案 |
|------|------|
| SQL 怎样变成磁盘操作？ | Parser → Optimizer → Executor → InnoDB |
| 为什么同样的 SQL 有时快有时慢？ | 优化器选择了不同的执行计划 |
| 为什么建了索引还是全表扫描？ | 优化器判断全表扫描成本更低 |
| 如何诊断慢查询？ | EXPLAIN + 慢查询日志 |
| 优化器的决策依据是什么？ | CBO：Cost Based Optimizer（IO 成本 + CPU 成本） |

---

## 三大主题

### Optimizer

优化器是 SQL 执行的决策者。从北京到上海可以高铁/飞机/开车，数据库也一样：可以走索引，也可以全表扫描。优化器选成本最低的那个。

→ 详见：[第八章：EXPLAIN §8.2-8.3](../../ch08-explain/README.md) | [第一章：数据库世界观 §1.3](../../ch01-worldview/README.md)

### EXPLAIN

查看优化器决策结果的工具。核心字段：`type`（访问方式）、`key`（使用的索引）、`rows`（预估扫描行数）、`Extra`（附加信息如 Using index/Using filesort）。

→ 详见：[第八章：EXPLAIN §8.4-8.8](../../ch08-explain/README.md)

### SlowSQL（慢查询）

> ⚠️ 此主题在推导链正文章节中尚未展开独立章节，以下为核心内容。

慢查询是线上性能问题的最常见入口。处理思路：

**发现慢查询：**

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过 1 秒记录
```

**分析慢查询：**

```sql
EXPLAIN SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC;
```

**常见原因与对策：**

| 原因 | EXPLAIN 表现 | 对策 |
|------|-------------|------|
| 没走索引 | `type=ALL` | 加索引 |
| 回表太多 | `key` 有值但 `rows` 大 | 覆盖索引 |
| 排序无索引 | `Extra=Using filesort` | 加联合索引覆盖排序 |
| 临时表 | `Extra=Using temporary` | 优化 GROUP BY 或加索引 |
| 索引失效 | `type=ALL, key=NULL` | 检查函数运算/前缀模糊/隐式类型转换 |
| 分页过深 | `LIMIT 1000000,10` | 游标分页或延迟关联 |
| Join 驱动表选错 | `rows` 差距大 | STRAIGHT_JOIN 或调整索引 |

**思考框架：**

```
慢查询 → EXPLAIN 定位瓶颈 → 判断是索引问题还是 SQL 写法问题
                          ↓
          索引问题 → 加索引 / 覆盖索引 / 联合索引
          SQL写法 → 改写查询 / 减少回表 / 避免 filesort
```

---

## 知识地图

```text
SQL执行篇
│
├── SQL 生命周期
│   ├── Parser：解析 SQL 是否合法 → 生成语法树
│   ├── Optimizer：选择执行路径（走索引 or 全表扫描？）
│   ├── Executor：按选好的路线执行
│   └── InnoDB：访问 Page / B+Tree / Buffer Pool → 返回结果
│
├── CBO（Cost Based Optimizer）
│   ├── 总代价 = IO 成本 + CPU 成本
│   ├── IO 成本 >> CPU 成本
│   └── 最终目标：减少 Page 访问
│
├── EXPLAIN
│   ├── type：访问方式（const > eq_ref > ref > range > index > ALL）
│   ├── key：实际使用的索引
│   ├── rows：预估扫描行数（越大越危险）
│   └── Extra：Using index / Using where / Using filesort / Using temporary
│
└── SlowSQL
    ├── 发现：慢查询日志
    ├── 分析：EXPLAIN + 对应章节知识
    └── 优化：覆盖索引 / 改写 SQL / 调整索引
```

---

## EXPLAIN 速查表

| 看到 | 想到 | 对策 |
|------|------|------|
| `type=ALL` | 全表扫描 | 加索引 |
| `type=index` | 索引全扫描 | 索引覆盖不够 |
| `type=range` | 范围扫描 | 可接受 |
| `type=ref` | 索引查找 | 正常 |
| `rows=10000000` | 大量 Page 访问 | 检查是否需要加索引 |
| `Using index` | 覆盖索引 | ✅ 最优 |
| `Using filesort` | 额外排序开销 | ⚠️ 加索引覆盖排序 |
| `Using temporary` | 创建临时表 | ⚠️ 优化 GROUP BY |
| `Using where` | 回表后还要过滤 | 考虑联合索引 |
| `key=NULL` | 没用索引 | 检查索引失效原因 |

---

## 快速通道

```
推导链中的对应章节：
  [ch01 数据库世界观 §1.3](../../ch01-worldview/README.md) — MySQL 内部架构
  → [ch08 EXPLAIN](../../ch08-explain/README.md) — 执行计划详解

交叉领域：
  SQL执行 + 索引 → type/rows 背后是聚簇/二级/覆盖索引（见 [存储篇](../01-存储篇/README.md)）
  SQL执行 + 并发 → SELECT ... FOR UPDATE 的锁定读（见 [并发篇](../03-并发篇/README.md)）
  SQL执行 + 内核 → Buffer Pool 命中率对查询性能的影响（见 [InnoDB内核篇](../05-InnoDB内核篇/README.md)）
```

---

## 核心推论

1. **优化器不关心"有没有索引"，只关心"哪个成本低"** — 数据分布会导致有索引也不走
2. **EXPLAIN 只是预估** — `rows` 是估算值，不代表实际执行情况
3. **慢查询优化的黄金链路** — EXPLAIN → 回表/覆盖索引 → Page/Buffer Pool → IO 代价

---

> 返回：[领域视角总览](../README.md)
