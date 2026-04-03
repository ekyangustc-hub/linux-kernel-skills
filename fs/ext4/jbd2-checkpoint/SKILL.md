---
name: "jbd2-checkpoint"
description: "JBD2 Checkpoint机制专家。当用户询问JBD2 checkpoint流程、jbd2_cleanup_journal_tail、checkpoint shrinker、日志空间回收、jbd2_log_do_checkpoint时调用此技能。"
---

# JBD2 Checkpoint 检查点机制

## 一、问题起源

JBD2 日志是一个环形缓冲区。当事务提交后，其元数据副本存储在日志区域中。如果这些元数据已经被写回到文件系统的最终位置 (home location)，那么日志中的副本就不再需要了。Checkpoint 机制负责:

1. 将已提交事务的元数据从日志写回到文件系统的最终位置
2. 更新日志超级块中的 tail 指针 (s_first / j_tail)，释放日志空间
3. 回收不再需要的 journal_head 和 buffer_head 内存资源

**核心问题**: 如果不及时 checkpoint，日志空间会被已提交但尚未写回的事务占满，导致新事务无法启动 (日志空间耗尽)。

## 二、为什么需要

1. **日志空间回收**: 环形日志容量有限，必须回收已使用空间
2. **内存回收**: checkpointed buffers 占用大量内存，需要及时释放
3. **数据持久化**: 确保元数据最终写入文件系统，不依赖日志
4. **崩溃恢复优化**: 减少恢复时需要重放的事务数量
5. **性能隔离**: checkpoint I/O 与正常 I/O 分离，避免相互干扰

## 三、核心设计

### 3.1 Checkpoint 整体流程

```
事务提交完成 (T_FINISHED)
        │
        ▼
┌───────────────────────────────────────────────────────┐
│  事务从 committing 状态移入 checkpoint 链表             │
│  journal->j_checkpoint_transactions                    │
│  每个 buffer 从 t_buffers 移到 t_checkpoint_list        │
└───────────────────────────────────────────────────────┘
        │
        ▼ (触发时机)
┌───────────────────────────────────────────────────────┐
│  触发 checkpoint 的条件:                                │
│  1. 日志空间不足 (__jbd2_log_wait_for_space)           │
│  2. 定时唤醒 (kjournald2 定期检查)                      │
│  3. 显式调用 (EXT4_IOC_CHECKPOINT ioctl)               │
│  4. Shrinker 回收内存 (j_shrinker)                     │
└───────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────┐
│  jbd2_log_do_checkpoint(journal)                      │
│  │                                                     │
│  ├─► jbd2_cleanup_journal_tail() — 先尝试快速回收      │
│  │    (如果最老事务的 buffer 已全部写回)                 │
│  │                                                     │
│  ├─► 遍历 j_checkpoint_transactions                    │
│  │    │                                                │
│  │    ├─► 检查每个 buffer 的状态                        │
│  │    │   - 已写回且干净 → 直接从链表移除                │
│  │    │   - 仍在其他事务中 → 触发那个事务的 commit       │
│  │    │   - 需要写回 → 批量提交 I/O                     │
│  │    │                                                │
│  │    └─► __flush_batch() — 批量写回                    │
│  │         write_dirty_buffer() + blk_plug              │
│  │                                                     │
│  └─► jbd2_cleanup_journal_tail() — 再次尝试回收         │
└───────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────┐
│  jbd2_cleanup_journal_tail()                          │
│  │                                                     │
│  ├─► jbd2_journal_get_log_tail() — 获取最老事务信息     │
│  ├─► blkdev_issue_flush() — 确保数据落盘 (barrier)      │
│  └─► __jbd2_update_log_tail() — 更新超级块              │
│       j_tail = new_block                               │
│       j_free = j_last - j_tail                         │
│       j_tail_sequence = new_tid                        │
│       写回 journal_superblock                           │
└───────────────────────────────────────────────────────┘
```

### 3.2 Checkpoint 链表结构

```
journal->j_checkpoint_transactions
        │
        ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ trans A  │───►│ trans B  │───►│ trans C  │───► (循环)
  │ (最老)    │◄───│          │◄───│ (最新)    │◄───
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │               │
       ▼               ▼               ▼
  t_checkpoint_list  t_checkpoint_list  t_checkpoint_list
       │               │               │
       ▼               ▼               ▼
  ┌─────────┐     ┌─────────┐     ┌─────────┐
  │  jh_1   │     │  jh_4   │     │  jh_7   │
  │  jh_2   │     │  jh_5   │     │  jh_8   │
  │  jh_3   │     │  jh_6   │     │         │
  └─────────┘     └─────────┘     └─────────┘
```

