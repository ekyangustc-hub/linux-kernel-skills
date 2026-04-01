---
name: "ext4-extent"
description: "ext4文件系统Extent树专家。当用户询问ext4 extent树、ext4_extent、ext4_extent_idx、文件数据布局时调用此技能。"
---

# ext4 Extent Tree

## 〇、为什么需要这个机制？

为什么需要 extent 树？ext2/ext3 使用间接块映射，每个块指针占 4 字节，一个 4K 间接块只能映射 1024 个块。大文件需要多级间接块，导致严重的碎片和性能问题。Extent 用一个 (起始块, 长度) 对描述连续块，一个 extent 可以描述 32768 个块，大幅减少元数据开销。ext4 将 extents 作为 INCOMPAT 特性，确保老内核不会错误挂载。

间接块映射的另一个问题是删除大文件时需要遍历所有间接块，I/O 开销巨大。Extent 树结构使得删除操作只需修改少量元数据节点。

没有 extent 树，ext4 就无法高效处理现代工作负载中的大文件（数据库、虚拟机镜像、视频文件等），也无法支持 bigalloc 等需要连续块分配的特性。

---

## 一、磁盘结构 (verified: ext4_extents.h:48-87)

```c
struct ext4_extent_tail {
	__le32	et_checksum;	/* crc32c(uuid+inum+extent_block) */
};

struct ext4_extent {
	__le32	ee_block;	/* first logical block extent covers */
	__le16	ee_len;		/* number of blocks covered by extent */
	__le16	ee_start_hi;	/* high 16 bits of physical block */
	__le32	ee_start_lo;	/* low 32 bits of physical block */
};

struct ext4_extent_idx {
	__le32	ei_block;	/* index covers logical blocks from 'block' */
	__le32	ei_leaf_lo;	/* pointer to the physical block of the next level */
	__le16	ei_leaf_hi;	/* high 16 bits of physical block */
	__u16	ei_unused;
};

struct ext4_extent_header {
	__le16	eh_magic;	/* 0xf30a */
	__le16	eh_entries;	/* number of valid entries */
	__le16	eh_max;		/* capacity of store in entries */
	__le16	eh_depth;	/* has tree real underlying blocks? */
	__le32	eh_generation;	/* generation of the tree */
};

#define EXT4_EXT_MAGIC		cpu_to_le16(0xf30a)
#define EXT4_MAX_EXTENT_DEPTH 5	/* NOT 3 */
```

---

## 二、Extent 块布局

```
┌─────────────────────────────────────────────────────────┐
│                    Extent Block (non-inode)              │
│  ┌───────────────────────────────────────────────────┐  │
│  │ ext4_extent_header (12 bytes)                      │  │
│  ├───────────────────────────────────────────────────┤  │
│  │ ext4_extent[] or ext4_extent_idx[] (12 bytes each) │  │
│  ├───────────────────────────────────────────────────┤  │
│  │ ext4_extent_tail (4 bytes) — et_checksum          │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**注意**: inode 内的 extent 根节点 **没有** `ext4_extent_tail`。只有外部 extent 块有。

```c
#define EXT4_EXTENT_TAIL_OFFSET(hdr) \
	(sizeof(struct ext4_extent_header) + \
	 (sizeof(struct ext4_extent) * le16_to_cpu((hdr)->eh_max)))

static inline struct ext4_extent_tail *
find_ext4_extent_tail(struct ext4_extent_header *eh)
{
	return (struct ext4_extent_tail *)(((void *)eh) +
					   EXT4_EXTENT_TAIL_OFFSET(eh));
}
```

---

## 三、Extent 长度标志

```c
#define EXT_INIT_MAX_LEN	(1UL << 15)	/* 32768 */
#define EXT_UNWRITTEN_MAX_LEN	(EXT_INIT_MAX_LEN - 1)	/* 32767 */
```

- `ee_len <= 0x8000`: initialized extent, actual length = ee_len
- `ee_len > 0x8000`: unwritten extent, actual length = ee_len - 0x8000
- `ee_len == 0x8000`: special — treated as initialized extent of length 32768

```c
static inline void ext4_ext_mark_unwritten(struct ext4_extent *ext)
{
	BUG_ON((le16_to_cpu(ext->ee_len) & ~EXT_INIT_MAX_LEN) == 0);
	ext->ee_len |= cpu_to_le16(EXT_INIT_MAX_LEN);
}

static inline int ext4_ext_is_unwritten(struct ext4_extent *ext)
{
	return (le16_to_cpu(ext->ee_len) > EXT_INIT_MAX_LEN);
}

static inline int ext4_ext_get_actual_len(struct ext4_extent *ext)
{
	return (le16_to_cpu(ext->ee_len) <= EXT_INIT_MAX_LEN ?
		le16_to_cpu(ext->ee_len) :
		(le16_to_cpu(ext->ee_len) - EXT_INIT_MAX_LEN));
}
```

---

## 四、Extent 树深度

```
EXT4_MAX_EXTENT_DEPTH = 5

