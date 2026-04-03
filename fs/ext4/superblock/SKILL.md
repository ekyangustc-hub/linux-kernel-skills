---
name: "ext4-superblock"
description: "ext4文件系统超级块专家。当用户询问ext4超级块结构、s_magic、s_log_block_size、s_feature_incompat、文件系统参数时调用此技能。"
---

# ext4 Super Block

超级块是 ext4 文件系统的核心元数据结构，存储所有文件系统配置和运行时统计信息。

## 〇、为什么需要这个机制？

为什么需要超级块？没有超级块，文件系统就不知道自己的大小、块大小、inode 数量等基本信息。超级块是文件系统的"身份证"。ext4 的超级块经历了多次扩展：从 ext2 的简单参数，到 ext3 的日志信息，再到 ext4 的 bigalloc/encrypt/fast_commit 等新特性参数。

超级块的设计面临一个根本矛盾：它包含的信息越来越多，但必须保持在一个块（通常 4K）内。解决方案是分阶段扩展：先利用保留字段，再添加高位扩展字段（如 s_blocks_count_hi），最后通过特性标志启用新字段。

没有超级块，挂载操作就无法进行——内核无法确认这是一个 ext4 文件系统，也无法知道如何解释磁盘上的其他结构。

---

## 一、结构定义 (verified: ext4.h:1332-1463)

