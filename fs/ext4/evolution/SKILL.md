---
name: ext4-evolution
description: ext4文件系统10年演进专家。当用户询问ext4历史发展、特性演进时间线、内核版本特性、ext4与XFS/Btrfs对比、主要贡献者、未来方向时调用此技能。
---

# ext4 十年演进 (2015-2025)

## 1. ext4 历史概述

### 1.1 从 ext2 到 ext4

```
ext2 (1993)
  └─ 第二代扩展文件系统
  └─ 无日志，断电恢复慢
  └─ Remy Card 设计

ext3 (2001, 2.4.15)
  └─ 添加 JBD 日志层
  └─ 在线转换 ext2↔ext3
  └─ Stephen Tweedie 实现

ext4 (2008, 2.6.28)
  └─ Extent 树替代块映射
  └─ 多块分配器 (mballoc)
  └─ 延迟分配 (delalloc)
  └─ 48位块号 (1EB 支持)
  └─ Theodore Ts'o 主导
```

### 1.2 ext4 的设计哲学

- **向后兼容**: ext4 可以挂载 ext2/ext3 文件系统
- **渐进式改进**: 不追求革命性变化，而是持续优化
- **稳定性优先**: 新特性经过充分测试才合入
- **实用性**: 面向真实工作负载优化

### 1.3 ext4 的核心架构

```
用户空间
  └─ VFS (Virtual File System)
       └─ ext4
            ├─ 文件操作 (file.c, namei.c)
            ├─ 块分配 (mballoc.c)
            ├─ 日志层 (JBD2)
            ├─ 回写 (page-io.c, fsync.c)
            └─ 扩展属性 (xattr.c)
```

## 2. 2015-2017: 稳定性与特性完善

### 2.1 2015: 加密与国际化

#### fscrypt (文件系统级加密)

```c
// fs/ext4/crypto.c (后被 fs/crypto/ 替代)
// Linux 4.1 引入 fscrypt 框架
// ext4 是最早支持的文件系统之一

// 加密模式:
// - 文件名加密: AES-CTS-CBC
// - 内容加密: AES-XTS
// - IV 生成: ESSIV 或 IV_INO_LBLK_64

// 挂载选项:
// - test_dummy_encryption (测试用)
```

**影响**:
- Android 采用 fscrypt 作为默认加密方案
- 支持 per-file 加密密钥
- 与 dm-crypt 互补（fscrypt 加密文件名）

#### Casefold (大小写不敏感)

```c
// fs/ext4/namei.c
// Linux 5.2 正式支持 (开发始于 2015)
// 使用 UTF-8 casefold 算法

// 挂载选项:
// - encoding=utf8
// - encoding_flags=casefold
```

### 2.2 2016: 元数据校验和完善

#### metadata_csum 完善

```c
// 2012 年引入，2016 年完善
// 所有元数据块都有 CRC32C 校验和

struct ext4_inode {
    // ...
    __le32  i_checksum;  // inode 校验和
};

struct ext4_group_desc {
    // ...
    __le16  bg_checksum;  // 块组描述符校验和
};
```

**校验范围**:
- 超级块 (superblock)
- 块组描述符 (group descriptors)
- Inode 表 (inode table)
- 目录块 (directory blocks)
- 扩展属性块 (xattr blocks)
- Extent 树块 (extent tree blocks)

#### 在线碎片整理改进

```bash
# e4defrag 工具改进
# 使用 EXT4_IOC_MOVE_EXT ioctl
# 支持在线移动文件数据块
```

### 2.3 2017: fsverity 支持

#### fsverity (文件完整性验证)

```c
// fs/ext4/verity.c
// Linux 4.12 引入 fsverity 框架
// 基于 Merkle 树的文件完整性验证

// 使用场景:
// - 系统文件完整性
// - 容器镜像验证
// - 安全启动

// 启用方式:
// fsverity enable <file>
```

**Merkle 树结构**:
```
文件数据
  └─ 第 0 层: 数据块哈希
       └─ 第 1 层: 哈希的哈希
            └─ ...
                 └─ 根哈希 (存储在 inode 中)
```

### 2.4 其他重要改进 (2015-2017)

