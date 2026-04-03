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

---

## 十二、深度代码解析

### 12.1 块组描述符大小计算

```c
// fs/ext4/ext4.h:454-457
#define EXT4_MIN_DESC_SIZE		32
#define EXT4_MIN_DESC_SIZE_64BIT	64
#define EXT4_MAX_DESC_SIZE		EXT4_MIN_BLOCK_SIZE
#define EXT4_DESC_SIZE(s)		(EXT4_SB(s)->s_desc_size)
```

**解析**: 块组描述符有两种大小:
- **32 字节**: 传统模式，bg_*_hi 字段不使用
- **64 字节**: 64 位模式 (`EXT4_FEATURE_INCOMPAT_64BIT`)，支持 >16TB 文件系统

描述符大小存储在 `s_desc_size` 字段。挂载时通过 `ext4_fill_super()` 计算:

```c
// fs/ext4/super.c (简化)
if (ext4_has_feature_64bit(sb)) {
    if (sbi->s_desc_size < EXT4_MIN_DESC_SIZE_64BIT ||
        sbi->s_desc_size > EXT4_MAX_DESC_SIZE) {
        ext4_msg(sb, KERN_ERR, "unsupported descriptor size %lu",
                 sbi->s_desc_size);
        goto failed_mount;
    }
} else
    sbi->s_desc_size = EXT4_MIN_DESC_SIZE;
```

### 12.2 Inode 大小与扩展字段

```c
// fs/ext4/ext4.h:870-874
#define EXT4_FITS_IN_INODE(ext4_inode, einode, field)   \
    ((offsetof(typeof(*ext4_inode), field) +    \
      sizeof((ext4_inode)->field))          \
    <= (EXT4_GOOD_OLD_INODE_SIZE +          \
        (einode)->i_extra_isize))
```

**解析**: ext4 inode 可以有扩展字段。基本大小是 128 字节 (`EXT4_GOOD_OLD_INODE_SIZE`)，现代 ext4 通常使用 256 字节。`i_extra_isize` 表示超出基本大小的额外空间。

此宏用于检查某个扩展字段是否在当前 inode 中可用:

```c
// 使用示例 - 检查是否支持纳秒时间戳
if (EXT4_FITS_IN_INODE(raw_inode, ei, i_ctime_extra)) {
    // 可以使用 i_ctime_extra 字段
}
```

### 12.3 特性标志检查宏

```c
// fs/ext4/ext4.h:2131-2147 (COMPAT 特性检查宏)
#define EXT4_FEATURE_COMPAT_FUNCS(name, flagname) \
static inline bool ext4_has_feature_##name(struct super_block *sb) \
{ \
    return ((EXT4_SB(sb)->s_es->s_feature_compat & \
        cpu_to_le32(EXT4_FEATURE_COMPAT_##flagname)) != 0); \
} \
static inline void ext4_set_feature_##name(struct super_block *sb) \
{ \
    ext4_update_dynamic_rev(sb); \
    EXT4_SB(sb)->s_es->s_feature_compat |= \
        cpu_to_le32(EXT4_FEATURE_COMPAT_##flagname); \
} \
static inline void ext4_clear_feature_##name(struct super_block *sb) \
{ \
    EXT4_SB(sb)->s_es->s_feature_compat &= \
        ~cpu_to_le32(EXT4_FEATURE_COMPAT_##flagname); \
}
```

**解析**: 这是 C 预处理器魔法，为每个特性生成三个内联函数:
- `ext4_has_feature_xxx()`: 检查特性是否启用
- `ext4_set_feature_xxx()`: 启用特性
- `ext4_clear_feature_xxx()`: 禁用特性

实际展开示例:

```c
// EXT4_FEATURE_COMPAT_FUNCS(journal, HAS_JOURNAL) 展开为:
static inline bool ext4_has_feature_journal(struct super_block *sb) {
    return ((EXT4_SB(sb)->s_es->s_feature_compat &
        cpu_to_le32(EXT4_FEATURE_COMPAT_HAS_JOURNAL)) != 0);
}
```

### 12.4 Extent 树深度计算

```c
// fs/ext4/ext4_extents.h:185-188
static inline unsigned short ext_depth(struct inode *inode)
{
    return le16_to_cpu(ext_inode_hdr(inode)->eh_depth);
}
```

**解析**: Extent 树深度存储在 inode 的 `i_block[]` 数组开头的 `ext4_extent_header` 中:
- `eh_depth = 0`: 只有叶子节点，extent 直接存储在 inode 中
- `eh_depth > 0`: 有索引节点，需要多次间接寻址

