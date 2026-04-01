---
name: "ext4-data-path-write"
description: "ext4文件系统写入数据路径专家。当用户询问ext4写路径、write_iter、buffered write、direct I/O、delalloc延迟分配、writeback、writepages、unwritten extent转换、data=journal/writeback/ordered模式、iomap迁移时调用此技能。"
---

# ext4 写入数据路径 (Write Data Path)

## 一、为什么需要了解这个路径？

写入路径是文件系统最复杂的操作之一。理解 ext4 写路径有助于:

1. **性能调优**: 理解 delalloc、multi-block allocation、writeback 策略对吞吐和延迟的影响
2. **数据一致性**: 理解 data=journal/writeback/ordered 三种模式对崩溃一致性的保证
3. **空间管理**: 理解延迟分配如何减少碎片、提高分配效率
4. **Unwritten extent**: 理解预分配+异步转换机制如何保证原子性和性能
5. **iomap 迁移**: 理解 ext4 从 buffer_head 到 iomap 的演进路径

---

## 二、完整流程图

### 2.1 缓冲写 (Buffered Write) 路径

```
用户空间 write(fd, buf, count)
  │
  ▼
sys_write() / vfs_write()
  │
  ▼
ext4_file_write_iter()                  [fs/ext4/file.c:690]
  │
  ├── IS_DAX? → ext4_dax_write_iter()   (DAX 直通)
  ├── IOCB_DIRECT? → ext4_dio_write_iter() (直接 I/O)
  └── 默认 → ext4_buffered_write_iter()  (缓冲写)
       │
       ▼
  inode_lock(inode)
       │
       ▼
  ext4_write_checks()                   (权限、大小检查)
       │
       ▼
  generic_perform_write()               (VFS 通用写循环)
       │
       ├── 循环处理 iov_iter 中的每个 segment
       │    │
       │    ▼
       │  __generic_file_write_iter()
       │    │
       │    ├── fault_in_iov_iter_readable()  (预取用户数据)
       │    │
       │    ▼
       │  ext4_write_begin() / ext4_da_write_begin()
       │    │
       │    ├── 检查 inline data
       │    │    └── ext4_generic_write_inline_data()
       │    │
       │    ├── 获取/创建 folio
       │    │    └── write_begin_get_folio()
       │    │
       │    ▼
       │  ext4_block_write_begin()
       │    │
       │    └── ext4_da_get_block_prep()  ← 延迟分配映射
       │         │
       │         ├── ext4_da_map_blocks()
       │         │    │
       │         │    ├── ext4_es_lookup_extent()  ← ES Tree 查找
       │         │    │    ├── DELAYED 状态? → 已预留, 直接返回
       │         │    │    └── 未找到:
       │         │    │
       │         │    └── ext4_mb_new_blocks()  ← 多块分配器
       │         │         │
       │         │         ├── 分配物理块
       │         │         ├── 标记为 unwritten extent
       │         │         └── ext4_es_insert_extent(DELAYED)
       │         │
       │         └── 返回映射 (m_flags = EXT4_MAP_DELAYED)
       │
       │    ▼
       │  copy_folio_from_iter()          (拷贝用户数据到 folio)
       │    │
       │    ▼
       │  ext4_write_end()
       │    │
       │    ├── 标记 folio dirty
       │    │    └── filemap_dirty_folio()
       │    │
       │    └── 更新 i_size (如需要)
       │
       ▼
  inode_unlock(inode)
       │
       ▼
  generic_write_sync()                    (O_SYNC/O_DSYNC 处理)
       │
       └── filemap_write_and_wait_range()
            └── ext4_sync_file() (fsync)
```

### 2.2 Writeback (回写) 路径

