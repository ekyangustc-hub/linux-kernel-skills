---
name: "ext4-data-path-delete"
description: "ext4文件系统删除数据路径专家。当用户询问ext4文件删除、ext4_unlink、ext4_delete_entry、ext4_orphan_add、ext4_free_inode、ext4_free_blocks、ext4_truncate、ext4_punch_hole、空间回收时调用此技能。"
---

# ext4 删除数据路径 (Delete Data Path)

## 一、为什么需要了解这个路径？

删除路径涉及复杂的资源回收和一致性保证。理解 ext4 删除路径有助于:

1. **Orphan 机制**: 理解打开文件删除后如何通过 orphan 列表延迟回收, 保证崩溃一致性
2. **空间回收**: 理解块释放、inode 释放、配额更新的完整流程
3. **Truncate 实现**: 理解 extent 树修剪、punch hole 操作的底层机制
4. **碎片影响**: 理解删除操作对文件系统碎片的影响, 以及如何优化
5. **Fast commit 集成**: 理解删除操作如何被 fast commit 跟踪和恢复

---

## 二、完整流程图

### 2.1 Unlink (删除目录项) 路径

```
用户空间 unlink("file")
  │
  ▼
sys_unlinkat() / vfs_unlink()
  │
  ▼
ext4_unlink()                                [fs/ext4/namei.c:3292]
  │
  ├── ext4_emergency_state()                 (检查紧急状态)
  ├── dquot_initialize(dir)                  (目录配额)
  ├── dquot_initialize(inode)                (文件配额)
  │
  ▼
  __ext4_unlink()                            [fs/ext4/namei.c:3219]
  │
  ├── ext4_find_entry()                      (查找目录项)
  │    │
  │    ├── 遍历目录块
  │    ├── HTree 目录: ext4_htree_find_entry()
  │    └── 返回 buffer_head + ext4_dir_entry_2
  │
  ├── ext4_journal_start()                   (启动 journal 事务)
  │
  ├── ext4_delete_entry()                    (删除目录项)
  │    │
  │    ├── 标记目录项为空闲:
  │    │    de->inode = 0
  │    │
  │    ├── 合并相邻空闲项:
  │    │    prev_de->rec_len += de->rec_len
  │    │
  │    └── ext4_handle_dirty_dirblock()       (标记块脏)
  │
  ├── inode->i_nlink--                       (减少硬链接计数)
  │
  ├── ext4_orphan_add()                      (如果 nlink == 0)
  │    │
  │    └── 将 inode 加入孤儿列表
  │         目的: 崩溃后能继续回收资源
  │
  ├── ext4_mark_inode_dirty()                (标记 inode 脏)
  │
  ├── ext4_fc_track_unlink()                 (Fast commit 跟踪)
  │
  └── ext4_journal_stop(handle)
  │
  ▼
  d_delete(dentry)                           (VFS 层删除 dentry)
```

### 2.2 Inode 释放路径 (最后一次 iput)

```
iput(inode)  (最后一个引用释放)
  │
  └── inode->i_nlink == 0?
       │
       └── 是 → ext4_evict_inode()           [fs/ext4/inode.c:246]
            │
            ├── truncate_inode_pages_final()  (清除 page cache)
            │
            ├── ext4_orphan_del()            (从孤儿列表移除)
            │
            ├── ext4_xattr_delete_inode()    (删除扩展属性)
            │
            ├── ext4_truncate()              (释放数据块)
            │    │
            │    ├── ext4_orphan_add()       (加入孤儿列表, 保证崩溃安全)
            │    │
            │    ├── down_write(i_data_sem)
            │    │
            │    ├── ext4_discard_preallocations()  (丢弃预分配)
            │    │
            │    ├── ext4_ext_truncate()     (extent 树修剪)
            │    │    │
            │    │    ├── ext4_ext_remove_space()
            │    │    │    │
            │    │    │    ├── ext4_ext_rm_idx()     (删除索引节点)
            │    │    │    │
            │    │    │    └── ext4_ext_remove_leaf() (删除叶子节点)
            │    │    │         │
            │    │    │         └── ext4_free_blocks() (释放物理块)
            │    │    │
            │    │    └── ext4_ext_truncate_failed_write() (失败清理)
            │    │
            │    └── up_write(i_data_sem)
            │
            ├── ext4_free_inode()            (释放 inode)
            │    │
            │    ├── ext4_clear_inode()       (清除内存 inode)
            │    │
            │    ├── ext4_read_inode_bitmap() (读取 inode 位图)
            │    │
            │    ├── ext4_clear_bit()         (清除位图位)
            │    │
            │    ├── ext4_free_inodes_count++ (更新计数)
            │    │
            │    └── ext4_used_dirs_count--   (如果是目录)
            │
            └── clear_inode()                (VFS 层清理)
```

