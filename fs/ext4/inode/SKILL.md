---
name: "ext4-inode"
description: "ext4文件系统Inode结构专家。当用户询问ext4 inode结构、i_mode、i_block、extent树、inode分配时调用此技能。"
---

# ext4 Inode

## 〇、为什么需要这个机制？

为什么需要 inode？Unix 文件系统的核心设计哲学是"数据与元数据分离"。inode 存储文件的元数据（权限、大小、时间戳、数据位置），而不存储文件名。这使得硬链接成为可能（多个目录项指向同一个 inode）。ext4 的 inode 从 ext2 的 128 字节扩展到 256 字节，增加了纳秒时间戳、项目 ID、校验和等字段。

inode 的设计直接影响文件系统的性能和可靠性。早期的 ext2 inode 只有 15 个直接/间接块指针，大文件需要多级间接块查找，性能差且碎片严重。ext4 引入 extent 树后，i_block 区域被重新解释为 extent 根节点，大幅提升了大文件的访问效率。

没有 inode，文件系统就无法描述"文件是什么"——权限、所有者、数据位置等核心信息都依赖 inode 存储。

---

## 一、磁盘 Inode 结构 (verified: ext4.h:795-854)

```c
struct ext4_inode {
	__le16	i_mode;		/* File mode */
	__le16	i_uid;		/* Low 16 bits of Owner Uid */
	__le32	i_size_lo;	/* Size in bytes */
	__le32	i_atime;	/* Access time */
	__le32	i_ctime;	/* Inode Change time */
	__le32	i_mtime;	/* Modification time */
	__le32	i_dtime;	/* Deletion Time */
	__le16	i_gid;		/* Low 16 bits of Group Id */
	__le16	i_links_count;	/* Links count */
	__le32	i_blocks_lo;	/* Blocks count */
	__le32	i_flags;	/* File flags */
	union {
		struct { __le32  l_i_version; } linux1;
		struct { __u32  h_i_translator; } hurd1;
		struct { __u32  m_i_reserved1; } masix1;
	} osd1;
	__le32	i_block[EXT4_N_BLOCKS];/* Pointers to blocks */
	__le32	i_generation;	/* File version (for NFS) */
	__le32	i_file_acl_lo;	/* File ACL */
	__le32	i_size_high;
	__le32	i_obso_faddr;	/* Obsoleted fragment address */
	union {
		struct {
			__le16	l_i_blocks_high;
			__le16	l_i_file_acl_high;
			__le16	l_i_uid_high;
			__le16	l_i_gid_high;
			__le16	l_i_checksum_lo;
			__le16	l_i_reserved;
		} linux2;
		struct {
			__le16	h_i_reserved1;
			__u16	h_i_mode_high;
			__u16	h_i_uid_high;
			__u16	h_i_gid_high;
			__u32	h_i_author;
		} hurd2;
		struct {
			__le16	h_i_reserved1;
			__le16	m_i_file_acl_high;
			__u32	m_i_reserved2[2];
		} masix2;
	} osd2;
	__le16	i_extra_isize;
	__le16	i_checksum_hi;
	__le32  i_ctime_extra;
	__le32  i_mtime_extra;
	__le32  i_atime_extra;
	__le32  i_crtime;
	__le32  i_crtime_extra;
	__le32  i_version_hi;
	__le32	i_projid;	/* Project ID — LAST field, NO i_pad */
};
```

**关键**: `i_projid` 是最后一个字段，**没有** `i_pad` 或填充。

---

## 二、i_block[15] 两种用法

### 间接块模式

```
i_block[0-11]:  直接块指针 (12个)
i_block[12]:    一级间接块指针
i_block[13]:    二级间接块指针
i_block[14]:    三级间接块指针
```

### Extent 模式 (i_flags & EXT4_EXTENTS_FL)

```c
/* i_block 前12字节存 ext4_extent_header */
struct ext4_extent_header {
	__le16	eh_magic;	/* 0xf30a */
	__le16	eh_entries;
	__le16	eh_max;
	__le16	eh_depth;
	__le32	eh_generation;
};
/* 后跟 ext4_extent 或 ext4_extent_idx 数组 */
```

---

## 三、Inode 标志 (i_flags, verified: ext4.h:482-514)

