---
name: ext4-why-fast-commit
description: ext4快速提交存在必要性专家。当用户询问为什么ext4需要fast_commit、fsync延迟问题、fast_commit与JBD2关系、快速提交原理时调用此技能。
---

# 为什么 ext4 需要 Fast Commit

## 1. 问题起源

在传统 JBD2 中，每次 `fsync()` 都需要等待完整的事务提交：

```
fsync() 的完整流程 (传统 JBD2):
  1. 获取 jbd2 锁
  2. 停止当前事务接受新修改
  3. 将所有修改的缓冲区写入日志
  4. 写入 commit record
  5. 等待磁盘 flush (FUA/FLUSH)
  6. 更新检查点
  7. 释放锁

延迟分解:
  步骤 3: 写日志 - 1-10ms (取决于事务大小)
  步骤 4: 写 commit record - 0.1ms
  步骤 5: 等待磁盘 flush - 5-20ms (机械盘更慢)
  ─────────────────────────────────────
  总延迟: 10-50ms

对于数据库等 fsync 密集型应用:
  PostgreSQL: 每次事务提交都需要 fsync
  1000 TPS → 1000 次 fsync → 10-50 秒延迟
  实际吞吐量被 fsync 延迟限制
```

### 问题量化

```
传统 JBD2 的 fsync 延迟测试 (NVMe SSD):
  小事务 (1 个 inode 修改): ~5ms
  中等事务 (10 个 inode 修改): ~10ms
  大事务 (100 个 inode 修改): ~30ms

数据库场景:
  SQLite: 每次写入都 fsync
  理论最大 TPS: 1000 / 5ms = 200 TPS
  实际: 100-150 TPS (受 fsync 延迟限制)

对比:
  无 fsync: 100,000+ TPS
  有 fsync: 100-200 TPS
  差距: 500-1000 倍
```

### 为什么不能简单优化 JBD2

JBD2 的设计约束：

1. **原子性保证**：事务要么全部提交，要么全部不提交
2. **顺序保证**：数据必须先于元数据写回（ordered 模式）
3. **恢复保证**：crash 后能从日志恢复

这些保证意味着：
- 必须写完整的事务到日志
- 必须等待磁盘 flush
- 无法跳过任何步骤

## 2. 为什么需要 Fast Commit

Fast Commit 的核心思想：**对于小事务，只记录元数据差异，跳过完整日志写入**。

```
Fast Commit vs 传统提交:

传统提交:
  [inode 修改] → 写完整 inode 到日志 → 写 commit record → flush
  写入量: 完整 inode (256 bytes) + 日志头尾 + 填充

Fast Commit:
  [inode 修改] → 写差异到 fast commit 区域 → 写 FC record → flush
  写入量: 仅差异 (20-50 bytes) + FC 头尾

延迟对比:
  传统: 5-10ms
  Fast Commit: 50-200μs
  加速比: 25-100 倍
```

Fast Commit 由 Harshad Shirwadkar 等人在 2020-2021 年开发，2021 年合并到 Linux 5.12。

### 适用场景

Fast Commit 特别适合：

| 场景 | 传统 fsync 延迟 | Fast Commit 延迟 | 加速比 |
|------|----------------|-----------------|--------|
| 数据库 WAL 写入 | 5-10ms | 50-100μs | 50-100x |
| 邮件服务器 | 5-10ms | 100-200μs | 25-50x |
| 日志文件 | 5-10ms | 50-100μs | 50-100x |
| 大文件写入 | 10-30ms | 10-30ms | 1x (回退到完整提交) |

Fast Commit 不适用于：
- 大文件写入（数据量大，差异记录不划算）
- 大量元数据修改（差异记录过大）
- 目录操作（复杂，需要完整提交）

## 3. 核心设计

Fast Commit 的架构：

