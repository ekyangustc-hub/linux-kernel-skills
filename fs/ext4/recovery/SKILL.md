---
name: ext4-recovery
description: ext4文件系统恢复机制专家。当用户询问ext4断电恢复、JBD2日志重放、Fast Commit恢复、孤儿文件清理、eshutdown、在线修复、e2fsck离线修复时调用此技能。
---

# ext4 恢复机制

## 1. 为什么需要恢复机制？

文件系统崩溃的主要原因：

- **断电 (Power Loss)**: 最危险的情况，内存中的脏数据来不及写入磁盘
- **内核崩溃 (Kernel Panic)**: 文件系统状态不一致
- **硬件故障**: 存储设备突然断开或损坏
- **强制重启**: 未正常卸载文件系统

ext4 面临的挑战：
- 元数据操作通常是多步骤的（分配 inode → 分配数据块 → 更新目录项）
- 任何一步中断都会导致不一致
- 现代 SSD 的写放大和 FTL 层增加了不确定性

**恢复目标**:
1. 保证元数据一致性（不能出现循环目录、悬空指针等）
2. 最小化恢复时间（Fast Commit 的核心价值）
3. 尽可能保留用户数据

## 2. JBD2 日志恢复

### 2.1 触发条件

挂载时检测到 `EXT4_FEATURE_INCOMPAT_RECOVER` 标志：

```c
// fs/ext4/super.c
if (es->s_state & cpu_to_le16(EXT4_FEATURE_INCOMPAT_RECOVER)) {
    need_recovery = true;
    ext4_msg(sb, KERN_INFO, "recovery required");
}
```

### 2.2 恢复流程

```
mount
  └─ ext4_fill_fs()
       └─ ext4_load_journal()
            └─ jbd2_journal_load()
                 ├─ jbd2_journal_recover()  ← 核心恢复函数
                 │    ├─ jbd2_journal_revoke()  ← 阶段1: REVOKE
                 │    ├─ jbd2_scan_journal()    ← 阶段2: SCAN
                 │    └─ jbd2_journal_replay()  ← 阶段3: REPLAY
                 └─ jbd2_journal_clear_recover_flag()
```

### 2.3 恢复三阶段

#### 阶段 1: REVOKE（撤销阶段）

```c
// fs/jbd2/recovery.c
static int do_one_pass(journal_t *journal, struct recovery_info *info,
                       enum passtype pass)
{
    // pass == PASS_REVOKE
    // 读取 revoke table，记录哪些块应该被忽略
}
```

**作用**: 处理已提交但后来被撤销的操作。例如：
- 块被分配后又释放
- 目录项创建后被删除

revoke table 存储在日志中，格式为 `journal_revoke_header_t` + 一系列块号。

#### 阶段 2: SCAN（扫描阶段）

```c
// pass == PASS_SCAN
// 扫描整个日志，找到最新的完整事务
// 确定恢复的起点和终点
```

关键操作：
- 读取每个事务的 `journal_header_t`
- 验证事务序列号 (`h_sequence`)
- 找到最后一个完整提交的事务 (`JBD2_COMMIT_BLOCK`)
- 记录需要恢复的事务范围

#### 阶段 3: REPLAY（重放阶段）

```c
// pass == PASS_REPLAY
// 将日志中的元数据块写回原始位置
```

重放过程：
1. 读取日志中的 `JBD2_DESCRIPTOR_BLOCK`
2. 对于每个被记录的块：
   - 检查是否在 revoke table 中（如果是则跳过）
   - 将数据写回原始磁盘位置
3. 处理 `JBD2_COMMIT_BLOCK` 确认事务完整性
4. 更新超级块状态

### 2.4 日志事务结构

```
+------------------+
| Journal Header   |  (s_start, s_sequence)
+------------------+
| Descriptor Block |  (描述后续数据块)
+------------------+
| Data Block 1     |  (元数据副本)
| Data Block 2     |
| ...              |
+------------------+
| Commit Block     |  (事务提交标记)
+------------------+
| Revoke Block     |  (可选，撤销表)
+------------------+
```

### 2.5 关键数据结构

```c
// include/linux/jbd2.h
struct journal_s {
    unsigned long       j_flags;        // JBD2_ABORT, JBD2_RECOVERED 等
    tid_t               j_tail_sequence; // 日志中最早的事务序列号
    tid_t               j_transaction_sequence; // 下一个事务序列号
    __u32               j_head;         // 日志头块
    __u32               j_tail;         // 日志尾块
    // ...
};

struct recovery_info {
    tid_t       start_transaction;
    tid_t       end_transaction;
    int         nr_replays;     // 重放的块数
    int         nr_revokes;     // 撤销的块数
    int         nr_revoke_hits; // revoke 命中数
};
```

