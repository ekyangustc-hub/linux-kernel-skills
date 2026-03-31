---
name: "ext4-group-descriptor"
description: "ext4文件系统块组描述符专家。当用户询问ext4块组描述符、bg_block_bitmap、bg_inode_bitmap、bg_inode_table、块组管理时调用此技能。"
---

# ext4 Group Descriptor (块组描述符)

## 一、概述

块组描述符 (Group Descriptor, GDT) 存储每个块组的元数据位置和统计信息。

---

## 二、块组描述符位置

### 2.1 主描述符位置

主块组描述符位于超级块之后：

```
块组 0:
┌─────────────────────────────────────────────────────────────┐
│  Super Block (1024 字节)                                    │
├─────────────────────────────────────────────────────────────┤
│  Group Descriptors (多个描述符)                              │
│  - GDT[0]: 块组 0 描述符                                     │
│  - GDT[1]: 块组 1 描述符                                     │
│  - ...                                                      │
├─────────────────────────────────────────────────────────────┤
│  Reserved GDT Blocks (预留扩展空间)                          │
├─────────────────────────────────────────────────────────────┤
│  Block Bitmap                                               │
├─────────────────────────────────────────────────────────────┤
│  Inode Bitmap                                               │
├─────────────────────────────────────────────────────────────┤
│  Inode Table                                                │
├─────────────────────────────────────────────────────────────┤
│  Data Blocks                                                │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 备份描述符位置

取决于 sparse_super 特性：

```
默认备份位置：块组 1, 3, 5, 7, 9, 25, 27, 29, 31, ...
```

---

## 三、结构定义

**位置**: `fs/ext4/ext4.h`

```c
struct ext4_group_desc {
    __le32  bg_block_bitmap_lo;     /* 块位图块号 (低 32 位) */
    __le32  bg_inode_bitmap_lo;     /* inode 位图块号 (低 32 位) */
    __le32  bg_inode_table_lo;      /* inode 表块号 (低 32 位) */
    __le16  bg_free_blocks_count_lo;/* 空闲块数 (低 16 位) */
    __le16  bg_free_inodes_count_lo;/* 空闲 inode 数 (低 16 位) */
    __le16  bg_used_dirs_count_lo;  /* 已用目录数 (低 16 位) */
    __le16  bg_flags;               /* 标志 */
    __le32  bg_exclude_bitmap_lo;   /* 排除位图 (低 32 位) */
    __le16  bg_block_bitmap_csum_lo;/* 块位图校验和 (低 16 位) */
    __le16  bg_inode_bitmap_csum_lo;/* inode 位图校验和 (低 16 位) */
    __le16  bg_itable_unused_lo;    /* 未用 inode 数 (低 16 位) */
    __le16  bg_checksum;            /* 校验和 */
    __le32  bg_block_bitmap_hi;     /* 块位图块号 (高 32 位) */
    __le32  bg_inode_bitmap_hi;     /* inode 位图块号 (高 32 位) */
    __le32  bg_inode_table_hi;      /* inode 表块号 (高 32 位) */
    __le16  bg_free_blocks_count_hi;/* 空闲块数 (高 16 位) */
    __le16  bg_free_inodes_count_hi;/* 空闲 inode 数 (高 16 位) */
    __le16  bg_used_dirs_count_hi;  /* 已用目录数 (高 16 位) */
    __le16  bg_itable_unused_hi;    /* 未用 inode 数 (高 16 位) */
    __le32  bg_exclude_bitmap_hi;   /* 排除位图 (高 32 位) */
    __le16  bg_block_bitmap_csum_hi;/* 块位图校验和 (高 16 位) */
    __le16  bg_inode_bitmap_csum_hi;/* inode 位图校验和 (高 16 位) */
    __u32   bg_reserved;            /* 保留 */
};
```

---

## 四、关键字段详解

### 4.1 元数据位置

| 字段 | 说明 |
|------|------|
| `bg_block_bitmap` | 块位图的起始块号 |
| `bg_inode_bitmap` | inode 位图的起始块号 |
| `bg_inode_table` | inode 表的起始块号 |

### 4.2 统计信息

| 字段 | 说明 |
|------|------|
| `bg_free_blocks_count` | 块组中空闲块数 |
| `bg_free_inodes_count` | 块组中空闲 inode 数 |
| `bg_used_dirs_count` | 块组中已用目录数 |
| `bg_itable_unused` | inode 表中未使用的 inode 数 |

### 4.3 标志

```c
#define EXT4_BG_INODE_UNINIT    0x0001  /* inode 表未初始化 */
#define EXT4_BG_BLOCK_UNINIT    0x0002  /* 块位图未初始化 */
#define EXT4_BG_INODE_ZEROED    0x0004  /* inode 表已清零 */
```

---

## 五、块组布局

### 5.1 标准布局

```
块组 N:
┌─────────────────────────────────────────────────────────────┐
│  块位图 (1 块)                                               │
│  - 每位表示一个块的使用状态                                   │
│  - 0 = 空闲, 1 = 已用                                        │
├─────────────────────────────────────────────────────────────┤
│  inode 位图 (1 块)                                           │
│  - 每位表示一个 inode 的使用状态                              │
│  - 0 = 空闲, 1 = 已用                                        │
├─────────────────────────────────────────────────────────────┤
│  inode 表 (多块)                                             │
│  - 存储所有 inode                                            │
│  - 块数 = s_inodes_per_group * s_inode_size / block_size     │
├─────────────────────────────────────────────────────────────┤
│  数据块                                                      │
│  - 存储文件数据                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 flex_bg 布局

