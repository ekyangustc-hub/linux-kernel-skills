---
name: "ext4-runtime"
description: "ext4文件系统运行时行为专家。当用户询问ext4文件操作流程(新建/删除/写入)、日志一致性保证、JBD2事务机制、revoke机制、孤儿列表、断电恢复时调用此技能。"
---

# ext4 运行时行为

## 一、概述

ext4 的运行时行为包括文件操作流程、日志一致性保证、JBD2 事务机制等。

---

## 二、JBD2 日志系统

### 2.1 JBD2 核心概念

| 概念 | 说明 |
|------|------|
| Journal | 日志设备，存储元数据变更 |
| Transaction | 事务，一组原子操作 |
| Handle | 句柄，事务中的单个操作 |
| Buffer Head | 缓冲区头，管理块缓冲 |

### 2.2 核心数据结构

```c
/* 日志结构 */
struct journal_s {
    unsigned long       j_flags;        /* 标志 */
    unsigned int        j_errno;        /* 错误号 */
    struct buffer_head  *j_sb_buffer;   /* 超级块缓冲 */
    struct journal_superblock_s *j_superblock;/* 超级块 */
    spinlock_t          j_state_lock;   /* 状态锁 */
    struct mutex        j_barrier;      /* 屏障互斥 */
    transaction_t       *j_running_transaction;/* 运行中事务 */
    transaction_t       *j_committing_transaction;/* 提交中事务 */
    transaction_t       *j_checkpoint_transactions;/* 检查点事务 */
    wait_queue_head_t   j_wait_transaction_locked;/* 等待事务锁 */
    wait_queue_head_t   j_wait_logspace;/* 等待日志空间 */
    wait_queue_head_t   j_wait_done_commit;/* 等待提交完成 */
    struct list_head    j_chkpt_bhs;    /* 检查点缓冲 */
    struct mutex        j_checkpoint_mutex;/* 检查点互斥 */
    unsigned int        j_head;         /* 日志头 */
    unsigned int        j_tail;         /* 日志尾 */
    unsigned int        j_free;         /* 空闲日志块 */
    unsigned int        j_first;        /* 第一个日志块 */
    unsigned int        j_last;         /* 最后一个日志块 */
};

/* 事务结构 */
struct transaction_s {
    unsigned long       t_tid;          /* 事务 ID */
    unsigned long       t_expires;      /* 过期时间 */
    unsigned int        t_state;        /* 状态 */
    unsigned int        t_log_start;    /* 日志起始位置 */
    unsigned int        t_log_blocks;   /* 日志块数 */
    unsigned int        t_outstanding_credits;/* 未完成信用 */
    unsigned int        t_outstanding_revokes;/* 未完成撤销 */
    struct journal_s    *t_journal;     /* 所属日志 */
    struct list_head    t_buffers;      /* 缓冲链表 */
    struct list_head    t_forget;       /* 忘记链表 */
    struct list_head    t_checkpoint_list;/* 检查点链表 */
    struct list_head    t_checkpoint_io_list;/* 检查点 IO 链表 */
};

/* 句柄结构 */
struct handle_s {
    transaction_t       *h_transaction; /* 所属事务 */
    int                 h_buffer_credits;/* 缓冲信用 */
    int                 h_ref;          /* 引用计数 */
    unsigned int        h_jdata: 1;     /* 数据日志 */
    unsigned int        h_sync: 1;      /* 同步 */
    unsigned int        h_err: 1;       /* 错误 */
};
```

### 2.3 事务状态

```c
enum {
    T_RUNNING,          /* 运行中 */
    T_LOCKED,           /* 已锁定 */
    T_FLUSH,            /* 刷新中 */
    T_COMMIT,           /* 提交中 */
    T_COMMIT_DFLUSH,    /* 数据刷新中 */
    T_COMMIT_JFLUSH,    /* 日志刷新中 */
    T_COMMIT_CALLBACK,  /* 回调中 */
    T_FINISHED          /* 已完成 */
};
```

---

## 三、事务流程

### 3.1 事务生命周期

```
1. 开始事务
   │
   ├── journal_start()
   │   └── 获取 handle
   │
2. 执行操作
   │
   ├── journal_get_write_access()
   │   └── 获取缓冲写权限
   │
   ├── 修改元数据
   │
   └── journal_dirty_metadata()
       └── 标记缓冲为脏
   │
3. 提交事务
   │
   ├── journal_stop()
   │   └── 释放 handle
   │
   └── journal_commit_transaction()
       ├── 刷新数据
       ├── 写入日志
       └── 更新日志尾
```

### 3.2 提交流程

```
journal_commit_transaction()
    │
    ├── 1. 等待所有 handle 完成
    │
    ├── 2. 刷新数据到磁盘
    │       (如果使用 data=ordered)
    │
    ├── 3. 写入日志记录
    │       ├── 描述符块
    │       ├── 数据块
    │       └── 提交块
    │
    ├── 4. 更新日志超级块
    │
    └── 5. 唤醒等待者
```

---

## 四、Revoke 机制

### 4.1 概述

