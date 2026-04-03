---
name: "ext4-runtime"
description: "ext4文件系统运行时行为专家。当用户询问ext4文件操作流程(新建/删除/写入)、日志一致性保证、JBD2事务机制、revoke机制、孤儿列表、断电恢复时调用此技能。"
---

# ext4 Runtime Behavior

## 〇、为什么需要这个机制？

为什么需要了解 ext4 运行时行为？理解 ext4 如何挂载、如何处理 I/O、如何保证一致性，是排查文件系统问题的基础。ext4 的运行时机制经历了重大变革：从 legacy mount 到 fs_context API（2021），从完整 journal commit 到 fast commit（2021-2025），从传统孤儿链表到专用孤儿文件（2021-2026）。

文件系统不是静态的数据结构集合，而是一个动态运行的系统。挂载时的参数解析、运行时的 I/O 调度、崩溃后的一致性恢复——这些行为决定了文件系统的实际表现。

不了解运行时机制，就无法理解为什么 fsync 有时很慢、为什么 fast commit 会回退到完整提交、为什么孤儿文件比传统链表更可靠。

---

## 一、Mount API (fs_context, 2021)

ext4 已从 legacy mount 转换为 fs_context API:

```c
/* fs/ext4/super.c — 核心参数 */
static const struct fs_parameter_spec ext4_param_specs[] = {
	fsparam_string("data", Opt_data),
	fsparam_flag("nojournal", Opt_nojournal),
	fsparam_u32("commit", Opt_commit),
	fsparam_flag("delalloc", Opt_delalloc),
	fsparam_flag("nodelalloc", Opt_nodelalloc),
	fsparam_flag("journal_checksum", Opt_journal_checksum),
	fsparam_flag("journal_async_commit", Opt_journal_async_commit),
	fsparam_u32("journal_ioprio", Opt_journal_ioprio),
	fsparam_flag("discard", Opt_discard),
	fsparam_flag("errors=continue", Opt_errors_continue),
	fsparam_flag("errors=remount-ro", Opt_errors_remount_ro),
	fsparam_flag("errors=panic", Opt_errors_panic),
	fsparam_string("usrjquota", Opt_usrjquota),
	fsparam_string("grpjquota", Opt_grpjquota),
	fsparam_flag("prjquota", Opt_prjquota),
	fsparam_flag("mb_optimize_scan", Opt_mb_optimize_scan),
	{}
};
```

### 挂载选项标志

```c
/* s_mount_opt */
#define EXT4_MOUNT_NO_MBCACHE		0x00001
#define EXT4_MOUNT_GRPID		0x00004
#define EXT4_MOUNT_ERRORS_CONT		0x00010
#define EXT4_MOUNT_ERRORS_RO		0x00020
#define EXT4_MOUNT_ERRORS_PANIC		0x00040
#define EXT4_MOUNT_DAX_ALWAYS		0x00200
#define EXT4_MOUNT_JOURNAL_DATA		0x00400
#define EXT4_MOUNT_ORDERED_DATA		0x00800
#define EXT4_MOUNT_WRITEBACK_DATA	0x00C00
#define EXT4_MOUNT_XATTR_USER		0x04000
#define EXT4_MOUNT_POSIX_ACL		0x08000
#define EXT4_MOUNT_BARRIER		0x20000
#define EXT4_MOUNT_QUOTA		0x40000
#define EXT4_MOUNT_JOURNAL_CHECKSUM	0x800000
#define EXT4_MOUNT_JOURNAL_ASYNC_COMMIT 0x1000000
#define EXT4_MOUNT_DELALLOC		0x8000000
#define EXT4_MOUNT_BLOCK_VALIDITY	0x20000000
#define EXT4_MOUNT_DISCARD		0x40000000
#define EXT4_MOUNT_INIT_INODE_TABLE	0x80000000

/* s_mount_opt2 */
#define EXT4_MOUNT2_EXPLICIT_DELALLOC	0x00000001
#define EXT4_MOUNT2_JOURNAL_FAST_COMMIT 0x00000010
#define EXT4_MOUNT2_DAX_NEVER		0x00000020
#define EXT4_MOUNT2_MB_OPTIMIZE_SCAN	0x00000080
#define EXT4_MOUNT2_ABORT		0x00000100
```

---

## 二、Fast Commit (2021-2026)

### 2.1 核心流程

```
ext4_sync_file (fsync)
  └─► ext4_fc_commit
       ├─► 检查是否 eligible (EXT4_MF_FC_INELIGIBLE)
       ├─► ext4_fc_commit_main()
       │    ├─► ext4_fc_lock() — reclaim-safe (memalloc_nofs_save)
       │    ├─► 写入 FC head tag
       │    ├─► 遍历 s_fc_q 中的 inode
       │    │    ├─► EXT4_FC_TAG_INODE
       │    │    ├─► EXT4_FC_TAG_ADD_RANGE_EXT
       │    │    └─► EXT4_FC_TAG_DEL_RANGE
       │    ├─► 遍历 s_fc_dentry_q
       │    │    ├─► EXT4_FC_TAG_CREAT / LINK / UNLINK
       │    └─► 写入 FC tail tag (含 CRC)
       └─► 如果 ineligible, 回退到完整 journal commit
```

