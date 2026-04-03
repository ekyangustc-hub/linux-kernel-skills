---
name: ext4-why-metadata-csum
description: ext4元数据校验和存在必要性专家。当用户询问为什么ext4需要metadata_csum、静默数据损坏、校验和算法、校验和与日志关系时调用此技能。
---

# 为什么 ext4 需要 Metadata Checksum

## 1. 问题起源

在 metadata_csum 之前，ext4 无法检测**静默数据损坏**（silent data corruption）：

```
静默数据损坏的来源:
  1. 磁盘位翻转 (bit rot)
     - 磁性介质退化
     - SSD 电荷泄漏
     - 宇宙射线引起的软错误

  2. 内存错误
     - ECC 内存也有未纠正错误率
     - 非 ECC 内存错误率更高

  3. 传输错误
     - SATA/SAS 链路 CRC 错误（罕见但存在）
     - USB 存储控制器 bug

  4. 固件 bug
     - 磁盘固件错误
     - RAID 控制器 bug
```

没有校验和时的场景：

```
场景 1: inode 损坏
  正常: inode 指向 blocks [100, 101, 102, 103]
  损坏: 磁盘位翻转 → inode 指向 blocks [100, 101, 0xFFFF0102, 103]
  结果: 读到错误数据，但文件系统认为一切正常

场景 2: 块位图损坏
  正常: 块 1000-2000 已分配
  损坏: 位图翻转 → 块 1500 标记为空闲
  结果: 块 1500 被双重分配 → 数据互相覆盖

场景 3: 目录项损坏
  正常: 目录项 "file.txt" → inode 12345
  损坏: 目录项 "file.txt" → inode 0xDEADBEEF
  结果: 访问不存在的 inode → 内核 panic 或返回错误数据
```

e2fsck 只能检测**结构不一致**，无法检测**内容损坏**：
- 块位图与 inode 不一致 → 能检测
- inode 内部指针指向错误位置 → 无法检测（没有校验和）

## 2. 为什么需要 Metadata Checksum

Metadata checksum 解决的核心问题：**检测所有元数据的静默损坏**。

| 问题 | 无校验和 | 有校验和 |
|------|---------|---------|
| inode 损坏 | 无法检测 | 校验和不匹配，标记损坏 |
| 块位图损坏 | 可能检测（与 inode 对比） | 直接检测 |
| 目录项损坏 | 无法检测 | 校验和不匹配 |
| extent 树损坏 | 无法检测 | 校验和不匹配 |
| 日志损坏 | 部分检测 | 完整校验和保护 |

Metadata checksum 在 2012 年（内核 3.5）由 Darrick J. Wong 引入，他是 XFS 和 ext4 的主要开发者，后来成为 XFS 的维护者。

### 与数据校验和的区别

ext4 的 metadata_csum **只保护元数据**，不保护文件数据：

```
元数据 (有校验和):
  - 超级块
  - 块组描述符
  - inode
  - 块位图
  - inode 位图
  - 目录项
  - extent 树
  - 扩展属性
  - 日志元数据

数据 (无校验和):
  - 文件内容
  - 需要上层应用或 dm-integrity 保护
```

## 3. 核心设计

校验和的设计原则：

1. **覆盖所有元数据**：每个元数据块都有独立校验和
2. **存储位置**：校验和存储在元数据块内部（不占用额外空间）
3. **算法选择**：CRC32C（硬件加速，性能好）
4. **种子随机化**：使用随机种子防止恶意构造
5. **向后兼容**：作为 RO_COMPAT 特性，旧内核可以只读挂载

```
校验和计算流程:
  1. 准备元数据块
  2. 将校验和字段清零
  3. 计算 CRC32C(uuid + 元数据)
  4. 将校验和写入校验和字段
  5. 写入磁盘

校验和验证流程:
  1. 读取元数据块
  2. 保存校验和字段
  3. 将校验和字段清零
  4. 计算 CRC32C(uuid + 元数据)
  5. 对比计算的校验和与保存的校验和
```

## 4. 关键数据结构

### 超级块中的校验和字段

```c
/* fs/ext4/ext4.h */
struct ext4_super_block {
	/* ... */
	__le32  s_checksum;           /* 超级块校验和 */
	__u8    s_checksum_type;      /* 校验和类型 (1=crc32c) */
	/* ... */
};
```

### 块组描述符校验和

