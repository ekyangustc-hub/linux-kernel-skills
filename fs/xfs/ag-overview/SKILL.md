---
name: "xfs-ag-overview"
description: "XFS文件系统AG概述专家。当用户询问XFS整体架构、AG概念、AG布局、AG内部结构时调用此技能。"
---

# XFS AG (Allocation Group) 概述

## 一、AG 核心概念

### 1.1 什么是 AG

XFS 将整个文件系统划分为多个独立的分配组 (Allocation Group, AG)，每个 AG 可以理解为一个小型的独立文件系统。

```
|<--------------------            XFS              ----------------------->|
+--------------+--------------+--------------+--------------+--------------+
|     AG-0     |     AG-1     |     AG-2     |     ....     |     AG-N     |
+--------------+--------------+--------------+--------------+--------------+
```

### 1.2 AG 的优势

| 特性 | 说明 |
|------|------|
| 独立管理 | 每个 AG 独立管理自己的空间和 inode |
| 并行操作 | 多个 AG 可以并行操作，充分利用多核 CPU |
| 可扩展性 | 提高并发性能和可扩展性 |
| 容错性 | 单个 AG 损坏不影响其他 AG |

### 1.3 AG 配置参数

| 参数 | 默认值 | 限制 |
|------|--------|------|
| AG 数量 | 4 | 最少 1，最多取决于文件系统大小 |
| AG 大小 | 自动计算 | 最小 16MB，最大 1TB |
| AG 块数 | 自动计算 | 最多 2^32 - 1 块 |

---

## 二、AG 内部结构

### 2.1 AG 头部结构

每个 AG 的头部包含以下结构（按顺序）：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AG 结构                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Sector 0:   ┌─────────────────────────────────────────────────────────┐   │
│              │                    Super Block (SB)                      │   │
│              │                    512 bytes                              │   │
│              │                    魔数: XFSB                              │   │
│              └─────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Sector 1:   ┌─────────────────────────────────────────────────────────┐   │
│              │                    AGF (AG Free space)                    │   │
│              │                    512 bytes                              │   │
│              │                    魔数: XAGF                              │   │
│              └─────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Sector 2:   ┌─────────────────────────────────────────────────────────┐   │
│              │                    AGI (AG Inode info)                    │   │
│              │                    512 bytes                              │   │
│              │                    魔数: XAGI                              │   │
│              └─────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Sector 3:   ┌─────────────────────────────────────────────────────────┐   │
│              │                    AGFL (AG Free List)                    │   │
│              │                    512 bytes                              │   │
│              │                    魔数: XAFL                              │   │
│              └─────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Block 1:    ┌─────────────────────────────────────────────────────────┐   │
│              │              BNO B+tree Root (按块号索引)                 │   │
│              │                    4096 bytes                            │   │
│              │                    魔数: AB3B                              │   │
│              └─────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Block 2:    ┌─────────────────────────────────────────────────────────┐   │
│              │              CNT B+tree Root (按块数索引)                 │   │
│              │                    4096 bytes                            │   │
│              │                    魔数: AB3C                              │   │
│              └─────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Block 3:    ┌─────────────────────────────────────────────────────────┐   │
│              │              INO B+tree Root (inode 索引)                 │   │
│              │                    4096 bytes                            │   │
│              │                    魔数: IAB3                              │   │
│              └─────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Block 4:    ┌─────────────────────────────────────────────────────────┐   │
│  (可选)      │              FINO B+tree Root (free inode 索引)           │   │
│              │                    4096 bytes                            │   │
│              │                    魔数: FIB3                              │   │
│              └─────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Block 5:    ┌─────────────────────────────────────────────────────────┐   │
│  (可选)      │              RMAP B+tree Root (反向映射)                  │   │
│              │                    4096 bytes                            │   │
│              │                    魔数: RMAP                              │   │
│              └─────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Block 6:    ┌─────────────────────────────────────────────────────────┐   │
│  (可选)      │              REFC B+tree Root (引用计数)                  │   │
│              │                    4096 bytes                            │   │
│              │                    魔数: R3FC                              │   │
│              └─────────────────────────────────────────────────────────┘   │
│                                                                             │
│              ┌─────────────────────────────────────────────────────────┐   │
│              │              Inode Chunks + Data Blocks                  │   │
│              │                    ...                                   │   │
│              └─────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 AG 头部初始化代码

**位置**: `fs/xfs/libxfs/xfs_ag.c`

