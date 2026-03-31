---
name: "ext4-journal-disk"
description: "ext4文件系统日志磁盘结构专家。当用户询问ext4日志磁盘布局、journal_superblock、日志记录格式、JBD2结构时调用此技能。"
---

# ext4 日志磁盘结构

## 一、概述

ext4 使用 JBD2 (Journaling Block Device 2) 作为日志系统，保证文件系统的一致性。

---

## 二、日志位置

### 2.1 内部日志

日志位于文件系统内部的一个 inode：

```c
/* 超级块中的日志 inode */
s_journal_inum = 8;  /* 通常使用 inode 8 */
```

### 2.2 外部日志

日志可以位于独立的块设备：

```bash
# 创建外部日志
mkfs.ext4 -O journal_dev /dev/sdY
mkfs.ext4 -J device=/dev/sdY /dev/sdX
```

### 2.3 查看日志信息

```bash
dumpe2fs /dev/sdX | grep -i journal
```

---

## 三、日志超级块

### 3.1 结构定义

**位置**: `fs/jbd2/journal.h`

```c
struct journal_superblock_s {
    __be32  s_header.h_magic;       /* 魔数 JBD2_MAGIC_NUMBER */
    __be32  s_header.h_blocktype;   /* 块类型 */
    __be32  s_blocksize;            /* 块大小 */
    __be32  s_maxlen;               /* 日志最大长度 (块数) */
    __be32  s_first;                /* 第一个日志块号 */
    __be32  s_sequence;             /* 事务序列号 */
    __be32  s_start;                /* 日志起始位置 */
    __be32  s_errno;                /* 错误号 */
    __be32  s_feature_compat;       /* 兼容特性 */
    __be32  s_feature_incompat;     /* 不兼容特性 */
    __be32  s_feature_ro_compat;    /* 只读兼容特性 */
    __u8    s_uuid[16];             /* UUID */
    __be32  s_nr_users;             /* 用户数 */
    __be32  s_dynsuper;             /* 动态超级块 */
    __be32  s_max_transaction;      /* 最大事务 */
    __be32  s_max_trans_data;       /* 最大事务数据 */
    __u8    s_padding[1784];        /* 填充 */
    __be32  s_checksum;             /* 校验和 */
};
```

### 3.2 魔数

```c
#define JBD2_MAGIC_NUMBER  0xc03b3998U
```

### 3.3 块类型

```c
#define JBD2_SUPERBLOCK_V1    1   /* 超级块 V1 */
#define JBD2_SUPERBLOCK_V2    2   /* 超级块 V2 */
#define JBD2_DESCRIPTOR_BLOCK 3   /* 描述符块 */
#define JBD2_COMMIT_BLOCK     4   /* 提交块 */
#define JBD2_REVOKE_BLOCK     5   /* 撤销块 */
```

---

## 四、日志记录类型

### 4.1 描述符块

描述符块记录事务中包含的数据块：

```c
struct journal_header_s {
    __be32  h_magic;        /* 魔数 */
    __be32  h_blocktype;    /* 块类型 */
    __be32  h_sequence;     /* 序列号 */
};

/* 数据块标签 */
struct journal_block_tag_s {
    __be32  t_blocknr;      /* 块号 (低 32 位) */
    __be16  t_checksum;     /* 校验和 */
    __be16  t_flags;        /* 标志 */
    __be32  t_blocknr_high; /* 块号 (高 32 位) */
};
```

### 4.2 数据块

数据块包含实际的元数据或数据内容。

### 4.3 提交块

提交块标记事务完成：

```c
struct commit_header {
    __be32  h_magic;        /* 魔数 */
    __be32  h_blocktype;    /* 块类型 */
    __be32  h_sequence;     /* 序列号 */
    __u8    h_chksum_type;  /* 校验和类型 */
    __u8    h_chksum_size;  /* 校验和大小 */
    __u8    h_padding[2];
    __be32  h_chksum[JBD2_CHECKSUM_BYTES];
    __u8    h_commit_sec[8];
    __be32  h_commit_nsec;
};
```

### 4.4 撤销块

撤销块记录需要撤销的块：

```c
struct jbd2_journal_revoke_header_s {
    __be32  r_count;        /* 记录数 */
    __be32  r_header;       /* 头部 */
};
```

---

## 五、日志布局

### 5.1 日志循环

日志是循环使用的：

```
日志头 (s_start) ──→ 事务1 ──→ 事务2 ──→ ... ──→ 日志尾
                                              │
                                              └── 回绕到日志头
```

### 5.2 事务布局

```
事务 N:
┌─────────────────────────────────────────────────────────────┐
│  描述符块                                                    │
│  - h_blocktype = JBD2_DESCRIPTOR_BLOCK                      │
│  - h_sequence = N                                           │
│  - 块标签列表                                                │
├─────────────────────────────────────────────────────────────┤
│  数据块 1                                                    │
│  - 元数据块内容                                              │
├─────────────────────────────────────────────────────────────┤
│  数据块 2                                                    │
│  - 元数据块内容                                              │
├─────────────────────────────────────────────────────────────┤
│  ...                                                         │
├─────────────────────────────────────────────────────────────┤
│  提交块                                                      │
│  - h_blocktype = JBD2_COMMIT_BLOCK                          │
│  - h_sequence = N                                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 六、日志特性

### 6.1 兼容特性

```c
#define JBD2_FEATURE_COMPAT_CHECKSUM    0x00000001
```

### 6.2 不兼容特性

```c
#define JBD2_FEATURE_INCOMPAT_REVOKE    0x00000001
#define JBD2_FEATURE_INCOMPAT_64BIT     0x00000002
#define JBD2_FEATURE_INCOMPAT_ASYNC_COMMIT 0x00000004
#define JBD2_FEATURE_INCOMPAT_CSUM_V2   0x00000008
#define JBD2_FEATURE_INCOMPAT_CSUM_V3   0x00000010
```

---

## 七、日志大小计算

### 7.1 默认日志大小

```c
/* 默认日志大小计算 */
if (fs_size < 128MB)
    journal_size = 4MB;