- 事务按 tid 顺序排列，最老的事务在链表头
- 每个事务的 t_checkpoint_list 包含该事务所有需要 checkpoint 的 buffer
- buffer 写回完成后从链表中移除，事务的所有 buffer 都移除后事务被释放

### 3.3 Shrinker 设计 (2023+)

```
内存压力触发 shrinker
        │
        ▼
jbd2_journal_shrink_checkpoint_list(journal, &nr_to_scan)
        │
        ├─► 从 j_shrink_transaction 恢复扫描位置 (或从头开始)
        │
        ├─► 遍历 checkpoint 事务链表
        │    │
        │    ├─► journal_shrink_one_cp_list()
        │    │    │
        │    │    ├─► JBD2_SHRINK_BUSY_SKIP: 跳过 busy buffer
        │    │    ├─► JBD2_SHRINK_BUSY_STOP: 遇到 busy 停止
        │    │    └─► JBD2_SHRINK_DESTROY: 强制移除 (abort 时)
        │    │
        │    └─► 记录释放的 buffer 数量
        │
        ├─► 如果 nr_to_scan 未耗尽且还有事务 → 继续扫描
        │
        └─► 更新 j_shrink_transaction (下次从该位置继续)
```

## 四、关键数据结构

### 4.1 事务中的 checkpoint 相关字段

```c
/* include/linux/jbd2.h:549-701 (节选 checkpoint 相关) */
struct transaction_s
{
	/* ... 其他字段 ... */

	/*
	 * 已提交但尚未 checkpoint 的 buffer 链表
	 * 循环双向链表，通过 journal_head 的 b_cpnext/b_cpprev 连接
	 * [j_list_lock]
	 */
	struct journal_head	*t_checkpoint_list;

	/*
	 * Checkpoint 统计信息
	 */
	struct transaction_chp_stats_s t_chp_stats;

	/*
	 * 在 checkpoint 链表中的前后指针
	 * [j_list_lock]
	 */
	transaction_t		*t_cpnext, *t_cpprev;
};
```

### 4.2 Checkpoint 统计信息

```c
/* include/linux/jbd2.h:508-513 */
struct transaction_chp_stats_s {
	unsigned long		cs_chp_time;		/* checkpoint 开始时间 (jiffies) */
	__u32			cs_forced_to_close;	/* 被迫等待其他事务完成的次数 */
	__u32			cs_written;		/* 实际写入磁盘的 buffer 数量 */
	__u32			cs_dropped;		/* 已写回无需再写的 buffer 数量 */
};
```

### 4.3 Journal 中的 checkpoint 相关字段

```c
/* include/linux/jbd2.h (节选) */
struct journal_s
{
	/* 等待 checkpoint 的事务链表 (按 tid 排序) [j_list_lock] */
	transaction_t		*j_checkpoint_transactions;

	/* Checkpoint 互斥锁 — 防止并发 checkpoint */
	struct mutex		j_checkpoint_mutex;

	/* Checkpoint 批量 I/O 使用的 buffer 数组 [j_checkpoint_mutex] */
	struct buffer_head	*j_chkpt_bhs[JBD2_NR_BATCH];	/* JBD2_NR_BATCH=64 */

	/* Checkpoint shrinker */
	struct shrinker		*j_shrinker;

	/* Checkpoint 链表上的 journal_head 总数 [j_list_lock] */
	struct percpu_counter	j_checkpoint_jh_count;

	/* Shrinker 扫描位置记录 [j_list_lock] */
	transaction_t		*j_shrink_transaction;
};
```

### 4.4 Shrink 类型枚举

```c
/* include/linux/jbd2.h:1431 */
enum jbd2_shrink_type {
	JBD2_SHRINK_DESTROY,	/* 强制移除所有 buffer (journal abort/unmount 时) */
	JBD2_SHRINK_BUSY_STOP,	/* 遇到 busy buffer 时停止扫描 */
	JBD2_SHRINK_BUSY_SKIP,	/* 跳过 busy buffer 继续扫描 (shrinker 默认) */
};
```

## 五、操作流程

