---
name: "jbd2-revoke"
description: "JBD2 Revoke机制专家。当用户询问JBD2 revoke表、jbd2_journal_revoke、revoke恢复、revoke记录、recovery跳过块、jbd2_journal_cancel_revoke时调用此技能。"
---

# JBD2 Revoke 撤销机制

## 一、问题起源

Revoke 机制解决的是日志恢复过程中的一个关键问题: **同一个物理块在同一个事务中被释放后又被重新分配**。

**场景示例**:
```
事务 T100:
  1. 释放 inode 的间接块 A (block #5000)
  2. 分配新文件的间接块 B (也分配到 block #5000)
  3. 写入新数据到 block #5000

如果系统在步骤 3 之后、事务提交之前崩溃:
  - 日志中包含 block #5000 的旧数据 (步骤 1 之前)
  - 如果不做处理，恢复时会重放旧数据到 block #5000
  - 这会覆盖步骤 3 写入的新数据!
```

Revoke 机制通过在日志中记录 "block #5000 已被撤销"，告诉恢复代码: "不要重放这个块在撤销记录之前的任何日志条目"。

## 二、为什么需要

1. **防止旧数据覆盖新数据**: 块被释放并重新分配后，日志中的旧副本不应被重放
2. **恢复正确性**: 确保崩溃恢复后的文件系统状态与崩溃前一致
3. **块重用安全**: 允许在同一事务中安全地释放和重新分配块
4. **元数据完整性**: 特别重要于 inode 表、间接块、目录块等元数据

## 三、核心设计

### 3.1 Revoke 核心思想

```
运行时 (Runtime):
  ┌─────────────────────────────────────────────────┐
  │  jbd2_journal_revoke(handle, blocknr, bh)       │
  │  │                                               │
  │  ├─► 在 revoke hash 表中记录 (blocknr, tid)      │
  │  ├─► 设置 buffer 的 BH_Revoked + BH_RevokeValid  │
  │  └─► 消耗一个 h_revoke_credits                   │
  │                                                   │
  │  jbd2_journal_cancel_revoke(handle, jh)          │
  │  │                                               │
  │  └─► 如果块被重新分配 (get_write_access)          │
  │       从 hash 表中移除 revoke 记录                │
  └─────────────────────────────────────────────────┘

提交时 (Commit):
  ┌─────────────────────────────────────────────────┐
  │  jbd2_journal_write_revoke_records()            │
  │  │                                               │
  │  ├─► 遍历 revoke hash 表                         │
  │  ├─► 为每个 revoked block 写入 revoke record     │
  │  │    (在 Revoke Block 中)                       │
  │  └─► 清空 hash 表，切换到下一个 revoke table     │
  └─────────────────────────────────────────────────┘

恢复时 (Recovery):
  ┌─────────────────────────────────────────────────┐
  │  PASS_SCAN:  找到日志末尾                         │
  │  PASS_REVOKE: 收集所有 revoke records            │
  │  PASS_REPLAY: 重放未被 revoke 的 blocks          │
  │  │                                               │
  │  └─► 对每个日志块:                               │
  │       if (jbd2_journal_test_revoke(block, tid))  │
  │          └─► 跳过 (nr_revoke_hits++)             │
  │       else                                       │
  │          └─► 重放到最终位置 (nr_replays++)        │
  └─────────────────────────────────────────────────┘
```

### 3.2 Revoke 与 Journal 数据的交互规则

```
Block 被 revoke 然后被 journaled:
  ─► 取消 revoke (cancel_revoke)，新数据优先
  ─► 最终结果: journaling 新块

Block 被 journaled 然后被 revoke:
  ─► Revoke 优先，需要取消 journal entry 或在日志后面写 revoke
  ─► 选择后者: journaling 一个块会取消该块的 revoke 记录
  ─► 所以事务中的 revoke 一定发生在 journaling 之后
  ─► 最终结果: revoke 生效

Block 被 revoke 然后作为数据写入:
  ─► 数据写入被允许，但 revoke 不取消
  ─► 仍然需要防止旧日志记录覆盖新数据
  ─► 最终结果: revoke 仍然生效
```

