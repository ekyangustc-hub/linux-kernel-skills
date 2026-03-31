---
name: "ext4-directory"
description: "ext4文件系统目录结构专家。当用户询问ext4目录项、HTREE索引、目录格式、目录操作时调用此技能。"
---

# ext4 目录结构

## 一、概述

ext4 支持多种目录格式，从小型线性目录到大型 HTREE 索引目录。

---

## 二、目录格式

### 2.1 格式类型

| 格式 | 说明 | 适用场景 |
|------|------|----------|
| 线性目录 | 简单的目录项列表 | 小目录 |
| HTREE 索引 | 哈希树索引 | 大目录 |

### 2.2 格式判断

```c
/* 检查是否使用 HTREE 索引 */
#define EXT4_INDEX_FL    0x00001000

if (inode->i_flags & EXT4_INDEX_FL) {
    /* 使用 HTREE 索引 */
} else {
    /* 线性目录 */
}
```

---

## 三、目录项结构

### 3.1 基本目录项

**位置**: `fs/ext4/ext4.h`

```c
struct ext4_dir_entry {
    __le32  inode;          /* inode 号 */
    __le16  rec_len;        /* 记录长度 */
    __le16  name_len;       /* 文件名长度 */
    char    name[EXT4_NAME_LEN]; /* 文件名 */
};
```

### 3.2 扩展目录项 (ext4_dir_entry_2)

```c
struct ext4_dir_entry_2 {
    __le32  inode;          /* inode 号 */
    __le16  rec_len;        /* 记录长度 */
    __u8    name_len;       /* 文件名长度 */
    __u8    file_type;      /* 文件类型 */
    char    name[EXT4_NAME_LEN]; /* 文件名 */
};
```

### 3.3 文件类型

| 值 | 类型 |
|----|------|
| 0 | 未知 |
| 1 | 普通文件 |
| 2 | 目录 |
| 3 | 字符设备 |
| 4 | 块设备 |
| 5 | FIFO |
| 6 | Socket |
| 7 | 符号链接 |

---

## 四、线性目录

### 4.1 布局

```
┌─────────────────────────────────────────────────────────────┐
│                      线性目录块                              │
├─────────────────────────────────────────────────────────────┤
│  entry[0]: inode=2, name=".", rec_len=12                    │
│  entry[1]: inode=2, name="..", rec_len=12                   │
│  entry[2]: inode=12, name="lost+found", rec_len=20          │
│  entry[3]: inode=131, name="file1.txt", rec_len=16          │
│  ...                                                        │
│  unused: inode=0, rec_len=剩余空间                          │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 目录项查找

```c
/* 线性查找 */
struct ext4_dir_entry_2 *
ext4_find_entry_linear(struct inode *dir, const struct qstr *name)
{
    struct ext4_dir_entry_2 *de;
    struct buffer_head *bh;
    char *dir_data;

    /* 读取目录数据块 */
    bh = ext4_bread(NULL, dir, 0, 0);

    /* 线性扫描 */
    dir_data = bh->b_data;
    while ((char *)de < dir_data + dir->i_sb->s_blocksize) {
        if (de->inode != 0 &&
            de->name_len == name->len &&
            !memcmp(de->name, name->name, name->len)) {
            return de;  /* 找到 */
        }
        de = (struct ext4_dir_entry_2 *)((char *)de + de->rec_len);
    }

    return NULL;  /* 未找到 */
}
```

---

## 五、HTREE 索引

### 5.1 概述

HTREE (Hash Tree) 是一种基于哈希的目录索引结构，用于加速大目录的查找。

### 5.2 HTREE 结构

```
                    ┌─────────────┐
                    │  Root Node  │ (在 inode 内)
                    │  dx_root    │
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │ Index Node  │ │ Index Node  │ │ Index Node  │
    │ dx_node     │ │ dx_node     │ │ dx_node     │
    └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
           │               │               │
           ▼               ▼               ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │ Data Block  │ │ Data Block  │ │ Data Block  │
    │ 目录项      │ │ 目录项      │ │ 目录项      │
    └─────────────┘ └─────────────┘ └─────────────┘
```

### 5.3 HTREE 节点结构

```c
/* HTREE 根节点 */
struct dx_root {
    struct fake_dirent dot;
    struct fake_dirent dotdot;
    struct dx_root_info info;
    struct dx_entry entries[0];
};

/* HTREE 索引节点 */
struct dx_node {
    struct fake_dirent fake;
    struct dx_entry entries[0];
};

