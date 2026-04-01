---
name: "ext4-es-tree"
description: "ext4 Extent Status Tree专家。当用户询问ext4 extent status缓存、es_cache、seq counter、es_insert_extent、es_cache_extent、extent状态管理时调用此技能。"
---

# ext4 Extent Status Tree (ES Tree)

## 〇、为什么需要这个机制？

为什么需要 Extent Status Tree？ext4 的 extent 树在磁盘上，每次访问都需要磁盘 I/O。ES Tree 是内存中的缓存，不仅缓存已分配的 extent，还缓存 hole、delayed、unwritten 等状态。没有 ES Tree，ext4 需要频繁访问磁盘来查询块映射，性能会大幅下降。2025 年引入的序列号机制解决了并发 I/O 中 extent 信息过期的问题。

ES Tree 的诞生源于延迟分配（delalloc）的需求。在 delalloc 模式下，文件写入时并不立即分配物理块，而是标记为"delayed"状态。ES Tree 是唯一能跟踪这种中间状态的机制——磁盘上的 extent 树只记录已确认的映射。

没有 ES Tree，ext4 的延迟分配、holes 处理、unwritten extent 转换等核心功能都无法高效实现。

---

## 一、概述

Extent Status Tree (ES Tree) 是 ext4 的内存缓存结构，用于跟踪文件逻辑块到物理块的状态映射。它不仅缓存 on-disk extent 信息，还跟踪 delayed、unwritten、hole 等状态。

---

## 二、核心数据结构

### 2.1 ext4_extent_status

**位置**: `fs/ext4/extents_status.c`

```c
struct ext4_extent_status {
    struct rb_node rb_node;       /* 红黑树节点 */
    ext4_lblk_t es_lblk;          /* 起始逻辑块号 */
    ext4_fsblk_t es_pblk;         /* 起始物理块号 */
    ext4_lblk_t es_len;           /* 长度 (块数) */
    unsigned int es_seq;          /* 序列号 (validity cookie) */
    unsigned short es_type;       /* 状态类型 */
};

/* 状态类型 */
#define EXT4_EXTENT_WRITTEN       0   /* 已写入 */
#define EXT4_EXTENT_UNWRITTEN     1   /* 未写入 (预分配) */
#define EXT4_EXTENT_DELAYED       2   /* 延迟分配 */
#define EXT4_EXTENT_HOLE          3   /* 空洞 */

/* 类型掩码 */
#define EXT4_EXTENT_TYPE_MASK     0x3
```

### 2.2 ext4_inode 中的 ES Tree

```c
struct ext4_inode_info {
    ...
    struct rb_root i_es_root;        /* ES Tree 根节点 */
    struct ext4_es_tree i_es_tree;   /* ES Tree 管理结构 */
    rwlock_t i_es_lock;              /* ES Tree 锁 */
    unsigned int i_es_all_nr;        /* 总 extent 数 */
    unsigned int i_es_shk_nr;        /* 可回收 extent 数 */
    unsigned int i_es_shrinker_batch;/* 收缩批次 */
    ...
};
```

---

## 三、序列号机制 (Seq Counter)

### 3.1 概述

**引入时间**: 2025年

类似 XFS 的 extent 序列号机制，为 ES Tree 引入序列号作为有效性 cookie。

### 3.2 工作原理

```c
/* 每次 ES Tree 变更时递增序列号 */
static void ext4_es_seq_inc(struct inode *inode)
{
    struct ext4_inode_info *ei = EXT4_I(inode);
    ei->i_es_seq++;
}

/* 查找 extent 时返回序列号 */
int ext4_es_lookup_extent(struct inode *inode, ext4_lblk_t lblk,
                           ext4_lblk_t *es_seq);
```

### 3.3 使用场景

| 场景 | 问题 | 解决方案 |
|------|------|----------|
| iomap buffered write | 查询映射和写入之间 extent 可能变更 | 写入前检查序列号 |
| writeback | 提交 I/O 前 extent 可能变更 | 提交前检查序列号 |
| move extent | 移动时 extent 类型可能变更 | 持有 folio lock 检查序列号 |

### 3.4 序列号检查流程

```
1. 查询 extent 映射 (无锁)
   │
   ├── 获取 es_seq (序列号)
   │
2. 获取 folio lock
   │
3. 检查 es_seq 是否变更
   │
   ├── 未变更: extent 信息仍然有效
   │
   └── 已变更: 重新查询映射
       └── 返回 -ESTALE
```

