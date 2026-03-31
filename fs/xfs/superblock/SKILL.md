---
name: "xfs-superblock"
description: "XFS文件系统超级块专家。当用户询问XFS超级块结构、sb_magicnum、sb_blocksize、sb_agcount、文件系统参数时调用此技能。"
---

# XFS Super Block (超级块)

## 一、概述

Super Block 是 XFS 文件系统的第一个结构，位于每个 AG 的起始位置（Sector 0），存储文件系统的核心元数据信息。

---

## 二、结构定义

**位置**: `fs/xfs/libxfs/xfs_format.h`

```c
struct xfs_sb {
    __be32  sb_magicnum;            /* 魔数 XFSB */
    __be32  sb_blocksize;           /* 块大小 */
    __be64  sb_dblocks;             /* 数据块总数 */
    __be64  sb_rblocks;             /* 实时设备块数 */
    __be64  sb_rextents;            /* 实时 extent 数 */
    uuid_t  sb_uuid;                /* 文件系统 UUID */
    __be64  sb_logstart;            /* 日志起始块号 */
    __be64  sb_rootino;             /* 根目录 inode 号 */
    __be64  sb_rbmino;              /* 实时位图 inode 号 */
    __be64  sb_rsumino;             /* 实时汇总 inode 号 */
    __be32  sb_rextsize;            /* 实时 extent 大小 */
    __be32  sb_agblocks;            /* 每个 AG 的块数 */
    __be32  sb_agcount;             /* AG 数量 */
    __be32  sb_rbmblocks;           /* 实时位图块数 */
    __be32  sb_logblocks;           /* 日志块数 */
    __be16  sb_versionnum;          /* 版本号 */
    __be16  sb_sectsize;            /* 扇区大小 */
    __be16  sb_inodesize;           /* inode 大小 */
    __be16  sb_inopblock;           /* 每块 inode 数 */
    __be32  sb_inoalignmt;          /* inode 对齐 */
    __be32  sb_unit;                /* 条带单元 */
    __be32  sb_width;               /* 条带宽度 */
    __be32  sb_features2;           /* 特性标志 2 */
    __be32  sb_bad_features2;       /* 错误的特性标志 2 */
    __be32  sb_features_compat;     /* 兼容特性 */
    __be32  sb_features_ro_compat;  /* 只读兼容特性 */
    __be32  sb_features_incompat;   /* 不兼容特性 */
    __be32  sb_features_log_incompat; /* 日志不兼容特性 */
    __be32  sb_crc;                 /* CRC 校验和 */
    __be32  sb_spino_align;         /* 稀疏 inode 对齐 */
    __be64  sb_pquotino;            /* 项目配额 inode */
    __be64  sb_lsn;                 /* 日志序列号 */
    __be64  sb_meta_uuid;           /* 元数据 UUID */
};
```

---

## 三、关键字段详解

### 3.1 基本参数

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `sb_magicnum` | __be32 | 魔数，固定为 0x58465342 ("XFSB") | 0x58465342 |
| `sb_blocksize` | __be32 | 文件系统块大小（字节） | 4096 |
| `sb_dblocks` | __be64 | 数据块总数 | - |
| `sb_sectsize` | __be16 | 扇区大小（字节） | 512 |

### 3.2 AG 相关参数

| 字段 | 类型 | 说明 | 限制 |
|------|------|------|------|
| `sb_agblocks` | __be32 | 每个 AG 的块数 | 最小 1024，最大 2^32-1 |
| `sb_agcount` | __be32 | AG 数量 | 最小 1 |
| `sb_agblklog` | 计算 | AG 块号的位数 | log2(sb_agblocks) |

### 3.3 Inode 相关参数

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `sb_inodesize` | __be16 | inode 大小（字节） | 256 或 512 |
| `sb_inopblock` | __be16 | 每块 inode 数 | blocksize / inodesize |
| `sb_inoalignmt` | __be32 | inode chunk 对齐（块数） | 通常 1 |

### 3.4 日志相关参数

| 字段 | 类型 | 说明 |
|------|------|------|
| `sb_logstart` | __be64 | 日志起始块号（内部日志） |
| `sb_logblocks` | __be32 | 日志块数 |

### 3.5 特殊 Inode

| 字段 | 说明 | 典型值 |
|------|------|--------|
| `sb_rootino` | 根目录 inode 号 | 128 |
| `sb_rbmino` | 实时位图 inode 号 | - |
| `sb_rsumino` | 实时汇总 inode 号 | - |
| `sb_pquotino` | 项目配额 inode 号 | - |

---

## 四、版本号与特性

### 4.1 版本号 (sb_versionnum)