| 年份 | 内核版本 | 特性 | 描述 |
|------|---------|------|------|
| 2015 | 4.2 | inline_data 完善 | 小文件数据存储在 inode 中 |
| 2015 | 4.3 | DAX 支持 | 直接访问持久内存 |
| 2016 | 4.5 | 改进的 fallocate | 更好的空间预分配 |
| 2016 | 4.6 | 目录索引改进 | HTREE 性能优化 |
| 2016 | 4.7 | 改进的 writeback | 更好的回写策略 |
| 2017 | 4.9 | 改进的 mballoc | 块分配器优化 |
| 2017 | 4.11 | 改进的 xattr | 扩展属性性能提升 |
| 2017 | 4.13 | 项目配额 (project quota) | 目录树配额支持 |

## 3. 2018-2020: 性能优化与新特性

### 3.1 2018: 多队列 I/O 优化

#### blk-mq 适配

```c
// ext4 适配多队列块层
// 改进并发 I/O 性能
// 减少锁竞争
```

#### 改进的读取预取

```c
// fs/ext4/readpage.c
// 改进的预读算法
// 基于访问模式的自适应预取
```

### 3.2 2019: 写入路径优化

#### 改进的延迟分配

```c
// fs/ext4/inode.c
// ext4_da_writepages() 改进
// 更好的簇分配策略
// 减少碎片
```

#### 异步 I/O 改进

```c
// fs/ext4/file.c
// ext4_file_write_iter() 改进
// 支持更好的 AIO 并发
```

### 3.3 2020: 挂载 API 转换

#### 新的挂载 API

```c
// fs/ext4/super.c
// 从 mount() 转换到 fs_context
// Linux 5.8 完成转换

// 旧方式:
// mount -t ext4 -o data=ordered /dev/sda1 /mnt

// 新方式 (内部):
// fsconfig(fd, FSCONFIG_SET_STRING, "data", "ordered", 0);
```

**优势**:
- 统一的挂载参数处理
- 更好的错误报告
- 支持命名空间感知
- 二进制安全的参数传递

### 3.4 其他重要改进 (2018-2020)

| 年份 | 内核版本 | 特性 | 描述 |
|------|---------|------|------|
| 2018 | 4.16 | 改进的错误处理 | 更细粒度的错误报告 |
| 2018 | 4.17 | 改进的 fscrypt | 性能优化 |
| 2018 | 4.19 | 改进的 fsverity | 更好的 Merkle 树实现 |
| 2019 | 5.0 | 改进的 extent 树 | extent 分裂优化 |
| 2019 | 5.1 | 改进的配额 | 配额性能提升 |
| 2019 | 5.3 | 改进的 discard | 更好的 TRIM 支持 |
| 2020 | 5.5 | 改进的 fsync | fsync 性能优化 |
| 2020 | 5.7 | 改进的 DAX | 持久内存支持增强 |
| 2020 | 5.9 | 改进的 bigalloc | 大块分配优化 |

## 4. 2021-2023: Fast Commit 与现代 I/O

### 4.1 2021: Fast Commit 革命

#### Fast Commit 的引入

```c
// fs/ext4/fast_commit.c
// Linux 5.12 引入
// Ritesh Harjani 主导开发

// 核心思想:
// - 只记录元数据变更的描述
// - 不记录完整的数据块副本
// - 恢复时只扫描 FC 区域
```

**性能对比**:
```
场景: 创建 100,000 个文件后断电恢复

传统 JBD2:
  - 日志大小: 数 GB
  - 恢复时间: 60-120 秒
  - 扫描范围: 整个日志区域

Fast Commit:
  - FC 区域: 数 MB
  - 恢复时间: 1-3 秒
  - 扫描范围: 仅 FC 区域
```

#### Fast Commit 架构

```
ext4 操作
  └─ ext4_fc_track_*()
       └─ 记录到 FC 区域
            └─ ext4_fc_commit()
                 ├─ 写入 FC Header
                 ├─ 写入 TLV tags
                 └─ 写入 FC Tail
```

**TLV 标签格式**:
```c
struct ext4_fc_tl {
    __le16  fc_tag;    // 标签类型
    __le16  fc_len;    // 数据长度
    __u8    fc_val[0]; // 数据
};
```

### 4.2 2022: Fast Commit 优化

#### 稳定性改进

```c
// 修复 FC 与并发操作的竞争条件
// 改进 FC 与完整提交的协调
// 处理 FC 回退到完整提交的情况
```

#### 性能调优

