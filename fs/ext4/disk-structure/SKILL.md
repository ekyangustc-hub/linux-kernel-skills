---
name: "ext4-disk-structure"
description: "ext4文件系统静态磁盘结构专家。当用户询问ext4磁盘布局、超级块、块组描述符、inode结构、extent树、目录项、特性耦合(flex_bg/sparse_bigalloc等)时调用此技能。"
---

# ext4 静态磁盘结构

本技能详细描述 ext4 文件系统的静态磁盘布局，包括超级块、块组描述符、inode、extent 树、目录项等核心结构，以及各种特性之间的耦合关系。

---

## 一、整体布局

### 1.1 文件系统结构

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              Block Group 0                                  │
│  ┌─────────┬─────────────┬───────────────┬───────────────┬───────────────┐ │
│  │Superblock│ Group Desc  │  Block Bitmap │ Inode Bitmap  │  Inode Table  │ │
│  │ (块 0)   │  Table      │               │               │               │ │
│  └─────────┴─────────────┴───────────────┴───────────────┴───────────────┘ │
│                                    Data Blocks                              │
└────────────────────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────────────────────┐
│                              Block Group 1                                  │
│  ┌───────────────┬───────────────┬───────────────┬───────────────────────┐ │
│  │  Block Bitmap │ Inode Bitmap  │  Inode Table  │     Data Blocks       │ │
│  └───────────────┴───────────────┴───────────────┴───────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────┘
                                    ...
```

### 1.2 核心结构关系

```
ext4_super_block (超级块)
        │
        ▼
ext4_group_desc (块组描述符) × N
        │
        ├──────────────────┬──────────────────┐
        ▼                  ▼                  ▼
  Block Bitmap       Inode Bitmap       Inode Table
                                          │
                                          ▼
                                    ext4_inode
                                          │
                        ┌─────────────────┼─────────────────┐
                        ▼                 ▼                 ▼
                   数据块            目录块             Extent块
```

---

## 二、超级块 (ext4_super_block)

### 2.1 结构定义

**位置**: `fs/ext4/ext4.h:1332-1463`

```c
struct ext4_super_block {
    /* 基本信息 */
    __le32  s_inodes_count;         /* Inodes 总数 */
    __le32  s_blocks_count_lo;      /* 块总数 (低32位) */
    __le32  s_r_blocks_count_lo;    /* 保留块数 */
    __le32  s_free_blocks_count_lo; /* 空闲块数 */
    __le32  s_free_inodes_count;    /* 空闲 inode 数 */
    __le32  s_first_data_block;     /* 第一个数据块 (1K块大小为1，否则为0) */
    __le32  s_log_block_size;       /* 块大小 = 1024 << s_log_block_size */
    __le32  s_log_cluster_size;     /* 簇大小 (bigalloc) */
    
    /* 块组信息 */
    __le32  s_blocks_per_group;     /* 每组块数 */
    __le32  s_clusters_per_group;   /* 每组簇数 */
    __le32  s_inodes_per_group;     /* 每组 inode 数 */
    
    /* 时间戳 */
    __le32  s_mtime;                /* 最后挂载时间 */
    __le32  s_wtime;                /* 最后写入时间 */
    
    /* 挂载信息 */
    __le16  s_mnt_count;            /* 挂载次数 */
    __le16  s_max_mnt_count;        /* 最大挂载次数 (fsck检查) */
    __le16  s_magic;                /* 魔数 0xEF53 */
    __le16  s_state;                /* 文件系统状态 */
    __le16  s_errors;               /* 错误处理策略 */
    
    /* 标识信息 */
    __le16  s_minor_rev_level;      /* 次版本号 */
    __le32  s_lastcheck;            /* 最后检查时间 */
    __le32  s_checkinterval;        /* 检查间隔 */
    __le32  s_creator_os;           /* 创建者 OS */
    __le32  s_rev_level;            /* 版本号 */
    __le16  s_def_resuid;           /* 保留块默认 UID */
    __le16  s_def_resgid;           /* 保留块默认 GID */
    
    /* EXT4 动态特性 */
    __le32  s_first_ino;            /* 第一个非保留 inode */
    __le16  s_inode_size;           /* inode 大小 */
    __le16  s_block_group_nr;       /* 当前块组号 (备份超级块) */
    __le32  s_feature_compat;       /* 兼容特性 */
    __le32  s_feature_incompat;     /* 不兼容特性 */
    __le32  s_feature_ro_compat;    /* 只读兼容特性 */
    
    /* UUID 和卷名 */
    __u8    s_uuid[16];             /* 128-bit UUID */
    char    s_volume_name[16];      /* 卷名 */
    char    s_last_mounted[64];     /* 最后挂载路径 */
    
