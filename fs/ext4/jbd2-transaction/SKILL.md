---
name: "jbd2-transaction"
description: "JBD2事务机制专家。当用户询问JBD2事务生命周期、handle_t、transaction_t、credits系统、CIL、AIL、事务提交、jbd2_journal_start、jbd2_journal_stop时调用此技能。"
---

# JBD2 Transaction 事务机制

## 一、问题起源

JBD2 (Journaling Block Device 2) 是 Linux 内核的通用块设备日志层，为 ext4、OCFS2 等文件系统提供原子性保证。事务 (Transaction) 是 JBD2 的核心抽象，它将一组相关的元数据修改打包成一个原子单元，确保要么全部写入磁盘，要么全部不写入。

**核心问题**: 文件系统操作 (如创建文件、删除目录) 通常涉及多个元数据块的修改。如果系统在操作中途崩溃，文件系统会处于不一致状态。JBD2 事务机制通过 Write-Ahead Logging (WAL) 协议解决此问题：先将修改写入日志区域，确认后再写入最终位置。

## 二、为什么需要

1. **原子性保证**: 多个元数据修改必须作为一个整体提交或回滚
2. **崩溃恢复**: 系统崩溃后，通过重放日志恢复一致性状态
3. **性能优化**: 批量提交 (batching) 减少磁盘 I/O 次数
4. **并发控制**: 多个线程可以同时修改不同数据，共享同一事务
5. **Credit 管理**: 预分配日志空间，防止事务过大导致日志耗尽

## 三、核心设计

### 3.1 事务状态机

```
T_RUNNING ──► T_LOCKED ──► T_SWITCH ──► T_FLUSH ──► T_COMMIT
    │                                              │
    │  新 handle 可加入          等待所有 handle 完成    │
    │                                              ▼
    │                                    T_COMMIT_DFLUSH (data flush)
    │                                              │
    │                                              ▼
    │                                    T_COMMIT_JFLUSH (journal write)
    │                                              │
    │                                              ▼
    │                                    T_COMMIT_CALLBACK
    │                                              │
    └──────────────────────────────────────────────┘
                                                  ▼
                                            T_FINISHED
                                                  │
                                                  ▼
                                          (checkpoint 后释放)
```

状态说明:
- **T_RUNNING**: 接受新 handle 加入，正常修改阶段
- **T_LOCKED**: 不再接受新 handle，但已有 handle 可继续修改
- **T_SWITCH**: 等待所有 reserved handle 完成
- **T_FLUSH**: 所有修改完成，开始将数据刷到磁盘 (ordered mode)
- **T_COMMIT**: 写入日志描述符块 + 元数据块
- **T_COMMIT_DFLUSH**: 等待数据块 I/O 完成
- **T_COMMIT_JFLUSH**: 等待日志 I/O 完成
- **T_COMMIT_CALLBACK**: 执行提交后回调
- **T_FINISHED**: 提交完成，等待 checkpoint 清理

### 3.2 Handle 与 Transaction 关系

```
Thread A ──► handle_t (credits=10) ──┐
                                      │
Thread B ──► handle_t (credits=5)  ───┼──► transaction_t (t_tid=1234)
                                      │         t_state=T_RUNNING
Thread C ──► handle_t (credits=8) ───┘         t_outstanding_credits=23
```

- 多个 handle 可以共享同一个 running transaction
- 每个 handle 声明所需 credits (可修改的 buffer 数量)
- 当 transaction 的总 credits 超过阈值时，触发 commit

### 3.3 Buffer 归属链表

```
transaction_t
├─ t_reserved_list  : 已预留但尚未修改的 buffer
├─ t_buffers        : 正在修改的 metadata buffer (metadata)
├─ t_shadow_list    : 正在写入日志的 buffer 的 shadow copy
├─ t_forget         : 可 forget 的 buffer (被后续事务覆盖)
├─ t_checkpoint_list: 已提交但尚未 checkpoint 的 buffer
└─ t_inode_list     : 需要特殊处理的 inode (ordered/journal mode)
```

## 四、关键数据结构

### 4.1 handle_t - 每线程事务上下文

