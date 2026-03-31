---
name: "xfs-datafork-regular"
description: "XFS文件系统普通文件Datafork专家。当用户询问XFS普通文件extent、xfs_bmbt_rec、B+tree映射、文件数据布局时调用此技能。"
---

# XFS 普通文件 Datafork

## 一、概述

XFS 普通文件的数据 fork 存储文件数据的块映射，支持三种格式：
1. **本地存储**: 小文件直接存储在 inode 中
2. **Extent 列表**: 中等大小的文件使用 extent 列表
3. **B+tree**: 大文件使用 B+tree 存储 extent 映射

---

## 二、Datafork 格式

### 2.1 格式定义

```c
/* inode.di_format 字段 */
#define XFS_DINODE_FMT_DEV      0   /* 设备文件 */
#define XFS_DINODE_FMT_LOCAL    1   /* 本地存储 (≤156字节) */
#define XFS_DINODE_FMT_EXTENTS  2   /* Extent 列表 */
#define XFS_DINODE_FMT_BTREE    3   /* B+tree */

/* inode.di_nextents 字段 */
- 格式为 EXTENTS: di_nextents = extent 数
- 格式为 BTREE: di_nextents = B+tree 使用的块数
```

### 2.2 本地存储格式 (FMT_LOCAL)

```
小文件直接存储在 inode 的数据 fork 中：

┌─────────────────────────────────────────────────────────────────┐
│                        Inode (512B)                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ xfs_dinode 核心结构 (176B)                                   ││
│  ├─────────────────────────────────────────────────────────────┤│
│  │ 数据 fork (336B)                                             ││
│  │  ┌─────────────────────────────────────────────────────────┐││
│  │  │ 文件数据 (最多 156字节)                                    │││
│  │  └─────────────────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘

最大大小: 512 - 176 = 336 字节
减去可能的属性 fork: 更小
```

### 2.3 Extent 列表格式 (FMT_EXTENTS)

```
中等大小的文件使用 extent 列表：

┌─────────────────────────────────────────────────────────────────┐
│                        Inode (512B)                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ xfs_dinode 核心结构 (176B)                                   ││
│  ├─────────────────────────────────────────────────────────────┤│
│  │ 数据 fork (extent 列表)                                       ││
│  │  ┌──────────┬──────────┬──────────┬──────────┐              ││
│  │  │ extent 0 │ extent 1 │ extent 2 │ ...      │              ││
│  │  └──────────┴──────────┴──────────┴──────────┘              ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘

每个 extent: 16字节
最大 extent 数: (512-176)/16 = 21 个
```

### 2.4 B+tree 格式 (FMT_BTREE)

```
大文件使用 B+tree 存储 extent 映射：

┌─────────────────────────────────────────────────────────────────┐
│                        Inode (512B)                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ xfs_dinode 核心结构 (176B)                                   ││
│  ├─────────────────────────────────────────────────────────────┤│
│  │ 数据 fork (B+tree 根节点)                                     ││
│  │  ┌─────────────────────────────────────────────────────────┐││
│  │  │ xfs_bmdr_block (B+tree 根)                               │││
│  │  │   bb_numrecs = 2                                        │││
│  │  │   keys[0] = [逻辑块0]                                    │││
│  │  │   ptrs[0] = 子块号                                       │││
│  │  └─────────────────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、Extent 结构

### 3.1 Extent 记录

**位置**: `fs/xfs/libxfs/xfs_format.h:1174-1194`

```c
struct xfs_bmbt_rec {
    __be64  l0;
    __be64  l1;
};

/* 解码 extent 记录 */
static inline xfs_fileoff_t xfs_bmbt_rec_startoff(struct xfs_bmbt_rec *r)
{
    return ((xfs_fileoff_t)be64_to_cpu(r->l0) &
            xfs_mask64lo(64 - BMBT_EXNTFLAG_BITLEN)) >> 9;
}

static inline xfs_fsblock_t xfs_bmbt_rec_startblock(struct xfs_bmbt_rec *r)
{
    return (((xfs_fsblock_t)be64_to_cpu(r->l0) &
             xfs_mask64lo(9)) << 43) |
           (((xfs_fsblock_t)be64_to_cpu(r->l1)) >> 21);
}

