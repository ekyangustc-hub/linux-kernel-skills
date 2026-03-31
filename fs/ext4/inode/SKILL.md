---
name: "ext4-inode"
description: "ext4文件系统Inode结构专家。当用户询问ext4 inode结构、i_mode、i_block、extent树、inode分配、fsverity读取路径时调用此技能。"
---

# ext4 Inode 结构

## 一、概述

Inode 是 ext4 文件系统的核心数据结构，存储文件/目录的元数据和数据位置信息。

---

## 二、Inode 结构定义

**位置**: `fs/ext4/ext4.h`

```c
struct ext4_inode {
    __le16  i_mode;                 /* 文件类型和权限 */
    __le16  i_uid;                  /* 用户 ID (低 16 位) */
    __le32  i_size_lo;              /* 文件大小 (低 32 位) */
    __le32  i_atime;                /* 访问时间 */
    __le32  i_ctime;                /* 创建时间 */
    __le32  i_mtime;                /* 修改时间 */
    __le32  i_dtime;                /* 删除时间 */
    __le16  i_gid;                  /* 组 ID (低 16 位) */
    __le16  i_links_count;          /* 链接计数 */
    __le32  i_blocks_lo;            /* 块数 (低 32 位) */
    __le32  i_flags;                /* 标志 */
    union {
        struct {
            __le32  l_i_version;    /* 版本号 */
        } linux1;
        struct {
            __le32  h_i_translator; /* 翻译器 */
        } hurd1;
    } osd1;
    __le32  i_block[EXT4_N_BLOCKS];/* 块指针或 extent 树 */
    __le32  i_generation;           /* 代号 */
    __le32  i_file_acl_lo;          /* ACL 块号 (低 32 位) */
    __le32  i_size_hi;              /* 文件大小 (高 32 位) */
    __le32  i_obso_faddr;           /* 废弃字段 */
    union {
        struct {
            __le16  l_i_blocks_high;/* 块数 (高 16 位) */
            __le16  l_i_file_acl_high;/* ACL 块号 (高 16 位) */
            __le16  l_i_uid_high;   /* 用户 ID (高 16 位) */
            __le16  l_i_gid_high;   /* 组 ID (高 16 位) */
            __le16  l_i_checksum_lo;/* 校验和 (低 16 位) */
            __le16  l_i_reserved;   /* 保留 */
        } linux2;
    } osd2;
    __le16  i_extra_isize;          /* 额外 inode 大小 */
    __le16  i_checksum_hi;          /* 校验和 (高 16 位) */
    __le32  i_ctime_extra;          /* 创建时间 (额外) */
    __le32  i_mtime_extra;          /* 修改时间 (额外) */
    __le32  i_atime_extra;          /* 访问时间 (额外) */
    __le32  i_crtime;               /* 创建时间 */
    __le32  i_crtime_extra;         /* 创建时间 (额外) */
    __le32  i_version_hi;           /* 版本号 (高 32 位) */
    __le32  i_projid;               /* 项目 ID */
};
```

---

## 三、关键字段详解

### 3.1 文件类型和权限 (i_mode)

```c
/* 文件类型 (高 4 位) */
#define S_IFIFO   0010000   /* FIFO */
#define S_IFCHR   0020000   /* 字符设备 */
#define S_IFDIR   0040000   /* 目录 */
#define S_IFBLK   0060000   /* 块设备 */
#define S_IFREG   0100000   /* 普通文件 */
#define S_IFLNK   0120000   /* 符号链接 */
#define S_IFSOCK  0140000   /* Socket */

/* 权限位 (低 12 位) */
#define S_ISUID   0004000   /* Set-UID */
#define S_ISGID   0002000   /* Set-GID */
#define S_ISVTX   0001000   /* Sticky */
```

### 3.2 i_block 布局

```c
#define EXT4_N_BLOCKS      15
#define EXT4_IND_BLOCK     12   /* 一级间接块 */
#define EXT4_DIND_BLOCK    13   /* 二级间接块 */
#define XFS_TIND_BLOCK    14   /* 三级间接块 */
```

### 3.3 Inode 标志 (i_flags)

```c
#define EXT4_SECRM_FL              0x00000001 /* 安全删除 */
#define EXT4_UNRM_FL               0x00000002 /* 可恢复删除 */
#define EXT4_COMPR_FL              0x00000004 /* 压缩文件 */
#define EXT4_SYNC_FL               0x00000008 /* 同步更新 */
#define EXT4_IMMUTABLE_FL          0x00000010 /* 不可变 */
#define EXT4_APPEND_FL             0x00000020 /* 仅追加 */
#define EXT4_NODUMP_FL             0x00000040 /* 不转储 */
#define EXT4_NOATIME_FL            0x00000080 /* 不更新访问时间 */
#define EXT4_INDEX_FL              0x00001000 /* HTREE 索引 */
#define EXT4_IMAGIC_FL             0x00002000 /* imagic */
#define EXT4_JOURNAL_DATA_FL       0x00004000 /* 数据日志 */
#define EXT4_NOTAIL_FL             0x00008000 /* 无尾部合并 */
#define EXT4_DIRSYNC_FL            0x00010000 /* 目录同步 */
#define EXT4_TOPDIR_FL             0x00020000 /* 顶层目录 */
#define EXT4_HUGE_FILE_FL          0x00040000 /* 大文件 */
#define EXT4_EXTENTS_FL            0x00080000 /* 使用 extent */
#define EXT4_EA_INODE_FL           0x00200000 /* EA inode */
#define EXT4_EOFBLOCKS_FL          0x00400000 /* EOF 块 */
#define EXT4_NOCOW_FL              0x00800000 /* 无 CoW */
#define EXT4_INLINE_DATA_FL        0x10000000 /* 内联数据 */
#define EXT4_PROJINHERIT_FL        0x20000000 /* 项目继承 */
#define EXT4_CASEFOLD_FL           0x40000000 /* 大小写不敏感 */
```

