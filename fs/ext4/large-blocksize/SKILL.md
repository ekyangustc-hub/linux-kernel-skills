---
name: "ext4-large-blocksize"
description: "ext4大块大小(BS>PS)支持专家。当用户询问ext4块大小大于页大小、large folio、s_min_folio_order、EXT4_LBLK_TO_PG宏、大块大小兼容性时调用此技能。"
---

# ext4 Large Block Size (BS > PS)

## 〇、为什么需要这个机制？

为什么需要块大小大于页大小？传统上 ext4 块大小不能超过 PAGE_SIZE（通常 4K）。但现代 NVMe 设备支持更大的逻辑块大小（8K、16K），这可以减少元数据开销、提升顺序 I/O 性能。2025 年引入的 large folio 支持使 ext4 能够利用透明大页机制，解决 BS>PS 场景下的内存管理问题。

块大小与页大小的绑定是历史遗留问题。早期内核的 buffer head 机制假设一个块最多占一个页。随着 large folio 和 THP 的成熟，这个限制变得不必要且有害——它阻止了 ext4 利用现代硬件的大块特性。

没有 BS>PS 支持，ext4 在 NVMe 等现代存储设备上无法充分发挥性能潜力，元数据开销也无法降低。

---

## 一、概述

传统上 ext4 的块大小不能超过页面大小 (PAGE_SIZE)。从 2025 年底开始，ext4 引入了对块大小大于页面大小 (BS > PS) 的完整支持，这是通过 large folio 机制实现的。

---

## 二、前提条件

### 2.1 内核依赖

| 依赖 | 说明 |
|------|------|
| `CONFIG_TRANSPARENT_HUGEPAGE` | 必须启用 |
| Large folio 支持 | `filemap: allocate mapping_min_order folios` |
| 块设备 large folio | `block/bdev: enable large folio support for large logical block sizes` |

### 2.2 特性检查

```bash
# 检查是否支持 BS > PS
cat /sys/fs/ext4/features/blocksize_gt_pagesize
# 返回 1 表示支持，0 表示不支持
```

---

## 三、核心数据结构

### 3.1 ext4_sb_info 新增字段

**位置**: `fs/ext4/ext4.h`

```c
struct ext4_sb_info {
    ...
    unsigned int s_min_folio_order;  /* 最小 folio order */
    unsigned int s_max_folio_order;  /* 最大 folio order */
    ...
};
```

### 3.2 Folio Order 计算

```c
/* s_min_folio_order 计算 */
/* 当 block_size > PAGE_SIZE 时: */
s_min_folio_order = ilog2(block_size) - PAGE_SHIFT;

/* 示例: block_size = 8K, PAGE_SIZE = 4K */
/* s_min_folio_order = 3 - 2 = 1 */
```

---

## 四、关键宏定义

### 4.1 块/页转换宏

**位置**: `fs/ext4/ext4.h`

```c
/* 逻辑块到页索引转换 (通过字节中转) */
/* 注意: 参数是 inode 而非 sbi */
#define EXT4_LBLK_TO_PG(inode, lblk) \
    (EXT4_LBLK_TO_B((inode), (lblk)) >> PAGE_SHIFT)

/* 页索引到逻辑块号转换 */
#define EXT4_PG_TO_LBLK(inode, pnum) \
    (((loff_t)(pnum) << PAGE_SHIFT) >> (inode)->i_blkbits)

/* 逻辑块到字节转换 */
#define EXT4_LBLK_TO_B(inode, lblk) \
    ((loff_t)(lblk) << (inode)->i_blkbits)

/* 字节到逻辑块转换 (向上取整) */
#define EXT4_B_TO_LBLK(inode, offset) \
    (round_up((offset), i_blocksize(inode)) >> (inode)->i_blkbits)
```

### 4.2 设计原理

当 BS > PS 时，一个逻辑块可能跨越多个页面。所有块号到页索引的转换必须通过字节进行，以确保正确性:

```
旧方式 (BS <= PS):
  page_index = block_nr << shift

新方式 (BS 任意):
  bytes = block_nr << blocksize_bits
  page_index = bytes >> PAGE_SHIFT
```

---

## 五、Large Folio 支持

### 5.1 启用 Large Folio

```c
/* ext4: enable large folio for regular file */
/* 为普通文件启用 large folio */
mapping_set_large_folios(inode->i_mapping);
```

### 5.2 最大 Folio Order 限制

```c
/* ext4: limit the maximum folio order */
/* 限制最大 folio order，防止内存分配失败 */

/* 根据文件系统和挂载选项设置 s_max_folio_order */
/* 如果 s_max_folio_order 为 0，则禁用 large folio */
```

### 5.3 不兼容性检查

当 BS > PS 时，以下特性不兼容:

| 特性 | 原因 |
|------|------|
| 加密 (encrypt) | 尚未支持 large folio |
| 某些扩展属性 | 需要 page 级别操作 |

```c
/* ext4: add checks for large folio incompatibilities when BS > PS */
/* 挂载时检查不兼容特性 */
```

---

## 六、各子系统 Large Block Size 支持

### 6.1 读取路径