### 2.6 恢复后的清理

```c
// 清除恢复标志
es->s_state &= ~cpu_to_le16(EXT4_FEATURE_INCOMPAT_RECOVER);
ext4_commit_super(sb);

// 清空日志
jbd2_journal_flush(journal);
```

## 3. Fast Commit 恢复

### 3.1 为什么需要 Fast Commit？

传统 JBD2 日志重放的瓶颈：
- 必须扫描整个日志区域
- 即使只有一个文件被修改，也要检查所有事务
- 大型文件系统（TB 级）恢复时间可达数分钟

Fast Commit 的设计目标：
- 只记录实际修改的元数据
- 恢复时只扫描 FC 区域（通常很小）
- 恢复时间从分钟级降到秒级

### 3.2 Fast Commit 区域结构

```
+------------------+
| Full Commit Area |  (传统 JBD2 日志)
|                  |
+------------------+
| Fast Commit Area |  (FC 区域)
| +--------------+ |
| | FC Header    | |
| | FC TLV Tags  | |  ← 每个 tag 描述一个操作
| | FC Tail      | |
| +--------------+ |
+------------------+
```

FC TLV (Type-Length-Value) 标签类型：

```c
// fs/ext4/fast_commit.c
#define EXT4_FC_TAG_ADD_RANGE    1  // 添加文件数据范围
#define EXT4_FC_TAG_DEL_RANGE    2  // 删除文件数据范围
#define EXT4_FC_TAG_LINK         3  // 硬链接
#define EXT4_FC_TAG_UNLINK       4  // 删除链接
#define EXT4_FC_TAG_CREAT        5  // 创建文件
#define EXT4_FC_TAG_INODE        6  // inode 更新
#define EXT4_FC_TAG_PAD          7  // 填充
#define EXT4_FC_TAG_TAIL         8  // FC 尾部标记
```

### 3.3 Fast Commit 恢复流程

```
mount
  └─ ext4_fc_replay_check()
       └─ ext4_fc_replay()
            ├─ ext4_fc_replay_scan()     ← 扫描 FC 区域
            ├─ ext4_fc_replay_cleanup()  ← 清理状态
            └─ 逐个处理 FC TLV tags
```

```c
// fs/ext4/fast_commit.c
int ext4_fc_replay(struct super_block *sb, struct buffer_head *bh,
                   enum ext4_fc_replay_state state, int off)
{
    struct ext4_fc_tl *tl;
    u8 *val;
    
    // 解析 TLV 标签
    tl = (struct ext4_fc_tl *)(bh->b_data + off);
    val = (u8 *)tl + sizeof(*tl);
    
    switch (le16_to_cpu(tl->fc_tag)) {
    case EXT4_FC_TAG_ADD_RANGE:
        ext4_fc_replay_add_range(sb, val, len);
        break;
    case EXT4_FC_TAG_DEL_RANGE:
        ext4_fc_replay_del_range(sb, val, len);
        break;
    case EXT4_FC_TAG_CREAT:
        ext4_fc_replay_create(sb, val, len);
        break;
    // ... 处理其他标签
    }
}
```

### 3.4 Fast Commit vs 完整日志重放

| 特性 | 完整 JBD2 重放 | Fast Commit 重放 |
|------|---------------|-----------------|
| 扫描范围 | 整个日志区域 | 仅 FC 区域 |
| 恢复时间 | 分钟级（大文件系统） | 秒级 |
| 记录内容 | 所有元数据块副本 | 仅变更描述 |
| 适用场景 | 所有元数据操作 | 文件创建/删除/修改 |
| 回退机制 | 无 | FC 失败时回退到完整提交 |

### 3.5 Fast Commit 状态机

```
IDLE → RUNNING → COMMITTING → COMMITTED
  ↑                                        |
  └────────────────────────────────────────┘
```

- **IDLE**: 没有活跃的 FC 事务
- **RUNNING**: 正在收集 FC 变更
- **COMMITTING**: 正在写入 FC 区域
- **COMMITTED**: FC 事务已提交，等待完整提交

## 4. 孤儿文件恢复

### 4.1 什么是孤儿文件？

孤儿文件 (Orphan File/Inode) 是指：
- inode 引用计数 > 0（有进程打开）
- 但目录项已被删除（nlink == 0）