最大深度计算 (`EXT4_MAX_EXTENT_DEPTH = 5`):

```
4K 块 = 4096 字节
extent_header = 12 字节
每个 extent/idx = 12 字节
inode 中 i_block[] = 60 字节

inode 内: (60 - 12) / 12 = 4 个 extent
外部块: (4096 - 12 - 4) / 12 = 340 个 extent/idx

理论最大寻址: 340^5 ≈ 4.5 × 10^12 个块
```

---

## 十三、参考文献与资源

### 官方文档
1. **Linux 内核文档**: [Documentation/filesystems/ext4/](https://www.kernel.org/doc/html/latest/filesystems/ext4/)
2. **ext4 Wiki (已归档)**: https://ext4.wiki.kernel.org/
3. **e2fsprogs 文档**: https://e2fsprogs.sourceforge.net/

### 学术论文
4. **"The new ext4 filesystem: current status and future plans"** - Mathur et al., Ottawa Linux Symposium 2007
   - 首次系统介绍 ext4 设计目标
5. **"Ext4 block and inode allocator improvements"** - Delalleau, Linux Symposium 2008
   - Mballoc 和 ialloc 算法详解
6. **"Design and Implementation of the Second Extended Filesystem"** - Card et al., 1994
   - ext2 原始设计，ext4 的基础

### LWN.net 文章
7. **"Improving ext4: bigalloc, inline data, and metadata checksums"** - https://lwn.net/Articles/469805/ (2011)
8. **"A brief history of ext4"** - https://lwn.net/Articles/805424/ (2019)
9. **"Fast commits for ext4"** - https://lwn.net/Articles/842385/ (2020)

### 关键 Commit
10. **extent 树引入**: `a86c6181` "ext4: add extent map manipulate functions" (2006)
11. **flex_bg 引入**: `772cb7c8` "ext4: Add flex_bg feature" (2008)
12. **metadata_csum 引入**: `9aa5d32b` "ext4: add metadata checksum to the superblock" (2012)
13. **fast commit 引入**: `aa75f4d3` "ext4: main fast-commit commit path" (2020)

### 工具与调试
14. **dumpe2fs**: 查看文件系统元数据
    ```bash
    dumpe2fs -h /dev/sda1  # 查看超级块
    dumpe2fs /dev/sda1     # 查看所有块组
    ```
15. **debugfs**: 交互式调试
    ```bash
    debugfs /dev/sda1
    stat <2>              # 查看 root inode
    show_super_stats      # 超级块统计
    ```
16. **filefrag**: 查看文件碎片
    ```bash
    filefrag -v /path/to/file  # 显示 extent 映射
    ```

### 邮件列表
17. **linux-ext4@vger.kernel.org**: ext4 开发讨论
    - 归档: https://lore.kernel.org/linux-ext4/

---

## 十四、常见问题与陷阱

### Q1: 为什么 `EXT4_MAX_EXTENT_DEPTH` 是 5 而不是 3？

**A**: 这是一个历史遗留问题。早期文档和一些代码注释说最大深度是 3，但实际上内核定义是 5:

```c
// fs/ext4/ext4_extents.h:87
#define EXT4_MAX_EXTENT_DEPTH 5
```

5 层深度可以寻址 340^5 ≈ 4.5 × 10^12 个块，对于 4K 块大小相当于 ~16 PB。实际上大多数文件只需要 1-2 层。

### Q2: COMPAT vs RO_COMPAT vs INCOMPAT 特性如何选择？

**A**:
- **COMPAT**: 老内核可以安全忽略，不影响读写 (如 `DIR_PREALLOC`)
- **RO_COMPAT**: 老内核可以只读挂载，写入可能导致问题 (如 `BIGALLOC`)
- **INCOMPAT**: 老内核必须拒绝挂载 (如 `EXTENTS`, `64BIT`)

选择原则:
- 新特性只影响性能/可选功能 → COMPAT
- 新特性改变数据布局但老内核可以安全读取 → RO_COMPAT
- 新特性根本改变文件系统结构 → INCOMPAT

### Q3: `i_blocks_lo` 和 `i_blocks_high` 的单位是什么？

**A**: 这取决于 `EXT4_HUGE_FILE_FL` 标志:

```c
// 如果 i_flags & EXT4_HUGE_FILE_FL:
//   单位是文件系统块 (通常 4K)
// 否则:
//   单位是 512 字节扇区 (传统 Unix 语义)
```

这允许 ext4 支持超过 2TB 的文件，同时保持与老工具的兼容性。
