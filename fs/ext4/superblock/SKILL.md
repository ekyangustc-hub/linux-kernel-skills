---
name: "ext4-superblock"
description: "ext4文件系统超级块专家。当用户询问ext4超级块结构、s_magic、s_log_block_size、s_feature_incompat、文件系统参数时调用此技能。"
---

# ext4 Super Block (超级块)

## 一、概述

超级块是 ext4 文件系统的核心结构，存储文件系统的元数据和配置信息。

---

## 二、超级块位置

### 2.1 主超级块位置

| 块大小 | 超级块起始字节 |
|--------|---------------|
| 1024 字节 | 字节 1024 (块 1) |
| 2048 字节 | 字节 1024 (块 0 的后半部分) |
| 4096 字节 | 字节 1024 (块 0 的后半部分) |

### 2.2 备份超级块

备份超级块位于特定的块组中，取决于 sparse_super 特性：

```
默认备份位置：块组 1, 3, 5, 7, 9, 25, 27, 29, 31, ...
```

---

## 三、结构定义

**位置**: `fs/ext4/ext4.h`

```c
struct ext4_super_block {
    __le32  s_inodes_count;         /* inode 总数 */
    __le32  s_blocks_count_lo;      /* 块总数 (低 32 位) */
    __le32  s_r_blocks_count_lo;    /* 保留块数 (低 32 位) */
    __le32  s_free_blocks_count_lo; /* 空闲块数 (低 32 位) */
    __le32  s_free_inodes_count;    /* 空闲 inode 数 */
    __le32  s_first_data_block;     /* 第一个数据块号 */
    __le32  s_log_block_size;       /* 块大小 = 1024 << s_log_block_size */
    __le32  s_log_cluster_size;     /* 簇大小 */
    __le32  s_blocks_per_group;     /* 每组块数 */
    __le32  s_clusters_per_group;   /* 每组簇数 */
    __le32  s_inodes_per_group;     /* 每组 inode 数 */
    __le32  s_mtime;                /* 最后挂载时间 */
    __le32  s_wtime;                /* 最后写入时间 */
    __le16  s_mnt_count;            /* 挂载计数 */
    __le16  s_max_mnt_count;        /* 最大挂载计数 */
    __le16  s_magic;                /* 魔数 0xEF53 */
    __le16  s_state;                /* 文件系统状态 */
    __le16  s_errors;               /* 错误处理方式 */
    __le16  s_minor_rev_level;      /* 次版本号 */
    __le32  s_lastcheck;            /* 最后检查时间 */
    __le32  s_checkinterval;        /* 检查间隔 */
    __le32  s_creator_os;           /* 创建操作系统 */
    __le32  s_rev_level;            /* 版本号 */
    __le16  s_def_resuid;           /* 默认保留 UID */
    __le16  s_def_resgid;           /* 默认保留 GID */
    __le32  s_first_ino;            /* 第一个非保留 inode */
    __le16  s_inode_size;           /* inode 大小 */
    __le16  s_block_group_nr;       /* 块组号 */
    __le32  s_feature_compat;       /* 兼容特性 */
    __le32  s_feature_incompat;     /* 不兼容特性 */
    __le32  s_feature_ro_compat;    /* 只读兼容特性 */
    __u8    s_uuid[16];             /* UUID */
    char    s_volume_name[16];      /* 卷名 */
    char    s_last_mounted[64];     /* 最后挂载点 */
    __le32  s_algorithm_usage_bitmap;/* 算法位图 */
    __u8    s_prealloc_blocks;      /* 预分配块数 */
    __u8    s_prealloc_dir_blocks;  /* 目录预分配块数 */
    __le16  s_reserved_gdt_blocks;  /* 保留 GDT 块数 */
    __u8    s_journal_uuid[16];     /* 日志 UUID */
    __le32  s_journal_inum;         /* 日志 inode */
    __le32  s_journal_dev;          /* 日志设备 */
    __le32  s_last_orphan;          /* 孤儿 inode 链表头 */
    __le32  s_hash_seed[4];         /* HTREE 哈希种子 */
    __u8    s_def_hash_version;     /* 默认哈希版本 */
    __u8    s_jnl_backup_type;      /* 日志备份类型 */
    __le16  s_desc_size;            /* 块组描述符大小 */
    __le32  s_default_mount_opts;   /* 默认挂载选项 */
    __le32  s_first_meta_bg;        /* 第一个 meta_bg */
    __le32  s_mkfs_time;            /* 创建时间 */
    __le32  s_jnl_blocks[17];       /* 日志备份 */
    __le32  s_blocks_count_hi;      /* 块总数 (高 32 位) */
    __le32  s_r_blocks_count_hi;    /* 保留块数 (高 32 位) */
    __le32  s_free_blocks_count_hi; /* 空闲块数 (高 32 位) */
    __le16  s_min_extra_isize;      /* 最小额外 inode 空间 */
    __le16  s_want_extra_isize;     /* 期望额外 inode 空间 */
    __le32  s_flags;                /* 标志 */
    __le16  s_raid_stride;          /* RAID 步长 */
    __le16  s_mmp_interval;         /* MMP 检查间隔 */
    __le64  s_mmp_block;            /* MMP 块号 */
    __le32  s_raid_stripe_width;    /* RAID 条带宽度 */
    __u8    s_log_groups_per_flex;  /* flex_bg 大小 */
    __u8    s_checksum_type;        /* 校验和类型 */
    __le16  s_reserved_pad;         /* 保留 */
    __le64  s_kbytes_written;       /* 写入字节数 */
    __le32  s_snapshot_inum;        /* 快照 inode */
    __le32  s_snapshot_id;          /* 快照 ID */
    __le64  s_snapshot_r_blocks_count;/* 快照保留块 */
    __le32  s_snapshot_list;        /* 快照链表 */
    __le32  s_error_count;          /* 错误计数 */
    __le32  s_first_error_time;     /* 第一次错误时间 */
    __le32  s_first_error_ino;      /* 第一次错误 inode */
    __le64  s_first_error_block;    /* 第一次错误块 */
    __u8    s_first_error_func[32]; /* 第一次错误函数 */
    __le32  s_first_error_line;     /* 第一次错误行号 */
    __le32  s_last_error_time;      /* 最后错误时间 */
    __le32  s_last_error_ino;       /* 最后错误 inode */
    __le32  s_last_error_line;      /* 最后错误行号 */
    __le64  s_last_error_block;     /* 最后错误块 */
    __u8    s_last_error_func[32];  /* 最后错误函数 */
    __u8    s_mount_opts[64];       /* 挂载选项 */
    __le32  s_usr_quota_inum;       /* 用户配额 inode */
    __le32  s_grp_quota_inum;       /* 组配额 inode */
    __le32  s_overhead_blocks;      /* 开销块数 */
    __le32  s_backup_bgs[2];        /* 备份块组 */
    __u8    s_encrypt_algos[4];     /* 加密算法 */
    __u8    s_encrypt_pw_salt[16];  /* 加密盐 */
    __le32  s_lpf_ino;              /* lost+found inode */
    __le32  s_prj_quota_inum;       /* 项目配额 inode */
    __le32  s_checksum_seed;        /* 校验和种子 */
    __le32  s_reserved[98];         /* 保留 */
    __le32  s_checksum;             /* 校验和 */
};
```