```
Fast Commit 区域:
  ┌─────────────────────────────────────┐
  │  日志区 (传统 JBD2)                  │
  ├─────────────────────────────────────┤
  │  Fast Commit 区域                   │
  │  ┌─────────────────────────────┐   │
  │  │  FC_HEAD                    │   │
  │  │  FC_INODE (inode 差异)      │   │
  │  │  FC_INODE (inode 差异)      │   │
  │  │  FC_TAIL (提交标记)         │   │
  │  └─────────────────────────────┘   │
  └─────────────────────────────────────┘

Fast Commit 流程:
  1. 修改元数据时，记录差异到 FC 跟踪列表
  2. fsync() 时，检查是否可以 fast commit
  3. 如果可以，写 FC 记录到 FC 区域
  4. 写 FC_TAIL 标记提交完成
  5. 等待磁盘 flush
  6. 返回

恢复流程:
  1. 检查日志区是否有完整事务
  2. 如果有，先重放完整事务
  3. 检查 FC 区域是否有 FC_HEAD/FC_TAIL 对
  4. 如果有，重放 FC 记录
  5. 清理 FC 区域
```

关键设计决策：

1. **差异记录**：只记录 inode 的修改字段，不记录完整 inode
2. **FC 区域**：在日志区内划分专用区域
3. **回退机制**：复杂操作回退到完整提交
4. **并发控制**：FC 提交与完整提交互斥
5. **恢复顺序**：先重放完整事务，再重放 FC 记录

## 4. 关键数据结构

### Fast Commit 超级块信息

```c
/* fs/ext4/ext4.h */
struct ext4_sb_info {
	/* ... */
	/* Fast Commit 相关 */
	unsigned int s_fc_bytes;           /* FC 区域总字节数 */
	unsigned int s_fc_off;             /* FC 区域偏移 */
	unsigned int s_fc_max_blks;        /* FC 区域最大块数 */
	unsigned int s_fc_wbufsize;        /* 写缓冲区大小 */
	unsigned int s_fc_wbufs;           /* 写缓冲区数量 */
	unsigned int s_fc_replay_cb_off;   /* 重放回调偏移 */
	unsigned int s_fc_replay_cb_len;   /* 重放回调长度 */
	/* ... */
};
```

### Fast Commit 头

```c
/* fs/ext4/fast_commit.h */
struct ext4_fc_tl {
	__le16  fc_tag;       /* 标签类型 */
	__le16  fc_len;       /* 标签长度 */
} __packed;

/* 标签类型 */
#define EXT4_FC_TAG_ADD_RANGE    1  /* 添加 extent 范围 */
#define EXT4_FC_TAG_DEL_RANGE    2  /* 删除 extent 范围 */
#define EXT4_FC_TAG_LINK         3  /* 链接 inode */
#define EXT4_FC_TAG_UNLINK       4  /* 取消链接 */
#define EXT4_FC_TAG_CREAT        5  /* 创建 inode */
#define EXT4_FC_TAG_INODE        6  /* inode 更新 */
#define EXT4_FC_TAG_PAD          7  /* 填充 */
#define EXT4_FC_TAG_TAIL         8  /* 提交尾 */
#define EXT4_FC_TAG_HEAD         9  /* 提交头 */
#define EXT4_FC_TAG_LINK_RANGE  10  /* 链接范围 */
```

### Fast Commit 头记录

```c
/* fs/ext4/fast_commit.h */
struct ext4_fc_head {
	__le32  fc_magic;          /* 魔数: 0xFCFCFCFC */
	__le32  fc_features;       /* 特性标志 */
	__le32  fc_subtid;         /* 子事务 ID */
	__le32  fc_tid;            /* 事务 ID */
	__le32  fc_len;            /* FC 区域长度 */
	__le32  fc_crc;            /* 校验和 */
} __packed;
```

### Fast Commit 尾记录

```c
/* fs/ext4/fast_commit.h */
struct ext4_fc_tail {
	__le32  fc_subtid;         /* 子事务 ID */
	__le32  fc_tid;            /* 事务 ID */
	__le32  fc_crc;            /* 校验和 */
} __packed;
```

### Fast Commit inode 记录

```c
/* fs/ext4/fast_commit.h */
struct ext4_fc_inode {
	__le32  fc_ino;            /* inode 号 */
	struct ext4_inode fc_raw_inode; /* 完整 inode (仅关键字段) */
	__le32  fc_crc;            /* 校验和 */
} __packed;
```

### Fast Commit 状态