```c
#define XFS_SB_VERSION_1        0   /* v1 格式 */
#define XFS_SB_VERSION_2        1   /* v2 格式 */
#define XFS_SB_VERSION_3        2   /* v3 格式 */
#define XFS_SB_VERSION_4        3   /* v4 格式 */
#define XFS_SB_VERSION_5        4   /* v5 格式 (当前) */

/* 版本号标志位 */
#define XFS_SB_VERSION_ATTRBIT          (1 << 0)  /* 扩展属性 */
#define XFS_SB_VERSION_NLINKBIT         (1 << 1)  /* 32位链接计数 */
#define XFS_SB_VERSION_QUOTABIT         (1 << 2)  /* 配额 */
#define XFS_SB_VERSION_ALIGNBIT         (1 << 3)  /* inode 对齐 */
#define XFS_SB_VERSION_DALIGNBIT        (1 << 4)  /* 数据对齐 */
#define XFS_SB_VERSION_SHAREDBIT        (1 << 5)  /* 共享版本 */
#define XFS_SB_VERSION_LOGV2BIT         (1 << 6)  /* 日志 v2 */
#define XFS_SB_VERSION_SECTORBIT        (1 << 7)  /* 扇区大小 > 512 */
#define XFS_SB_VERSION_EXTFLGBIT        (1 << 8)  /* extent 标志 */
#define XFS_SB_VERSION_DIRV2BIT         (1 << 9)  /* 目录 v2 */
#define XFS_SB_VERSION_BORGBIT          (1 << 10) /* ASCII CI 目录 */
#define XFS_SB_VERSION_MOREBITSBIT      (1 << 11) /* 更多特性位 */
```

### 4.2 兼容特性 (sb_features_compat)

```c
#define XFS_SB_FEAT_COMPAT_ATTR1        (1 << 0)  /* 属性 v1 */
#define XFS_SB_FEAT_COMPAT_ATTR2        (1 << 1)  /* 属性 v2 */
```

### 4.3 只读兼容特性 (sb_features_ro_compat)

```c
#define XFS_SB_FEAT_RO_COMPAT_FINOBT    (1 << 0)  /* Free inode B+tree */
#define XFS_SB_FEAT_RO_COMPAT_RMAPBT    (1 << 1)  /* 反向映射 B+tree */
#define XFS_SB_FEAT_RO_COMPAT_REFLINK   (1 << 2)  /* Reflink */
#define XFS_SB_FEAT_RO_COMPAT_INOBTCNT  (1 << 3)  /* Inode B+tree 计数 */
```

### 4.4 不兼容特性 (sb_features_incompat)

```c
#define XFS_SB_FEAT_INCOMPAT_FTYPE      (1 << 0)  /* 目录项包含文件类型 */
#define XFS_SB_FEAT_INCOMPAT_SPINODES   (1 << 1)  /* 稀疏 inode */
#define XFS_SB_FEAT_INCOMPAT_META_UUID  (1 << 2)  /* 元数据 UUID */
#define XFS_SB_FEAT_INCOMPAT_BIGTIME    (1 << 3)  /* 大时间戳 */
#define XFS_SB_FEAT_INCOMPAT_NEEDSREPAIR (1 << 4) /* 需要修复 */
#define XFS_SB_FEAT_INCOMPAT_NREXT64    (1 << 5)  /* 64位 extent 数 */
```

### 4.5 日志不兼容特性 (sb_features_log_incompat)

```c
#define XFS_SB_FEAT_INCOMPAT_LOG_XLOGSUPPORT    (1 << 0)  /* XLog 支持 */
#define XFS_SB_FEAT_INCOMPAT_LOG_BIGTIME        (1 << 1)  /* 大时间戳日志 */
```

---

## 五、磁盘布局示例

### 5.1 使用 xxd 查看 Super Block

```bash
xxd -l 512 -g 1 /dev/sdX
```

### 5.2 预期输出

```
00000000: 58 46 53 42 00 00 10 00 00 00 00 00 00 28 00 00  XFSB.........(..
          ^^^^^^^^^^^^ ^^^^^^^^^^
          魔数 XFSB    块大小 0x1000=4096

00000010: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000020: 1e bc 4d bc 7d 4f 4a 23 ab eb cf c8 eb 4a c1 6c  ..M.}OJ#.....J.l
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
          UUID

00000030: 00 00 00 00 00 20 00 06 00 00 00 00 00 00 00 80  ..... ..........
          ^^^^^^^^^^ ^^^^^^^^^^
          AG块数      AG数量

00000040: 00 00 00 00 00 00 00 81 00 00 00 00 00 00 00 82  ................
          ^^^^^^^^^^ ^^^^^^^^^^
          日志起始    根目录inode

00000050: 00 00 00 01 00 0a 00 00 00 00 00 04 00 00 00 00  ................
          ^^^^^^^^^^ ^^^^^^^^^^
          版本号     扇区大小

00000060: 00 00 0a 00 b4 a5 02 00 02 00 00 08 00 00 00 00  ................
          ^^^^^^^^^^ ^^^^^^^^^^
          inode大小  每块inode数
```

