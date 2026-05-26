# PostgreSQL 与 InnoDB 崩溃恢复实现对比（基于 ARIES 算法）

## ARIES 算法与两者的关系

ARIES（Algorithm for Recovery and Isolation Exploiting Semantics）是 IBM 在 1992 年提出的数据库崩溃恢复算法，核心思想：

1. **WAL（Write-Ahead Logging）**：修改数据前必须先写日志
2. **三阶段恢复**：Analysis → Redo → Undo
3. **Redo 重放历史**：不区分事务是否提交，全部重做
4. **Undo 物理回滚**：用 CLR（Compensation Log Record）记录回滚操作，保证回滚本身可恢复
5. **Physiological logging**：page 级定位 + page 内逻辑操作

**PostgreSQL 与 ARIES 的关系**：借用了 WAL-first、physiological logging、redo 幂等性等核心思想，但 UNDO 阶段完全偏离——ARIES 要求物理回滚 + CLR，PG 靠 MVCC 将 undo 简化为标记 abort，没有 CLR 机制。

**InnoDB 与 ARIES 的关系**：最接近经典 ARIES 的工业实现。三阶段恢复、物理 undo、回滚写 redo（类似 CLR 的作用）都保留了。主要差异是用 mini-transaction（mtr）作为原子 redo 单位（ARIES 是单条日志记录），且 undo log 独立存储在专用 tablespace 而非嵌入 WAL 流中。

---

## 1. PostgreSQL 的崩溃恢复实现

### WAL（Write-Ahead Log）基础

- 日志格式：物理逻辑混合（physiological）——以 page 为单位定位，page 内记录逻辑操作
- 日志标识：LSN（Log Sequence Number），单调递增的 64-bit 字节偏移
- 事务开始：**不写 WAL 记录**。活跃事务通过"有数据修改记录但无 COMMIT/ABORT 记录"来推断
- 事务提交：写 `XLOG_XACT_COMMIT` 记录，刷盘（`synchronous_commit=on` 时）

### Phase 1: Analysis（分析阶段）

从最近的 checkpoint 记录开始前向扫描 WAL：

- 读取 checkpoint 记录中的 `nextXid`、`redo` 位置
- 确定需要重放的 WAL 范围

PG 的 analysis 相对简单，因为不需要构建复杂的事务表——没有 undo 的需求。

### Phase 2: REDO（重做阶段）

从 checkpoint 的 redo point 开始，前向重放所有 WAL 记录：

```
for each WAL record from redo_point to end:
    page = read_page(record.target_block)
    if page.LSN < record.LSN:
        apply(record, page)   // 幂等操作
        page.LSN = record.LSN
```

关键点：

- **幂等性**：通过比较 page LSN 与 record LSN 判断是否需要重放
- **Full Page Write（FPW）**：checkpoint 后首次修改某 page 时，WAL 中写入完整 page 镜像，解决 torn page 问题
- 重放所有记录，不区分事务是否已提交

### Phase 3: UNDO（撤销阶段）

**这是 PG 与传统 ARIES 最大的区别——几乎无操作：**

```
for each uncommitted xid:
    pg_xact[xid] = TRANSACTION_STATUS_ABORTED  // 写几个 bit
```

为什么这样就够了：

- MVCC 机制下，UPDATE = 插入新 tuple（旧 tuple 保留）
- 未提交事务的 tuple 的 `xmin` 对应的事务状态为 aborted
- 任何后续事务做可见性判断时，发现 `xmin` 对应事务已 abort → tuple 不可见
- 这些"死" tuple 后续由 VACUUM 清理
- **零数据页 I/O，零物理回滚**

### 防 torn page 机制

- **Full Page Write（full_page_writes=on）**
- checkpoint 后首次脏写某 page 时，WAL 中包含完整 8KB page 镜像
- 代价：WAL 体积膨胀（通常 2-3 倍），但保证原子性

---

## 2. MySQL InnoDB 的崩溃恢复实现

### 日志体系（双日志架构）

| 日志 | 用途 | 特征 |
|------|------|------|
| Redo Log | 重做已提交事务 | 物理日志，循环写入固定大小文件 |
| Undo Log | 回滚未提交事务 | 逻辑日志，存储在 undo tablespace |

- Redo Log：`ib_logfile0`, `ib_logfile1`（8.0.30+ 改为 `#innodb_redo/` 目录）
- Undo Log：`undo_001`, `undo_002`（独立 undo tablespace）

### Mini-Transaction（mtr）

InnoDB 的原子 redo 单位不是用户事务，而是 **mini-transaction**：

- 一个 mtr 保证一组页面修改的原子性（如 B+Tree 分裂涉及多个 page）
- mtr 提交时将 redo log 写入 log buffer
- 用户事务由多个 mtr 组成

### Undo Log 结构

```
┌─────────────────────────────────────┐
│         Undo Tablespace             │
├─────────────────────────────────────┤
│  Rollback Segment 1                 │
│    ├── Undo Log (trx_1): INSERT     │
│    │     slot → slot → slot         │
│    ├── Undo Log (trx_2): UPDATE     │
│    │     slot → slot → slot         │
│    └── ...                          │
│  Rollback Segment 2                 │
│    └── ...                          │
│  ... (最多 128 个 rollback segment)  │
└─────────────────────────────────────┘
```

