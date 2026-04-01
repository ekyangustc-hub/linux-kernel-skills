---
name: "ext4-directory"
description: "ext4文件系统目录结构专家。当用户询问ext4目录项、HTREE索引、目录格式、目录操作时调用此技能。"
---

# ext4 Directory

## 〇、为什么需要这个机制？

为什么需要 HTREE 索引目录？线性搜索目录项的时间复杂度是 O(n)，大目录性能极差。HTREE 使用 B+ 树索引目录项，查找复杂度降为 O(log n)。ext4 支持 3 层 HTREE（largedir 特性），可容纳数千万目录项。加密+casefold 场景下，目录项需要存储哈希值，这是 HTREE 查找的基础。

在 ext2 时代，目录只是一个线性列表的目录项。当目录包含数万个文件时，查找一个文件需要遍历整个目录，性能急剧下降。HTREE 的引入是 ext3 最重要的性能改进之一。

没有 HTREE，现代 Linux 系统的 /usr、/var 等大型目录将无法高效运行，ls、find 等常用命令在大目录中会严重卡顿。

---

## 一、目录项结构 (verified: ext4.h:2396-2454)

```c
#define EXT4_NAME_LEN 255

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

/* Encrypted+casefolded entries need hash stored on disk */
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
```

### 文件类型

```c
#define EXT4_FT_UNKNOWN		0
#define EXT4_FT_REG_FILE	1
#define EXT4_FT_DIR		2
#define EXT4_FT_CHRDEV		3
#define EXT4_FT_BLKDEV		4
#define EXT4_FT_FIFO		5
#define EXT4_FT_SOCK		6
#define EXT4_FT_SYMLINK		7
#define EXT4_FT_MAX		8
#define EXT4_FT_DIR_CSUM	0xDE	/* fake type for dir tail */
```

---

## 二、Encrypted+Casefolded 目录项

当目录同时启用加密和 casefold 时，目录项末尾附加哈希值:

```c
static inline bool ext4_hash_in_dirent(const struct inode *inode)
{
	return IS_CASEFOLDED(inode) && IS_ENCRYPTED(inode);
}

#define EXT4_DIRENT_HASHES(entry) \
	((struct ext4_dir_entry_hash *) \
		(((void *)(entry)) + \
		((8 + (entry)->name_len + EXT4_DIR_ROUND) & ~EXT4_DIR_ROUND)))
#define EXT4_DIRENT_HASH(entry) le32_to_cpu(EXT4_DIRENT_HASHES(entry)->hash)
#define EXT4_DIRENT_MINOR_HASH(entry) \
		le32_to_cpu(EXT4_DIRENT_HASHES(entry)->minor_hash)
```

### rec_len 计算

```c
static inline unsigned int ext4_dir_rec_len(__u8 name_len,
						const struct inode *dir)
{
	int rec_len = (name_len + 8 + EXT4_DIR_ROUND);
	if (dir && ext4_hash_in_dirent(dir))
		rec_len += sizeof(struct ext4_dir_entry_hash);
	return (rec_len & ~EXT4_DIR_ROUND);
}
```

---

## 三、HTREE 索引目录

### 3.1 哈希版本

```c
#define DX_HASH_LEGACY			0
#define DX_HASH_HALF_MD4		1
#define DX_HASH_TEA			2
#define DX_HASH_LEGACY_UNSIGNED		3
#define DX_HASH_HALF_MD4_UNSIGNED	4
#define DX_HASH_TEA_UNSIGNED		5
#define DX_HASH_SIPHASH			6	/* casefold+encrypted */
```

### 3.2 HTREE 级别

```c
#define	EXT4_HTREE_LEVEL_COMPAT	2	/* 无 largedir */
#define	EXT4_HTREE_LEVEL	3	/* 有 largedir */

static inline int ext4_dir_htree_level(struct super_block *sb)
{
	return ext4_has_feature_largedir(sb) ?
		EXT4_HTREE_LEVEL : EXT4_HTREE_LEVEL_COMPAT;
}
```

### 3.3 HTREE 根节点布局

```
Block 0 (root):
┌───────────────────────────────────────────────────────┐
│ "." entry (ext4_dir_entry_2)                          │
│ ".." entry (ext4_dir_entry_2)                         │
│ dx_root_info:                                         │
│   reserved_zero (4 bytes)                             │
│   hash_version (1 byte)                               │
│   info_length (1 byte, = 8)                           │
│   indirect_levels (1 byte)                            │
│   unused_flags (1 byte)                               │
│ dx_entry[]:                                           │
│   { hash, block } × N                                 │
└───────────────────────────────────────────────────────┘
```

### 3.4 HTREE 内部节点

```
Internal/Leaf Block:
┌───────────────────────────────────────────────────────┐
│ fake_dirent:                                          │
│   inode=0, rec_len=blocksize, name_len=0, ft=0xDE    │
│ (如果是内部节点) dx_entry[]: { hash, block } × N       │
│ (如果是叶子节点) ext4_dir_entry_2[] × N                │
│ ext4_dir_entry_tail (metadata_csum)                   │
└───────────────────────────────────────────────────────┘
```

---

## 四、目录操作

### 4.1 判断是否 HTREE

```c
#define is_dx(dir) (ext4_has_feature_dir_index((dir)->i_sb) && \
		    ext4_test_inode_flag((dir), EXT4_INODE_INDEX))
```

### 4.2 HTREE EOF

```c
#define EXT4_HTREE_EOF_32BIT   ((1UL  << (32 - 1)) - 1)
#define EXT4_HTREE_EOF_64BIT   ((1ULL << (64 - 1)) - 1)
```

### 4.3 目录项填充

```c
#define EXT4_DIR_PAD			4
#define EXT4_DIR_ROUND			(EXT4_DIR_PAD - 1)
#define EXT4_MAX_REC_LEN		((1<<16)-1)
```

---

## 五、readdir 私有数据

```c
struct dir_private_info {
	struct rb_root	root;
	struct rb_node	*curr_node;
	struct fname	*extra_fname;
	loff_t		last_pos;
	__u32		curr_hash;
	__u32		curr_minor_hash;
	__u32		next_hash;
	u64		cookie;
	bool		initialized;
};
```

---

## 六、关键代码位置

| 功能 | 文件 |
|------|------|
| 结构定义 | fs/ext4/ext4.h:2396-2454 |
| 目录操作 | fs/ext4/dir.c |
| 名称查找 | fs/ext4/namei.c |
| HTREE 实现 | fs/ext4/namei.c |
| 哈希计算 | fs/ext4/hash.c |