```
内核回写线程 (flush-ext4/sdX)
  │
  ▼
balance_dirty_pages() 触发回写
  │
  ▼
ext4_writepages()                        [fs/ext4/inode.c:3002]
  │
  ├── ext4_writepages_down_read(sb)      (获取 mballoc 读锁)
  │
  ▼
  writeback_iter() / mpage_prepare_extent_to_map()
  │
  ├── 遍历 dirty folio
  │    │
  │    ├── ext4_map_blocks() with EXT4_GET_BLOCKS_CREATE
  │    │    │
  │    │    ├── 如果 DELAYED: 实际分配物理块
  │    │    │    └── ext4_mb_new_blocks()
  │    │    │
  │    │    └── 创建 unwritten extent
  │    │
  │    └── mpage_map_and_submit_buffers()
  │         │
  │         ├── 构建 bio (写操作)
  │         ├── ext4_io_submit()
  │         └── 提交到块层
  │
  ▼
  ───────────────── 异步 I/O 分割线 ─────────────────
  │
  ▼
  ext4_end_io_end() / ext4_end_bio()     (I/O 完成回调)
  │
  ├── 如果 unwritten extent:
  │    └── ext4_convert_unwritten_io_end_vec()
  │         │
  │         ├── 启动 journal 事务
  │         ├── ext4_ext_convert_unwritten()
  │         │    └── 将 unwritten → written
  │         │         ├── 分裂 extent (如需要)
  │         │         └── 更新 ee_len 标志位
  │         ├── 提交事务
  │         └── ext4_es_remove_extent()  ← 清理 ES Tree
  │
  ▼
  ext4_writepages_up_read(sb)
```

### 2.3 直接 I/O (Direct I/O) 路径

```
ext4_dio_write_iter()
  │
  ├── 检查对齐 (块大小对齐)
  ├── 如果未对齐: 回退到 buffered write
  │
  ▼
  iomap_dio_rw()
  │
  ├── ext4_iomap_ops (.iomap_begin / .iomap_end)
  │    │
  │    ├── ext4_iomap_begin()
  │    │    └── ext4_map_blocks() with EXT4_GET_BLOCKS_CREATE
  │    │         └── 分配 unwritten extent
  │    │
  │    └── ext4_iomap_end()
  │         └── ext4_handle_inode_extension() (扩展文件大小)
  │
  ▼
  直接提交 I/O 到块层 (绕过 page cache)
  │
  ▼
  I/O 完成 → ext4_dio_write_end_io()
  │
  └── ext4_convert_unwritten_io_end_vec()
       └── unwritten → written 转换
```

---

## 三、关键函数调用链

### 缓冲写核心链

```
vfs_write()
  └─► ext4_file_write_iter()                        file.c:690-720
       └─► ext4_buffered_write_iter()               file.c:286-307
            ├─► inode_lock()
            ├─► ext4_write_checks()
            ├─► generic_perform_write()
            │    └─► ext4_da_write_begin()          inode.c:3105-3167
            │         ├─► write_begin_get_folio()
            │         ├─► ext4_block_write_begin()
            │         │    └─► ext4_da_get_block_prep()
            │         │         └─► ext4_da_map_blocks()
            │         │              ├─► ext4_es_lookup_extent()
            │         │              └─► ext4_mb_new_blocks()
            │         └─► 标记 DELAYED 状态
            │
            ├─► copy_folio_from_iter()
            │
            └─► ext4_write_end()
                 └─► filemap_dirty_folio()

回写核心链:
  └─► ext4_writepages()                             inode.c:3002
       ├─► mpage_prepare_extent_to_map()
       ├─► ext4_map_blocks() (CREATE flag)
       ├─► mpage_map_and_submit_buffers()
       │    └─► ext4_io_submit()
       └─► ext4_end_io_end() (end_io 回调)
            └─► ext4_convert_unwritten_io_end_vec()
                 └─► ext4_ext_convert_unwritten()
```

### 关键函数签名

```c
/* fs/ext4/file.c:690 */
static ssize_t ext4_file_write_iter(struct kiocb *iocb, struct iov_iter *from);

/* fs/ext4/file.c:286 */
static ssize_t ext4_buffered_write_iter(struct kiocb *iocb,
                    struct iov_iter *from);

/* fs/ext4/inode.c:3105 */
static int ext4_da_write_begin(const struct kiocb *iocb,
        struct address_space *mapping, loff_t pos, unsigned len,
        struct folio **foliop, void **fsdata);

/* fs/ext4/inode.c:3002 */
static int ext4_writepages(struct address_space *mapping,
                           struct writeback_control *wbc);
```

---

## 四、关键数据结构

### ext4_map_blocks (映射请求/结果)