    /* 日志相关 */
    __le32  s_algorithm_usage_bitmap;
    __u8    s_prealloc_blocks;      /* 预分配块数 */
    __u8    s_prealloc_dir_blocks;  /* 目录预分配块数 */
    __le16  s_reserved_gdt_blocks;  /* 为扩展预留的 GDT 块 */
    __u8    s_journal_uuid[16];     /* 日志 UUID */
    __le32  s_journal_inum;         /* 日志 inode 号 */
    __le32  s_journal_dev;          /* 日志设备号 */
    __le32  s_last_orphan;          /* 孤儿 inode 链表头 */
    
    /* 64位扩展 */
    __le32  s_blocks_count_hi;      /* 块总数 (高32位) */
    __le32  s_r_blocks_count_hi;
    __le32  s_free_blocks_count_hi;
    __le16  s_min_extra_isize;      /* 最小扩展 inode 大小 */
    __le16  s_want_extra_isize;     /* 期望扩展 inode 大小 */
    __le32  s_flags;                /* 各种标志 */
    __le16  s_raid_stride;          /* RAID 步长 */
    __le16  s_mmp_interval;         /* MMP 检查间隔 */
    __le64  s_mmp_block;            /* MMP 块号 */
    __le32  s_raid_stripe_width;    /* RAID 条带宽度 */
    
    /* Flex BG 相关 */
    __u8    s_log_groups_per_flex;  /* 每 flex 组的块组数 (log2) */
    __u8    s_checksum_type;        /* 校验和类型 */
    __le16  s_reserved_pad;
    __le64  s_kbytes_written;       /* 写入字节数 */
    __le32  s_snapshot_inum;
    __le32  s_snapshot_id;
    __le64  s_snapshot_r_blocks_count;
    __le32  s_snapshot_list;
    
    /* 校验和 */
    __le32  s_checksum;             /* 超级块校验和 */
    __le32  s_error_count;          /* 错误计数 */
    __le32  s_first_error_time;     /* 首次错误时间 */
    __le32  s_first_error_ino;
    __le64  s_first_error_block;
    __u8    s_first_error_func[32];
    __le32  s_first_error_line;
    __le32  s_last_error_time;
    __le32  s_last_error_ino;
    __le64  s_last_error_block;
    __u8    s_last_error_func[32];
    __le32  s_last_error_line;
    
    /* 加密相关 */
    __u8    s_encrypt_pw_salt[16];
    __le32  s_lpf_ino;              /* lost+found inode */
    __le32  s_prj_quota_inum;       /* 项目配额 inode */
    __le32  s_checksum_seed;        /* 校验和种子 */
    
    /* 大文件系统扩展 */
    __u8    s_wtime_hi;
    __u8    s_mtime_hi;
    __u8    s_mkfs_time_hi;
    __u8    s_lastcheck_hi;
    __u8    s_first_error_time_hi;
    __u8    s_last_error_time_hi;
    __u8    s_pad[2];
    __le16  s_encoding;             /* 字符编码 */
    __le16  s_encoding_flags;
    __le32  s_orphan_file_inum;     /* 孤儿文件 inode */
    __le32  s_reserved[94];
    __le32  s_checksum;             /* CRC32c 校验和 */
};
```

### 2.2 关键字段详解

| 字段 | 说明 | 计算方式 |
|------|------|----------|
| `s_log_block_size` | 块大小对数 | `block_size = 1024 << s_log_block_size` |
| `s_blocks_per_group` | 每组块数 | 通常是 `8 * block_size` (一个位图块能表示的块数) |
| `s_inodes_per_group` | 每组 inode 数 | 取决于 inode 大小和预留空间 |
| `s_inode_size` | inode 大小 | 通常 256 字节 (EXT4_MIN_INODE_SIZE=128) |
| `s_desc_size` | 块组描述符大小 | 32 字节 (无64bit) 或 64 字节 (有64bit) |

### 2.3 特性标志位

**位置**: `fs/ext4/ext4.h:2072-2127`

```c
/* COMPAT 特性 - 老内核可读写 */
#define EXT4_FEATURE_COMPAT_DIR_PREALLOC    0x0001
#define EXT4_FEATURE_COMPAT_IMAGIC_INODES   0x0002
#define EXT4_FEATURE_COMPAT_HAS_JOURNAL     0x0004
#define EXT4_FEATURE_COMPAT_EXT_ATTR        0x0008
#define EXT4_FEATURE_COMPAT_RESIZE_INODE    0x0010
#define EXT4_FEATURE_COMPAT_DIR_INDEX       0x0020
#define EXT4_FEATURE_COMPAT_SPARSE_SUPER2   0x0200

