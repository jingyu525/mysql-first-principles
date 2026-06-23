# 领域视角：并发篇

> 事务保证单个操作的正确性。当 10000 个事务同时执行时，如何保证既正确又高效？MVCC 解决读写冲突，Lock 解决写写冲突。两者共同构成 InnoDB 的并发控制体系。

---

## 核心问题

| 问题 | 答案 |
|------|------|
| 读写冲突怎么解决？ | MVCC：读旧版本，写新版本，互不阻塞 |
| 写写冲突怎么解决？ | Lock：行锁、间隙锁、Next-Key Lock |
| 数据怎么选版本？ | Read View + Undo 版本链 |
| 什么是幻读？ | 同一事务内两次范围查询结果不同 |
| 如何解决幻读？ | Gap Lock + Next-Key Lock |

---

## 三大主题

### MVCC

多版本并发控制。核心思想：**保存历史版本，读写分离**。Read 不阻塞 Write，Write 不阻塞 Read。

→ 详见：[第六章：MVCC](../../ch06-mvcc/README.md)

### ReadView

"快照机制"。事务开始时拍照记录当前活跃事务，后续查询都按照这张照片判断哪些版本可见。

→ 详见：[第六章：MVCC §6.6-6.7](../../ch06-mvcc/README.md)

### Lock

MVCC 只能解决读写冲突。写写冲突、幻读仍然需要锁。InnoDB 提供三级锁：Record Lock → Gap Lock → Next-Key Lock。

→ 详见：[第七章：锁](../../ch07-lock/README.md)

---

## 知识地图

```text
并发篇
│
├── MVCC（多版本并发控制）
│   ├── 解决什么问题？→ 读写冲突
│   ├── 核心思想：读历史版本，写最新版本
│   ├── 隐藏字段：trx_id + roll_pointer
│   ├── 版本链：通过 Undo Log 串联
│   ├── Read View（快照）
│   │   └── 事务开始时拍照 → 按快照判断可见性
│   └── 隔离级别
│       ├── Read Uncommitted → 脏读
│       ├── Read Committed → 不可重复读
│       ├── Repeatable Read → MySQL 默认
│       └── Serializable → 全加锁
│
└── Lock（锁）
    ├── 解决什么问题？→ 写写冲突 + 幻读
    ├── 锁的本质：访问资源的许可证
    ├── Record Lock（行锁）
    │   └── 锁一条记录，解决写写冲突
    ├── Gap Lock（间隙锁）
    │   └── 锁记录之间的空隙，阻止插入
    ├── Next-Key Lock（默认）
    │   └── Record + Gap，防止幻读
    ├── Lock Wait（锁等待）
    │   └── 排队等待锁释放
    └── Deadlock（死锁）
        ├── 循环等待 → 谁也走不出来
        ├── 四个必要条件（互斥/持有并等待/不可抢占/循环等待）
        └── InnoDB 检测 + 自动回滚
```

---

## MVCC vs Lock 分工

```text
并发控制
│
├── MVCC 负责
│   ├── 读 ↔ 写互不阻塞
│   ├── Read View + 版本链
│   └── 解决：脏读、不可重复读、部分幻读（快照读）
│
└── Lock 负责
    ├── 写 ↔ 写 互斥
    ├── Next-Key Lock
    └── 解决：丢失更新、当前读幻读
```

| 冲突类型 | 由谁解决 | 机制 |
|------|------|------|
| 读 ↔ 写 | MVCC | 读旧版本，写新版本 |
| 写 ↔ 写 | Lock | 行锁互斥 |
| 幻读（快照读） | MVCC | Read View 快照隔离 |
| 幻读（当前读） | Lock | Next-Key Lock |

---

## 快速通道

```
推导链中的对应章节：
  [ch06 MVCC](../../ch06-mvcc/README.md)
  → [ch07 锁](../../ch07-lock/README.md)

交叉领域：
  并发 + 事务 → Undo Log 的版本链（见 [事务篇](../02-事务篇/README.md)）
  并发 + 执行 → SELECT ... FOR UPDATE 的锁定读（见 [SQL执行篇](../04-SQL执行篇/README.md)）
  并发 + 内核 → Purge Thread 清理旧版本（见 [InnoDB内核篇](../05-InnoDB内核篇/README.md)）
```

---

## 核心推论

1. **MVCC 本质是空间换时间** — 保留历史版本，换取读写不阻塞
2. **Repeatable Read 的实现** — Read View（事务开始时拍快照）+ Undo 版本链（沿链表回退）
3. **死锁不可避免** — 只要满足四个必要条件就可能发生，MySQL 只能检测 + 回滚，不能预防

---

> 返回：[领域视角总览](../README.md)