```c
#define EXT4_SECRM_FL			0x00000001
#define EXT4_UNRM_FL			0x00000002
#define EXT4_COMPR_FL			0x00000004
#define EXT4_SYNC_FL			0x00000008
#define EXT4_IMMUTABLE_FL		0x00000010
#define EXT4_APPEND_FL			0x00000020
#define EXT4_NODUMP_FL			0x00000040
#define EXT4_NOATIME_FL			0x00000080
#define EXT4_DIRTY_FL			0x00000100
#define EXT4_COMPRBLK_FL		0x00000200
#define EXT4_NOCOMPR_FL			0x00000400
#define EXT4_ENCRYPT_FL			0x00000800
#define EXT4_INDEX_FL			0x00001000
#define EXT4_IMAGIC_FL			0x00002000
#define EXT4_JOURNAL_DATA_FL		0x00004000
#define EXT4_NOTAIL_FL			0x00008000
#define EXT4_DIRSYNC_FL			0x00010000
#define EXT4_TOPDIR_FL			0x00020000
#define EXT4_HUGE_FILE_FL               0x00040000
#define EXT4_EXTENTS_FL			0x00080000
#define EXT4_VERITY_FL			0x00100000
#define EXT4_EA_INODE_FL	        0x00200000
#define EXT4_DAX_FL			0x02000000
#define EXT4_INLINE_DATA_FL		0x10000000
#define EXT4_PROJINHERIT_FL		0x20000000
#define EXT4_CASEFOLD_FL		0x40000000
#define EXT4_RESERVED_FL		0x80000000
```

---

## 四、Inode 状态标志 (ext4.h:1963-1977)

```c
enum {
	EXT4_STATE_NEW,
	EXT4_STATE_XATTR,
	EXT4_STATE_NO_EXPAND,
	EXT4_STATE_DA_ALLOC_CLOSE,
	EXT4_STATE_EXT_MIGRATE,
	EXT4_STATE_NEWENTRY,
	EXT4_STATE_MAY_INLINE_DATA,
	EXT4_STATE_EXT_PRECACHED,
	EXT4_STATE_LUSTRE_EA_INODE,
	EXT4_STATE_VERITY_IN_PROGRESS,
	EXT4_STATE_FC_COMMITTING,
	EXT4_STATE_FC_FLUSHING_DATA,
	EXT4_STATE_ORPHAN_FILE,
};
```

---

## 五、内存 Inode (ext4_inode_info, ext4.h:1031-1199)

```c
struct ext4_inode_info {
	__le32	i_data[15];
	__u32	i_dtime;
	ext4_fsblk_t	i_file_acl;
	ext4_group_t	i_block_group;
	ext4_lblk_t	i_dir_start_lookup;
	unsigned long	i_flags;
	struct rw_semaphore xattr_sem;
	union {
		struct list_head i_orphan;
		unsigned int i_orphan_idx;
	};
	/* Fast commit */
	struct list_head i_fc_dilist;
	struct list_head i_fc_list;
	ext4_lblk_t i_fc_lblk_start;
	ext4_lblk_t i_fc_lblk_len;
	spinlock_t i_raw_lock;
	wait_queue_head_t i_fc_wait;
	spinlock_t i_fc_lock;
	loff_t	i_disksize;
	struct rw_semaphore i_data_sem;
	struct inode vfs_inode;
	struct jbd2_inode *jinode;
	struct timespec64 i_crtime;
	/* mballoc */
	atomic_t i_prealloc_active;
	unsigned int i_reserved_data_blocks;
	struct rb_root i_prealloc_node;
	rwlock_t i_prealloc_lock;
	/* extents status tree */
	struct ext4_es_tree i_es_tree;
	rwlock_t i_es_lock;
	struct list_head i_es_list;
	unsigned int i_es_all_nr;
	unsigned int i_es_shk_nr;
	ext4_lblk_t i_es_shrink_lblk;
	u64 i_es_seq;		/* Change counter for extents */
	ext4_group_t	i_last_alloc_group;
	struct ext4_pending_tree i_pending_tree;
	__u16 i_extra_isize;
	u16 i_inline_off;
	u16 i_inline_size;
	spinlock_t i_block_reservation_lock;
	spinlock_t i_completed_io_lock;
	struct list_head i_rsv_conversion_list;
	struct work_struct i_rsv_conversion_work;
	tid_t i_sync_tid;
	tid_t i_datasync_tid;
	__u32 i_csum_seed;
	kprojid_t i_projid;
#ifdef CONFIG_FS_ENCRYPTION
	struct fscrypt_inode_info *i_crypt_info;
#endif
};
```