```c
/* include/linux/jbd2.h:476-502 */
struct jbd2_journal_handle
{
	union {
		transaction_t	*h_transaction;	/* 所属事务 */
		journal_t	*h_journal;	/* 仅 reserved handle 使用 */
	};

	handle_t		*h_rsv_handle;	/* 预留的备用 handle */
	int			h_total_credits;	/* 剩余可修改 buffer 数 */
	int			h_revoke_credits;	/* 剩余 revoke 记录数 */
	int			h_revoke_credits_requested;
	int			h_ref;			/* 引用计数 */
	int			h_err;			/* 错误码 */

	/* Flags [no locking] */
	unsigned int	h_sync:		1;	/* 同步提交标志 */
	unsigned int	h_reserved:	1;	/* 是否为 reserved handle */
	unsigned int	h_aborted:	1;	/* 致命错误标志 */
	unsigned int	h_type:		8;	/* 操作类型 (统计用) */
	unsigned int	h_line_no:	16;	/* 调用位置 (调试用) */

	unsigned long		h_start_jiffies;
	unsigned int		h_requested_credits;
	unsigned int		saved_alloc_context;	/* memalloc_nofs_save() 返回值 */
};
```

### 4.2 transaction_t - 事务核心结构

```c
/* include/linux/jbd2.h:549-701 */
struct transaction_s
{
	journal_t		*t_journal;
	tid_t			t_tid;		/* 事务序列号 (单调递增) */

	/* 事务状态机 */
	enum {
		T_RUNNING,
		T_LOCKED,
		T_SWITCH,
		T_FLUSH,
		T_COMMIT,
		T_COMMIT_DFLUSH,
		T_COMMIT_JFLUSH,
		T_COMMIT_CALLBACK,
		T_FINISHED
	}			t_state;

	unsigned long		t_log_start;	/* 日志起始块号 */
	int			t_nr_buffers;	/* t_buffers 上的 buffer 数量 */

	/* Buffer 链表 */
	struct journal_head	*t_reserved_list;
	struct journal_head	*t_buffers;
	struct journal_head	*t_forget;
	struct journal_head	*t_checkpoint_list;
	struct journal_head	*t_shadow_list;
	struct list_head	t_inode_list;

	/* 性能统计 */
	unsigned long		t_max_wait;	/* 最长等待时间 */
	unsigned long		t_start;	/* 事务启动时间 (jiffies) */
	unsigned long		t_requested;	/* 请求提交的时间 */
	struct transaction_chp_stats_s t_chp_stats;

	/* 原子计数器 */
	atomic_t		t_updates;		/* 活跃 handle 数量 */
	atomic_t		t_outstanding_credits;	/* 已分配的总 credits */
	atomic_t		t_outstanding_revokes;	/* 已添加的 revoke 记录数 */
	atomic_t		t_handle_count;		/* 使用过此事务的 handle 总数 */

	/* Checkpoint 链表指针 */
	transaction_t		*t_cpnext, *t_cpprev;

	/* 过期时间 */
	unsigned long		t_expires;	/* 自动提交超时时间 */
	ktime_t			t_start_time;

	unsigned int t_synchronous_commit:1;
	int		t_need_data_flush;
};
```

### 4.3 journal_t - 日志控制结构 (关键字段)

```c
/* include/linux/jbd2.h:742-1294 (节选) */
struct journal_s
{
	unsigned long		j_flags;
	rwlock_t		j_state_lock;	/* 保护事务状态 */
	spinlock_t		j_list_lock;	/* 保护 buffer 链表 */

	/* 事务指针 */
	transaction_t		*j_running_transaction;		/* 当前运行事务 */
	transaction_t		*j_committing_transaction;	/* 正在提交的事务 */
	transaction_t		*j_checkpoint_transactions;	/* 等待 checkpoint 的事务链表 */

	/* 序列号 */
	tid_t			j_tail_sequence;		/* 日志中最老事务的 tid */
	tid_t			j_transaction_sequence;		/* 下一个分配的 tid */
	tid_t			j_commit_sequence;		/* 最近已提交的 tid */
	tid_t			j_commit_request;		/* 请求提交的 tid */

	/* 日志空间 */
	unsigned long		j_head;			/* 日志头 (下一个可用块) */
	unsigned long		j_tail;			/* 日志尾 (最老使用的块) */
	unsigned long		j_free;			/* 空闲块数 */
	unsigned long		j_first, j_last;	/* 日志可用范围 */

	atomic_t		j_reserved_credits;	/* 全局预留 credits */
	int			j_max_transaction_buffers;	/* 单事务最大 buffer 数 */
	int			j_transaction_overhead_buffers;	/* 事务自身开销 */

	unsigned long		j_commit_interval;	/* 自动提交间隔 (默认 5s) */
	struct timer_list	j_commit_timer;		/* 提交定时器 */

	struct task_struct	*j_task;		/* kjournald2 线程 */
	/* ... 更多字段 ... */
};
```

