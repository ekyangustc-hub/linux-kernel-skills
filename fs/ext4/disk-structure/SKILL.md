---
name: "ext4-disk-structure"
description: "ext4文件系统静态磁盘结构专家。当用户询问ext4磁盘布局、超级块、块组描述符、inode结构、extent树、目录项、特性耦合(flex_bg/sparse_bigalloc等)时调用此技能。"
---

# ext4 静态磁盘结构

本技能描述 ext4 文件系统的静态磁盘布局，所有结构定义已验证自内核源码。

## 〇、为什么需要这个机制？

为什么需要了解 ext4 的磁盘布局？因为 ext4 是一个精心设计的系统，每个结构的存在都有其历史原因。超级块记录全局参数，块组描述符管理局部空间，inode 描述文件元数据，extent 树映射数据位置。理解这些结构的必要性，才能理解为什么 ext4 比 ext2/ext3 更可靠。

ext4 的磁盘布局经历了三代演进：ext2 奠定了基本框架（超级块+块组+inode表），ext3 加入了日志支持，ext4 引入了 extent 树、flex_bg、bigalloc 等特性。每个新特性的引入都伴随着磁盘格式的变更，通过特性标志位控制兼容性。

没有清晰的磁盘结构认知，就无法理解文件系统损坏的根因、无法正确解读 e2fsck 的输出、也无法评估新特性对可靠性的影响。

---

## 一、整体布局

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              Block Group 0                                  │
│  ┌─────────┬─────────────┬───────────────┬───────────────┬───────────────┐ │
│  │Superblock│ Group Desc  │  Block Bitmap │ Inode Bitmap  │  Inode Table  │ │
│  │ (1024B) │  Table      │               │               │               │ │
│  └─────────┴─────────────┴───────────────┴───────────────┴───────────────┘ │
│                                    Data Blocks                              │
└────────────────────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────────────────────┐
│                              Block Group 1+                                 │
│  ┌───────────────┬───────────────┬───────────────┬───────────────────────┐ │
│  │  Block Bitmap │ Inode Bitmap  │  Inode Table  │     Data Blocks       │ │
│  └───────────────┴───────────────┴───────────────┴───────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────┘
```

### 核心结构关系

```
ext4_super_block ──► ext4_group_desc[] ──┬─► Block Bitmap
                                         ├─► Inode Bitmap
                                         └─► Inode Table ──► ext4_inode ──► data/extent/xattr
```

---

## 二、特性标志位 (verified: ext4.h:2072-2128)

### COMPAT (老内核可读可写)

```c
#define EXT4_FEATURE_COMPAT_DIR_PREALLOC    0x0001
#define EXT4_FEATURE_COMPAT_IMAGIC_INODES   0x0002
#define EXT4_FEATURE_COMPAT_HAS_JOURNAL     0x0004
#define EXT4_FEATURE_COMPAT_EXT_ATTR        0x0008
#define EXT4_FEATURE_COMPAT_RESIZE_INODE    0x0010
#define EXT4_FEATURE_COMPAT_DIR_INDEX       0x0020
#define EXT4_FEATURE_COMPAT_SPARSE_SUPER2   0x0200
#define EXT4_FEATURE_COMPAT_FAST_COMMIT     0x0400  /* COMPAT, not INCOMPAT */
#define EXT4_FEATURE_COMPAT_STABLE_INODES   0x0800
#define EXT4_FEATURE_COMPAT_ORPHAN_FILE     0x1000  /* COMPAT, not INCOMPAT */
```

### RO_COMPAT (老内核只读挂载)

```c
#define EXT4_FEATURE_RO_COMPAT_SPARSE_SUPER 0x0001
#define EXT4_FEATURE_RO_COMPAT_LARGE_FILE   0x0002
#define EXT4_FEATURE_RO_COMPAT_BTREE_DIR    0x0004
#define EXT4_FEATURE_RO_COMPAT_HUGE_FILE    0x0008
#define EXT4_FEATURE_RO_COMPAT_GDT_CSUM     0x0010
#define EXT4_FEATURE_RO_COMPAT_DIR_NLINK    0x0020
#define EXT4_FEATURE_RO_COMPAT_EXTRA_ISIZE  0x0040
#define EXT4_FEATURE_RO_COMPAT_QUOTA        0x0100
#define EXT4_FEATURE_RO_COMPAT_BIGALLOC     0x0200
#define EXT4_FEATURE_RO_COMPAT_METADATA_CSUM 0x0400
#define EXT4_FEATURE_RO_COMPAT_READONLY     0x1000
#define EXT4_FEATURE_RO_COMPAT_PROJECT      0x2000
#define EXT4_FEATURE_RO_COMPAT_VERITY       0x8000  /* RO_COMPAT, not INCOMPAT */
#define EXT4_FEATURE_RO_COMPAT_ORPHAN_PRESENT 0x10000
```

### INCOMPAT (老内核无法挂载)

```c
#define EXT4_FEATURE_INCOMPAT_COMPRESSION   0x0001
#define EXT4_FEATURE_INCOMPAT_FILETYPE      0x0002
#define EXT4_FEATURE_INCOMPAT_RECOVER       0x0004
#define EXT4_FEATURE_INCOMPAT_JOURNAL_DEV   0x0008
#define EXT4_FEATURE_INCOMPAT_META_BG       0x0010
#define EXT4_FEATURE_INCOMPAT_EXTENTS       0x0040
#define EXT4_FEATURE_INCOMPAT_64BIT         0x0080
#define EXT4_FEATURE_INCOMPAT_MMP           0x0100
#define EXT4_FEATURE_INCOMPAT_FLEX_BG       0x0200
#define EXT4_FEATURE_INCOMPAT_EA_INODE      0x0400
#define EXT4_FEATURE_INCOMPAT_DIRDATA       0x1000
#define EXT4_FEATURE_INCOMPAT_CSUM_SEED     0x2000
#define EXT4_FEATURE_INCOMPAT_LARGEDIR      0x4000
#define EXT4_FEATURE_INCOMPAT_INLINE_DATA   0x8000
#define EXT4_FEATURE_INCOMPAT_ENCRYPT       0x10000
#define EXT4_FEATURE_INCOMPAT_CASEFOLD      0x20000
```

### EXT4_FEATURE_*_SUPP (内核支持集合, ext4.h:2243-2270)

```c
EXT4_FEATURE_COMPAT_SUPP    = EXT_ATTR | ORPHAN_FILE
EXT4_FEATURE_INCOMPAT_SUPP  = FILETYPE | RECOVER | META_BG | EXTENTS | 64BIT |
                              FLEX_BG | EA_INODE | MMP | INLINE_DATA |
                              ENCRYPT | CASEFOLD | CSUM_SEED | LARGEDIR