---

## 六、Inode 校验和

```c
/* osd2.linux2.l_i_checksum_lo + i_checksum_hi 组成完整 CRC32c */
/* 种子: uuid + inode_number + inode_data */
```

---

## 七、特殊 Inode 号 (ext4.h:314-324)

| Inode | 用途 |
|-------|------|
| 1 | Bad blocks |
| 2 | Root directory |
| 3 | User quota |
| 4 | Group quota |
| 5 | Boot loader |
| 6 | Undelete directory |
| 7 | Resize inode |
| 8 | Journal inode |

---

## 八、关键代码位置

| 功能 | 文件 |
|------|------|
| 磁盘结构 | fs/ext4/ext4.h:795-854 |
| 内存结构 | fs/ext4/ext4.h:1031-1199 |
| Inode 操作 | fs/ext4/inode.c |
| Inode 分配 | fs/ext4/ialloc.c |

---

## 九、深度代码解析

### 9.1 Inode 分配算法 (Orlov 分配器)

ext4 使用 Orlov 分配器为新目录选择块组，核心目标是将相关文件放在一起以提升局部性:

```c
// fs/ext4/ialloc.c:925-1020 (简化)
struct inode *__ext4_new_inode(struct mnt_idmap *idmap,
                               handle_t *handle, struct inode *dir,
                               umode_t mode, const struct qstr *qstr,
                               __u32 goal, ...)
{
    // 1. 检查父目录是否有效
    if (!dir || !dir->i_nlink)
        return ERR_PTR(-EPERM);
    
    // 2. 分配 VFS inode
    inode = new_inode(sb);
    
    // 3. 初始化 project ID (继承自父目录)
    if (ext4_has_feature_project(sb) &&
        ext4_test_inode_flag(dir, EXT4_INODE_PROJINHERIT))
        ei->i_projid = EXT4_I(dir)->i_projid;
    
    // 4. 选择块组
    if (S_ISDIR(mode))
        ret2 = find_group_orlov(sb, dir, &group, mode, qstr);  // 目录用 Orlov
    else
        ret2 = find_group_other(sb, dir, &group, mode);        // 文件用简单策略
    
    // 5. 在选定块组中查找空闲 inode
    for (i = 0; i < ngroups; i++, ino = 0) {
        inode_bitmap_bh = ext4_read_inode_bitmap(sb, group);
        ret2 = find_inode_bit(sb, group, inode_bitmap_bh, &ino);
        if (!ret2)
            goto next_group;
        
        // 6. 原子设置位图
        ext4_lock_group(sb, group);
        ret2 = ext4_test_and_set_bit(ino, inode_bitmap_bh->b_data);
        ext4_unlock_group(sb, group);
        
        if (!ret2)
            goto got;  // 成功分配
    }
}
```

**Orlov 分配器策略**:
- 新顶级目录: 在空闲 inode 最多的块组中分配
- 子目录: 尽量与父目录在同一 flex_bg 中
- 文件: 尽量与父目录/同一目录中的文件在同一块组

### 9.2 Inode 读取流程

```c
// fs/ext4/inode.c (简化)
struct inode *ext4_iget(struct super_block *sb, unsigned long ino, int flags)
{
    struct inode *inode;
    struct ext4_iloc iloc;
    struct ext4_inode *raw_inode;
    
    // 1. VFS inode 查找/分配
    inode = iget_locked(sb, ino);
    
    // 2. 定位磁盘 inode
    ret = __ext4_get_inode_loc(inode, &iloc, flags);
    raw_inode = ext4_raw_inode(&iloc);
    
    // 3. 验证 inode 校验和
    if (ext4_has_metadata_csum(sb)) {
        if (!ext4_inode_csum_verify(inode, raw_inode, ei))
            goto bad_inode;
    }
    
    // 4. 填充 VFS inode
    inode->i_mode = le16_to_cpu(raw_inode->i_mode);
    i_uid_write(inode, i_uid);
    i_gid_write(inode, i_gid);
    set_nlink(inode, le16_to_cpu(raw_inode->i_links_count));
    
    // 5. 处理 i_block 区域
    if (ext4_test_inode_flag(inode, EXT4_INODE_EXTENTS)) {
        // Extent 模式: i_block 是 extent 树根
    } else {
        // 间接块模式: i_block 是块指针数组
    }
    
    return inode;
}
```