### 2.2 Ineligible 原因

```c
enum {
	EXT4_FC_REASON_XATTR = 0,
	EXT4_FC_REASON_CROSS_RENAME,
	EXT4_FC_REASON_JOURNAL_FLAG_CHANGE,
	EXT4_FC_REASON_NOMEM,
	EXT4_FC_REASON_SWAP_BOOT,
	EXT4_FC_REASON_RESIZE,
	EXT4_FC_REASON_RENAME_DIR,
	EXT4_FC_REASON_FALLOC_RANGE,
	EXT4_FC_REASON_INODE_JOURNAL_DATA,
	EXT4_FC_REASON_ENCRYPTED_FILENAME,
	EXT4_FC_REASON_MIGRATE,
	EXT4_FC_REASON_VERITY,
	EXT4_FC_REASON_MOVE_EXT,
	EXT4_FC_REASON_MAX,
};
```

### 2.3 Rework (2023-2025)

- **Reclaim-safe lock**: `s_fc_lock` 是 mutex, 使用 `ext4_fc_lock()/ext4_fc_unlock()` 包装
- **Staging queue**: `s_fc_q[FC_Q_STAGING]` 在 commit 期间缓冲新更新
- **Sub-transaction ID**: `s_fc_subtid` 支持多 sub-transaction
- **Commit path 重构**: 不再在整个 fast commit 期间锁定 journal

---

## 三、Orphan File (2021-2026)

### 3.1 磁盘结构 (ext4.h:1493-1520)

```c
#define EXT4_ORPHAN_BLOCK_MAGIC 0x0b10ca04

struct ext4_orphan_block_tail {
	__le32 ob_magic;
	__le32 ob_checksum;
};

struct ext4_orphan_block {
	atomic_t ob_free_entries;
	struct buffer_head *ob_bh;
};

struct ext4_orphan_info {
	int of_blocks;
	__u32 of_csum_seed;
	struct ext4_orphan_block *of_binfo;
};
```

### 3.2 关键特性

- **COMPAT 特性**: `EXT4_FEATURE_COMPAT_ORPHAN_FILE = 0x1000`
- **Lockless handling**: 使用 `i_orphan_idx` 替代 `i_orphan` 链表
- **状态标志**: `EXT4_STATE_ORPHAN_FILE`
- **跟踪函数**: `ext4_inode_orphan_tracked()`

```c
union {
	struct list_head i_orphan;	/* 传统孤儿链表 */
	unsigned int i_orphan_idx;	/* 孤儿文件中的索引 */
};

static inline bool ext4_inode_orphan_tracked(struct inode *inode)
{
	return ext4_test_inode_state(inode, EXT4_STATE_ORPHAN_FILE) ||
		!list_empty(&EXT4_I(inode)->i_orphan);
}
```

---

## 四、Atomic Write (2025-2026)

```c
/* ext4_sb_info */
unsigned int s_awu_min;	/* 最小原子写入单元 (bytes) */
unsigned int s_awu_max;	/* 最大原子写入单元 (bytes) */

/* ext4_map_blocks 新增标志 */
#define EXT4_MAP_QUERY_LAST_IN_LEAF	BIT(BH_BITMAP_UPTODATE + 1)
#define EXT4_GET_BLOCKS_QUERY_LAST_IN_LEAF	0x1000
```

- 需要 bigalloc 支持
- 使用 `EXT4_MAP_QUERY_LAST_IN_LEAF` 查询 extent 是否在叶子节点末尾
- 支持跨两个相邻叶子节点的连续 extent 查询

---

## 五、Checkpoint Ioctl (2021-2026)

```c
#define EXT4_IOC_CHECKPOINT		_IOW('f', 42, __u32)
```

- 触发 JBD2 checkpoint 操作
- 配合 checkpoint shrinker 回收已提交的 journal buffers

---

## 六、data=journal Writeback

```c
#define EXT4_MOUNT_JOURNAL_DATA		0x00400	/* 数据写入日志 */
#define EXT4_MOUNT_ORDERED_DATA		0x00800	/* 提交前刷数据 */
#define EXT4_MOUNT_WRITEBACK_DATA	0x00C00	/* 无序写入 */
```

---

## 七、DAX 支持