```c
struct ext4_super_block {
/*00*/	__le32	s_inodes_count;		/* Inodes count */
	__le32	s_blocks_count_lo;	/* Blocks count */
	__le32	s_r_blocks_count_lo;	/* Reserved blocks count */
	__le32	s_free_blocks_count_lo;	/* Free blocks count */
/*10*/	__le32	s_free_inodes_count;	/* Free inodes count */
	__le32	s_first_data_block;	/* First Data Block */
	__le32	s_log_block_size;	/* Block size */
	__le32	s_log_cluster_size;	/* Allocation cluster size */
/*20*/	__le32	s_blocks_per_group;	/* # Blocks per group */
	__le32	s_clusters_per_group;	/* # Clusters per group */
	__le32	s_inodes_per_group;	/* # Inodes per group */
	__le32	s_mtime;		/* Mount time */
/*30*/	__le32	s_wtime;		/* Write time */
	__le16	s_mnt_count;		/* Mount count */
	__le16	s_max_mnt_count;	/* Maximal mount count */
	__le16	s_magic;		/* Magic signature */
	__le16	s_state;		/* File system state */
	__le16	s_errors;		/* Behaviour when detecting errors */
	__le16	s_minor_rev_level;	/* minor revision level */
/*40*/	__le32	s_lastcheck;		/* time of last check */
	__le32	s_checkinterval;	/* max. time between checks */
	__le32	s_creator_os;		/* OS */
	__le32	s_rev_level;		/* Revision level */
/*50*/	__le16	s_def_resuid;		/* Default uid for reserved blocks */
	__le16	s_def_resgid;		/* Default gid for reserved blocks */
	__le32	s_first_ino;		/* First non-reserved inode */
	__le16  s_inode_size;		/* size of inode structure */
	__le16	s_block_group_nr;	/* block group # of this superblock */
	__le32	s_feature_compat;	/* compatible feature set */
/*60*/	__le32	s_feature_incompat;	/* incompatible feature set */
	__le32	s_feature_ro_compat;	/* readonly-compatible feature set */
/*68*/	__u8	s_uuid[16];		/* 128-bit uuid for volume */
/*78*/	char	s_volume_name[16];	/* volume name */
/*88*/	char	s_last_mounted[64];	/* directory where last mounted */
/*C8*/	__le32	s_algorithm_usage_bitmap; /* For compression */
	__u8	s_prealloc_blocks;	/* Nr of blocks to try to preallocate*/
	__u8	s_prealloc_dir_blocks;	/* Nr to preallocate for dirs */
	__le16	s_reserved_gdt_blocks;	/* Per group desc for online growth */
/*D0*/	__u8	s_journal_uuid[16];	/* uuid of journal superblock */
/*E0*/	__le32	s_journal_inum;		/* inode number of journal file */
	__le32	s_journal_dev;		/* device number of journal file */
	__le32	s_last_orphan;		/* start of list of inodes to delete */
	__le32	s_hash_seed[4];		/* HTREE hash seed */
	__u8	s_def_hash_version;	/* Default hash version to use */
	__u8	s_jnl_backup_type;
	__le16  s_desc_size;		/* size of group descriptor */
/*100*/	__le32	s_default_mount_opts;
	__le32	s_first_meta_bg;	/* First metablock block group */
	__le32	s_mkfs_time;		/* When the filesystem was created */
	__le32	s_jnl_blocks[17];	/* Backup of the journal inode */
	/* 64bit support valid if EXT4_FEATURE_INCOMPAT_64BIT */
/*150*/	__le32	s_blocks_count_hi;
	__le32	s_r_blocks_count_hi;
	__le32	s_free_blocks_count_hi;
	__le16	s_min_extra_isize;
	__le16	s_want_extra_isize;
	__le32	s_flags;
	__le16  s_raid_stride;
	__le16  s_mmp_update_interval;
	__le64  s_mmp_block;
	__le32  s_raid_stripe_width;
	__u8	s_log_groups_per_flex;
	__u8	s_checksum_type;
	__u8	s_encryption_level;
	__u8	s_reserved_pad;
	__le64	s_kbytes_written;
	__le32	s_snapshot_inum;
	__le32	s_snapshot_id;
	__le64	s_snapshot_r_blocks_count;
	__le32	s_snapshot_list;
	__le32	s_error_count;
	__le32	s_first_error_time;
	__le32	s_first_error_ino;
	__le64	s_first_error_block;
	__u8	s_first_error_func[32];
	__le32	s_first_error_line;
	__le32	s_last_error_time;
	__le32	s_last_error_ino;
	__le32	s_last_error_line;
	__le64	s_last_error_block;
	__u8	s_last_error_func[32];
	__u8	s_mount_opts[64];
	__le32	s_usr_quota_inum;
	__le32	s_grp_quota_inum;
	__le32	s_overhead_clusters;
	__le32	s_backup_bgs[2];
	__u8	s_encrypt_algos[4];
	__u8	s_encrypt_pw_salt[16];
	__le32	s_lpf_ino;
	__le32	s_prj_quota_inum;
	__le32	s_checksum_seed;
	__u8	s_wtime_hi;
	__u8	s_mtime_hi;
	__u8	s_mkfs_time_hi;
	__u8	s_lastcheck_hi;
	__u8	s_first_error_time_hi;
	__u8	s_last_error_time_hi;
	__u8	s_first_error_errcode;
	__u8    s_last_error_errcode;
	__le16  s_encoding;		/* Filename charset encoding */
	__le16  s_encoding_flags;	/* Filename charset encoding flags */
	__le32  s_orphan_file_inum;	/* Inode for tracking orphan inodes */
	__le16	s_def_resuid_hi;
	__le16	s_def_resgid_hi;
	__le32	s_reserved[93];		/* Padding to the end of the block */
	__le32	s_checksum;		/* crc32c(superblock) */
};
```

---

## 二、关键字段

### 2.1 基本参数

| 字段 | 说明 |
|------|------|
| `s_magic` | 0xEF53 |
| `s_log_block_size` | block_size = 1024 << s_log_block_size |
| `s_log_cluster_size` | cluster_size (bigalloc), 0 表示等于 block_size |
| `s_inode_size` | 通常 256 字节 |
| `s_desc_size` | 32 (无 64bit) 或 64 (有 64bit) |

### 2.2 错误记录字段 (s_error_count 到 s_last_error_func)

```c
#define EXT4_S_ERR_START offsetof(struct ext4_super_block, s_error_count)
#define EXT4_S_ERR_END   offsetof(struct ext4_super_block, s_mount_opts)
#define EXT4_S_ERR_LEN   (EXT4_S_ERR_END - EXT4_S_ERR_START)
```

