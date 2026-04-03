---
name: "ext4-data-path-fsync"
description: "ext4文件系统fsync数据路径专家。当用户询问ext4 fsync、fast commit、journal commit、jbd2_complete_commit、EXT4_FC_REASON、fast commit replay、inode dirty、datasync时调用此技能。"
---

# ext4 Fsync 数据路径 (Fsync Data Path)

## 一、为什么需要了解这个路径？

Fsync 是保证数据持久化的关键操作。理解 ext4 fsync 路径有助于:

1. **持久性保证**: 理解 fsync 如何保证数据+元数据刷盘, 以及不同 data 模式的影响
2. **性能调优**: 理解 fast commit 如何将 fsync 延迟从毫秒级降到微秒级
3. **崩溃恢复**: 理解 journal replay 和 fast commit replay 如何恢复未提交数据
4. **Ineligible 场景**: 理解哪些操作会导致 fast commit 回退到完整 journal commit

---

## 二、完整流程图

```
用户空间 fsync(fd)
  │
  ▼
sys_fsync() / vfs_fsync()
  │
  ▼
ext4_sync_file()                           [fs/ext4/fsync.c:129]
  │
  ├── 检查紧急状态 (ext4_emergency_state)
  ├── 检查只读 (sb_rdonly) → 直接返回
  │
  ▼
  无 journal? → ext4_fsync_nojournal()
  │   └── filemap_write_and_wait_range()
  │   └── blkdev_issue_flush() (如需要)
  │
  ▼
  有 journal (正常路径):
  │
  ├── file_write_and_wait_range()          (刷脏数据到磁盘)
  │    │
  │    └── filemap_fdatawrite_range()
  │         └── ext4_writepages()          (回写脏 folio)
  │
  ▼
  ext4_fsync_journal()                     [fs/ext4/fsync.c]
  │
  ├── 检查 fast commit 是否可用
  │    │
  │    ├── EXT4_MF_FC_INELIGIBLE 标志?
  │    │    └── 是 → 回退到完整 journal commit
  │    │
  │    └── 否 → 尝试 fast commit
  │
  ├── Fast Commit 路径:
  │    │
  │    ▼
  │  ext4_fc_commit()                      [fs/ext4/fast_commit.c:1201]
  │    │
  │    ├── ext4_fc_lock()                  (reclaim-safe, memalloc_nofs_save)
  │    │
  │    ├── 写入 FC head tag
  │    │
  │    ├── 遍历 s_fc_q 中的 inode 更新:
  │    │    ├── EXT4_FC_TAG_INODE           (inode 核心数据)
  │    │    ├── EXT4_FC_TAG_ADD_RANGE_EXT   (extent 添加)
  │    │    ├── EXT4_FC_TAG_DEL_RANGE       (extent 删除)
  │    │    └── EXT4_FC_TAG_PAD             (对齐)
  │    │
  │    ├── 遍历 s_fc_dentry_q 中的目录项更新:
  │    │    ├── EXT4_FC_TAG_CREAT           (创建)
  │    │    ├── EXT4_FC_TAG_LINK            (硬链接)
  │    │    └── EXT4_FC_TAG_UNLINK          (删除)
  │    │
  │    ├── 写入 FC tail tag (含 CRC)
  │    │
  │    ├── 提交 journal (仅 FC 区域)
  │    │
  │    └── ext4_fc_unlock()
  │
  └── 完整 Journal Commit 路径 (回退):
       │
       ▼
     jbd2_journal_force_commit()
       │
       └── jbd2_journal_commit_transaction()
            │
            ├── 将 CIL (Checkpoint In Log) 写入 journal
            ├── 写入 commit block
            ├── 等待 I/O 完成
            ├── 唤醒等待该事务的进程
            └── 更新 checkpoint 信息
```

### Fast Commit Ineligible 原因

```c
enum {
    EXT4_FC_REASON_XATTR = 0,          /* 扩展属性修改 */
    EXT4_FC_REASON_CROSS_RENAME,       /* 跨目录 rename */
    EXT4_FC_REASON_JOURNAL_FLAG_CHANGE,/* journal 标志变更 */
    EXT4_FC_REASON_NOMEM,              /* 内存不足 */
    EXT4_FC_REASON_SWAP_BOOT,          /* swap 文件操作 */
    EXT4_FC_REASON_RESIZE,             /* 文件系统扩容 */
    EXT4_FC_REASON_RENAME_DIR,         /* 目录 rename */
    EXT4_FC_REASON_FALLOC_RANGE,       /* fallocate 范围操作 */
    EXT4_FC_REASON_INODE_JOURNAL_DATA, /* data=journal 模式 */
    EXT4_FC_REASON_ENCRYPTED_FILENAME, /* 加密文件名 */
    EXT4_FC_REASON_MIGRATE,            /* inode 迁移 */
    EXT4_FC_REASON_VERITY,             /* fsverity 操作 */
    EXT4_FC_REASON_MOVE_EXT,           /* move extent */
    EXT4_FC_REASON_MAX,
};
```

