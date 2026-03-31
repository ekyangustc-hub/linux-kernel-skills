---
name: "ext4-fsverity"
description: "ext4 fs-verity集成专家。当用户询问ext4 fs-verity、Merkle树验证、fsverity_info查找、读取路径优化、fsverity与large folio兼容性时调用此技能。"
---

# ext4 Fs-verity 集成

## 一、概述

Fs-verity 是 Linux 内核的文件完整性验证机制，ext4 通过 Merkle 树实现文件数据的只读验证。近年来经历了大规模重构，包括查找优化、读取路径整合、large folio 支持等。

---

## 二、核心概念

### 2.1 Merkle 树

```
                    ┌─────────────┐
                    │   Root Hash │ (存储在 inode 扩展属性中)
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Hash 0,1 │ │ Hash 2,3 │ │ Hash 4,5 │ (中间层)
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │            │
      ┌──────┴──────┐     │     ┌──────┴──────┐
      ▼             ▼     ▼     ▼             ▼
  ┌────────┐  ┌────────┐ ┌────────┐  ┌────────┐
  │ Data 0 │  │ Data 1 │ │ Data 2 │  │ Data 3 │ (数据块)
  └────────┘  └────────┘ └────────┘  └────────┘
```

### 2.2 关键结构

**位置**: `include/linux/fsverity.h`

```c
struct fsverity_info {
    struct fsverity_descriptor *desc;  /* Merkle 树描述符 */
    struct ahash_request *hash_req;    /* 哈希请求 */
    struct merkle_tree_params tree_params; /* 树参数 */
    struct file *file;                 /* 所属文件 */
    ...
};
```

---

## 三、Fs-verity 读取路径优化

### 3.1 重构概述

Christoph Hellwig 主导的 fsverity 重构系列 (2026年初) 对读取路径进行了全面优化:

| 变更 | 说明 |
|------|------|
| 哈希查找优化 | 使用 hashtable 替代线性查找 |
| fsverity_info 查找合并 | 在 `ext4_mpage_readpages` 中只查找一次 |
| 读取代码整合 | `->read_folio` 和 `->readahead` 移至 `readpage.c` |
| 哈希预读 | 在数据 I/O 提交时启动 hash readahead |
| fsverity_info 位置 | 移至 fs-specific inode 部分 |

### 3.2 读取路径流程

```
ext4_mpage_readpages()
    │
    ├── 1. 查找 fsverity_info (仅一次)
    │   └── fsverity_get_info(inode)
    │       └── 使用 hashtable 查找
    │
    ├── 2. 遍历请求的页
    │   │
    │   ├── 2.1 普通读取
    │   │   └── 提交 bio
    │   │
    │   └── 2.2 fs-verity 验证
    │       ├── 启动 hash readahead (在 I/O 提交时)
    │       ├── bio_post_read_ctx 携带 fsverity_info
    │       └── I/O 完成后验证 Merkle 树哈希
    │
    └── 3. I/O 完成
        └── ext4_finish_bio()
            └── fsverity 验证回调
```

### 3.3 fsverity_info 查找优化

**旧流程**:
```c
/* 每次读取操作都查找 fsverity_info */
for each page:
    info = fsverity_get_info(inode);  /* 重复查找 */
    verify_page(info, page);
    fsverity_put_info(info);
```

**新流程**:
```c
/* 只查找一次，通过 bio_post_read_ctx 传递 */
info = fsverity_get_info(inode);  /* 一次查找 */
ctx->fsverity_info = info;
submit_bio(bio);
/* I/O 完成后使用 ctx->fsverity_info 验证 */
```

### 3.4 Hash Readahead

```c
/* fsverity: kick off hash readahead at data I/O submission time */
/* 在数据 I/O 提交时启动 Merkle 树哈希块的预读 */
/* 减少验证时的 I/O 延迟 */
```

---

## 四、Fs-verity 与 Large Folio

### 4.1 支持状态

```c
/* ext4: support verifying data from large folios with fs-verity */
/* 支持从 large folios 验证数据完整性 */
```

### 4.2 不兼容性检查

当 BS > PS (块大小 > 页大小) 时，以下特性不兼容 fs-verity:

| 特性 | 原因 |
|------|------|
| 加密 (encrypt) | 尚未支持 large folio |
| fs-verity | 需要 large folio 验证支持 |

```c
/* 挂载时检查 */
if (block_size > PAGE_SIZE && !large_folio_supported) {
    /* 拒绝挂载 */
}
```

---

## 五、Fs-verity 启用流程

### 5.1 启用步骤

```
ext4_ioctl_setflags(..., FS_VERITY_FL)
    │
    ├── 1. 构建 Merkle 树
    │   └── fsverity_build_merkle_tree()
    │       ├── 写入 Merkle 树块
    │       └── 计算根哈希
    │
    ├── 2. 更新 inode
    │   ├── 设置 FS_VERITY_FL 标志
    │   └── 存储 Merkle 树参数
    │
    ├── 3. 标记 fast-commit ineligible
    │   └── 强制完整提交 (确保一致性)
    │
    └── 4. 清理孤儿 inode
```

### 5.2 Fast Commit 不兼容

启用 fs-verity 会标记 fast-commit ineligible，因为:
- 构建 Merkle 树更新 inode 和孤儿状态
- 这些操作不在 fast commit 回放标签描述范围内
- 强制回退到完整提交确保一致性

---

## 六、setattr 保护

### 6.1 大小变更拒绝

```c
/* fs,fsverity: reject size changes on fsverity files in setattr_prepare */
/* fs-verity 文件拒绝大小变更 */
/* 防止破坏 Merkle 树完整性 */
```

### 6.2 fsverity_info 清理

```c
/* fs,fsverity: clear out fsverity_info from common code */
/* 在通用代码中清理 fsverity_info */
/* 确保文件状态变更时正确清理验证信息 */
```

---

## 七、代码位置

| 功能 | 文件路径 |
|------|----------|
| ext4 verity 实现 | `fs/ext4/verity.c` |
| ext4 读取路径 | `fs/ext4/readpage.c` |
| fsverity 核心 | `fs/fsverity/` |
| fsverity 头文件 | `include/linux/fsverity.h` |
| fast-commit 标记 | `fs/ext4/fast_commit.c` |

---

## 八、参考资源

- fs-verity 文档: `Documentation/filesystems/fsverity.rst`
- ext4 verity 代码: `fs/ext4/verity.c`
- 补丁系列: "fsverity: consolidate pagecache code" (2026)