### 3.3 Revoke Hash 表双缓冲设计

```
journal->j_revoke_table[0]  ──┐
                               ├──► journal->j_revoke (当前活跃表)
journal->j_revoke_table[1]  ──┘

运行时:
  - 所有 jbd2_journal_revoke() 操作写入 journal->j_revoke
  - 由 running transaction 的 handle 操作
  - 需要 j_revoke_lock 保护

提交时:
  - kjournald2 确定哪个表是 committing transaction 的
  - 从该表写出 revoke records 到磁盘
  - 清空该表
  - journal_switch_revoke_table() 切换指针

恢复时:
  - 使用独立的 revoke table (不来自 j_revoke_table[])
  - 扫描日志时构建 revoke 记录
  - 恢复完成后清除
```

### 3.4 Buffer 状态三态机

```
BH_RevokeValid  BH_Revoked  含义
───────────────────────────────────────────
    0              X        无缓存状态，需要查找 hash 表
    1              0        块未被 revoke，cancel_revoke 无需操作
    1              1        块已被 revoke

状态转换:
  初始 (无缓存) ──► jbd2_journal_revoke() ──► Revoked=1, RevokeValid=1
       │
       └──► jbd2_journal_cancel_revoke() ──► Revoked=0, RevokeValid=1
```

## 四、关键数据结构

### 4.1 Revoke Record

```c
/* fs/jbd2/revoke.c:102-107 */
struct jbd2_revoke_record_s
{
	struct list_head  hash;		/* hash 链表节点 */
	tid_t		  sequence;	/* 事务序列号 (仅恢复时使用) */
	unsigned long long blocknr;	/* 被撤销的块号 */
};
```

### 4.2 Revoke Table

```c
/* fs/jbd2/revoke.c:111-118 */
struct jbd2_revoke_table_s
{
	int		  hash_size;	/* hash 表大小 (必须是 2 的幂) */
	int		  hash_shift;	/* log2(hash_size)，用于快速 hash */
	struct list_head *hash_table;	/* hash 桶数组 */
};
```

### 4.3 磁盘上的 Revoke Block 头

```c
/* include/linux/jbd2.h:210-214 */
typedef struct jbd2_journal_revoke_header_s
{
	journal_header_t r_header;	/* 通用日志头 (magic, blocktype, sequence) */
	__be32		 r_count;	/* 块中已使用的字节数 */
} jbd2_journal_revoke_header_t;
```

### 4.4 Revoke 相关的 Buffer 状态位

```c
/* include/linux/jbd2.h:305-317 */
enum jbd2_state_bits {
	/* ... */
	BH_Revoked,		/* Has been revoked from the log */
	BH_RevokeValid,		/* Revoked flag is valid (缓存有效) */
	/* ... */
};

BUFFER_FNS(Revoked, revoked)		/* buffer_revoked() / set_buffer_revoked() */
TAS_BUFFER_FNS(Revoked, revoked)	/* test_and_set_buffer_revoked() */
BUFFER_FNS(RevokeValid, revokevalid)	/* buffer_revokevalid() / set_buffer_revokevalid() */
TAS_BUFFER_FNS(RevokeValid, revokevalid)/* test_and_set_buffer_revokevalid() */
```

### 4.5 Journal 中的 Revoke 相关字段

```c
/* include/linux/jbd2.h (节选) */
struct journal_s
{
	/* Revoke 表 [j_revoke_lock] */
	struct jbd2_revoke_table_s *j_revoke;		/* 当前活跃的 revoke 表 */
	struct jbd2_revoke_table_s *j_revoke_table[2];	/* 双缓冲 */
	spinlock_t		j_revoke_lock;		/* 保护 revoke 表 */

	/* 每个块能容纳的 revoke 记录数 */
	int			j_revoke_records_per_block;

	/* 事务中的 revoke 计数 */
	/* (在 transaction_t 中) */
};

/* transaction_t 中的 revoke 计数 */
struct transaction_s
{
	atomic_t	t_outstanding_revokes;	/* 已停止 handle 添加的 revoke 记录数 */
};
```

