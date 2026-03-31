---
name: "xfs-attrfork"
description: "XFS文件系统扩展属性(Attribute Fork)专家。当用户询问XFS扩展属性、属性fork结构、本地/远程属性、属性B+树时调用此技能。"
---

# XFS Attribute Fork (扩展属性)

## 一、概述

### 1.1 什么是 Attribute Fork

Attribute Fork（属性fork）是XFS inode中用于存储扩展属性的数据结构。与Data Fork（数据fork）存储文件内容不同，Attribute Fork存储文件的元数据属性。

```
+----------------------------+
|        XFS Inode          |
+----------------------------+
|  Core inode fields        |
|  (128字节)                |
+----------------------------+
|  Data Fork                |
|  (文件内容)               |
+----------------------------+
|  Attribute Fork           |
|  (扩展属性)               |
+----------------------------+
```

### 1.2 属性类型

| 属性类型 | 说明 |
|----------|------|
| 本地属性 | 属性值直接存储在inode中 |
| 远程属性 | 属性值存储在外部块中，inode中只存储指针 |
| 命名空间 | user, trusted, security, system |

---

## 二、Attribute Fork 结构

### 2.1 Inode中的表示

**位置**: `fs/xfs/xfs_inode.h`

```c
struct xfs_dinode {
    __be16      di_core.di_aformat;    /* 属性fork格式 */
    __be32      di_core.di_anextents;  /* 属性extent数量 */
    __be32      di_core.di_anumber;    /* 属性inode数量 */
    __be32      di_u.di_a[3];          /* 属性fork位置 */
};
```

### 2.2 属性fork格式

| 格式值 | 格式名称 | 说明 |
|--------|----------|------|
| 0 | XFS_DINODE_FMT_EXTENTS | Extent列表格式 |
| 1 | XFS_DINODE_FMT_BTREE | B+树格式 |
| 2 | XFS_DINODE_FMT_LOCAL | 本地存储格式 |
| 3 | XFS_DINODE_FMT_DEV | 设备文件格式 |

---

## 三、属性存储方式

### 3.1 本地属性 (Local Attributes)

当属性值较小时，直接存储在inode中：

```
+------------------------------+
| Inode (256字节)              |
+------------------------------+
| Core fields (128字节)        |
+------------------------------+
| Data Fork pointers           |
+------------------------------+
| Attribute Fork:              |
|  - 属性头 (xfs_attr_sf_hdr)  |
|  - 属性1: 名称+值            |
|  - 属性2: 名称+值            |
|  - ...                       |
+------------------------------+
```

### 3.2 远程属性 (Remote Attributes)

当属性值较大时，存储在外部数据块中：

```
+------------------------------+
| Inode                        |
+------------------------------+
| Attribute Fork:              |
|  - 属性头 (xfs_attr_leaf_hdr)|
|  - B+树根节点                |
+------------------------------+
       ↓
+------------------------------+
| 属性数据块                   |
+------------------------------+
| 属性值数据                   |
+------------------------------+
```

### 3.3 属性B+树结构

对于大量属性，使用B+树进行索引：

```
               [根节点]
                  |
           +------+------+
           |             |
        [内部节点]    [内部节点]
           |             |
      +----+----+   +----+----+
      |         |   |         |
   [叶节点]  [叶节点] [叶节点] [叶节点]
      |         |   |         |
   属性条目   属性条目 属性条目 属性条目
```

---

## 四、属性操作

### 4.1 属性设置流程

```
1. 检查属性大小
   ↓
2. 小属性 → 本地存储
   ↓
3. 大属性 → 检查现有空间
   ↓
4. 空间不足 → 分配新块
   ↓
5. 更新B+树索引
   ↓
6. 写入属性值
```

### 4.2 属性读取流程

```
1. 查找属性名称
   ↓
2. 本地属性 → 直接从inode读取
   ↓
3. 远程属性 → 通过B+树查找
   ↓
4. 读取属性值块
   ↓
5. 返回属性值
```

### 4.3 属性删除流程

```
1. 查找属性位置
   ↓
2. 本地属性 → 从inode删除
   ↓
3. 远程属性 → 释放数据块
   ↓
4. 更新B+树
   ↓
5. 可能触发B+树收缩
```

---

## 五、内核源码参考

### 5.1 主要数据结构

- `xfs_attr_sf_hdr`: 本地属性头
- `xfs_attr_leaf_hdr`: 叶节点属性头
- `xfs_attr_leaf_entry`: 属性条目
- `xfs_attr_leaf_name_local`: 本地属性名称
- `xfs_attr_leaf_name_remote`: 远程属性名称

### 5.2 关键函数

- `xfs_attr_get()`: 获取属性
- `xfs_attr_set()`: 设置属性
- `xfs_attr_remove()`: 删除属性
- `xfs_attr_list()`: 列出属性

### 5.3 文件位置

- `fs/xfs/xfs_attr.h`: 属性数据结构定义
- `fs/xfs/xfs_attr_leaf.c`: 属性叶节点操作
- `fs/xfs/xfs_attr_remote.c`: 远程属性操作
- `fs/xfs/xfs_attr_list.c`: 属性列表操作