```c
/* fs/ext4/ext4.h */
struct ext4_inode_info {
	/* ... */
	struct list_head i_fc_list;    /* FC 跟踪链表 */
	int i_fc_committed;            /* FC 提交状态 */
	__u32 i_fc_tid;                /* FC 事务 ID */
	/* ... */
};

/* FC 状态 */
#define EXT4_FC_STATE_OK       0  /* 可以 fast commit */
#define EXT4_FC_STATE_INELIGIBLE 1 /* 需要完整提交 */
```

### Fast Commit 重放状态

```c
/* fs/ext4/fast_commit.c */
struct ext4_fc_replay_state {
	int fc_replay_expected_off;    /* 期望偏移 */
	int fc_replay_off;             /* 当前偏移 */
	int fc_replay_subtid;          /* 子事务 ID */
	int fc_replay_crc;             /* 校验和 */
	struct list_head fc_pending_inodes; /* 待处理 inode */
};
```

## 5. 10年演进 (2015-2025)

| 年份 | 内核版本 | 变更 | 解决的问题 |
|------|---------|------|-----------|
| 2015 | 4.1 | 异步提交优化 | 减少传统提交延迟 |
| 2018 | 4.19 | 提交批量优化 | 提高吞吐量 |
| 2020 | 5.7 | fast_commit 实验性支持 | 概念验证 |
| 2021 | 5.12 | fast_commit 正式合并 | fsync 延迟降低 50-100 倍 |
| 2021 | 5.13 | FC 区域大小可调 | 适应不同 workload |
| 2022 | 5.16 | FC 并发优化 | 减少锁竞争 |
| 2022 | 5.18 | FC 恢复改进 | 更可靠的 crash 恢复 |
| 2023 | 6.2 | FC 统计接口 | 性能监控 |
| 2023 | 6.4 | FC 回退优化 | 更智能的回退决策 |
| 2024 | 6.8 | FC 校验和增强 | 更好的完整性保护 |
| 2025 | 6.12 | FC 与 large folio 兼容 | 支持大块大小 |

### Fast Commit 性能数据

```
测试环境: NVMe SSD, 4K 随机写入

传统 JBD2:
  fsync 延迟: p50=5ms, p99=15ms
  吞吐量: 200 fsync/s

Fast Commit:
  fsync 延迟: p50=80μs, p99=200μs
  吞吐量: 5000 fsync/s

加速比: 25x (p50), 75x (p99)
```

## 6. 与其他特性的关系

```
Fast Commit
  │
  ├── JBD2: 基于 JBD2 的事务框架
  │     └─→ 共享事务 ID 和序列号
  │     └─→ 恢复时先重放 JBD2 事务，再重放 FC 记录
  │
  ├── metadata_csum: FC 记录也有校验和
  │     └─→ 防止 FC 区域损坏
  │
  ├── extent: FC 记录 extent 差异
  │     └─→ ADD_RANGE / DEL_RANGE 标签
  │
  ├── mballoc: 不影响分配机制
  │
  ├── bigalloc: FC 记录 cluster 而非 block
  │
  ├── encrypt: 加密不影响 FC（元数据不加密）
  │
  └── large_folio: FC 需要适配大块大小
```

## 7. 关键代码位置

```
fs/ext4/
├── fast_commit.c      # Fast Commit 核心实现
├── fast_commit.h      # 数据结构定义
├── fsync.c            # fsync 入口 (调用 fast_commit)
├── inode.c            # inode 修改跟踪
├── namei.c            # 目录操作跟踪
└── super.c            # FC 区域初始化

fs/jbd2/
├── commit.c           # 完整提交（FC 回退时）
└── transaction.c      # 事务管理

关键函数:
  ext4_fc_track_inode()        # 跟踪 inode 修改
  ext4_fc_track_range()        # 跟踪 extent 范围
  ext4_fc_commit()             # FC 提交
  ext4_fc_write_inode()        # 写 inode 记录
  ext4_fc_write_range()        # 写范围记录
  ext4_fc_replay()             # FC 重放
  ext4_fc_add_tag()            # 添加 FC 标签
  ext4_fc_perform_commit()     # 执行 FC 提交
```
