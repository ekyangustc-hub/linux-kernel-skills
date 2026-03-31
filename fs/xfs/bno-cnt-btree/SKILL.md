---
name: "xfs-bno-cnt-btree"
description: "XFS文件系统BNO/CNT B+树专家。当用户询问XFS空闲空间B+树、BNO B+tree、CNT B+tree、xfs_alloc_rec时调用此技能。"
---

# XFS BNO/CNT B+tree (空闲空间 B+树)

## 一、概述

XFS 使用两个 B+树来管理 AG 内的空闲空间：
- **BNO B+tree**: 按起始块号排序，用于按位置查找空闲块
- **CNT B+tree**: 按块数量排序，用于按大小查找空闲块

---

## 二、空闲空间记录

### 2.1 记录结构

**位置**: `fs/xfs/libxfs/xfs_format.h`

```c
struct xfs_alloc_rec {
    __be32  ar_startblock;      /* 起始块号 */
    __be32  ar_blockcount;      /* 块数量 */
};
```

### 2.2 记录含义

```
ar_startblock = 100
ar_blockcount = 50

表示：从块号 100 开始，有 50 个连续空闲块 (100-149)
```

### 2.3 键的比较

| B+树 | 排序键 | 比较方式 |
|------|--------|----------|
| BNO | ar_startblock | 按起始块号升序 |
| CNT | ar_blockcount, ar_startblock | 先按块数升序，再按块号升序 |

---

## 三、B+树块结构

### 3.1 块头结构 (V5)

```c
struct xfs_btree_block {
    __be32  bb_magic;            /* 魔数 */
    __be16  bb_level;            /* 层级 (0=叶子) */
    __be16  bb_numrecs;          /* 记录数 */
    __be32  bb_leftsib;          /* 左兄弟块号 */
    __be32  bb_rightsib;         /* 右兄弟块号 */
    __be64  bb_blkno;            /* 当前块号 */
    __be64  bb_lsn;              /* 日志序列号 */
    uuid_t  bb_uuid;             /* UUID */
    __be32  bb_owner;            /* 所有者 (AG 号) */
    __be32  bb_crc;              /* CRC 校验和 */
    __be32  bb_pad;              /* 填充 */
};
```

### 3.2 魔数

| B+树 | 魔数 | 十六进制 | ASCII |
|------|------|----------|-------|
| BNO (V5) | 0x41423342 | AB3B | "AB3B" |
| CNT (V5) | 0x41423343 | AB3C | "AB3C" |
| BNO (V4) | 0x41425442 | ABTB | "ABTB" |
| CNT (V4) | 0x41425443 | ABTC | "ABTC" |

---

## 四、B+树布局

### 4.1 叶子节点

```
┌─────────────────────────────────────────────────────────────────┐
│                      B+tree Leaf Block                          │
├─────────────────────────────────────────────────────────────────┤
│ bb_magic = AB3B/AB3C                                            │
│ bb_level = 0                                                    │
│ bb_numrecs = N                                                  │
│ bb_leftsib = 左兄弟块号                                          │
│ bb_rightsib = 右兄弟块号                                         │
├─────────────────────────────────────────────────────────────────┤
│ rec[0]: ar_startblock=100, ar_blockcount=50                     │
│ rec[1]: ar_startblock=200, ar_blockcount=30                     │
│ rec[2]: ar_startblock=300, ar_blockcount=100                    │
│ ...                                                             │
│ rec[N-1]: ar_startblock=XXX, ar_blockcount=YYY                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 内部节点

```
┌─────────────────────────────────────────────────────────────────┐
│                      B+tree Internal Block                      │
├─────────────────────────────────────────────────────────────────┤
│ bb_magic = AB3B/AB3C                                            │
│ bb_level = 1                                                    │
│ bb_numrecs = N                                                  │
├─────────────────────────────────────────────────────────────────┤
│ key[0]: ar_startblock=100 (或 ar_blockcount)                    │
│ key[1]: ar_startblock=200                                       │
│ ...                                                             │
│ key[N-1]: ar_startblock=XXX                                     │
├─────────────────────────────────────────────────────────────────┤
│ ptr[0]: 子节点块号 (包含 < key[0] 的记录)                         │
│ ptr[1]: 子节点块号 (包含 key[0] <= x < key[1] 的记录)             │
│ ...                                                             │
│ ptr[N]: 子节点块号 (包含 >= key[N-1] 的记录)                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、查找操作

### 5.1 BNO B+tree 查找

```c
/* 查找特定块号附近的空闲空间 */
int xfs_alloc_lookup_eq(
    struct xfs_btree_cur  *cur,
    xfs_agblock_t         bno,
    xfs_extlen_t          len,
    int                   *stat);
```

### 5.2 CNT B+tree 查找

```c
/* 查找大于等于指定大小的空闲空间 */
int xfs_alloc_lookup_ge(
    struct xfs_btree_cur  *cur,
    xfs_agblock_t         bno,
    xfs_extlen_t          len,
    int                   *stat);
```

---

## 六、分配策略

### 6.1 最佳匹配 (Best Fit)