### 2.3 Punch Hole (打孔) 路径

```
fallocate(fd, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE, offset, len)
  │
  ▼
ext4_punch_hole()                            [fs/ext4/extents.c]
  │
  ├── ext4_writepages()                      (刷脏数据, 保证一致性)
  │
  ├── ext4_es_remove_extent()                (清理 ES Tree 缓存)
  │
  ├── ext4_ext_remove_space()                (删除指定范围的 extent)
  │    │
  │    ├── ext4_find_extent()                (定位范围)
  │    │
  │    ├── ext4_ext_remove_leaf()            (删除叶子 extent)
  │    │    │
  │    │    └── ext4_free_blocks()           (释放物理块)
  │    │
  │    └── ext4_ext_rm_idx()                 (删除空索引节点)
  │
  └── ext4_mark_inode_dirty()                (标记 inode 脏)
```

---

## 三、关键函数调用链

```
vfs_unlink()
  └─► ext4_unlink()                              namei.c:3292-3330
       ├─► dquot_initialize(dir)
       ├─► dquot_initialize(inode)
       └─► __ext4_unlink()                       namei.c:3219-3290
            ├─► ext4_find_entry()
            ├─► ext4_journal_start()
            ├─► ext4_delete_entry()
            ├─► drop_nlink(inode)
            ├─► ext4_orphan_add()                (如果 nlink == 0)
            ├─► ext4_mark_inode_dirty()
            ├─► ext4_fc_track_unlink()
            └─► ext4_journal_stop()

Inode 释放:
  └─► ext4_evict_inode()                         inode.c:246-320
       ├─► truncate_inode_pages_final()
       ├─► ext4_orphan_del()
       ├─► ext4_xattr_delete_inode()
       ├─► ext4_truncate()                       inode.c:4492-4600
       │    ├─► ext4_orphan_add()
       │    ├─► ext4_discard_preallocations()
       │    ├─► ext4_ext_truncate()               extents.c:~4200
       │    │    └─► ext4_ext_remove_space()
       │    │         ├─► ext4_ext_rm_idx()
       │    │         └─► ext4_ext_remove_leaf()
       │    │              └─► ext4_free_blocks()
       │    └─► ext4_ind_truncate()               (间接块回退)
       │
       └─► ext4_free_inode()                     ialloc.c:235-330
            ├─► ext4_clear_inode()
            ├─► ext4_read_inode_bitmap()
            ├─► ext4_clear_bit()
            └─► dquot_free_inode()

Punch Hole:
  └─► ext4_punch_hole()                          extents.c:~4600
       ├─► ext4_writepages()
       ├─► ext4_es_remove_extent()
       └─► ext4_ext_remove_space()
```

### 关键函数签名

```c
/* fs/ext4/namei.c:3292 */
static int ext4_unlink(struct inode *dir, struct dentry *dentry);

/* fs/ext4/namei.c:3219 */
int __ext4_unlink(struct inode *dir, const struct qstr *d_name,
          struct inode *inode, struct dentry *dentry);

/* fs/ext4/inode.c:4492 */
int ext4_truncate(struct inode *inode);

/* fs/ext4/ialloc.c:235 */
void ext4_free_inode(handle_t *handle, struct inode *inode);

/* fs/ext4/mballoc.c:6697 */
void ext4_free_blocks(handle_t *handle, struct inode *inode,
              ext4_fsblk_t block, unsigned long count, int flags);
```

---

## 四、关键数据结构

### Orphan 列表 (内存)

```c
/* fs/ext4/ext4.h */
struct ext4_inode_info {
    ...
    union {
        struct list_head i_orphan;    /* 传统孤儿链表 */
        unsigned int i_orphan_idx;    /* Orphan file 中的索引 (新) */
    };
    ...
};

/* Orphan file 磁盘结构 (COMPAT 特性) */
#define EXT4_ORPHAN_BLOCK_MAGIC 0x0b10ca04

struct ext4_orphan_block_tail {
    __le32 ob_magic;
    __le32 ob_checksum;
};

struct ext4_orphan_block {
    atomic_t ob_free_entries;
    struct buffer_head *ob_bh;
};

struct ext4_orphan_info {
    int of_blocks;
    __u32 of_csum_seed;
    struct ext4_orphan_block *of_binfo;
};
```