两种 undo log 类型：

- **INSERT undo**：记录插入行的主键，回滚时按主键删除
- **UPDATE undo**：记录修改前的旧值（before image），回滚时恢复旧值；同时支撑 MVCC 读

### Phase 1: Redo（前滚阶段）

从最近的 checkpoint LSN 开始，前向扫描 redo log：

```
scan redo log from checkpoint_lsn to end:
    for each mtr group:
        for each redo record in mtr:
            page = read_page(space_id, page_no)
            if page.LSN < record.LSN:
                apply_redo(record, page)
```

关键点：

- 重放所有 redo，不区分事务是否提交
- 这一步之后，所有 page 恢复到崩溃前的最新状态（包括未提交事务的修改）
- Undo log 本身也是存储在 page 中的，所以 redo 阶段也会恢复 undo page

### Phase 2: Undo（回滚阶段）

Redo 完成后，扫描 undo segment 找到所有活跃事务，执行物理回滚：

```
for each active transaction (state = TRX_ACTIVE):
    while trx has undo records:
        undo_rec = read_next_undo_record(trx)  // 反向遍历

        switch undo_rec.type:
            case INSERT:
                // 按主键删除该行
                delete_row_by_pk(undo_rec.table, undo_rec.pk)

            case UPDATE:
                // 用 before image 恢复旧值
                restore_row(undo_rec.table, undo_rec.pk, undo_rec.old_values)

        // 回滚操作本身也写 redo log（保证回滚的持久性）
        write_redo_for_rollback(...)

    mark_trx_committed(trx)  // 标记事务已回滚完成
```

关键特征：

- **真正的物理回滚**：修改数据页，恢复旧值
- **回滚也写 redo**：如果回滚过程中再次崩溃，重启后继续回滚
- **可能很慢**：大事务的回滚可能需要数小时（经典运维事故）
- **后台执行**：InnoDB 启动后可以接受新连接，undo 在后台线程继续

### 防 torn page 机制

- **Doublewrite Buffer**
- 刷脏页时，先将 page 写入 doublewrite buffer（连续磁盘区域）
- 再写入实际数据文件位置
- 崩溃时若数据文件 page 损坏，从 doublewrite buffer 恢复完整 page
- 代价：写放大 2x（但是顺序写，实际影响有限）

---

## 3. 核心对比

| 维度 | PostgreSQL | InnoDB |
|------|-----------|--------|
| UNDO 代价 | 几乎为零（标记 bit） | 真实物理回滚（可能很慢） |
| UNDO 机制 | MVCC + 可见性判断 | Undo log 物理恢复 |
| 日志架构 | 单一 WAL | Redo + Undo 双日志 |
| 原子写单位 | 单条 WAL record | Mini-transaction（mtr） |
| Torn page 防护 | Full Page Write | Doublewrite Buffer |
| 事务开始标记 | 无（不写 WAL） | 有（分配 trx_id，写 undo header） |
| MVCC 旧版本存储 | Heap 中（旧 tuple 原地保留） | Undo log 中（通过 roll_ptr 链接） |
| 恢复后可用性 | REDO 完即可服务 | REDO 完即可服务（UNDO 后台继续） |
| 大事务崩溃恢复风险 | 无（标记 abort 即可） | 高（回滚可能耗时数小时） |
| 空间回收 | VACUUM（异步，有膨胀问题） | Purge 线程清理 undo log |

### 设计哲学差异

**PostgreSQL**：把复杂度推到运行时（VACUUM），换取恢复时的简单和快速。代价是表膨胀（bloat）和 VACUUM 调优负担。

**InnoDB**：把复杂度放在恢复时（物理回滚），运行时通过 undo log 集中管理旧版本，数据文件保持紧凑。代价是大事务回滚风险和 undo tablespace 管理。

---

## 4. 其他实现差异简述

### Checkpoint 机制

- **PG**：Checkpoint 进程定期将所有脏页刷盘，写 checkpoint WAL 记录。期间 I/O 可能有尖峰（`checkpoint_completion_target` 控制平滑度）
- **InnoDB**：Fuzzy checkpoint，持续异步刷脏页（page cleaner 线程），checkpoint LSN 推进不需要一次性刷完所有脏页。通过 adaptive flushing 根据 redo log 空间压力动态调整刷盘速率

### 并发控制粒度

- **PG**：Tuple 级 MVCC，无间隙锁（Serializable 级别用 SSI——Serializable Snapshot Isolation）
- **InnoDB**：行锁 + 间隙锁（Next-Key Lock），REPEATABLE READ 下通过间隙锁防幻读

### Buffer Pool 管理

- **PG**：Shared buffers，clock-sweep 淘汰算法（近似 LRU）
- **InnoDB**：Buffer pool，young/old 分区 LRU（解决全表扫描污染问题）

### 日志空间管理

- **PG**：WAL segment 文件（默认 16MB/个），写满创建新文件，旧文件归档或回收。理论上无空间上限
- **InnoDB**：Redo log 循环使用固定空间（8.0.30 前），空间用尽时强制 checkpoint。8.0.30+ 支持动态调整大小
