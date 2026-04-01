---
name: "ext4-journal-disk"
description: "ext4文件系统日志磁盘结构专家。当用户询问ext4日志磁盘布局、journal_superblock、日志记录格式、JBD2结构时调用此技能。"
---

# ext4 JBD2 Disk Structures

## 〇、为什么需要这个机制？

为什么需要 JBD2 日志？没有日志，文件系统崩溃后需要运行 e2fsck 扫描整个文件系统（可能数小时）。JBD2 通过"先写日志再写数据"的方式，保证崩溃后只需重放日志即可恢复（通常几秒）。ext4 的 JBD2 经历了多次演进：从 v1 到 v2（64 位支持），从同步提交到异步提交（async_commit），从完整校验和到 32 位完整校验和（CSUM_V3）。

日志是 ext4 可靠性的基石。在没有日志的文件系统中，更新一个文件可能需要修改 inode、数据块、目录项、位图等多个位置。如果在这个过程中发生崩溃，文件系统就会处于不一致状态。JBD2 通过原子事务保证：要么所有修改都完成，要么都不完成。

没有 JBD2，ext4 就无法提供崩溃一致性保证，生产环境中的文件系统损坏风险将大幅增加。

---

## 一、JBD2 磁盘块类型 (jbd2.h:122-126)

```c
#define JBD2_DESCRIPTOR_BLOCK	1
#define JBD2_COMMIT_BLOCK	2
#define JBD2_SUPERBLOCK_V1	3
#define JBD2_SUPERBLOCK_V2	4
#define JBD2_REVOKE_BLOCK	5
```

---

## 二、通用日志头 (jbd2.h:131-136)

```c
typedef struct journal_header_s {
	__be32		h_magic;	/* JBD2_MAGIC_NUMBER 0xc03b3998 */
	__be32		h_blocktype;
	__be32		h_sequence;	/* Transaction sequence number */
} journal_header_t;
```

---

## 三、日志超级块 (jbd2.h:201-260)

```c
typedef struct journal_superblock_s {
	journal_header_t s_header;
	__be32  s_blocksize;
	__be32  s_maxlen;
	__be32  s_first;
	__be32  s_sequence;
	__be32  s_start;
	__be32  s_errno;
	__be32  s_feature_compat;
	__be32  s_feature_incompat;
	__be32  s_feature_ro_compat;
	__u8    s_uuid[16];
	__be32  s_nr_users;
	__be32  s_dynsuper;
	__be32  s_max_transaction;
	__be32  s_max_trans_data;
	__be32  s_checksum_type;
	__be32  s_padding2[55];
	__be32  s_checksum;
} journal_superblock_t;
```

### JBD2 特性标志

```c
#define JBD2_FEATURE_COMPAT_CHECKSUM		0x00000001
#define JBD2_FEATURE_INCOMPAT_REVOKE		0x00000001
#define JBD2_FEATURE_INCOMPAT_64BIT		0x00000002
#define JBD2_FEATURE_INCOMPAT_ASYNC_COMMIT	0x00000004
#define JBD2_FEATURE_INCOMPAT_CSUM_V2		0x00000008
#define JBD2_FEATURE_INCOMPAT_CSUM_V3		0x00000010
#define JBD2_FEATURE_INCOMPAT_FAST_COMMIT	0x00000020
```

---

## 四、Descriptor Block Tags

### journal_block_tag3_t (CSUM_V3, full 32-bit checksum)

```c
typedef struct journal_block_tag3_s {
	__be32		t_blocknr;
	__be32		t_flags;
	__be32		t_blocknr_high;
	__be32		t_checksum;	/* crc32c(uuid+seq+block) */
} journal_block_tag3_t;
```

### journal_block_tag_t (CSUM_V2, truncated 16-bit checksum)

```c
typedef struct journal_block_tag_s {
	__be32		t_blocknr;
	__be16		t_checksum;	/* truncated crc32c */
	__be16		t_flags;
	__be32		t_blocknr_high;
} journal_block_tag_t;
```

### Tag Flags

```c
#define JBD2_FLAG_ESCAPE		1	/* on-disk block is escaped */
#define JBD2_FLAG_SAME_UUID		2	/* block has same uuid */
#define JBD2_FLAG_DELETED		4	/* block deleted by this transaction */
#define JBD2_FLAG_LAST_TAG		8	/* last tag in this descriptor */
#define JBD2_FLAG_UUID			16	/* UUID follows */
```

---

## 五、Commit Block (jbd2.h:167-177)

```c
struct commit_header {
	__be32		h_magic;
	__be32          h_blocktype;
	__be32          h_sequence;
	unsigned char   h_chksum_type;
	unsigned char   h_chksum_size;
	unsigned char 	h_padding[2];
	__be32 		h_chksum[JBD2_CHECKSUM_BYTES];
	__be64		h_commit_sec;
	__be32		h_commit_nsec;
};
```

---

## 六、Revoke Block

```c
typedef struct journal_revoke_header_s {
	journal_header_t r_header;
	__be32  r_count;	/* Count of bytes used in the block */
} journal_revoke_header_t;
```

---

## 七、Fast Commit 磁盘结构

Fast commit 使用 JBD2_FEATURE_INCOMPAT_FAST_COMMIT (0x20)。

### FC 头 (fs/ext4/fast_commit.h)

```c
struct ext4_fc_tl {
	__le16 fc_tag;
	__le16 fc_len;
};

struct ext4_fc_head {
	__le32 fc_features;
	__le32 fc_magic;
	__le32 fc_cid;
	__le32 fc_subtid;
	__le32 fc_num_blks;
	__le32 fc_committed_tid;
};
```

### FC Tags

```c
#define EXT4_FC_TAG_ADD_RANGE		1
#define EXT4_FC_TAG_DEL_RANGE		2
#define EXT4_FC_TAG_LINK		3
#define EXT4_FC_TAG_UNLINK		4
#define EXT4_FC_TAG_CREAT		5
#define EXT4_FC_TAG_INODE		6
#define EXT4_FC_TAG_PAD			7
#define EXT4_FC_TAG_TAIL		8
#define EXT4_FC_TAG_ADD_RANGE_EXT	9
```

### FC Tail

```c
struct ext4_fc_tail {
	__le32 fc_crc;
	__le32 fc_subtid;
};
```

---

## 八、JBD2 关键修复 (10年内核演进)

### 8.1 Softlockup Fix
- 长事务在 journal commit 时可能触发 softlockup
- 引入 cond_resched() 在 commit 循环中

### 8.2 Checkpoint IO Priority
- checkpoint I/O 使用独立 I/O 优先级
- 避免与正常 I/O 竞争

### 8.3 Checksum Fix
- commit block checksum 计算修复
- journal_block_tag3_t 引入 32-bit 完整校验和

### 8.4 Data-Race Fix
- jbd2 事务状态并发访问修复
- 使用 READ_ONCE/WRITE_ONCE 保护共享状态

### 8.5 Checkpoint Shrinker
- 为 checkpointed buffers 引入 shrinker
- 自动回收已提交的 journal 内存

---

## 九、关键代码位置

| 功能 | 文件 |
|------|------|
| JBD2 磁盘结构 | include/linux/jbd2.h |
| JBD2 实现 | fs/jbd2/ |
| Fast commit 磁盘结构 | fs/ext4/fast_commit.h |
| ext4 日志集成 | fs/ext4/super.c, fs/ext4/inode.c |