```c
/* fs/ext4/ext4.h */
struct ext4_map_blocks {
    ext4_fsblk_t m_pblk;       /* 输出: 物理块号 */
    ext4_lblk_t  m_lblk;       /* 输入: 逻辑块号 */
    unsigned int m_len;        /* 输入/输出: 块数 */
    unsigned int m_flags;      /* MAP_NEW/MAP_MAPPED/MAP_UNWRITTEN/MAP_DELAYED */
    u64          m_seq;        /* 序列号 */
};
```

### ext4_extents (Unwritten Extent 标志)

```c
/* fs/ext4/ext4_extents.h */
#define EXT_INIT_MAX_LEN    (1UL << 15)    /* 32768 */

/* ee_len > 0x8000 表示 unwritten extent */
/* ee_len <= 0x8000 表示 written extent */
```

### ext4_inode_info (写相关字段)

```c
/* fs/ext4/ext4.h */
struct ext4_inode_info {
    ...
    spinlock_t i_block_reservation_lock;
    unsigned int i_reserved_data_blocks;  /* 预留数据块数 */
    unsigned int i_allocated_meta_blocks; /* 已分配元数据块数 */
    ...
    struct ext4_es_tree i_es_tree;        /* Extent Status Tree */
    rwlock_t i_es_lock;
    ...
};
```

### ext4_io_end (I/O 完成上下文)

```c
/* fs/ext4/ext4.h */
typedef struct ext4_io_end {
    struct list_head    list;       /* 链接到 inode 的 io_end 列表 */
    struct inode        *inode;     /* 关联 inode */
    struct kiocb        *iocb;      /* 关联 kiocb (DIO) */
    int                 result;     /* I/O 结果 */
    ext4_io_end_vec_t   *io_end_vec;/* 向量列表 (多个范围) */
    ...
} ext4_io_end_t;
```

### ext4_get_blocks 标志

```c
/* fs/ext4/ext4.h:697-739 */
#define EXT4_GET_BLOCKS_CREATE          0x0001  /* 创建新映射 */
#define EXT4_GET_BLOCKS_UNWRIT_EXT      0x0002  /* 创建 unwritten */
#define EXT4_GET_BLOCKS_DELALLOC_RESERVE 0x0004 /* delalloc 预留 */
#define EXT4_GET_BLOCKS_CONVERT         0x0010  /* 转换 unwritten */
#define EXT4_GET_BLOCKS_IO_SUBMIT       0x0400  /* I/O 提交路径 */
```

---

## 五、并发与锁

### 锁层次

```
i_rwsem (inode 写锁, 排他)
  │
  ├── s_umount (超级块锁, 防止卸载)
  │
  └── i_data_sem (写锁, 保护 extent 树修改)
       │
       ├── s_mb_sem / mballoc 锁 (块分配器)
       │
       └── i_es_lock (写锁, 修改 ES Tree)
```

### 并发场景

| 场景 | 锁 | 说明 |
|------|-----|------|
| 多进程写同一文件 | i_rwsem 互斥 | 串行化写 |
| 写 + 读 | i_rwsem 互斥 | 写者排他 |
| 写 + writeback | i_data_sem 互斥 | writeback 需要读锁 |
| 并发 DIO | i_rwsem 共享 (对齐) | 对齐 DIO 可并发 |
| mballoc 并发 | s_mb_lock (per-group) | 组级锁, 高并发 |

### Writeback 并发保护

```c
/* ext4_writepages 使用 mballoc 锁保护 */
alloc_ctx = ext4_writepages_down_read(sb);   /* 获取 mballoc 读锁 */
...
ext4_writepages_up_read(sb, alloc_ctx);      /* 释放 */

/* 防止 mballoc 在 writeback 期间改变分配策略 */
```

---

## 六、优化点 (2014-2026)