/* RO_COMPAT 特性 - 老内核只读挂载 */
#define EXT4_FEATURE_RO_COMPAT_SPARSE_SUPER 0x0001  /* 稀疏超级块 */
#define EXT4_FEATURE_RO_COMPAT_LARGE_FILE   0x0002
#define EXT4_FEATURE_RO_COMPAT_BTREE_DIR    0x0004
#define EXT4_FEATURE_RO_COMPAT_HUGE_FILE    0x0008
#define EXT4_FEATURE_RO_COMPAT_GDT_CSUM     0x0010  /* 块组描述符校验 */
#define EXT4_FEATURE_RO_COMPAT_DIR_NLINK    0x0020
#define EXT4_FEATURE_RO_COMPAT_EXTRA_ISIZE  0x0040
#define EXT4_FEATURE_RO_COMPAT_HAS_SNAPSHOT 0x0080
#define EXT4_FEATURE_RO_COMPAT_QUOTA        0x0100
#define EXT4_FEATURE_RO_COMPAT_BIGALLOC     0x0200  /* 大块分配 */
#define EXT4_FEATURE_RO_COMPAT_METADATA_CSUM 0x0400 /* 元数据校验 */
#define EXT4_FEATURE_RO_COMPAT_READONLY     0x1000
#define EXT4_FEATURE_RO_COMPAT_PROJECT      0x2000

/* INCOMPAT 特性 - 老内核无法挂载 */
#define EXT4_FEATURE_INCOMPAT_COMPRESSION   0x0001
#define EXT4_FEATURE_INCOMPAT_FILETYPE      0x0002
#define EXT4_FEATURE_INCOMPAT_RECOVER       0x0004  /* 需要日志恢复 */
#define EXT4_FEATURE_INCOMPAT_JOURNAL_DEV   0x0008
#define EXT4_FEATURE_INCOMPAT_META_BG       0x0010  /* 元块组 */
#define EXT4_FEATURE_INCOMPAT_EXTENTS       0x0040  /* Extent 支持 */
#define EXT4_FEATURE_INCOMPAT_64BIT         0x0080  /* 64位支持 */
#define EXT4_FEATURE_INCOMPAT_MMP           0x0100  /* 多挂载保护 */
#define EXT4_FEATURE_INCOMPAT_FLEX_BG       0x0200  /* 灵活块组 */
#define EXT4_FEATURE_INCOMPAT_EA_INODE      0x0400
#define EXT4_FEATURE_INCOMPAT_DIRDATA       0x1000
#define EXT4_FEATURE_INCOMPAT_CSUM_SEED     0x2000
#define EXT4_FEATURE_INCOMPAT_LARGEDIR      0x4000  /* 大目录 */
#define EXT4_FEATURE_INCOMPAT_INLINE_DATA   0x8000
#define EXT4_FEATURE_INCOMPAT_ENCRYPT       0x10000
#define EXT4_FEATURE_INCOMPAT_CASEFOLD      0x20000
#define EXT4_FEATURE_INCOMPAT_ORPHAN_FILE   0x40000
```

---

## 三、块组描述符 (ext4_group_desc)

### 3.1 结构定义

**位置**: `fs/ext4/ext4.h:403-428`

```c
struct ext4_group_desc {
    /* 32位部分 - 基本字段 */
    __le32  bg_block_bitmap_lo;     /* 块位图块号 (低32位) */
    __le32  bg_inode_bitmap_lo;     /* inode 位图块号 (低32位) */
    __le32  bg_inode_table_lo;      /* inode 表起始块号 (低32位) */
    __le16  bg_free_blocks_count_lo;/* 空闲块数 (低16位) */
    __le16  bg_free_inodes_count_lo;/* 空闲 inode 数 (低16位) */
    __le16  bg_used_dirs_count_lo;  /* 已用目录数 (低16位) */
    __le16  bg_flags;               /* 块组标志 */
    __le32  bg_exclude_bitmap_lo;   /* 排除位图 (快照) */
    __le16  bg_block_bitmap_csum_lo;/* 块位图校验和 (低16位) */
    __le16  bg_inode_bitmap_csum_lo;/* inode 位图校验和 (低16位) */
    __le16  bg_itable_unused_lo;    /* 未用 inode 数 */
    __le16  bg_checksum;            /* 块组描述符校验和 */
    