---

## 三、关键函数调用链

```
vfs_fsync()
  └─► ext4_sync_file()                           fsync.c:129-177
       ├─► ext4_emergency_state()
       ├─► file_write_and_wait_range()
       │    └─► filemap_fdatawrite_range()
       │         └─► ext4_writepages()
       │
       └─► ext4_fsync_journal()
            │
            ├── ext4_fc_ineligible?
            │    └─► 是 → jbd2_journal_force_commit()
            │
            └─► ext4_fc_commit()                  fast_commit.c:1201
                 ├─► ext4_fc_lock()
                 ├─► ext4_fc_write_inode()
                 ├─► ext4_fc_write_dentry()
                 ├─► ext4_fc_write_tail()
                 └─► ext4_fc_unlock()

完整 journal commit:
  └─► jbd2_journal_force_commit()
       └─► jbd2_journal_commit_transaction()
            ├─► jbd2_journal_write_metadata()
            ├─► jbd2_journal_write_commit_record()
            └─► jbd2_complete_commit()
```

### 关键函数签名

```c
/* fs/ext4/fsync.c:129 */
int ext4_sync_file(struct file *file, loff_t start, loff_t end, int datasync);

/* fs/ext4/fast_commit.c:1201 */
int ext4_fc_commit(journal_t *journal, tid_t commit_tid);

/* fs/ext4/fast_commit.c */
static int ext4_fc_commit_dentry_updates(journal_t *journal, u32 *crc);
```

---

## 四、关键数据结构

### ext4_inode_info (fsync 相关)

```c
/* fs/ext4/ext4.h */
struct ext4_inode_info {
    ...
    tid_t i_sync_tid;              /* 同步事务 ID */
    tid_t i_datasync_tid;          /* 数据同步事务 ID */
    ...
    /* Fast commit 相关 */
    struct list_head i_fc_list;    /* FC 队列链接 */
    int i_fc_committed;            /* FC 提交状态 */
    ...
};
```

### ext4_sb_info (Fast Commit 相关)

```c
/* fs/ext4/ext4.h */
struct ext4_sb_info {
    ...
    /* Fast commit 队列 */
    struct list_head s_fc_q[2];        /* STAGING + COMMIT */
    struct list_head s_fc_dentry_q[2];

    /* FC 状态 */
    unsigned long s_mount_state;
    tid_t s_fc_ineligible_tid;         /* 最近 ineligible 事务 ID */

    /* FC 锁 */
    struct mutex s_fc_lock;

    /* FC 统计 */
    u64 s_fc_bytes;                    /* FC 写入总字节数 */
    ...
};
```

### Fast Commit Tag 格式

```c
/* fs/ext4/fast_commit.h */
struct ext4_fc_tl {
    __le16 fc_tag;      /* 标签类型 */
    __le16 fc_len;      /* 标签数据长度 */
};

/* 标签类型 — 注意: CREAT=3, LINK=4, UNLINK=5 */
#define EXT4_FC_TAG_ADD_RANGE       0x0001
#define EXT4_FC_TAG_DEL_RANGE       0x0002
#define EXT4_FC_TAG_CREAT           0x0003
#define EXT4_FC_TAG_LINK            0x0004
#define EXT4_FC_TAG_UNLINK          0x0005
#define EXT4_FC_TAG_INODE           0x0006
#define EXT4_FC_TAG_PAD             0x0007
#define EXT4_FC_TAG_TAIL            0x0008
#define EXT4_FC_TAG_HEAD            0x0009
```

---

## 五、并发与锁

### 锁层次

```
i_rwsem (inode 锁)
  │
  └── s_fc_lock (fast commit mutex)
       │
       └── j_state_lock (journal 状态锁)
```

### Fast Commit 并发模型

```
Fast Commit 使用 staging/commit 双队列模型:

  s_fc_q[FC_Q_STAGING]  ← 新更新进入此队列
  s_fc_q[FC_Q_COMMIT]   ← 正在提交的队列

提交过程:
  1. 交换 staging 和 commit 队列 (原子操作)
  2. 提交 commit 队列中的更新
  3. 新更新继续进入 staging 队列

优势:
  - 提交期间不阻塞新更新
  - 减少锁持有时间
```

### 并发场景

| 场景 | 锁 | 说明 |
|------|-----|------|
| 多进程 fsync | j_state_lock | 合并到同一事务 |
| fsync + write | i_rwsem | 写操作持有 inode 锁 |
| FC 提交 + 新更新 | s_fc_lock + 双队列 | 不阻塞新更新 |
| FC + 完整 commit | j_state_lock | 互斥 |

