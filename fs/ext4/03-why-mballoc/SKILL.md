---
name: ext4-why-mballoc
description: ext4多块分配器存在必要性专家。当用户询问为什么ext4需要mballoc、单块分配器的问题、块分配策略、碎片管理、buddy系统时调用此技能。
---

# 为什么 ext4 需要 Mballoc

## 1. 问题起源

ext2/ext3 使用**单块分配器**（single-block allocator），每次只分配一个 block：

```c
/* ext2/ext3 的分配方式 (简化) */
ext3_new_block() {
    // 每次只找一个空闲块
    for (group = 0; group < groups_count; group++) {
        if (block_bitmap_has_free(group)) {
            block = find_first_free_bit(group);
            mark_block_used(block);
            return block;  // 只返回 1 个块
        }
    }
}
```

单块分配器的问题：

1. **碎片化严重**：文件数据分散在各处，随机读性能差
2. **分配慢**：写入 1MB 文件需要 256 次分配调用（4K block）
3. **无预分配**：无法提前预留连续空间
4. **组选择简单**：只找第一个有空间的组，不考虑碎片
5. **无延迟分配**：立即分配，无法优化布局

```
单块分配器的碎片化示例:

磁盘布局 (写入 10MB 文件):
  [文件A块1] [文件B块1] [文件A块2] [文件C块1] [文件A块3] ...
  ↑ 文件A的块被其他文件插入，严重碎片化

随机读 10MB 文件:
  需要 2560 次随机 seek (4K block)
  机械硬盘: ~2560 × 10ms = 25.6 秒
```

## 2. 为什么需要 Mballoc

Mballoc（Multi-Block Allocator）解决的核心问题：**一次分配多个连续块，减少碎片，提高性能**。

Mballoc 的设计目标：

| 目标 | 单块分配器 | Mballoc |
|------|-----------|---------|
| 分配粒度 | 1 block | N blocks（请求大小） |
| 连续性 | 不保证 | 优先分配连续空间 |
| 组选择 | 第一个有空间的组 | 基于碎片评分选择最佳组 |
| 预分配 | 不支持 | 支持（per-CPU 预分配窗口） |
| 碎片管理 | 无 | buddy 系统管理空闲空间 |

Mballoc 由 Mingming Cao 等人在 2007 年开发，2008 年随 ext4 合并到内核。

### Buddy 系统原理

```
Buddy 系统将所有空闲空间按大小组织:

order 0: 1 block  的空闲块列表
order 1: 2 blocks 的空闲块列表
order 2: 4 blocks 的空闲块列表
order 3: 8 blocks 的空闲块列表
...
order N: 2^N blocks 的空闲块列表

分配 5 blocks:
  1. 检查 order 3 (8 blocks) - 有
  2. 拆分 8 → 4 + 4
  3. 拆分 4 → 2 + 2
  4. 分配 4 + 1 (从 order 2 和 order 0)
  或
  1. 直接找 order 3 (8 blocks) 分配
  2. 剩余 3 blocks 放回对应 order
```

## 3. 核心设计

Mballoc 的核心架构：

```
ext4_mb_new_blocks()
  │
  ├── 1. 检查预分配窗口 (pa)
  │     └─→ 如果有足够空间，直接分配
  │
  ├── 2. 组选择 (ext4_mb_regular_allocator)
  │     ├── 扫描块组
  │     ├── 计算组碎片评分
  │     └─→ 选择最佳组
  │
  ├── 3. 块分配 (ext4_mb_simple_scan / ext4_mb_complex_scan)
  │     ├── buddy 系统查找
  │     └─→ 分配连续空间
  │
  └── 4. 更新预分配窗口
        └─→ 为下次分配预留空间
```

关键设计决策：

1. **预分配窗口**：每个 inode/CPU 有预分配窗口，减少锁竞争
2. **组预分配**：针对顺序写入优化
3. **inode 预分配**：针对随机写入优化
4. **碎片评分**：选择碎片最少的组
5. **buddy 缓存**：避免重复扫描位图