```c
// 优化 FC 区域大小
// 改进 TLV 编码效率
// 减少 FC 提交延迟
```

### 4.3 2023: iomap 迁移开始

#### iomap 框架

```c
// fs/ext4/inode.c
// 开始从 buffer_head 迁移到 iomap
// 目标: 统一的 I/O 映射框架

// iomap 优势:
// - 更简洁的代码
// - 更好的 large folio 支持
// - 与 XFS 共享代码路径
// - 更好的性能
```

**迁移状态**:
```
已完成:
  - 直接 I/O (dio)
  - fallocate
  - fiemap
  - seek data/hole

进行中:
  - 缓冲写入 (buffered write)
  - 缓冲读取 (buffered read)
```

### 4.4 其他重要改进 (2021-2023)

| 年份 | 内核版本 | 特性 | 描述 |
|------|---------|------|------|
| 2021 | 5.12 | **Fast Commit** | 革命性的恢复机制 |
| 2021 | 5.13 | 改进的 mballoc | 块分配器优化 |
| 2021 | 5.14 | 改进的 fsync | fsync 性能提升 |
| 2022 | 5.16 | Fast Commit 修复 | 稳定性改进 |
| 2022 | 5.17 | 改进的加密 | fscrypt 性能优化 |
| 2022 | 5.19 | 改进的 bigalloc | 大块分配优化 |
| 2023 | 6.1 | iomap dio 迁移 | 直接 I/O 迁移到 iomap |
| 2023 | 6.3 | 改进的 writeback | 回写优化 |
| 2023 | 6.5 | 改进的 extent 树 | extent 操作优化 |

## 5. 2024-2026: 大规模与架构革新

### 5.1 2024: 内存与并发优化

#### Xarray Mballoc 优化

```c
// fs/ext4/mballoc.c
// Baokun Li 主导开发
// 使用 xarray 替代传统位图扫描

// 传统方式:
// - 线性扫描位图
// - O(n) 复杂度
// - 缓存不友好

// Xarray 方式:
// - 树形结构
// - O(log n) 复杂度
// - 更好的缓存局部性
```

**性能提升**:
```
场景: 大文件系统块分配

传统:
  - 分配延迟: 高
  - 扫描范围: 大
  - 缓存命中: 低

Xarray:
  - 分配延迟: 降低 30-50%
  - 扫描范围: 减少
  - 缓存命中: 提高
```

#### Extent Status Tree Seq Counter

```c
// fs/ext4/es_tree.c
// 使用 seq counter 保护 ES tree
// 改进并发读取性能

// 传统方式:
// - 读写锁
// - 读取者互斥
// - 扩展状态缓存

// Seq counter 方式:
// - 无锁读取
// - 更好的并发
// - 减少锁竞争
```

### 5.2 2025: 大块大小与原子写入

#### Large Block Size (BS > PS)

```c
// fs/ext4/super.c
// Baokun Li, Zhang Yi 主导开发
// 支持块大小 > 页大小

// 传统限制:
// - 块大小 <= 页大小 (4KB)
// - 大页系统浪费空间

// 新支持:
// - 块大小可达 64KB
// - 更好的大 I/O 性能
// - 减少元数据开销
```

**关键宏**:
```c
// 逻辑块到页的转换
#define EXT4_LBLK_TO_PG(sbi, lblk) \
    ((lblk) >> (PAGE_SHIFT - (sbi)->s_blocksize_bits))

// 最小 folio order
sbi->s_min_folio_order = max(0, sbi->s_blocksize_bits - PAGE_SHIFT);
```

#### Atomic Write

```c
// fs/ext4/file.c
// 支持原子写入
// 保证写入的原子性

// 使用场景:
// - 数据库事务日志
// - 关键配置文件
// - 需要原子更新的场景
```

#### Orphan File 改进

```c
// fs/ext4/orphan.c
// 改进的孤儿文件处理
// 更好的并发清理
// 更可靠的恢复
```

### 5.3 2026: Mballoc 可扩展性改进

#### Linux 6.17 Mballoc 改进

```c
// 目标: 改善多核系统的块分配性能
// 减少全局锁竞争
// 改进组选择算法

// 关键改进:
// - 每 CPU 缓存
// - 改进的预分配
// - 更好的碎片管理
```

### 5.4 其他重要改进 (2024-2026)