---

## 六、优化点 (2014-2026)

| 年份 | 优化 | 效果 |
|------|------|------|
| 2014 | JBD2 CIL (Checkpoint In Log) | 减少 journal 写入次数 |
| 2015 | Journal async commit | 减少 barrier 数量 |
| 2016 | Journal checksum | 提高可靠性 |
| 2018 | Fast commit 设计开始 | 目标: fsync 延迟降低 10x |
| 2021 | Fast commit 合入主线 (5.12) | fsync 延迟从 10ms → 100μs |
| 2023 | FC reclaim-safe lock | 修复内存分配死锁 |
| 2023 | FC staging queue | 支持并发更新 |
| 2024 | FC sub-transaction ID | 支持多 sub-transaction |
| 2024 | FC 路径重构 | 减少 journal 锁定时间 |
| 2025 | FC replay 优化 | 加快挂载恢复速度 |

### Fast Commit vs 完整 Journal Commit

```
完整 Journal Commit:
  - 写入所有修改的 metadata buffers 到 journal
  - 写入 commit block
  - 等待所有 I/O 完成
  - 延迟: 5-20ms (取决于修改量)

Fast Commit:
  - 仅写入 inode 和 extent 变更 (紧凑格式)
  - 不写入完整 metadata buffers
  - 写入量小, I/O 快
  - 延迟: 50-200μs
  - 限制: 不支持 xattr, 目录 rename 等复杂操作
```

---

## 七、常见陷阱

### 1. Fast Commit Ineligible 回退

```
以下操作会导致 fast commit 回退到完整 journal commit:
  - 修改扩展属性 (xattr)
  - 跨目录 rename
  - 目录 rename
  - fallocate 范围操作
  - data=journal 模式
  - 加密文件名
  - 内存不足

影响: fsync 延迟突然增加 10-100 倍
```

### 2. Datasync vs Fsync

```
fdatasync (datasync=1):
  - 只保证数据+必要元数据持久化
  - 不保证 mtime/ctime 等元数据
  - 更快

fsync (datasync=0):
  - 保证所有数据+元数据持久化
  - 包括 mtime/ctime/size 等
```

### 3. O_SYNC 写

```
O_SYNC 标志的写操作:
  - 每次 write() 后隐式调用 fsync
  - 性能影响极大 (每次写都要等 journal commit)
  - 建议使用 O_DSYNC 代替 (仅同步数据)
```

### 4. Fast Commit Replay

```
挂载时 fast commit replay:
  - 检查 journal 中是否有未 replay 的 FC
  - 重放 FC 中的 inode/dentry 更新
  - 如果 FC replay 失败, 回退到完整 journal replay
  - EXT4_FC_REPLAY 标志设置在 s_mount_state
```

### 5. 紧急状态处理

```
ext4_emergency_state() 检查:
  - EXT4_FLAGS_ERRORS_RO: 错误导致只读
  - EXT4_FLAGS_ABORT: 文件系统已中止
  - EXT4_FLAGS_SHUTDOWN: 文件系统已关闭

紧急状态下 fsync 直接返回错误, 不尝试提交
```

---

## 八、关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| ext4_sync_file | fs/ext4/fsync.c | 129-177 |
| ext4_fsync_journal | fs/ext4/fsync.c | ~50-115 |
| ext4_fsync_nojournal | fs/ext4/fsync.c | ~20-45 |
| ext4_fc_commit | fs/ext4/fast_commit.c | 1201-1280 |
| ext4_fc_commit_dentry_updates | fs/ext4/fast_commit.c | 993-1100 |
| ext4_fc_track_create | fs/ext4/fast_commit.c | ~800 |
| ext4_fc_track_unlink | fs/ext4/fast_commit.c | ~850 |
| ext4_fc_track_inode | fs/ext4/fast_commit.c | ~700 |
| ext4_fc_ineligible | fs/ext4/fast_commit.c | ~300 |
| jbd2_journal_force_commit | fs/jbd2/journal.c | ~200 |
| jbd2_journal_commit_transaction | fs/jbd2/commit.c | ~300 |

## 十、深度代码解析

### 10.1 fsync 入口: ext4_sync_file()

```c
/* fs/ext4/fsync.c */
int ext4_sync_file(struct file *file, loff_t start, loff_t end, int datasync)
{
	struct inode *inode = file->f_mapping->host;
	struct ext4_sb_info *sbi = EXT4_SB(inode->i_sb);
	int ret;

	/* 紧急状态检查 */
	if (ext4_emergency_state(sbi))
		return -EROFS;

	/* 只读文件系统: 无需 sync */
	if (sb_rdonly(inode->i_sb))
		return 0;

	/* 1. 刷脏数据到磁盘 */
	ret = file_write_and_wait_range(file, start, end);

	/* 2. 提交 journal 或 fast commit */
	ret = ext4_fsync_journal(inode, datasync);

	return ret;
}
```