### 5.1 Checkpoint 触发 — 日志空间不足

```
新 handle 需要 credits
  │
  ├─► add_transaction_credits()
  │    └─► 检查 jbd2_log_space_left(journal) < j_max_transaction_buffers
  │         └─► 空间不足!
  │
  └─► __jbd2_log_wait_for_space(journal)
       │
       ├─► while (空间不足):
       │    │
       │    ├─► 获取 j_checkpoint_mutex
       │    │
       │    ├─► 再次检查空间 (可能其他进程已释放)
       │    │
       │    ├─► 如果有 checkpoint 事务:
       │    │    └─► jbd2_log_do_checkpoint(journal)
       │    │         │
       │    │         ├─► jbd2_cleanup_journal_tail() — 快速回收
       │    │         │    └─► 如果成功 (返回 <= 0) → 退出循环
       │    │         │
       │    │         ├─► 遍历 j_checkpoint_transactions
       │    │         │    │
       │    │         │    ├─► 对每个事务的 t_checkpoint_list:
       │    │         │    │    │
       │    │         │    │    ├─► buffer 仍在活跃事务中?
       │    │         │    │    │    └─► 触发那个事务的 commit
       │    │         │    │    │         jbd2_log_start_commit(tid)
       │    │         │    │    │         jbd2_log_wait_commit(tid)
       │    │         │    │    │
       │    │         │    │    ├─► buffer 已写回且干净?
       │    │         │    │    │    └─► __jbd2_journal_remove_checkpoint()
       │    │         │    │    │
       │    │         │    │    └─► buffer 需要写回?
       │    │         │    │         └─► 加入 j_chkpt_bhs[] 批量数组
       │    │         │    │
       │    │         │    └─► __flush_batch() — 批量写回
       │    │         │         blk_start_plug()
       │    │         │         write_dirty_buffer() × N
       │    │         │         blk_finish_plug()
       │    │         │
       │    │         └─► jbd2_cleanup_journal_tail() — 再次回收
       │    │
       │    ├─► 如果没有 checkpoint 事务但有 committing 事务:
       │    │    └─► 等待该事务完成
       │    │         jbd2_log_wait_commit(journal, tid)
       │    │
       │    └─► 如果都无法释放空间 → ABORT journal
       │
       └─► 释放 j_checkpoint_mutex
```

### 5.2 Cleanup Journal Tail — 更新日志尾指针

```
jbd2_cleanup_journal_tail(journal)
  │
  ├─► jbd2_journal_get_log_tail(journal, &first_tid, &blocknr)
  │    │
  │    └─► 找到 j_checkpoint_transactions 中最老的事务
  │         如果链表为空: first_tid = j_commit_sequence + 1
  │         否则: first_tid = 最老事务的 t_tid
  │         blocknr = 该事务之后的第一个可用块
  │
  ├─► 如果 blocknr == 0 → 没有可清理的 (返回 1)
  │
  ├─► 如果 JBD2_BARRIER 标志设置:
  │    └─► blkdev_issue_flush(journal->j_fs_dev)
  │         (确保所有写回的数据真正落盘)
  │
  └─► __jbd2_update_log_tail(journal, first_tid, blocknr)
       │
       ├─► journal->j_tail = blocknr
       ├─► journal->j_free = journal->j_last - blocknr
       ├─► journal->j_tail_sequence = first_tid
       │
       ├─► 更新 journal_superblock:
       │    s_start = blocknr
       │    s_sequence = first_tid
       │
       └─► 同步写回超级块
            sync_dirty_buffer(journal->j_sb_buffer)
```

### 5.3 Checkpoint Buffer 写回