flex_bg 特性将多个块组合并：

```
flex 组 (4 个块组):
┌─────────────────────────────────────────────────────────────┐
│  块组 0: Super Block + GDT                                   │
├─────────────────────────────────────────────────────────────┤
│  所有块位图 (连续存放)                                        │
│  - bg 0 块位图                                               │
│  - bg 1 块位图                                               │
│  - bg 2 块位图                                               │
│  - bg 3 块位图                                               │
├─────────────────────────────────────────────────────────────┤
│  所有 inode 位图 (连续存放)                                  │
│  - bg 0 inode 位图                                           │
│  - bg 1 inode 位图                                           │
│  - bg 2 inode 位图                                           │
│  - bg 3 inode 位图                                           │
├─────────────────────────────────────────────────────────────┤
│  所有 inode 表 (连续存放)                                    │
│  - bg 0 inode 表                                             │
│  - bg 1 inode 表                                             │
│  - bg 2 inode 表                                             │
│  - bg 3 inode 表                                             │
├─────────────────────────────────────────────────────────────┤
│  数据块                                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 六、块位图

### 6.1 概述

块位图用于跟踪块组中每个块的使用状态。

### 6.2 位图大小

```c
/* 块位图大小 */
bitmap_size = sb->s_blocks_per_group / 8;

/* 如果块大小为 4096，每组 32768 块 */
bitmap_size = 32768 / 8 = 4096 字节 = 1 块
```

### 6.3 位图操作

```c
/* 检查块是否空闲 */
int ext4_test_block_bitmap(struct ext4_group_info *grp, ext4_fsblk_t block)
{
    return test_bit(block, grp->bb_bitmap);
}

/* 设置块为已用 */
void ext4_set_block_bitmap(struct ext4_group_info *grp, ext4_fsblk_t block)
{
    set_bit(block, grp->bb_bitmap);
}

/* 清除块（标记为空闲） */
void ext4_clear_block_bitmap(struct ext4_group_info *grp, ext4_fsblk_t block)
{
    clear_bit(block, grp->bb_bitmap);
}
```

---

## 七、Inode 位图

### 7.1 概述

inode 位图用于跟踪块组中每个 inode 的使用状态。

### 7.2 位图大小

```c
/* inode 位图大小 */
bitmap_size = sb->s_inodes_per_group / 8;

/* 如果每组 8192 个 inode */
bitmap_size = 8192 / 8 = 1024 字节
```

---

## 八、调试命令

### 8.1 dumpe2fs

```bash
# 查看块组信息
dumpe2fs /dev/sdX | grep -A 10 "Group 0"

# 输出示例
Group 0: (Blocks 0-32767)
  Primary superblock at 0, Group descriptors at 1-1
  Reserved GDT blocks at 2-1025
  Block bitmap at 1026 (+1026)
  Inode bitmap at 1042 (+1042)
  Inode table at 1058-1569 (+1058)
  28673 free blocks, 8181 free inodes, 2 directories
  Free blocks: 4096-32767
  Free inodes: 12-8192
```

### 8.2 debugfs

```bash
debugfs /dev/sdX

# 查看块组描述符
debugfs: stats

# 查看块位图
debugfs: testb <block>

# 查看 inode 位图
debugfs: testi <inode>
```

---

## 九、代码位置

| 功能 | 文件路径 |
|------|----------|
| 结构定义 | `fs/ext4/ext4.h` |
| 块组操作 | `fs/ext4/balloc.c` |
| inode 分配 | `fs/ext4/ialloc.c` |

---

## 十、参考资源

- ext4 官方文档: `Documentation/filesystems/ext4/`
- ext4 内核代码: `fs/ext4/ext4.h`