场景示例：
```c
fd = open("file.txt", O_RDWR);
unlink("file.txt");  // 目录项删除，但文件仍存在
// 此时 file.txt 成为孤儿文件
// 如果此时断电，需要恢复机制清理
```

### 4.2 孤儿文件的两种实现

#### 传统方式：s_last_orphan 链表

```c
// 超级块中的孤儿链表头
__le32  s_last_orphan;  // 最后一个孤儿 inode 的编号

// 孤儿 inode 通过 i_dtime 字段链接
struct ext4_inode {
    __le32  i_dtime;  // 删除时间，也用作孤儿链表指针
    // ...
};
```

链表结构：
```
s_last_orphan → inode_A → inode_B → inode_C → 0
```

#### 新方式：Orphan File（EXT4_FEATURE_INCOMPAT_ORPHAN_PRESENT）

```c
// 使用专用 inode 存储孤儿列表
// 优点：
// 1. 支持更多孤儿文件（不受块大小限制）
// 2. 更可靠的恢复
// 3. 更好的并发处理
```

### 4.3 孤儿文件清理流程

```c
// fs/ext4/super.c
static void ext4_orphan_cleanup(struct super_block *sb,
                                struct ext4_super_block *es)
{
    unsigned int s_flags = sb->s_flags;
    __le32 s_last_orphan = es->s_last_orphan;
    struct inode *inode;
    
    if (!ext4_feature_set_has_orphan_file(sb, false) &&
        es->s_last_orphan == 0)
        return;  // 没有孤儿文件需要清理
    
    // 设置 SB_RDONLY 防止新操作
    sb->s_flags |= SB_RDONLY;
    
    // 遍历孤儿链表
    while (es->s_last_orphan) {
        inode = ext4_orphan_get(sb, le32_to_cpu(es->s_last_orphan));
        if (IS_ERR(inode))
            break;
            
        // 截断文件并释放 inode
        ext4_free_inode_handle_error(sb, inode);
        iput(inode);
        
        ext4_msg(sb, KERN_INFO, 
                 "%s: orphan file cleanup (%lu inodes)",
                 __func__, inodes_freed);
    }
    
    sb->s_flags = s_flags;
}
```

### 4.4 孤儿文件处理的关键点

1. **挂载时清理**: 在 `ext4_fill_fs()` 中调用
2. **只读模式**: 清理期间文件系统设为只读
3. **截断优先**: 先释放数据块，再释放 inode
4. **错误处理**: 清理失败时标记文件系统错误

## 5. 错误处理与文件系统关闭

### 5.1 错误级别

ext4 定义了三种错误处理级别：

```c
// fs/ext4/super.c
#define EXT4_ERRORS_CONTINUE    1  // 继续运行，记录错误
#define EXT4_ERRORS_RO          2  // 切换为只读
#define EXT4_ERRORS_PANIC       3  // 内核恐慌
#define EXT4_ERRORS_DEFAULT     4  // 使用编译时默认值
```

挂载选项：
```
errors=continue   # 继续运行
errors=remount-ro # 重新挂载为只读（默认）
errors=panic      # 触发 panic
```

### 5.2 错误处理函数

#### ext4_error() - 记录错误

```c
void __ext4_error(struct super_block *sb, const char *function,
                  unsigned int line, ext4_fsblk_t block,
                  const char *fmt, ...)
{
    struct ext4_sb_info *sbi = EXT4_SB(sb);
    
    // 记录错误信息
    ext4_msg(sb, KERN_CRIT, "error: %s:%d: ", function, line);
    
    // 更新超级块错误计数
    es->s_error_count++;
    
    // 根据配置处理
    switch (sbi->s_mount_opt & EXT4_MOUNT_ERROR_MASK) {
    case EXT4_ERRORS_CONTINUE:
        break;
    case EXT4_ERRORS_RO:
        ext4_msg(sb, KERN_CRIT, "Remounting filesystem read-only");
        sb->s_flags |= SB_RDONLY;
        break;
    case EXT4_ERRORS_PANIC:
        panic("ext4: panic from mount point %s", sb->s_id);
    }
}
```

#### ext4_abort() - 中止文件系统

```c
void __ext4_abort(struct super_block *sb, const char *function,
                  unsigned int line, const char *fmt, ...)
{
    // 设置 ESHUTDOWN 错误
    EXT4_SB(sb)->s_mount_state |= EXT4_ERROR_FS;
    
    // 切换为只读
    sb->s_flags |= SB_RDONLY;
    
    // 记录错误
    ext4_msg(sb, KERN_CRIT, "abort from %s:%d", function, line);
}
```