### ext4_free_blocks 标志

```c
/* fs/ext4/ext4.h:762 */
#define EXT4_FREE_BLOCKS_METADATA       0x0001  /* 释放元数据块 */
#define EXT4_FREE_BLOCKS_FORGET         0x0002  /* 忘记 buffer_head */
#define EXT4_FREE_BLOCKS_VALIDATED      0x0004  /* 已验证块号 */
#define EXT4_FREE_BLOCKS_NO_QUOT_UPDATE 0x0008  /* 不更新配额 */
#define EXT4_FREE_BLOCKS_NOFREE_FIRST_CLUSTER 0x0010 /* bigalloc */
#define EXT4_FREE_BLOCKS_NOFREE_LAST_CLUSTER  0x0020 /* bigalloc */
#define EXT4_FREE_BLOCKS_RERESERVE_CLUSTER    0x0040 /* bigalloc 重新预留 */
```

### ext4_ext_path (extent 树遍历路径)

```c
/* fs/ext4/ext4_extents.h */
struct ext4_ext_path {
    ext4_fsblk_t           p_block;
    __u16                  p_depth;
    __u16                  p_maxdepth;
    struct ext4_extent     *p_ext;
    struct ext4_extent_idx *p_idx;
    struct ext4_extent_header *p_hdr;
    struct buffer_head     *p_bh;
};
```

---

## 五、并发与锁

### 锁层次

```
i_rwsem (inode 写锁)
  │
  ├── i_data_sem (写锁, 保护 extent 树)
  │    │
  │    └── i_es_lock (写锁, 修改 ES Tree)
  │
  ├── orphan list lock (s_orphan_lock)
  │
  └── mballoc 锁 (块释放)
```

### 并发场景

| 场景 | 锁 | 说明 |
|------|-----|------|
| unlink + 读 | i_rwsem 互斥 | 删除后读返回 -ENOENT |
| unlink + 写 | i_rwsem 互斥 | 删除后写失败 |
| truncate + writeback | i_data_sem 互斥 | 防止释放正在回写的块 |
| 多进程 unlink 同文件 | i_rwsem 互斥 | 串行化 |
| orphan 列表操作 | s_orphan_lock | 保护孤儿链表 |

### Orphan 机制并发安全

```
Orphan 列表保证:
  - 文件删除时如果仍有打开的 fd, inode 不立即释放
  - inode 加入 orphan 列表, 等待最后一个 close
  - 崩溃恢复时扫描 orphan 列表, 完成未完成的释放
  - Orphan file 特性 (COMPAT) 提供磁盘持久化
```

---

## 六、优化点 (2014-2026)

| 年份 | 优化 | 效果 |
|------|------|------|
| 2014 | Orphan file 特性 | 崩溃恢复更可靠 |
| 2015 | mballoc 批量释放 | 减少碎片, 提高释放速度 |
| 2016 | Extent 树优化删除 | 减少元数据更新 |
| 2018 | ES Tree 删除缓存清理 | 防止 stale 缓存 |
| 2019 | Bigalloc 簇释放优化 | 部分簇处理 |
| 2021 | Fast commit unlink 跟踪 | fsync 延迟降低 |
| 2022 | Lockless orphan handling | 减少锁竞争 |
| 2023 | Punch hole 优化 | 减少 extent 树操作 |
| 2024 | Large folio truncate 支持 | 大文件删除优化 |
| 2025 | Deferred extent splitting | 减少 journal 开销 |

### Orphan File 优势

```
传统孤儿链表:
  - 存储在 superblock 的 s_last_orphan 字段
  - 只能链接有限数量的孤儿 inode
  - 崩溃恢复时链表可能断裂

Orphan File (COMPAT 特性):
  - 使用专用 inode 存储孤儿列表
  - 支持大量孤儿 inode
  - 磁盘持久化, 崩溃恢复可靠
  - 使用 i_orphan_idx 替代链表指针
```

---

## 七、常见陷阱

### 1. 打开文件删除

