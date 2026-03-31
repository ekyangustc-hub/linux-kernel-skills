---
name: "ext4-mballoc"
description: "ext4多块分配器(Mballoc)专家。当用户询问ext4块分配策略、mb_optimize_scan、组选择算法、xarray优化、全局目标、碎片管理、 buddy系统时调用此技能。"
---

# ext4 Mballoc (多块分配器)

## 一、概述

Mballoc (Multi-Block Allocator) 是 ext4 的核心块分配子系统，负责为文件数据分配连续的物理块。近年来经历了大规模重构，引入 xarray、多全局目标、跳过忙组等优化。

---

## 二、核心数据结构

### 2.1 ext4_allocation_context

**位置**: `fs/ext4/mballoc.c`

```c
struct ext4_allocation_context {
    struct inode *ac_inode;           /* 请求分配的 inode */
    struct super_block *ac_sb;        /* 超级块 */

    ext4_lblk_t ac_o_ex.fe_logical;   /* 原始请求逻辑块 */
    ext4_fsblk_t ac_o_ex.fe_start;    /* 原始请求物理块 */
    ext4_group_t ac_o_ex.fe_group;    /* 原始请求块组 */

    ext4_lblk_t ac_g_ex.fe_logical;   /* 目标逻辑块 */
    ext4_fsblk_t ac_g_ex.fe_start;    /* 目标物理块 */
    ext4_group_t ac_g_ex.fe_group;    /* 目标块组 */

    ext4_lblk_t ac_f_ex.fe_logical;   /* 最终分配逻辑块 */
    ext4_fsblk_t ac_f_ex.fe_start;    /* 最终分配物理块 */
    ext4_group_t ac_f_ex.fe_group;    /* 最终分配块组 */

    ext4_lblk_t ac_len;               /* 请求块数 */
    ext4_lblk_t ac_glen;              /* 目标块数 */
    ext4_lblk_t ac_b_ex.fe_len;       /* 实际分配块数 */

    unsigned int ac_flags;            /* 分配标志 */
    unsigned int ac_criteria;         /* 当前搜索标准 (CR) */
    ext4_group_t ac_last_group;       /* 上次搜索的块组 */
    ext4_group_t ac_groups_scanned;   /* 已扫描的块组数 */
    ...
};
```

### 2.2 搜索标准 (Criteria)

```c
/* 搜索标准的优先级顺序，从严格到宽松 */
#define CR_POWER2_ALIGNED       0   /* 2的幂次对齐 */
#define CR_GOAL_LEN_SLOW        1   /* 目标长度 (慢速) */
#define CR_GOAL_LEN_FAST        2   /* 目标长度 (快速) */
#define CR_BEST_AVAIL_LEN       3   /* 最佳可用长度 */
#define CR_ANY_FREE             4   /* 任意空闲块 */
```

### 2.3 Buddy 系统

```c
struct ext4_buddy {
    struct super_block *bd_sb;
    ext4_group_t bd_group;          /* 块组号 */
    struct ext4_group_info *bd_info; /* 组信息 */
    void *bd_bitmap;                /* buddy 位图 */
    struct page *bd_buddy_page;     /* buddy 页 */
    struct page *bd_bitmap_page;    /* 位图页 */
    ...
};
```

---

## 三、块分配流程

### 3.1 整体流程

```
ext4_mb_new_blocks()
    │
    ├── 1. 初始化分配上下文 (ac)
    │   ├── 设置 ac_o_ex (原始请求)
    │   ├── 设置 ac_g_ex (目标请求)
    │   └── 计算 ac_flags
    │
    ├── 2. ext4_mb_regular_allocator()
    │   │
    │   ├── 2.1 预取 (Prefetch)
    │   │   └── ext4_mb_prefetch()
    │   │
    │   ├── 2.2 组选择循环
    │   │   └── ext4_mb_scan_groups()
    │   │       │
    │   │       ├── 线性扫描 (ext4_mb_scan_groups_linear)
    │   │       │   └── 从指定组开始，扫描最多 s_mb_max_linear_groups 个组
    │   │       │
    │   │       └── 非线性扫描 (ext4_mb_scan_group)
    │   │           ├── 使用 xarray 有序遍历
    │   │           ├── 使用 ext4_try_lock_group() 跳过忙组
    │   │           └── 多全局目标减少竞争
    │   │
    │   └── 2.3 Buddy 分配
    │       └── ext4_mb_simple_scan_group()
    │
    ├── 3. ext4_mb_mark_diskspace_used()
    │   └── 更新位图和元数据
    │
    └── 4. 返回分配结果
```

### 3.2 组选择算法重构 (scan group)

**旧流程 (choose group)**:
```
选择"好组" → 扫描该组 → 失败则重新选择
问题: 在列表遍历时持有 spin_lock，无法使用 ext4_try_lock_group()，
      导致"弹跳"问题 (反复从同一组开始)
```

**新流程 (scan group)**:
```
ext4_mb_scan_groups()
    │
    ├── ext4_mb_scan_groups_linear()
    │   └── 从指定组开始线性扫描，最多 s_mb_max_linear_groups 次
    │
    └── ext4_mb_scan_group()
        └── 使用 xarray 有序遍历，支持 ext4_try_lock_group() 跳过忙组
```