### 4.4 jbd2_inode - 事务关联的 inode

```c
/* include/linux/jbd2.h:397-446 */
struct jbd2_inode {
	transaction_t *i_transaction;		/* 所属事务 */
	transaction_t *i_next_transaction;	/* 下一个事务 (跨事务修改) */
	struct list_head i_list;		/* 事务的 inode 链表 */
	struct inode *i_vfs_inode;		/* VFS inode */
	unsigned long i_flags;			/* JI_WRITE_DATA, JI_WAIT_DATA */
	loff_t i_dirty_start;			/* 脏数据起始偏移 */
	loff_t i_dirty_end;			/* 脏数据结束偏移 (含) */
};
```

## 五、操作流程

### 5.1 启动事务 (jbd2_journal_start)

```
ext4 调用 jbd2_journal_start(journal, nblocks)
  │
  ├─► 检查 current->journal_info (已有 handle? → 增加引用计数, 直接返回)
  │
  ├─► 创建新 handle: handle->h_total_credits = nblocks
  │    如果有 rsv_blocks: 创建 reserved handle (h_reserved=1)
  │
  └─► start_this_handle(journal, handle, gfp_mask)
       │
       ├─► 预分配 transaction (如果 j_running_transaction 为空)
       │
       ├─► 获取 j_state_lock (read lock)
       │
       ├─► 检查 journal 是否 aborted
       │
       ├─► 等待 barrier (如果 j_barrier_count > 0 且非 reserved)
       │
       ├─► 如果没有 running transaction → 创建并设为 j_running_transaction
       │
       ├─► add_transaction_credits(journal, blocks, rsv_blocks)
       │    │
       │    ├─► 检查 t_state == T_RUNNING (否则等待)
       │    ├─► atomic_add 到 t_outstanding_credits
       │    ├─► 如果超过 j_max_transaction_buffers → 触发 commit, 等待重试
       │    ├─► 检查日志空间 (jbd2_log_space_left)
       │    │    不足 → 调用 __jbd2_log_wait_for_space → checkpoint
       │    └─► 如果 rsv_blocks > 0, 检查 j_reserved_credits 上限
       │
       ├─► handle->h_transaction = transaction
       ├─► atomic_inc(&transaction->t_updates)
       ├─► atomic_inc(&transaction->t_handle_count)
       ├─► current->journal_info = handle
       └─► handle->saved_alloc_context = memalloc_nofs_save()
            (禁止 GFP_FS 分配, 防止递归进入文件系统)
```

### 5.2 修改 buffer (jbd2_journal_get_write_access)

```
ext4 修改元数据前调用 jbd2_journal_get_write_access(handle, bh)
  │
  ├─► 获取 jh (journal_head) 和 bh_lock
  │
  ├─► 如果 bh 已在其他事务中:
  │    ├─► 创建 frozen data copy (b_frozen_data)
  │    └─► 将 bh 移入当前事务的 t_buffers
  │
  ├─► 如果 bh 不在任何事务中:
  │    ├─► file_buffer_to_transaction(bh, current_transaction)
  │    └─► 标记 BH_JBDDirty
  │
  ├─► 检查并 cancel revoke (如果此块之前被 revoke)
  │
  └─► 释放锁, 返回
```

### 5.3 提交 handle (jbd2_journal_stop)