| 年份 | 内核版本 | 特性 | 描述 |
|------|---------|------|------|
| 2024 | 6.6 | 改进的 iomap | 缓冲 I/O 迁移继续 |
| 2024 | 6.8 | ES tree seq counter | 并发优化 |
| 2024 | 6.10 | Xarray mballoc | 块分配器优化 |
| 2025 | 6.12 | Large block size | 块大小 > 页大小 |
| 2025 | 6.13 | Atomic write | 原子写入支持 |
| 2025 | 6.14 | Orphan file 改进 | 孤儿文件优化 |
| 2026 | 6.17 | Mballoc 可扩展性 | 多核性能改进 |

## 6. 关键特性时间线

### 6.1 完整特性时间线

```
2008  2.6.28  ext4 初始合并
              - Extent 树
              - Mballoc
              - 延迟分配
              - 48位块号

2009  2.6.30  稳定的 ext4
              - 默认启用 ext4

2010  2.6.35  改进的 ext4
              - 在线碎片整理
              - 改进的 fsync

2011  3.0     ext4 成熟
              - 改进的 mballoc
              - 更好的性能

2012  3.5     metadata_csum
              - 元数据校验和

2013  3.10    bigalloc, inline_data
              - 大块分配
              - 内联数据

2014  3.15    改进的 ext4
              - 更好的加密支持准备

2015  4.1     fscrypt
              - 文件系统级加密
              - casefold 开始

2016  4.8     metadata_csum 完善
              - 在线碎片整理改进

2017  4.12    fsverity
              - 文件完整性验证
              - 项目配额

2018  4.16    多队列 I/O
              - blk-mq 适配

2019  5.2     casefold 完成
              - UTF-8 大小写不敏感

2020  5.8     挂载 API 转换
              - fs_context

2021  5.12    Fast Commit ★
              - 革命性恢复机制
              - 挂载 API 完成

2022  5.17    Fast Commit 稳定
              - 性能优化

2023  6.1     iomap 迁移开始
              - 直接 I/O 迁移

2024  6.8     ES tree seq counter
              - Xarray mballoc

2025  6.12    Large block size ★
              - Atomic write
              - Orphan file 改进

2026  6.17    Mballoc 可扩展性
              - 多核性能改进
```

### 6.2 特性分类时间线

#### 存储效率
```
2008  Extent 树
2013  bigalloc
2013  inline_data
2025  Large block size
```

#### 数据完整性
```
2012  metadata_csum
2017  fsverity
2021  Fast Commit
2025  Atomic write
```

#### 安全性
```
2015  fscrypt
2019  casefold
2020  挂载 API
```

#### 性能
```
2008  Mballoc
2008  延迟分配
2018  多队列 I/O
2021  Fast Commit
2024  Xarray mballoc
2024  ES tree seq counter
2026  Mballoc 可扩展性
```

#### 架构
```
2020  挂载 API 转换
2023  iomap 迁移开始
2025  Large folio 支持
```

## 7. 主要贡献者

### 7.1 核心维护者

#### Theodore Ts'o (tytso)

```
角色: ext4 维护者 (2006-至今)
贡献:
  - ext4 整体架构设计
  - JBD2 日志层
  - 加密支持 (fscrypt)
  - Fast Commit 审查
  - 长期稳定性维护

背景:
  - ext2/ext3/ext4 的长期维护者
  - Linux 基金会研究员
  - 文件系统安全专家
```

#### Andreas Dilger

```
角色: ext3/ext4 原始架构师
贡献:
  - ext3 日志实现
  - ext4 初始设计
  - 大块分配概念
  - Lustre 文件系统

背景:
  - Sun/Oracle/Intel 工程师
  - 分布式文件系统专家
  - 现从事 Lustre 开发
```

### 7.2 核心开发者

#### Jan Kara

```
贡献领域:
  - 回写机制 (writeback)
  - 配额系统 (quota)
  - UDF 文件系统
  - ext4 回写优化

关键贡献:
  - 改进的 writeback 策略
  - 配额性能优化
  - 文件系统通用层改进
```

#### Ritesh Harjani

```
贡献领域:
  - Fast Commit (主要作者)
  - Mballoc 优化
  - 块分配器改进

关键贡献:
  - ext4_fc_replay() 实现
  - FC TLV 编码
  - mballoc 组选择优化
```

