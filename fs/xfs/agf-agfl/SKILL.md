---
name: "xfs-agf-agfl"
description: "XFS文件系统AGF和AGFL专家。当用户询问XFS AGF结构、AGFL结构、空闲空间管理、agf_freeblks、agf_longest时调用此技能。"
---

# XFS AGF & AGFL

## 一、AGF (AG Free Space Info)

### 1.1 概述

AGF (Allocation Group Free Space Info) 是每个 AG 的第二个扇区，用于管理 AG 内的空闲空间信息。

**位置**: 每个 AG 的 Sector 1 (偏移 512 字节)
**大小**: 512 字节
**魔数**: 0x58414746 ("XAGF")

### 1.2 结构定义

**位置**: `fs/xfs/libxfs/xfs_format.h`

```c
struct xfs_agf {
    __be32  agf_magicnum;           /* 魔数 XAGF */
    __be32  agf_versionnum;         /* 版本号 */
    __be32  agf_seqno;              /* AG 序号 */
    __be32  agf_length;             /* AG 长度 (块数) */
    __be32  agf_roots[XFS_BTNUM_AGF]; /* B+树根块号 */
    __be32  agf_spare0;             /* 保留 */
    __be32  agf_levels[XFS_BTNUM_AGF]; /* B+树深度 */
    __be32  agf_flfirst;            /* AGFL 第一个空闲块索引 */
    __be32  agf_fllast;             /* AGFL 最后一个空闲块索引 */
    __be32  agf_flcount;            /* AGFL 空闲块计数 */
    __be32  agf_freeblks;           /* 空闲块总数 */
    __be32  agf_longest;            /* 最大连续空闲块数 */
    __be32  agf_btreeblks;          /* B+树使用的块数 */
    uuid_t  agf_uuid;               /* UUID */
    __be32  agf_rmap_blocks;        /* RMAP B+树块数 */
    __be32  agf_refcount_blocks;    /* 引用计数 B+树块数 */
    __be32  agf_refcount_root;      /* 引用计数 B+树根 */
    __be32  agf_refcount_level;     /* 引用计数 B+树深度 */
    __be32  agf_spare2[23];         /* 保留 */
    __be32  agf_crc;                /* CRC 校验和 */
};
```

### 1.3 关键字段详解

| 字段 | 类型 | 说明 |
|------|------|------|
| `agf_magicnum` | __be32 | 魔数，固定为 0x58414746 ("XAGF") |
| `agf_seqno` | __be32 | AG 序号，从 0 开始 |
| `agf_length` | __be32 | AG 的块数 |
| `agf_roots[0]` | __be32 | BNO B+树根块号 |
| `agf_roots[1]` | __be32 | CNT B+树根块号 |
| `agf_levels[0]` | __be32 | BNO B+树深度 |
| `agf_levels[1]` | __be32 | CNT B+树深度 |
| `agf_flfirst` | __be32 | AGFL 第一个空闲块索引 |
| `agf_fllast` | __be32 | AGFL 最后一个空闲块索引 |
| `agf_flcount` | __be32 | AGFL 中空闲块数量 |
| `agf_freeblks` | __be32 | AG 中空闲块总数 |
| `agf_longest` | __be32 | 最大连续空闲块数 |
| `agf_btreeblks` | __be32 | B+树使用的块数 |

### 1.4 B+树索引常量

```c
#define XFS_BTNUM_AGF   2       /* AGF 中的 B+树数量 */

/* agf_roots 和 agf_levels 数组索引 */
#define XFS_BTNUM_BNO   0       /* BNO B+tree (按块号索引) */
#define XFS_BTNUM_CNT   1       /* CNT B+tree (按块数索引) */
```

---

## 二、AGFL (AG Free List)

### 2.1 概述

AGFL (Allocation Group Free List) 是每个 AG 的第四个扇区，存储预留的空闲块号列表。

**位置**: 每个 AG 的 Sector 3 (偏移 1536 字节)
**大小**: 512 字节
**魔数**: 0x5841464C ("XAFL")

### 2.2 结构定义

```c
struct xfs_agfl {
    __be32  agfl_magicnum;      /* 魔数 XAFL */
    __be32  agfl_seqno;         /* AG 序号 */
    uuid_t  agfl_uuid;          /* UUID */
    __be64  agfl_lsn;           /* 日志序列号 */
    __be32  agfl_crc;           /* CRC 校验和 */
    __be32  agfl_bno[];         /* 空闲块号数组 */
};
```

### 2.3 AGFL 的作用

| 用途 | 说明 |
|------|------|
| B+树分裂 | 当 B+树分裂时需要分配新块，从 AGFL 获取 |
| 死锁避免 | 确保空间分配操作不会死锁 |
| 预留空间 | 维护一定数量的预留块 |

### 2.4 AGFL 管理

AGFL 是一个循环队列，由 AGF 中的三个字段管理：

```
agf_flfirst ──┐
              │
              ▼
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │  <- AGFL 数组索引
├───┼───┼───┼───┼───┼───┼───┼───┤
│ A │ B │ C │ D │ E │   │   │   │  <- 空闲块号
└───┴───┴───┴───┴───┴───┴───┴───┘
                              ▲
                              │
                     agf_fllast ──┘

agf_flcount = 5 (A, B, C, D, E)
```

---

## 三、双 B+树设计

### 3.1 为什么需要两个 B+树

XFS 使用两个 B+树来管理空闲空间：

| B+树 | 索引键 | 用途 |
|------|--------|------|
| BNO (bnobt) | 起始块号 | 快速查找特定位置的空闲块 |
| CNT (cntbt) | 块数量 | 快速查找指定大小的空闲块 |

### 3.2 分配策略