    /* 64位扩展部分 - 仅当 EXT4_FEATURE_INCOMPAT_64BIT */
    __le32  bg_block_bitmap_hi;     /* 块位图块号 (高32位) */
    __le32  bg_inode_bitmap_hi;     /* inode 位图块号 (高32位) */
    __le32  bg_inode_table_hi;      /* inode 表起始块号 (高32位) */
    __le16  bg_free_blocks_count_hi;/* 空闲块数 (高16位) */
    __le16  bg_free_inodes_count_hi;/* 空闲 inode 数 (高16位) */
    __le16  bg_used_dirs_count_hi;  /* 已用目录数 (高16位) */
    __le16  bg_itable_unused_hi;    /* 未用 inode 数 */
    __le32  bg_exclude_bitmap_hi;   /* 排除位图 (高32位) */
    __le16  bg_block_bitmap_csum_hi;/* 块位图校验和 (高16位) */
    __le16  bg_inode_bitmap_csum_hi;/* inode 位图校验和 (高16位) */
    __le32  bg_reserved;
};
```

### 3.2 块组标志 (bg_flags)

```c
#define EXT4_BG_INODE_UNINIT    0x0001  /* inode 位图未初始化 */
#define EXT4_BG_BLOCK_UNINIT    0x0002  /* 块位图未初始化 */
#define EXT4_BG_INODE_ZEROED    0x0004  /* inode 表已清零 */
```

### 3.3 块组描述符表位置

| 模式 | 位置 |
|------|------|
| 传统模式 | 超级块之后 (块 1 开始) |
| meta_bg 模式 | 分散在各 meta_group 的第一个块组 |

---

## 四、Inode 结构 (ext4_inode)

### 4.1 结构定义

**位置**: `fs/ext4/ext4.h:795-854`

```c
struct ext4_inode {
    __le16  i_mode;         /* 文件模式 (类型+权限) */
    __le16  i_uid;          /* 用户 ID (低16位) */
    __le32  i_size_lo;      /* 文件大小 (低32位) */
    __le32  i_atime;        /* 访问时间 */
    __le32  i_ctime;        /* 状态改变时间 */
    __le32  i_mtime;        /* 修改时间 */
    __le32  i_dtime;        /* 删除时间 */
    __le16  i_gid;          /* 组 ID (低16位) */
    __le16  i_links_count;  /* 硬链接数 */
    __le32  i_blocks_lo;    /* 块数 (512字节为单位) */
    __le32  i_flags;        /* 文件标志 */
    
    /* OS 相关字段 */
    union {
        struct {
            __le32  l_i_version;
        } linux1;
        struct {
            __le32  h_i_translator;
        } hurd1;
        struct {
            __le32  m_i_reserved1;
        } masix1;
    } osd1;
    
    /* 块指针或 extent 头 */
    __le32  i_block[15];    /* 见下文详细说明 */
    
    __le32  i_generation;   /* 文件版本 (NFS) */
    __le32  i_file_acl_lo;  /* 扩展属性块 (低32位) */
    __le32  i_size_high;    /* 文件大小 (高32位) 或 目录 ACL */
    
    /* 扩展字段 (EXT4_GOOD_OLD_INODE_SIZE=128 之后) */
    __le32  i_obso_faddr;   /* 已废弃 */
    union {
        struct {
            __le16  l_i_blocks_high; /* 块数高16位 */
            __le16  l_i_file_acl_high; /* 扩展属性块高16位 */
            __le16  l_i_uid_high;    /* 用户 ID 高16位 */
            __le16  l_i_gid_high;    /* 组 ID 高16位 */
            __le16  l_i_checksum_lo; /* inode 校验和低16位 */
            __le16  l_i_reserved;
        } linux2;
        struct {
            __le16  h_i_reserved1;
            __le16  h_i_mode_high;
            __le16  h_i_uid_high;
            __le16  h_i_gid_high;
            __le32  h_i_author;
        } hurd2;
    } osd2;
    
    /* 进一步扩展字段 */
    __le16  i_extra_isize;  /* 扩展字段大小 */
    __le16  i_checksum_hi;  /* inode 校验和高16位 */
    __le32  i_ctime_extra;  /* ctime 纳秒部分 */
    __le32  i_mtime_extra;  /* mtime 纳秒部分 */
    __le32  i_atime_extra;  /* atime 纳秒部分 */
    __le32  i_crtime;       /* 创建时间 */
    __le32  i_crtime_extra; /* 创建时间纳秒部分 */
    __le32  i_projid;       /* 项目 ID */
    char    i_pad[4];       /* 填充 */
};
```

### 4.2 i_block[15] 的两种用法

#### 传统间接块模式

```
i_block[0-11]:  直接块指针 (12个)
i_block[12]:    一级间接块指针
i_block[13]:    二级间接块指针
i_block[14]:    三级间接块指针

最大文件大小 (4K块):
= 12 + 256 + 256² + 256³ 个块
= 12 + 256 + 65536 + 16777216 个块
≈ 64GB
```

#### Extent 模式 (i_flags & EXT4_EXTENTS_FL)

```c
/* i_block 存储的内容 */
struct ext4_extent_header {
    __le16  eh_magic;       /* 0xF30A */
    __le16  eh_entries;     /* 有效条目数 */
    __le16  eh_max;         /* 最大条目数 */
    __le16  eh_depth;       /* 树深度 (0=叶子) */
    __le32  eh_generation;  /* 树版本 */
};

