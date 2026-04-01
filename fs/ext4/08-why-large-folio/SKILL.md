---
name: ext4-why-large-folio
description: ext4大块大小支持存在必要性专家。当用户询问为什么ext4需要large folio、BS>PS问题、64K页支持、2025年新特性时调用此技能。
---

# 为什么 ext4 需要 Large Folio (BS > PS)

## 1. 问题起源

Linux 内核长期假设 **页大小 (PAGE_SIZE) >= 文件系统块大小 (block_size)**：

```
传统假设:
  PAGE_SIZE = 4K (x86) 或 64K (ARM64)
  block_size = 1K, 2K, 4K
  关系: PAGE_SIZE >= block_size

这意味着:
  1 个页可以容纳 1 个或多个文件系统块
  页缓存管理简单: 1 个页 = N 个块
```

### 问题出现：64K 页系统 + 4K 块文件系统

```
ARM64 服务器配置:
  PAGE_SIZE = 64K
  磁盘扇区 = 4K
  ext4 block_size = 4K

问题:
  1 个页 = 16 个块
  页缓存粒度太粗:
  - 读取 4K 数据需要分配 64K 页
  - 写入 4K 数据需要读取-修改-写入 64K 页
  - 内存浪费: 15/16 的页空间未使用
```

### 问题出现：大容量 NVMe 设备

```
现代 NVMe SSD:
  物理扇区大小 = 4K
  最优 I/O 大小 = 256K-1MB
  设备支持 64K 逻辑块大小

传统 ext4 的限制:
  block_size 最大 = PAGE_SIZE
  x86: 最大 4K block
  ARM64: 最大 64K block

问题:
  4K block 在 100TB 设备上:
  - 块号需要 32 位以上
  - 元数据开销巨大
  - I/O 效率低
```

### 具体问题

```
问题 1: 内存浪费
  64K 页 + 4K 块:
  读取 4K 数据 → 分配 64K 页 → 浪费 60K

问题 2: I/O 放大
  写入 4K 数据:
  1. 读取 64K 页 (包含目标 4K 块)
  2. 修改 4K
  3. 写回 64K
  I/O 放大: 16x

问题 3: 并发问题
  同一页内的不同块被不同进程访问:
  - 需要页级锁，而非块级锁
  - 并发度降低

问题 4: 设备支持
  某些设备原生支持 64K 块:
  - 无法充分利用设备特性
  - 需要额外的转换层
```

## 2. 为什么需要 Large Folio (BS > PS)

Large Folio 解决的核心问题：**支持文件系统块大小大于页大小**。

| 问题 | 传统 ext4 | Large Folio |
|------|----------|-------------|
| 块大小限制 | block_size <= PAGE_SIZE | block_size > PAGE_SIZE |
| 内存效率 | 64K 页 + 4K 块 = 浪费 | 精确分配 |
| I/O 效率 | 读-修改-写放大 | 直接 I/O |
| 并发 | 页级锁 | folio 级锁 |
| 设备支持 | 无法利用大块设备 | 原生支持 |

Large Folio 由 Matthew Wilcox 等人在 2023-2025 年开发，是内核内存管理子系统的重大改进。

### Folio 概念

```
Page (传统):
  struct page - 4K 内存单元
  页缓存以 page 为单位

Folio (新):
  struct folio - N 个连续 page 的集合
  可以是 4K, 8K, 16K, 64K, 2MB, ...
  页缓存以 folio 为单位

Folio 的优势:
  1. 减少 struct page 开销 (1 个 folio vs N 个 page)
  2. 减少页缓存条目 (1 个条目 vs N 个条目)
  3. 减少锁竞争 (1 个锁 vs N 个锁)
  4. 支持大块大小 (BS > PS)
```

## 3. 核心设计

Large Folio 的核心架构：

```
传统架构 (BS <= PS):
  页缓存: [page][page][page][page]  (每个 page = 4K)
  块映射:  block0  block1  block2  block3
  关系:    1 page = 1 block (4K = 4K)

Large Folio 架构 (BS > PS):
  页缓存: [folio (16 pages = 64K)]
  块映射:  block0-block15 (16 blocks = 64K)
  关系:    1 folio = 16 blocks (64K = 64K)

或者 (BS = 256K, PS = 4K):
  页缓存: [folio (64 pages = 256K)]
  块映射:  block0-block63 (64 blocks = 256K)
  关系:    1 folio = 64 blocks (256K = 256K)
```

关键设计决策：

1. **Folio 替代 Page**：页缓存全面使用 folio
2. **块映射适配**：extent 树映射到 folio 而非 page
3. **I/O 路径适配**：读写操作按 folio 大小对齐
4. **向后兼容**：传统配置 (BS <= PS) 继续工作

## 4. 关键数据结构

### Folio 结构

```c
/* include/linux/pagemap.h */
struct folio {
	struct page page;           /* 第一个 page (嵌入) */
	/* folio 特有的字段 */
	unsigned long _flags_1;     /* 标志 */
	struct address_space *mapping; /* 地址空间 */
	pgoff_t index;              /* 在地址空间中的索引 */
	struct {
		unsigned char flags;
		unsigned char nr_pages; /* folio 包含的 page 数 */
		unsigned char large_rmappable;
		unsigned char dma_pin;
	};
	/* ... */
};

/* 获取 folio 大小 */
static inline unsigned int folio_size(const struct folio *folio)
{
	return folio_nr_pages(folio) * PAGE_SIZE;
}

/* 判断是否为大 folio */
static inline bool folio_test_large(const struct folio *folio)
{
	return folio_nr_pages(folio) > 1;
}
```