### 4.6 Recovery 信息结构

```c
/* fs/jbd2/recovery.c:29-38 */
struct recovery_info
{
	tid_t		start_transaction;	/* 恢复的第一个事务 tid */
	tid_t		end_transaction;	/* 恢复的最后一个事务 tid */
	unsigned long	head_block;		/* 日志头块位置 */

	int		nr_replays;		/* 重放的块数 */
	int		nr_revokes;		/* 收集的 revoke 记录数 */
	int		nr_revoke_hits;		/* 因 revoke 跳过的块数 */
};
```

## 五、操作流程

### 5.1 运行时: Revoke 一个块

```
ext4 释放元数据块时调用 jbd2_journal_revoke(handle, blocknr, bh)
  │
  ├─► 确保 revoke 特性已启用
  │    jbd2_journal_set_features(journal, 0, 0, JBD2_FEATURE_INCOMPAT_REVOKE)
  │
  ├─► 查找 blocknr 对应的 buffer_head (如果未传入)
  │    __find_get_block_nonatomic(bdev, blocknr, blocksize)
  │
  ├─► 检查 h_revoke_credits > 0 (必须有足够的 revoke credits)
  │
  ├─► 如果找到了 bh:
  │    ├─► 检查不能连续 revoke 两次 (assert: !buffer_revoked(bh))
  │    ├─► set_buffer_revoked(bh)
  │    ├─► set_buffer_revokevalid(bh)
  │    └─► 如果传入了 bh_in:
  │         └─► jbd2_journal_forget(handle, bh_in)
  │              (将 buffer 从事务中移除)
  │
  ├─► handle->h_revoke_credits--
  │
  └─► insert_revoke_hash(journal, blocknr, handle->h_transaction->t_tid)
       │
       ├─► 分配 jbd2_revoke_record_s (kmem_cache_alloc, __GFP_NOFAIL)
       ├─► record->sequence = tid
       ├─► record->blocknr = blocknr
       ├─► hash_list = &journal->j_revoke->hash_table[hash(journal, blocknr)]
       ├─► spin_lock(&journal->j_revoke_lock)
       ├─► list_add(&record->hash, hash_list)
       └─► spin_unlock(&journal->j_revoke_lock)
```

### 5.2 运行时: Cancel Revoke (块被重新分配)

```
ext4 获取块写权限时调用 jbd2_journal_cancel_revoke(handle, jh)
  (在 jbd2_journal_get_write_access() 内部调用)
  │
  ├─► 检查 BH_RevokeValid 状态:
  │    │
  │    ├─► 如果 RevokeValid 已设置:
  │    │    └─► need_cancel = test_and_clear_buffer_revoked(bh)
  │    │         (仅当 Revoked=1 时才需要 cancel)
  │    │
  │    └─► 如果 RevokeValid 未设置:
  │         └─► need_cancel = 1 (保守处理，总是搜索)
  │              clear_buffer_revoked(bh)
  │
  ├─► 如果 need_cancel:
  │    │
  │    ├─► record = find_revoke_record(journal, bh->b_blocknr)
  │    │    │
  │    │    ├─► hash_list = &journal->j_revoke->hash_table[hash(journal, blocknr)]
  │    │    ├─► spin_lock(&journal->j_revoke_lock)
  │    │    ├─► 遍历 hash_list 查找匹配的 blocknr
  │    │    └─► spin_unlock(&journal->j_revoke_lock)
  │    │
  │    └─► 如果找到 record:
  │         ├─► spin_lock(&journal->j_revoke_lock)
  │         ├─► list_del(&record->hash)
  │         ├─► spin_unlock(&journal->j_revoke_lock)
  │         └─► kmem_cache_free(jbd2_revoke_record_cache, record)
  │
  └─► 清除所有 alias buffer 的 revoked 标志
       (确保同一块的所有 buffer_head 状态一致)
```

### 5.3 提交时: 写出 Revoke Records