```c
int
xfs_ag_init_headers(
    struct xfs_mount        *mp,
    struct aghdr_init_data  *id)
{
    struct xfs_aghdr_grow_data aghdr_data[] = {
        { /* SB */
            .daddr = XFS_AG_DADDR(mp, id->agno, XFS_SB_DADDR),
            .numblks = XFS_FSS_TO_BB(mp, 1),
            .need_init = true
        },
        { /* AGF */
            .daddr = XFS_AG_DADDR(mp, id->agno, XFS_AGF_DADDR(mp)),
            .numblks = XFS_FSS_TO_BB(mp, 1),
            .need_init = true
        },
        { /* AGFL */
            .daddr = XFS_AG_DADDR(mp, id->agno, XFS_AGFL_DADDR(mp)),
            .numblks = XFS_FSS_TO_BB(mp, 1),
            .need_init = true
        },
        { /* AGI */
            .daddr = XFS_AG_DADDR(mp, id->agno, XFS_AGI_DADDR(mp)),
            .numblks = XFS_FSS_TO_BB(mp, 1),
            .need_init = true
        },
        { /* BNO root block */
            .daddr = XFS_AGB_TO_DADDR(mp, id->agno, XFS_BNO_BLOCK(mp)),
            .numblks = BTOBB(mp->m_sb.sb_blocksize),
            .need_init = true
        },
        { /* CNT root block */
            .daddr = XFS_AGB_TO_DADDR(mp, id->agno, XFS_CNT_BLOCK(mp)),
            .numblks = BTOBB(mp->m_sb.sb_blocksize),
            .need_init = true
        },
        { /* INO root block */
            .daddr = XFS_AGB_TO_DADDR(mp, id->agno, XFS_IBT_BLOCK(mp)),
            .numblks = BTOBB(mp->m_sb.sb_blocksize),
            .need_init = true
        },
        { /* FINO root block */
            .daddr = XFS_AGB_TO_DADDR(mp, id->agno, XFS_FIBT_BLOCK(mp)),
            .numblks = BTOBB(mp->m_sb.sb_blocksize),
            .need_init = xfs_sb_version_hasfinobt(&mp->m_sb)
        },
        { /* RMAP root block */
            .daddr = XFS_AGB_TO_DADDR(mp, id->agno, XFS_RMAP_BLOCK(mp)),
            .numblks = BTOBB(mp->m_sb.sb_blocksize),
            .need_init = xfs_sb_version_hasrmapbt(&mp->m_sb)
        },
        { /* REFC root block */
            .daddr = XFS_AGB_TO_DADDR(mp, id->agno, xfs_refc_block(mp)),
            .numblks = BTOBB(mp->m_sb.sb_blocksize),
            .need_init = xfs_sb_version_hasreflink(&mp->m_sb)
        },
        { /* NULL terminating block */
            .daddr = XFS_BUF_DADDR_NULL,
        }
    };
    /* ... */
}
```

---

## 三、魔数标识

### 3.1 AG 头部魔数

| 结构 | Magic Number | 十六进制 | ASCII |
|------|-------------|----------|-------|
| Super Block | 0x58465342 | XFSB | "XFSB" |
| AGF | 0x58414746 | XAGF | "XAGF" |
| AGI | 0x58414749 | XAGI | "XAGI" |
| AGFL | 0x5841464C | XAFL | "XAFL" |

### 3.2 B+树根块魔数

| B+树 | Magic Number | 十六进制 | ASCII |
|------|-------------|----------|-------|
| BNO B+tree | 0x41423342 | AB3B | "AB3B" |
| CNT B+tree | 0x41423343 | AB3C | "AB3C" |
| INO B+tree | 0x49414233 | IAB3 | "IAB3" |
| FINO B+tree | 0x46494233 | FIB3 | "FIB3" |
| RMAP B+tree | 0x524D4150 | RMAP | "RMAP" |
| REFC B+tree | 0x52334643 | R3FC | "R3FC" |

---

## 四、磁盘布局验证

### 4.1 使用 xxd 查看原始数据

```bash
# 创建 XFS 文件系统
mkfs.xfs -f /dev/sdX

# 查看 AG0 的原始数据
xxd -l $((655360*4096)) -g 1 /dev/sdX | head -100
```

### 4.2 预期输出