### 超级块中的 Large Folio 字段

```c
/* fs/ext4/ext4.h */
struct ext4_sb_info {
	/* ... */
	unsigned int s_min_folio_order; /* 最小 folio order */
	/* ... */
};

/* 计算最小 folio 大小 */
#define EXT4_MIN_FOLIO_SIZE(sb) (PAGE_SIZE << EXT4_SB(sb)->s_min_folio_order)

/* 逻辑块到 folio 的转换 */
#define EXT4_LBLK_TO_PG(sbi, lblk) \
	((lblk) >> (sbi)->s_min_folio_order)
#define EXT4_PG_TO_LBLK(sbi, pg) \
	((pg) << (sbi)->s_min_folio_order)
```

### 地址空间操作

```c
/* fs/ext4/inode.c */
static const struct address_space_operations ext4_aops = {
	.read_folio      = ext4_read_folio,
	.readahead       = ext4_readahead,
	.writepages      = ext4_writepages,
	.write_begin     = ext4_write_begin,
	.write_end       = ext4_write_end,
	/* ... */
};

/* 读取 folio */
static int ext4_read_folio(struct file *file, struct folio *folio)
{
	/* 读取整个 folio (可能包含多个 block) */
	return ext4_readpage(file, &folio->page);
}

/* 预读 */
static void ext4_readahead(struct readahead_control *rac)
{
	/* 预读整个 folio 范围 */
	ext4_mpage_readpages(rac);
}
```

### I/O 提交

```c
/* fs/ext4/page-io.c */
struct ext4_io_end {
	struct list_head list;      /* I/O 链表 */
	struct inode *inode;        /* 目标 inode */
	ext4_io_end_t *next;        /* 下一个 */
	ext4_lblk_t block;          /* 起始逻辑块 */
	size_t size;                /* I/O 大小 */
	struct bio *bio;            /* 关联的 bio */
	/* ... */
};

/* Large Folio 下的 I/O 提交 */
static int ext4_io_submit(struct ext4_io_end *io)
{
	/* 提交整个 folio 的 I/O */
	/* bio 大小适配 folio 大小 */
}
```

## 5. 10年演进 (2015-2025)

| 年份 | 内核版本 | 变更 | 解决的问题 |
|------|---------|------|-----------|
| 2015 | 4.1 | 页缓存改进 | 为 folio 铺路 |
| 2018 | 4.19 | 大页缓存实验 | 初步探索 |
| 2020 | 5.8 | folio 概念引入 | Matthew Wilcox 提出 |
| 2021 | 5.12 | folio 初步实现 | 替代部分 page 操作 |
| 2022 | 5.17 | 页缓存 folio 化 | 全面使用 folio |
| 2023 | 6.2 | ext4 folio 适配 | ext4 开始使用 folio API |
| 2024 | 6.6 | BS>PS 实验支持 | 块大小大于页大小 |
| 2025 | 6.12 | Large Folio 正式支持 | BS>PS 生产就绪 |

### Folio 演进的里程碑

```
2020: struct folio 引入
  - 作为 struct page 的包装
  - 不改变现有行为

2022: 页缓存 folio 化
  - page_cache_get/put → folio_get/put
  - find_get_page → find_get_folio
  - 全面替换 page API

2024: BS>PS 支持
  - block_size 可以大于 PAGE_SIZE
  - 需要 folio 大小 >= block_size
  - ext4 适配新的块映射逻辑

2025: 生产就绪
  - 所有 I/O 路径适配
  - 性能测试通过
  - 稳定性验证
```

## 6. 与其他特性的关系

```
Large Folio
  │
  ├── extent: extent 树映射到 folio 而非 page
  │     └─→ 单个 extent 可能覆盖多个 folio
  │     └─→ 查找逻辑需要适配
  │
  ├── mballoc: 分配需要 folio 对齐
  │     └─→ 最小分配单位 = folio 大小
  │
  ├── JBD2: 日志块大小需要适配
  │     └─→ 日志 I/O 按 folio 大小对齐
  │
  ├── metadata_csum: 校验和计算适配大块大小
  │
  ├── fast_commit: FC 记录适配大块大小
  │
  ├── encrypt: 加解密操作按 folio 大小对齐
  │     └─→ 加密粒度可能大于 page 大小
  │
  └── bigalloc: cluster 大小与 folio 大小的关系
        └─→ 需要协调分配粒度
```

## 7. 关键代码位置

```
fs/ext4/
├── inode.c            # folio 读写操作
├── readpage.c         # folio 读取
├── writeback.c        # folio 回写
├── page-io.c          # folio I/O 提交
├── extents.c          # extent 到 folio 映射
├── mballoc.c          # folio 对齐分配
└── super.c            # folio 初始化

mm/
├── filemap.c          # 页缓存 folio API
├── folio-compat.c     # 兼容性层
└── large_folio.c      # 大 folio 管理

include/linux/
├── folio.h            # folio 核心头文件
├── pagemap.h          # 页缓存 folio API
└── fs.h               # 文件系统 folio 接口

关键函数:
  ext4_read_folio()            # 读取 folio
  ext4_writepages()            # 回写 folio
  ext4_write_begin()           # 开始写入 folio
  ext4_write_end()             # 结束写入 folio
  ext4_map_blocks()            # 映射块到 folio
  ext4_io_submit()             # 提交 folio I/O
  folio_start_writeback()      # 开始 folio 回写
  folio_end_writeback()        # 结束 folio 回写
```
