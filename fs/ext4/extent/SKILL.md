---
name: "ext4-extent"
description: "ext4文件系统Extent树专家。当用户询问ext4 extent树、ext4_extent、ext4_extent_idx、文件数据布局时调用此技能。"
---

# ext4 Extent 树

## 一、概述

Extent 树是 ext4 用于管理文件数据位置的高效数据结构，取代了传统的直接/间接块映射方式。

---

## 二、Extent 概念

### 2.1 什么是 Extent

Extent 描述一段连续的磁盘块：

```
一个 extent = 起始逻辑块号 + 起始物理块号 + 长度
```

### 2.2 与间接块的区别

| 特性 | 间接块 | Extent |
|------|--------|--------|
| 大文件效率 | 低 (多级间接) | 高 (B+树) |
| 碎片处理 | 差 | 好 |
| 连续分配 | 不感知 | 优化 |
| 最大文件 | 2TB (4K块) | 16TB |

---

## 三、Extent 结构定义

### 3.1 Extent 头

**位置**: `fs/ext4/ext4.h`

```c
struct ext4_extent_header {
    __le16  eh_magic;       /* 魔数 0xF30A */
    __le16  eh_entries;     /* 条目数 */
    __le16  eh_max;         /* 最大条目数 */
    __le16  eh_depth;       /* 深度 (0=叶子) */
    __le32  eh_generation;  /* 代号 */
};
```

### 3.2 Extent 索引节点

```c
struct ext4_extent_idx {
    __le32  ei_block;       /* 逻辑块号 */
    __le32  ei_leaf_lo;     /* 子节点块号 (低 32 位) */
    __le16  ei_leaf_hi;     /* 子节点块号 (高 16 位) */
    __u16   ei_unused;      /* 未用 */
};
```

### 3.3 Extent 叶子节点

```c
struct ext4_extent {
    __le32  ee_block;       /* 逻辑块号 */
    __le16  ee_len;         /* 长度 */
    __le16  ee_start_hi;    /* 物理块号 (高 16 位) */
    __le32  ee_start_lo;    /* 物理块号 (低 32 位) */
};
```

---

## 四、Extent 树布局

### 4.1 Inode 内的 Extent 树

```
i_block[0-14] (60 字节):
┌─────────────────────────────────────────────────────────────┐
│  ext4_extent_header (12 字节)                               │
│    eh_magic = 0xF30A                                        │
│    eh_entries = N                                           │
│    eh_max = 4                                               │
│    eh_depth = 0 或 1                                        │
├─────────────────────────────────────────────────────────────┤
│  ext4_extent[0] (12 字节)                                   │
│    ee_block = 0                                             │
│    ee_start = 1000                                          │
│    ee_len = 100                                             │
├─────────────────────────────────────────────────────────────┤
│  ext4_extent[1] (12 字节)                                   │
│    ee_block = 100                                           │
│    ee_start = 2000                                          │
│    ee_len = 50                                              │
├─────────────────────────────────────────────────────────────┤
│  ext4_extent[2] (12 字节)                                   │
│  ext4_extent[3] (12 字节)                                   │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 多级 Extent 树

```
                    ┌─────────────┐
                    │  Root       │ (在 inode 内)
                    │  depth=1    │
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │  Leaf       │ │  Leaf       │ │  Leaf       │
    │  depth=0    │ │  depth=0    │ │  depth=0    │
    │  extents[]  │ │  extents[]  │ │  extents[]  │
    └─────────────┘ └─────────────┘ └─────────────┘
```

---

## 五、Extent 操作

### 5.1 查找 Extent

```c
/* 查找指定逻辑块号对应的 extent */
struct ext4_extent *
ext4_find_extent(struct inode *inode, ext4_lblk_t block,
                 struct ext4_ext_path **path, int flags)
{
    struct ext4_extent_header *eh;
    struct ext4_ext_path *p;

    /* 1. 从 inode 内的 extent 头开始 */
    eh = ext_inode_hdr(inode);

    /* 2. 遍历 B+树 */
    while (eh->eh_depth > 0) {
        /* 二分查找索引节点 */
        ix = ext4_ext_binsearch_idx(eh, block);
        /* 读取子节点 */
        bh = sb_bread(inode->i_sb, ext4_idx_pblock(ix));
        eh = ext_block_hdr(bh);
    }

    /* 3. 在叶子节点中查找 extent */
    ex = ext4_ext_binsearch(eh, block);

    return ex;
}
```

### 5.2 插入 Extent

```c
/* 插入 extent */
int ext4_ext_insert_extent(handle_t *handle, struct inode *inode,
                           struct ext4_ext_path *path,
                           struct ext4_extent *newext, int flags)
{
    /* 1. 检查是否需要分裂 */
    if (le16_to_cpu(eh->eh_entries) < le16_to_cpu(eh->eh_max)) {
        /* 直接插入 */
        ext4_ext_insert_leaf(path, newext);
    } else {
        /* 需要分裂节点 */
        ext4_ext_split(handle, inode, path, newext);
    }

