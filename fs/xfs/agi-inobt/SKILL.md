---
name: "xfs-agi-inobt"
description: "XFS文件系统AGI和Inode B+树专家。当用户询问XFS AGI结构、inode B+tree、free inode B+tree、xfs_inobt_rec时调用此技能。"
---

# XFS AGI & Inode B+tree

## 一、AGI (AG Inode Info)

### 1.1 概述

AGI (Allocation Group Inode Info) 是每个 AG 的第三个扇区，用于管理 AG 内的 inode 分配信息。

**位置**: 每个 AG 的 Sector 2 (偏移 1024 字节)
**大小**: 512 字节
**魔数**: 0x58414749 ("XAGI")

### 1.2 结构定义

**位置**: `fs/xfs/libxfs/xfs_format.h`

```c
struct xfs_agi {
    __be32  agi_magicnum;           /* 魔数 XAGI */
    __be32  agi_versionnum;         /* 版本号 */
    __be32  agi_seqno;              /* AG 序号 */
    __be32  agi_length;             /* AG 长度 (块数) */
    __be32  agi_count;              /* 已分配 inode 数 */
    __be32  agi_root;               /* inode B+tree 根块号 */
    __be32  agi_level;              /* inode B+tree 深度 */
    __be32  agi_freecount;          /* 空闲 inode 数 */
    __be32  agi_newino;             /* 最近分配的 inode 号 */
    __be32  agi_dirino;             /* 最近分配的目录 inode 号 */
    __be32  agi_unlinked[64];       /* 删除链表头 */
    __be32  agi_uuid;               /* UUID */
    __be32  agi_crc;                /* CRC 校验和 */
    __be32  agi_pad32;              /* 填充 */
    __be64  agi_lsn;                /* 日志序列号 */
    __be32  agi_free_root;          /* free inode B+tree 根块号 */
    __be32  agi_free_level;         /* free inode B+tree 深度 */
};
```

### 1.3 关键字段详解

| 字段 | 类型 | 说明 |
|------|------|------|
| `agi_magicnum` | __be32 | 魔数，固定为 0x58414749 ("XAGI") |
| `agi_seqno` | __be32 | AG 序号，从 0 开始 |
| `agi_count` | __be32 | AG 中已分配的 inode 数 |
| `agi_root` | __be32 | inode B+tree 根块号 |
| `agi_level` | __be32 | inode B+tree 深度 |
| `agi_freecount` | __be32 | AG 中空闲 inode 数 |
| `agi_newino` | __be32 | 最近分配的 inode 号 |
| `agi_unlinked[64]` | __be32[64] | 已删除但未释放的 inode 链表头 |
| `agi_free_root` | __be32 | free inode B+tree 根块号 |
| `agi_free_level` | __be32 | free inode B+tree 深度 |

---

## 二、Inode 号编码

### 2.1 Inode 号结构

XFS inode 号是一个 64 位值，编码了 AG 号和 AG 内偏移：

```
  63           32 31             0
┌───────────────┬───────────────┐
│   AG 序号     │  AG 内 inode 号 │
│  (32 bits)    │   (32 bits)    │
└───────────────┴───────────────┘
```

### 2.2 编码/解码宏

```c
/* 从 inode 号提取 AG 号 */
#define XFS_INO_TO_AGNO(mp, ino) \
    ((xfs_agnumber_t)((ino) >> (mp)->m_agino_log))

/* 从 inode 号提取 AG 内 inode 号 */
#define XFS_INO_TO_AGINO(mp, ino) \
    ((xfs_agino_t)((ino) & ((1ULL << (mp)->m_agino_log) - 1)))

/* 组合 AG 号和 AG 内 inode 号 */
#define XFS_AGINO_TO_INO(mp, agno, agino) \
    (((xfs_ino_t)(agno) << (mp)->m_agino_log) | (agino))
```

### 2.3 Inode 号示例

```
inode 号 = 128
AG 号 = 0
AG 内 inode 号 = 128

inode 号 = 655488
AG 号 = 1
AG 内 inode 号 = 128
```

---

## 三、Inode B+tree

### 3.1 记录结构

```c
struct xfs_inobt_rec {
    __be32  ir_startino;       /* 起始 inode 号 */
    __be32  ir_freecount;      /* 空闲 inode 数 */
    __be64  ir_free;           /* 空闲 inode 位图 */
};
```

### 3.2 记录含义

```
ir_startino = 128      // 从 inode 128 开始
ir_freecount = 5       // 有 5 个空闲 inode
ir_free = 0x000000000000001F  // 位图：低 5 位为 1

表示：inode 128-132 是空闲的
```

### 3.3 Inode Chunk

每个 inode B+tree 记录管理一个 inode chunk：

```c
#define XFS_INODES_PER_CHUNK   64   /* 每个 chunk 64 个 inode */
```