/* 后跟 extent 或 extent_idx 数组 */
```

### 4.3 Inode 标志 (i_flags)

```c
#define EXT4_SECRM_FL           0x00000001 /* 安全删除 */
#define EXT4_UNRM_FL            0x00000002 /* 可恢复删除 */
#define EXT4_COMPR_FL           0x00000004 /* 压缩文件 */
#define EXT4_SYNC_FL            0x00000008 /* 同步更新 */
#define EXT4_IMMUTABLE_FL       0x00000010 /* 不可变 */
#define EXT4_APPEND_FL          0x00000020 /* 只能追加 */
#define EXT4_NODUMP_FL          0x00000040 /* 不转储 */
#define EXT4_NOATIME_FL         0x00000080 /* 不更新 atime */
#define EXT4_INDEX_FL           0x00001000 /* HTREE 索引 */
#define EXT4_IMAGIC_FL          0x00002000 /* AFS 目录 */
#define EXT4_JOURNAL_DATA_FL    0x00004000 /* 数据日志模式 */
#define EXT4_NOTAIL_FL          0x00008000 /* 不合并尾部 */
#define EXT4_DIRSYNC_FL         0x00010000 /* 目录同步 */
#define EXT4_TOPDIR_FL          0x00020000 /* 顶级目录 */
#define EXT4_HUGE_FILE_FL       0x00040000 /* 大文件 */
#define EXT4_EXTENTS_FL         0x00080000 /* 使用 Extent */
#define EXT4_EA_INODE_FL        0x00200000 /* 扩展属性 inode */
#define EXT4_EOFBLOCKS_FL       0x00400000 /* 预分配块 */
#define EXT4_DAX_FL             0x02000000 /* DAX 模式 */
#define EXT4_INLINE_DATA_FL     0x10000000 /* 内联数据 */
#define EXT4_PROJINHERIT_FL     0x20000000 /* 项目继承 */
#define EXT4_CASEFOLD_FL        0x40000000 /* 大小写折叠 */
```

---

## 五、Extent 树结构

### 5.1 结构定义

**位置**: `fs/ext4/ext4_extents.h:48-84`

```c
/* Extent 头 - 每个 extent 块的开头 */
struct ext4_extent_header {
    __le16  eh_magic;       /* 魔数 0xF30A */
    __le16  eh_entries;     /* 有效条目数 */
    __le16  eh_max;         /* 最大条目数 */
    __le16  eh_depth;       /* 树深度 (0=叶子节点) */
    __le32  eh_generation;  /* 树版本号 */
};

/* Extent 结构 - 叶子节点 */
struct ext4_extent {
    __le32  ee_block;       /* 起始逻辑块号 */
    __le16  ee_len;         /* 连续块数 (最大 32768) */
    __le16  ee_start_hi;    /* 物理块号高16位 */
    __le32  ee_start_lo;    /* 物理块号低32位 */
};

/* Extent 索引 - 内部节点 */
struct ext4_extent_idx {
    __le32  ei_block;       /* 起始逻辑块号 */
    __le32  ei_leaf_lo;     /* 子节点物理块号低32位 */
    __le16  ee_leaf_hi;     /* 子节点物理块号高16位 */
    __u16   ei_unused;
};

/* Extent 块尾部校验 */
struct ext4_extent_tail {
    __le32  et_checksum;    /* CRC32c 校验和 */
};
```

### 5.2 Extent 树布局

```
                    ┌─────────────────────────────┐
                    │     i_block[15] (根节点)     │
                    │  ext4_extent_header          │
                    │  eh_depth = 2 (树高度3层)    │
                    │  ext4_extent_idx[]           │
                    └──────────────┬──────────────┘
                                   │
            ┌──────────────────────┼──────────────────────┐
            ▼                      ▼                      ▼
    ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
    │ 索引块 (深度1) │      │ 索引块 (深度1) │      │ 索引块 (深度1) │
    │ eh_depth = 1  │      │ eh_depth = 1  │      │ eh_depth = 1  │
    │ ext4_extent_  │      │ ext4_extent_  │      │ ext4_extent_  │
    │ idx[]         │      │ idx[]         │      │ idx[]         │
    └───────┬───────┘      └───────┬───────┘      └───────┬───────┘
            │                      │                      │
     ┌──────┴──────┐        ┌──────┴──────┐        ┌──────┴──────┐
     ▼             ▼        ▼             ▼        ▼             ▼
┌─────────┐  ┌─────────┐ ┌─────────┐  ┌─────────┐ ┌─────────┐  ┌─────────┐
│叶子块   │  │叶子块   │ │叶子块   │  │叶子块   │ │叶子块   │  │叶子块   │
│depth=0  │  │depth=0  │ │depth=0  │  │depth=0  │ │depth=0  │  │depth=0  │
│extent[] │  │extent[] │ │extent[] │  │extent[] │ │extent[] │  │extent[] │
└─────────┘  └─────────┘ └─────────┘  └─────────┘ └─────────┘  └─────────┘
     │             │          │             │          │             │
     ▼             ▼          ▼             ▼          ▼             ▼
  数据块        数据块      数据块        数据块      数据块        数据块