包含: s_error_count, s_first_error_time, s_first_error_ino, s_first_error_block, s_first_error_func[32], s_first_error_line, s_last_error_time, s_last_error_ino, s_last_error_line, s_last_error_block, s_last_error_func[32]

### 2.3 新增字段 (近年内核)

| 字段 | 说明 |
|------|------|
| `s_orphan_file_inum` | 孤儿文件 inode 号 (COMPAT ORPHAN_FILE) |
| `s_encoding` | 文件名编码 (UTF-8 12.1 = 1) |
| `s_encoding_flags` | 编码标志 |
| `s_def_resuid_hi` | 32-bit resuid 高16位 |
| `s_def_resgid_hi` | 32-bit resgid 高16位 |
| `s_first_error_errcode` | 首次错误 errno 代码 |
| `s_last_error_errcode` | 最近错误 errno 代码 |
| `s_reserved[93]` | 填充到块末尾 |

### 2.4 32-bit UID/GID 访问

```c
static inline int ext4_get_resuid(struct ext4_super_block *es)
{
	return le16_to_cpu(es->s_def_resuid) |
		le16_to_cpu(es->s_def_resuid_hi) << 16;
}
```

---

## 三、文件系统状态

```c
#define EXT4_VALID_FS		0x0001	/* Unmounted cleanly */
#define EXT4_ERROR_FS		0x0002	/* Errors detected */
#define EXT4_ORPHAN_FS		0x0004	/* Orphans being recovered */
#define EXT4_FC_REPLAY		0x0020	/* Fast commit replay ongoing */
```

---

## 四、错误代码 (ext4.h:1942-1958)

```c
#define EXT4_ERR_UNKNOWN	 1
#define EXT4_ERR_EIO		 2
#define EXT4_ERR_ENOMEM		 3
#define EXT4_ERR_EFSBADCRC	 4
#define EXT4_ERR_EFSCORRUPTED	 5
#define EXT4_ERR_ENOSPC		 6
#define EXT4_ERR_ENOKEY		 7
#define EXT4_ERR_EROFS		 8
#define EXT4_ERR_EFBIG		 9
#define EXT4_ERR_EEXIST		10
#define EXT4_ERR_ERANGE		11
#define EXT4_ERR_EOVERFLOW	12
#define EXT4_ERR_EBUSY		13
#define EXT4_ERR_ENOTDIR	14
#define EXT4_ERR_ENOTEMPTY	15
#define EXT4_ERR_ESHUTDOWN	16
#define EXT4_ERR_EFAULT		17
```

---

## 五、ext4_sb_info 关键字段 (ext4.h:1525-1800)

```c
struct ext4_sb_info {
	/* 基本参数 */
	unsigned long s_desc_size;
	unsigned long s_blocks_per_group;
	unsigned long s_clusters_per_group;
	unsigned long s_inodes_per_group;
	unsigned long s_itb_per_group;
	unsigned long s_gdb_count;
	unsigned long s_desc_per_block;
	ext4_group_t s_groups_count;

	/* bigalloc */
	unsigned int s_cluster_ratio;
	unsigned int s_cluster_bits;

	/* 挂载选项 */
	unsigned int s_mount_opt;
	unsigned int s_mount_opt2;
	unsigned long s_mount_flags;

	/* 孤儿文件 */
	struct ext4_orphan_info s_orphan_info;

	/* mballoc xarray (2025) */
	struct xarray *s_mb_avg_fragment_size;
	struct xarray *s_mb_largest_free_orders;
	unsigned int s_mb_nr_global_goals;

	/* Fast commit */
	struct list_head s_fc_q[2];
	struct list_head s_fc_dentry_q[2];
	struct mutex s_fc_lock;
	unsigned int s_fc_bytes;
	tid_t s_fc_ineligible_tid;

	/* 错误跟踪 */
	spinlock_t s_error_lock;
	int s_first_error_code;
	int s_last_error_code;

	/* Atomic write */
	unsigned int s_awu_min;
	unsigned int s_awu_max;

	/* Large folio */
	u16 s_min_folio_order;
	u16 s_max_folio_order;

	/* ES shrinker */
	struct shrinker *s_es_shrinker;
	struct list_head s_es_list;
	long s_es_nr_inode;

	/* Errseq tracking */
	errseq_t s_bdev_wb_err;
	spinlock_t s_bdev_wb_lock;
};
```

