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

---

## 十、深度代码解析

### 10.1 Extent 查找算法 (ext4_find_extent)

```c
// fs/ext4/extents.c:885-968 (简化)
struct ext4_ext_path *
ext4_find_extent(struct inode *inode, ext4_lblk_t block,
                 struct ext4_ext_path *path, int flags)
{
    struct ext4_extent_header *eh;
    short int depth, i, ppos = 0;
    
    // 1. 获取 inode 中的 extent 根节点
    eh = ext_inode_hdr(inode);      // 指向 i_block[] 开头
    depth = ext_depth(inode);        // eh->eh_depth
    
    // 2. 分配路径数组 (depth + 2 用于可能的树增长)
    if (!path) {
        path = kzalloc_objs(struct ext4_ext_path, depth + 2, gfp_flags);
        path[0].p_maxdepth = depth + 1;
    }
    path[0].p_hdr = eh;
    
    // 3. 可选: 缓存 extent 到 ES Tree
    if (!(flags & EXT4_EX_NOCACHE) && depth == 0)
        ext4_cache_extents(inode, eh);
    
    // 4. 从根向下遍历索引节点
    i = depth;
    while (i) {
        // 4.1 二分查找当前层的索引
        ext4_ext_binsearch_idx(inode, path + ppos, block);
        
        // 4.2 获取下一层块地址
        path[ppos].p_block = ext4_idx_pblock(path[ppos].p_idx);
        
        // 4.3 读取下一层块
        bh = read_extent_tree_block(inode, path[ppos].p_idx, --i, flags);
        eh = ext_block_hdr(bh);
        
        ppos++;
        path[ppos].p_bh = bh;
        path[ppos].p_hdr = eh;
    }
    
    // 5. 在叶子节点中二分查找 extent
    ext4_ext_binsearch(inode, path + ppos, block);
    
    return path;
}
```

**算法复杂度**: O(depth × log(entries_per_block)) ≈ O(5 × log(340)) ≈ O(42)

### 10.2 二分查找实现

```c
// fs/ext4/extents.c (简化)
static void ext4_ext_binsearch_idx(struct inode *inode,
                                   struct ext4_ext_path *path, ext4_lblk_t block)
{
    struct ext4_extent_header *eh = path->p_hdr;
    struct ext4_extent_idx *r, *l, *m;
    
    l = EXT_FIRST_INDEX(eh) + 1;
    r = EXT_LAST_INDEX(eh);
    
    // 标准二分查找: 找到最后一个 ei_block <= block 的索引
    while (l <= r) {
        m = l + (r - l) / 2;
        if (block < le32_to_cpu(m->ei_block))
            r = m - 1;
        else
            l = m + 1;
    }
    path->p_idx = l - 1;
}

static void ext4_ext_binsearch(struct inode *inode,
                               struct ext4_ext_path *path, ext4_lblk_t block)
{
    // 类似逻辑，但查找 extent 而非 index
    // 找到最后一个 ee_block <= block 的 extent
}
```

### 10.3 Extent 插入与分裂

```c
// fs/ext4/extents.c (简化概念)
int ext4_ext_insert_extent(handle_t *handle, struct inode *inode,
                           struct ext4_ext_path *path,
                           struct ext4_extent *newext, int flag)
{
    struct ext4_extent_header *eh;
    struct ext4_extent *nearex;
    int depth = ext_depth(inode);
    
    // 1. 尝试合并相邻 extent
    if (ext4_ext_try_to_merge(handle, inode, path, newext))
        goto merge_done;
    
    // 2. 检查是否需要分裂
    if (le16_to_cpu(eh->eh_entries) >= le16_to_cpu(eh->eh_max)) {
        // 叶子节点满了，需要分裂
        path = ext4_ext_create_new_leaf(handle, inode, path, newext);
    }
    
    // 3. 在正确位置插入新 extent
    nearex = path[depth].p_ext;
    if (!nearex) {
        // 空叶子，直接插入
        path[depth].p_ext = EXT_FIRST_EXTENT(eh);
    } else if (le32_to_cpu(newext->ee_block) > le32_to_cpu(nearex->ee_block)) {
        // 插入到 nearex 之后，需要移动后续 extent
        memmove(nearex + 2, nearex + 1, ...);
        path[depth].p_ext = nearex + 1;
    } else {
        // 插入到 nearex 之前
        memmove(nearex + 1, nearex, ...);
    }
    
    // 4. 复制新 extent 数据
    *path[depth].p_ext = *newext;
    eh->eh_entries = cpu_to_le16(le16_to_cpu(eh->eh_entries) + 1);
    
    return 0;
}
```

