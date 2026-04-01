---
name: "ext4-inode"
description: "ext4文件系统Inode结构专家。当用户询问ext4 inode结构、i_mode、i_block、extent树、inode分配时调用此技能。"
---

# ext4 Inode

## 〇、为什么需要这个机制？

为什么需要 inode？Unix 文件系统的核心设计哲学是"数据与元数据分离"。inode 存储文件的元数据（权限、大小、时间戳、数据位置），而不存储文件名。这使得硬链接成为可能（多个目录项指向同一个 inode）。ext4 的 inode 从 ext2 的 128 字节扩展到 256 字节，增加了纳秒时间戳、项目 ID、校验和等字段。

inode 的设计直接影响文件系统的性能和可靠性。早期的 ext2 inode 只有 15 个直接/间接块指针，大文件需要多级间接块查找，性能差且碎片严重。ext4 引入 extent 树后，i_block 区域被重新解释为 extent 根节点，大幅提升了大文件的访问效率。

没有 inode，文件系统就无法描述"文件是什么"——权限、所有者、数据位置等核心信息都依赖 inode 存储。

---

## 一、磁盘 Inode 结构 (verified: ext4.h:795-854)

```c
struct ext4_inode {
	__le16	i_mode;		/* File mode */
	__le16	i_uid;		/* Low 16 bits of Owner Uid */
	__le32	i_size_lo;	/* Size in bytes */
	__le32	i_atime;	/* Access time */
	__le32	i_ctime;	/* Inode Change time */
	__le32	i_mtime;	/* Modification time */
	__le32	i_dtime;	/* Deletion Time */
	__le16	i_gid;		/* Low 16 bits of Group Id */
	__le16	i_links_count;	/* Links count */
	__le32	i_blocks_lo;	/* Blocks count */
	__le32	i_flags;	/* File flags */
	union {
		struct { __le32  l_i_version; } linux1;
		struct { __u32  h_i_translator; } hurd1;
		struct { __u32  m_i_reserved1; } masix1;
	} osd1;
	__le32	i_block[EXT4_N_BLOCKS];/* Pointers to blocks */
	__le32	i_generation;	/* File version (for NFS) */
	__le32	i_file_acl_lo;	/* File ACL */
	__le32	i_size_high;
	__le32	i_obso_faddr;	/* Obsoleted fragment address */
	union {
		struct {
			__le16	l_i_blocks_high;
			__le16	l_i_file_acl_high;
			__le16	l_i_uid_high;
			__le16	l_i_gid_high;
			__le16	l_i_checksum_lo;
			__le16	l_i_reserved;
		} linux2;
		struct {
			__le16	h_i_reserved1;
			__u16	h_i_mode_high;
			__u16	h_i_uid_high;
			__u16	h_i_gid_high;
			__u32	h_i_author;
		} hurd2;
		struct {
			__le16	h_i_reserved1;
			__le16	m_i_file_acl_high;
			__u32	m_i_reserved2[2];
		} masix2;
	} osd2;
	__le16	i_extra_isize;
	__le16	i_checksum_hi;
	__le32  i_ctime_extra;
	__le32  i_mtime_extra;
	__le32  i_atime_extra;
	__le32  i_crtime;
	__le32  i_crtime_extra;
	__le32  i_version_hi;
	__le32	i_projid;	/* Project ID — LAST field, NO i_pad */
};
```

**关键**: `i_projid` 是最后一个字段，**没有** `i_pad` 或填充。

---

## 二、i_block[15] 两种用法

### 间接块模式

```
i_block[0-11]:  直接块指针 (12个)
i_block[12]:    一级间接块指针
i_block[13]:    二级间接块指针
i_block[14]:    三级间接块指针
```

### Extent 模式 (i_flags & EXT4_EXTENTS_FL)

```c
/* i_block 前12字节存 ext4_extent_header */
struct ext4_extent_header {
	__le16	eh_magic;	/* 0xf30a */
	__le16	eh_entries;
	__le16	eh_max;
	__le16	eh_depth;
	__le32	eh_generation;
};
/* 后跟 ext4_extent 或 ext4_extent_idx 数组 */
```

---

## 三、Inode 标志 (i_flags, verified: ext4.h:482-514)