EXT4_FEATURE_RO_COMPAT_SUPP = SPARSE_SUPER | LARGE_FILE | GDT_CSUM | DIR_NLINK |
                              EXTRA_ISIZE | BTREE_DIR | HUGE_FILE | BIGALLOC |
                              METADATA_CSUM | QUOTA | PROJECT | VERITY | ORPHAN_PRESENT
```

---

## 三、ext4_inode (verified: ext4.h:795-854)

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
	__le32	i_projid;	/* Project ID — LAST field, NO i_pad after it */
};
```

**注意**: `i_projid` 是最后一个字段，**没有** `i_pad`。

---

## 四、ext4_group_desc (verified: ext4.h:403-428)

```c
struct ext4_group_desc {
	__le32	bg_block_bitmap_lo;
	__le32	bg_inode_bitmap_lo;
	__le32	bg_inode_table_lo;
	__le16	bg_free_blocks_count_lo;
	__le16	bg_free_inodes_count_lo;
	__le16	bg_used_dirs_count_lo;
	__le16	bg_flags;
	__le32  bg_exclude_bitmap_lo;
	__le16  bg_block_bitmap_csum_lo;
	__le16  bg_inode_bitmap_csum_lo;
	__le16  bg_itable_unused_lo;
	__le16  bg_checksum;
	__le32	bg_block_bitmap_hi;
	__le32	bg_inode_bitmap_hi;
	__le32	bg_inode_table_hi;
	__le16	bg_free_blocks_count_hi;
	__le16	bg_free_inodes_count_hi;
	__le16	bg_used_dirs_count_hi;
	__le16  bg_itable_unused_hi;
	__le32  bg_exclude_bitmap_hi;
	__le16  bg_block_bitmap_csum_hi;
	__le16  bg_inode_bitmap_csum_hi;
	__u32   bg_reserved;
};
```

---

## 五、Extent 树 (verified: ext4_extents.h:48-87)

```c
struct ext4_extent_tail {
	__le32	et_checksum;	/* crc32c(uuid+inum+extent_block) */
};

struct ext4_extent {
	__le32	ee_block;
	__le16	ee_len;
	__le16	ee_start_hi;
	__le32	ee_start_lo;
};

struct ext4_extent_idx {
	__le32	ei_block;
	__le32	ei_leaf_lo;
	__le16	ei_leaf_hi;
	__u16	ei_unused;
};

struct ext4_extent_header {
	__le16	eh_magic;	/* 0xf30a */
	__le16	eh_entries;
	__le16	eh_max;
	__le16	eh_depth;
	__le32	eh_generation;
};

#define EXT4_MAX_EXTENT_DEPTH 5	/* NOT 3 */
```

