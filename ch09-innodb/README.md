# 第九章：InnoDB 内核

> 从「数据库使用者」切换到「数据库开发者」的视角。InnoDB = 管理数据的微型操作系统。

---

## 9.1 重新定义 InnoDB

很多人认为：MySQL = 数据库软件。

实际上：

```text
MySQL Server = SQL 层
InnoDB = 存储引擎
```

更准确地说：**InnoDB = 管理数据的微型操作系统**。

它负责：内存管理、磁盘管理、事务管理、日志管理、并发控制、崩溃恢复。

---

## 9.2 为什么需要 Page？

很多初学者认为 `SELECT * FROM user WHERE id=1` 读取一条记录。实际上：**磁盘无法按记录读取，只能按块读取**。

```text
Page = InnoDB 最小 IO 单位（默认 16KB）
```

类比：

```text
CPU 最小执行单位 → 指令
OS 最小调度单位 → 线程
InnoDB 最小读写单位 → Page
```

---

## 9.3 Page 为什么重要？

因为整个 InnoDB 都建立在 Page 上：

```text
B+Tree 节点 = Page
Buffer Pool 缓存 = Page
Redo Log 恢复 = Page
刷盘 = Page
```

**理解 Page ≈ 理解 InnoDB**。

---

## 9.4 Extent（区）

100GB 数据，一个 Page 16KB → 约 600 多万个 Page。如何管理？

于是出现 **Extent（区）**：

```text
Page = 书页
Extent = 章节

1 Extent = 64 Page = 1MB
```

磁盘分配更高效。

---

## 9.5 Segment（段）

数百万 Page、数万个 Extent，仍然不好管理。

于是 **Segment（段）** 出现：

```text
Segment = 文件夹
  ├── Extent
  ├── Extent
  └── Extent
```

例如：聚簇索引一个 Segment，二级索引一个 Segment。

完整结构：

```text
Table → Segment → Extent → Page
```

---

## 9.6 Buffer Pool

这是 InnoDB 最核心模块。

问题：磁盘太慢。解决：**用内存缓存热点 Page**。

```text
查询流程：
先查 Buffer Pool
  命中 → 直接返回
  未命中 → 读磁盘 → 放入 Buffer Pool → 返回
```

本质：**缓存系统**。类似 CPU Cache、Redis — 思想一致。

---

## 9.7 脏页（Dirty Page）

修改数据 `UPDATE ...` 时：

```text
磁盘 → Page → Buffer Pool → 修改
```

此时：**内存 = 新数据，磁盘 = 旧数据**。

这种 Page 叫 **Dirty Page（脏页）**。不是错误，只是内存与磁盘不一致。

---

## 9.8 Flush（刷盘）

脏页最终必须落盘：

```text
Dirty Page → Flush → Disk
```

类似：Word 编辑文档 → Ctrl+S → 保存到磁盘。

---

## 9.9 为什么不能每次修改都刷盘？

执行 `UPDATE` 10000 次，每次都刷盘 → 10000 次随机 IO → 性能极差。

InnoDB 选择**延迟刷盘**：先改 Buffer Pool，后台线程异步 Flush。

这就是 **Write Back Cache** 思想。

### 风险与应对

数据只在内存 → 突然断电 → 怎么办？

这就是 **Redo Log** 存在的原因。

---

## 9.10 Redo Log & WAL

核心思想：**先记账，再干活**。

```text
修改：Page 100, Offset 50, 写入 900
  ↓
先写 Redo Log
  ↓
Commit 成功
  ↓
以后慢慢刷盘
```

这就是 **WAL（Write Ahead Logging）**。

---

## 9.11 Checkpoint

问题：Redo Log 会无限增长吗？显然不行。

于是 **Checkpoint** 出现。理解：**游戏存档点**。

表示：Checkpoint 之前，所有 Page 已安全落盘。前面部分的 Redo Log 可以删除 → 形成循环日志。

---

## 9.12 Crash Recovery

系统崩溃 → Buffer Pool 全部丢失 → 怎么办？

数据库启动 → 扫描 Redo Log → 发现哪些事务已提交 → 重新执行 Redo → 恢复最新状态。

---

## 9.13 Purge Thread

MVCC 保留历史版本，随时间推移版本越来越多。必须清理。

**Purge Thread（后台线程）**：删除不再需要的历史版本，回收空间。

类似：**Go GC**。

---

## 9.14 InnoDB 完整流程

```sql
UPDATE account SET balance=900 WHERE id=1;
```

实际发生：

```text
SQL → Optimizer → B+Tree → 找到 Page
  → 加载 Buffer Pool → 生成 Undo Log
  → 修改 Page → 生成 Redo Log → Commit
  → 后台 Flush → Checkpoint → Purge
```

---

## 最终一页纸地图

```text
InnoDB
│
├── 存储结构
│   ├── Segment
│   ├── Extent
│   └── Page
│
├── 内存系统
│   └── Buffer Pool
│
├── 索引系统
│   └── B+Tree
│
├── 事务系统
│   ├── Undo Log
│   ├── Redo Log
│   └── MVCC
│
├── 并发控制
│   ├── Lock
│   └── Read View
│
├── 刷盘机制
│   ├── Dirty Page
│   ├── Flush
│   └── Checkpoint
│
└── 崩溃恢复
    ├── WAL
    ├── Crash Recovery
    └── Purge Thread
```

---

## 最终结论

```text
数据库不是 SQL 语言。

数据库本质上是：
一个管理磁盘数据的操作系统。
```

而 InnoDB 就是这个「数据操作系统」的内核。

至此，你已经完成了从 **数据结构 → B+Tree → 索引 → 事务 → MVCC → 锁 → EXPLAIN → InnoDB** 的完整主线。

---

> 上一章：[第八章：EXPLAIN](../ch08-explain/README.md)
> 返回：[目录](../SUMMARY.md)
