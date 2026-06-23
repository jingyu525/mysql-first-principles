# 领域视角：InnoDB 内核篇

> 从「数据库使用者」切换到「数据库开发者」视角。InnoDB 本质上是一个**管理数据的微型操作系统**。它负责：内存管理、磁盘管理、事务管理、日志管理、并发控制、崩溃恢复。

---

## 核心问题

| 问题 | 答案 |
|------|------|
| InnoDB 是什么？ | 管理数据的微型操作系统 |
| 数据在内存中的组织形式？ | Buffer Pool（缓存热点 Page） |
| 内存改了，磁盘没改怎么办？ | Dirty Page → Flush 刷盘 |
| 刷盘前断电怎么办？ | Redo Log（WAL）→ Crash Recovery |
| Redo Log 无限增长怎么办？ | Checkpoint（存档点） |
| 历史版本太多怎么办？ | Purge Thread（后台清理） |
| 崩溃后怎么恢复？ | 启动时扫描 Redo Log → 重做已提交事务 |

---

## 四大主题

### Buffer Pool

InnoDB 最核心的内存模块。磁盘太慢，用内存缓存热点 Page。类似 CPU Cache、Redis — 思想一致。

→ 详见：[第九章：InnoDB 内核 §9.6-9.7](../../ch09-innodb/README.md) | [第三章：存储引擎 §3.4](../../ch03-storage-engine/README.md)

### Flush & Checkpoint

Dirty Page 最终必须落盘（Flush），但不能每次都刷（性能太差）。Checkpoint 是存档点，之前的 Redo Log 可以安全删除。

→ 详见：[第九章：InnoDB 内核 §9.8-9.11](../../ch09-innodb/README.md)

### Recovery（崩溃恢复）

> ⚠️ 此主题在正文章节中分布在多处，此处做统一梳理。

崩溃恢复的完整流程：

```text
系统崩溃（Buffer Pool 全部丢失）
             │
             ▼
    数据库重启
             │
             ▼
    扫描 Redo Log
             │
    ┌────────┴────────┐
    │                 │
    ▼                 ▼
  已提交事务         未提交事务
    │                 │
    ▼                 ▼
  重做 Redo         读取 Undo → 回滚
    │                 │
    ▼                 ▼
  恢复到最新状态    恢复到事务开始前
```

**恢复的两个阶段：**

1. **Redo 阶段**：扫描 Redo Log，重做所有已提交事务的修改（包括 Page 在 Checkpoint 之后但 Redo 已写盘的）
2. **Undo 阶段**：回滚所有未提交事务（通过 Undo Log 恢复旧值）

**为什么恢复可行？**

- Redo Log 先于数据 Page 落盘（WAL 思想）
- 只要 Redo Log 在磁盘上，数据就不会丢
- Checkpoint 之后的 Redo Log 必须保留，之前的可以删除

### Purge Thread

MVCC 保留历史版本，随时间推移版本越来越多。Purge Thread（后台线程）删除不再需要的历史版本，回收空间。

→ 详见：[第九章：InnoDB 内核 §9.13](../../ch09-innodb/README.md)

---

## 知识地图

```text
InnoDB内核篇
│
├── 存储结构（物理层级）
│   ├── Table（表空间）
│   ├── Segment（段：聚簇索引一个段，二级索引一个段）
│   ├── Extent（区：1 Extent = 64 Page = 1MB）
│   └── Page（页：最小 IO 单位 = 16KB）
│
├── 内存系统
│   ├── Buffer Pool（缓存热点 Page）
│   ├── Dirty Page（内存 ≠ 磁盘）
│   ├── Flush（脏页刷盘）
│   └── 类比：CPU Cache / Redis / Write Back Cache
│
├── 日志系统
│   ├── Redo Log（WAL：先写日志，再写数据）
│   ├── Undo Log（保存历史版本）
│   ├── Checkpoint（存档点，循环日志）
│   └── Crash Recovery（崩溃恢复）
│
├── 并发控制
│   ├── MVCC ← 读写不阻塞
│   ├── Lock ← 写写互斥
│   └── Read View ← 快照隔离
│
└── 后台线程
    ├── Master Thread（主线程：刷盘/合并/清理）
    ├── IO Thread（异步 IO）
    ├── Purge Thread（清理旧版本）
    └── Page Cleaner Thread（脏页刷盘）
```

---

## 一条 UPDATE 的完整旅程

```sql
UPDATE account SET balance = 900 WHERE id = 1;
```

```text
1. SQL → Parser → Optimizer → 选择走主键索引
2. B+Tree 定位 → 找到对应的 Page
3. 加载 Page 到 Buffer Pool
4. 生成 Undo Log（记录旧值 balance=1000）
5. 修改 Buffer Pool 中的 Page（balance=900，标记为 Dirty Page）
6. 生成 Redo Log（记录修改内容）
7. 写 Redo Log 到磁盘
8. Commit 成功（用户收到"更新成功"）
9. 后台线程异步 Flush（Dirty Page → 磁盘）
10. 将来某个 Checkpoint → 前面的 Redo Log 删除
```

---

## 快速通道

```
推导链中的对应章节：
  [ch03 存储引擎](../../ch03-storage-engine/README.md) — Page/Buffer Pool
  → [ch09 InnoDB 内核](../../ch09-innodb/README.md) — 完整内核剖析

交叉领域：
  内核 + 存储 → Page/B+Tree/聚簇索引的物理实现（见 [存储篇](../01-存储篇/README.md)）
  内核 + 事务 → Redo/Undo 的日志机制（见 [事务篇](../02-事务篇/README.md)）
  内核 + 并发 → MVCC 版本链 + Lock 管理器（见 [并发篇](../03-并发篇/README.md)）
```

---

## 核心推论

1. **InnoDB = 数据操作系统** — 类比 OS：Page=内存页，Buffer Pool=物理内存，Redo Log=WAL，Checkpoint=存档
2. **WAL 是持久化的基石** — 只要 Redo Log 落盘，数据就不会丢
3. **延迟刷盘是性能关键** — 修改只改内存 + 写小日志，刷盘交给后台线程异步做
4. **Checkpoint 是循环日志的关键** — 没有 Checkpoint，Redo Log 会无限增长

---

> 返回：[领域视角总览](../README.md)