```
场景1: 需要分配 N 个连续块
       → 使用 CNT B+树，按大小查找
       → 找到 >= N 的最小空闲块

场景2: 合并相邻空闲块
       → 使用 BNO B+树，按位置查找
       → 快速定位相邻块

场景3: 查找特定位置的空闲块
       → 使用 BNO B+树
       → 按起始块号查找
```

---

## 四、磁盘布局示例

### 4.1 使用 xxd 查看 AGF

```bash
xxd -s 512 -l 512 -g 1 /dev/sdX
```

### 4.2 预期输出

```
地址                         内容                                字符表示
00000200: 58 41 47 46 00 00 00 01 00 00 00 00 00 0a 00 00  XAGF............
          ^^^^^^^^^^^^ ^^^^^^^^^^ ^^^^^^^^^^ ^^^^^^^^^^
          魔数 XAGF    版本号     AG序号=1   AG长度

00000210: 00 00 00 01 00 00 00 02 00 00 00 00 00 00 00 01  ................
          ^^^^^^^^^^ ^^^^^^^^^^
          BNO根块=1  CNT根块=2

00000220: 00 00 00 01 00 00 00 00 00 00 00 01 00 00 00 04  ................
          ^^^^^^^^^^ ^^^^^^^^^^
          BNO深度=1  CNT深度=1

00000230: 00 00 00 04 00 09 ff ee 00 09 ff e8 00 00 00 00  ................
          ^^^^^^^^^^ ^^^^^^^^^^ ^^^^^^^^^^
          flfirst=4  fllast     flcount

00000240: 1e bc 4d bc 7d 4f 4a 23 ab eb cf c8 eb 4a c1 6c  ..M.}OJ#.....J.l
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
          UUID
```

### 4.3 使用 xxd 查看 AGFL

```bash
xxd -s 1536 -l 512 -g 1 /dev/sdX
```

### 4.4 预期输出

```
地址                         内容                                字符表示
00000600: 58 41 46 4c 00 00 00 00 1e bc 4d bc 7d 4f 4a 23  XAFL......M.}OJ#
          ^^^^^^^^^^^^ ^^^^^^^^^^
          魔数 XAFL    AG序号=0

00000610: ab eb cf c8 eb 4a c1 6c 00 00 00 00 00 00 00 00  .....J.l........
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
          UUID

00000620: aa f0 2b dd ff ff ff ff 00 00 00 06 00 00 00 07  ..+.............
          ^^^^^^^^^^             ^^^^^^^^^^ ^^^^^^^^^^
          LSN                    块号6     块号7

00000630: 00 00 00 08 00 00 00 09 ff ff ff ff ff ff ff ff  ................
          ^^^^^^^^^^ ^^^^^^^^^^
          块号8     块号9
```

---

## 五、字段偏移表

### 5.1 AGF 字段偏移

| 偏移 | 大小 | 字段 | 说明 |
|------|------|------|------|
| 0x00 | 4 | agf_magicnum | 魔数 |
| 0x04 | 4 | agf_versionnum | 版本号 |
| 0x08 | 4 | agf_seqno | AG 序号 |
| 0x0C | 4 | agf_length | AG 长度 |
| 0x10 | 4 | agf_roots[0] | BNO 根块号 |
| 0x14 | 4 | agf_roots[1] | CNT 根块号 |
| 0x18 | 4 | agf_spare0 | 保留 |
| 0x1C | 4 | agf_levels[0] | BNO 深度 |
| 0x20 | 4 | agf_levels[1] | CNT 深度 |
| 0x24 | 4 | agf_flfirst | AGFL 首索引 |
| 0x28 | 4 | agf_fllast | AGFL 尾索引 |
| 0x2C | 4 | agf_flcount | AGFL 计数 |
| 0x30 | 4 | agf_freeblks | 空闲块总数 |
| 0x34 | 4 | agf_longest | 最大连续空闲块 |
| 0x38 | 4 | agf_btreeblks | B+树块数 |

---

## 六、调试命令

### 6.1 xfs_db 查看 AGF

```bash
xfs_db /dev/sdX

# 查看 AG0 的 AGF
xfs_db> agf 0
xfs_db> p

# 输出示例
magicnum = 0x58414746
versionnum = 1
seqno = 0
length = 655360
bno_root = 1
cnt_root = 2
bno_level = 1
cnt_level = 1
flfirst = 4
fllast = 4
flcount = 1
freeblks = 655358
longest = 655358
btreeblks = 2
```

### 6.2 xfs_db 查看 AGFL

```bash
xfs_db> agfl 0
xfs_db> p

# 输出示例
magicnum = 0x5841464c
seqno = 0
uuid = 1ebc4dbc-7d4f-4a23-abeb-cfc8eb4ac16c
lsn = 0
bno[0] = 6
bno[1] = 7
bno[2] = 8
bno[3] = 9
```

### 6.3 查看 B+树根块

```bash
# 查看 BNO B+tree 根块
xfs_db> addr agf_bno_root
xfs_db> p

# 查看 CNT B+tree 根块
xfs_db> addr agf_cnt_root
xfs_db> p
```

---

## 七、代码位置

| 功能 | 文件路径 |
|------|----------|
| 结构定义 | `fs/xfs/libxfs/xfs_format.h` |
| AGF 操作 | `fs/xfs/libxfs/xfs_alloc.c` |
| AGF 初始化 | `fs/xfs/libxfs/xfs_ag.c` |
| 空间分配 | `fs/xfs/libxfs/xfs_alloc.c` |

---

## 八、参考资源

- XFS 官方文档: `git://git.kernel.org/pub/scm/fs/xfs/xfs-documentation.git`
- XFS 内核代码: `fs/xfs/libxfs/xfs_alloc.c`