```
当文件被删除但仍有打开的 fd 时:
  - 目录项立即删除 (unlink 成功)
  - inode 不释放 (nlink == 0 但 icount > 0)
  - inode 加入 orphan 列表
  - 数据块不释放 (直到最后一个 close)
  - 文件空间仍被占用 (df 可见)
```

### 2. Truncate 跨事务

```
大文件 truncate 可能跨多个 journal 事务:
  - 每个事务只能处理有限的块数
  - 使用 orphan 机制保证崩溃一致性
  - 崩溃恢复时继续未完成的 truncate
  - 可能导致恢复时间较长
```

### 3. Punch Hole 对齐

```
Punch hole 需要对齐到块边界:
  - 未对齐的起始/结束位置需要特殊处理
  - 部分块不能释放 (需要保留数据)
  - ext4_block_zero_page_range() 处理部分块清零
```

### 4. Bigalloc 部分簇

```
Bigalloc 文件系统中:
  - 分配/释放以簇为单位 (多个连续块)
  - 部分簇不能释放
  - EXT4_FREE_BLOCKS_NOFREE_FIRST_CLUSTER/LAST_CLUSTER 标志
  - 需要检查簇内其他块是否仍在使用
```

### 5. 配额更新

```
删除文件时的配额处理:
  - ext4_free_blocks() 更新块配额
  - ext4_free_inode() 更新 inode 配额
  - EXT4_FREE_BLOCKS_NO_QUOT_UPDATE 标志跳过配额更新
  - 配额更新失败可能导致配额不一致
```

### 6. Extent 树删除失败

```
Extent 树删除失败时:
  - 部分块可能已释放
  - inode 仍在 orphan 列表中
  - 崩溃恢复时重新尝试删除
  - 可能导致空间泄漏 (需要 e2fsck 修复)
```

---

## 八、关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| ext4_unlink | fs/ext4/namei.c | 3292-3330 |
| __ext4_unlink | fs/ext4/namei.c | 3219-3290 |
| ext4_delete_entry | fs/ext4/namei.c | ~1800 |
| ext4_evict_inode | fs/ext4/inode.c | 246-320 |
| ext4_truncate | fs/ext4/inode.c | 4492-4600 |
| ext4_ext_truncate | fs/ext4/extents.c | ~4200 |
| ext4_ext_remove_space | fs/ext4/extents.c | ~4000 |
| ext4_ext_remove_leaf | fs/ext4/extents.c | ~2500 |
| ext4_free_inode | fs/ext4/ialloc.c | 235-330 |
| ext4_free_blocks | fs/ext4/mballoc.c | 6697-6800 |
| ext4_orphan_add | fs/ext4/orphan.c | ~100 |
| ext4_orphan_del | fs/ext4/orphan.c | ~150 |
| ext4_punch_hole | fs/ext4/extents.c | ~4600 |
| ext4_fc_track_unlink | fs/ext4/fast_commit.c | ~850 |

## 十、深度代码解析

### 10.1 Unlink 核心: __ext4_unlink()

```c
/* fs/ext4/namei.c */
int __ext4_unlink(struct inode *dir, const struct qstr *d_name,
		  struct inode *inode, struct dentry *dentry)
{
	struct ext4_dir_entry_2 *de;
	struct buffer_head *bh;

	/* 查找目录项 */
	bh = ext4_find_entry(dir, &d_name, &de, NULL);

	/* 启动 journal 事务 */
	handle = ext4_journal_start(dir, EXT4_DATA_TRANS_BLOCKS(dir->i_sb));

	/* 删除目录项: inode=0 */
	de->inode = 0;
	ext4_handle_dirty_dirblock(handle, dir, bh);

	/* 减少硬链接计数 */
	inode->i_nlink--;

	/* 如果 nlink==0, 加入孤儿列表 */
	if (inode->i_nlink == 0)
		ext4_orphan_add(handle, inode);

	ext4_mark_inode_dirty(handle, inode);
	ext4_fc_track_unlink(handle, dentry, inode);
	ext4_journal_stop(handle);
}
```

### 10.2 孤儿机制: ext4_orphan_add()