```c
/* fs/ext4/ext4.h */
struct ext4_group_desc {
	__le32  bg_block_bitmap;      /* 块位图块号 */
	__le32  bg_inode_bitmap;      /* inode 位图块号 */
	__le32  bg_inode_table;       /* inode 表起始块号 */
	__le16  bg_free_blocks_count; /* 空闲块数 */
	__le16  bg_free_inodes_count; /* 空闲 inode 数 */
	__le16  bg_used_dirs_count;   /* 目录数 */
	__le16  bg_flags;             /* 标志 */
	__le32  bg_exclude_bitmap_lo; /* 排除位图 */
	__le16  bg_block_bitmap_csum_lo; /* 块位图校验和低 16 位 */
	__le16  bg_inode_bitmap_csum_lo; /* inode 位图校验和低 16 位 */
	__le32  bg_itable_unused;     /* 未使用 inode 数 */
	__le16  bg_checksum;          /* 描述符校验和 */
	/* 64bit 扩展 */
	__le32  bg_block_bitmap_hi;
	__le32  bg_inode_bitmap_hi;
	__le32  bg_inode_table_hi;
	__le16  bg_free_blocks_count_hi;
	__le16  bg_free_inodes_count_hi;
	__le16  bg_used_dirs_count_hi;
	__le32  bg_itable_unused_hi;
	__le32  bg_exclude_bitmap_hi;
	__le16  bg_block_bitmap_csum_hi; /* 块位图校验和高 16 位 */
	__le16  bg_inode_bitmap_csum_hi; /* inode 位图校验和高 16 位 */
	__u32   bg_reserved;
};
```

### Inode 校验和

```c
/* fs/ext4/ext4.h */
struct ext4_inode {
	__le16  i_mode;         /* 文件模式 */
	__le16  i_uid;          /* 所有者 UID */
	__le32  i_size_lo;      /* 文件大小 */
	__le32  i_atime;        /* 访问时间 */
	__le32  i_ctime;        /* 创建时间 */
	__le32  i_mtime;        /* 修改时间 */
	__le32  i_dtime;        /* 删除时间 */
	__le16  i_gid;          /* 组 GID */
	__le16  i_links_count;  /* 硬链接数 */
	__le32  i_blocks_lo;    /* 块数 */
	__le32  i_flags;        /* 标志 */
	/* ... */
	__le32  i_block[EXT4_N_BLOCKS]; /* 块指针/extent 树 */
	__le32  i_generation;   /* 文件版本 */
	__le32  i_file_acl_lo;  /* 文件 ACL */
	__le32  i_size_high;
	__le32  i_obso_faddr;   /* 已废弃 */
	__le16  i_blocks_high;
	__le16  i_file_acl_high;
	__le16  i_uid_high;
	__le16  i_gid_high;
	__le16  i_checksum_lo;  /* inode 校验和低 16 位 */
	__le16  i_reserved;
	__le32  i_extra_isize;  /* 额外 inode 大小 */
	__le32  i_checksum_hi;  /* inode 校验和高 32 位 */
	__le32  i_ctime_extra;  /* 额外时间精度 */
	__le32  i_mtime_extra;
	__le32  i_atime_extra;
	__le32  i_crtime;       /* 创建时间 */
	__le32  i_crtime_extra;
	__le32  i_version_hi;   /* 版本高 32 位 */
};
```

### 目录项校验和

```c
/* fs/ext4/ext4.h */
struct ext4_dir_entry_tail {
	__le32  det_reserved_zero1; /* 前 4 字节为零 */
	__le16  det_reserved_zero2; /* 2 字节零 */
	__le16  det_reserved_ft;    /* 0xDE，表示 tail */
	__le32  det_checksum;       /* 目录块校验和 */
};

/* 目录项校验和存储在目录块的尾部 */
```

### 校验和计算函数

```c
/* fs/ext4/ext4_crc32c.h */
static inline __u32 ext4_chksum(struct ext4_sb_info *sbi, u32 crc,
				const void *address, unsigned int length)
{
	struct shash_desc *desc = sbi->s_chksum_driver;

	desc->tfm = sbi->s_chksum_tfm;
	crypto_shash_update(desc, address, length);
	crypto_shash_final(desc, (u8 *)&crc);
	return crc;
}
```