#### Baokun Li

```
贡献领域:
  - Large block size (BS>PS)
  - Xarray mballoc 优化
  - 内存管理改进

关键贡献:
  - EXT4_LBLK_TO_PG 宏
  - xarray 替代位图扫描
  - 大块大小兼容性处理
```

#### Zhang Yi

```
贡献领域:
  - Large folio 支持
  - 回写机制优化
  - 内存管理

关键贡献:
  - Large folio 与 ext4 兼容
  - writeback 性能优化
  - 内存分配改进
```

### 7.3 其他重要贡献者

| 贡献者 | 主要贡献 |
|--------|---------|
| Mingming Cao | 初始 ext4 实现 |
| Alex Tomas | Mballoc 原始作者 |
| Kalpak Shah | 早期 ext4 开发 |
| Johann Lombardi | 加密支持 |
| Michael Halcrow | eCryptfs/fscrypt |
| Eric Biggers | fscrypt/fsverity |
| Gabriel Krisman Bertazi | casefold |
| Ojaswin Mujoo | mballoc 优化 |
| Kemeng Shi | mballoc/回写 |
| Zhang Xiaoxu | 错误处理 |

### 7.4 贡献统计 (2015-2025)

```
Top contributors by commits (ext4 only):
  1. Theodore Ts'o: ~2000 commits
  2. Jan Kara: ~500 commits
  3. Ritesh Harjani: ~300 commits
  4. Baokun Li: ~200 commits
  5. Zhang Yi: ~150 commits
  6. Eric Biggers: ~100 commits
  7. Ojaswin Mujoo: ~80 commits
  8. Kemeng Shi: ~70 commits
```

## 8. 与 XFS/Btrfs 的对比

### 8.1 架构对比

| 特性 | ext4 | XFS | Btrfs |
|------|------|-----|-------|
| 日志 | JBD2 (外部) | 内部日志 | COW (无日志) |
| 分配 | Mballoc | 空闲 B+树 | 空闲树 |
| 映射 | Extent 树 | B+树 | B+树 |
| 校验和 | 元数据 | 元数据+数据 | 元数据+数据 |
| 快照 | 无 | 无 | 有 |
| 压缩 | 无 | 无 | 有 |
| RAID | 无 | 无 | 有 |
| 加密 | fscrypt | fscrypt | fscrypt |

### 8.2 性能对比

| 工作负载 | ext4 | XFS | Btrfs |
|---------|------|-----|-------|
| 小文件创建 | ★★★★ | ★★★ | ★★ |
| 大文件写入 | ★★★ | ★★★★★ | ★★★ |
| 并发写入 | ★★★ | ★★★★★ | ★★ |
| 删除大量文件 | ★★ | ★★★★ | ★★ |
| fsync 延迟 | ★★★★ | ★★★ | ★★ |
| 恢复时间 | ★★★★ (FC) | ★★★ | ★★ |

### 8.3 可靠性对比

| 特性 | ext4 | XFS | Btrfs |
|------|------|-----|-------|
| 元数据校验和 | 有 | 有 | 有 |
| 数据校验和 | 无 | 无 | 有 |
| 在线修复 | 有限 | 有限 | 有 |
| 离线修复 | e2fsck | xfs_repair | btrfs check |
| 恢复机制 | JBD2+FC | 日志 | COW |
| 成熟度 | ★★★★★ | ★★★★★ | ★★★ |

### 8.4 使用场景

#### ext4 最适合:
- 通用桌面/服务器
- 需要稳定性的场景
- 小到中型文件系统
- 需要快速恢复的场景 (Fast Commit)

#### XFS 最适合:
- 大型文件系统 (TB+)
- 高并发写入
- 媒体服务器
- 科学计算

#### Btrfs 最适合:
- 需要快照
- 需要压缩
- 需要 RAID
- 开发/测试环境

### 8.5 市场份额 (2025)

```
Linux 文件系统使用率估计:
  ext4:  ~60% (默认选择，最广泛)
  XFS:   ~25% (企业服务器，大存储)
  Btrfs: ~10% (桌面，NAS)
  其他:  ~5%  (ZFS, F2FS, etc.)
```

## 9. 未来方向

### 9.1 短期目标 (2025-2026)

#### iomap 完整迁移