```

### 5.3 Extent 特殊标志

```c
/* ee_len 的特殊值 */
#define EXT4_INIT_MAX_LEN       32768   /* 最大长度 */
#define EXT4_MAX_LEN            32768   /* 最大长度 */
#define EXT4_EXTENT_LEN_MASK    0x7FFF  /* 长度掩码 */

/* 未初始化 extent */
#define EXT4_EXT_STATUS_UNWRITTEN 0x8000  /* ee_len 最高位 */
```

---

## 六、目录项结构

### 6.1 基本目录项

**位置**: `fs/ext4/ext4.h:2396-2454`

```c
/* 传统目录项 */
struct ext4_dir_entry {
    __le32  inode;          /* inode 号 */
    __le16  rec_len;        /* 目录项长度 */
    __le16  name_len;       /* 文件名长度 */
    char    name[255];      /* 文件名 */
};

/* 新版目录项 (file_type) */
struct ext4_dir_entry_2 {
    __le32  inode;          /* inode 号 */
    __le16  rec_len;        /* 目录项长度 */
    __u8    name_len;       /* 文件名长度 */
    __u8    file_type;      /* 文件类型 */
    char    name[255];      /* 文件名 */
};

/* 文件类型 */
#define EXT4_FT_UNKNOWN     0
#define EXT4_FT_REG_FILE    1   /* 普通文件 */
#define EXT4_FT_DIR         2   /* 目录 */
#define EXT4_FT_CHRDEV      3   /* 字符设备 */
#define EXT4_FT_BLKDEV      4   /* 块设备 */
#define EXT4_FT_FIFO        5   /* FIFO */
#define EXT4_FT_SOCK        6   /* Socket */
#define EXT4_FT_SYMLINK     7   /* 符号链接 */
```

### 6.2 目录块尾部校验

```c
struct ext4_dir_entry_tail {
    __le32  det_reserved_zero1;
    __le16  det_rec_len;        /* 12 */
    __u8    det_reserved_ft;    /* 0xDE */
    __u8    det_reserved_zero2;
    __le32  det_checksum;       /* CRC32c 校验和 */
};
```

### 6.3 HTREE 索引结构

**位置**: `fs/ext4/namei.c:235-292`

```c
/* 索引条目 */
struct dx_entry {
    __le32  hash;           /* 哈希值 */
    __le32  block;          /* 目录块号 */
};

/* HTREE 根节点 */
struct dx_root {
    struct ext4_dir_entry_2 dot;      /* "." 条目 */
    char                    dot_name[4];
    struct ext4_dir_entry_2 dotdot;   /* ".." 条目 */
    char                    dotdot_name[4];
    struct dx_root_info {
        __le32  reserved_zero;
        __u8    hash_version;   /* 哈希版本 */
        __u8    info_length;    /* 信息长度 (8) */
        __u8    indirect_levels; /* 间接层数 (0-1) */
        __u8    unused_flags;
    } info;
    struct dx_entry         entries[];
};

/* HTREE 内部节点 */
struct dx_node {
    struct ext4_fake_dirent fake;
    struct dx_entry         entries[];
};

/* 哈希版本 */
#define DX_HASH_LEGACY          0   /* 传统哈希 */
#define DX_HASH_HALF_MD4        1   /* 半 MD4 */
#define DX_HASH_TEA             2   /* TEA */
#define DX_HASH_LEGACY_UNSIGNED 3
#define DX_HASH_HALF_MD4_UNSIGNED 4
#define DX_HASH_TEA_UNSIGNED    5
#define DX_HASH_SIPHASH         6   /* SipHash (casefold) */
```

### 6.4 HTREE 目录布局

```
┌─────────────────────────────────────────────────────────────────┐
│                        目录块 0 (根块)                           │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ "."  → inode=dir_inode                                      ││
│  │ ".." → inode=parent_inode                                   ││
│  │ dx_root_info: hash_version, indirect_levels                 ││
│  │ dx_entry[0]: hash=0x0000, block=1                           ││
│  │ dx_entry[1]: hash=0x4000, block=2                           ││
│  │ dx_entry[2]: hash=0x8000, block=3                           ││
│  │ ...                                                         ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   目录块 1     │    │   目录块 2     │    │   目录块 3     │
│ hash < 0x4000 │    │ 0x4000-0x7FFF │    │ 0x8000-0xBFFF │
│ dir_entry[]   │    │ dir_entry[]   │    │ dir_entry[]   │
│ file_a        │    │ file_m        │    │ file_x        │
│ file_b        │    │ file_n        │    │ file_y        │
│ ...           │    │ ...           │    │ ...           │
└───────────────┘    └───────────────┘    └───────────────┘
```

---

## 七、特性耦合关系

### 7.1 特性依赖图

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                    ext4 特性耦合关系                      │
                    └─────────────────────────────────────────────────────────┘
                                               │
           ┌───────────────────────────────────┼───────────────────────────────────┐
           │                                   │                                   │
           ▼                                   ▼                                   ▼
    ┌─────────────┐                    ┌─────────────┐                    ┌─────────────┐
    │   64bit     │                    │   flex_bg   │                    │  bigalloc   │
    │  (INCOMPAT) │                    │  (INCOMPAT) │                    │ (RO_COMPAT) │
    └──────┬──────┘                    └──────┬──────┘                    └──────┬──────┘
           │                                  │                                   │
           │ 扩展块组描述符                    │ 改变元数据布局                    │ 必须依赖
           │ 从 32→64 字节                    │ 位图/inode表可跨组                │
           ▼                                  ▼                                   ▼
    ┌─────────────┐                    ┌─────────────┐                    ┌─────────────┐
    │ 支持超大卷   │                    │ 优化分配策略 │                    │   extents   │
    │ (>16TB)     │                    │ 减少碎片     │                    │  (INCOMPAT) │
    └─────────────┘                    └─────────────┘                    └─────────────┘
```