**注意**: `ext4_extent_tail` 存在于每个非 inode 的 extent 块尾部。`EXT4_MAX_EXTENT_DEPTH = 5`。

---

## 六、Xattr (verified: xattr.h:30-51)

```c
struct ext4_xattr_header {
	__le32	h_magic;
	__le32	h_refcount;
	__le32	h_blocks;
	__le32	h_hash;
	__le32	h_checksum;
	__u32	h_reserved[3];	/* 5 fields + 3 reserved */
};

struct ext4_xattr_entry {
	__u8	e_name_len;
	__u8	e_name_index;
	__le16	e_value_offs;
	__le32	e_value_inum;	/* NOT e_value_block */
	__le32	e_value_size;
	__le32	e_hash;
	char	e_name[];
};
```

---

## 七、Directory (verified: ext4.h:2396-2454)

```c
struct ext4_dir_entry {
	__le32	inode;
	__le16	rec_len;
	__le16	name_len;
	char	name[EXT4_NAME_LEN];
};

struct ext4_dir_entry_2 {
	__le32	inode;
	__le16	rec_len;
	__u8	name_len;
	__u8	file_type;
	char	name[EXT4_NAME_LEN];
};

/* For encrypted+casefolded directories */
struct ext4_dir_entry_hash {
	__le32 hash;
	__le32 minor_hash;
};

struct ext4_dir_entry_tail {
	__le32	det_reserved_zero1;
	__le16	det_rec_len;		/* 12 */
	__u8	det_reserved_zero2;
	__u8	det_reserved_ft;	/* 0xDE */
	__le32	det_checksum;
};

#define EXT4_DIRENT_HASHES(entry) \
	((struct ext4_dir_entry_hash *) \
		(((void *)(entry)) + \
		((8 + (entry)->name_len + EXT4_DIR_ROUND) & ~EXT4_DIR_ROUND)))
```

---

## 八、ext4_super_block 关键字段 (verified: ext4.h:1332-1463)

```c
struct ext4_super_block {
	/* ... standard fields ... */
	__le32  s_orphan_file_inum;	/* Inode for tracking orphan inodes */
	__le16	s_def_resuid_hi;
	__le16	s_def_resgid_hi;
	__le16  s_encoding;
	__le16  s_encoding_flags;
	__le32	s_reserved[93];		/* Padding to the end of the block */
	__le32	s_checksum;		/* crc32c(superblock) */
};
```

**注意**: `s_reserved[93]` (NOT 94/98)，`s_orphan_file_inum` 存在，`s_encoding`/`s_encoding_flags` 存在，`s_def_resuid_hi`/`s_def_resgid_hi` 存在。

---

## 九、特性耦合关系

### flex_bg + bigalloc
- bigalloc (RO_COMPAT) 要求 extents (INCOMPAT)
- flex_bg 改变元数据布局，位图/inode 表可跨组

### metadata_csum
- metadata_csum (RO_COMPAT) 是 gdt_csum 的超集
- 两者互斥: metadata_csum 设置时 gdt_csum 不设置

### encrypt + casefold
- 同时启用时，目录项需要 `ext4_dir_entry_hash` 存储哈希值
- 哈希版本使用 DX_HASH_SIPHASH (6)

---

## 十、Sysfs / Proc

```
/sys/fs/ext4/<dev>/err_report_sec    — 错误报告定时器周期
/proc/fs/ext4/<dev>/fc_info          — Fast commit 统计
/sys/fs/ext4/features/blocksize_gt_pagesize  — Large block size 支持标志
```

---

## 十一、关键代码位置

| 结构 | 文件 | 行号 |
|------|------|------|
| ext4_super_block | fs/ext4/ext4.h | 1332-1463 |
| ext4_group_desc | fs/ext4/ext4.h | 403-428 |
| ext4_inode | fs/ext4/ext4.h | 795-854 |
| ext4_extent_header | fs/ext4/ext4_extents.h | 78-84 |
| ext4_extent | fs/ext4/ext4_extents.h | 56-61 |
| ext4_extent_idx | fs/ext4/ext4_extents.h | 67-73 |
| ext4_extent_tail | fs/ext4/ext4_extents.h | 48-50 |
| ext4_xattr_header | fs/ext4/xattr.h | 30-37 |
| ext4_xattr_entry | fs/ext4/xattr.h | 43-51 |
| ext4_dir_entry_2 | fs/ext4/ext4.h | 2420-2426 |
| ext4_dir_entry_hash | fs/ext4/ext4.h | 2409-2412 |
| Feature flags | fs/ext4/ext4.h | 2072-2128 |