static inline xfs_filblks_t xfs_bmbt_rec_blockcount(struct xfs_bmbt_rec *r)
{
    return be64_to_cpu(r->l1) & xfs_mask64lo(21);
}

static inline int xfs_bmbt_rec_state(struct xfs_bmbt_rec *r)
{
    return be64_to_cpu(r->l0) >> (64 - BMBT_EXNTFLAG_BITLEN);
}
```

### 3.2 Extent 标志

```c
#define BMBT_EXNTFLAG_BITLEN     1   /* 标志位长度 */
#define BMBT_EXNTFLAG_UNWRITTEN  1   /* 未初始化 extent */

/* 已初始化 vs 未初始化 */
#define XFS_EXT_NORM             0   /* 已初始化 */
#define XFS_EXT_UNWRITTEN        1   /* 未初始化 */
```

---

## 四、B+tree 结构

### 4.1 B+tree 节点

**位置**: `fs/xfs/libxfs/xfs_format.h:1196-1246`

```c
/* 磁盘上的 B+tree 节点 */
struct xfs_btree_block {
    __be32  bb_magic;          /* 魔数: 0x424D4150 ('BMAP') */
    __be16  bb_level;          /* 节点层级 */
    __be16  bb_numrecs;        /* 记录数 */
    __be64  bb_leftsib;        /* 左兄弟块号 */
    __be64  bb_rightsib;       /* 右兄弟块号 */
};

/* 内部节点的键和指针 */
struct xfs_bmbt_key {
    __be64  br_startoff;       /* 起始逻辑块号 */
};

typedef __be64 xfs_bmbt_ptr_t; /* 子节点块号 */

/* 内存中的 B+tree 游标 */
struct xfs_btree_cur {
    struct xfs_mount    *bc_mp;
    union {
        struct xfs_btree_cur_ag ag;
        struct xfs_btree_cur_ino ino;
    } bc_private;
    unsigned int        bc_btnum;  /* B+tree 类型 */
    int                 bc_nlevels; /* 树高度 */
    struct xfs_btree_ops *bc_ops;
};
```

### 4.2 B+tree 布局

```
                    ┌─────────────────────────────┐
                    │     根节点 (在 inode 中)     │
                    │     di_u.bmdr               │
                    │     level = 2               │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
        ┌───────────┐        ┌───────────┐        ┌───────────┐
        │ 中间节点   │        │ 中间节点   │        │ 中间节点   │
        │ key[0]:   │        │ key[0]:   │        │ key[0]:   │
        │ 0         │        │ 1000      │        │ 2000      │
        │ ptr[0]    │        │ ptr[0]    │        │ ptr[0]    │
        └─────┬─────┘        └─────┬─────┘        └─────┬─────┘
              │                    │                    │
              ▼                    ▼                    ▼
        ┌───────────┐        ┌───────────┐        ┌───────────┐
        │ 叶子节点   │        │ 叶子节点   │        │ 叶子节点   │
        │ rec[0]:   │        │ rec[0]:   │        │ rec[0]:   │
        │ [0,100,10]│        │[1000,200,5]│       │[2000,300,8]│
        │ rec[1]:   │        │ rec[1]:   │        │           │
        │[100,150,5]│        │[1200,180,3]│       │           │
        └───────────┘        └───────────┘        └───────────┘
```

---

## 五、文件布局示例

### 5.1 小文件 (本地存储)

```
文件大小: 100字节
inode.di_format = XFS_DINODE_FMT_LOCAL
数据直接存储在 inode 的数据 fork 中
不占用额外数据块
```

### 5.2 中等文件 (Extent 列表)

```
文件大小: 100KB (25个4K块)
inode.di_format = XFS_DINODE_FMT_EXTENTS
inode.di_nextents = 3

extent[0]: [逻辑块0, 物理块1000, 10块]  (40KB)
extent[1]: [逻辑块10, 物理块2000, 8块]   (32KB)
extent[2]: [逻辑块18, 物理块3000, 7块]   (28KB)
```

### 5.3 大文件 (B+tree)

```
文件大小: 10GB (2,621,440个4K块)
inode.di_format = XFS_DINODE_FMT_BTREE
inode.di_nextents = 5 (B+tree使用的块数)