### 7.2 flex_bg 特性

**定义**: `EXT4_FEATURE_INCOMPAT_FLEX_BG 0x0200`

**核心改变**: 允许块位图、inode 位图、inode 表不在本块组内

```c
/* 超级块字段 */
__u8  s_log_groups_per_flex;  /* 每 flex 组的块组数 (log2) */

/* 内存结构 */
struct flex_groups {
    atomic64_t  free_clusters;  /* 空闲簇数 */
    atomic_t    free_inodes;    /* 空闲 inode 数 */
    atomic_t    used_dirs;      /* 已用目录数 */
};
```

**布局变化**:

```
传统布局 (每个块组独立):
┌─────────────────────────────────────────────────────────────────┐
│ Block Group 0                                                    │
│ ┌─────────┬─────────┬───────────┬───────────┬─────────────────┐│
│ │Superblock│ GDT    │Block Bitmap│Inode Bitmap│ Inode Table    ││
│ └─────────┴─────────┴───────────┴───────────┴─────────────────┘│
│                          Data Blocks                            │
└─────────────────────────────────────────────────────────────────┘

flex_bg 布局 (多个块组共享):
┌─────────────────────────────────────────────────────────────────┐
│ Flex Group 0 (包含 Block Group 0, 1, 2, ...)                    │
│ ┌─────────┬─────────┬───────────┬───────────┬─────────────────┐│
│ │Superblock│ GDT    │Block Bitmap│Inode Bitmap│ Inode Table    ││
│ │         │         │(所有BG的)  │(所有BG的)  │(所有BG的)      ││
│ └─────────┴─────────┴───────────┴───────────┴─────────────────┘│
│ ┌─────────────────────────────────────────────────────────────┐│
│ │                    所有块组的数据块                          ││
│ └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### 7.3 sparse_super 特性

**定义**: `EXT4_FEATURE_RO_COMPAT_SPARSE_SUPER 0x0001`

**超级块备份位置判断**:

```c
int ext4_bg_has_super(struct super_block *sb, ext4_group_t group)
{
    if (group == 0) return 1;
    
    /* sparse_super2: 只在指定的两个备份组 */
    if (ext4_has_feature_sparse_super2(sb)) {
        if (group == le32_to_cpu(es->s_backup_bgs[0]) ||
            group == le32_to_cpu(es->s_backup_bgs[1]))
            return 1;
        return 0;
    }
    
    /* 标准 sparse_super: 组号是 3, 5, 7 的幂次 */
    if (!(group & 1)) return 0;  /* 偶数组无备份 */
    if (test_root(group, 3) || test_root(group, 5) || test_root(group, 7))
        return 1;
    return 0;
}
```

### 7.4 meta_bg 特性

**定义**: `EXT4_FEATURE_INCOMPAT_META_BG 0x0010`

**块组描述符位置变化**:

```c
ext4_fsblk_t ext4_get_descriptor_location(struct super_block *sb,
                                          ext4_fsblk_t logical_sb_block,
                                          int nr)
{
    first_meta_bg = le32_to_cpu(sbi->s_es->s_first_meta_bg);
    
    /* 传统模式: GDT 在超级块之后 */
    if (!ext4_has_feature_meta_bg(sb) || nr < first_meta_bg)
        return logical_sb_block + nr + 1;
    
    /* meta_bg 模式: GDT 在每个 meta_group 的第一个块组 */
    bg = sbi->s_desc_per_block * nr;
    return (has_super + ext4_group_first_block_no(sb, bg));
}
```

### 7.5 bigalloc 特性

**定义**: `EXT4_FEATURE_RO_COMPAT_BIGALLOC 0x0200`

**强制依赖**: 必须启用 extents 特性

```c
if (ext4_has_feature_bigalloc(sb) && !ext4_has_feature_extents(sb)) {
    ext4_msg(sb, KERN_ERR,
             "Can't support bigalloc feature without extents feature\n");
    return 0;
}
```

**簇大小处理**:

```c
/* 超级块字段 */
__le32  s_log_cluster_size;     /* 簇大小对数 */
__le32  s_clusters_per_group;   /* 每组簇数 */

