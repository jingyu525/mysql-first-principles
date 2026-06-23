# 第七章：锁（Lock）

> MVCC 解决读写冲突，但写写冲突、幻读、范围查询仍然需要锁。

---

## 7.1 为什么 MVCC 还不够？

很多初学者学完 MVCC 会产生一个误解：MVCC 已经解决并发问题了。

实际上：**MVCC 只解决「读↔写」冲突，不能解决「写↔写」冲突**。

### 丢失更新（Lost Update）

```text
balance = 1000

事务A: UPDATE ... SET balance = balance - 100  -- 读取 1000，写入 900
事务B: UPDATE ... SET balance = balance - 50   -- 读取 1000，写入 950
```

同时执行 → 最终 950，正确结果应该是 **850**。

**MVCC 解决读写冲突，Lock 解决写写冲突。**

---

## 7.2 锁的第一性原理

很多人认为锁是某种数据库功能。其实：

**锁 = 访问资源的许可证**。

生活例子：厕所。一个坑位同时只能一个人使用。进入=获得锁，离开=释放锁。

修改数据：**先获得锁 → 再修改 → 最后释放锁**。

---

## 7.3 行锁（Record Lock）

最基础的锁。

事务 A：`UPDATE user SET age=20 WHERE id=1;` → id=1 被锁定。

事务 B：`UPDATE user SET age=30 WHERE id=1;` → 必须等待事务 A 提交。

```text
事务A          事务B
获得 id=1 锁    等待
修改数据          ↓
Commit           ↓
释放锁           获得 id=1 锁
                 修改数据
```

这种锁叫 **Record Lock（记录锁）**。

---

## 7.4 为什么会出现幻读？

表：id=1,2,3。

事务 A：`SELECT * FROM user WHERE id > 3;` → 空。

事务 B：`INSERT INTO user VALUES(4); COMMIT;`。

事务 A 再次查询：`SELECT * FROM user WHERE id > 3;` → 出现 id=4。

第一次没有，第二次有，像突然出现的幻觉。所以叫 **Phantom Read（幻读）**。

---

## 7.5 为什么行锁解决不了幻读？

因为 id=4 **原来根本不存在**。你无法给不存在的记录加锁。

Record Lock 只能锁存在的数据，无法阻止新数据插入。

---

## 7.6 Gap Lock（间隙锁）

为了解决幻读，MySQL 发明了 **Gap Lock（间隙锁）**。

表：1, 3, 5, 7。

`SELECT * FROM user WHERE id BETWEEN 3 AND 5 FOR UPDATE;` → 锁住区间 `(3,5)`。

`INSERT INTO user VALUES(4);` → 被阻塞（4 落在锁定区间）。

**Gap Lock 锁的不是记录，而是记录之间的空隙。**

---

## 7.7 Next-Key Lock

InnoDB 默认使用的锁。

```text
Next-Key Lock = Record Lock + Gap Lock
```

即：**锁记录 + 锁前面的间隙**。

锁定 id=5 → 实际锁 `(3,5]`。

```text
3 -------- 5
↑         ↑
Gap      Record
```

目的：**防止幻读**。

---

## 7.8 锁等待（Lock Wait）

事务 A 锁住 id=1 未提交，事务 B 也想更新 id=1 → 等待（Lock Wait）。

可以理解为**排队**。

---

## 7.9 死锁（Deadlock）

数据库最经典问题。

```text
事务A：锁住 id=1，等待 id=2
事务B：锁住 id=2，等待 id=1
```

```text
A ──► B
▲     │
│     ▼
└─────┘
```

循环等待 → 谁也走不出来 → **Deadlock（死锁）**。

### MySQL 如何发现死锁？

InnoDB 维护 **Wait For Graph（等待图）**。发现环路 → 判定 Deadlock → 自动回滚一个事务 → 释放资源。

### 为什么死锁不可避免？

只要满足：**互斥 + 持有并等待 + 不可抢占 + 循环等待**，死锁就可能发生。这是操作系统理论中的经典结论。MySQL 无法彻底避免，只能**检测 + 回滚**。

---

## 7.10 并发控制完整体系

```text
并发控制
│
├── MVCC
│   ├── Undo Log
│   ├── Version Chain
│   └── Read View
│
└── Lock
    ├── Record Lock
    ├── Gap Lock
    ├── Next-Key Lock
    ├── Lock Wait
    └── Deadlock
```

核心结论：

- **MVCC 负责**：读写不阻塞
- **锁负责**：写写不冲突

两者共同构成 InnoDB 的高并发架构。

---

> 上一章：[第六章：MVCC](../ch06-mvcc/README.md)
> 下一章：[第八章：EXPLAIN](../ch08-explain/README.md)
