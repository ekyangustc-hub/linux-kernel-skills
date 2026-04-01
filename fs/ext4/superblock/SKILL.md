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