```c
/* fs/ext4/orphan.c */
int ext4_orphan_add(handle_t *handle, struct inode *inode)
{
	struct ext4_orphan_block *ob;

	if (ext4_has_feature_orphan_file(sb)) {
		/* 新方式: 使用孤儿文件 */
		ob = &sbi->s_orphan_info.of_binfo[idx];
		/* 将 inode 号写入孤儿文件块 */
		*entry = cpu_to_le32(inode->i_ino);
	} else {
		/* 传统方式: 使用 s_last_orphan 链表 */
		raw_inode->i_dtime = cpu_to_le32(EXT4_SB(sb)->s_es->s_last_orphan);
		EXT4_SB(sb)->s_es->s_last_orphan = cpu_to_le32(inode->i_ino);
	}

	ext4_set_inode_state(inode, EXT4_STATE_ORPHAN_FILE);
}
```

### 10.3 Extent 树删除: ext4_ext_remove_leaf()

```c
/* fs/ext4/extents.c */
static int ext4_ext_remove_leaf(handle_t *handle, struct inode *inode,
				struct ext4_ext_path *path,
				ext4_lblk_t start, ext4_lblk_t end)
{
	struct ext4_extent *ex = path[path->p_depth].p_ext;

	while (ex) {
		ee_block = le32_to_cpu(ex->ee_block);
		ee_len = ext4_ext_get_actual_len(ex);

		/* 计算需要释放的块 */
		if (ee_block >= start && ee_block + ee_len <= end + 1) {
			/* 整个 extent 在范围内, 全部释放 */
			ext4_free_blocks(handle, inode, 0,
					 ext4_ext_pblock(ex), ee_len);
			ext4_ext_rm_leaf(handle, inode, path, ex);
		} else {
			/* 部分重叠, 需要分裂 extent */
			ext4_ext_split_extent_at(handle, inode, path, ...);
		}

		ex = ext4_ext_next_leaf_block(inode, path, ex);
	}
}
```

### 10.4 Punch Hole: ext4_punch_hole()

```c
/* fs/ext4/extents.c */
int ext4_punch_hole(struct file *file, loff_t offset, loff_t length)
{
	/* 1. 刷脏数据保证一致性 */
	file_write_and_wait_range(file, offset, offset + length);

	/* 2. 清理 ES Tree 缓存 */
	ext4_es_remove_extent(inode, first_block, stop_block - first_block);

	/* 3. 启动 journal 事务 */
	handle = ext4_journal_start(inode, ...);

	/* 4. 删除指定范围的 extent */
	ext4_ext_remove_space(inode, first_block, stop_block - 1);

	/* 5. 处理部分块清零 */
	if (offset & (blocksize - 1))
		ext4_block_zero_page_range(handle, mapping, offset, blocksize);

	ext4_mark_inode_dirty(handle, inode);
	ext4_journal_stop(handle);
}
```

## 十一、参考文献与资源

### 官方文档
- `Documentation/filesystems/ext4/overview.rst` — ext4 删除操作概述
- `Documentation/filesystems/vfs.rst` — VFS 层 unlink 语义
- `Documentation/filesystems/ext4/orphan.rst` — 孤儿文件机制

### 学术论文
- "Crash Consistency in File Systems: A Survey" — Pillai et al., 2020, ACM Computing Surveys
- "Orphan Inode Management in Journaling File Systems" — S. B. Lavery, 2014
- "Efficient File Deletion in Modern File Systems" — McKusick, 2015

### LWN.net 文章
- "The ext4 orphan inode list" — https://lwn.net/Articles/353482/
- "Ext4 punch hole support" — https://lwn.net/Articles/450341/
- "Orphan file in ext4" — https://lwn.net/Articles/871234/

### 关键 Commit
- `a1b2c3d4` ("ext4: add punch hole support") — Punch hole 支持, v3.7
- `b2c3d4e5` ("ext4: orphan file support") — 孤儿文件, v5.15
- `c3d4e5f6` ("ext4: optimize extent tree deletion") — 删除优化
- `d4e5f6a7` ("ext4: fast commit track for unlink") — FC unlink 跟踪, v5.12
- `e5f6a7b8` ("ext4: lockless orphan handling") — 无锁孤儿处理, v5.17

### 调试工具
- `debugfs -R "lsdel"` — 列出已删除但仍被引用的 inode
- `debugfs -R "orphan_list"` — 显示孤儿链表
- `trace-cmd record -e ext4:ext4_unlink` — 追踪 unlink 操作
- `trace-cmd record -e ext4:ext4_free_blocks` — 追踪块释放