---

## 六、关键代码位置

| 功能 | 文件 |
|------|------|
| 结构定义 | fs/ext4/ext4.h:1332-1463 |
| 超级块读取/写入 | fs/ext4/super.c |
| 挂载选项解析 | fs/ext4/super.c |
| Mount API (fs_context) | fs/ext4/super.c |

---

## 七、深度代码解析

### 7.1 超级块加载流程

挂载时，超级块加载是最早的步骤之一。核心流程如下:

```c
// fs/ext4/super.c:5302-5325 (简化)
static int __ext4_fill_super(struct fs_context *fc, struct super_block *sb)
{
    struct ext4_super_block *es = NULL;
    struct ext4_sb_info *sbi = EXT4_SB(sb);
    
    // 步骤1: 从磁盘读取超级块
    err = ext4_load_super(sb, &logical_sb_block, silent);
    if (err)
        goto out_fail;
    
    es = sbi->s_es;  // 指向内存中的超级块副本
    
    // 步骤2: 初始化元数据校验和
    err = ext4_init_metadata_csum(sb, es);
    
    // 步骤3: 设置默认选项
    ext4_set_def_opts(sb, es);
    
    // 步骤4: 检查特性兼容性
    err = ext4_check_feature_compatibility(sb, es, silent);
    
    // 步骤5: 初始化块组元数据
    err = ext4_block_group_meta_init(sb, silent);
    
    // ... 后续初始化
}
```

**关键点**: `ext4_load_super()` 负责:
1. 计算超级块的物理位置 (通常在块1，偏移1024字节)
2. 读取超级块到 buffer_head
3. 验证 magic number (0xEF53)
4. 分配并填充 `ext4_sb_info`

### 7.2 超级块位置计算

```c
// fs/ext4/super.c (简化)
static int ext4_load_super(struct super_block *sb, ext4_fsblk_t *lsb,
                           int silent)
{
    // 超级块始终在 1024 字节偏移处开始
    // 对于 1K 块，在块 1
    // 对于 2K/4K 块，在块 0 的后 1024 字节
    
    *lsb = EXT4_MIN_BLOCK_SIZE / blocksize;  // 通常 = 1
    // ...
}
```

**布局示例 (4K 块)**:
```
块 0: [boot sector (512B)] [padding (512B)] [superblock (1024B)] [padding (2048B)]
块 1: [块组描述符表开始...]
```

### 7.3 64 位块计数访问

```c
// fs/ext4/ext4.h (简化)
static inline ext4_fsblk_t ext4_blocks_count(struct ext4_super_block *es)
{
    return ((ext4_fsblk_t)le32_to_cpu(es->s_blocks_count_hi) << 32) |
            le32_to_cpu(es->s_blocks_count_lo);
}

static inline void ext4_blocks_count_set(struct ext4_super_block *es,
                                         ext4_fsblk_t blk)
{
    es->s_blocks_count_lo = cpu_to_le32((u32)blk);
    es->s_blocks_count_hi = cpu_to_le32(blk >> 32);
}
```

**解析**: ext4 使用分离的 `_lo` 和 `_hi` 字段支持 64 位值，同时保持与 32 位 ext3 的二进制兼容性。只有当 `EXT4_FEATURE_INCOMPAT_64BIT` 启用时，`_hi` 字段才有效。

### 7.4 超级块校验和计算