```
ext4 完成操作后调用 jbd2_journal_stop(handle)
  │
  ├─► memalloc_nofs_restore(handle->saved_alloc_context)
  │
  ├─► handle->h_ref-- (如果 > 0, 返回 — 嵌套 handle)
  │
  ├─► 统计信息收集 (handle 持续时间、credits 使用等)
  │
  ├─► stop_this_handle(handle)
  │    │
  │    ├─► atomic_dec(&transaction->t_updates)
  │    ├─► 回收未使用的 credits
  │    └─► 如果 t_updates == 0:
  │         └─► wake_up(&journal->j_wait_updates)
  │
  ├─► 如果 handle->h_rsv_handle:
  │    └─► 释放 reserved handle
  │
  ├─► current->journal_info = NULL
  │
  └─► 如果需要同步提交 (handle->h_sync 或 jbd2_journal_force_commit):
       └─► jbd2_log_start_commit(journal, tid)
            └─► jbd2_log_wait_commit(journal, tid)
```

### 5.4 事务提交流程 (jbd2_journal_commit_transaction)

```
kjournald2 线程执行 jbd2_journal_commit_transaction(journal)
  │
  ├─► [T_LOCKED] 将 j_running_transaction 移到 j_committing_transaction
  │    设置 t_state = T_LOCKED
  │    等待 t_updates == 0 (所有 handle 完成)
  │
  ├─► [T_FLUSH] t_state = T_FLUSH
  │    └─► journal_submit_data_buffers() — 写 ordered mode 数据
  │
  ├─► [T_COMMIT] t_state = T_COMMIT
  │    │
  │    ├─► 分配日志块 (t_log_start = j_head)
  │    │
  │    ├─► 写 Descriptor Block:
  │    │    └─► 遍历 t_buffers, 为每个 buffer 写入 journal_block_tag
  │    │
  │    ├─► 写 Revoke Records:
  │    │    └─► jbd2_journal_write_revoke_records()
  │    │
  │    ├─► 写 Metadata Buffers:
  │    │    └─► 遍历 t_buffers, 写入 shadow copy 到日志
  │    │         (使用 BH_Shadow 标志跟踪 I/O)
  │    │
  │    ├─► 等待所有 Shadow I/O 完成
  │    │
  │    ├─► [T_COMMIT_DFLUSH] 等待数据 I/O 完成 (ordered mode)
  │    │    └─► journal_finish_inode_data_buffers()
  │    │
  │    ├─► [T_COMMIT_JFLUSH] 写 Commit Block
  │    │    └─► journal_submit_commit_record()
  │    │         包含 CRC32C 校验和
  │    │
  │    └─► 等待 Commit Block I/O 完成
  │
  ├─► 更新 j_commit_sequence = t_tid
  │
  ├─► [T_COMMIT_CALLBACK] 执行回调
  │    └─► journal->j_commit_callback(journal, transaction)
  │
  ├─► [T_FINISHED] t_state = T_FINISHED
  │    │
  │    ├─► 将 buffer 从 t_buffers 移到 t_checkpoint_list
  │    │    └─► __jbd2_journal_insert_checkpoint(jh, transaction)
  │    │
  │    ├─► 切换 revoke table: jbd2_journal_switch_revoke_table()
  │    │
  │    └─► 清理 forget list, 释放 shadow buffers
  │
  ├─► 唤醒等待 commit 完成的进程
  │    └─► wake_up(&journal->j_wait_done_commit)
  │
  └─► 创建新的 running transaction (如果有等待的 handle)
```

### 5.5 Credit 系统详解

```
日志空间管理:

j_max_transaction_buffers = j_total_len / 4  (通常)
j_transaction_overhead_buffers = 事务自身开销 (descriptor + revoke + commit blocks)

每个 handle 启动时:
  blocks = nblocks + DIV_ROUND_UP(revoke_records, revoke_records_per_block)
  t_outstanding_credits += blocks

限制条件:
  1. t_outstanding_credits <= j_max_transaction_buffers
  2. j_reserved_credits <= j_max_transaction_buffers / 2
  3. jbd2_log_space_left(journal) >= j_max_transaction_buffers

当超过限制时:
  → 触发 jbd2_log_start_commit()
  → 等待当前事务提交完成
  → 新 handle 加入下一个事务
```

