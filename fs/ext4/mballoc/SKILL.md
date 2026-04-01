---
name: "ext4-mballoc"
description: "ext4多块分配器(Mballoc)专家。当用户询问ext4块分配策略、mb_optimize_scan、组选择算法、xarray优化、全局目标、碎片管理、 buddy系统时调用此技能。"
---

# ext4 Mballoc

## 〇、为什么需要这个机制？

为什么需要多块分配器？ext2/ext3 的单块分配器一次只分配一个块，导致严重的碎片和低效。Mballoc 一次分配多个连续块，减少碎片、提升性能。2025 年的 xarray 优化替代了链表遍历，使组选择算法可以跳过忙组，大幅提升多核扩展性。

单块分配器的问题在于：它不考虑文件未来的访问模式，也不尝试保持文件的连续性。随着文件系统的长期使用，文件变得越来越碎片化，读写性能急剧下降。Mballoc 通过预分配、 locality group、多级搜索标准等策略，显著改善了这一问题。

没有 Mballoc，ext4 在长期使用后会产生严重碎片，大文件的顺序读写性能会大幅下降，多核并发分配也会成为瓶颈。

---

## 一、分配标志 (verified: ext4.h:186-209)

```c
#define EXT4_MB_HINT_MERGE		0x0001	/* prefer goal again */
#define EXT4_MB_HINT_FIRST		0x0008	/* first blocks in the file */
#define EXT4_MB_HINT_DATA		0x0020	/* data is being allocated */
#define EXT4_MB_HINT_NOPREALLOC		0x0040	/* don't preallocate */
#define EXT4_MB_HINT_GROUP_ALLOC	0x0080	/* allocate for locality group */
#define EXT4_MB_HINT_GOAL_ONLY		0x0100	/* allocate goal blocks or none */
#define EXT4_MB_HINT_TRY_GOAL		0x0200	/* goal is meaningful */
#define EXT4_MB_DELALLOC_RESERVED	0x0400	/* blocks already pre-reserved */
#define EXT4_MB_STREAM_ALLOC		0x0800	/* stream allocation */
#define EXT4_MB_USE_ROOT_BLOCKS		0x1000	/* Use reserved root blocks */
#define EXT4_MB_USE_RESERVED		0x2000	/* Use blocks from reserved pool */
#define EXT4_MB_STRICT_CHECK		0x4000	/* strict check for free blocks */
```

---

## 二、搜索标准 (Criteria, verified: ext4.h:137-177)

```c
enum criteria {
	CR_POWER2_ALIGNED,	/* 2的幂次对齐, 最快, 无磁盘IO */
	CR_GOAL_LEN_FAST,	/* 目标长度, 快速, 仅内存 */
	CR_BEST_AVAIL_LEN,	/* 最佳可用长度, 允许缩减 */
	CR_GOAL_LEN_SLOW,	/* 目标长度, 慢速, 需要磁盘IO */
	CR_ANY_FREE,		/* 任意空闲块, 最后手段 */
	EXT4_MB_NUM_CRS,
};

static inline bool ext4_mb_cr_expensive(enum criteria cr)
{
	return cr >= CR_GOAL_LEN_SLOW;
}
```

---

## 三、ext4_allocation_request (verified: ext4.h:211-230)

```c
struct ext4_allocation_request {
	struct inode *inode;
	unsigned int len;
	ext4_lblk_t logical;
	ext4_lblk_t lleft;
	ext4_lblk_t lright;
	ext4_fsblk_t goal;
	ext4_fsblk_t pleft;
	ext4_fsblk_t pright;
	unsigned int flags;
};
```

---

## 四、ext4_sb_info 中的 mballoc 字段 (verified: ext4.h:1605-1661)

```c
struct ext4_sb_info {
	/* buddy allocator */
	struct ext4_group_info ** __rcu *s_group_info;
	struct inode *s_buddy_cache;
	spinlock_t s_md_lock;
	unsigned short *s_mb_offsets;
	unsigned int *s_mb_maxs;
	unsigned int s_group_info_size;
	atomic_t s_mb_free_pending;
	struct list_head s_freed_data_list[2];
	struct list_head s_discard_list;
	struct work_struct s_discard_work;
	atomic_t s_retry_alloc_pending;
	struct xarray *s_mb_avg_fragment_size;	/* xarray, 2025 */
	struct xarray *s_mb_largest_free_orders; /* xarray, 2025 */

	/* tunables */
	unsigned long s_stripe;
	unsigned int s_mb_max_linear_groups;
	unsigned int s_mb_stream_request;
	unsigned int s_mb_max_to_scan;
	unsigned int s_mb_min_to_scan;
	unsigned int s_mb_stats;
	unsigned int s_mb_order2_reqs;
	unsigned int s_mb_group_prealloc;
	unsigned int s_mb_prefetch;
	unsigned int s_mb_prefetch_limit;
	unsigned int s_mb_best_avail_max_trim_order;

	/* 多全局目标, 2025 */
	ext4_group_t *s_mb_last_groups;
	unsigned int s_mb_nr_global_goals;

	/* stats */
	atomic_t s_bal_reqs;
	atomic_t s_bal_success;
	atomic_t s_bal_allocated;
	atomic_t s_bal_ex_scanned;
	atomic_t s_bal_cX_ex_scanned[EXT4_MB_NUM_CRS];
	atomic_t s_bal_groups_scanned;
	atomic_t s_bal_goals;
	atomic_t s_bal_stream_goals;
	atomic_t s_bal_len_goals;
	atomic_t s_bal_breaks;
	atomic_t s_bal_2orders;
	atomic64_t s_bal_cX_groups_considered[EXT4_MB_NUM_CRS];
	atomic64_t s_bal_cX_hits[EXT4_MB_NUM_CRS];
	atomic64_t s_bal_cX_failed[EXT4_MB_NUM_CRS];
	atomic_t s_mb_buddies_generated;
	atomic64_t s_mb_generation_time;
	atomic_t s_mb_lost_chunks;
	atomic_t s_mb_preallocated;
	atomic_t s_mb_discarded;
	atomic_t s_lock_busy;

	/* locality groups */
	struct ext4_locality_group __percpu *s_locality_groups;
};
```

---

## 五、Xarray 优化 (2025)

- **旧**: 链表遍历需要 spin_lock, 无法跳过忙组
- **新**: xarray 有序遍历, 支持 `ext4_try_lock_group()` 跳过忙组
- **s_mb_avg_fragment_size**: 按平均碎片大小排序的 xarray
- **s_mb_largest_free_orders**: 按最大空闲长度排序的 xarray

---

## 六、多全局目标 (2025)

```c
s_mb_nr_global_goals = min(num_possible_cpus(), total_groups / 4);
goal_index = inode->i_ino % s_mb_nr_global_goals;
```

解决单一 `s_mb_last_group` 导致的竞争问题。

---

## 七、关键代码位置

| 功能 | 文件 |
|------|------|
| 核心分配 | fs/ext4/mballoc.c |
| Buddy 系统 | fs/ext4/mballoc.c |
| 头文件 | fs/ext4/ext4.h (标志/结构) |