| 年份 | 优化 | 效果 |
|------|------|------|
| 2014 | Delalloc 改进 | 减少元数据更新, 提升随机写 ~40% |
| 2015 | Multi-block allocation (mballoc) | 批量分配, 减少碎片 |
| 2016 | Unwritten extent 转换优化 | 减少 journal 事务开销 |
| 2018 | ext4_io_end_vec 向量化 | 支持多个不连续范围 |
| 2019 | ES Tree 缓存写入路径 | 减少 extent 树查找 |
| 2020 | Inline data 写优化 | 小文件零磁盘分配 |
| 2021 | Fast commit 写路径集成 | fsync 延迟降低 ~10x |
| 2023 | iomap buffered write 迁移开始 | 统一 VFS 写接口 |
| 2024 | Large folio 写支持 | 减少 page cache 开销 |
| 2025 | Deferred extent splitting | 减少 journal handle 开销, DIO 性能 +25% |
| 2025 | ES Tree seq counter | 并发 I/O 安全 |

### Delalloc 核心优势

```
传统分配:
  write() → 分配块 → 写数据 → 更新元数据 (每个写操作)

延迟分配:
  write() → 标记 DELAYED → 写数据到 page cache (无磁盘分配)
  writeback → 批量分配连续块 → 写数据到磁盘 → 更新元数据 (一次)

优势:
  - 减少元数据更新次数
  - 提高块分配连续性 (减少碎片)
  - 合并相邻写操作
```

### Unwritten Extent 转换优化

```
旧方式 (同步转换):
  分配 → 转换 unwritten→written (journal) → I/O → 完成

新方式 (异步转换):
  分配 → I/O (无 journal) → 完成回调 → 转换 unwritten→written (journal)

优势:
  - I/O 和元数据更新解耦
  - 减少 journal handle 持有时间
  - 提升并发 DIO 性能 ~25%
```

---

## 七、常见陷阱

### 1. Delalloc 预留块泄漏

```
当写操作失败时:
  - 已预留的 DELAYED 块必须释放
  - ext4_da_write_begin 失败路径调用 ext4_truncate_failed_write()
  - i_reserved_data_blocks 计数器必须正确更新
```

### 2. Unwritten Extent 转换失败

```
如果 unwritten→written 转换失败:
  - 数据已写入磁盘但映射仍是 unwritten
  - 读操作会看到全零 (unwritten extent 返回零)
  - 需要重试转换或标记文件系统错误
```

### 3. ENOSPC 处理

```
空间不足时的重试机制:
  - ext4_should_retry_alloc() 检查是否有可回收空间
  - 触发 discard/journal commit 释放空间
  - 最多重试次数有限制 (防止死循环)
```

### 4. i_size 与 i_disksize 不一致

```
i_size: 文件逻辑大小 (用户可见)
i_disksize: 已持久化到磁盘的大小

写路径:
  - 写操作更新 i_size
  - writeback 完成后更新 i_disksize
  - 崩溃时 i_disksize 到 i_size 之间的数据可能丢失

data=ordered 模式保证:
  - 数据先刷盘, 然后元数据提交
  - 崩溃后不会出现陈旧数据
```

### 5. Large Folio 写对齐

```
Large folio 写需要处理部分更新:
  - folio 可能覆盖多个逻辑块
  - 只更新 folio 中的部分块
  - ext4_block_write_begin() 处理块级映射
  - 未映射块保持 DELAYED 状态
```

### 6. Inline Data 转换

```
小文件使用 inline data (数据存储在 inode 内):
  - 当文件增长超过 inline 容量时
  - 需要转换为 extent 树存储
  - 转换过程需要 journal 事务
  - 转换失败可能导致数据丢失
```

---

## 八、关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| ext4_file_write_iter | fs/ext4/file.c | 690-720 |
| ext4_buffered_write_iter | fs/ext4/file.c | 286-307 |
| ext4_da_write_begin | fs/ext4/inode.c | 3105-3167 |
| ext4_writepages | fs/ext4/inode.c | 3002-3100 |
| ext4_da_map_blocks | fs/ext4/inode.c | ~1989 |
| ext4_map_blocks | fs/ext4/inode.c | ~1900 |
| ext4_mb_new_blocks | fs/ext4/mballoc.c | ~4500 |
| ext4_convert_unwritten_io_end_vec | fs/ext4/inode.c | ~2800 |
| ext4_ext_truncate | fs/ext4/extents.c | ~4200 |
| ext4_iomap_begin | fs/ext4/inode.c | ~3500 |
| mpage_map_and_submit_buffers | fs/ext4/page-io.c | ~300 |