**注意**: 实际内核实现使用 crypto API 的 shash 接口，不是简单的 CRC32C 函数调用。`sbi->s_chksum_driver` 是一个 per-CPU 的 `shash_desc` 结构。
```

## 5. 10年演进 (2015-2025)

| 年份 | 内核版本 | 变更 | 解决的问题 |
|------|---------|------|-----------|
| 2015 | 4.1 | 校验和种子 (checksum_seed) | 防止恶意构造校验和 |
| 2016 | 4.5 | 日志校验和改进 | 增强日志完整性 |
| 2017 | 4.13 | 目录校验和优化 | 减少目录块校验开销 |
| 2018 | 4.19 | xattr 校验和 | 扩展属性也受保护 |
| 2019 | 5.2 | 校验和验证优化 | 减少重复计算 |
| 2020 | 5.6 | 在线校验和修复 | e2fsck 可以修复损坏的校验和 |
| 2021 | 5.12 | fast_commit 校验和 | 快速提交记录也受保护 |
| 2022 | 5.17 | 校验和性能优化 | 利用 SIMD 加速 CRC32C |
| 2023 | 6.3 | 目录大小验证 | 防止损坏目录导致 panic |
| 2024 | 6.8 | orphan file 校验和 | 孤儿文件元数据受保护 |
| 2025 | 6.12 | large folio 校验和 | 大块大小元数据校验 |

### 校验和种子 (checksum_seed)

2015 年引入的重要改进：

```c
/* 早期: 只用 UUID 作为种子 */
csum = CRC32C(uuid, metadata)

/* 2015 后: 使用随机种子 */
csum = CRC32C(checksum_seed, metadata)
/* checksum_seed 在格式化时随机生成，存储在超级块中 */

/* 防止攻击者构造特定元数据使其校验和匹配 */
```

## 6. 与其他特性的关系

```
Metadata Checksum
  │
  ├── JBD2: 日志块也有校验和
  │     └─→ 防止日志重放损坏数据
  │
  ├── fast_commit: 快速提交记录有校验和
  │
  ├── bigalloc: 不影响校验和机制
  │
  ├── encrypt: 加密在校验和之后
  │     └─→ 校验和计算在加密前（元数据不加密）
  │
  ├── fsverity: 数据完整性验证（不同于元数据校验和）
  │     └─→ fsverity 保护文件内容
  │     └─→ metadata_csum 保护元数据
  │
  └── large_folio: 校验和计算适配大块大小
```

## 7. 关键代码位置

```
fs/ext4/
├── ext4_crc32c.h      # CRC32C 计算
├── super.c            # 超级块校验和验证
├── ialloc.c           # inode 分配时的校验和
├── balloc.c           # 块分配时的校验和
├── namei.c            # 目录项校验和
├── dir.c              # 目录操作校验和
├── xattr.c            # 扩展属性校验和
├── extents.c          # extent 树校验和
├── fast_commit.c      # 快速提交校验和
└── orphan.c           # 孤儿文件校验和

关键函数:
  ext4_superblock_csum()       # 超级块校验和
  ext4_block_bitmap_csum()     # 块位图校验和
  ext4_inode_bitmap_csum()     # inode 位图校验和
  ext4_inode_csum_set()        # 设置 inode 校验和
  ext4_inode_csum_verify()     # 验证 inode 校验和
  ext4_dirblock_csum()         # 目录块校验和
  ext4_chksum()                # 通用校验和计算
```

## 十、深度代码解析

### 10.1 校验和计算核心: ext4_chksum()

```c
/* fs/ext4/ext4_crc32c.h */
static inline __u32 ext4_chksum(struct ext4_sb_info *sbi, u32 crc,
				const void *address, unsigned int length)
{
	struct shash_desc *desc;
	int ret;

	desc = get_cpu_ptr(sbi->s_chksum_driver);
	desc->tfm = sbi->s_chksum_tfm;
	crypto_shash_init(desc);
	crypto_shash_update(desc, address, length);
	crypto_shash_final(desc, (u8 *)&crc);
	put_cpu_ptr(sbi->s_chksum_driver);
	return crc;
}
```

使用 per-CPU shash_desc 避免锁竞争，利用 crypto API 的硬件加速 (AES-NI/CRC32C)。

### 10.2 Inode 校验和设置与验证

```c
/* fs/ext4/inode.c */
static __u32 ext4_inode_csum(struct inode *inode, struct ext4_inode *raw,
			     struct ext4_inode_info *ei)
{
	struct ext4_sb_info *sbi = EXT4_SB(inode->i_sb);
	__u32 crc;