```
jbd2_journal_commit_transaction() 中:
  │
  └─► jbd2_journal_write_revoke_records(transaction, &log_bufs)
       │
       ├─► 确定 committing transaction 的 revoke table:
       │    revoke = (journal->j_revoke == journal->j_revoke_table[0])
       │             ? journal->j_revoke_table[1] : journal->j_revoke_table[0]
       │
       ├─► 遍历所有 hash 桶:
       │    for i = 0 to revoke->hash_size - 1:
       │      while (!list_empty(hash_table[i])):
       │        record = hash_table[i].next
       │        write_one_revoke_record(transaction, &log_bufs,
       │                                &descriptor, &offset, record)
       │        list_del(&record->hash)
       │        kmem_cache_free(jbd2_revoke_record_cache, record)
       │
       └─► 如果有未刷新的 descriptor:
            flush_descriptor(journal, descriptor, offset)
```

### 5.4 写出单个 Revoke Record

```
write_one_revoke_record(transaction, log_bufs, &descriptor, &offset, record)
  │
  ├─► 计算 revoke record 大小:
  │    sz = 8 (64bit journal) 或 4 (32bit journal)
  │
  ├─► 检查当前 descriptor 是否有空间:
  │    if (offset + sz > blocksize - csum_tail_size):
  │      flush_descriptor(journal, descriptor, offset)
  │      descriptor = NULL
  │
  ├─► 如果没有 descriptor，创建新的:
  │    descriptor = jbd2_journal_get_descriptor_buffer(transaction,
  │                                                    JBD2_REVOKE_BLOCK)
  │    offset = sizeof(jbd2_journal_revoke_header_t)
  │    jbd2_file_log_bh(log_bufs, descriptor)
  │
  ├─► 写入 blocknr 到 descriptor:
  │    if (64bit):
  │      *(__be64 *)&descriptor->b_data[offset] = cpu_to_be64(record->blocknr)
  │    else:
  │      *(__be32 *)&descriptor->b_data[offset] = cpu_to_be32(record->blocknr)
  │
  └─► offset += sz
```

### 5.5 刷新 Revoke Descriptor

```
flush_descriptor(journal, descriptor, offset)
  │
  ├─► header = (jbd2_journal_revoke_header_t *)descriptor->b_data
  ├─► header->r_count = cpu_to_be32(offset)
  │
  ├─► 计算并设置校验和:
  │    jbd2_descriptor_block_csum_set(journal, descriptor)
  │
  ├─► set_buffer_jwrite(descriptor)
  ├─► set_buffer_dirty(descriptor)
  └─► write_dirty_buffer(descriptor, JBD2_JOURNAL_REQ_FLAGS)
       (提交 I/O 到日志设备)
```

### 5.6 切换 Revoke Table

```
jbd2_journal_switch_revoke_table(journal)
  (在事务提交的最后阶段调用)
  │
  ├─► 切换指针:
  │    if (journal->j_revoke == journal->j_revoke_table[0])
  │      journal->j_revoke = journal->j_revoke_table[1]
  │    else
  │      journal->j_revoke = journal->j_revoke_table[0]
  │
  └─► 清空新表:
       for i = 0 to hash_size - 1:
         INIT_LIST_HEAD(&journal->j_revoke->hash_table[i])
```

### 5.7 恢复时: 三阶段恢复