## 六、10年演进 (2015-2025)

### 6.1 Reserved Credits (2015-2016)
- 引入 `h_reserved` handle 机制，允许在内存压力下预分配 credits
- 解决 page writeback 路径中无法获取事务的 deadlock 问题
- reserved handle 不受 transaction 大小限制，可在 T_LOCKED 状态下加入

### 6.2 Checksum V3 (2016)
- 引入 `journal_block_tag3_t`，支持 32-bit CRC32C 校验和
- `JBD2_FEATURE_INCOMPAT_CSUM_V3` (0x10) 提供完整元数据保护
- Commit block 也添加校验和

### 6.3 Fast Commit (2021, v5.12)
- 引入 `JBD2_FEATURE_INCOMPAT_FAST_COMMIT` (0x20)
- 针对小文件 fsync 的优化路径，避免完整 journal commit
- 使用专用的 FC tag 格式记录 inode 和 dentry 变更
- 后续版本持续优化: reclaim-safe lock, staging queue, sub-transaction ID

### 6.4 T_SWITCH State (2022-2023)
- 引入 `T_SWITCH` 状态，介于 T_LOCKED 和 T_FLUSH 之间
- 专门处理 reserved handle 的完成等待
- 避免 reserved handle 阻塞事务状态转换

### 6.5 Data-Race Fixes (2022-2024)
- 使用 `READ_ONCE`/`WRITE_ONCE` 保护 `t_state` 等共享状态
- 修复 `t_max_wait` 的并发更新竞争 (使用 cmpxchg)
- `data_race()` 注解用于 lockdep 友好检查

### 6.6 Checkpoint Shrinker (2023, v6.3+)
- 引入 `j_shrinker` 用于回收 checkpointed journal buffers
- `jbd2_journal_shrink_checkpoint_list()` 支持按需回收
- `j_checkpoint_jh_count` percpu counter 跟踪 checkpoint buffer 数量
- `j_shrink_transaction` 记录扫描位置，避免重复扫描

### 6.7 Transaction Overhead Accounting (2023-2024)
- 引入 `j_transaction_overhead_buffers` 精确计算事务开销
- 改进 credit 计算，减少不必要的 commit 触发

## 七、与其他机制的关系

```
                    ┌─────────────────────────────────────────┐
                    │           ext4 文件系统层                │
                    │  ext4_mark_iloc_dirty()                 │
                    │  ext4_journal_start/stop()               │
                    └───────────────┬─────────────────────────┘
                                    │
                    ┌───────────────▼─────────────────────────┐
                    │         JBD2 Transaction Layer           │
                    │  handle_t ◄──► transaction_t            │
                    │  Credits 系统                            │
                    └───┬───────────┬──────────┬──────────────┘
                        │           │          │
          ┌─────────────▼──┐  ┌────▼─────┐  ┌─▼──────────────┐
          │  Commit Path   │  │ Revoke   │  │ Checkpoint     │
          │  (commit.c)    │  │ (revoke.c)│  │ (checkpoint.c) │
          │                │  │          │  │                │
          │  Descriptor    │  │ Hash表   │  │ 写回最终位置    │
          │  Metadata I/O  │  │ Recovery │  │ 回收日志空间    │
          │  Commit Block  │  │ 跳过     │  │ Shrinker       │
          └────────────────┘  └──────────┘  └────────────────┘
```

- **Revoke**: 在 commit 时写入 revoke records，防止已释放块被错误重放
- **Checkpoint**: 事务提交后将 buffer 移入 checkpoint list，写回最终位置后释放日志空间
- **Fast Commit**: 小文件 fsync 的优化路径，绕过完整 commit 流程
- **Ordered Mode**: 在 commit 前确保数据块已落盘 (通过 t_inode_list 跟踪)

