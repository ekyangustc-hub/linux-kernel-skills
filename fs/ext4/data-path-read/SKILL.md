---
name: "ext4-data-path-read"
description: "ext4文件系统读取数据路径专家。当用户询问ext4读路径、read_iter、readahead、read_folio、mpage、ES Tree缓存查找、extent树遍历、large folio、fsverity验证、fscrypt解密、bio_post_read时调用此技能。"
---

# ext4 读取数据路径 (Read Data Path)

## 一、为什么需要了解这个路径？

读取路径是文件系统最频繁执行的操作。理解 ext4 读路径有助于:

1. **性能调优**: 了解 readahead 策略、large folio 合并、ES Tree 缓存命中对吞吐的影响
2. **延迟分析**: 区分 extent 树遍历、磁盘 I/O、fsverity 验证、fscrypt 解密各阶段耗时
3. **加密/验证集成**: 理解 fscrypt 和 fsverity 如何在 bio 完成后异步处理
4. **内存管理**: 理解 folio 生命周期、page cache 交互、readahead 窗口调整

---

## 二、完整流程图

```
用户空间 read(fd, buf, count)
  │
  ▼
sys_read() / vfs_read()
  │
  ▼
ext4_file_read_iter()                   [fs/ext4/file.c:131]
  │
  ├── IS_DAX? → ext4_dax_read_iter()    (DAX 直通)
  ├── IOCB_DIRECT? → ext4_dio_read_iter() (直接 I/O)
  └── 默认 → generic_file_read_iter()    (缓冲 I/O)
       │
       ▼
  filemap_read()
       │
       ├── folio 已在 page cache?
       │     └── 直接拷贝到用户空间 (零磁盘访问)
       │
       └── folio 缺失 → filemap_get_folio()
            │
            ▼
       __filemap_get_folio() → 分配 folio
            │
            ▼
       ext4_read_folio()                 [fs/ext4/readpage.c:395]
            │
            ├── inline data? → ext4_readpage_inline()
            ├── fsverity? → fsverity_get_info()
            │
            ▼
       ext4_mpage_readpages()            [fs/ext4/readpage.c:211]
            │
            ├── 遍历 folio 内每个 block
            │    │
            │    ▼
            │  ext4_map_blocks(NULL, inode, &map, 0)
            │    │
            │    ├── ext4_es_lookup_extent()  ← ES Tree 缓存查找 (快速路径)
            │    │     ├── 命中: 直接返回物理块号 + es_seq
            │    │     └── 未命中:
            │    │
            │    └── ext4_ext_map_blocks()    ← extent 树遍历 (慢速路径)
            │         │
            │         ├── ext4_find_extent()  ← B+树从根到叶子
            │         │     └── 读取 extent 块 (可能需要磁盘 I/O)
            │         │
            │         ├── ext4_ext_cache_extent()  ← 回填 ES Tree
            │         └── 返回物理块映射
            │
            ├── 合并连续物理块到单个 bio
            │    └── bio_add_folio()
            │
            ▼
       blk_crypto_submit_bio()             (提交 bio 到块层)
            │
            ▼
       ───────────────── 异步 I/O 分割线 ─────────────────
            │
            ▼
       mpage_end_io() (bio 完成回调)
            │
            └── bio_post_read_processing()  [fs/ext4/readpage.c:56-70]
                 │
                 ├── STEP_DECRYPT (fscrypt)
                 │    └── fscrypt_decrypt_bio()
                 │         └── decrypt_work() (workqueue)
                 │
                 ├── STEP_VERITY (fsverity)
                 │    └── fsverity_verify_bio()
                 │         └── verity_work() (workqueue)
                 │
                 └── __read_end_io()
                      └── folio_end_read(folio, success)
                           │
                           └── 唤醒等待该 folio 的进程
                                │
                                ▼
                           用户空间拿到数据
```

### Readahead 路径

```
readahead (预读)
  │
  ▼
ext4_readahead()                        [fs/ext4/readpage.c:416]
  │
  └── ext4_mpage_readpages(inode, vi, rac, NULL)
       │
       ├── rac (readahead_control) 包含多个待预读的 folio
       ├── 批量映射连续块
       └── 提交单个大 bio (REQ_RAHEAD 标志)
            │
            └── 预读失败不报错, 仅影响后续性能
```