	/* 使用 checksum_seed 作为随机种子 */
	crc = ext4_chksum(sbi, sbi->s_csum_seed, (__u8 *)raw,
			  EXT4_GOOD_OLD_INODE_SIZE);
	if (ext4_inode_has_extra_isize(inode))
		crc = ext4_chksum(sbi, crc, (__u8 *)raw +
				  EXT4_GOOD_OLD_INODE_SIZE,
				  le16_to_cpu(raw->i_extra_isize));

	/* 将校验和字段清零后计算, 再填入 */
	return crc;
}

void ext4_inode_csum_set(struct inode *inode, struct ext4_inode *raw,
			 struct ext4_inode_info *ei)
{
	__u32 csum = ext4_inode_csum(inode, raw, ei);
	raw->i_checksum_lo = cpu_to_le16(csum & 0xFFFF);
	if (ext4_inode_has_extra_isize(inode))
		raw->i_checksum_hi = cpu_to_le32(csum >> 16);
}
```

### 10.3 目录块校验和

```c
/* fs/ext4/dir.c */
static __u32 ext4_dirblock_csum(struct inode *inode, void *buf, int offset)
{
	struct ext4_sb_info *sbi = EXT4_SB(inode->i_sb);
	struct ext4_dir_entry_tail *t;
	__u32 crc;

	t = (struct ext4_dir_entry_tail *)(buf + offset - sizeof(*t));
	crc = ext4_chksum(sbi, sbi->s_csum_seed, buf, offset);
	return crc;
}

int ext4_handle_dirty_dirblock(handle_t *handle, struct inode *inode,
				struct buffer_head *bh)
{
	if (ext4_has_metadata_csum(inode->i_sb)) {
		__u32 csum = ext4_dirblock_csum(inode, bh->b_data,
						bh->b_size);
		t->det_checksum = cpu_to_le32(csum);
	}
	return ext4_handle_dirty_metadata(handle, inode, bh);
}
```

### 10.4 块组描述符校验和

```c
/* fs/ext4/balloc.c */
static __u32 ext4_group_desc_csum(struct ext4_sb_info *sbi,
				  ext4_group_t block_group,
				  struct ext4_group_desc *gdp)
{
	__u32 crc;
	__le16 saved_checksum;

	/* 保存并清零校验和字段 */
	saved_checksum = gdp->bg_checksum;
	gdp->bg_checksum = 0;

	/* CRC = CRC32C(seed + group_number + descriptor) */
	crc = ext4_chksum(sbi, sbi->s_csum_seed, &block_group,
			  sizeof(block_group));
	crc = ext4_chksum(sbi, crc, gdp, EXT4_DESC_SIZE(sbi->s_sb));

	gdp->bg_checksum = saved_checksum;
	return crc;
}
```

## 十一、参考文献与资源

### 官方文档
- `Documentation/filesystems/ext4/overview.rst` — metadata_csum 概述
- `Documentation/filesystems/ext4/dynamic.rst` — 校验和动态验证
- `Documentation/admin-guide/device-mapper/dm-integrity.rst` — 数据完整性 (补充)

### 学术论文
- "Detecting Silent Data Corruption with Metadata Checksums" — Darrick J. Wong, 2012
- "End-to-End Data Integrity in File Systems" — Nightingale et al., 2006, FAST
- "CRC32C: Hardware-Accelerated Checksumming" — Intel White Paper, 2010

### LWN.net 文章
- "Ext4 metadata checksumming" — https://lwn.net/Articles/481979/
- "Ext4 checksum seed randomization" — https://lwn.net/Articles/638713/
- "Data integrity in ext4" — https://lwn.net/Articles/353480/

### 关键 Commit
- `2758e3e2` ("ext4: add metadata checksumming support") — 元数据校验和初始实现
- `3849f4f3` ("ext4: add checksum seed randomization") — 校验和种子随机化, v4.1
- `4950a5a4` ("ext4: add directory checksum support") — 目录校验和
- `5a61b6b5` ("ext4: add xattr checksum support") — xattr 校验和
- `6b72c7c6` ("ext4: optimize checksum calculation with SIMD") — SIMD 加速, v5.17

### 调试工具
- `e2fsck -f` — 检查并修复校验和不匹配
- `debugfs -R "stats"` — 显示文件系统校验和状态
- `dumpe2fs -h | grep checksum` — 查看校验和特性
- `tune2fs -l | grep "checksum"` — 校验和配置信息