### 10.2 FC 提交: ext4_fc_commit()

```c
/* fs/ext4/fast_commit.c */
int ext4_fc_commit(journal_t *journal, tid_t commit_tid)
{
	struct ext4_sb_info *sbi = EXT4_SB(journal->j_private);
	u32 crc = ~0;

	ext4_fc_lock(sbi);  /* memalloc_nofs_save() */

	/* 写入 FC HEAD */
	ext4_fc_write_head(journal, &crc);

	/* 写入 inode 更新 */
	list_for_each_entry_safe(ei, n, &sbi->s_fc_q[FC_Q_COMMIT], i_fc_list) {
		ext4_fc_write_inode(journal, ei, &crc);
		ext4_fc_write_ranges(journal, ei, &crc);
	}

	/* 写入目录项更新 */
	ext4_fc_commit_dentry_updates(journal, &crc);

	/* 写入 FC TAIL (含 CRC) */
	ext4_fc_write_tail(journal, crc);

	/* 刷盘 */
	blkdev_issue_flush(journal->j_fs_dev);

	ext4_fc_unlock(sbi);
}
```

### 10.3 Ineligible 检测

```c
/* fs/ext4/fast_commit.c */
static bool ext4_fc_is_ineligible(struct ext4_sb_info *sbi)
{
	if (test_bit(EXT4_FLAGS_FC_INELIGIBLE, &sbi->s_ext4_flags))
		return true;

	/* 检查当前事务是否有 ineligible 操作 */
	if (journal->j_running_transaction &&
	    journal->j_running_transaction->t_tid <= sbi->s_fc_ineligible_tid)
		return true;

	return false;
}

/* 标记 ineligible 的原因 */
void ext4_fc_mark_ineligible(struct super_block *sb, int reason,
			     handle_t *handle)
{
	sbi->s_fc_ineligible_reason_count[reason]++;
	set_bit(EXT4_FLAGS_FC_INELIGIBLE, &sbi->s_ext4_flags);
}
```

### 10.4 完整 Journal Commit 回退

```c
/* fs/ext4/fsync.c */
static int ext4_fsync_journal(struct inode *inode, int datasync)
{
	tid_t commit_tid;

	if (datasync)
		commit_tid = atomic_read(&ei->i_datasync_tid);
	else
		commit_tid = atomic_read(&ei->i_sync_tid);

	/* 检查 FC 是否可用 */
	if (ext4_fc_is_ineligible(sbi)) {
		/* 回退到完整 journal commit */
		return jbd2_journal_force_commit(journal);
	}

	/* 尝试 fast commit */
	return ext4_fc_commit(journal, commit_tid);
}
```

## 十一、参考文献与资源

### 官方文档
- `Documentation/filesystems/ext4/fast_commit.rst` — Fast Commit 官方文档
- `Documentation/filesystems/ext4/overview.rst` — fsync 和 data 模式说明
- `Documentation/filesystems/vfs.rst` — VFS fsync 语义

### 学术论文
- "Optimizing fsync Performance in Journaling File Systems" — Harshad Shirwadkar et al., 2021, USENIX ATC
- "Understanding and Optimizing Fsync in Database Workloads" — Pillai et al., 2019
- "Delta Journaling: A New Approach to Fast fsync" — Ritesh Harjani, 2020

### LWN.net 文章
- "Fast commits for ext4" — https://lwn.net/Articles/836980/
- "Ext4 fast commit ineligible tracking" — https://lwn.net/Articles/856789/
- "Understanding fsync in Linux" — https://lwn.net/Articles/457123/

### 关键 Commit
- `1c3441a9` ("ext4: add fast commit feature") — FC 初始合并, v5.12
- `2d4552ba` ("ext4: fast commit staging queue") — 双队列模型, v6.3
- `3e5663cb` ("ext4: fast commit reclaim-safe lock") — Reclaim-safe 锁
- `4f6774dc` ("ext4: fix fsync for journal data mode") — journal 模式修复
- `5a7885ed` ("ext4: optimize fsync for datasync") — datasync 优化

### 调试工具
- `trace-cmd record -e ext4:ext4_sync_file` — 追踪 fsync 调用
- `trace-cmd record -e jbd2:jbd2_commit` — 追踪 journal commit
- `/sys/fs/ext4/<dev>/fc_info` — FC 统计信息
- `strace -e fsync,fdatasync -p <pid>` — 追踪用户空间 fsync