```
jbd2_journal_recover(journal)
  │
  ├─► 如果 journal->j_tail == 0 (干净卸载):
  │    └─► 无需恢复，直接返回
  │
  ├─► PASS_SCAN: do_one_pass(journal, &info, PASS_SCAN)
  │    │
  │    └─► 顺序扫描日志，找到最后一个有效的 commit block
  │         确定 start_transaction 和 end_transaction
  │         确定 head_block (日志头位置)
  │
  ├─► PASS_REVOKE: do_one_pass(journal, &info, PASS_REVOKE)
  │    │
  │    └─► 再次扫描日志，收集所有 revoke records
  │         │
  │         ├─► 遇到 Descriptor Block: 跳过 (不处理数据块)
  │         ├─► 遇到 Revoke Block:
  │         │    └─► scan_revoke_records()
  │         │         │
  │         │         ├─► 读取 header->r_count 确定有效数据量
  │         │         ├─► 遍历块中的 revoke entries
  │         │         │    │
  │         │         │    ├─► 读取 blocknr (4 或 8 字节)
  │         │         │    └─► jbd2_journal_set_revoke(journal, blocknr, sequence)
  │         │         │         │
  │         │         │         ├─► 查找现有记录
  │         │         │         │    └─► 如果有多个 revoke，保留最新的 sequence
  │         │         │         └─► 否则插入新记录
  │         │         │
  │         │         └─► nr_revokes += 记录数
  │         │
  │         └─► 遇到 Commit Block: 更新当前 sequence
  │
  └─► PASS_REPLAY: do_one_pass(journal, &info, PASS_REPLAY)
       │
       └─► 第三次扫描日志，重放未被 revoke 的 blocks
            │
            ├─► 遇到 Descriptor Block:
            │    └─► 对每个 tag:
            │         block = tag->t_blocknr
            │         if (jbd2_journal_test_revoke(journal, block, sequence)):
            │           └─► nr_revoke_hits++ (跳过)
            │         else:
            │           └─► 将日志中的数据块写入最终位置
            │                nr_replays++
            │
            └─► 完成后:
                 journal->j_transaction_sequence = ++end_transaction
                 journal->j_head = head_block
                 jbd2_journal_clear_revoke(journal)
                 sync_blockdev() + blkdev_issue_flush()
```

### 5.8 恢复时: Test Revoke

```
jbd2_journal_test_revoke(journal, blocknr, sequence)
  │
  ├─► record = find_revoke_record(journal, blocknr)
  │
  ├─► 如果没有记录 → 返回 0 (不需要跳过)
  │
  └─► 如果有记录:
       └─► 如果 sequence > record->sequence → 返回 0
            (日志条目比 revoke 记录更新，应该重放)
       └─► 否则 → 返回 1
            (revoke 记录比日志条目更新或相同，应该跳过)
```

## 六、10年演进 (2015-2025)

### 6.1 Revoke Credits 系统 (2015-2017)
- 引入 `h_revoke_credits` 独立于普通 credits
- 每个 handle 必须声明所需的 revoke 记录数
- 防止 revoke 记录耗尽导致的问题
- `j_revoke_records_per_block` 动态计算每块可容纳的记录数

### 6.2 64-bit Revoke Records (2016-2018)
- 支持 `JBD2_FEATURE_INCOMPAT_64BIT` 时使用 8 字节块号
- 32-bit 模式下使用 4 字节块号
- 向后兼容: 32-bit 文件系统仍然使用 4 字节格式

### 6.3 Revoke Checksumming (2017-2019)
- Revoke Block 添加 `jbd2_journal_block_tail` 校验和尾部
- 使用 CRC32C 校验整个 revoke descriptor block
- 与 metadata checksumming (CSUM_V2/V3) 集成

### 6.4 Revoke Table 优化 (2019-2021)
- 使用 `kvmalloc_objs()` 分配 hash 表，支持大表分配
- 改进 hash 函数: `hash_64(block, shift)` 替代简单取模
- 减少锁竞争: 仅在插入/查找时持有 `j_revoke_lock`

### 6.5 Recovery 优化 (2020-2023)
- 改进 readahead 策略: 128KB 预读，批量提交最多 8 个 buffer
- Fast commit replay 集成: `fc_do_one_pass()` 专用路径
- Recovery 统计增强: `nr_replays`, `nr_revokes`, `nr_revoke_hits`

### 6.6 Buffer Revoke 状态缓存 (2021-2024)
- `BH_Revoked` 和 `BH_RevokeValid` 位标志优化
- 减少不必要的 hash 表查找
- `jbd2_clear_buffer_revoked_flags()` 在事务切换时批量清除

## 七、与其他机制的关系