#### ext4_handle_error() - 事务句柄错误

```c
// 当 journalled 操作失败时调用
// 会触发 journal abort
void ext4_handle_error(struct super_block *sb, bool force_ro,
                       int err, ext4_fsblk_t block)
{
    journal_t *journal = EXT4_SB(sb)->s_journal;
    
    if (journal)
        jbd2_journal_abort(journal, err);
        
    __ext4_error(sb, __func__, __LINE__, block,
                 "Journal has aborted");
}
```

### 5.3 ESHUTDOWN 错误码

```c
// include/linux/ext4_fs.h
#define EXT4_ERR_ESHUTDOWN    117  /* Filesystem has been shutdown */
```

ESHUTDOWN 的含义：
- 文件系统检测到严重错误
- 所有新操作应立即失败
- 返回 `-EIO` 或 `-ESHUTDOWN` 给用户空间

触发 ESHUTDOWN 的场景：
1. Journal abort
2. 元数据校验和失败
3. 检测到文件系统损坏
4. 硬件 I/O 错误

```c
// 检查文件系统是否已关闭
static inline bool ext4_forced_shutdown(struct ext4_sb_info *sbi)
{
    return test_bit(EXT4_FLAGS_SHUTDOWN, &sbi->s_ext4_flags);
}

// 在操作开始前检查
if (ext4_forced_shutdown(sbi))
    return -EIO;
```

### 5.4 文件系统关闭流程

```
错误检测
  └─ ext4_error/abort
       ├─ 设置 EXT4_ERROR_FS 标志
       ├─ 设置 EXT4_FLAGS_SHUTDOWN
       ├─ sb->s_flags |= SB_RDONLY
       ├─ jbd2_journal_abort()
       └─ 唤醒等待的进程
```

## 6. 在线修复

### 6.1 在线调整大小 (EXT4_IOC_RESIZE_FS)

```c
// fs/ext4/ioctl.c
static long ext4_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    case EXT4_IOC_RESIZE_FS: {
        ext4_fsblk_t n_blocks_count;
        
        if (copy_from_user(&n_blocks_count, arg, sizeof(n_blocks_count)))
            return -EFAULT;
            
        return ext4_resize_fs(sb, n_blocks_count);
    }
}
```

`ext4_resize_fs()` 流程：
1. 验证新大小
2. 更新块组描述符
3. 更新超级块
4. 更新 inode 表
5. 在线更新（不需要卸载）

### 6.2 交换 Bootloader (EXT4_IOC_SWAP_BOOT)

```c
case EXT4_IOC_SWAP_BOOT: {
    // 交换 boot loader inode 和数据
    // 用于更新 boot 分区内容
    // 需要 CAP_SYS_ADMIN 权限
}
```

### 6.3 在线碎片整理

```c
// 使用 EXT4_IOC_MOVE_EXT 进行在线碎片整理
// e4defrag 工具使用此 ioctl
```

### 6.4 在线坏块标记

```c
// EXT4_IOC_SETFLAGS 可以设置 IMMUTABLE 等标志
// 配合 e2fsck 可以标记坏块
```

### 6.5 限制

在线修复的限制：
- 不能修复严重的元数据损坏
- 不能修复循环目录
- 不能修复 inode 表损坏
- 严重损坏仍需 e2fsck

## 7. e2fsck 离线修复

### 7.1 e2fsck 概述

e2fsck 是 ext2/3/4 的离线文件系统检查工具：

```bash
# 基本用法
e2fsck /dev/sdXN

# 自动修复
e2fsck -y /dev/sdXN

# 只读检查
e2fsck -n /dev/sdXN

# 强制检查（即使标记为 clean）
e2fsck -f /dev/sdXN

# 显示进度
e2fsck -C 0 /dev/sdXN
```

### 7.2 e2fsck 的五个阶段

#### 阶段 1: 检查 Inodes、Blocks、和 Sizes

```
Pass 1: Checking inodes, blocks, and sizes
```

检查内容：
- inode 类型和模式有效性
- 块范围有效性（extent 树检查）
- 文件大小与块数一致性
- inode 引用计数
- 坏块检测

关键数据结构：
```c
// e2fsck 内部使用的 inode 扫描
ext2fs_scan_inode_table(fs, &scan);
while (!ext2fs_scan_inode_next(scan, &ino, &inode)) {
    check_inode(fs, ino, &inode);
}
```