/* 内存计算 */
clustersize = BLOCK_SIZE << le32_to_cpu(es->s_log_cluster_size);
sbi->s_cluster_bits = le32_to_cpu(es->s_log_cluster_size) -
                      le32_to_cpu(es->s_log_block_size);
sbi->s_cluster_ratio = clustersize / sb->s_blocksize;
```

### 7.6 校验和特性

**gdt_csum (uninit_bg)**: `EXT4_FEATURE_RO_COMPAT_GDT_CSUM 0x0010`
**metadata_csum**: `EXT4_FEATURE_RO_COMPAT_METADATA_CSUM 0x0400`

**互斥关系**: metadata_csum 是 gdt_csum 的超集，两者不能同时设置

```c
static inline int ext4_has_group_desc_csum(struct super_block *sb)
{
    return ext4_has_feature_gdt_csum(sb) ||
           ext4_has_feature_metadata_csum(sb);
}
```

**校验和算法差异**:

| 特性 | 算法 | 范围 |
|------|------|------|
| gdt_csum | CRC16 | 仅块组描述符 |
| metadata_csum | CRC32c | 超级块、inode、目录、extent 等 |

---

## 八、不同类型文件的块组织

### 8.1 普通文件

```
┌─────────────────────────────────────────────────────────────────┐
│                          普通文件                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Extent 模式 (推荐):                                            │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ inode.i_block = ext4_extent_header + ext4_extent[]          ││
│  │                                                              ││
│  │ extent[0]: logical=0,    len=100, physical=1000             ││
│  │ extent[1]: logical=100,  len=50,  physical=2000             ││
│  │ extent[2]: logical=150,  len=200, physical=3000             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  间接块模式 (兼容):                                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ i_block[0-11]:  直接块指针 (12个)                            ││
│  │ i_block[12]:    一级间接块 → 256个直接块指针                  ││
│  │ i_block[13]:    二级间接块 → 256个一级间接块                  ││
│  │ i_block[14]:    三级间接块 → 256个二级间接块                  ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 目录文件

```
┌─────────────────────────────────────────────────────────────────┐
│                          目录文件                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  小目录 (线性扫描):                                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 目录块: ext4_dir_entry_2[] 按顺序排列                        ││
│  │ ["."][inode=100][rec_len=12]                                ││
│  │ [".."][inode=2][rec_len=12]                                 ││
│  │ ["file1"][inode=101][rec_len=16]                            ││
│  │ ["file2"][inode=102][rec_len=16]                            ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  大目录 (HTREE 索引):                                            │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 块 0: dx_root (根节点)                                       ││
│  │       ├── "." 和 ".." 目录项                                 ││
│  │       ├── dx_root_info                                       ││
│  │       └── dx_entry[] (索引条目)                              ││
│  │                                                              ││
│  │ 块 1-N: dx_node (内部节点) 或 叶子块                         ││
│  │         └── ext4_dir_entry_2[]                               ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3 符号链接

```
┌─────────────────────────────────────────────────────────────────┐
│                          符号链接                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  短链接 (目标路径 ≤ 60 字节):                                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ inode.i_block[] 直接存储目标路径                             ││
│  │ i_block[0-14]: "/path/to/target"                            ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  长链接 (目标路径 > 60 字节):                                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 分配数据块存储目标路径                                       ││
│  │ inode.i_block[0] → 数据块 → "/very/long/path/to/target"     ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.4 特殊文件

```
┌─────────────────────────────────────────────────────────────────┐
│                        特殊文件                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  设备文件 (字符/块设备):                                         │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ i_block[0] 存储设备号 (major:minor)                          ││
│  │ 不占用数据块                                                 ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  FIFO/Socket:                                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 不占用数据块，只有 inode                                     ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 九、关键代码位置索引

| 结构 | 定义位置 | 说明 |
|------|----------|------|
| ext4_super_block | `fs/ext4/ext4.h:1332-1463` | 超级块 |
| ext4_group_desc | `fs/ext4/ext4.h:403-428` | 块组描述符 |
| ext4_inode | `fs/ext4/ext4.h:795-854` | Inode |
| ext4_extent_header | `fs/ext4/ext4_extents.h:48-56` | Extent 头 |
| ext4_extent | `fs/ext4/ext4_extents.h:58-64` | Extent 结构 |
| ext4_extent_idx | `fs/ext4/ext4_extents.h:66-72` | Extent 索引 |
| ext4_dir_entry_2 | `fs/ext4/ext4.h:2396-2454` | 目录项 |
| dx_root | `fs/ext4/namei.c:235-260` | HTREE 根节点 |
| dx_node | `fs/ext4/namei.c:262-266` | HTREE 内部节点 |
| flex_groups | `fs/ext4/ext4.h:441-445` | Flex 组统计 |