---

## 四、Xarray 优化

### 4.1 概述

将空闲组列表从链表转换为有序 xarray，实现无锁遍历。

**位置**: `fs/ext4/mballoc.c`

```c
/* xarray 结构 */
struct xarray s_mb_avg_fragment_size_order[MAX_ORDER];  /* 按平均碎片大小排序 */
struct xarray s_mb_largest_free_orders[MAX_ORDER];      /* 按最大空闲长度排序 */

/* xarray 索引 = 块组号, 值 = 块组信息 */
/* 非空值表示该块组在列表中 */
```

### 4.2 优势

| 特性 | 旧链表 | 新 xarray |
|------|--------|-----------|
| 遍历方式 | 需要 spin_lock | 无锁有序遍历 |
| 忙组处理 | 无法跳过 | ext4_try_lock_group() 跳过 |
| 插入/删除 | O(1) | O(1) |
| 查找 | O(1) | O(nlogn) |
| 多进程性能 | 差 (弹跳问题) | 好 (线性遍历) |

### 4.3 查找函数

```c
/* 从指定位置开始在 xarray 中查找好组 */
ext4_group_t ext4_mb_find_good_group_xarray(struct xarray *xa,
                                            ext4_group_t start,
                                            ext4_group_t ngroups);
/* 到达 ngroups-1 后回绕到 0，再到 start-1，确保有序遍历 */
```

---

## 五、多全局目标 (Multiple Global Goals)

### 5.1 问题

单一全局目标 (`s_mb_last_group`) 导致多进程竞争同一块组，降低 extent 合并概率，增加文件碎片。

### 5.2 解决方案

```c
/* 全局目标数量 = min(可能CPU数, 总块组数/4) */
s_mb_num_global_goals = min(num_possible_cpus(), total_groups / 4);

/* 为每个 inode 选择固定目标 */
goal_index = inode->i_ino % s_mb_num_global_goals;
```

### 5.3 性能提升

| 平台 | 场景 | 旧 | 新 | 提升 |
|------|------|----|----|------|
| Kunpeng 920 | P80, mb_optimize_scan=0 | 9636 | 19628 | +103% |
| AMD 9654*2 | P96, mb_optimize_scan=0 | 22341 | 53760 | +140% |
| AMD 9654*2 | P96, mb_optimize_scan=1 | 9177 | 12716 | +38.5% |

---

## 六、跳过忙组 (ext4_try_lock_group)

### 6.1 概述

当多个进程/容器执行相似分配模式时，会竞争同一块组。`ext4_try_lock_group()` 在组被锁定时跳过，避免竞争。

```c
int ext4_try_lock_group(struct ext4_buddy *e4b, struct ext4_group_info *grp);
/* 返回 0 表示成功获取锁，-EBUSY 表示组忙应跳过 */
```

### 6.2 使用限制

- `ac_criteria == CR_ANY_FREE` 时不跳过忙组 (确保能找到空闲块)
- 仅在非线扫描时使用

### 6.3 性能提升

| 平台 | 场景 | 旧 | 新 | 提升 |
|------|------|----|----|------|
| AMD 9654*2 | P96, mb_optimize_scan=0 | 3450 | 15371 | +345% |
| Kunpeng 920 | P80, mb_optimize_scan=0 | 2667 | 4821 | +80.7% |

---

## 七、空闲 extent 合并优化

### 7.1 概述

在插入新的空闲 extent 前，先与已存在的空闲 extent 合并，减少锁持有次数。

**旧流程**:
```
prev 合并到 new → 持锁
next 合并到 new → 持锁
插入 new → 持锁
总计: 3次持锁
```

**新流程**:
```
new 合并到 next → 无锁
next 合并到 prev → 持锁
总计: 1次持锁
```

---

## 八、Mount 选项

### 8.1 mb_optimize_scan

```bash
# 启用优化扫描 (默认启用)
mount -o mb_optimize_scan /dev/sdX /mnt

# 禁用优化扫描 (使用传统线性扫描)
mount -o no_mb_optimize_scan /dev/sdX /mnt
```

**行为变化**:
- 优化扫描对 indirect block inode 和 extent inode 都生效 (2026年变更)
- 传统模式仅对 extent inode 生效

---

## 九、关键代码位置

| 功能 | 文件路径 |
|------|----------|
| 核心分配逻辑 | `fs/ext4/mballoc.c` |
| Buddy 系统 | `fs/ext4/mballoc.c` |
| 组选择/扫描 | `fs/ext4/mballoc.c` |
| Xarray 管理 | `fs/ext4/mballoc.c` |
| KUnit 测试 | `fs/ext4/mballoc-test.c` |
| 头文件定义 | `fs/ext4/mballoc.h` |

---

## 十、参考资源

- ext4 官方文档: `Documentation/filesystems/ext4/`
- ext4 内核代码: `fs/ext4/mballoc.c`
- 补丁系列: "ext4: mballoc improvements" (2025)