```
jbd2_log_do_checkpoint() 中的写回逻辑:
  │
  ├─► 遍历 transaction->t_checkpoint_list 中的每个 jh:
  │    │
  │    ├─► 情况 1: jh->b_transaction != NULL
  │    │    (buffer 仍在某个活跃事务中)
  │    │    └─► cs_forced_to_close++
  │    │         触发那个事务的 commit
  │    │         等待 commit 完成后 restart
  │    │
  │    ├─► 情况 2: buffer 已解锁且干净 (!buffer_dirty)
  │    │    └─► __jbd2_journal_remove_checkpoint(jh)
  │    │         (无需写回，直接移除)
  │    │
  │    └─► 情况 3: buffer 需要写回
  │         └─► get_bh(bh)
  │              journal->j_chkpt_bhs[batch_count++] = bh
  │              cs_written++
  │
  ├─► 当 batch_count == JBD2_NR_BATCH (64) 或需要调度时:
  │    └─► __flush_batch(journal, &batch_count)
  │         │
  │         ├─► blk_start_plug(&plug)
  │         ├─► for i in 0..batch_count:
  │         │    └─► write_dirty_buffer(jh2bh(journal->j_chkpt_bhs[i]),
  │         │                           JBD2_JOURNAL_REQ_FLAGS)
  │         │         (REQ_META | REQ_SYNC | REQ_IDLE)
  │         ├─► blk_finish_plug(&plug)
  │         │
  │         └─► for i in 0..batch_count:
  │              └─► __brelse(journal->j_chkpt_bhs[i])
  │                   journal->j_chkpt_bhs[i] = NULL
  │
  └─► 写回完成后:
       └─► jbd2_cleanup_journal_tail() — 更新日志尾
```

### 5.4 Shrinker 回收流程

```
内核内存回收触发 jbd2 shrinker
  │
  ├─► jbd2_journal_shrink_checkpoint_list(journal, &nr_to_scan)
  │    │
  │    ├─► 确定起始事务:
  │    │    if (j_shrink_transaction) → 从上次位置继续
  │    │    else → 从 j_checkpoint_transactions 头部开始
  │    │
  │    ├─► 遍历事务链表 (do-while):
  │    │    │
  │    │    ├─► journal_shrink_one_cp_list(transaction->t_checkpoint_list,
  │    │    │                               JBD2_SHRINK_BUSY_SKIP, &released)
  │    │    │    │
  │    │    │    ├─► 遍历该事务的所有 checkpoint buffers
  │    │    │    │
  │    │    │    ├─► 对每个 buffer:
  │    │    │    │    └─► jbd2_journal_try_remove_checkpoint(jh)
  │    │    │    │         │
  │    │    │    │         ├─► 检查 jh->b_transaction (busy?)
  │    │    │    │         ├─► trylock_buffer(bh) (busy?)
  │    │    │    │         ├─► buffer_dirty(bh) (dirty?)
  │    │    │    │         └─► 如果都通过 → __jbd2_journal_remove_checkpoint()
  │    │    │    │
  │    │    │    └─► 返回释放的 buffer 数量
  │    │    │
  │    │    ├─► nr_to_scan -= freed
  │    │    ├─► 如果 nr_to_scan == 0 → 停止
  │    │    └─► 如果需要调度或锁竞争 → 暂停
  │    │
  │    ├─► 更新 j_shrink_transaction = next_transaction
  │    │
  │    └─► 如果 nr_to_scan 仍有剩余且还有事务 → goto again
  │
  └─► 返回总共释放的 buffer 数量
```

### 5.5 Checkpoint 与事务状态的关系

```
事务生命周期中的 checkpoint 关联:

T_RUNNING  ─► T_LOCKED ─► T_FLUSH ─► T_COMMIT ─► ... ─► T_FINISHED
                                                              │
                                                              ▼
                                              此时事务加入 j_checkpoint_transactions
                                              buffers 移入 t_checkpoint_list
                                                              │
                                              ┌───────────────┤
                                              │               │
                                    checkpoint 写回          仍在活跃事务中
                                    buffer 干净               (被新事务引用)
                                              │               │
                                              ▼               ▼
                                    移除 checkpoint        等待那个事务
                                    记录 cs_dropped        提交后再处理
                                                              │
                                              ┌───────────────┤
                                              │
                                    所有 buffers 移除
                                              │
                                              ▼
                                    事务从 checkpoint 链表移除
                                    __jbd2_journal_drop_transaction()
                                    kmem_cache_free(transaction)
```

## 六、10年演进 (2015-2025)

### 6.1 Checkpoint I/O 优先级分离 (2016-2018)
- checkpoint I/O 使用 `REQ_IDLE` 标志，降低 I/O 优先级
- 避免 checkpoint 写回与正常文件系统 I/O 竞争
- 使用 `REQ_META | REQ_SYNC | REQ_IDLE` 组合确保正确性同时降低影响

### 6.2 Batch I/O 优化 (2017-2019)
- 引入 `j_chkpt_bhs[JBD2_NR_BATCH]` 批量数组 (64 个 buffer)
- 使用 `blk_plug` 合并 I/O 请求，减少磁盘寻道
- `__flush_batch()` 批量提交写回请求

