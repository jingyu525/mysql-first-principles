# MySQL 第一性原理

> 不要从 SQL 语法开始学 MySQL，而是先建立一套「数据库世界观」。

很多教程直接讲 `SELECT`、`INSERT`、`JOIN`，最后你还是不会设计表、不会分析慢查询。本质原因是**缺少底层认知**。

本教程采用**第一性原理**方法，从"为什么会有这个设计"出发，层层推导 MySQL 的核心机制。目标是让你把 MySQL 看成一个**管理磁盘数据的操作系统**，而不是一堆 SQL 语法。

---

## 一页纸地图

```text
MySQL
│
├── 数据存储
│   ├── 表
│   ├── 行
│   └── 列
│
├── 数据查询
│   ├── SQL
│   ├── Parser
│   ├── Optimizer
│   └── Executor
│
├── 数据定位
│   ├── 主键
│   ├── 索引
│   └── B+Tree
│
├── 数据一致性
│   ├── 事务
│   ├── 锁
│   ├── MVCC
│   └── ACID
│
├── 数据持久化
│   ├── Buffer Pool
│   ├── Redo Log
│   ├── Undo Log
│   └── Checkpoint
│
└── 性能优化
    ├── EXPLAIN
    ├── 慢查询
    ├── 索引优化
    └── SQL优化
```

---

## 目标读者

涵盖**从入门到进阶**的全阶段：

| 阶段 | 你可能是 | 本教程给你的 |
|------|---------|------------|
| 入门 | MySQL 初学者 | 建立底层世界观，知道每个概念为什么存在 |
| 进阶 | 有 SQL 基础的后端 | 理解索引、事务、MVCC 的内部机理 |
| 高级 | 需要分析慢查询的工程师 | 连接 EXPLAIN → 索引 → Page → IO 的完整链路 |

如果你有 Go Runtime、操作系统、数据结构等背景，本教程中的类比会帮助你更快建立直觉。

---

## 学习路线

```text
数据结构
    ↓
B+Tree
    ↓
索引
    ↓
事务
    ↓
MVCC
    ↓
锁
    ↓
执行计划(EXPLAIN)
    ↓
InnoDB源码思想
```

---

## 目录

| 章节 | 主题 | 核心问题 |
|------|------|---------|
| [第一章](ch01-worldview/README.md) | 数据库世界观 | 为什么会有 MySQL？ |
| [第二章](ch02-data-structure/README.md) | 数据结构到 B+树 | 为什么要用 B+Tree？ |
| [第三章](ch03-storage-engine/README.md) | 存储引擎 | 磁盘→Page→Buffer Pool |
| [第四章](ch04-index/README.md) | 索引体系 | 索引到底怎么工作？ |
| [第五章](ch05-transaction/README.md) | 事务与 ACID | 如何保证数据正确？ |
| [第六章](ch06-mvcc/README.md) | MVCC | 如何既正确又高并发？ |
| [第七章](ch07-lock/README.md) | 锁 | MVCC 还不够怎么办？ |
| [第八章](ch08-explain/README.md) | EXPLAIN | SQL 到底怎么执行的？ |
| [第九章](ch09-innodb/README.md) | InnoDB 内核 | 把 MySQL 看成数据操作系统 |

---

## 与 Go Runtime 的对应关系

如果你熟悉 Go Runtime，可以建立如下类比：

| Go Runtime | InnoDB |
|-----------|--------|
| G | Transaction |
| GMP 调度 | Lock/MVCC 协调 |
| Heap | Buffer Pool |
| GC | Purge Thread |
| pprof | EXPLAIN |
| Channel | Lock Wait Queue |
| 状态机 | Transaction State |

---

## 如何使用本教程

1. **按顺序阅读**：每章建立在前一章的基础上
2. **先理解"为什么"**：每个概念都从它要解决的问题开始推导
3. **关注推导过程**：数据结构的演变（数组→链表→BST→B树→B+树）比结论更重要
4. **配合动手实验**：建议在阅读时打开 MySQL 客户端，执行每章的 SQL 示例

---

## 项目结构

```text
mysql-first-principles/
├── README.md              ← 你在这里
├── SUMMARY.md             ← 目录索引
├── roadmap.md             ← 学习路线图
├── ch01-worldview/        ← 数据库世界观
├── ch02-data-structure/   ← 数据结构到 B+树
├── ch03-storage-engine/   ← 存储引擎
├── ch04-index/            ← 索引体系
├── ch05-transaction/      ← 事务与 ACID
├── ch06-mvcc/             ← MVCC
├── ch07-lock/             ← 锁
├── ch08-explain/          ← EXPLAIN
├── ch09-innodb/           ← InnoDB 内核
└── appendix/              ← 附录（思维导图/配图/练习）
```

---

## License

MIT