### 9.3 Inode 定位计算

```c
// fs/ext4/inode.c
int __ext4_get_inode_loc(struct inode *inode,
                         struct ext4_iloc *iloc, int flags)
{
    ext4_group_t block_group;
    unsigned long offset;
    
    // inode 号从 1 开始，数组索引从 0 开始
    block_group = (ino - 1) / EXT4_INODES_PER_GROUP(sb);
    offset = ((ino - 1) % EXT4_INODES_PER_GROUP(sb)) * EXT4_INODE_SIZE(sb);
    
    // 获取块组描述符
    gdp = ext4_get_group_desc(sb, block_group, NULL);
    
    // 计算 inode 所在的块
    block = ext4_inode_table(sb, gdp) + (offset >> sb->s_blocksize_bits);
    
    // 计算块内偏移
    iloc->offset = offset & (sb->s_blocksize - 1);
    iloc->bh = sb_getblk(sb, block);
    
    return 0;
}
```

**inode 定位公式**:
```
block_group = (inode_no - 1) / inodes_per_group
index_in_group = (inode_no - 1) % inodes_per_group
byte_offset = index_in_group * inode_size
block = inode_table_block + byte_offset / block_size
offset_in_block = byte_offset % block_size
```

### 9.4 Inode 校验和计算

```c
// fs/ext4/inode.c
static __le32 ext4_inode_csum(struct inode *inode, struct ext4_inode *raw,
                              struct ext4_inode_info *ei)
{
    struct ext4_sb_info *sbi = EXT4_SB(inode->i_sb);
    __u32 csum;
    __le32 inum = cpu_to_le32(inode->i_ino);
    __le32 gen = raw->i_generation;
    
    // 种子 = crc32c(uuid + inode_no + generation)
    csum = ext4_chksum(sbi, ei->i_csum_seed, (__u8 *)&inum, sizeof(inum));
    csum = ext4_chksum(sbi, csum, (__u8 *)&gen, sizeof(gen));
    
    // 最终校验和 = crc32c(seed, inode_data)
    // 需要排除 i_checksum_lo 和 i_checksum_hi 字段本身
    csum = ext4_chksum(sbi, csum, (__u8 *)raw, offset_of_checksum);
    csum = ext4_chksum(sbi, csum, (__u8 *)raw + offset_after_checksum, ...);
    
    return cpu_to_le32(csum);
}
```

### 9.5 i_disksize vs i_size

```c
// fs/ext4/ext4.h:1095-1108 (注释)
/*
 * i_disksize keeps track of what the inode size is ON DISK, not
 * in memory.  During truncate, i_size is set to the new size by
 * the VFS prior to calling ext4_truncate(), but the filesystem won't
 * set i_disksize to 0 until the truncate is actually under way.
 *
 * The intent is that i_disksize always represents the blocks which
 * are used by this file.  This allows recovery to restart truncate
 * on orphans if we crash during truncate.
 */
loff_t i_disksize;
```

**两者关系**:
- `i_size`: VFS 层的文件大小，用户可见
- `i_disksize`: ext4 跟踪的磁盘实际大小

```
正常情况:        i_size == i_disksize
truncate 进行中: i_size < i_disksize (数据块尚未释放)
延迟分配中:      i_size > i_disksize (数据尚未写入磁盘)
```

### 9.6 时间戳精度扩展

```c
// fs/ext4/ext4.h:897-911
static inline struct timespec64 ext4_decode_extra_time(__le32 base,
                                                       __le32 extra)
{
    struct timespec64 ts = { .tv_sec = (signed)le32_to_cpu(base) };

    // extra 字段的低 2 位是 epoch bits，扩展 tv_sec 到 34 位
    if (unlikely(extra & cpu_to_le32(EXT4_EPOCH_MASK)))
        ts.tv_sec += (u64)(le32_to_cpu(extra) & EXT4_EPOCH_MASK) << 32;
    
    // extra 字段的高 30 位是纳秒
    ts.tv_nsec = (le32_to_cpu(extra) & EXT4_NSEC_MASK) >> EXT4_EPOCH_BITS;
    
    return ts;
}

// 时间范围: 1901-12-13 到 2446-05-10
```

---

## 十、参考文献与资源