```c
#define EXT4_SECRM_FL			0x00000001
#define EXT4_UNRM_FL			0x00000002
#define EXT4_COMPR_FL			0x00000004
#define EXT4_SYNC_FL			0x00000008
#define EXT4_IMMUTABLE_FL		0x00000010
#define EXT4_APPEND_FL			0x00000020
#define EXT4_NODUMP_FL			0x00000040
#define EXT4_NOATIME_FL			0x00000080
#define EXT4_DIRTY_FL			0x00000100
#define EXT4_COMPRBLK_FL		0x00000200
#define EXT4_NOCOMPR_FL			0x00000400
#define EXT4_ENCRYPT_FL			0x00000800
#define EXT4_INDEX_FL			0x00001000
#define EXT4_IMAGIC_FL			0x00002000
#define EXT4_JOURNAL_DATA_FL		0x00004000
#define EXT4_NOTAIL_FL			0x00008000
#define EXT4_DIRSYNC_FL			0x00010000
#define EXT4_TOPDIR_FL			0x00020000
#define EXT4_HUGE_FILE_FL               0x00040000
#define EXT4_EXTENTS_FL			0x00080000
#define EXT4_VERITY_FL			0x00100000
#define EXT4_EA_INODE_FL	        0x00200000
#define EXT4_DAX_FL			0x02000000
#define EXT4_INLINE_DATA_FL		0x10000000
#define EXT4_PROJINHERIT_FL		0x20000000
#define EXT4_CASEFOLD_FL		0x40000000
#define EXT4_RESERVED_FL		0x80000000
```

---

## 四、Inode 状态标志 (ext4.h:1963-1977)

```c
enum {
	EXT4_STATE_NEW,
	EXT4_STATE_XATTR,
	EXT4_STATE_NO_EXPAND,
	EXT4_STATE_DA_ALLOC_CLOSE,
	EXT4_STATE_EXT_MIGRATE,
	EXT4_STATE_NEWENTRY,
	EXT4_STATE_MAY_INLINE_DATA,
	EXT4_STATE_EXT_PRECACHED,
	EXT4_STATE_LUSTRE_EA_INODE,
	EXT4_STATE_VERITY_IN_PROGRESS,
	EXT4_STATE_FC_COMMITTING,
	EXT4_STATE_FC_FLUSHING_DATA,
	EXT4_STATE_ORPHAN_FILE,
};
```

---

## 五、内存 Inode (ext4_inode_info, ext4.h:1031-1199)

```c
struct ext4_inode_info {
	__le32	i_data[15];
	__u32	i_dtime;
	ext4_fsblk_t	i_file_acl;
	ext4_group_t	i_block_group;
	ext4_lblk_t	i_dir_start_lookup;
	unsigned long	i_flags;
	struct rw_semaphore xattr_sem;
	union {
		struct list_head i_orphan;
		unsigned int i_orphan_idx;
	};
	/* Fast commit */
	struct list_head i_fc_dilist;
	struct list_head i_fc_list;
	ext4_lblk_t i_fc_lblk_start;
	ext4_lblk_t i_fc_lblk_len;
	spinlock_t i_raw_lock;
	wait_queue_head_t i_fc_wait;
	spinlock_t i_fc_lock;
	loff_t	i_disksize;
	struct rw_semaphore i_data_sem;
	struct inode vfs_inode;
	struct jbd2_inode *jinode;
	struct timespec64 i_crtime;
	/* mballoc */
	atomic_t i_prealloc_active;
	unsigned int i_reserved_data_blocks;
	struct rb_root i_prealloc_node;
	rwlock_t i_prealloc_lock;
	/* extents status tree */
	struct ext4_es_tree i_es_tree;
	rwlock_t i_es_lock;
	struct list_head i_es_list;
	unsigned int i_es_all_nr;
	unsigned int i_es_shk_nr;
	ext4_lblk_t i_es_shrink_lblk;
	u64 i_es_seq;		/* Change counter for extents */
	ext4_group_t	i_last_alloc_group;
	struct ext4_pending_tree i_pending_tree;
	__u16 i_extra_isize;
	u16 i_inline_off;
	u16 i_inline_size;
	spinlock_t i_block_reservation_lock;
	spinlock_t i_completed_io_lock;
	struct list_head i_rsv_conversion_list;
	struct work_struct i_rsv_conversion_work;
	tid_t i_sync_tid;
	tid_t i_datasync_tid;
	__u32 i_csum_seed;
	kprojid_t i_projid;
#ifdef CONFIG_FS_ENCRYPTION
	struct fscrypt_inode_info *i_crypt_info;
#endif
};
```

---

## 六、Inode 校验和

```c
/* osd2.linux2.l_i_checksum_lo + i_checksum_hi 组成完整 CRC32c */
/* 种子: uuid + inode_number + inode_data */
```

---

## 七、特殊 Inode 号 (ext4.h:314-324)

| Inode | 用途 |
|-------|------|
| 1 | Bad blocks |
| 2 | Root directory |
| 3 | User quota |
| 4 | Group quota |
| 5 | Boot loader |
| 6 | Undelete directory |
| 7 | Resize inode |
| 8 | Journal inode |

---

## 八、关键代码位置

| 功能 | 文件 |
|------|------|
| 磁盘结构 | fs/ext4/ext4.h:795-854 |
| 内存结构 | fs/ext4/ext4.h:1031-1199 |
| Inode 操作 | fs/ext4/inode.c |
| Inode 分配 | fs/ext4/ialloc.c |
