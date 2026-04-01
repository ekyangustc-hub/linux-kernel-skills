---
name: "ext4-fsverity"
description: "ext4 fs-verity集成专家。当用户询问ext4 fs-verity、Merkle树验证、fsverity_info查找、读取路径优化、fsverity与large folio兼容性时调用此技能。"
---

# ext4 Fs-verity

## 〇、为什么需要这个机制？

为什么需要 fs-verity？对于需要保证文件完整性的场景（如软件包、容器镜像），需要在读取时验证文件内容是否被篡改。fs-verity 使用 Merkle 树为文件构建密码学哈希，每次读取时验证。ext4 的 fs-verity 与 DAX 互斥，因为 DAX 绕过页缓存，无法集成验证逻辑。

传统的文件完整性检查（如 IMA/EVM）依赖于外部存储的哈希值，存在"谁来保护哈希值"的循环依赖问题。fs-verity 将 Merkle 树根哈希存储在文件 inode 中，读取路径自动验证，从根本上解决了信任链问题。

没有 fs-verity，系统就无法在文件系统层面提供高效的、硬件无关的文件完整性保护，容器镜像验证、安全启动等场景将依赖更复杂的方案。

---

## 一、Inode 标志

```c
#define EXT4_VERITY_FL		0x00100000	/* Verity protected inode */
#define EXT4_INODE_VERITY	20		/* bit position */
```

## 二、Feature Flag

```c
/* RO_COMPAT, not INCOMPAT */
#define EXT4_FEATURE_RO_COMPAT_VERITY	0x8000
```

## 三、DAX 互斥

```c
#define EXT4_DAX_MUT_EXCL (EXT4_VERITY_FL | EXT4_ENCRYPT_FL |\
			   EXT4_JOURNAL_DATA_FL | EXT4_INLINE_DATA_FL)
```

## 四、Inode 状态

```c
EXT4_STATE_VERITY_IN_PROGRESS	/* building fs-verity Merkle tree */

static inline bool ext4_verity_in_progress(struct inode *inode)
{
	return IS_ENABLED(CONFIG_FS_VERITY) &&
	       ext4_test_inode_state(inode, EXT4_STATE_VERITY_IN_PROGRESS);
}
```

## 五、读取路径 (readpage.c)

现代 ext4 将 `->read_folio` 和 `->readahead` 操作移至 `readpage.c`:

- `fsverity_info` 查找在 `ext4_mpage_readpages` 中只执行一次
- 查找结果用于 readahead、空洞验证和 I/O 完成工作队列
- 通过 `struct bio_post_read_ctx` 传递 `fsverity_info` 到 I/O 完成路径

## 六、Hash Readahead

在数据 I/O 提交时启动 Merkle 树哈希块的预读, 减少验证时的 I/O 延迟。

## 七、setattr 保护

fs-verity 文件拒绝大小变更 (`setattr_prepare`), 防止破坏 Merkle 树完整性。

## 八、Fast Commit Ineligible

启用 fs-verity 会标记 fast-commit ineligible, 因为构建 Merkle 树更新 inode 和孤儿状态不在 fast commit 回放标签描述范围内。

## 九、关键代码位置

| 功能 | 文件 |
|------|------|
| ext4 verity 实现 | fs/ext4/verity.c |
| ext4 读取路径 | fs/ext4/readpage.c |
| fsverity 核心 | fs/fsverity/ |
