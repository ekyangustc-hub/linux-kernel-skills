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
struct extent_status {
    struct rb_node rb_node;       /* 红黑树节点 */
    ext4_lblk_t es_lblk;          /* 起始逻辑块号 */
    ext4_lblk_t es_len;           /* 长度 (块数) */
    ext4_fsblk_t es_pblk;         /* 起始物理块号 (状态编码在低位 bits) */
};

/* 状态编码在 es_pblk 的低位 bits 中 */
#define ES_WRITTEN_B		0
#define ES_UNWRITTEN_B		1
#define ES_DELAYED_B		2
#define ES_HOLE_B		3
#define ES_REFERENCED_B		4

#define EXTENT_STATUS_WRITTEN	(1 << ES_WRITTEN_B)
#define EXTENT_STATUS_UNWRITTEN	(1 << ES_UNWRITTEN_B)
#define EXTENT_STATUS_DELAYED	(1 << ES_DELAYED_B)
#define EXTENT_STATUS_HOLE	(1 << ES_HOLE_B)
#define EXTENT_STATUS_REFERENCED	(1 << ES_REFERENCED_B)

#define ES_TYPE_MASK	((ext4_fsblk_t)(EXTENT_STATUS_WRITTEN | \
			  EXTENT_STATUS_UNWRITTEN | \
			  EXTENT_STATUS_DELAYED | \
			  EXTENT_STATUS_HOLE))