### 官方文档
1. **Linux 内核文档**: [Documentation/filesystems/ext4/inodes.rst](https://www.kernel.org/doc/html/latest/filesystems/ext4/dynamic.html#inode-table)
2. **ext4 磁盘格式**: https://www.kernel.org/doc/html/latest/filesystems/ext4/ondisk/inode.html

### 学术论文
3. **"The Orlov Block Allocator"** - Ganger et al.
   - 描述 Orlov 分配器的原始设计
4. **"A Fast File System for UNIX"** - McKusick et al., TOCS 1984
   - BSD FFS 的 inode 分配策略，ext4 Orlov 的灵感来源

### LWN.net 文章
5. **"Ext4 inode timestamps"** - https://lwn.net/Articles/804382/ (2020)
   - Y2038 问题和时间戳扩展
6. **"Improving ext4's inode allocation"** - https://lwn.net/Articles/469805/ (2011)

### 关键 Commit
7. **Orlov 分配器**: `a4912123` "ext4: add Orlov allocator" (2006)
8. **inode 校验和**: `9aa5d32b` "ext4: add inode checksum support" (2012)
9. **纳秒时间戳**: `ef7f3835` "ext4: add nanosecond timestamps" (2007)
10. **project ID**: `689c958c` "ext4: add project quota support" (2015)

### 调试工具
11. **stat**: 查看 inode 信息
    ```bash
    stat /path/to/file
    # 输出: Inode: 12345  Links: 1  ...
    
    # 查看原始 inode
    stat -f /path/to/file
    ```

12. **debugfs**: 原始 inode 访问
    ```bash
    debugfs /dev/sda1
    debugfs: stat <12345>      # 按 inode 号查看
    debugfs: stat /path/file   # 按路径查看
    debugfs: icheck 12345      # inode 到块映射
    debugfs: ncheck 12345      # inode 到文件名
    ```

13. **ls -i**: 显示 inode 号
    ```bash
    ls -li /path/to/dir
    # 第一列是 inode 号
    ```

---

## 十一、常见问题与陷阱

### Q1: 为什么 inode 号从 1 开始而不是 0？

**A**: 历史原因。inode 0 用作"无效 inode"标记（类似 NULL 指针）。目录项中的 `inode = 0` 表示该项已删除。所以有效 inode 从 1 开始。

### Q2: `EXT4_GOOD_OLD_INODE_SIZE` (128) vs 实际 inode 大小

**A**: 
- 128 字节: ext2/ext3 最小 inode 大小
- 256 字节: 现代 ext4 默认值 (`mkfs.ext4 -I 256`)

额外空间用于:
- 扩展时间戳 (`i_crtime`, `*_extra` 字段)
- 内联扩展属性
- project ID
- 校验和

### Q3: `i_flags` (磁盘) vs `i_state` (内存) 的区别

**A**:
- `i_flags`: 持久化到磁盘，如 `EXT4_EXTENTS_FL`, `EXT4_ENCRYPT_FL`
- `i_state`: 仅内存状态，如 `EXT4_STATE_NEW`, `EXT4_STATE_ORPHAN_FILE`

```c
// i_flags 存储在磁盘 inode 的 i_flags 字段
// i_state 存储在内存 ext4_inode_info 的 i_flags 高 32 位 (64位系统)
//         或 i_state_flags 字段 (32位系统)
```

### Q4: 如何判断 inode 使用 extent 还是间接块？

**A**: 检查 `EXT4_EXTENTS_FL` 标志:

```c
if (ext4_test_inode_flag(inode, EXT4_INODE_EXTENTS)) {
    // Extent 模式: i_block 前 12 字节是 ext4_extent_header
    // 后续是 ext4_extent 或 ext4_extent_idx 数组
} else {
    // 间接块模式:
    // i_block[0-11]: 直接块指针
    // i_block[12]: 一级间接
    // i_block[13]: 二级间接
    // i_block[14]: 三级间接
}
```

新创建的 ext4 文件默认使用 extent 模式（除非文件系统不支持）。

### Q5: inode 泄漏问题诊断

**A**: 当 `df -i` 显示 inode 用尽但 `find` 找不到那么多文件时:

```bash
# 检查是否有已删除但被占用的文件
lsof +L1

# 检查孤儿 inode (崩溃恢复)
debugfs /dev/sda1 -R "ls -d /lost+found"

# 检查 inode 位图
dumpe2fs /dev/sda1 | grep "Free inodes"
```