---

## 四、关键字段详解

### 4.1 基本参数

| 字段 | 说明 |
|------|------|
| `s_magic` | 魔数，固定为 0xEF53 |
| `s_log_block_size` | 块大小 = 1024 << s_log_block_size |
| `s_blocks_per_group` | 每个块组的块数 |
| `s_inodes_per_group` | 每个块组的 inode 数 |
| `s_inode_size` | inode 大小，通常 256 字节 |

### 4.2 块大小计算

```c
/* 块大小计算 */
block_size = 1024 << sb->s_log_block_size;

/* 常见值 */
s_log_block_size = 0  -> 1024 字节
s_log_block_size = 1  -> 2048 字节
s_log_block_size = 2  -> 4096 字节
```

### 4.3 文件系统状态

```c
#define EXT4_VALID_FS           0x0001  /* 文件系统干净卸载 */
#define EXT4_ERROR_FS           0x0002  /* 文件系统有错误 */
#define EXT4_ORPHAN_FS          0x0004  /* 文件系统有孤儿 inode */
```

---

## 五、特性标志

### 5.1 兼容特性 (s_feature_compat)

```c
#define EXT4_FEATURE_COMPAT_DIR_PREALLOC    0x0001  /* 目录预分配 */
#define EXT4_FEATURE_COMPAT_IMAGIC_INODES   0x0002  /* imagic inode */
#define EXT4_FEATURE_COMPAT_HAS_JOURNAL     0x0004  /* 有日志 */
#define EXT4_FEATURE_COMPAT_EXT_ATTR        0x0008  /* 扩展属性 */
#define EXT4_FEATURE_COMPAT_RESIZE_INO      0x0010  /* resize inode */
#define EXT4_FEATURE_COMPAT_DIR_INDEX       0x0020  /* 目录索引 */
```

### 5.2 不兼容特性 (s_feature_incompat)

