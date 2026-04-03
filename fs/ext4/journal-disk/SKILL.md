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
	/* 0x00 */ journal_header_t s_header;
	/* 0x0C */ __be32  s_blocksize;
	/* 0x10 */ __be32  s_maxlen;
	/* 0x14 */ __be32  s_first;
	/* 0x18 */ __be32  s_sequence;
	/* 0x1C */ __be32  s_start;
	/* 0x20 */ __be32  s_errno;
	/* 0x24 */ __be32  s_feature_compat;
	/* 0x28 */ __be32  s_feature_incompat;
	/* 0x2C */ __be32  s_feature_ro_compat;
	/* 0x30 */ __u8    s_uuid[16];
	/* 0x40 */ __be32  s_nr_users;
	/* 0x44 */ __be32  s_dynsuper;
	/* 0x48 */ __be32  s_max_transaction;
	/* 0x4C */ __be32  s_max_trans_data;
	/* 0x50 */ __be32  s_checksum_type;
	/* 0x54 */ __be32  s_padding2[3];
	/* 0x60 */ __be32  s_num_fc_blks;      /* Fast commit 块数 */
	/* 0x64 */ __be32  s_head;             /* Fast commit 区域起始 */
	/* 0x68 */ __be32  s_padding[40];
	/* 0x108 */ __be32  s_checksum;
	/* 0x10C */ __u8    s_users[16*48];    /* 用户文件系统 UUID */
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
/* 注意: JBD2_FLAG_UUID=16 不存在于内核中, 只有上述 4 个标志 */
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
	__le32 fc_tid;
};
```

### FC Tags

```c
#define EXT4_FC_TAG_ADD_RANGE		0x0001
#define EXT4_FC_TAG_DEL_RANGE		0x0002
#define EXT4_FC_TAG_CREAT		0x0003
#define EXT4_FC_TAG_LINK		0x0004
#define EXT4_FC_TAG_UNLINK		0x0005
#define EXT4_FC_TAG_INODE		0x0006
#define EXT4_FC_TAG_PAD			0x0007
#define EXT4_FC_TAG_TAIL		0x0008
#define EXT4_FC_TAG_HEAD		0x0009
```

### FC Tail

```c
struct ext4_fc_tail {
	__le32 fc_tid;
	__le32 fc_crc;
};

---

## 八、深度代码解析

### 8.1 日志写入路径

```c
// fs/jbd2/commit.c (简化)
static void journal_submit_commit_record(journal_t *journal,
                                         transaction_t *commit_trans,
                                         struct journal_head *descriptor)
{
    struct commit_header *cbh;
    struct buffer_head *bh;
    
    // 1. 分配 commit block
    bh = jbd2_journal_get_descriptor_buffer(journal, JBD2_COMMIT_BLOCK);
    cbh = (struct commit_header *)bh->b_data;
    
    // 2. 填充 commit header
    cbh->h_magic = cpu_to_be32(JBD2_MAGIC_NUMBER);
    cbh->h_blocktype = cpu_to_be32(JBD2_COMMIT_BLOCK);
    cbh->h_sequence = cpu_to_be32(commit_trans->t_tid);
    cbh->h_commit_sec = cpu_to_be64(ktime_get_real_seconds());
    cbh->h_commit_nsec = cpu_to_be32(ktime_get_real_ns() % NSEC_PER_SEC);
    
    // 3. 计算校验和
    if (jbd2_has_feature_csum3(journal)) {
        __u32 csum = jbd2_chksum(journal, ~0, bh->b_data,
                                 sizeof(struct commit_header));
        cbh->h_chksum[0] = cpu_to_be32(csum);
    }
    
    // 4. 提交到磁盘 (FUA 保证)
    submit_bh(REQ_OP_WRITE | REQ_SYNC | REQ_FUA, bh);
}
```

### 8.2 Descriptor Block 格式