#### 阶段 2: 检查目录结构

```
Pass 2: Checking directory structure
```

检查内容：
- 目录项格式有效性
- `.` 和 `..` 的正确性
- 目录项不重叠
- 目录项名称有效性
- 目录大小与内容一致性

#### 阶段 3: 检查目录连通性

```
Pass 3: Checking directory connectivity
```

检查内容：
- 所有目录都能从根目录到达
- 修复断开的目录（连接到 lost+found）
- 检查 `..` 指向正确的父目录

#### 阶段 4: 检查引用计数

```
Pass 4: Checking reference counts
```

检查内容：
- inode 链接计数 (nlink) 是否正确
- 块引用计数
- 修复不匹配的引用计数

#### 阶段 5: 检查组摘要信息

```
Pass 5: Checking group summary information
```

检查内容：
- 块组描述符的块计数
- inode 计数
- 空闲块位图
- 空闲 inode 位图
- 块组校验和

### 7.3 e2fsck 修复策略

```c
// e2fsck/unix.c
// 修复决策逻辑
if (problem & PR_FATAL) {
    // 严重错误，必须修复
    fix_problem(ctx, problem, pctx);
} else if (ctx->options & E2F_OPT_YES) {
    // 自动修复模式
    fix_problem(ctx, problem, pctx);
} else if (ctx->options & E2F_OPT_NO) {
    // 只读模式，不修复
    return 0;
} else {
    // 交互模式，询问用户
    if (ask(ctx, question))
        fix_problem(ctx, problem, pctx);
}
```

### 7.4 e2fsck 与 JBD2 的关系

```
e2fsck 启动
  └─ 检查是否需要日志重放
       ├─ 是: 先调用 libext2fs 重放日志
       └─ 否: 直接进行五阶段检查
```

```c
// e2fsck 中的日志重放
if (ext2fs_has_feature_journal_needs_recovery(fs->super)) {
    retval = ext2fs_journal_load(fs);
    if (retval) {
        // 日志损坏，尝试恢复
        retval = e2fsck_journal_load(ctx);
    }
    retval = e2fsck_journal_recover(ctx);
}
```

## 8. 10年演进 (2015-2025)

### 8.1 恢复机制的演进时间线

| 年份 | 内核版本 | 特性 | 影响 |
|------|---------|------|------|
| 2015 | 4.x | fscrypt 集成 | 加密文件的恢复支持 |
| 2016 | 4.8 | metadata_csum 完善 | 恢复时校验元数据完整性 |
| 2017 | 4.12 | fsverity 支持 | 文件完整性验证 |
| 2018 | 4.16 | 改进的错误处理 | 更细粒度的错误报告 |
| 2019 | 5.2 | 改进的孤儿处理 | 更好的并发恢复 |
| 2020 | 5.8 | 挂载 API 转换 | 更清晰的挂载选项处理 |
| 2021 | 5.12 | **Fast Commit** | 恢复时间从分钟降到秒 |
| 2022 | 5.17 | Fast Commit 优化 | 更稳定的 FC 实现 |
| 2023 | 6.x | 改进的 ESHUTDOWN | 更优雅的错误处理 |
| 2024 | 6.8 | ES tree seq counter | 恢复时更好的并发控制 |
| 2025 | 6.12 | Orphan file 改进 | 更可靠的孤儿清理 |
| 2025 | 6.14 | Large block size | 大块大小的恢复支持 |

### 8.2 Fast Commit 的引入

Fast Commit 是 10 年来最重要的恢复机制改进：

**问题**: 传统日志重放在大文件系统上太慢
**方案**: 只记录实际变更，不记录完整块副本
**结果**: 恢复时间从 O(日志大小) 降到 O(变更大小)

```
传统恢复: mount → scan 1GB journal → replay → done (60s+)
Fast Commit: mount → scan 1MB FC area → replay → done (<1s)
```

### 8.3 错误处理的演进

```
2015: 简单的 errors=continue/remount-ro/panic
      ↓
2018: 更细粒度的错误分类
      ↓
2021: ESHUTDOWN 机制完善
      ↓
2024: 更好的错误传播和恢复
```

## 9. 关键代码位置

### 9.1 JBD2 恢复

