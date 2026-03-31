---
name: "xfs-runtime-create"
description: "XFS文件系统运行时文件创建专家。当用户询问XFS文件创建流程、inode分配、目录项添加、事务管理时调用此技能。"
---

# XFS Runtime File Creation (运行时文件创建)

## 一、概述

### 1.1 文件创建流程

XFS文件创建是一个复杂的过程，涉及多个子系统协同工作：

```
用户空间: creat("file", mode) 或 open("file", O_CREAT)
    ↓
VFS层: vfs_create()
    ↓
XFS层: xfs_create()
    ↓
事务开始 → inode分配 → 目录项添加 → 事务提交
```

### 1.2 创建类型

| 创建类型 | 系统调用 | XFS函数 |
|----------|----------|----------|
| 普通文件 | creat(), open(O_CREAT) | xfs_create() |
| 目录 | mkdir() | xfs_mkdir() |
| 符号链接 | symlink() | xfs_symlink() |
| 设备文件 | mknod() | xfs_mknod() |
| 命名管道 | mkfifo() | xfs_mknod() |

### 1.3 创建阶段

```
阶段1: 准备阶段
  - 权限检查
  - 名称验证
  - 父目录检查

阶段2: 资源分配
  - inode分配
  - 磁盘空间预留
  - 事务准备

阶段3: 元数据创建
  - inode初始化
  - 目录项添加
  - 时间戳设置

阶段4: 提交阶段
  - 事务提交
  - 日志写入
  - 缓存更新
```

---

## 二、事务管理

### 2.1 事务生命周期

XFS使用事务确保操作原子性：

```
xfs_trans_alloc()      ← 开始事务
    ↓
xfs_trans_reserve()    ← 预留日志空间
    ↓
各种操作              ← inode分配、目录项添加等
    ↓
xfs_trans_commit()     ← 提交事务
    ↓
xfs_log_commit()       ← 写入日志
```

### 2.2 事务预留

创建文件需要预留多种资源：

```c
struct xfs_trans_res {
    uint    tr_logres;   /* 日志空间 (字节) */
    uint    tr_logcount; /* 日志记录数 */
    uint    tr_logflags; /* 日志标志 */
};

/* 文件创建所需预留 */
static const struct xfs_trans_res create_resv = {
    .tr_logres  = XFS_CREATE_LOG_RES(mp),  /* 约1500字节 */
    .tr_logcount= XFS_CREATE_LOG_COUNT,    /* 2个记录 */
    .tr_logflags= XFS_TRANS_PERM_LOG_RES,  /* 永久日志 */
};
```

### 2.3 事务内容

文件创建事务包含以下操作：

1. **inode分配**: 从AGI B+树分配inode
2. **inode初始化**: 设置inode字段
3. **目录项添加**: 父目录添加新条目
4. **链接计数更新**: 父目录nlink+1
5. **时间戳更新**: atime, mtime, ctime
6. **空间预留**: 预分配初始空间（可选）

---

## 三、inode分配

### 3.1 inode分配策略

XFS使用智能的inode分配策略：

| 策略 | 描述 | 适用场景 |
|------|------|----------|
| 局部性分配 | inode靠近父目录 | 目录操作频繁 |
| 负载均衡 | 分散到多个AG | 高并发创建 |
| 预留分配 | 预分配inode块 | 批量创建 |
| 延迟初始化 | 延迟inode写入 | 性能优化 |

### 3.2 inode分配流程

```
1. 选择目标AG
   - 父目录所在AG（局部性）
   - 空闲inode最多的AG
   ↓
2. 查找空闲inode
   - 查询AGI中的空闲inodeB+树
   - 或扫描inode块位图
   ↓
3. 分配inode号
   - 标记inode为已使用
   - 更新AGI统计
   ↓
4. 初始化inode
   - 设置模式、uid、gid
   - 初始化时间戳
   - 设置链接计数=1
```

### 3.3 inode结构初始化

```c
void xfs_setup_inode(
    struct xfs_inode *ip)  /* 新inode */
{
    /* 基础字段 */
    ip->i_d.di_mode = mode;      /* 文件模式 */
    ip->i_d.di_uid = uid;        /* 用户ID */
    ip->i_d.di_gid = gid;        /* 组ID */
    ip->i_d.di_nlink = 1;        /* 链接计数 */
    
    /* 时间戳 */
    ip->i_d.di_atime = current_time;
    ip->i_d.di_mtime = current_time;
    ip->i_d.di_ctime = current_time;
    
    /* 大小相关 */
    ip->i_d.di_size = 0;         /* 初始大小 */
    ip->i_d.di_nextents = 0;     /* 初始extent数 */
    ip->i_d.di_anextents = 0;    /* 属性extent数 */
    
    /* 格式 */
    ip->i_d.di_format = XFS_DINODE_FMT_EXTENTS;
    ip->i_d.di_aformat = XFS_DINODE_FMT_EXTENTS;
}
```

### 3.4 inode缓存管理

新分配的inode加入缓存：

```
inode分配 → 加入VFS inode缓存 → 加入XFS inode缓存
                                      ↓
                               定期刷写或内存压力时
                                      ↓
                               写入磁盘
```

---

## 四、目录项操作

### 4.1 目录项添加流程

向父目录添加新文件的目录项：

```
1. 查找插入位置
   - 计算文件名哈希值
   - 在目录B+树中查找位置
   ↓
2. 检查重名
   - 如果已存在同名文件，返回EEXIST
   ↓
3. 准备目录项数据
   struct xfs_dir2_data_entry {
       __be64  inumber;      /* inode号 */
       __u8    namelen;      /* 名称长度 */
       __u8    name[];       /* 文件名 */
   };
   ↓
4. 插入目录项
   - 短格式目录：直接插入
   - B+树目录：通过B+树插入
   ↓
5. 更新目录元数据
   - 目录大小增加
   - 时间戳更新
```