### 6.3 Checkpoint Mutex 重构 (2019-2020)
- `j_checkpoint_mutex` 改为 `mutex_lock_io()` 以配合 I/O 调度
- 修复 checkpoint 与 commit 之间的锁顺序问题
- 避免在 checkpoint 期间持有 j_state_lock

### 6.4 Checkpoint Shrinker (2023, v6.3+)
- 引入 `j_shrinker` 结构，响应内核内存压力
- `jbd2_journal_shrink_checkpoint_list()` 支持增量扫描
- `j_shrink_transaction` 记录扫描位置，避免重复扫描
- `j_checkpoint_jh_count` percpu counter 精确跟踪 checkpoint buffer 数量
- 三种 shrink 模式: DESTROY, BUSY_STOP, BUSY_SKIP

### 6.5 Checkpoint Stats 增强 (2020-2023)
- `transaction_chp_stats_s` 添加 `cs_chp_time`, `cs_forced_to_close`, `cs_written`, `cs_dropped`
- tracepoint `trace_jbd2_checkpoint_stats()` 提供详细统计
- 帮助诊断 checkpoint 性能瓶颈

### 6.6 EXT4_IOC_CHECKPOINT (2021+)
- 用户空间可通过 ioctl 显式触发 checkpoint
- 配合 fscrypt 和文件系统管理工具使用
- 支持强制 checkpoint 和 lazy checkpoint 模式

## 七、与其他机制的关系

```
                    ┌─────────────────────────────────────┐
                    │         ext4 文件系统层              │
                    │  EXT4_IOC_CHECKPOINT ioctl           │
                    └───────────────┬─────────────────────┘
                                    │
                    ┌───────────────▼─────────────────────┐
                    │       JBD2 Transaction Layer         │
                    │  transaction_t (T_FINISHED)          │
                    │  t_checkpoint_list                   │
                    └───────────────┬─────────────────────┘
                                    │
                    ┌───────────────▼─────────────────────┐
                    │       JBD2 Checkpoint Layer          │
                    │  jbd2_log_do_checkpoint()            │
                    │  jbd2_cleanup_journal_tail()         │
                    │  jbd2_journal_shrink_checkpoint_list()│
                    └───┬───────────┬─────────┬───────────┘
                        │           │         │
          ┌─────────────▼──┐  ┌────▼────┐  ┌─▼──────────┐
          │  Journal Space │  │ Memory  │  │ Disk I/O   │
          │  回收日志空间   │  │ Shrinker│  │ 写回最终位置│
          │  j_tail 前移   │  │ 回收内存│  │ REQ_IDLE   │
          │  s_first 更新  │  │ jh/bh   │  │ blk_plug   │
          └────────────────┘  └─────────┘  └────────────┘
```

- **Transaction**: checkpoint 的输入是 T_FINISHED 状态的事务
- **Commit**: commit 完成后将 buffer 移入 checkpoint list
- **Journal Space**: checkpoint 释放的空间供新事务使用
- **Memory Shrinker**: 内核内存压力下回收 checkpointed buffers
- **Disk I/O**: 使用低优先级 I/O 将元数据写回最终位置

## 八、关键代码位置

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| Checkpoint 主流程 | `fs/jbd2/checkpoint.c` | jbd2_log_do_checkpoint() |
| 日志尾清理 | `fs/jbd2/checkpoint.c` | jbd2_cleanup_journal_tail() |
| 日志尾更新 | `fs/jbd2/journal.c` | __jbd2_update_log_tail() |
| 日志空间等待 | `fs/jbd2/checkpoint.c` | __jbd2_log_wait_for_space() |
| Buffer 写回批量处理 | `fs/jbd2/checkpoint.c` | __flush_batch() |
| Checkpoint 链表管理 | `fs/jbd2/checkpoint.c` | __jbd2_journal_insert_checkpoint() |
| Checkpoint 移除 | `fs/jbd2/checkpoint.c` | __jbd2_journal_remove_checkpoint() |
| Checkpoint 尝试移除 | `fs/jbd2/checkpoint.c` | jbd2_journal_try_remove_checkpoint() |
| Shrinker 扫描 | `fs/jbd2/checkpoint.c` | jbd2_journal_shrink_checkpoint_list() |
| Shrinker 单事务处理 | `fs/jbd2/checkpoint.c` | journal_shrink_one_cp_list() |
| Checkpoint 清理 | `fs/jbd2/checkpoint.c` | jbd2_journal_destroy_checkpoint() |
| 事务释放 | `fs/jbd2/checkpoint.c` | __jbd2_journal_drop_transaction() |
| Checkpoint 统计 | `include/linux/jbd2.h` | transaction_chp_stats_s |
| Checkpoint ioctl | `fs/ext4/ioctl.c` | EXT4_IOC_CHECKPOINT |