depth=0: inode 内的根节点 (i_block[], 60 bytes)
depth=1: 第一层外部索引块
depth=2: 第二层外部索引块
depth=3: 第三层外部索引块
depth=4: 第四层外部索引块
depth=5: 叶子块

4K 块时: 每块可存 ~340 个 extent/idx 条目
最大寻址: 340^5 ≈ 4.5 × 10^12 个块
```

---

## 五、Extent 路径结构 (内存)

```c
struct ext4_ext_path {
	ext4_fsblk_t			p_block;
	__u16				p_depth;
	__u16				p_maxdepth;
	struct ext4_extent		*p_ext;
	struct ext4_extent_idx		*p_idx;
	struct ext4_extent_header	*p_hdr;
	struct buffer_head		*p_bh;
};
```

---

## 六、Extent 操作标志

### ext4_map_blocks 标志 (ext4.h:697-739)

```c
#define EXT4_GET_BLOCKS_CREATE			0x0001
#define EXT4_GET_BLOCKS_UNWRIT_EXT		0x0002
#define EXT4_GET_BLOCKS_DELALLOC_RESERVE	0x0004
#define EXT4_GET_BLOCKS_SPLIT_NOMERGE		0x0008
#define EXT4_GET_BLOCKS_CONVERT			0x0010
#define EXT4_GET_BLOCKS_METADATA_NOFAIL		0x0020
#define EXT4_GET_BLOCKS_NO_NORMALIZE		0x0040
#define EXT4_GET_BLOCKS_CONVERT_UNWRITTEN	0x0100
#define EXT4_GET_BLOCKS_ZERO			0x0200
#define EXT4_GET_BLOCKS_IO_SUBMIT		0x0400
#define EXT4_GET_BLOCKS_CACHED_NOWAIT		0x0800
#define EXT4_GET_BLOCKS_QUERY_LAST_IN_LEAF	0x1000	/* Atomic write */
```

### ext4_find_extent 标志 (ext4.h:750-752)

```c
#define EXT4_EX_NOCACHE				0x40000000
#define EXT4_EX_FORCE_CACHE			0x20000000
#define EXT4_EX_NOFAIL				0x10000000
```

---

## 七、Extent 树操作

### 7.1 查找

```c
/* ext4_find_extent(): 从 inode 根节点遍历 B+ 树到叶子 */
/* 使用二分查找在每一层定位正确的子节点 */
/* 返回 ext4_ext_path 数组, 记录从根到叶子的完整路径 */
```

### 7.2 插入/分裂

```c
/* ext4_ext_insert_extent(): 插入新 extent */
/* 如果节点满 (eh_entries == eh_max), 触发分裂 */
/* 分裂可能向上传播到根节点, 增加树深度 */
```

### 7.3 延迟分裂 (Deferred Splitting)

现代 ext4 将 extent 分裂推迟到 I/O 完成:

```
旧: 分配 → 分裂 extent (journal handle) → I/O → endio 转换
新: 分配 → I/O (无 journal handle) → endio 分裂+转换
```

优势: 减少 journal handle 开销, 提升并发 DIO 性能 ~25%

### 7.4 部分簇处理 (bigalloc)

```c
struct partial_cluster {
	ext4_fsblk_t pclu;  /* physical cluster number */
	ext4_lblk_t lblk;   /* logical block number within logical cluster */
	enum {initial, tofree, nofree} state;
};
```

用于 bigalloc 文件系统在删除 extent 时处理部分簇。

---

## 八、ext4_map_blocks 状态

```c
#define EXT4_MAP_NEW		BIT(BH_New)
#define EXT4_MAP_MAPPED		BIT(BH_Mapped)
#define EXT4_MAP_UNWRITTEN	BIT(BH_Unwritten)
#define EXT4_MAP_BOUNDARY	BIT(BH_Boundary)
#define EXT4_MAP_DELAYED	BIT(BH_Delay)
#define EXT4_MAP_QUERY_LAST_IN_LEAF	BIT(BH_BITMAP_UPTODATE + 1)

struct ext4_map_blocks {
	ext4_fsblk_t m_pblk;
	ext4_lblk_t m_lblk;
	unsigned int m_len;
	unsigned int m_flags;
	u64 m_seq;		/* seq counter for concurrent I/O safety */
};
```

---

## 九、关键代码位置

| 功能 | 文件 |
|------|------|
| 磁盘结构 | fs/ext4/ext4_extents.h:48-87 |
| Extent 树操作 | fs/ext4/extents.c |
| Extent 状态树 | fs/ext4/extents_status.c |
| 映射查询 | fs/ext4/inode.c (ext4_map_blocks) |