B+tree 高度: 3
根节点在 inode 中
中间节点: 3个块
叶子节点: 100个块，每个存储约260个 extent
```

---

## 六、Extent 操作

### 6.1 查找 extent

```c
/* 查找逻辑块对应的 extent */
int xfs_bmapi_read(
    struct xfs_inode    *ip,
    xfs_fileoff_t       bno,
    xfs_filblks_t       len,
    struct xfs_bmbt_irec *mval,
    int                 *nmap,
    int                 flags)
{
    struct xfs_btree_cur    *cur;
    struct xfs_bmbt_rec     rec;
    int                     error;
    
    /* 获取 B+tree 游标 */
    cur = xfs_bmbt_init_cursor(ip->i_mount, NULL, ip, XFS_DATA_FORK);
    
    /* 查找第一个 >= bno 的 extent */
    cur->bc_rec.b.br_startoff = bno;
    error = xfs_btree_lookup_ge(cur, 0, &i);
    
    if (i == 1) {
        /* 获取 extent 记录 */
        error = xfs_bmbt_get_rec(cur, &rec, &i);
        
        /* 解码 extent */
        mval->br_startoff = xfs_bmbt_rec_startoff(&rec);
        mval->br_startblock = xfs_bmbt_rec_startblock(&rec);
        mval->br_blockcount = xfs_bmbt_rec_blockcount(&rec);
        mval->br_state = xfs_bmbt_rec_state(&rec);
    }
    
    xfs_btree_del_cursor(cur, error);
    return error;
}
```

### 6.2 插入 extent

```c
/* 插入新 extent */
int xfs_bmapi_write(
    struct xfs_trans    *tp,
    struct xfs_inode    *ip,
    xfs_fileoff_t       bno,
    xfs_filblks_t       len,
    int                 flags,
    xfs_fsblock_t       *firstblock,
    int                 total,
    struct xfs_bmbt_irec *mval,
    int                 *nmap)
{
    struct xfs_btree_cur    *cur;
    struct xfs_bmbt_irec    new;
    
    /* 初始化新 extent */
    new.br_startoff = bno;
    new.br_startblock = *firstblock;
    new.br_blockcount = len;
    new.br_state = (flags & XFS_BMAPI_PREALLOC) ?
                   XFS_EXT_UNWRITTEN : XFS_EXT_NORM;
    
    /* 获取游标 */
    cur = xfs_bmbt_init_cursor(ip->i_mount, tp, ip, XFS_DATA_FORK);
    
    /* 插入到 B+tree */
    error = xfs_bmbt_insert(cur, &new);
    
    /* 更新 inode 大小 */
    if (bno + len > ip->i_d.di_size)
        ip->i_d.di_size = bno + len;
    
    xfs_btree_del_cursor(cur, error);
    return error;
}
```

---

## 七、调试命令

### 7.1 xfs_db

```bash
# 查看文件的 extent 映射
xfs_db -c "inode 128" -c "bmap" /dev/sda1

# 输出示例
  0: [0..9] 10 blocks, startblock 1000 (in AG 0)
  1: [10..17] 8 blocks, startblock 2000 (in AG 0)
  2: [18..24] 7 blocks, startblock 3000 (in AG 0)

# 查看 B+tree 结构
xfs_db -c "inode 129" -c "addr u3.bmbt" -c "p" /dev/sda1

magic = 0x424d4150
level = 1
numrecs = 3
keys[1-3] = [startoff] 1:0 2:1000 3:2000
ptrs[1-3] = 1:100 2:101 3:102
```

---

## 八、代码位置

| 功能 | 文件路径 |
|------|----------|
| 结构定义 | `fs/xfs/libxfs/xfs_format.h:1174-1246` |
| Extent 操作 | `fs/xfs/libxfs/xfs_bmap.c` |
| B+tree 操作 | `fs/xfs/libxfs/xfs_bmap_btree.c` |