---

## 六、字段偏移表

| 偏移 | 大小 | 字段 | 说明 |
|------|------|------|------|
| 0x00 | 4 | sb_magicnum | 魔数 |
| 0x04 | 4 | sb_blocksize | 块大小 |
| 0x08 | 8 | sb_dblocks | 数据块总数 |
| 0x10 | 8 | sb_rblocks | 实时设备块数 |
| 0x18 | 8 | sb_rextents | 实时 extent 数 |
| 0x20 | 16 | sb_uuid | UUID |
| 0x30 | 8 | sb_logstart | 日志起始块号 |
| 0x38 | 8 | sb_rootino | 根目录 inode 号 |
| 0x40 | 8 | sb_rbmino | 实时位图 inode 号 |
| 0x48 | 8 | sb_rsumino | 实时汇总 inode 号 |
| 0x50 | 4 | sb_rextsize | 实时 extent 大小 |
| 0x54 | 4 | sb_agblocks | 每个 AG 的块数 |
| 0x58 | 4 | sb_agcount | AG 数量 |
| 0x5C | 4 | sb_rbmblocks | 实时位图块数 |
| 0x60 | 4 | sb_logblocks | 日志块数 |
| 0x64 | 2 | sb_versionnum | 版本号 |
| 0x66 | 2 | sb_sectsize | 扇区大小 |
| 0x68 | 2 | sb_inodesize | inode 大小 |
| 0x6A | 2 | sb_inopblock | 每块 inode 数 |

---

## 七、mkfs 选项与 Super Block

### 7.1 常用 mkfs 选项

```bash
# 基本格式化
mkfs.xfs /dev/sdX

# 指定块大小
mkfs.xfs -b size=4096 /dev/sdX

# 指定 AG 数量
mkfs.xfs -d agcount=4 /dev/sdX

# 指定 inode 大小
mkfs.xfs -i size=512 /dev/sdX

# 启用 reflink 和 rmapbt
mkfs.xfs -m reflink=1,rmapbt=1 /dev/sdX

# 指定日志大小
mkfs.xfs -l size=256m /dev/sdX

# 完整示例
mkfs.xfs -f -b size=4096 -d agcount=4 -i size=512 -m reflink=1,rmapbt=1 /dev/sdX
```

### 7.2 mkfs 输出解读

```
meta-data=/dev/sda1              isize=512    agcount=4, agsize=655360 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0, reflink=1
         ^                       ^            ^
         |                       |            |
    设备路径               inode大小      AG数量和大小
                                    扇区大小      特性标志

data     =                       bsize=4096   blocks=2621440, imaxpct=25
         =                       sunit=0      swidth=0 blks
         ^
         |
     块大小和数据块总数

naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
         ^
         |
     目录版本和文件类型支持

log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
         ^
         |
     日志类型和大小

realtime =none                   extsz=4096   blocks=0, rtextents=0
```

---

## 八、调试命令

### 8.1 xfs_db 查看 Super Block

```bash
xfs_db /dev/sdX

# 查看 AG0 的 Super Block
xfs_db> sb 0
xfs_db> p

# 输出示例
magicnum = 0x58465342
blocksize = 4096
dblocks = 2621440
rblocks = 0
rextents = 0
uuid = 1ebc4dbc-7d4f-4a23-abeb-cfc8eb4ac16c
logstart = 129
rootino = 128
rbmino = 129
rsumino = 130
agblocks = 655360
agcount = 4
rbmblocks = 0
logblocks = 2560
versionnum = 0xb4a5
sectsize = 512
inodesize = 512
inopblock = 8
```

### 8.2 xfs_info 查看文件系统信息

```bash
xfs_info /mount/point
xfs_info /dev/sdX
```

### 8.3 查看 Super Block 特性

```bash
xfs_db> sb 0
xfs_db> p versionnum
versionnum = 0xb4a5

xfs_db> p features_incompat
features_incompat = 0x3

xfs_db> p features_ro_compat
features_ro_compat = 0x5
```

---

## 九、代码位置

| 功能 | 文件路径 |
|------|----------|
| 结构定义 | `fs/xfs/libxfs/xfs_format.h` |
| Super Block 操作 | `fs/xfs/libxfs/xfs_sb.c` |
| Super Block 验证 | `fs/xfs/libxfs/xfs_sb.c` |
| mkfs 实现 | `mkfs/xfs_mkfs.c` (xfsprogs) |

---

## 十、参考资源

- XFS 官方文档: `git://git.kernel.org/pub/scm/fs/xfs/xfs-documentation.git`
- XFS 内核代码: `fs/xfs/libxfs/xfs_sb.c`
- xfsprogs 用户工具: `mkfs/xfs_mkfs.c`