| 文件 | 函数 | 作用 |
|------|------|------|
| `fs/jbd2/recovery.c` | `jbd2_journal_recover()` | 恢复入口 |
| `fs/jbd2/recovery.c` | `do_one_pass()` | 三阶段扫描 |
| `fs/jbd2/recovery.c` | `scan_revoke_records()` | 处理 revoke 记录 |
| `fs/jbd2/recovery.c` | `do_read_revoke_rec()` | 读取 revoke 表 |
| `fs/jbd2/journal.c` | `jbd2_journal_load()` | 加载日志 |
| `fs/jbd2/commit.c` | `journal_commit_transaction()` | 事务提交 |

### 9.2 Fast Commit 恢复

| 文件 | 函数 | 作用 |
|------|------|------|
| `fs/ext4/fast_commit.c` | `ext4_fc_replay()` | FC 恢复入口 |
| `fs/ext4/fast_commit.c` | `ext4_fc_replay_scan()` | 扫描 FC 区域 |
| `fs/ext4/fast_commit.c` | `ext4_fc_replay_add_range()` | 重放添加范围 |
| `fs/ext4/fast_commit.c` | `ext4_fc_replay_del_range()` | 重放删除范围 |
| `fs/ext4/fast_commit.c` | `ext4_fc_replay_create()` | 重放创建操作 |
| `fs/ext4/fast_commit.c` | `ext4_fc_replay_cleanup()` | 清理 FC 状态 |
| `fs/ext4/fast_commit.c` | `ext4_fc_track_range()` | 跟踪范围变更 |
| `fs/ext4/fast_commit.c` | `ext4_fc_track_inode()` | 跟踪 inode 变更 |

### 9.3 孤儿文件处理

| 文件 | 函数 | 作用 |
|------|------|------|
| `fs/ext4/super.c` | `ext4_orphan_cleanup()` | 孤儿清理入口 |
| `fs/ext4/super.c` | `ext4_orphan_add()` | 添加孤儿 inode |
| `fs/ext4/super.c` | `ext4_orphan_del()` | 删除孤儿 inode |
| `fs/ext4/super.c` | `ext4_orphan_get()` | 获取孤儿 inode |
| `fs/ext4/namei.c` | `ext4_unlink()` | 触发孤儿添加 |
| `fs/ext4/orphan.c` | `ext4_orphan_file_init()` | 初始化孤儿文件 |
| `fs/ext4/orphan.c` | `ext4_orphan_file_cleanup()` | 孤儿文件清理 |

### 9.4 错误处理

| 文件 | 函数 | 作用 |
|------|------|------|
| `fs/ext4/super.c` | `__ext4_error()` | 记录错误 |
| `fs/ext4/super.c` | `__ext4_abort()` | 中止文件系统 |
| `fs/ext4/super.c` | `ext4_handle_error()` | 句柄错误处理 |
| `fs/ext4/super.c` | `ext4_msg()` | 消息输出 |
| `fs/ext4/super.c` | `ext4_update_dynamic_err()` | 更新动态错误信息 |
| `fs/ext4/sysfs.c` | `errors_show/store` | sysfs 错误配置 |

### 9.5 e2fsck (用户空间)

| 文件 (e2fsprogs) | 作用 |
|-----------------|------|
| `e2fsck/pass1.c` | 阶段 1: inode 检查 |
| `e2fsck/pass1b.c` | 阶段 1b: 重复块检查 |
| `e2fsck/pass2.c` | 阶段 2: 目录检查 |
| `e2fsck/pass3.c` | 阶段 3: 连通性检查 |
| `e2fsck/pass4.c` | 阶段 4: 引用计数 |
| `e2fsck/pass5.c` | 阶段 5: 组摘要 |
| `e2fsck/journal.c` | 日志恢复 |
| `e2fsck/unix.c` | 主入口 |

### 9.6 关键数据结构

```c
// 超级块恢复相关字段
struct ext4_super_block {
    __le32  s_inodes_count;
    __le32  s_blocks_count;
    __le16  s_state;           // EXT4_VALID_FS, EXT4_ERROR_FS
    __le16  s_errors;          // 错误处理策略
    __le32  s_lastcheck;       // 上次检查时间
    __le32  s_checkinterval;   // 检查间隔
    __le32  s_last_orphan;     // 孤儿链表头
    __le32  s_flags;           // 动态标志
    __le16  s_errors_count;    // 错误计数
    // ...
};

// 挂载状态标志
#define EXT4_VALID_FS           0x0001
#define EXT4_ERROR_FS           0x0002
#define EXT4_ORPHAN_FS          0x0004
#define EXT4_FC_COMMITTING      0x0008
```