### 4.2 目录格式转换

当目录增长时自动转换格式：

```
短格式 (sf) → 块格式 (block) → B+树格式 (btree)
    ↓              ↓                  ↓
 小目录        中等目录           大目录
(<几十项)    (<几百项)          (>几百项)
```

转换触发条件：
1. 插入导致目录项超过阈值
2. 目录操作性能下降
3. 显式转换请求

### 4.3 目录项优化

XFS优化目录项存储：

| 优化 | 描述 | 收益 |
|------|------|------|
| 名称压缩 | 短名称直接存储 | 节省空间 |
| 哈希优化 | 优化哈希分布 | 减少冲突 |
| 空间回收 | 重用删除空间 | 减少碎片 |
| 预分配 | 预分配目录块 | 减少分裂 |

---

## 五、扩展属性创建

### 5.1 扩展属性初始化

创建文件时可设置初始扩展属性：

```c
int xfs_attr_set(
    struct xfs_inode *ip,
    const char       *name,   /* 属性名 */
    const void       *value,  /* 属性值 */
    int              valuelen,/* 值长度 */
    int              flags)   /* 标志位 */
{
    /* 1. 权限检查 */
    if (!xfs_attr_namecheck(name, valuelen))
        return -EINVAL;
    
    /* 2. 选择存储方式 */
    if (valuelen <= XFS_ATTR_SF_MAX_SIZE)
        return xfs_attr_shortform_add(ip, name, value, valuelen);
    else
        return xfs_attr_leaf_add(ip, name, value, valuelen);
}
```

### 5.2 属性存储选择

根据属性大小选择存储方式：

| 存储方式 | 大小限制 | 位置 |
|----------|----------|------|
| 短格式 | ≤ 70字节 | inode内 |
| 叶节点 | ≤ 块大小 | 单独块 |
| B+树节点 | 无限制 | B+树 |

### 5.3 默认属性

新文件可能有的默认属性：

| 属性名 | 命名空间 | 描述 |
|--------|----------|------|
| security.selinux | security | SELinux上下文 |
| system.posix_acl_default | system | 默认ACL |
| user.creator | user | 创建者信息 |
| trusted.glusterfs | trusted | GlusterFS标记 |

---

## 六、错误处理与回滚

### 6.1 创建失败场景

| 失败原因 | 错误码 | 处理方式 |
|----------|--------|----------|
| 权限不足 | EACCES | 中止创建 |
| 磁盘空间不足 | ENOSPC | 回滚分配 |
| inode耗尽 | ENOSPC | 回滚事务 |
| 名称无效 | EINVAL | 提前返回 |
| 重名文件 | EEXIST | 中止创建 |

### 6.2 事务回滚机制

创建过程中发生错误时回滚：

```
创建失败
    ↓
xfs_trans_cancel()  ← 取消事务
    ↓
释放已分配资源:
1. 释放inode (如果已分配)
2. 释放预留空间
3. 删除目录项 (如果已添加)
    ↓
清理缓存:
1. 从inode缓存移除
2. 清理脏页面
    ↓
返回错误码
```

### 6.3 原子性保证

XFS确保文件创建的原子性：

```
要么: 全部成功
  - inode分配成功
  - 目录项添加成功
  - 所有元数据更新成功

要么: 全部失败
  - 无残留inode
  - 无残留目录项
  - 文件系统状态一致
```

---

## 七、性能优化

### 7.1 批量创建优化

批量创建文件时的优化：

| 优化技术 | 描述 | 收益 |
|----------|------|------|
| 事务合并 | 多个创建合并为单个事务 | 减少日志写入 |
| inode预分配 | 预分配inode块 | 减少分配开销 |
| 目录预扩展 | 预分配目录空间 | 减少格式转换 |
| 延迟提交 | 延迟事务提交 | 合并更多操作 |

### 7.2 缓存优化

利用缓存提高创建性能：

1. **inode缓存**: 缓存常用inode
2. **目录缓存**: 缓存目录项
3. **扩展属性缓存**: 缓存常用属性
4. **事务缓存**: 重用事务结构

### 7.3 并发创建处理

多进程并发创建时的处理：

```c
/* 目录项插入时的锁处理 */
xfs_ilock(dp, XFS_ILOCK_EXCL);   /* 排它锁父目录 */
    /* 检查重名 */
    /* 插入目录项 */
xfs_iunlock(dp, XFS_ILOCK_EXCL);
```

锁策略：
- 父目录: 排它锁（插入时）
- 新inode: 排它锁（初始化时）
- AG: 读锁（inode分配时）

---

## 八、内核源码参考

### 8.1 主要数据结构

- `xfs_icreate_args`: inode创建参数
- `xfs_dir2_args`: 目录操作参数
- `xfs_trans_t`: 事务结构
- `xfs_buf_log_item_t`: 缓冲区日志项
- `xfs_inode_log_item_t`: inode日志项

### 8.2 关键函数

- `xfs_create()`: 文件创建主函数
- `xfs_dir_createname()`: 创建目录项
- `xfs_dir_cilookup()`: 目录查找
- `xfs_get_dir_entry()`: 获取目录项
- `xfs_setup_inode()`: 初始化inode
- `xfs_ialloc()`: inode分配

### 8.3 文件位置

- `fs/xfs/xfs_inode.c`: inode操作
- `fs/xfs/xfs_dir2.c`: 目录操作
- `fs/xfs/xfs_trans.c`: 事务管理
- `fs/xfs/xfs_ialloc.c`: inode分配
- `fs/xfs/xfs_symlink.c`: 符号链接创建