```c
// 目标: 完全迁移到 iomap 框架
// 状态: 直接 I/O 已完成，缓冲 I/O 进行中

// 好处:
// - 代码简化
// - 与 XFS 共享路径
// - 更好的 large folio 支持
// - 性能提升
```

#### Mballoc 可扩展性

```c
// 目标: 改善多核系统的块分配
// 方法:
// - 每 CPU 缓存
// - 减少全局锁
// - 改进组选择算法
```

#### Large Block Size 完善

```c
// 目标: 完善 BS>PS 支持
// 解决:
// - 兼容性测试
// - 性能调优
// - 边缘情况处理
```

### 9.2 中期目标 (2026-2028)

#### 在线修复增强

```c
// 目标: 更强大的在线修复
// 可能包括:
// - 元数据在线修复
// - 更好的错误恢复
// - 自动修复机制
```

#### 性能持续优化

```c
// 目标: 适应新硬件
// 方向:
// - PCIe 5.0/6.0 SSD 优化
// - CXL 内存支持
// - 计算存储优化
```

#### 安全性增强

```c
// 目标: 更强的安全特性
// 可能包括:
// - 改进的加密
// - 完整性保护
// - 访问控制
```

### 9.3 长期愿景

#### ext4 的定位

```
ext4 不会成为最新的文件系统
但会是:
  - 最稳定的选择
  - 最广泛使用的文件系统
  - 兼容性最好的选择
  - 恢复最快的文件系统 (Fast Commit)

ext4 的未来不是添加新特性
而是:
  - 保持稳定性
  - 优化性能
  - 改进恢复
  - 适应新硬件
```

#### 与新技术的融合

```
CXL 内存:
  - 持久内存支持 (DAX)
  - 内存映射文件系统

计算存储:
  - 卸载元数据操作
  - 智能数据放置

AI 优化:
  - 自适应预取
  - 智能碎片整理
  - 工作负载感知分配
```

### 9.4 ext4 的生命周期

```
2008  诞生 (2.6.28)
2010  成熟 (成为默认选择)
2021  革新 (Fast Commit)
2025  现代化 (iomap, large block)
2030  稳定 (维护模式)
2035+ 长期支持 (关键修复)

ext4 预计会支持到 2040+
作为 Linux 的"安全选择"
```

## 10. 关键代码位置

### 10.1 核心文件

| 文件 | 作用 |
|------|------|
| `fs/ext4/super.c` | 挂载/卸载/超级块 |
| `fs/ext4/inode.c` | Inode 操作/读写 |
| `fs/ext4/namei.c` | 目录操作/文件名 |
| `fs/ext4/mballoc.c` | 多块分配器 |
| `fs/ext4/fast_commit.c` | Fast Commit |
| `fs/ext4/es_tree.c` | Extent Status Tree |
| `fs/ext4/xattr.c` | 扩展属性 |
| `fs/ext4/ioctl.c` | ioctl 接口 |
| `fs/ext4/sysfs.c` | sysfs 接口 |
| `fs/ext4/verity.c` | fsverity 支持 |
| `fs/ext4/orphan.c` | 孤儿文件 |

### 10.2 JBD2 日志层

| 文件 | 作用 |
|------|------|
| `fs/jbd2/journal.c` | 日志管理 |
| `fs/jbd2/transaction.c` | 事务管理 |
| `fs/jbd2/commit.c` | 事务提交 |
| `fs/jbd2/recovery.c` | 日志恢复 |
| `fs/jbd2/revoke.c` | 撤销管理 |
| `fs/jbd2/checkpoint.c` | 检查点 |

### 10.3 头文件

| 文件 | 作用 |
|------|------|
| `include/linux/ext4_fs.h` | ext4 常量/结构 |
| `include/linux/ext4.h` | ext4 内部结构 |
| `include/linux/jbd2.h` | JBD2 接口 |
| `fs/ext4/ext4.h` | ext4 私有定义 |

### 10.4 用户空间工具

| 工具 | 作用 |
|------|------|
| `e2fsck` | 文件系统检查 |
| `mke2fs` | 创建文件系统 |
| `resize2fs` | 在线调整大小 |
| `tune2fs` | 调整参数 |
| `e4defrag` | 碎片整理 |
| `dumpe2fs` | 显示信息 |
| `debugfs` | 调试工具 |
| `fsverity` | 完整性验证 |