### 10.4 树分裂流程

```
情况: 叶子节点已满，需要插入新 extent

1. 分配新叶子块
2. 将原叶子的一半 extent 移动到新块
3. 在父索引节点插入新索引指向新叶子
4. 如果父节点也满，递归分裂
5. 如果根节点满，增加树深度

       [idx1|idx2|idx3]           [idx1|idx2] <-- 新根
              |                        |    \
       [e1|e2|e3|e4] 满         [e1|e2]   [e3|e4|new]
```

### 10.5 Unwritten Extent 转换

```c
// fs/ext4/extents.c
static int ext4_ext_convert_to_initialized(handle_t *handle,
                                           struct inode *inode,
                                           struct ext4_map_blocks *map,
                                           struct ext4_ext_path *path,
                                           int flags)
{
    // 场景: fallocate 创建的预分配 extent 在实际写入后需要转换
    
    // 情况 1: 整个 extent 被写入
    if (map covers entire extent) {
        // 简单清除 unwritten 标志
        ext4_ext_mark_initialized(extent);
    }
    
    // 情况 2: 写入覆盖 extent 的一部分
    else {
        // 需要分裂 extent:
        // [====unwritten====]
        //      [written]
        // 变成:
        // [unwr][written][unwr]
        
        ext4_split_extent(handle, inode, path, map, flags);
    }
}
```

### 10.6 Extent 块校验和

```c
// fs/ext4/extents.c:48-83
static __le32 ext4_extent_block_csum(struct inode *inode,
                                     struct ext4_extent_header *eh)
{
    struct ext4_inode_info *ei = EXT4_I(inode);
    __u32 csum;

    // 校验和 = crc32c(inode_csum_seed, extent_block_data)
    // 覆盖从 header 到 tail 之前的所有数据
    csum = ext4_chksum(ei->i_csum_seed, (__u8 *)eh,
                       EXT4_EXTENT_TAIL_OFFSET(eh));
    return cpu_to_le32(csum);
}

// 注意: inode 内的 extent 根节点没有单独的校验和
// 它被包含在 inode 校验和中
```

---

## 十一、参考文献与资源