---

## 三、关键函数调用链

```
vfs_read()
  └─► ext4_file_read_iter()                         file.c:131
       └─► generic_file_read_iter()                 file.c:148
            └─► filemap_read()
                 └─► filemap_get_folio()
                      └─► ext4_read_folio()         readpage.c:395
                           └─► ext4_mpage_readpages()  readpage.c:211
                                ├─► ext4_map_blocks()   inode.c
                                │    ├─► ext4_es_lookup_extent()  ← ES Tree 缓存
                                │    └─► ext4_ext_map_blocks()    ← extent 树
                                │         └─► ext4_find_extent()
                                ├─► bio_alloc()
                                ├─► bio_add_folio()
                                └─► blk_crypto_submit_bio()

mpage_end_io() (bio 完成)
  └─► bio_post_read_processing()
       ├─► fscrypt_decrypt_bio() → decrypt_work()
       ├─► fsverity_verify_bio() → verity_work()
       └─► __read_end_io()
            └─► folio_end_read()
```

### 关键函数签名

```c
/* fs/ext4/file.c:131 */
static ssize_t ext4_file_read_iter(struct kiocb *iocb, struct iov_iter *to);

/* fs/ext4/readpage.c:395 */
int ext4_read_folio(struct file *file, struct folio *folio);

/* fs/ext4/readpage.c:211 */
static int ext4_mpage_readpages(struct inode *inode, struct fsverity_info *vi,
        struct readahead_control *rac, struct folio *folio);

/* fs/ext4/inode.c — ext4_map_blocks 核心 */
int ext4_map_blocks(handle_t *handle, struct inode *inode,
                    struct ext4_map_blocks *map, int flags);
```

---

## 四、关键数据结构

### ext4_map_blocks (映射结果)

```c
/* fs/ext4/ext4.h */
struct ext4_map_blocks {
    ext4_fsblk_t m_pblk;       /* 物理块号 */
    ext4_lblk_t  m_lblk;       /* 逻辑块号 */
    unsigned int m_len;        /* 连续块数 */
    unsigned int m_flags;      /* EXT4_MAP_MAPPED/NEW/UNWRITTEN/DELAYED */
    u64          m_seq;        /* 序列号 (并发 I/O 安全) */
};
```

### ext4_extent_status (ES Tree 条目)

```c
/* fs/ext4/extents_status.c */
struct ext4_extent_status {
    struct rb_node   rb_node;    /* 红黑树节点 */
    ext4_lblk_t      es_lblk;    /* 起始逻辑块 */
    ext4_fsblk_t     es_pblk;    /* 起始物理块 */
    ext4_lblk_t      es_len;     /* 长度 */
    unsigned int     es_seq;     /* 序列号 (validity cookie) */
    unsigned short   es_type;    /* WRITTEN/UNWRITTEN/DELAYED/HOLE */
};
```

### bio_post_read_ctx (读后处理上下文)

```c
/* fs/ext4/readpage.c:64 */
enum bio_post_read_step {
    STEP_INITIAL = 0,
    STEP_DECRYPT,      /* fscrypt 解密 */
    STEP_VERITY,       /* fsverity 验证 */
    STEP_MAX,
};

struct bio_post_read_ctx {
    struct bio         *bio;
    struct fsverity_info *vi;
    struct work_struct work;
    unsigned int       cur_step;
    unsigned int       enabled_steps;  /* 位掩码 */
};
```

### ext4_ext_path (extent 树遍历路径)

```c
/* fs/ext4/ext4_extents.h */
struct ext4_ext_path {
    ext4_fsblk_t           p_block;    /* 当前块物理地址 */
    __u16                  p_depth;    /* 当前深度 */
    __u16                  p_maxdepth; /* 树最大深度 */
    struct ext4_extent     *p_ext;     /* 叶子 extent 指针 */
    struct ext4_extent_idx *p_idx;     /* 索引项指针 */
    struct ext4_extent_header *p_hdr;  /* 当前块头 */
    struct buffer_head     *p_bh;      /* 当前块 buffer */
};
```

---

## 五、并发与锁

### 锁层次

```
i_rwsem (inode 读锁, 多读者可并发)
  │
  └── i_data_sem (读锁, 保护 extent 树)
       │
       └── i_es_lock (rwlock, 保护 ES Tree)
```

