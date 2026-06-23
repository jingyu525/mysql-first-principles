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

如果你有操作系统、数据结构等背景，本教程中的类比会帮助你更快建立直觉。

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

## 两套阅读路径

本教程提供**双视角**学习，可按需选择：

| 路径 | 方式 | 适合 |
|------|------|------|
| **推导链视角** | [ch01 → ch02 → ... → ch09](SUMMARY.md)，因果递进 | 第一次系统学习 |
| **领域视角** | [domains/](domains/README.md)，按功能域横切 | 复习 / 查漏补缺 / 面试准备 |

两者内容互补，不重复：推导链回答"为什么"，领域视角回答"这一块有哪些东西"。

## 如何使用本教程

1. **按顺序阅读**：每章建立在前一章的基础上
2. **先理解"为什么"**：每个概念都从它要解决的问题开始推导
3. **关注推导过程**：数据结构的演变（数组→链表→BST→B树→B+树）比结论更重要
4. **配合动手实验**：建议在阅读时打开 MySQL 客户端，执行每章的 SQL 示例
5. **学完后切换视角**：通读一遍后，用 [领域视角](domains/README.md) 做全局回顾

---

## 项目结构

```text
mysql-first-principles/
├── README.md              ← 你在这里（双视角入口）
├── SUMMARY.md             ← 目录索引
├── roadmap.md             ← 学习路线图
├── ch01-worldview/        ← 推导链视角（纵向：从零到一）
├── ch02-data-structure/   │
├── ch03-storage-engine/   │
├── ch04-index/            │
├── ch05-transaction/      │
├── ch06-mvcc/             │
├── ch07-lock/             │
├── ch08-explain/          │
├── ch09-innodb/           ←
├── domains/               ← 领域视角（横向：按模块俯瞰）
│   ├── 00-数据库是什么/
│   ├── 01-存储篇/
│   ├── 02-事务篇/
│   ├── 03-并发篇/
│   ├── 04-SQL执行篇/
│   ├── 05-InnoDB内核篇/
│   └── 06-数据库设计哲学/
└── appendix/              ← 附录（思维导图/配图/练习）
```

---

## License

MIT