## 4. 关键数据结构

### 分配请求

```c
/* fs/ext4/mballoc.h */
struct ext4_allocation_request {
	/* 目标 inode */
	struct inode *inode;

	/* 逻辑块号（相对于文件起始） */
	ext4_lblk_t logical;

	/* 物理块号（相对于块组起始，仅 hint） */
	ext4_fsblk_t goal;

	/* 分配参数 */
	ext4_grpblk_t start;     /* 建议起始块 */
	unsigned int len;        /* 请求块数 */
	unsigned int flags;      /* 分配标志 */

	/* 结果 */
	ext4_fsblk_t pstart;     /* 分配的起始物理块 */
	ext4_fsblk_t plen;       /* 分配的块数 */
};
```

### Buddy 信息

```c
/* fs/ext4/mballoc.h */
struct ext4_buddy {
	struct list_head bb_list;        /* LRU 链表 */
	struct ext4_group_info *bb_info; /* 组信息 */
	int bb_fragments;                /* 碎片数 */
	int bb_free;                     /* 空闲块数 */
	int bb_first_free;               /* 第一个空闲块 */
	int bb_counters[EXT4_MB_NUM_ORDERS]; /* 各 order 计数 */
	void *bb_bitmap;                 /* 位图副本 */
	void *bb_buddy;                  /* buddy 表 */
	int bb_order;                    /* 最大 order */
	ext4_group_t bb_group;           /* 块组号 */
	struct super_block *bb_sb;       /* 超级块 */
};

/* Buddy 表: 每个 order 的空闲块信息 */
/* 对于 order n，每个 entry 表示 2^n 个块的状态 */
```

### 组信息

```c
/* fs/ext4/mballoc.h */
struct ext4_group_info {
	unsigned short bb_counters[EXT4_MB_NUM_ORDERS + 1]; /* 各 order 空闲块数 */
	unsigned short bb_free;          /* 总空闲块数 */
	unsigned short bb_fragments;     /* 碎片数 */
	unsigned short bb_first_free;    /* 第一个空闲块偏移 */
	ext4_grpblk_t bb_start;          /* 起始块 */
	struct rb_root bb_free_root;     /* 空闲空间红黑树 */
	struct list_head bb_prealloc_list; /* 预分配列表 */
	struct rw_semaphore alloc_sem;   /* 分配信号量 */
	unsigned long bb_state;          /* 状态标志 */
	unsigned long bb_tid;            /* 事务 ID */
};

#define EXT4_MB_NUM_ORDERS 9  /* 最多支持 512 blocks (2^9) */
```

### 预分配结构

```c
/* fs/ext4/mballoc.h */
struct ext4_prealloc_space {
	struct list_head pa_inode_list;  /* inode 预分配链表 */
	struct list_head pa_group_list;  /* 组预分配链表 */
	struct list_head pa_tmp_list;    /* 临时链表 */
	struct rcu_head pa_rcu;          /* RCU 头 */
	spinlock_t pa_lock;              /* 锁 */
	ext4_fsblk_t pa_pstart;          /* 物理起始块 */
	ext4_lblk_t pa_lstart;           /* 逻辑起始块 */
	ext4_grpblk_t pa_len;            /* 长度 */
	ext4_grpblk_t pa_free;           /* 剩余空间 */
	unsigned short pa_type;          /* 类型: inode/group */
	atomic_t pa_count;               /* 引用计数 */
	unsigned pa_deleted;             /* 删除标记 */
};

#define PA_INODE    0  /* inode 预分配 */
#define PA_GROUP    1  /* 组预分配 */
```

### 分配上下文

