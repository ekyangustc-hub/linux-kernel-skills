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

---

## 七、深度代码解析

### 7.1 目录项查找算法

```c
// fs/ext4/namei.c (简化)
static struct buffer_head *ext4_find_entry(struct inode *dir,
                                           const struct qstr *d_name,
                                           struct ext4_dir_entry_2 **res_dir)
{
    // 检查是否使用 HTREE 索引
    if (is_dx(dir)) {
        // HTREE 查找: O(log n)
        return ext4_dx_find_entry(dir, d_name, res_dir);
    }
    
    // 线性查找: O(n)
    for (block = 0; block < nblocks; block++) {
        bh = ext4_read_dirblock(dir, block, DIRENT);
        de = (struct ext4_dir_entry_2 *)bh->b_data;
        
        while ((char *)de < bh->b_data + bh->b_size) {
            if (ext4_match(dir, d_name, de)) {
                *res_dir = de;
                return bh;
            }
            de = ext4_next_entry(de, bh->b_size);
        }
    }
    return NULL;
}
```

### 7.2 HTREE 哈希计算

```c
// fs/ext4/hash.c
int ext4fs_dirhash(const struct inode *dir, const char *name, int len,
                   struct dx_hash_info *hinfo)
{
    // 根据 s_def_hash_version 选择算法
    switch (hinfo->hash_version) {
    case DX_HASH_LEGACY:
        // 简单累加
        break;
    case DX_HASH_HALF_MD4:
        // 基于 MD4 的变体
        break;
    case DX_HASH_TEA:
        // TEA 加密算法变体
        break;
    case DX_HASH_SIPHASH:
        // 用于 casefold+encrypt，更安全
        break;
    }
}
```

### 7.3 目录项插入

```c
// fs/ext4/namei.c (简化)
static int add_dirent_to_buf(handle_t *handle, struct ext4_filename *fname,
                             struct inode *dir, struct inode *inode,
                             struct ext4_dir_entry_2 *de, struct buffer_head *bh)
{
    unsigned int offset = 0;
    unsigned short reclen;
    int nlen = ext4_dir_rec_len(fname_len(fname), dir);
    
    // 查找有足够空间的位置
    while (offset < bh->b_size) {
        de = (struct ext4_dir_entry_2 *)(bh->b_data + offset);
        reclen = le16_to_cpu(de->rec_len);
        
        if (de->inode == 0 && reclen >= nlen) {
            // 找到空闲项
            goto add_entry;
        }
        
        // 检查当前项尾部是否有空间
        if (reclen >= nlen + ext4_dir_rec_len(de->name_len, dir)) {
            // 分割当前项
            goto split_and_add;
        }
        
        offset += reclen;
    }
    
    return -ENOSPC;  // 块满，需要扩展目录
}
```

---

## 八、参考文献与资源

### 官方文档
1. **Linux 内核文档**: [Documentation/filesystems/ext4/directory.rst](https://www.kernel.org/doc/html/latest/filesystems/ext4/dynamic.html#directory-entries)

### 学术论文
2. **"HTree: An Indexed Directory for ext2"** - Daniel Phillips, 2001
   - HTREE 原始设计论文

### LWN.net 文章
3. **"Large directories and htree"** - https://lwn.net/Articles/21148/ (2002)
4. **"Casefold and ext4"** - https://lwn.net/Articles/776039/ (2019)

### 关键 Commit
5. **HTREE 引入**: `ac27a0ec` "ext4: convert to use htree by default" (2006)
6. **Casefold 支持**: `b886ee3e` "ext4: Support case-insensitive file name lookups" (2019)
7. **加密目录项哈希**: `1c2f44e8` "ext4: handle encrypted and casefolded directories" (2020)

### 调试工具
8. **debugfs**: 查看目录内容
   ```bash
   debugfs /dev/sda1
   debugfs: ls -l /path/to/dir
   debugfs: htree /path/to/dir
   # 显示 HTREE 结构
   ```

9. **e2fsck**: 验证目录完整性
   ```bash
   e2fsck -f /dev/sda1
   # Pass 2: Checking directory structure
   ```

---

## 九、常见问题与陷阱

### Q1: 为什么小目录不使用 HTREE？

**A**: HTREE 需要额外的索引块开销。对于少量文件的目录，线性查找更快。阈值大约在 2 个块左右 (~500 文件)。

### Q2: `rec_len` 为什么可以大于实际目录项大小？

**A**: `rec_len` 是到下一个有效目录项的距离，不是当前项的大小。删除文件时，将 `inode` 设为 0，但不回收空间，只是增大前一个项的 `rec_len`。

### Q3: 加密目录下为什么还能 ls？

**A**: 加密只影响文件名内容，不影响目录结构。没有密钥时，文件名显示为编码后的乱码。