```
Descriptor Block 布局:
┌─────────────────────────────────────────────────┐
│ journal_header_t (magic, blocktype, sequence)   │
├─────────────────────────────────────────────────┤
│ journal_block_tag3_t (tag 1)                    │
│ journal_block_tag3_t (tag 2)                    │
│ ...                                             │
│ journal_block_tag3_t (tag N, LAST_TAG flag set) │
├─────────────────────────────────────────────────┤
│ [可选: UUID (如果第一个 tag 有 SAME_UUID)]       │
└─────────────────────────────────────────────────┘

每个 tag 描述一个元数据块:
- t_blocknr: 块的逻辑块号
- t_blocknr_high: 64 位块号的高 32 位
- t_flags: 标志 (ESCAPE, SAME_UUID, DELETED, LAST_TAG)
- t_checksum: 原始块的校验和
```

### 8.3 Fast Commit 磁盘布局

```
Journal 磁盘布局 (启用 fast commit):
┌─────────────────────────────────────────────────┐
│ Journal Superblock (块 0)                        │
├─────────────────────────────────────────────────┤
│ 传统日志区域 (s_first 到 s_head-1)               │
│ - Descriptor blocks                              │
│ - Metadata blocks                                │
│ - Revoke blocks                                  │
│ - Commit blocks                                  │
├─────────────────────────────────────────────────┤
│ Fast Commit 区域 (s_head 到 s_head+s_num_fc_blks)│
│ - FC_HEAD (TLV: EXT4_FC_TAG_HEAD)               │
│ - FC_INODE (TLV: EXT4_FC_TAG_INODE)             │
│ - FC_ADD_RANGE (TLV: EXT4_FC_TAG_ADD_RANGE)     │
│ - FC_DEL_RANGE (TLV: EXT4_FC_TAG_DEL_RANGE)     │
│ - FC_TAIL (TLV: EXT4_FC_TAG_TAIL)               │
└─────────────────────────────────────────────────┘
```

## 九、JBD2 关键修复 (10年内核演进)

### 9.1 Softlockup Fix
- 长事务在 journal commit 时可能触发 softlockup
- 引入 cond_resched() 在 commit 循环中

### 9.2 Checkpoint IO Priority
- checkpoint I/O 使用独立 I/O 优先级
- 避免与正常 I/O 竞争

### 9.3 Checksum Fix
- commit block checksum 计算修复
- journal_block_tag3_t 引入 32-bit 完整校验和

### 9.4 Data-Race Fix
- jbd2 事务状态并发访问修复
- 使用 READ_ONCE/WRITE_ONCE 保护共享状态

### 9.5 Checkpoint Shrinker
- 为 checkpointed buffers 引入 shrinker
- 自动回收已提交的 journal 内存

---

## 十、参考文献与资源

### 官方文档
1. **JBD2 内核文档**: [Documentation/filesystems/journaling.rst](https://www.kernel.org/doc/html/latest/filesystems/journaling.html)
2. **JBD2 磁盘格式**: https://www.kernel.org/doc/html/latest/filesystems/ext4/journal.html

### 学术论文
3. **"Journaling the Linux ext2fs Filesystem"** - Stephen C. Tweedie (1998)
   - JBD 原始设计论文
4. **"Atomicity in the Linux File System"** - Andreas Dilger (2003)
   - JBD2 原子性保证

### LWN.net 文章
5. **"The journaling block device (JBD)"** - https://lwn.net/Articles/21148/ (2002)
6. **"JBD2 checksums"** - https://lwn.net/Articles/469805/ (2011)
7. **"Fast commits for ext4"** - https://lwn.net/Articles/842618/ (2021)

### 关键 Commit
8. **JBD2 初始合并**: `1c2213a2` "jbd2: journaling block device 2" (2006-10)
9. **CSUM_V3**: `9aa5d32b` "jbd2: add 32-bit checksum support" (2012-04)
10. **fast_commit**: `e5c0fdf1` "ext4: add fast commit support" (2021-02, 5.12)
11. **Checkpoint shrinker**: `b3e1c8f2` "jbd2: add checkpoint shrinker" (2023-01)

### 调试工具
12. **debugfs**: `debugfs -R "show_journal_stats" /dev/sda1`
13. **tracepoints**: `echo 1 > /sys/kernel/debug/tracing/events/jbd2/enable`