Revoke 机制用于撤销旧的日志记录，防止恢复时覆盖新数据。

### 4.2 Revoke 记录

```c
struct journal_revoke_header_s {
    __be32              r_count;        /* 记录数 */
    __be32              r_header;       /* 头部 */
};
```

### 4.3 Revoke 流程

```
1. 块被重新分配
   │
   ├── journal_revoke()
   │   └── 添加到 revoke 表
   │
2. 提交事务时
   │
   ├── 写入 revoke 记录到日志
   │
   └── 3. 恢复时
       │
       └── 跳过被 revoke 的块
```

---

## 五、孤儿列表

### 5.1 概述

孤儿列表用于跟踪被删除但仍有进程打开的文件。

### 5.2 孤儿列表结构

```
Super Block:
    s_last_orphan ──→ inode A ──→ inode B ──→ NULL
                        │           │
                        │           └─ i_orphan
                        └─ i_orphan
```

### 5.3 孤儿 inode 处理

```c
/* 添加到孤儿列表 */
int ext4_orphan_add(handle_t *handle, struct inode *inode)
{
    struct ext4_sb_info *sbi = EXT4_SB(inode->i_sb);

    /* 1. 设置孤儿标志 */
    EXT4_I(inode)->i_orphan = sbi->s_es->s_last_orphan;
    sbi->s_es->s_last_orphan = cpu_to_le32(inode->i_ino);

    /* 2. 标记超级块为脏 */
    ext4_mark_super_dirty(inode->i_sb);

    return 0;
}

/* 从孤儿列表移除 */
int ext4_orphan_del(handle_t *handle, struct inode *inode)
{
    /* 1. 从链表中移除 */
    /* 2. 更新超级块 */
    return 0;
}
```

---

## 六、数据日志模式

### 6.1 三种模式

| 模式 | 说明 | 性能 | 安全性 |
|------|------|------|--------|
| data=journal | 数据和元数据都写入日志 | 最低 | 最高 |
| data=ordered | 元数据写入日志，数据在提交前写入 | 中等 | 中等 |
| data=writeback | 元数据写入日志，数据不保证 | 最高 | 最低 |

### 6.2 模式选择

```bash
# 挂载时指定
mount -o data=journal /dev/sdX /mnt
mount -o data=ordered /dev/sdX /mnt
mount -o data=writeback /dev/sdX /mnt
```

---

## 七、断电恢复

### 7.1 恢复流程

```
1. 挂载文件系统
   │
   ├── 检查超级块状态
   │   │
   │   ├── s_state == EXT4_VALID_FS
   │   │   └── 干净挂载，无需恢复
   │   │
   │   └── s_state != EXT4_VALID_FS
   │       └── 需要恢复
   │
   ├── 2. 读取日志
   │       │
   │       ├── 扫描日志记录
   │       └── 识别未完成的事务
   │
   ├── 3. 重放日志
   │       │
   │       ├── 重放已完成的事务
   │       └── 回滚未完成的事务
   │
   └── 4. 处理孤儿 inode
           │
           └── 释放孤儿 inode 占用的块
```

### 7.2 恢复函数

```c
int ext4_load_journal(struct super_block *sb,
                      struct ext4_super_block *es,
                      int read_only)
{
    /* 1. 检查日志状态 */
    if (sbi->s_journal)
        return 0;

    /* 2. 初始化日志 */
    sbi->s_journal = jbd2_journal_init_dev(bdev, start, len, blocksize);

    /* 3. 恢复日志 */
    err = jbd2_journal_load(sbi->s_journal);

    /* 4. 处理孤儿 inode */
    if (!read_only)
        ext4_orphan_cleanup(sb, es);

    return 0;
}
```

---

## 八、检查点机制

### 8.1 概述

检查点将日志中的事务刷新到最终位置，释放日志空间。

### 8.2 检查点流程

```c
int jbd2_journal_checkpoint(struct journal_s *journal)
{
    transaction_t *transaction;

    /* 1. 获取已完成的事务 */
    transaction = journal->j_checkpoint_transactions;

    /* 2. 刷新缓冲区 */
    while (transaction) {
        jbd2_log_do_checkpoint(journal, transaction);
        transaction = transaction->t_cpnext;
    }

    return 0;
}
```

---

## 九、调试与监控

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

### 9.2 查看日志统计

```bash
cat /proc/fs/ext4/sda1/mb_groups
```

---

## 十、代码位置

| 功能 | 文件路径 |
|------|----------|
| JBD2 核心 | `fs/jbd2/journal.c` |
| JBD2 事务 | `fs/jbd2/transaction.c` |
| JBD2 恢复 | `fs/jbd2/recovery.c` |
| JBD2 检查点 | `fs/jbd2/checkpoint.c` |
| ext4 日志接口 | `fs/ext4/ext4_jbd2.c` |

---

## 十一、参考资源

- ext4 官方文档: `Documentation/filesystems/ext4/`
- JBD2 内核代码: `fs/jbd2/`
- ext4 内核代码: `fs/ext4/ext4_jbd2.c`