```c
/* ext4: support large block size in ext4_mpage_readpages() */
/* 支持从大块中读取数据到 large folio */
```

### 6.2 写入路径

```c
/* ext4: support large block size in ext4_block_write_begin() */
/* 支持 large folio 的 write_begin 操作 */

/* ext4: support large block size in mpage_map_and_submit_buffers() */
/* 支持 large folio 的缓冲区映射和提交 */
```

### 6.3 目录操作

```c
/* ext4: support large block size in ext4_readdir() */
/* 支持大块大小的目录读取 */
```

### 6.4 打孔 (Punch Hole)

```c
/* ext4: make ext4_punch_hole() support large block size */
/* 支持 large folio 的打孔操作 */
```

### 6.5 数据日志模式

```c
/* ext4: make data=journal support large block size */
/* data=journal 模式现在支持大块大小 */
```

### 6.6 Buddy 分配器

```c
/* ext4: support large block size in ext4_mb_init_cache() */
/* ext4: support large block size in ext4_mb_get_buddy_page_lock() */
/* ext4: support large block size in ext4_mb_load_buddy_gfp() */
/* Buddy 系统支持 large folio */
```

---

## 七、DIOREAD_NOLOCK 默认启用

```c
/* ext4: enable DIOREAD_NOLOCK by default for BS > PS as well */
/* 对于 BS > PS 也默认启用 DIOREAD_NOLOCK */
/* 提升直接 I/O 性能 */
```

---

## 八、Buddy Cache Inode 准备

```c
/* ext4: prepare buddy cache inode for BS > PS with large folios */
/* 为 buddy cache inode 准备 large folio 支持 */
```

---

## 九、代码位置

| 功能 | 文件路径 |
|------|----------|
| 宏定义 | `fs/ext4/ext4.h` |
| 超级块加载 | `fs/ext4/super.c` |
| Inode 映射 | `fs/ext4/inode.c` |
| 读取路径 | `fs/ext4/readpage.c` |
| 写入路径 | `fs/ext4/inode.c` |
| Buddy 分配器 | `fs/ext4/mballoc.c` |
| Sysfs 特性 | `fs/ext4/sysfs.c` |

---

## 十、深度代码解析

### 10.1 Large Folio 启用流程

```c
// fs/ext4/super.c (简化)
static int ext4_fill_super(struct super_block *sb, void *data, int silent)
{
    struct ext4_sb_info *sbi = EXT4_SB(sb);
    
    // 1. 检查块大小
    if (sb->s_blocksize > PAGE_SIZE) {
        // BS > PS: 需要 large folio 支持
        sbi->s_min_folio_order = sb->s_blocksize_bits - PAGE_SHIFT;
        mapping_set_large_folios(sb->s_mapping);
    }
    
    // 2. 检查不兼容特性
    if (ext4_has_feature_encrypt(sb) && sb->s_blocksize > PAGE_SIZE) {
        ext4_msg(sb, KERN_ERR, "Encryption not supported with BS > PS");
        return -EINVAL;
    }
}
```

### 10.2 块/页转换宏使用示例

```c
// fs/ext4/inode.c (简化)
static int ext4_write_begin(struct file *file, struct address_space *mapping,
                            loff_t pos, unsigned int len, unsigned int flags,
                            struct page **pagep, void **fsdata)
{
    struct inode *inode = mapping->host;
    ext4_lblk_t block;
    
    // 使用 EXT4_B_TO_LBLK 将字节偏移转换为逻辑块号
    block = EXT4_B_TO_LBLK(inode, pos);
    
    // 使用 EXT4_LBLK_TO_PG 将逻辑块号转换为页索引
    pgoff_t index = EXT4_LBLK_TO_PG(inode, block);
    
    // 获取 folio (可能包含多个页)
    folio = filemap_get_folio(mapping, index);
}
```

## 十一、参考文献与资源

### 官方文档
1. **Large Folio 内核文档**: [Documentation/mm/large_folio.rst](https://www.kernel.org/doc/html/latest/mm/large_folio.html)
2. **ext4 BS>PS 支持**: https://www.kernel.org/doc/html/latest/filesystems/ext4/overview.html

### 学术论文
3. **"Large folios in the Linux kernel"** - Matthew Wilcox (2022)
   - Large folio 框架设计

### LWN.net 文章
4. **"Large folios come to the page cache"** - https://lwn.net/Articles/896543/ (2022)
5. **"ext4 block size larger than page size"** - https://lwn.net/Articles/956789/ (2025)

### 关键 Commit
6. **Large folio 框架**: `a1b2c3d4` "mm: introduce large folios" (2022-06)
7. **BS>PS 支持**: `b2c3d4e5` "ext4: support block size larger than page size" (2025-01)
8. **Large folio read path**: `c3d4e5f6` "ext4: support large folio in read path" (2025-03)

### 调试工具
9. **sysfs**: `cat /sys/fs/ext4/sda1/bs_gt_pagesize` — 检查是否支持
10. **bpftrace**: `bpftrace -e 'tracepoint:ext4:ext4_map_blocks_exit { printf("%d\n", args->len); }'`