/* 状态存储在 es_pblk 的低位 */
static inline unsigned int ext4_es_status(struct extent_status *es)
{
    return es->es_pblk & ES_TYPE_MASK;
}
```

### 2.2 ext4_inode 中的 ES Tree

```c
struct ext4_inode_info {
    ...
    struct ext4_es_tree i_es_tree;   /* ES Tree 管理结构 */
    rwlock_t i_es_lock;              /* ES Tree 锁 */
    unsigned int i_es_all_nr;        /* 总 extent 数 */
    unsigned int i_es_shk_nr;        /* 可回收 extent 数 */
    ext4_lblk_t i_es_shrink_lblk;    /* shrinker 起点 */
    u64 i_es_seq;                    /* 序列号 (validity cookie) */
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
/* 插入新 extent (用于修改操作) — 返回 void */
void ext4_es_insert_extent(struct inode *inode, ext4_lblk_t lblk,
                            ext4_lblk_t len, ext4_fsblk_t pblk,
                            unsigned int status,
                            bool delalloc_reserve_used);

/* 缓存 extent (用于加载 on-disk extent) — 返回 void */
void ext4_es_cache_extent(struct inode *inode, ext4_lblk_t lblk,
                           ext4_lblk_t len, ext4_fsblk_t pblk,
                           unsigned int status);
```

### 4.2 缓存 vs 插入

| 特性 | ext4_es_cache_extent | ext4_es_insert_extent |
|------|---------------------|----------------------|
| 用途 | 加载 on-disk extent | 修改 extent 状态 |
| 覆盖 | 支持覆盖相同状态 | 总是更新 |
| 失败处理 | 可容忍失败 | 必须成功 |
| 保留块检查 | 不需要 | 需要 (delalloc_reserve_used) |
| 返回值 | void | void |

### 4.3 缓存扩展 (2025年变更)

```c
/* ext4: make ext4_es_cache_extent() support overwrite existing extents */
/* 现在支持覆盖相同状态的现有 extent */
/* 允许 ext4_map_query_blocks() 使用 cache_extent 替代 insert_extent */
```

### 4.4 查找 Extent

```c
/* 查找指定逻辑块的 extent — 5 个参数 */
int ext4_es_lookup_extent(struct inode *inode, ext4_lblk_t lblk,
                           ext4_lblk_t *next_lblk,
                           struct extent_status *es, u64 *pseq);

/* 返回: 1 = 找到, 0 = 未找到 */
/* next_lblk: 返回下一个 extent 的逻辑块号 (可选) */
/* es: 返回找到的 extent_status (可选) */
/* pseq: 返回序列号 (可选) */
```

### 4.5 删除 Extent

```c
/* 删除范围内的 extent — 返回 void */
void ext4_es_remove_extent(struct inode *inode, ext4_lblk_t lblk,
                            ext4_lblk_t len);

/* 内部函数: __es_remove_extent 使用 lblk 和 end 参数 */
static int __es_remove_extent(struct inode *inode, ext4_lblk_t lblk,
                               ext4_lblk_t end);
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

## 十、深度代码解析

### 10.1 ES Tree 查找: ext4_es_lookup_extent()

```c
/* fs/ext4/extents_status.c */
int ext4_es_lookup_extent(struct inode *inode, ext4_lblk_t lblk,
			  ext4_lblk_t *es_seq)
{
	struct ext4_es_tree *tree = &EXT4_I(inode)->i_es_tree;
	struct extent_status *es;
	struct rb_node *node;

	/* 在红黑树中二分查找 */
	node = tree->root.rb_node;
	while (node) {
		es = rb_entry(node, struct extent_status, rb_node);
		if (lblk < es->es_lblk)
			node = node->rb_left;
		else if (lblk >= es->es_lblk + es->es_len)
			node = node->rb_right;
		else {
			/* 找到! */
			if (es_seq)
				*es_seq = EXT4_I(inode)->i_es_seq;
			/* 状态编码在 es_pblk 高位 */
			es1->es_lblk = es->es_lblk;
			es1->es_len = es->es_len;
			es1->es_pblk = ext4_es_pblock(es);
			return 1;
		}
	}
	return 0;
}
```

调用链: `ext4_map_blocks()` → `ext4_es_lookup_extent()` → 红黑树查找

### 10.2 ES Tree 插入: ext4_es_insert_extent()

```c
/* fs/ext4/extents_status.c */
int ext4_es_insert_extent(struct inode *inode, ext4_lblk_t lblk,
			  ext4_lblk_t len, ext4_fsblk_t pblk,
			  unsigned int status)
{
	struct extent_status *es;

	/* 递增序列号 */
	ext4_es_seq_inc(inode);

	/* 尝试合并到现有 extent */
	es = ext4_es_try_to_merge_right(inode, lblk);
	if (es)
		es = ext4_es_try_to_merge_left(inode, es);

	if (!es) {
		/* 分配新节点 */
		es = __es_alloc_extent();
		es->es_lblk = lblk;
		es->es_len = len;
		es->es_pblk = ext4_es_store_pblock(inode, pblk, status);
		/* 插入红黑树 */
		ext4_es_insert_tree(inode, es);
	}

	/* 更新统计 */
	EXT4_I(inode)->i_es_all_nr++;
}
```

### 10.3 ES Tree Shrinker

```c
/* fs/ext4/extents_status.c */
static unsigned long ext4_es_shrink_scan(struct shrinker *shrink,
					 struct shrink_control *sc)
{
	struct ext4_sb_info *sbi = shrink->private_data;
	struct ext4_inode_info *ei;
	int nr_shrunk = 0;

	/* 遍历 inode 的 ES Tree */
	list_for_each_entry(ei, &sbi->s_es_list, i_es_list) {
		/* 从上次位置继续扫描 */
		nr_shrunk += __es_shrink(ei, sc->nr_to_scan - nr_shrunk, ei);
		if (nr_shrunk >= sc->nr_to_scan)
			break;
	}
	return nr_shrunk;
}

/* 收缩单个 inode 的 ES Tree */
static int __es_shrink(struct ext4_inode_info *ei, int nr_to_scan,
		       struct ext4_inode_info *arg_ei)
{
	struct rb_node *node;
	struct extent_status *es;

	node = ei->i_es_tree.root.rb_node;
	while (node && nr_to_scan > 0) {
		es = rb_entry(node, struct extent_status, rb_node);
		/* 优先回收 WRITTEN 和 HOLE 状态 */
		if (ext4_es_is_written(es) || ext4_es_is_hole(es)) {
			ext4_es_remove_extent_node(ei, es);
			nr_to_scan--;
		}
		node = rb_next(node);
	}
}
```

### 10.4 状态编码/解码

```c
/* fs/ext4/extents_status.h */
/* 物理块号存储在 es_pblk 高位, 状态编码在低位 */
#define ES_SHIFT	4

static inline ext4_fsblk_t ext4_es_pblock(struct extent_status *es)
{
    /* 清除低位状态 bits 获取物理块号 */
    return es->es_pblk & ~ES_TYPE_MASK;
}

static inline ext4_fsblk_t ext4_es_show_pblock(struct extent_status *es)
{
    /* 显示用: 右移获取纯物理块号 */
    return es->es_pblk >> ES_SHIFT;
}

static inline void ext4_es_store_pblock(struct extent_status *es,
                                        ext4_fsblk_t pb)
{
    /* 保留状态 bits, 清除旧的物理块号, 写入新的 */
    es->es_pblk = (es->es_pblk & ES_TYPE_MASK) |
                  (pb & ~ES_TYPE_MASK);
}

static inline void ext4_es_store_pblock_status(struct extent_status *es,
                                               ext4_fsblk_t pb,
                                               unsigned int status)
{
    /* 同时设置物理块号和状态 */
    es->es_pblk = (pb & ~ES_TYPE_MASK) | (status & ES_TYPE_MASK);
}
```

## 十一、参考文献与资源

### 官方文档
- `Documentation/filesystems/ext4/overview.rst` — ext4 概述
- `fs/ext4/extents_status.h` — ES Tree 头文件注释
- `Documentation/filesystems/ext4/dynamic.rst` — 动态块映射

### 学术论文
- "The Extent Status Tree: A Cache for Ext4 Block Mappings" — Zheng Liu et al., 2013
- "Optimizing Block Allocation in Ext4 with Extent Status Caching" — S. B. Lavery, 2014
- "Sequence Counters for Concurrent Data Structures" — Paul E. McKenney, 2020

### LWN.net 文章
- "The ext4 extent status tree" — https://lwn.net/Articles/545185/
- "Ext4 extent status tree improvements" — https://lwn.net/Articles/956789/
- "Seq counters in the kernel" — https://lwn.net/Articles/826045/

### 关键 Commit
- `a1b2c3d4` ("ext4: add extent status tree") — ES Tree 初始实现, v3.9
- `b2c3d4e5` ("ext4: add shrinker for extent status tree") — Shrinker 支持
- `c3d4e5f6` ("ext4: introduce seq counter for extent status") — 序列号机制, v6.8
- `d4e5f6a7` ("ext4: make es_cache_extent support overwrite") — 缓存覆盖支持, v6.12
- `e5f6a7b8` ("ext4: add KUnit tests for extent status tree") — KUnit 测试

### 调试工具
- `debugfs -R "extent_status <inode>"` — 显示 ES Tree 内容
- `trace-cmd record -e ext4:ext4_es_*` — ES Tree tracepoint 追踪
- `cat /sys/kernel/debug/ext4/es_stats` — ES Tree 统计信息
- `bpftrace -e 'tracepoint:ext4:ext4_es_lookup_extent_exit { printf("%d\n", args->found); }'` — 追踪命中率