```
                    ┌─────────────────────────────────────────┐
                    │           ext4 文件系统层                │
                    │  ext4_free_blocks() → jbd2_journal_revoke()│
                    │  ext4_get_write_access() → cancel_revoke │
                    └───────────────┬─────────────────────────┘
                                    │
                    ┌───────────────▼─────────────────────────┐
                    │         JBD2 Transaction Layer           │
                    │  handle->h_revoke_credits                │
                    │  transaction->t_outstanding_revokes      │
                    └───────────────┬─────────────────────────┘
                                    │
                    ┌───────────────▼─────────────────────────┐
                    │          JBD2 Revoke Layer               │
                    │  j_revoke_table[2] (双缓冲)              │
                    │  jbd2_revoke_record_s (hash 表)          │
                    └───┬───────────┬─────────────────────────┘
                        │           │
          ┌─────────────▼──┐  ┌────▼────────────────────────┐
          │  Commit Path   │  │      Recovery Path           │
          │  (commit.c)    │  │      (recovery.c)            │
          │                │  │                              │
          │  写出 revoke   │  │  PASS_REVOKE: 收集 revoke    │
          │  records 到    │  │  PASS_REPLAY: 跳过 revoked   │
          │  Revoke Block  │  │  blocks                      │
          │                │  │  jbd2_journal_test_revoke()  │
          └────────────────┘  └──────────────────────────────┘
```

- **Transaction**: handle 声明 revoke credits，事务跟踪 outstanding revokes
- **Commit**: 将 revoke hash 表写出到磁盘的 Revoke Block
- **Recovery**: 三阶段恢复中 PASS_REVOKE 收集 revoke 记录，PASS_REPLAY 跳过被 revoke 的块
- **Buffer Cache**: 使用 BH_Revoked 和 BH_RevokeValid 位标志缓存 revoke 状态
- **Block Allocation**: 块释放时调用 revoke，块分配时 cancel revoke

## 八、关键代码位置

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| Revoke 数据结构 | `fs/jbd2/revoke.c` | jbd2_revoke_record_s, jbd2_revoke_table_s |
| Revoke 一个块 | `fs/jbd2/revoke.c` | jbd2_journal_revoke() |
| Cancel Revoke | `fs/jbd2/revoke.c` | jbd2_journal_cancel_revoke() |
| 写出 Revoke Records | `fs/jbd2/revoke.c` | jbd2_journal_write_revoke_records() |
| 写出单个 Revoke Record | `fs/jbd2/revoke.c` | write_one_revoke_record() |
| 刷新 Descriptor | `fs/jbd2/revoke.c` | flush_descriptor() |
| 切换 Revoke Table | `fs/jbd2/revoke.c` | jbd2_journal_switch_revoke_table() |
| Revoke Table 初始化 | `fs/jbd2/revoke.c` | jbd2_journal_init_revoke() |
| Revoke Table 销毁 | `fs/jbd2/revoke.c` | jbd2_journal_destroy_revoke() |
| 恢复: 设置 Revoke | `fs/jbd2/revoke.c` | jbd2_journal_set_revoke() |
| 恢复: 测试 Revoke | `fs/jbd2/revoke.c` | jbd2_journal_test_revoke() |
| 恢复: 清除 Revoke | `fs/jbd2/revoke.c` | jbd2_journal_clear_revoke() |
| 扫描 Revoke Records | `fs/jbd2/recovery.c` | scan_revoke_records() |
| 恢复主流程 | `fs/jbd2/recovery.c` | jbd2_journal_recover() |
| 恢复信息 | `fs/jbd2/recovery.c` | recovery_info, do_one_pass() |
| Revoke 磁盘格式 | `include/linux/jbd2.h` | jbd2_journal_revoke_header_t |
| Buffer Revoke 状态位 | `include/linux/jbd2.h` | BH_Revoked, BH_RevokeValid |
| Revoke 缓存管理 | `fs/jbd2/revoke.c` | jbd2_revoke_record_cache, jbd2_revoke_table_cache |
| 清除 Buffer Revoke 标志 | `fs/jbd2/revoke.c` | jbd2_clear_buffer_revoked_flags() |