---

## 四、ES Tree 操作

### 4.1 插入 Extent

```c
/* 插入新 extent (用于修改操作) */
int ext4_es_insert_extent(struct inode *inode, ext4_lblk_t lblk,
                            ext4_lblk_t len, ext4_fsblk_t pblk,
                            unsigned int status);

/* 缓存 extent (用于加载 on-disk extent) */
int ext4_es_cache_extent(struct inode *inode, ext4_lblk_t lblk,
                           ext4_lblk_t len, ext4_fsblk_t pblk,
                           unsigned int status);
```

### 4.2 缓存 vs 插入

| 特性 | ext4_es_cache_extent | ext4_es_insert_extent |
|------|---------------------|----------------------|
| 用途 | 加载 on-disk extent | 修改 extent 状态 |
| 覆盖 | 支持覆盖相同状态 | 总是更新 |
| 失败处理 | 可容忍失败 | 必须成功 |
| 保留块检查 | 不需要 | 需要 |

### 4.3 缓存扩展 (2025年变更)

```c
/* ext4: make ext4_es_cache_extent() support overwrite existing extents */
/* 现在支持覆盖相同状态的现有 extent */
/* 允许 ext4_map_query_blocks() 使用 cache_extent 替代 insert_extent */
```

### 4.4 查找 Extent

```c
/* 查找指定逻辑块的 extent */
int ext4_es_lookup_extent(struct inode *inode, ext4_lblk_t lblk,
                            ext4_lblk_t *es_seq);

/* 返回: 1 = 找到, 0 = 未找到 */
/* es_seq: 返回序列号 (可选) */
```

### 4.5 删除 Extent

```c
/* 删除范围内的 extent */
int __es_remove_extent(struct inode *inode, ext4_lblk_t lblk,
                         ext4_lblk_t end);

/* 现在检查 extent 状态，确保操作正确性 */
```

---

## 五、ES Tree 收缩 (Shrinker)

### 5.1 概述

ES Tree 会缓存大量 extent 信息，当内存压力时需要回收。

```c
/* 收缩器回调 */
static unsigned long ext4_es_shrink_count(struct shrinker *shrink,
                                           struct shrink_control *sc);

static unsigned long ext4_es_shrink_scan(struct shrinker *shrink,
                                          struct shrink_control *sc);
```

### 5.2 收缩策略

```
1. 遍历 inode 的 ES Tree
   │
2. 标记可回收的 extent (written/hole)
   │
3. 移除标记的 extent
   │
4. 更新统计信息
```

---

## 六、并发安全

### 6.1 缓存一致性问题

**问题**: `EXT4_IOC_GET_ES_CACHE` 和 `EXT4_IOC_PRECACHE_EXTENTS` 在无锁情况下调用 `ext4_ext_precache()`，可能与 `ext4_collapse_range()` 竞争。

**解决方案**:
```c
/* ext4: prevent stale extent cache entries caused by concurrent get es_cache */
/* 在 EXT4_IOC_GET_ES_CACHE 和 EXT4_IOC_PRECACHE_EXTENTS 期间持有 i_rwsem */
```

### 6.2 锁层次

```
i_rwsem (inode lock)
    │
    └── i_data_sem (extent tree lock)
        │
        └── i_es_lock (ES tree lock)
```

---

## 七、KUnit 测试

### 7.1 测试覆盖

```c
/* ext4: add extent status cache support to kunit tests */
/* ES Tree 现在有完整的 KUnit 测试覆盖 */

/* 测试场景: */
/* - 插入/查找/删除 extent */
/* - 缓存覆盖 */
/* - 序列号检查 */
/* - 并发操作 */
```

---

## 八、代码位置

| 功能 | 文件路径 |
|------|----------|
| ES Tree 核心 | `fs/ext4/extents_status.c` |
| ES Tree 头文件 | `fs/ext4/extents_status.h` |
| KUnit 测试 | `fs/ext4/es-test.c` |
| 序列号机制 | `fs/ext4/extents_status.c` |
| 缓存扩展 | `fs/ext4/extents_status.c` |

---

## 九、参考资源

- ext4 官方文档: `Documentation/filesystems/ext4/`
- 补丁系列: "ext4: introduce seq counter for the extent status entry" (2025)
- 补丁系列: "ext4: extent status tree improvements" (2025-2026)