```c
#define EXT4_FEATURE_INCOMPAT_COMPRESSION   0x0001  /* 压缩 */
#define EXT4_FEATURE_INCOMPAT_FILETYPE      0x0002  /* 目录项有文件类型 */
#define EXT4_FEATURE_INCOMPAT_RECOVER       0x0004  /* 需要恢复 */
#define EXT4_FEATURE_INCOMPAT_JOURNAL_DEV   0x0008  /* 外部日志 */
#define EXT4_FEATURE_INCOMPAT_META_BG       0x0010  /* meta_bg */
#define EXT4_FEATURE_INCOMPAT_EXTENTS       0x0040  /* extent 树 */
#define EXT4_FEATURE_INCOMPAT_64BIT         0x0080  /* 64 位 */
#define EXT4_FEATURE_INCOMPAT_MMP           0x0100  /* MMP */
#define EXT4_FEATURE_INCOMPAT_FLEX_BG       0x0200  /* flex_bg */
#define EXT4_FEATURE_INCOMPAT_EA_INODE      0x0400  /* EA inode */
#define EXT4_FEATURE_INCOMPAT_DIRDATA       0x1000  /* 目录数据 */
#define EXT4_FEATURE_INCOMPAT_CSUM_SEED     0x2000  /* 校验和种子 */
#define EXT4_FEATURE_INCOMPAT_LARGEDIR      0x4000  /* 大目录 */
#define EXT4_FEATURE_INCOMPAT_INLINE_DATA   0x8000  /* 内联数据 */
#define EXT4_FEATURE_INCOMPAT_ENCRYPT       0x10000 /* 加密 */
```

### 5.3 只读兼容特性 (s_feature_ro_compat)

```c
#define EXT4_FEATURE_RO_COMPAT_SPARSE_SUPER 0x0001  /* 稀疏超级块 */
#define EXT4_FEATURE_RO_COMPAT_LARGE_FILE   0x0002  /* 大文件 */
#define EXT4_FEATURE_RO_COMPAT_BTREE_DIR    0x0004  /* B-tree 目录 */
#define EXT4_FEATURE_RO_COMPAT_HUGE_FILE    0x0008  /* 巨大文件 */
#define EXT4_FEATURE_RO_COMPAT_GDT_CSUM     0x0010  /* GDT 校验和 */
#define EXT4_FEATURE_RO_COMPAT_DIR_NLINK    0x0020  /* 目录无限链接 */
#define EXT4_FEATURE_RO_COMPAT_EXTRA_ISIZE  0x0040  /* 额外 inode 空间 */
#define EXT4_FEATURE_RO_COMPAT_QUOTA        0x0100  /* 配额 */
#define EXT4_FEATURE_RO_COMPAT_BIGALLOC     0x0200  /* bigalloc */
#define EXT4_FEATURE_RO_COMPAT_METADATA_CSUM 0x0400 /* 元数据校验和 */
```

---

## 六、mkfs 选项

### 6.1 常用选项

```bash
# 基本格式化
mkfs.ext4 /dev/sdX

# 指定块大小
mkfs.ext4 -b 4096 /dev/sdX

# 指定 inode 大小
mkfs.ext4 -I 256 /dev/sdX

# 启用特性
mkfs.ext4 -O extent,flex_bg,metadata_csum /dev/sdX

# 禁用日志
mkfs.ext4 -O ^has_journal /dev/sdX

# 指定块组大小
mkfs.ext4 -g 32768 /dev/sdX

# 完整示例
mkfs.ext4 -b 4096 -I 256 -O extent,flex_bg,metadata_csum,64bit /dev/sdX
```

### 6.2 mkfs 输出解读

```
mke2fs 1.45.5 (07-Jan-2020)
Creating filesystem with 2621440 4k blocks and 655360 inodes
Filesystem UUID: 12345678-1234-1234-1234-123456789abc
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

---

## 七、调试命令

### 7.1 dumpe2fs

```bash
# 查看超级块
dumpe2fs /dev/sdX | head -100

# 输出示例
Filesystem volume name:   <none>
Last mounted on:          /mnt
Filesystem UUID:          12345678-1234-1234-1234-123456789abc
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal, extent, flex_bg, metadata_csum
Filesystem flags:         signed_directory_hash
Default mount options:    user_xattr acl
Filesystem state:         clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              655360
Block count:              2621440
Reserved block count:     131072
Free blocks:              2490368
Free inodes:              655349
First block:              0
Block size:               4096
Fragment size:            4096
Group descriptor size:    64
Reserved GDT blocks:      1024
Blocks per group:         32768
Inodes per group:         8192
Inode blocks per group:   512
Flex block group size:    16
```

### 7.2 debugfs

```bash
debugfs /dev/sdX

# 查看超级块
debugfs: stats

# 查看特性
debugfs: features
```

---

## 八、代码位置

| 功能 | 文件路径 |
|------|----------|
| 结构定义 | `fs/ext4/ext4.h` |
| 超级块操作 | `fs/ext4/super.c` |
| 超级块 I/O | `fs/ext4/super.c` |

---

## 九、参考资源

- ext4 官方文档: `Documentation/filesystems/ext4/`
- ext4 内核代码: `fs/ext4/ext4.h`