    return 0;
}
```

### 5.3 删除 Extent

```c
/* 删除 extent */
int ext4_ext_remove_space(struct inode *inode, ext4_lblk_t start,
                          ext4_lblk_t end)
{
    /* 1. 查找起始 extent */
    path = ext4_find_extent(inode, start, NULL, 0);

    /* 2. 删除范围内的 extent */
    while (start <= end) {
        ex = path[depth].p_ext;
        ext4_ext_remove_leaf(handle, inode, path, ex);
    }

    /* 3. 合并空节点 */
    ext4_ext_try_to_merge(handle, inode, path);

    return 0;
}
```

---

## 六、Extent 状态

### 6.1 初始化状态

```c
#define EXT4_EXT_STATUS_WRITTEN    0x1  /* 已写入 */
#define EXT4_EXT_STATUS_UNWRITTEN  0x2  /* 未写入 (预分配) */
```

### 6.2 标志语义 (flags)

```c
/* 转换标志 */
#define EXT4_GET_BLOCKS_CONVERT           /* 将分割后的 extent 转换为 written */
#define EXT4_GET_BLOCKS_CONVERT_UNWRITTEN /* 将分割后的 extent 转换为 unwritten */
#define EXT4_GET_BLOCKS_METADATA_NOFAIL   /* 使用预留元数据块，确保分割不失败 */
#define EXT4_EX_NOCACHE                   /* 不缓存到 extent status tree */

/* 已移除的标志 (不再使用) */
/* EXT4_GET_BLOCKS_IO_CREATE_EXT */     /* 旧: 在 I/O 提交前分割 extent */
```

### 6.3 Zeroout 回退机制

当 extent 树操作失败时，zeroout 作为安全回退:

| 转换类型 | Zeroout 行为 |
|----------|-------------|
| unwritten → written | Zeroout 映射范围之外的部分 |
| written → unwritten | Zeroout 仅映射范围 |

**最大 zeroout 限制**: 为防止 zeroout 耗时过长，设置了最大 zeroout 块数限制。

### 6.4 预分配 Extent

```c
/* 创建预分配 extent */
int ext4_ext_convert_to_initialized(handle_t *handle,
                                    struct inode *inode,
                                    struct ext4_map_blocks *map)
{
    /* 将 unwritten extent 转换为 written */
}
```

### 6.5 延迟分割策略 (Deferred Splitting)

现代 ext4 将 extent 分割推迟到 I/O 完成时:

```
旧流程 (已废弃):
  分配块 → 分割 extent (启动 journal handle) → 提交 I/O → endio 转换
  
新流程 (当前):
  分配块 → 提交 I/O (无 journal handle) → endio 分割+转换
```

**优势**:
- 避免在 I/O 提交时启动不必要的 journal handle
- 合并 I/O 完成时减少不必要的分割操作
- 提升并发 DIO 性能 (~25%)
- 使用 `EXT4_GET_BLOCKS_METADATA_NOFAIL` 确保分割不失败

---

## 七、延迟分配

### 7.1 概述

延迟分配 (delalloc) 在写入时不立即分配块，而是在回写时分配。

### 7.2 优势

| 优势 | 说明 |
|------|------|
| 减少碎片 | 分配时已知完整数据 |
| 提高性能 | 减少元数据更新 |
| 节省空间 | 预分配块可以合并 |

### 7.3 延迟分配流程

```
1. 写入数据
   │
   ├── 不分配块，只更新文件大小
   │
   └── 标记页面为脏

2. 回写触发
   │
   ├── 计算需要的块数
   │
   ├── 分配连续块
   │
   └── 创建 extent
```

---

## 八、调试命令

### 8.1 debugfs

```bash
debugfs /dev/sdX

# 查看 inode 的 extent
debugfs: extents <inode号>

# 输出示例
Level Entries       Logical             Physical          Length Flags
 0/ 1   1/  2     0 -    9999       1024 -    11023      10000
 1/ 1   1/  4     0 -    4999       2048 -     7047       5000
 1/ 1   2/  4  5000 -    9999       8192 -    13191       5000
```

### 8.2 查看文件 extent

```bash
# 使用 filefrag
filefrag -v /path/to/file

# 输出示例
Filesystem type is: ef53
File size of /path/to/file is 4096000 (1000 blocks of 4096 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..    999:       1024..    2023:   1000:
```

---

## 九、代码位置

| 功能 | 文件路径 |
|------|----------|
| 结构定义 | `fs/ext4/ext4.h` |
| Extent 操作 | `fs/ext4/extents.c` |
| Extent 状态 | `fs/ext4/extents_status.c` |

---

## 十、参考资源

- ext4 官方文档: `Documentation/filesystems/ext4/`
- ext4 内核代码: `fs/ext4/extents.c`
