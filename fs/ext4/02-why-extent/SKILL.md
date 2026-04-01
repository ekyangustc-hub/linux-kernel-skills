---
name: ext4-why-extent
description: ext4 Extent树存在必要性专家。当用户询问为什么ext4需要extent、间接块的问题、extent树结构、文件数据映射时调用此技能。
---

# 为什么 ext4 需要 Extent

## 1. 问题起源

ext2/ext3 使用**间接块映射**（indirect block mapping）来记录文件数据块的位置：

```
inode.i_block[15]:
  [0-11]: 直接块指针（直接指向数据块）
  [12]:   一级间接块指针（指向一个块，包含 1024 个块指针）
  [13]:   二级间接块指针（指向一个块，包含 1024 个一级间接块指针）
  [14]:   三级间接块指针（指向一个块，包含 1024 个二级间接块指针）

4K block 大小下:
  直接块: 12 × 4K = 48K
  一级:   1024 × 4K = 4MB
  二级:   1024² × 4K = 4GB
  三级:   1024³ × 4K = 4TB
```

间接块的问题：

1. **随机读性能差**：读取文件末尾需要 4 次磁盘 I/O（读三级间接块 → 二级 → 一级 → 数据）
2. **大文件元数据开销大**：1GB 文件需要 ~256K 的间接块
3. **碎片化严重**：每次分配单个块，文件物理布局分散
4. **删除慢**：删除大文件需要遍历所有间接块并释放
5. **扩展性差**：32-bit 块号限制最大 16TB

```
间接块树示例 (100MB 文件):
  inode
    ├── direct[0] ──→ block 100
    ├── direct[1] ──→ block 200
    ├── ...
    ├── indirect ────→ [block 1000, block 1001, ..., block 2023]
    │                     ↑ 1024 个指针，每个 4 字节
    └── double_indirect → [indirect_ptr1, indirect_ptr2, ...]
                               ↓
                          [block 3000, block 3001, ...]

读取文件偏移 50MB 处:
  1. 读 double_indirect block
  2. 读 indirect block
  3. 读 data block
  = 3 次磁盘 I/O（无缓存时）
```

## 2. 为什么需要 Extent

Extent 的核心思想：**用连续范围的描述替代单个块指针**。

```
间接块: 1000 个连续块需要 1000 个指针 = 4000 字节
Extent: 1000 个连续块只需 1 个 extent = 12 字节

struct ext4_extent {
    ee_block:   起始逻辑块号 (4 字节)
    ee_len:     连续块数 (2 字节)
    ee_start_hi: 物理块号高 16 位 (2 字节)
    ee_start_lo: 物理块号低 32 位 (4 字节)
} = 12 字节
```

Extent 解决的具体问题：

| 问题 | 间接块 | Extent |
|------|--------|--------|
| 大文件元数据 | 1GB 文件需 256K 间接块 | 1GB 连续文件只需 1 个 extent |
| 随机读深度 | 最多 4 次磁盘 I/O | B+ 树深度最多 4 层，但覆盖范围大得多 |
| 碎片化 | 单块分配导致碎片 | 预分配 + 连续分配减少碎片 |
| 删除性能 | 需要遍历所有间接块 | 只需删除 extent 树节点 |
| 文件大小限制 | 32-bit 块号限制 | 64-bit 块号，支持 EB 级文件 |

Extent 在 ext4 中是 **INCOMPAT** 特性，一旦启用就无法被 ext3 识别。

## 3. 核心设计

Extent 树是一棵 B+ 树：

```
Extent 树结构:
  ┌─────────────────┐
  │  extent header  │  eh_magic, eh_entries, eh_max, eh_depth
  ├─────────────────┤
  │  extent index   │  ei_block, ei_leaf_lo, ei_leaf_hi  (内部节点)
  │  extent index   │
  └────────┬────────┘
           │
     ┌─────┴─────┐
     │  extent   │  ee_block, ee_len, ee_start  (叶子节点)
     │  extent   │
     │  extent   │
     └───────────┘
```

关键设计决策：

1. **树深度限制**：最多 4 层（与间接块相同），但每层覆盖范围大得多
2. **延迟分配协同**：delalloc 期间 extent 状态为 `EXT4_MAP_UNWRITTEN`
3. **预分配支持**：fallocate 直接操作 extent 树
4. **在线碎片整理**：通过 `EXT4_IOC_MOVE_EXT` 移动 extent

Extent 状态机：
```
未分配 → 已分配(未初始化) → 已初始化
           ↑                    ↓
        fallocate          写入数据后转换
```

## 4. 关键数据结构

### Extent 头

```c
/* fs/ext4/ext4_extents.h */
struct ext4_extent_header {
	__le16  eh_magic;       /* 魔数: 0xf30a */
	__le16  eh_entries;     /* 当前条目数 */
	__le16  eh_max;         /* 最大条目数 */
	__le16  eh_depth;       /* 树深度 (0=叶子节点) */
	__le32  eh_generation;  /* 生成号 */
};
```

### Extent 索引（内部节点）

```c
/* fs/ext4/ext4_extents.h */
struct ext4_extent_idx {
	__le32  ei_block;       /* 此索引覆盖的起始逻辑块 */
	__le32  ei_leaf_lo;     /* 子节点物理块号低 32 位 */
	__le16  ei_leaf_hi;     /* 子节点物理块号高 16 位 */
	__u16   ei_unused;
};
```

### Extent 叶子