```
地址                         内容                                字符表示
00000000: 58 46 53 42 00 00 10 00 00 00 00 00 00 28 00 00  XFSB.........(..
          ^^^^^^^^^^^^
          Super Block 魔数 XFSB

00000200: 58 41 47 46 00 00 00 01 00 00 00 00 00 0a 00 00  XAGF............
          ^^^^^^^^^^^^
          AGF 魔数 XAGF

00000400: 58 41 47 49 00 00 00 01 00 00 00 00 00 0a 00 00  XAGI............
          ^^^^^^^^^^^^
          AGI 魔数 XAGI

00000600: 58 41 46 4c 00 00 00 00 1e bc 4d bc 7d 4f 4a 23  XAFL......M.}OJ#
          ^^^^^^^^^^^^
          AGFL 魔数 XAFL

00001000: 41 42 33 42 00 00 00 02 ff ff ff ff ff ff ff ff  AB3B............
          ^^^^^^^^^^^^
          BNO B+tree 魔数 AB3B

00002000: 41 42 33 43 00 00 00 02 ff ff ff ff ff ff ff ff  AB3C............
          ^^^^^^^^^^^^
          CNT B+tree 魔数 AB3C

00003000: 49 41 42 33 00 00 00 01 ff ff ff ff ff ff ff ff  IAB3............
          ^^^^^^^^^^^^
          INO B+tree 魔数 IAB3
```

---

## 五、AG 相关宏定义

### 5.1 地址计算宏

**位置**: `fs/xfs/libxfs/xfs_format.h`

```c
/* AG 号转换为 AG 内块号 */
#define XFS_AGB_TO_AGNO(mp, agbno) \
    ((agbno) >> (mp)->m_agblklog)

#define XFS_AGB_TO_AGBNO(mp, agbno) \
    ((agbno) & ((mp)->m_agblklog - 1))

/* AG 内块号转换为磁盘地址 */
#define XFS_AGB_TO_DADDR(mp, agno, agbno) \
    (((xfs_daddr_t)(agno) << (mp)->m_agblklog) + (agbno)) << (mp)->m_blkbb_log

/* 磁盘地址转换为 AG 号和块号 */
#define XFS_DADDR_TO_AGNO(mp, daddr) \
    ((xfs_agnumber_t)((daddr) >> (mp)->m_agblklog >> (mp)->m_blkbb_log))

#define XFS_DADDR_TO_AGBNO(mp, daddr) \
    ((xfs_agblock_t)((daddr) >> (mp)->m_blkbb_log) & ((mp)->m_agblklog - 1))
```

### 5.2 AG 头部块号定义

```c
#define XFS_SB_DADDR(mp)        0   /* Super Block 在 AG 内的扇区号 */
#define XFS_AGF_DADDR(mp)       1   /* AGF 在 AG 内的扇区号 */
#define XFS_AGI_DADDR(mp)       2   /* AGI 在 AG 内的扇区号 */
#define XFS_AGFL_DADDR(mp)      3   /* AGFL 在 AG 内的扇区号 */

#define XFS_BNO_BLOCK(mp)       1   /* BNO B+tree 根块号 */
#define XFS_CNT_BLOCK(mp)       2   /* CNT B+tree 根块号 */
#define XFS_IBT_BLOCK(mp)       3   /* INO B+tree 根块号 */
#define XFS_FIBT_BLOCK(mp)      4   /* FINO B+tree 根块号 */
#define XFS_RMAP_BLOCK(mp)      5   /* RMAP B+tree 根块号 */
```

---

## 六、调试命令

### 6.1 xfs_db 常用命令

```bash
# 打开设备
xfs_db /dev/sdX

# 查看 Super Block
xfs_db> sb 0
xfs_db> p

# 查看 AGF
xfs_db> agf 0
xfs_db> p

# 查看 AGI
xfs_db> agi 0
xfs_db> p

# 查看 AGFL
xfs_db> agfl 0
xfs_db> p

# 切换到不同的 AG
xfs_db> sb 1    # 查看 AG1 的 Super Block
xfs_db> agf 1   # 查看 AG1 的 AGF
```

### 6.2 xfs_info 命令

```bash
# 查看文件系统信息
xfs_info /mount/point

# 输出示例
meta-data=/dev/sda1              isize=512    agcount=4, agsize=655360 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0, reflink=1
data     =                       bsize=4096   blocks=2621440, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

---

## 七、代码位置

| 功能 | 文件路径 |
|------|----------|
| AG 初始化 | `fs/xfs/libxfs/xfs_ag.c` |
| 格式定义 | `fs/xfs/libxfs/xfs_format.h` |
| AG 头部操作 | `fs/xfs/libxfs/xfs_sb.c` |
| AGF 操作 | `fs/xfs/libxfs/xfs_alloc.c` |
| AGI 操作 | `fs/xfs/libxfs/xfs_ialloc.c` |

---

## 八、参考资源

- XFS 官方文档: `git://git.kernel.org/pub/scm/fs/xfs/xfs-documentation.git`
- XFS 内核代码: `fs/xfs/`
- xfsprogs 用户工具: `git://git.kernel.org/pub/scm/fs/xfs/xfsprogs-dev.git`
