# 附录：思维导图

> 使用 [Mermaid](https://mermaid.js.org/) 语法制作，支持 GitHub 原生渲染。

---

## 1. MySQL 整体架构图

```mermaid
mindmap
  root((MySQL 架构))
    SQL 层
      Parser 解析器
        词法分析
        语法分析
        生成 AST
      Optimizer 优化器
        查询重写
        索引选择
        JOIN 顺序
        代价估算
      Executor 执行器
        调用存储引擎接口
        返回结果集
    存储引擎层
      InnoDB 默认
        事务支持
        行级锁
        MVCC
      MyISAM 历史
        表级锁
        不支持事务
        FULLTEXT 索引
      Memory 内存表
        数据全在内存
        HASH 索引
    文件系统
      Redo Log 文件
      Undo Log 文件
      数据文件 .ibd
      Binlog 文件
```

---

## 2. 数据结构到 B+Tree 推导链路

```mermaid
mindmap
  root((推导链路))
    数组
      查找快 O1
      插入慢 On
      删除慢 On
      内存连续
    链表
      插入快 O1
      删除快 O1
      查找慢 On
      内存不连续
    二分查找
      OlogN
      要求数据有序
      只适合静态数据
    二叉搜索树 BST
      左小右大
      查找 OlogN
      退化问题
        插入1,2,3,4→变成链表
    平衡树 AVL 红黑树
      自动平衡
      保证 OlogN
      旋转代价高
    B 树
      节点存多个 key
      降低树高度
      减少磁盘 IO
      磁盘 IO 瓶颈
        CPU nanosecond
        Disk millisecond
        差距约10万倍
    B+Tree
      叶子节点形成链表
      范围查询极快
      非叶子只存 key
      叶子存完整数据或指针
```

---

## 3. 存储引擎核心：磁盘 → Page → Buffer Pool

```mermaid
mindmap
  root((存储引擎))
    磁盘基础
      机械硬盘 HDD
        寻道时间 ~10ms
        转速延迟
      固态硬盘 SSD
        随机读 ~0.1ms
        仍然比内存慢1000倍
      设计第一原则
        减少磁盘访问次数
    Page 页
      最小读写单位
      16KB 为什么
        太小 树高增加
        太大 内存浪费
        平衡点
      内部结构
        File Header
        Page Header
        Infimum Supremum
        User Records
        Free Space
        Page Directory
        File Trailer
    Buffer Pool
      本质 空间换时间
      工作流程
        命中 直接返回
        未命中 读磁盘
        修改 标记脏页
      LRU 链表
        热端 冷端
        防止全表扫描污染
      脏页 Flush
        时机 每10秒
        Checkpoint 触发
        Redo Log 满
    Page 与 B+Tree
      一个节点 一个Page
      10亿数据 约3次IO
      根节点常驻Buffer Pool
```

---

## 4. 索引体系全景

```mermaid
mindmap
  root((索引体系))
    本质
      空间换时间
      用额外存储换更少IO
    聚簇索引
      表本身就是 B+Tree
      叶子存完整行数据
      必须有主键
        否则生成rowid
      物理有序
    二级索引
      "叶子存索引列和主键值"
      为什么不存完整数据
        避免空间爆炸
        保持数据一致性
    回表
      二级索引定位
      聚簇索引取数据
      两次 B+Tree 查找
      explain Extra:Using index condition
    覆盖索引
      索引包含所有所需列
      不回表
      性能极高
      explain Extra:Using index
    联合索引
      多列排序
      优于独立索引
    最左匹配原则
      按最左列排序
      跳过左边 无法使用
      范围查询 右边失效
    索引失效
      函数运算
        对列使用函数导致索引失效
      隐式类型转换
        字符串列用数字查询
      前缀模糊
        LIKE前置通配符
      负向查询
        NOT IN NOT EXISTS
      联合索引不满足最左匹配
    设计原则
      服务查询 记住代价
      等值条件列放前面
      范围列放后面
```

---

## 5. 事务 & Redo/Undo 关系

```mermaid
mindmap
  root((事务))
    为什么需要事务
      一条SQL不是原子的
        找记录
        读Page
        改内存
        写日志
        写磁盘
      转账案例
        A-100 B加100
        必须全部成功或全部失败
    事务状态机
      BEGIN
      Running
      Commit
      Rollback
    ACID
      A 原子性
        Undo Log 实现
      C 一致性
        AID 共同保证
      I 隔离性
        MVCC加Lock实现
      D 持久性
        Redo Log 实现
    Buffer Pool 风险
      内存已改
      磁盘未改
      断电数据丢失
    Redo Log
      本质 WAL
      Write Ahead Logging
      为什么快
        顺序写 非随机写
      崩溃恢复
        Checkpoint
        前滚
      Redo Log 两阶段
        Prepare
        Commit
    Undo Log
      记录旧值
      Rollback 回滚
      MVCC 版本链
    两者关系
      Redo 保证持久性
      Undo 保证原子性
      配合实现ACID
```

---

## 6. MVCC 版本链 & ReadView

```mermaid
mindmap
  root((MVCC))
    本质
      多版本并发控制
      读写互不阻塞
      用空间换并发
    隐藏列
      DB_TRX_ID 事务ID
      DB_ROLL_PTR 回滚指针
      DB_ROW_ID 行ID
    版本链
      每次修改产生新版本
      Undo Log 串联历史
      ROLL_PTR 指向旧版本
    ReadView
      m_ids 活跃事务列表
      min_trx_id 最小活跃ID
      max_trx_id 下一个事务ID
      creator_trx_id 当前事务ID
    可见性规则
      trx_id < min_trx_id
        可见 已提交
      "trx_id 大于等于 max_trx_id"
        不可见 未来事务
      trx_id in m_ids
        不可见 活跃未提交
      否则 可见
    隔离级别与ReadView
      READ COMMITTED
        每次 SELECT 生成新 ReadView
        读已提交
      REPEATABLE READ
        事务开始时生成一次 ReadView
        可重复读
        解决幻读
          MVCC加GapLock
    Undo Log 清理
      Purge Thread
      无旧ReadView可见时清除
```

---

## 7. 锁体系

```mermaid
mindmap
  root((锁体系))
    粒度
      表锁
        MyISAM 默认
        并发低
      行锁
        InnoDB 默认
        并发高
      页锁
        少见
    类型
      共享锁 S锁
        "SELECT ... LOCK IN SHARE MODE"
      排他锁 X锁
        UPDATE DELETE INSERT
        "SELECT ... FOR UPDATE"
      意向锁 IS IX
        表级
        表锁前的标记
        快速判断表是否能加锁
    行锁算法
      Record Lock
        锁索引记录
      Gap Lock
        锁间隙
        防止插入幻影行
        RR 级别特有
      Next-Key Lock
        Record加Gap锁
        左开右闭区间
        默认加锁算法
    Insert Intention Lock
      插入意向锁
      信号量 等待插入
      不互斥 Gap Lock
    AUTO-INC Lock
      自增锁
      innodb_autoinc_lock_mode
      0 传统表锁
      1 连续简单插入轻量锁
      2 完全轻量锁
    死锁
      条件
        互斥
        持有并等待
        不可剥夺
        循环等待
      检测
        wait-for graph
        主动回滚代价小的事务
      避免
        固定顺序访问
        缩短事务
        合理使用索引
```

---

## 8. SQL 生命周期 & EXPLAIN

```mermaid
mindmap
  root((SQL 执行))
    客户端
      TCP 连接
      线程分配
    SQL 生命周期
      Parser
        生成 AST
        语法校验
      Optimizer
        查询重写
          常量折叠
          外连接转内连接
        代价估算
          IO 代价
          CPU 代价
        索引选择
          回表代价
          走索引不一定最优
        JOIN 优化
          驱动表选择
          NLJ vs BNl
    EXPLAIN
      id
        SELECT 序号
      select_type
        SIMPLE
        PRIMARY
        SUBQUERY
        DERIVED
      table
        访问的表名
      type 访问类型
        ALL 全表扫描 最差
        index 索引全扫描
        range 索引范围
        ref 非唯一索引
        eq_ref 唯一索引
        const 主键常量
        NULL 不查表
      possible_keys
        候选索引
      key
        实际使用索引
      key_len
        索引使用字节数
      ref
        索引比较的值
      rows
        预估扫描行数
      filtered
        WHERE过滤比例
      Extra
        Using index 覆盖索引
        Using where 过滤
        Using temporary 临时表
        Using filesort 文件排序
    SlowSQL
      发现
        slow_query_log
        long_query_time
      分析
        EXPLAIN 看执行计划
        SHOW PROFILE
        Performance Schema
      优化
        加索引
        改写SQL
        表结构优化
        参数调优
```

---

## 9. InnoDB 内核全景

```mermaid
mindmap
  root((InnoDB 内核))
    Buffer Pool
      默认128M
      数据页 索引页
      插入缓冲
      自适应哈希索引
      Change Buffer
    Redo Log
      WAL 机制
      循环写入
      ib_logfile0 ib_logfile1
      默认 48M×2
      刷盘时机
        Commit
        每1秒
        Redo Log 满百分之75
    Undo Log
      存储旧值
      构成版本链
      支持 MVCC
      Purge 清理
    双写 Doublewrite
      防止页断裂
      16K 写一半宕机
      doublewrite buffer
        先写共享表空间
        再写数据文件
    Checkpoint
      LSN 日志序列号
      脏页刷盘
      崩溃恢复起点
      类型
        Sharp Checkpoint
        Fuzzy Checkpoint
          脏页比例
          日志空间压力
    Change Buffer
      二级索引优化
      缓存非唯一二级索引的修改
      减少随机IO
      合并时机
        读Page时
        后台线程
    Purge Thread
      清理旧版本
      回收 Undo 空间
    Master Thread
      每1秒
        刷新日志
        脏页刷盘
        合并Change Buffer
      每10秒
        脏页刷盘
        回收Undo
    Recovery 崩溃恢复
      Phase 01
        扫描Redo Log
        重建脏页
      Phase 02
        回滚未提交事务
        Undo Log 辅助
```

---

## 10. 领域视角总览

```mermaid
mindmap
  root((MySQL 领域视角))
    00 数据库是什么
      数据持久化
      快速检索
      并发控制
      MySQL架构
      B+树
      事务 ACID
      MVCC
    01 存储篇
      Page 16KB
      Buffer Pool
      B+Tree 结构
      聚簇索引
      二级索引
      回表
    02 事务篇
      ACID 推导
      Redo Log WAL
      Undo Log
      Redo与Undo关系
      事务状态机
    03 并发篇
      MVCC 版本链
      ReadView 可见性
      Record Lock
      Gap Lock
      Next-Key Lock
      死锁
    04 SQL执行篇
      Optimizer
      EXPLAIN
      SlowSQL 发现
      SlowSQL 分析
      SlowSQL 优化
    05 InnoDB内核篇
      Buffer Pool LRU
      Redo Log 循环写
      Doublewrite
      Checkpoint
      Change Buffer
      Recovery
    06 数据库设计哲学
      减少磁盘IO
      空间换时间
      顺序写优于随机写
      分段 分层 缓存
      正确性优先
      简单优于复杂
```

---

## 11. 推导链 vs 领域视角 双翼图

```mermaid
mindmap
  root((MySQL 双视角))
    推导链 纵向
      ch01 数据库世界观
      ch02 数据结构到B+树
      ch03 存储引擎
      ch04 索引体系
      ch05 事务 ACID
      ch06 MVCC
      ch07 锁体系
      ch08 EXPLAIN
      ch09 InnoDB 内核
      特点
        因果递进
        从零到一
        回答 为什么
    领域视角 横向
      domains 00 数据库是什么
      domains 01 存储篇
      domains 02 事务篇
      domains 03 并发篇
      domains 04 SQL执行篇
      domains 05 InnoDB内核篇
      domains 06 设计哲学
      特点
        模块化
        知识地图
        回答 有哪些
```

---

> 💡 以上思维导图均使用 Mermaid `mindmap` 语法，在支持 Mermaid 的 Markdown 渲染器（如 GitHub、Typora、VS Code 插件）中均可正常显示。