### 官方文档
1. **Linux 内核文档**: [Documentation/filesystems/ext4/extents.rst](https://www.kernel.org/doc/html/latest/filesystems/ext4/dynamic.html#extent-tree)
2. **ext4 磁盘格式**: https://www.kernel.org/doc/html/latest/filesystems/ext4/ondisk/extents.html

### 学术论文
3. **"Extent Trees in ext4"** - Alex Tomas, Cluster File Systems
   - Extent 树的原始设计文档
4. **"Analysis of the Ext4 File System"** - Florian Buchholz, 2015
   - 详细的磁盘格式分析

### LWN.net 文章
5. **"ext4 and extent trees"** - https://lwn.net/Articles/234090/ (2007)
   - 首次介绍 extent 树概念
6. **"Improving ext4"** - https://lwn.net/Articles/469805/ (2011)
   - Extent 相关优化

### 关键 Commit
7. **Extent 树引入**: `a86c6181` "ext4: add extent map manipulate functions" (2006-10)
8. **Unwritten extent**: `56055d3a` "ext4: add support for unwritten extents" (2007)
9. **Extent 缓存 (ES Tree)**: `d100eef2` "ext4: improve extent status tree shrinker" (2015)
10. **延迟分裂**: `197217a5` "ext4: defer extent splitting to endio" (2023)
11. **Atomic write 支持**: 2025 系列 patches

### 调试工具
12. **debugfs**: 查看 extent 树
    ```bash
    debugfs /dev/sda1
    debugfs: extent_open <inode>
    debugfs: extent_stat
    # 显示: Level: 0, Entries: 3, Max: 4
    
    debugfs: extent_dump
    # 显示所有 extent 条目
    ```

13. **filefrag**: 查看文件碎片和 extent 映射
    ```bash
    filefrag -v /path/to/file
    # ext:     logical_offset:        physical_offset: length:   expected: flags:
    #   0:        0..     127:      12345..     12472:    128:             last
    ```

14. **e2fsck**: 验证 extent 树完整性
    ```bash
    e2fsck -f -n /dev/sda1
    # 检查 extent 树结构
    ```

### 邮件列表讨论
15. **Extent 优化讨论**: https://lore.kernel.org/linux-ext4/ 搜索 "extent"

---

## 十二、常见问题与陷阱

### Q1: `EXT4_MAX_EXTENT_DEPTH` 为什么是 5 而不是文档中常说的 3？

**A**: 历史原因。早期文档基于理论计算（4K块、1TB文件），但实际代码为了安全裕度设为 5:

```c
// fs/ext4/ext4_extents.h:87
#define EXT4_MAX_EXTENT_DEPTH 5
```

实际上大多数文件只需要 depth=0（extent 存储在 inode 中）或 depth=1。

### Q2: 为什么 `ee_len = 0x8000` 是初始化 extent 而不是 unwritten？

**A**: 设计权衡。`ee_len` 的最高位用作 unwritten 标志，但为了允许最大长度的初始化 extent (32768 块)，特殊处理了 `0x8000`:

```c
// 解释:
// ee_len <= 0x7FFF: initialized, length = ee_len
// ee_len == 0x8000: initialized, length = 32768 (特殊情况)
// ee_len > 0x8000:  unwritten, length = ee_len - 0x8000

// 所以 unwritten extent 最大长度是 0xFFFF - 0x8000 = 32767
```

### Q3: Extent 树和 ES Tree (Extent Status Tree) 的关系？

**A**:
- **Extent Tree**: 磁盘上的 B+ 树，存储物理块映射
- **ES Tree**: 内存中的红黑树，缓存 extent 信息 + 额外状态

```
Extent Tree (磁盘):     [ee_block, ee_len, ee_pblk] 只有映射
ES Tree (内存):         [es_lblk, es_len, es_pblk, es_type] 加了类型

ES Tree 类型:
- EXTENT_STATUS_WRITTEN:   已写入，对应磁盘 extent
- EXTENT_STATUS_UNWRITTEN: 预分配，对应磁盘 unwritten extent
- EXTENT_STATUS_DELAYED:   延迟分配，只存在于内存
- EXTENT_STATUS_HOLE:      空洞，无磁盘对应
```

### Q4: 为什么 extent 块有校验和但 inode 内的 extent 根没有？

**A**: inode 内的 extent 根被包含在 inode 校验和中。单独为它计算校验和是冗余的:

```c
// inode 校验和覆盖整个 inode 数据，包括 i_block[]
// i_block[] 存储 extent header + extent 条目

// 只有外部 extent 块需要单独的 ext4_extent_tail
```

### Q5: 如何处理 extent 树损坏？

**A**: 
1. **检测**: 挂载时/访问时校验和检查
2. **标记**: 设置 `EXT4_ERROR_FS`，根据 `errors=` 挂载选项处理
3. **修复**: 运行 `e2fsck -f` 离线修复

```bash
# 检查 extent 树
e2fsck -f /dev/sda1

# 常见错误:
# "Extent tree (at level N) could be shorter"
# "Bad extent length (X) or block number (Y)"
# "Extent tree checksum incorrect"
```