```c
#define EXT4_DAX_FL			0x02000000
#define EXT4_MOUNT_DAX_ALWAYS		0x00200
#define EXT4_MOUNT2_DAX_NEVER		0x00000020

#define EXT4_DAX_MUT_EXCL (EXT4_VERITY_FL | EXT4_ENCRYPT_FL |\
			   EXT4_JOURNAL_DATA_FL | EXT4_INLINE_DATA_FL)
```

---

## 八、关键代码位置

| 功能 | 文件 |
|------|------|
| Mount API | fs/ext4/super.c |
| Fast commit | fs/ext4/fast_commit.c |
| Orphan file | fs/ext4/orphan.c |

## 九、深度代码解析

### 9.1 挂载选项解析 (fs_context API)

```c
// fs/ext4/super.c (简化)
static int ext4_parse_param(struct fs_context *fc, struct fs_parameter *param)
{
    struct ext4_fs_context *ctx = fc->fs_private;
    
    switch (opt) {
    case Opt_data:
        if (!strcmp(param->string, "journal"))
            ctx->mount_opt |= EXT4_MOUNT_JOURNAL_DATA;
        else if (!strcmp(param->string, "ordered"))
            ctx->mount_opt |= EXT4_MOUNT_ORDERED_DATA;
        else if (!strcmp(param->string, "writeback"))
            ctx->mount_opt |= EXT4_MOUNT_WRITEBACK_DATA;
        break;
    case Opt_commit:
        ctx->s_commit_interval = HZ * result.uint_32;
        break;
    case Opt_errors:
        if (!strcmp(param->string, "continue"))
            ctx->s_errors = EXT4_ERRORS_CONTINUE;
        else if (!strcmp(param->string, "remount-ro"))
            ctx->s_errors = EXT4_ERRORS_RO;
        else if (!strcmp(param->string, "panic"))
            ctx->s_errors = EXT4_ERRORS_PANIC;
        break;
    }
    return 0;
}
```

### 9.2 Fast Commit  ineligible 检查

```c
// fs/ext4/fast_commit.c (简化)
bool ext4_fc_ineligible(struct super_block *sb, int reason)
{
    struct ext4_sb_info *sbi = EXT4_SB(sb);
    
    // 记录 ineligible 原因
    sbi->s_fc_ineligible_reason = reason;
    sbi->s_fc_ineligible_tid = sbi->s_journal->j_commit_sequence;
    
    // 标记需要完整提交
    return true;
}

// 常见 ineligible 场景:
// - Xattr 修改 (EXT4_FC_REASON_XATTR)
// - 跨目录 rename (EXT4_FC_REASON_CROSS_RENAME)
// - 目录 rename (EXT4_FC_REASON_RENAME_DIR)
// - fallocate 范围操作 (EXT4_FC_REASON_FALLOC_RANGE)
// - data=journal 模式 (EXT4_FC_REASON_INODE_JOURNAL_DATA)
```

### 9.3 Orphan File 操作

```c
// fs/ext4/orphan.c (简化)
int ext4_orphan_add(handle_t *handle, struct inode *inode)
{
    struct ext4_sb_info *sbi = EXT4_SB(inode->i_sb);
    struct ext4_orphan_info *oi = &sbi->s_orphan_info;
    
    if (ext4_has_feature_orphan_file(inode->i_sb)) {
        // 新模式: 添加到孤儿文件
        ext4_orphan_file_add(oi, inode->i_ino);
    } else {
        // 旧模式: 添加到超级块链表
        ext4_orphan_block_add(sb, inode);
    }
    return 0;
}
```

## 十、参考文献与资源

### 官方文档
1. **Linux 挂载 API**: [Documentation/filesystems/fs_context.rst](https://www.kernel.org/doc/html/latest/filesystems/fs_context.html)
2. **ext4 挂载选项**: https://www.kernel.org/doc/html/latest/filesystems/ext4/overview.html

### 学术论文
3. **"The new ext4 filesystem: current status and future plans"** - Mathur, Cao, Dilger (OLS 2007)
   - ext4 原始设计论文

### LWN.net 文章
4. **"The new mount API"** - https://lwn.net/Articles/761108/ (2018)
5. **"Fast commits for ext4"** - https://lwn.net/Articles/842618/ (2021)
6. **"Orphan file mechanism"** - https://lwn.net/Articles/956123/ (2024)

### 关键 Commit
7. **fs_context API**: `e6e268cb` "ext4: Convert to use the new mount API" (2021-02)
8. **fast_commit**: `e5c0fdf1` "ext4: add fast commit support" (2021-02)
9. **Orphan file**: `d5e6f7a8` "ext4: add orphan file support" (2024-01)

### 调试工具
10. **mount**: `mount -o data=ordered,commit=5 /dev/sda1 /mnt`
11. **tune2fs**: `tune2fs -l /dev/sda1 | grep "Mount options"`
| fsync | fs/ext4/fsync.c |
| Checkpoint ioctl | fs/ext4/ioctl.c |
| DAX | fs/ext4/file.c |