/* 索引条目 */
struct dx_entry {
    __le32 hash;        /* 哈希值 */
    __le32 block;       /* 数据块号 */
};
```

### 5.4 HTREE 查找

```c
/* HTREE 查找 */
struct ext4_dir_entry_2 *
ext4_find_entry_htree(struct inode *dir, const struct qstr *name)
{
    struct dx_frame frame;
    u32 hash;

    /* 1. 计算哈希值 */
    hash = ext4_htree_hash(name->name, name->len);

    /* 2. 从根节点开始查找 */
    dx_probe(hash, dir, &frame);

    /* 3. 遍历索引节点 */
    while (frame.entries <= frame.at) {
        /* 读取下一个索引块 */
        dx_get_block(frame.at);
    }

    /* 4. 在数据块中查找 */
    return ext4_search_dirblock(frame.bh, name);
}
```

---

## 六、哈希函数

### 6.1 哈希版本

```c
#define DX_HASH_LEGACY          0   /* 传统哈希 */
#define DX_HASH_HALF_MD4        1   /* 半 MD4 */
#define DX_HASH_TEA             2   /* TEA */
#define DX_HASH_LEGACY_UNSIGNED 3   /* 传统无符号 */
#define DX_HASH_HALF_MD4_UNSIGNED 4 /* 半 MD4 无符号 */
#define DX_HASH_TEA_UNSIGNED    5   /* TEA 无符号 */
#define DX_HASH_SIPHASH         6   /* SipHash */
```

### 6.2 哈希计算

```c
/* 计算文件名哈希值 */
u32 ext4_htree_hash(const char *name, int len)
{
    struct dx_hash_info hinfo;

    hinfo.hash_version = DX_HASH_HALF_MD4;
    hinfo.seed = EXT4_SB(dir->i_sb)->s_hash_seed;

    ext4fs_dirhash(name, len, &hinfo);

    return hinfo.hash;
}
```

---

## 七、目录操作

### 7.1 内联目录转换重构

```c
/* ext4: refactor the inline directory conversion and new directory codepaths */
/* 重构内联目录转换和新目录代码路径 */
/* 统一处理逻辑，减少代码重复 */
```

### 7.2 Large Block Size 支持

```c
/* ext4: support large block size in ext4_readdir() */
/* 目录读取现在支持大块大小 */

/* ext4: remove PAGE_SIZE checks for rec_len conversion */
/* 移除 rec_len 转换中的 PAGE_SIZE 检查 */

/* ext4: remove page offset calculation in ext4_block_truncate_page() */
/* ext4: remove page offset calculation in ext4_block_zero_page_range() */
/* 移除页偏移计算，使用 folio 替代 */
```

### 7.3 添加目录项

```c
int ext4_add_entry(handle_t *handle, struct dentry *dentry,
                   struct inode *inode)
{
    struct inode *dir = dentry->d_parent->d_inode;

    /* 1. 查找空闲位置 */
    de = ext4_find_entry_free(dir, &bh);

    /* 2. 创建目录项 */
    de->inode = cpu_to_le32(inode->i_ino);
    de->name_len = dentry->d_name.len;
    de->file_type = ext4_file_type(inode->i_mode);
    memcpy(de->name, dentry->d_name.name, dentry->d_name.len);

    /* 3. 标记为脏 */
    ext4_mark_inode_dirty(handle, dir);

    return 0;
}
```

### 7.2 删除目录项

```c
int ext4_delete_entry(handle_t *handle, struct inode *dir,
                      struct ext4_dir_entry_2 *de,
                      struct buffer_head *bh)
{
    /* 1. 找到前一个目录项 */
    prev_de = ext4_find_prev_entry(dir, de);

    /* 2. 合并空间 */
    prev_de->rec_len += de->rec_len;
    de->inode = 0;

    /* 3. 标记为脏 */
    ext4_handle_dirty_metadata(handle, dir, bh);

    return 0;
}
```

---

## 八、内联数据目录

### 8.1 概述

当目录很小时，目录数据可以内联存储在 inode 中。

### 8.2 布局

```
┌─────────────────────────────────────────────────────────────┐
│                        Inode                                │
├─────────────────────────────────────────────────────────────┤
│  i_block[0-59]: 内联目录数据                                 │
│  entry[0]: inode=2, name="."                                │
│  entry[1]: inode=2, name=".."                               │
│  entry[2]: inode=..., name="..."                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 九、调试命令

### 9.1 debugfs

```bash
debugfs /dev/sdX

# 查看目录内容
debugfs: ls -l <目录inode>

# 输出示例
  2  40755 (2)      0      0    4096 28-Oct-2020 10:30 .
  2  40755 (2)      0      0    4096 28-Oct-2020 10:30 ..
 11  40700 (2)      0      0   16384 28-Oct-2020 10:30 lost+found
131 100644 (1)      0      0       0 28-Oct-2020 10:30 file1.txt

# 查看 HTREE 索引
debugfs: htree <目录inode>
```

### 9.2 查看 HTREE 信息

```bash
# 使用 stat 查看目录标志
debugfs: stat <目录inode>
# 查看 Flags: 0x1000 表示使用 HTREE
```

---

## 十、代码位置

| 功能 | 文件路径 |
|------|----------|
| 结构定义 | `fs/ext4/ext4.h` |
| 目录操作 | `fs/ext4/dir.c` |
| HTREE 索引 | `fs/ext4/namei.c` |
| 内联数据 | `fs/ext4/inline.c` |

---

## 十一、参考资源

- ext4 官方文档: `Documentation/filesystems/ext4/`
- ext4 内核代码: `fs/ext4/dir.c`