```c
// fs/ext4/super.c
static __le32 ext4_superblock_csum(struct super_block *sb,
                                   struct ext4_super_block *es)
{
    struct ext4_sb_info *sbi = EXT4_SB(sb);
    int offset = offsetof(struct ext4_super_block, s_checksum);
    __u32 csum;

    // crc32c(checksum_seed, superblock_data_before_checksum_field)
    csum = ext4_chksum(sbi, ~0, (char *)es, offset);

    return cpu_to_le32(csum);
}
```

**关键点**: 
- 校验和覆盖 `s_checksum` 字段之前的所有数据
- 使用 crc32c 算法
- 只有 `EXT4_FEATURE_RO_COMPAT_METADATA_CSUM` 启用时才计算

### 7.5 挂载选项解析 (fs_context API)

```c
// fs/ext4/super.c:125-130
static const struct fs_context_operations ext4_context_ops = {
    .parse_param    = ext4_parse_param,      // 解析单个参数
    .get_tree       = ext4_get_tree,         // 获取文件系统树
    .reconfigure    = ext4_reconfigure,      // 重新挂载
    .free           = ext4_fc_free,          // 释放上下文
};
```

**参数解析示例**:

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
    // ...
    }
}
```

### 7.6 错误记录机制

```c
// fs/ext4/super.c (简化)
void __ext4_error(struct super_block *sb, const char *function,
                  unsigned int line, bool force_ro, int error,
                  __u64 block, const char *fmt, ...)
{
    struct ext4_sb_info *sbi = EXT4_SB(sb);
    
    // 记录第一次错误
    if (unlikely(!sbi->s_first_error_code)) {
        sbi->s_first_error_code = error;
        sbi->s_first_error_ino = ...; 
        sbi->s_first_error_block = block;
        sbi->s_first_error_line = line;
        strscpy(sbi->s_first_error_func, function, ...);
    }
    
    // 总是更新最后一次错误
    sbi->s_last_error_code = error;
    // ...
    
    // 触发错误处理策略
    ext4_handle_error(sb, force_ro, error, ...);
}
```

**错误处理策略** (由 `s_errors` 字段或挂载选项控制):
- `errors=continue`: 继续运行，记录错误
- `errors=remount-ro`: 重新挂载为只读
- `errors=panic`: 触发内核 panic

---

## 八、超级块备份策略

### 8.1 sparse_super 特性

传统上，超级块和块组描述符表在每个块组的开头都有备份。`sparse_super` 特性只在块组 0、1 和 3、5、7 的幂次方块组保留备份:

```
块组 0:  备份 (总是)
块组 1:  备份 (总是)
块组 3:  备份 (3^1)
块组 5:  备份 (5^1)
块组 7:  备份 (7^1)
块组 9:  无备份
块组 25: 备份 (5^2)
块组 27: 备份 (3^3)
块组 49: 备份 (7^2)
...
```

### 8.2 sparse_super2 特性

`sparse_super2` 进一步减少备份，只在 `s_backup_bgs[2]` 指定的两个块组保留备份:

```c
// ext4_super_block
__le32  s_backup_bgs[2];  // 存储备份位置的块组号
```

---

## 九、时间戳扩展

### 9.1 Y2038 问题解决

ext4 使用额外的 `_hi` 字节扩展时间戳范围:

```c
// 超级块中的时间戳扩展
__u8    s_wtime_hi;
__u8    s_mtime_hi;
__u8    s_mkfs_time_hi;
__u8    s_lastcheck_hi;
__u8    s_first_error_time_hi;
__u8    s_last_error_time_hi;
```

**解码逻辑**:

```c
// 完整的 40 位时间戳 (支持到 2446 年)
time64_t full_time = (time64_t)time_lo | ((time64_t)time_hi << 32);
```

---

## 十、参考文献与资源

### 官方文档
1. **Linux 内核文档**: [Documentation/filesystems/ext4/](https://www.kernel.org/doc/html/latest/filesystems/ext4/globals.html)
2. **ext4 磁盘布局**: https://www.kernel.org/doc/html/latest/filesystems/ext4/ondisk/super.html

### 学术论文
3. **"The new ext4 filesystem: current status and future plans"** - Mathur et al., Ottawa Linux Symposium 2007
   - 首次详细介绍 ext4 超级块扩展设计
4. **"Design and Implementation of the Second Extended Filesystem"** - Card et al., 1994
   - ext2 超级块原始设计

### LWN.net 文章
5. **"Ext4 and the Future of Timestamps"** - https://lwn.net/Articles/804382/ (2020)
   - Y2038 问题和时间戳扩展
6. **"Ext4 metadata checksums"** - https://lwn.net/Articles/469805/ (2011)
   - 超级块校验和实现

### 关键 Commit
7. **超级块校验和**: `9aa5d32b` "ext4: add metadata checksum to the superblock" (2012-04)
8. **64 位支持**: `bd81d8ee` "ext4: Add support for 64 bit block numbers" (2006-10)
9. **fs_context API**: `e6e268cb` "ext4: Convert ext4 to use the new mount API" (2021-02)
10. **时间戳扩展**: `6873fa0d` "ext4: add extra timestamp bits" (2019-04)

### 调试工具
11. **dumpe2fs**: 查看超级块详情
    ```bash
    # 查看所有超级块字段
    dumpe2fs -h /dev/sda1
    
    # 输出示例:
    # Filesystem magic number:  0xEF53
    # Filesystem features:      has_journal ext_attr resize_inode ...
    # Block count:              26214400
    # Block size:               4096
    ```

12. **tune2fs**: 修改超级块参数
    ```bash
    # 查看特性
    tune2fs -l /dev/sda1 | grep "Filesystem features"
    
    # 启用特性
    tune2fs -O metadata_csum /dev/sda1
    
    # 设置挂载计数
    tune2fs -C 0 /dev/sda1
    ```

13. **debugfs**: 原始超级块访问
    ```bash
    debugfs /dev/sda1
    debugfs: show_super_stats
    # 显示详细统计信息
    ```

### 邮件列表讨论
14. **超级块扩展讨论**: https://lore.kernel.org/linux-ext4/ 搜索 "superblock"

---

## 十一、常见问题与陷阱

### Q1: 为什么 `s_reserved[93]` 而不是 96 或其他数字？

**A**: 超级块必须精确占用一个块的开始 1024 字节。计算如下:

```
超级块结构大小 = 1024 字节
已定义字段大小 = 1024 - 4 - 93*4 = 1024 - 4 - 372 = 648 字节
s_checksum (4字节) + s_reserved[93] (372字节) = 376 字节
648 + 376 = 1024 字节 ✓
```

每次添加新字段，都要从 `s_reserved` 数组中"借"空间。

### Q2: `s_state` 和 `s_mount_state` 有什么区别？

**A**: 
- `s_state` (磁盘): 存储在超级块中，表示上次卸载时的状态
- `s_mount_state` (内存): 在 `ext4_sb_info` 中，表示当前运行时状态

挂载时会检查 `s_state`:
- `EXT4_VALID_FS`: 正常卸载，无需恢复
- `EXT4_ERROR_FS`: 检测到错误，建议 e2fsck
- `EXT4_ORPHAN_FS`: 存在孤儿 inode，需要清理

### Q3: 如何安全地在线修改超级块？

**A**: ext4 使用 JBD2 事务保护超级块修改:

```c
// 正确方式
handle_t *handle = ext4_journal_start_sb(sb, EXT4_HT_MISC, 1);
BUFFER_TRACE(sbh, "get_write_access");
ext4_superblock_csum_set(sb);
err = ext4_handle_dirty_metadata(handle, NULL, sbh);
ext4_journal_stop(handle);

// 立即同步 (可选)
sync_dirty_buffer(sbh);
```

### Q4: `s_kbytes_written` 的精度问题

**A**: 这个字段记录写入的总字节数，但:
- 只在卸载时更新到磁盘
- 崩溃后可能丢失最近的写入量
- 某些内核版本有溢出 bug

不应依赖此字段进行精确统计，它主要用于 SSD 寿命估算。