```
Inode Chunk 布局：
┌─────────────────────────────────────────────────────────────┐
│ inode 128 │ inode 129 │ inode 130 │ ... │ inode 191         │
└─────────────────────────────────────────────────────────────┘
     ↑
ir_startino = 128
```

---

## 四、Free Inode B+tree (finobt)

### 4.1 概述

Free Inode B+tree 是一个可选的 B+树，只记录包含空闲 inode 的 chunk，用于加速空闲 inode 查找。

### 4.2 与 inobt 的区别

| 特性 | inobt | finobt |
|------|-------|--------|
| 记录内容 | 所有 inode chunk | 只有包含空闲 inode 的 chunk |
| 查找速度 | 需要遍历 | 直接定位 |
| 是否可选 | 必须 | 可选 |

### 4.3 启用 finobt

```bash
mkfs.xfs -m finobt=1 /dev/sdX
```

---

## 五、B+树结构

### 5.1 魔数

| B+树 | 魔数 | 十六进制 | ASCII |
|------|------|----------|-------|
| inobt (V5) | 0x49414233 | IAB3 | "IAB3" |
| finobt (V5) | 0x46494233 | FIB3 | "FIB3" |
| inobt (V4) | 0x49414254 | IABT | "IABT" |
| finobt (V4) | 0x46494254 | FIBT | "FIBT" |

### 5.2 叶子节点布局

```
┌─────────────────────────────────────────────────────────────────┐
│                      Inode B+tree Leaf Block                    │
├─────────────────────────────────────────────────────────────────┤
│ bb_magic = IAB3                                                 │
│ bb_level = 0                                                    │
│ bb_numrecs = N                                                  │
├─────────────────────────────────────────────────────────────────┤
│ rec[0]: ir_startino=128, ir_freecount=5, ir_free=0x1F           │
│ rec[1]: ir_startino=192, ir_freecount=10, ir_free=0x3FF         │
│ ...                                                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、Unlinked List

### 6.1 概述

`agi_unlinked[64]` 是一个哈希表，存储已删除但引用计数不为 0 的 inode 链表头。

### 6.2 用途

当文件被删除时：
1. 如果文件还有引用（如被进程打开），inode 加入 unlinked list
2. 当最后一个引用关闭时，inode 从 unlinked list 移除并释放

### 6.3 链表结构

```
agi_unlinked[hash] ──→ inode A ──→ inode B ──→ ... ──→ NULL
                         │           │
                         │           └─ di_next_unlinked
                         └─ di_next_unlinked
```

---

## 七、磁盘布局示例

### 7.1 使用 xxd 查看 AGI

```bash
xxd -s 1024 -l 512 -g 1 /dev/sdX
```

### 7.2 预期输出

```
地址                         内容                                字符表示
00000400: 58 41 47 49 00 00 00 01 00 00 00 00 00 0a 00 00  XAGI............
          ^^^^^^^^^^^^ ^^^^^^^^^^ ^^^^^^^^^^ ^^^^^^^^^^
          魔数 XAGI    版本号     AG序号     AG长度

00000410: 00 00 00 40 00 00 00 03 00 00 00 01 00 00 00 3d  ...@...........=
          ^^^^^^^^^^ ^^^^^^^^^^ ^^^^^^^^^^ ^^^^^^^^^^
          已分配数    root块号   level     空闲数

00000420: 00 00 00 80 ff ff ff ff ff ff ff ff ff ff ff ff  ................
          ^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
          newino     unlinked[0] = -1 (空)
```

---

## 八、调试命令

### 8.1 xfs_db 查看 AGI

```bash
xfs_db /dev/sdX

# 查看 AG0 的 AGI
xfs_db> agi 0
xfs_db> p

# 输出示例
magicnum = 0x58414749
versionnum = 1
seqno = 0
length = 655360
count = 64
root = 3
level = 1
freecount = 59
newino = 128
dirino = null
unlinked[0-63] = 0xffffffff
free_root = 4
free_level = 1
```

### 8.2 查看 inode B+tree

```bash
# 查看 inobt 根块
xfs_db> addr agi_root
xfs_db> p

# 查看 finobt 根块
xfs_db> addr agi_free_root
xfs_db> p
```

### 8.3 查看特定 inode

```bash
# 查看 inode 128
xfs_db> inode 128
xfs_db> p
```

---

## 九、代码位置

| 功能 | 文件路径 |
|------|----------|
| 结构定义 | `fs/xfs/libxfs/xfs_format.h` |
| AGI 操作 | `fs/xfs/libxfs/xfs_ialloc.c` |
| Inode B+tree | `fs/xfs/libxfs/xfs_ialloc_btree.c` |
| Inode 分配 | `fs/xfs/libxfs/xfs_ialloc.c` |

---

## 十、参考资源

- XFS 官方文档: `git://git.kernel.org/pub/scm/fs/xfs/xfs-documentation.git`
- XFS 内核代码: `fs/xfs/libxfs/xfs_ialloc.c`
