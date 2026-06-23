# MySQL 第一性原理 —— 目录索引

---

## 第一章：数据库世界观 → [ch01-worldview/README.md](ch01-worldview/README.md)

- 1.1 为什么会有 MySQL？
- 1.2 数据库是什么？
- 1.3 MySQL 内部架构
- 1.4 表（Table）是什么？
- 1.5 主键是什么？
- 1.6 索引是什么？
- 1.7 B+树是什么？
- 1.8 事务是什么？
- 1.9 MVCC 是什么？
- 1.10 学习路线图 & 一页纸地图

---

## 第二章：从数据结构到 B+树 → [ch02-data-structure/README.md](ch02-data-structure/README.md)

- 2.1 数组与链表
- 2.2 二分查找 & 二叉搜索树（BST）
- 2.3 为什么 BST 还不够？ → 平衡树
- 2.4 为什么还要 B 树？ → 磁盘 IO 瓶颈
- 2.5 B 树出现
- 2.6 为什么是 B+Tree？ → 范围查询 & 链表
- 2.7 索引出现
- 2.8 为什么事务出现？
- 2.9 为什么 MVCC 出现？
- 2.10 为什么还需要锁？
- 2.11 执行计划（EXPLAIN）
- 2.12 InnoDB 源码思想

---

## 第三章：存储引擎 → [ch03-storage-engine/README.md](ch03-storage-engine/README.md)

- 3.1 从磁盘开始
- 3.2 为什么磁盘慢？
- 3.3 Page（页）—— 数据库最小读写单位
- 3.4 为什么需要 Buffer Pool？
- 3.5 Page 为什么是 16KB？
- 3.6 Page 内部结构
- 3.7 Page 与 B+Tree 的关系
- 3.8 聚簇索引出现
- 3.9 二级索引 & 回表

---

## 第四章：索引体系 → [ch04-index/README.md](ch04-index/README.md)

- 4.1 索引到底是什么？
- 4.2 聚簇索引
- 4.3 二级索引
- 4.4 回表
- 4.5 覆盖索引
- 4.6 联合索引
- 4.7 最左匹配原则
- 4.8 索引失效场景
- 4.9 索引设计原则

---

## 第五章：事务与 ACID → [ch05-transaction/README.md](ch05-transaction/README.md)

- 5.1 为什么需要事务？
- 5.2 事务是什么？（状态机模型）
- 5.3 ACID 逐一推导
- 5.4 缓冲池带来的风险
- 5.5 Redo Log（WAL 思想）
- 5.6 崩溃恢复（Crash Recovery）
- 5.7 Undo Log
- 5.8 Redo 与 Undo 的关系
- 5.9 事务的真实结构

---

## 第六章：MVCC → [ch06-mvcc/README.md](ch06-mvcc/README.md)

- 6.1 MVCC 到底在解决什么？
- 6.2 版本化思想
- 6.3 MVCC 本质
- 6.4 Undo Log 的新身份
- 6.5 隐藏字段（trx_id / roll_pointer）
- 6.6 Read View
- 6.7 为什么 Repeatable Read 成立？
- 6.8 隔离级别
- 6.9 MVCC 为什么这么强？
- 6.10 MVCC 的边界

---

## 第七章：锁 → [ch07-lock/README.md](ch07-lock/README.md)

- 7.1 为什么 MVCC 还不够？
- 7.2 锁的第一性原理
- 7.3 行锁（Record Lock）
- 7.4 为什么会出现幻读？
- 7.5 间隙锁（Gap Lock）
- 7.6 Next-Key Lock
- 7.7 锁等待（Lock Wait）
- 7.8 死锁（Deadlock）
- 7.9 MySQL 如何发现死锁？
- 7.10 并发控制完整体系

---

## 第八章：EXPLAIN → [ch08-explain/README.md](ch08-explain/README.md)

- 8.1 EXPLAIN 到底是什么？
- 8.2 优化器为什么存在？
- 8.3 一条 SQL 的生命周期
- 8.4 type（访问方式）
- 8.5 key（实际使用的索引）
- 8.6 rows（预估扫描行数）
- 8.7 Extra（关键附加信息）
- 8.8 为什么建了索引还是慢？
- 8.9 执行计划背后的本质（CBO）

---

## 第九章：InnoDB 内核 → [ch09-innodb/README.md](ch09-innodb/README.md)

- 9.1 重新定义 InnoDB
- 9.2 为什么需要 Page？
- 9.3 Page 为什么重要？
- 9.4 Extent（区）
- 9.5 Segment（段）
- 9.6 Buffer Pool
- 9.7 脏页（Dirty Page）
- 9.8 Flush（刷盘）
- 9.9 Redo Log & WAL
- 9.10 Checkpoint
- 9.11 Crash Recovery
- 9.12 Purge Thread
- 9.13 InnoDB 完整流程图

---

## 附录

- [思维导图](appendix/mind-maps/)
- [配图/示意图](appendix/diagrams/)
- [练习与自测](appendix/exercises/)
- [架构演化：库表容量增长与架构决策](appendix/architecture-evolution.md)