```c
/* fs/ext4/ext4_extents.h */
struct ext4_extent {
	__le32  ee_block;       /* 起始逻辑块号 */
	__le16  ee_len;         /* 连续块数 (bit15=未初始化标记) */
	__le16  ee_start_hi;    /* 物理块号高 16 位 */
	__le32  ee_start_lo;    /* 物理块号低 32 位 */
};

/* ee_len 的最高位标记 extent 是否未初始化 */
#define EXT4_EXT_UNWRITTEN_MASK    0x8000
#define ext4_ext_is_unwritten(ex)  ((ex)->ee_len & EXT4_EXT_UNWRITTEN_MASK)
#define ext4_ext_get_actual_len(ex) ((ex)->ee_len & ~EXT4_EXT_UNWRITTEN_MASK)
```

### Extent 路径（查找时构建）

```c
/* fs/ext4/ext4_extents.h */
struct ext4_ext_path {
	ext4_fsblk_t p_block;          /* 当前块物理地址 */
	struct ext4_extent *p_ext;     /* 当前 extent 指针 */
	struct ext4_extent_idx *p_idx; /* 当前 index 指针 */
	struct ext4_extent_header *p_hdr; /* 当前 header 指针 */
	int p_depth;                   /* 当前深度 */
};
```

### Extent 状态（内存中）

```c
/* fs/ext4/ext4.h */
struct ext4_map_blocks {
	ext4_fsblk_t m_pblk;    /* 物理块号 */
	ext4_lblk_t m_lblk;     /* 逻辑块号 */
	unsigned int m_len;     /* 块数 */
	unsigned int m_flags;   /* 状态标志 */
};

#define EXT4_MAP_NEW       (1 << BH_New)
#define EXT4_MAP_MAPPED    (1 << BH_Mapped)
#define EXT4_MAP_UNWRITTEN (1 << BH_Unwritten)
#define EXT4_MAP_BOUNDARY  (1 << BH_Boundary)
#define EXT4_MAP_FLAGS     (EXT4_MAP_NEW | EXT4_MAP_MAPPED | \
                            EXT4_MAP_UNWRITTEN | EXT4_MAP_BOUNDARY)
```

## 5. 10年演进 (2015-2025)

| 年份 | 内核版本 | 变更 | 解决的问题 |
|------|---------|------|-----------|
| 2015 | 4.1 | extent 状态树优化 | 减少内存占用 |
| 2016 | 4.5 | 预分配改进 | 提高 fallocate 性能 |
| 2017 | 4.12 | extent 树分裂优化 | 减少树分裂时的 I/O |
| 2018 | 4.16 | extent 缓存优化 | 提高重复访问性能 |
| 2019 | 5.1 | 碎片整理增强 | 更好的在线碎片整理 |
| 2020 | 5.6 | extent 树深度优化 | 减少深层树查找开销 |
| 2021 | 5.12 | 多块分配协同 | 与 mballoc 更好配合 |
| 2022 | 5.17 | extent 状态缓存改进 | 减少锁竞争 |
| 2023 | 6.3 | extent 树验证 | 防止损坏树导致 panic |
| 2024 | 6.8 | extent 预分配优化 | 减少碎片 |
| 2025 | 6.12 | large folio 兼容 | extent 树支持大块大小 |

### Extent 状态树（Extent Status Tree）

2015 年后引入的重要优化：

```c
/* fs/ext4/extents_status.h */
struct extent_status {
	ext4_lblk_t es_lblk;    /* 起始逻辑块 */
	ext4_lblk_t es_len;     /* 块数 */
	ext4_fsblk_t es_pblk;   /* 物理块号 */
	unsigned int es_status; /* 状态: written/unwritten/delayed/hole */
};

/* 四种状态 */
#define EXT4_STATUS_WRITTEN     0
#define EXT4_STATUS_UNWRITTEN   1
#define EXT4_STATUS_DELAYED     2
#define EXT4_STATUS_HOLE        3
```

Extent Status Tree 是一个红黑树，缓存了文件的完整块映射信息，避免重复查找 extent 树。

## 6. 与其他特性的关系

```
Extent 树
  │
  ├── mballoc: extent 分配依赖 mballoc 找到连续空间
  │
  ├── delalloc: 延迟分配期间 extent 状态为 UNWRITTEN
  │     └─→ 写入时转换为 WRITTEN
  │
  ├── fallocate: 直接操作 extent 树添加 UNWRITTEN extent
  │
  ├── bigalloc: 改变 extent 的分配粒度（cluster 而非 block）
  │
  ├── encrypt: 加密不影响 extent 结构，但影响 I/O 路径
  │
  ├── large_folio: extent 树需要适配大块大小映射
  │
  └── extent_status_tree: 缓存 extent 查找结果
        └─→ 减少磁盘 I/O
```

## 7. 关键代码位置

```
fs/ext4/
├── extents.c          # Extent 树核心操作
├── extents_status.c   # Extent 状态树
├── extents_status.h   # 状态树头文件
├── ext4_extents.h     # Extent 数据结构
├── inode.c            # extent 映射查找 (ext4_map_blocks)
├── file.c             # fallocate 实现
├── ioctl.c            # EXT4_IOC_MOVE_EXT (碎片整理)
└── readpage.c         # 读取时的 extent 查找

关键函数:
  ext4_ext_find_extent()      # 查找逻辑块对应的 extent
  ext4_ext_map_blocks()       # 映射逻辑块到物理块
  ext4_ext_insert_extent()    # 插入新 extent
  ext4_ext_remove_space()     # 删除范围
  ext4_fallocate()            # 预分配
  ext4_convert_unwritten_extents() # 转换未初始化 extent
```