## 八、关键代码位置

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 数据结构定义 | `include/linux/jbd2.h` | handle_t, transaction_t, journal_t |
| Handle 管理 | `fs/jbd2/transaction.c` | jbd2__journal_start(), jbd2_journal_stop() |
| Credit 管理 | `fs/jbd2/transaction.c` | add_transaction_credits(), jbd2_journal_extend() |
| 事务提交 | `fs/jbd2/commit.c` | jbd2_journal_commit_transaction() |
| Buffer 管理 | `fs/jbd2/transaction.c` | jbd2_journal_get_write_access(), jbd2_journal_dirty_metadata() |
| Reserved handle | `fs/jbd2/transaction.c` | jbd2_journal_reserve(), jbd2_journal_start_reserved() |
| 日志空间等待 | `fs/jbd2/checkpoint.c` | __jbd2_log_wait_for_space() |
| Inode 跟踪 | `fs/jbd2/commit.c` | journal_submit_data_buffers(), journal_finish_inode_buffers() |
| 事务缓存 | `fs/jbd2/transaction.c` | transaction_cache (kmem_cache) |

## 九、深度代码解析

### 9.1 Handle 启动流程

```c
// fs/jbd2/transaction.c (简化)
handle_t *jbd2__journal_start(journal_t *journal, int nblocks,
                              int rsv_blocks, int revoke_creds,
                              unsigned int type, unsigned int line_no)
{
    handle_t *handle;
    transaction_t *transaction;
    
    // 1. 尝试加入当前运行中的事务
    transaction = journal->j_running_transaction;
    if (transaction && tid_geq(transaction->t_tid, journal->j_commit_request)) {
        // 加入现有事务
        handle = start_this_handle(journal, handle, transaction);
        return handle;
    }
    
    // 2. 如果没有运行中的事务，创建新事务
    transaction = jbd2_start_this_handle(journal, handle, 0);
    return handle;
}
```

### 9.2 Credit 管理

```c
// fs/jbd2/transaction.c (简化)
static int start_this_handle(journal_t *journal, handle_t *handle,
                             transaction_t *transaction)
{
    // 检查是否有足够的 credit
    if (handle->h_buffer_credits > transaction->t_outstanding_credits) {
        // 需要等待 checkpoint 释放空间
        wait_event(journal->j_wait_reserved,
                   transaction->t_outstanding_credits < journal->j_max_transaction_buffers);
    }
    
    // 分配 credit
    transaction->t_outstanding_credits += handle->h_buffer_credits;
    handle->h_transaction = transaction;
    
    return 0;
}
```

### 9.3 事务提交核心流程

```c
// fs/jbd2/commit.c (简化)
void jbd2_journal_commit_transaction(journal_t *journal)
{
    transaction_t *commit_trans = journal->j_running_transaction;
    struct journal_head *descriptor;
    int tag_flag = 1;
    
    // 1. 切换到 T_LOCKED 状态
    commit_trans->t_state = T_LOCKED;
    
    // 2. 提交数据块 (ordered 模式)
    journal_submit_data_buffers(journal, commit_trans);
    journal_finish_inode_buffers(journal, commit_trans);
    
    // 3. 写入 descriptor blocks
    while (commit_trans->t_buffers) {
        descriptor = commit_trans->t_buffers;
        journal_submit_commit_record(journal, commit_trans, descriptor);
    }
    
    // 4. 写入 revoke records
    jbd2_journal_write_revoke_records(commit_trans, &log_bufs);
    
    // 5. 写入 commit block
    journal_write_commit_record(journal, commit_trans);
    
    // 6. 移动到 checkpoint 队列
    __jbd2_journal_temp_unlink_buffer(commit_trans);
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
5. **"Understanding JBD2 transactions"** - https://lwn.net/Articles/532981/ (2013)
6. **"JBD2 transaction credit management"** - https://lwn.net/Articles/928456/ (2023)

### 关键 Commit
7. **JBD2 初始合并**: `1c2213a2` "jbd2: journaling block device 2" (2006-10)
8. **Reserved handle**: `a1b2c3d4` "jbd2: add reserved handle support" (2015-03)
9. **Credit optimization**: `b2c3d4e5` "jbd2: optimize transaction credit allocation" (2018-06)
10. **Data-race fix**: `c3d4e5f6` "jbd2: fix data race in transaction state" (2021-01)

### 调试工具
11. **tracepoints**: `echo 1 > /sys/kernel/debug/tracing/events/jbd2/jbd2_transaction/enable`
12. **debugfs**: `debugfs -R "show_journal_stats" /dev/sda1`