else if (fs_size < 512MB)
    journal_size = 16MB;
else if (fs_size < 4GB)
    journal_size = 32MB;
else if (fs_size < 64GB)
    journal_size = 128MB;
else
    journal_size = 256MB;
```

### 7.2 指定日志大小

```bash
# 创建指定大小的日志
mkfs.ext4 -J size=256 /dev/sdX
```

---

## 八、日志恢复

### 8.1 恢复流程

```
1. 读取日志超级块
   │
   ├── 检查 s_start 是否为 0
   │   │
   │   ├── s_start = 0: 日志干净，无需恢复
   │   │
   │   └── s_start != 0: 需要恢复
   │
   ├── 2. 扫描日志
   │       │
   │       ├── 找到所有完整事务
   │       └── 识别未完成事务
   │
   ├── 3. 重放事务
   │       │
   │       ├── 重放描述符块
   │       ├── 写入数据块
   │       └── 确认提交块
   │
   └── 4. 更新日志超级块
```

### 8.2 恢复函数

```c
int jbd2_journal_recover(journal_t *journal)
{
    /* 1. 扫描日志 */
    err = jbd2_journal_scan_revoke(journal);

    /* 2. 重放事务 */
    err = jbd2_journal_replay(journal);

    /* 3. 更新超级块 */
    jbd2_journal_update_superblock(journal, 1);

    return 0;
}
```

---

## 九、调试命令

### 9.1 查看日志信息

```bash
# 查看日志信息
dumpe2fs /dev/sdX | grep -i journal

# 输出示例
Journal inode:            8
Journal backup:           inode blocks
Journal features:         journal_incompat_revoke
Journal size:             128M
Journal length:           32768
Journal sequence:         0x00000001
Journal start:            0
```

### 9.2 debugfs

```bash
debugfs /dev/sdX

# 查看日志 inode
debugfs: stat 8
```

---

## 十、代码位置

| 功能 | 文件路径 |
|------|----------|
| 结构定义 | `fs/jbd2/journal.h` |
| 日志超级块 | `fs/jbd2/journal.c` |
| 日志恢复 | `fs/jbd2/recovery.c` |
| 日志事务 | `fs/jbd2/transaction.c` |

---

## 十一、JBD2 关键修复 (2025-2026)

### 11.1 软死锁修复

**问题**: `jbd2_log_do_checkpoint()` 可能触发软死锁 (softlockup)

```
watchdog: BUG: soft lockup - CPU#3 stuck for 156s!
Call trace:
 native_queued_spin_lock_slowpath
 jbd2_log_do_checkpoint
 __jbd2_log_wait_for_space
```

**原因**: `jbd2_log_do_checkpoint()` 依赖 `__flush_batch()` 或 `wait_on_buffer()` 触发调度，如果这些函数不睡眠，会导致软死锁。

**修复**: 显式调用 `cond_resched()` 避免软死锁。

### 11.2 检查点 I/O 优先级

```c
/* jbd2: increase IO priority of checkpoint */
/* 提高检查点的 I/O 优先级 */
/* 确保检查点尽快完成，释放日志空间 */
```

### 11.3 释放块前等待 I/O 完成

**问题**: `jbd2_journal_forget()` 在释放元数据块时，不等待正在进行的 I/O 完成。

**风险**:
1. 如果缓冲区正在写回，不等待完成就重新分配，可能导致旧数据写入新分配的块
2. 并发写回可能在 `clear_buffer_dirty()` 之前发生

**修复**: 显式确保所有正在进行的 I/O 完成后再释放块。

### 11.4 日志超级块校验和修复

**问题**: 只读挂载时复制文件系统，可能导致日志超级块校验和不匹配。

**原因**: `set_journal_csum_feature_set()` 修改日志超级块数据但未更新校验和。

**修复**: 修改日志超级块后更新内存中的校验和。

### 11.5 其他 JBD2 修复

| 修复 | 说明 |
|------|------|
| `avoid bug_on in jbd2_journal_get_create_access()` | 文件系统损坏时避免 BUG_ON |
| `store more accurate errno in superblock` | 在超级块中存储更准确的错误号 |
| `use a per-journal lock_class_key` | 每个 journal 使用独立的 lock_class_key |
| `use a weaker annotation in journal handling` | 使用更弱的锁注解 |
| `fix data-race and null-ptr-deref` | 修复数据竞争和空指针解引用 |
| `convert jbd2_journal_blocks_per_page() to support large folio` | 支持 large folio |

---

## 十二、参考资源

- ext4 官方文档: `Documentation/filesystems/ext4/`
- JBD2 内核代码: `fs/jbd2/`
