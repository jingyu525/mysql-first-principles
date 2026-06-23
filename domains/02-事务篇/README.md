# 领域视角：事务篇

> 索引解决「如何快速找到数据」，事务解决「如何保证数据永远正确」。这是从性能问题进入正确性问题的转折点。

---

## 核心问题

| 问题 | 答案 |
|------|------|
| 为什么需要事务？ | 单个"业务动作"在数据库中是多个物理操作，中间崩溃会导致数据不一致 |
| 事务的本质是什么？ | **状态机**：只有 Commit 或 Rollback 两种终态 |
| 如何保证已提交不丢？ | Redo Log（WAL 思想：先写日志，再写数据） |
| 如何保证未提交可回滚？ | Undo Log（保存历史版本） |
| 崩溃后如何恢复？ | Redo Log → 重做已提交事务 → Crash Recovery |

---

## 三大主题

### ACID

事务四大特性的逐一推导 — 不是背概念，而是理解四个问题及其解决方案。

→ 详见：[第五章：事务与 ACID §5.3](../../ch05-transaction/README.md)

### Undo Log

双身份：① 回滚时恢复旧值；② 为 MVCC 提供历史版本。保存的是"旧状态"。

→ 详见：[第五章：事务与 ACID §5.7-5.8](../../ch05-transaction/README.md) + [第六章：MVCC §6.4](../../ch06-mvcc/README.md)

### Redo Log

InnoDB 最伟大的设计之一。WAL（Write Ahead Logging）思想：先写几十字节的日志，再慢慢刷 16KB 的 Page。速度快且不丢数据。

→ 详见：[第五章：事务与 ACID §5.5-5.6](../../ch05-transaction/README.md) + [第九章：InnoDB 内核 §9.10-9.12](../../ch09-innodb/README.md)

---

## 知识地图

```text
事务篇
│
├── 为什么需要事务？
│   └── 转账案例：A-100, B+100 必须同时成功或同时失败
│
├── ACID（四个问题，四个解决方案）
│   ├── A 原子性：做了一半怎么办？→ 全部成功 or 全部失败
│   ├── C 一致性：数据规则会不会被破坏？→ 事务前后状态一致
│   ├── I 隔离性：多个事务同时执行怎么办？→ MVCC 解决（见并发篇）
│   └── D 持久性：提交后断电怎么办？→ Redo Log 保证
│
├── 事务状态机
│   └── BEGIN → Running → Commit / Rollback
│
├── Buffer Pool 带来的风险
│   └── 内存改了，磁盘没改 → 断电丢失 → 需要 Redo Log
│
├── Undo Log（回滚机制）
│   ├── 记录旧值（"我原来是什么"）
│   ├── 事务失败 → 读取 Undo Log → 恢复旧值
│   └── 第二用途：为 MVCC 提供历史版本链（见并发篇）
│
└── Redo Log（持久化机制）
    ├── WAL：先写日志，再写数据
    ├── Redo = 记录"我做了什么"（Page 8, Offset 100, 写入 900）
    ├── 为什么快？几十字节 vs 16KB Page
    ├── Crash Recovery：扫描 Redo Log → 重新执行已提交事务
    └── Checkpoint：存档点，前面的 Redo Log 可以删除
```

---

## Redo 与 Undo 对照

```text
                事务
                  │
        ┌─────────┴─────────┐
        │                   │
        ▼                   ▼
   Redo Log            Undo Log
   新状态                旧状态
   用于恢复              用于回滚
   面向未来              面向过去
```

| | Redo Log | Undo Log |
|------|----------|----------|
| 记录什么 | 做了什么修改 | 原来的值是什么 |
| 用途 | 崩溃恢复 | 事务回滚 + MVCC |
| 方向 | 面向未来 | 面向过去 |
| 写入时机 | 修改 Buffer Pool 后 | 修改 Buffer Pool 前 |

---

## 快速通道

```
事务的执行流程（简化）：
  BEGIN → 生成 Undo → 修改 Buffer Pool → 生成 Redo → 写 Redo 到磁盘 → Commit

推导链中的对应章节：
  [ch05 事务与 ACID](../../ch05-transaction/README.md)
  → Redo/Undo 的物理实现见 [ch09 InnoDB 内核](../../ch09-innodb/README.md)
  → Undo 的第二身份（MVCC）见 [ch06 MVCC](../../ch06-mvcc/README.md)

交叉领域：
  事务 + 并发 → MVCC 隔离级别（见 [并发篇](../03-并发篇/README.md)）
  事务 + 存储 → Buffer Pool/Page/刷盘（见 [存储篇](../01-存储篇/README.md)）
  事务 + 恢复 → WAL/Checkpoint/Crash Recovery（见 [InnoDB内核篇](../05-InnoDB内核篇/README.md)）
```

---

## 核心推论

1. **WAL 是数据库持久化的基石** — 几乎所有数据库都有类似机制
2. **Undo 是一石二鸟** — 既服务回滚，又服务 MVCC
3. **事务的代价** — Redo/Undo 都要写磁盘，这是事务的性能成本

---

> 返回：[领域视角总览](../README.md)