### 并发场景

| 场景 | 锁 | 说明 |
|------|-----|------|
| 多进程读同一文件 | i_rwsem 共享 | 可并发读 |
| 读 + 写 | i_rwsem 互斥 | 写者排他 |
| 读 + truncate | i_data_sem 互斥 | truncate 需要写锁 |
| ES Tree 查找 | i_es_lock 读锁 | 无锁快速路径 (RCU-like) |
| ES Tree 插入 | i_es_lock 写锁 | 修改时持有 |

### 序列号并发安全 (2025)

```c
/* 读取路径中 es_seq 的使用 */
int ext4_es_lookup_extent(struct inode *inode, ext4_lblk_t lblk,
                           ext4_lblk_t *es_seq);
```

ES Tree 查找返回 `es_seq`，调用者可在后续操作中验证 extent 信息是否仍然有效。

---

## 六、优化点 (2014-2026)

| 年份 | 优化 | 效果 |
|------|------|------|
| 2014 | ext4_mpage_readpages 替代 mpage_readpages | 支持加密文件读 |
| 2016 | fscrypt 集成 (bio_post_read_ctx) | 异步解密, 不阻塞 I/O |
| 2018 | fsverity 集成 | Merkle 树验证, 异步验证 |
| 2019 | ES Tree 缓存优化 | 减少 extent 树遍历 ~70% |
| 2021 | readahead 改进 (readahead_control) | 更好的预读窗口 |
| 2023 | Large folio 支持 (5.16+) | 减少 page cache 开销 ~30% |
| 2024 | blk_crypto 集成 | 硬件加速加密/解密 |
| 2025 | ES Tree seq counter | 并发 I/O 安全, 支持 iomap |
| 2025 | Large folio read path | EXT4_LBLK_TO_PG 宏, 块大小>页大小 |

### Large Folio 关键变更 (2025)

```c
/* 块号到 folio page 索引的转换 */
#define EXT4_PG_TO_LBLK(inode, pg_idx) \
    ((pg_idx) << (PAGE_SHIFT - (inode)->i_blkbits))

/* 支持 large folio 的 readahead */
void ext4_readahead(struct readahead_control *rac);
```

---

## 七、常见陷阱

### 1. Hole 处理

```
hole (未分配块) 读取时:
  - ext4_map_blocks() 返回 m_flags = 0 (未映射)
  - ext4_mpage_readpages() 调用 folio_zero_segment() 清零
  - 不会提交磁盘 I/O
```

### 2. fsverity 验证失败

```
fsverity_verify_folio() 返回 false 时:
  - folio 标记为错误
  - 返回 -EIO 给用户
  - 不会将损坏数据暴露给用户空间
```

### 3. 加密文件读

```
fscrypt 解密在 bio 完成后异步执行:
  - 如果解密失败, folio 标记错误
  - 密钥未加载时返回 -ENOKEY
  - 解密使用 workqueue, 可能增加延迟
```

### 4. ES Tree 缓存失效

```
并发 truncate 可能导致 ES Tree 缓存失效:
  - 查找返回旧映射
  - 序列号机制检测变更 (2025+)
  - 旧内核可能读到错误数据
```

### 5. Large Folio 对齐

```
Large folio 需要正确对齐:
  - folio 大小必须是块大小的整数倍
  - EXT4_PG_TO_LBLK 宏处理偏移转换
  - 未对齐访问导致 confused 回退路径
```

---

## 八、关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| ext4_file_read_iter | fs/ext4/file.c | 131-149 |
| ext4_read_folio | fs/ext4/readpage.c | 395-414 |
| ext4_readahead | fs/ext4/readpage.c | 416-434 |
| ext4_mpage_readpages | fs/ext4/readpage.c | 211-393 |
| bio_post_read_ctx | fs/ext4/readpage.c | 56-120 |
| ext4_map_blocks | fs/ext4/inode.c | ~1900 |
| ext4_es_lookup_extent | fs/ext4/extents_status.c | ~350 |
| ext4_ext_map_blocks | fs/ext4/extents.c | ~3900 |
| ext4_find_extent | fs/ext4/extents.c | ~1700 |
| mpage_end_io | fs/ext4/readpage.c | ~180 |