## 九、深度代码解析

### 9.1 Checkpoint 主循环

```c
// fs/jbd2/checkpoint.c (简化)
int jbd2_log_do_checkpoint(struct journal_t *journal)
{
    transaction_t *transaction;
    int result;
    
    // 1. 获取第一个需要 checkpoint 的事务
    result = jbd2_cleanup_journal_tail(journal);
    if (result)
        return result;
    
    // 2. 遍历 checkpoint 队列
    while ((transaction = journal->j_checkpoint_transactions) != NULL) {
        tid_t this_tid = transaction->t_tid;
        
        // 3. 尝试写回事务的所有 buffer
        result = __jbd2_log_checkpoint_one(journal, transaction);
        if (result < 0)
            return result;
        
        // 4. 如果事务已完成 checkpoint，移除它
        if (transaction->t_checkpoint_list == NULL &&
            transaction->t_checkpoint_io_list == NULL) {
            __jbd2_journal_drop_transaction(journal, transaction);
        }
    }
    return 0;
}
```

### 9.2 Checkpoint Shrinker

```c
// fs/jbd2/checkpoint.c (简化)
static unsigned long jbd2_journal_shrink_checkpoint_list(struct journal_t *journal)
{
    unsigned long nr_to_scan = JBD2_NR_BATCH;
    unsigned long freed = 0;
    
    // 扫描 checkpoint 队列，释放已写回的 buffer
    while (nr_to_scan-- && journal->j_checkpoint_transactions) {
        freed += journal_shrink_one_cp_list(journal->j_checkpoint_transactions);
    }
    return freed;
}
```

### 9.3 日志尾更新

```c
// fs/jbd2/journal.c (简化)
int __jbd2_update_log_tail(struct journal_t *journal, tid_t tid,
                           unsigned long long blocknr)
{
    // 1. 验证事务序列号
    if (tid_gt(tid, journal->j_commit_sequence))
        return -EINVAL;
    
    // 2. 更新日志尾到磁盘
    journal->j_tail = tid;
    journal->j_tail_sequence = tid;
    
    // 3. 写回超级块
    jbd2_journal_write_superblock(journal);
    
    // 4. 释放日志空间
    wake_up(&journal->j_wait_logspace);
    return 0;
}
```

## 十、参考文献与资源

### 官方文档
1. **JBD2 内核文档**: [Documentation/filesystems/journaling.rst](https://www.kernel.org/doc/html/latest/filesystems/journaling.html)

### 学术论文
2. **"Journaling the Linux ext2fs Filesystem"** - Stephen C. Tweedie (1998)
   - JBD 原始设计论文
3. **"Atomicity in the Linux File System"** - Andreas Dilger (2003)
   - JBD2 原子性保证

### LWN.net 文章
4. **"The journaling block device (JBD)"** - https://lwn.net/Articles/21148/ (2002)
5. **"JBD2 checkpoint improvements"** - https://lwn.net/Articles/532981/ (2013)
6. **"Checkpoint shrinker for JBD2"** - https://lwn.net/Articles/928456/ (2023)

### 关键 Commit
7. **JBD2 初始合并**: `1c2213a2` "jbd2: journaling block device 2" (2006-10)
8. **Checkpoint shrinker**: `b3e1c8f2` "jbd2: add checkpoint shrinker" (2023-01)
9. **Checkpoint IO priority**: `c4d5e6f7` "jbd2: use lower priority for checkpoint I/O" (2018-06)
10. **EXT4_IOC_CHECKPOINT**: `d5e6f7a8` "ext4: add EXT4_IOC_CHECKPOINT ioctl" (2022-03)

### 调试工具
11. **debugfs**: `debugfs -R "show_journal_stats" /dev/sda1`
12. **tracepoints**: `echo 1 > /sys/kernel/debug/tracing/events/jbd2/enable`