```c
/* 使用 CNT B+tree 查找最小满足需求的空闲块 */
xfs_alloc_lookup_ge(cnt_cur, 0, minlen, &stat);
```

### 6.2 最近匹配 (Near Best Fit)

```c
/* 使用 BNO B+tree 查找指定位置附近的空闲块 */
xfs_alloc_lookup_le(bno_cur, near_bno, 0, &stat);
```

### 6.3 分配类型

```c
#define XFS_ALLOCTYPE_FIRST_AG  0x01  /* 从第一个 AG 开始 */
#define XFS_ALLOCTYPE_START_AG  0x02  /* 从指定 AG 开始 */
#define XFS_ALLOCTYPE_THIS_AG   0x04  /* 在指定 AG 内 */
#define XFS_ALLOCTYPE_START_BNO 0x08  /* 从指定块号开始 */
#define XFS_ALLOCTYPE_NEAR_BNO  0x10  /* 在指定块号附近 */
#define XFS_ALLOCTYPE_THIS_BNO  0x20  /* 在指定块号 */
```

---

## 七、磁盘布局示例

### 7.1 查看根块

```bash
xfs_db /dev/sdX

# 查看 BNO 根块
xfs_db> addr agf_bno_root
xfs_db> p

# 输出示例
magic = 0x41423342
level = 1
numrecs = 10
leftsib = null
rightsib = null
bno[0-1] = [startblock,count] [0,100] [200,50] ...
```

### 7.2 使用 xxd 查看

```bash
# BNO 根块在 Block 1 (偏移 4096)
xxd -s 4096 -l 4096 -g 1 /dev/sdX | head -50

# 预期输出
00001000: 41 42 33 42 00 00 00 02 ff ff ff ff ff ff ff ff  AB3B............
          ^^^^^^^^^^^^ ^^^^^^^^^^ ^^^^^^^^^^
          魔数 AB3B    level=2    numrecs

00001010: ff ff ff ff ff ff ff ff 00 00 00 00 00 00 00 00  ................
          ^^^^^^^^^^ ^^^^^^^^^^
          leftsib    rightsib

00001020: 1e bc 4d bc 7d 4f 4a 23 ab eb cf c8 eb 4a c1 6c  ..M.}OJ#.....J.l
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
          UUID
```

---

## 八、B+树操作

### 8.1 插入记录

```c
/* 插入空闲空间记录 */
int xfs_alloc_insert(
    struct xfs_btree_cur  *cur,
    int                   *stat);
```

### 8.2 删除记录

```c
/* 删除空闲空间记录 */
int xfs_alloc_delete(
    struct xfs_btree_cur  *cur,
    int                   *stat);
```

### 8.3 更新记录

```c
/* 更新空闲空间记录 */
int xfs_alloc_update(
    struct xfs_btree_cur  *cur,
    xfs_agblock_t         bno,
    xfs_extlen_t          len);
```

---

## 九、关键函数

### 9.1 分配函数

**位置**: `fs/xfs/libxfs/xfs_alloc.c`

```c
/* 分配空闲块 */
int xfs_alloc_vextent(
    struct xfs_alloc_arg  *args);

/* 分配空闲块 (精确位置) */
int xfs_alloc_exact_bno(
    struct xfs_alloc_arg  *args);

/* 分配空闲块 (附近位置) */
int xfs_alloc_near_bno(
    struct xfs_alloc_arg  *args);
```

### 9.2 释放函数

```c
/* 释放块 */
int xfs_free_extent(
    struct xfs_trans      *tp,
    struct xfs_perag      *pag,
    xfs_agblock_t         bno,
    xfs_extlen_t          len,
    const struct xfs_owner_info *oinfo,
    enum xfs_ag_resv_type type);
```

---

## 十、调试命令

### 10.1 xfs_db 命令

```bash
xfs_db /dev/sdX

# 查看 AGF
xfs_db> agf 0
xfs_db> p

# 查看 BNO 根块
xfs_db> addr agf_bno_root
xfs_db> p

# 查看 CNT 根块
xfs_db> addr agf_cnt_root
xfs_db> p

# 遍历 B+树
xfs_db> addr agf_bno_root
xfs_db> ptrs
xfs_db> addr ptrs[1]
xfs_db> p
```

### 10.2 查看空闲空间统计

```bash
xfs_db> agf 0
xfs_db> p freeblks
xfs_db> p longest
```

---

## 十一、代码位置

| 功能 | 文件路径 |
|------|----------|
| 结构定义 | `fs/xfs/libxfs/xfs_format.h` |
| B+tree 操作 | `fs/xfs/libxfs/xfs_alloc_btree.c` |
| 空间分配 | `fs/xfs/libxfs/xfs_alloc.c` |
| B+tree 通用 | `fs/xfs/libxfs/xfs_btree.c` |

---

## 十二、参考资源

- XFS 官方文档: `git://git.kernel.org/pub/scm/fs/xfs/xfs-documentation.git`
- XFS 内核代码: `fs/xfs/libxfs/xfs_alloc_btree.c`