```c
/* fs/ext4/mballoc.h */
struct ext4_allocation_context {
	struct inode *ac_inode;          /* 目标 inode */
	struct super_block *ac_sb;       /* 超级块 */

	ext4_fsblk_t ac_o_ex_sc_group;   /* 原始期望组 */
	ext4_grpblk_t ac_o_ex_fe_goal;   /* 原始期望目标块 */

	struct ext4_prealloc_space *ac_pa; /* 预分配空间 */
	struct ext4_buddy ac_buddy;      /* buddy 信息 */
	struct ext4_group_info *ac_f_pg; /* 首选组 */

	ext4_group_t ac_groups_scanned;  /* 扫描的组数 */
	ext4_grpblk_t ac_b_ex;           /* 最佳分配结果 */
	ext4_grpblk_t ac_g_ex;           /* 期望分配参数 */

	unsigned int ac_flags;           /* 分配标志 */
	unsigned int ac_op;              /* 操作类型 */
};
```

## 5. 10年演进 (2015-2025)

| 年份 | 内核版本 | 变更 | 解决的问题 |
|------|---------|------|-----------|
| 2015 | 4.1 | 预分配窗口优化 | 减少小文件碎片 |
| 2016 | 4.6 | 组选择算法改进 | 更好的碎片分布 |
| 2017 | 4.13 | buddy 缓存优化 | 减少重复扫描 |
| 2018 | 4.19 | 多队列优化 | 提高并发分配性能 |
| 2019 | 5.2 | 预分配回收优化 | 减少内存浪费 |
| 2020 | 5.8 | 碎片整理协同 | 与在线碎片整理更好配合 |
| 2021 | 5.12 | 组扫描优化 | 减少大文件系统扫描开销 |
| 2022 | 5.17 | xarray 优化 | 替换传统链表，提高查找效率 |
| 2023 | 6.3 | 预分配窗口自适应 | 根据 I/O 模式调整窗口大小 |
| 2024 | 6.8 | 全局目标优化 | 更好的跨组分配策略 |
| 2025 | 6.12 | large folio 兼容 | 支持大块大小分配 |

### 关键改进：组选择算法

```
早期 (2008):
  扫描组直到找到足够空间 → 容易集中在前几个组

2016 改进:
  计算组碎片评分 → 选择碎片最少的组
  评分 = f(空闲块数, 碎片数, 最大连续块数)

2022 改进:
  引入 mb_optimize_scan
  根据 workload 类型选择扫描策略
  - 顺序写入: 优先相邻组
  - 随机写入: 优先空闲最多的组
```

## 6. 与其他特性的关系

```
Mballoc
  │
  ├── extent: 分配的结果以 extent 形式记录
  │     └─→ 连续分配 → 单个 extent
  │     └─→ 不连续分配 → 多个 extent
  │
  ├── delalloc: 延迟分配与 mballoc 协同
  │     └─→ 积累更多写入后再分配，提高连续性
  │
  ├── flex_bg: 逻辑组组合，扩大分配范围
  │     └─→ mballoc 可以在 flex_bg 内跨组分配
  │
  ├── bigalloc: 改变分配粒度
  │     └─→ 分配 cluster 而非 block
  │
  ├── fast_commit: 不影响分配，但减少提交延迟
  │
  └── large_folio: 分配需要适配大块大小
        └─→ 最小分配单位变为 folio 大小
```

## 7. 关键代码位置

```
fs/ext4/
├── mballoc.c        # 多块分配器核心
├── mballoc.h        # 数据结构定义
├── balloc.c         # 块分配基础接口
├── extents.c        # extent 分配（调用 mballoc）
├── inode.c          # ext4_map_blocks（分配入口）
└── sysfs.c          # mballoc 统计接口

关键函数:
  ext4_mb_new_blocks()        # 核心分配函数
  ext4_mb_regular_allocator() # 常规分配器
  ext4_mb_simple_scan()       # 简单扫描
  ext4_mb_complex_scan()      # 复杂扫描
  ext4_mb_find_by_goal()      # 按目标查找
  ext4_mb_use_preallocated()  # 使用预分配
  ext4_mb_new_preallocation() # 创建预分配
  ext4_mb_release()           # 释放
  ext4_mb_init()              # 初始化
```