---

## 四、数据存储方式

### 4.1 方式一：直接/间接块

```
i_block[0-11]: 直接块指针 (12 个)
i_block[12]:   一级间接块指针
i_block[13]:   二级间接块指针
i_block[14]:   三级间接块指针

最大文件大小计算：
- 直接块: 12 * block_size
- 一级间接: block_size/4 * block_size
- 二级间接: (block_size/4)^2 * block_size
- 三级间接: (block_size/4)^3 * block_size
```

### 4.2 方式二：Extent 树

当设置 EXT4_EXTENTS_FL 标志时，i_block 存储 extent 树：

```c
/* i_block 作为 extent 树头 */
struct ext4_extent_header {
    __le16  eh_magic;       /* 魔数 0xF30A */
    __le16  eh_entries;     /* 条目数 */
    __le16  eh_max;         /* 最大条目数 */
    __le16  eh_depth;       /* 深度 (0=叶子) */
    __le32  eh_generation;  /* 代号 */
};
```

---

## 五、Inode 大小

### 5.1 基本大小

| 版本 | inode 大小 |
|------|-----------|
| ext2/3 | 128 字节 |
| ext4 | 256 字节 (默认) |

### 5.2 额外空间

```c
/* 额外 inode 空间 */
i_extra_isize = inode_size - EXT4_GOOD_OLD_INODE_SIZE;

/* 用于存储：
 * - 创建时间 (i_crtime)
 * - 项目 ID (i_projid)
 * - 扩展属性
 */
```

---

## 六、Inode 号计算

### 6.1 Inode 号结构

```
inode 号 = 块组号 * 每组 inode 数 + 组内偏移
```

### 6.2 计算

```c
/* 计算 inode 所在块组 */
group = (inode - 1) / sb->s_inodes_per_group;

/* 计算 inode 在组内偏移 */
offset = (inode - 1) % sb->s_inodes_per_group;

/* 计算 inode 在 inode 表中的块号 */
block = offset / (block_size / inode_size);

/* 计算 inode 在块内偏移 */
offset_in_block = (offset % (block_size / inode_size)) * inode_size;
```

---

## 七、特殊 Inode

| Inode 号 | 用途 |
|----------|------|
| 1 | 坏块列表 |
| 2 | 根目录 |
| 3 | ACL 索引 |
| 4 | ACL 数据 |
| 5 | 日志 inode |
| 6 | 保留 |
| 7 | 保留 |
| 8 | 日志 (外部日志) |
| 11 | lost+found 目录 |
| 12-13 | 保留 |
| 14-15 | 配额文件 |

---

## 八、调试命令

### 8.1 debugfs

```bash
debugfs /dev/sdX

# 查看 inode 信息
debugfs: stat <inode号>

# 输出示例
Inode: 12   Type: regular    Mode:  0644   Flags: 0x0
Generation: 0    Version: 0x00000000
User:     0   Group:     0   Size: 1024
File ACL: 0    Directory ACL: 0
Links: 1   Blockcount: 2
Fragment:  Address: 0    Number: 0    Size: 0
 ctime: 0x5f8a1b2c -- Mon Oct 19 10:30:36 2020
 atime: 0x5f8a1b2c -- Mon Oct 19 10:30:36 2020
 mtime: 0x5f8a1b2c -- Mon Oct 19 10:30:36 2020
BLOCKS:
(0): 1024
TOTAL: 1
```

### 8.2 查看文件 inode

```bash
ls -i /path/to/file
stat /path/to/file
```

---

## 九、代码位置

| 功能 | 文件路径 |
|------|----------|
| 结构定义 | `fs/ext4/ext4.h` |
| Inode 操作 | `fs/ext4/inode.c` |
| Inode 分配 | `fs/ext4/ialloc.c` |

---

## 十、Inode 缓存优化

### 10.1 空闲指针偏移

`ext4_inode_cache` 使用 `kmem_cache_args` 接口并指定空闲指针偏移在 `i_flags` 字段，减少对象大小。由于构造函数不使用 `i_flags`，可以安全地复用该字段作为空闲指针。

```c
/* fs/ext4/super.c */
struct kmem_cache_args args = {
    .free_pointer_offset = offsetof(struct ext4_inode, i_flags),
};
sbi->s_inode_cache = kmem_cache_create("ext4_inode_cache",
                                       sizeof(struct ext4_inode), &args);
```

---

## 十一、Fsverity 读取路径

### 11.1 读取路径优化

现代 ext4 将 `->read_folio` 和 `->readahead` 操作移至 `readpage.c`，所有 pagecache 读取代码集中在一个文件中。

**核心优化**:
- `fsverity_info` 查找在 `ext4_mpage_readpages` 中只执行一次
- 查找结果用于 readahead、空洞验证和 I/O 完成工作队列
- 通过 `struct bio_post_read_ctx` 传递 `fsverity_info` 到 I/O 完成路径

### 11.2 代码布局

| 功能 | 文件路径 |
|------|----------|
| read_folio / readahead | `fs/ext4/readpage.c` |
| fsverity_info 查找 | `fs/ext4/readpage.c` |
| I/O 完成上下文 | `fs/ext4/readpage.c` |

---

## 十二、参考资源

- ext4 官方文档: `Documentation/filesystems/ext4/`
- ext4 内核代码: `fs/ext4/ext4.h`, `fs/ext4/readpage.c`
