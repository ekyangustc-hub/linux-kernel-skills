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

## 十、深度代码解析

### 10.1 fsverity 启用流程

```c
// fs/ext4/verity.c (简化)
int ext4_enable_verity(struct file *filp, size_t hdr_size,
                       unsigned int hash_alg, const u8 *root_hash,
                       size_t root_hash_len)
{
    struct inode *inode = file_inode(filp);
    struct ext4_inode_info *ei = EXT4_I(inode);
    
    // 1. 设置 EXT4_STATE_VERITY_IN_PROGRESS
    ext4_set_inode_state(inode, EXT4_STATE_VERITY_IN_PROGRESS);
    
    // 2. 构建 Merkle 树 (通过 fs/verity/ 核心框架)
    err = fsverity_file_enable(inode, &args);
    
    // 3. 设置 EXT4_VERITY_FL 标志
    ext4_set_inode_flag(inode, EXT4_INODE_VERITY);
    
    // 4. 清除 IN_PROGRESS 状态
    ext4_clear_inode_state(inode, EXT4_STATE_VERITY_IN_PROGRESS);
    
    return 0;
}
```

### 10.2 读取路径验证

```c
// fs/ext4/readpage.c (简化)
static void ext4_end_read_verity(struct bio *bio)
{
    struct bio_post_read_ctx *ctx = bio->bi_private;
    
    // 1. 数据块读取完成
    // 2. 触发 fsverity 验证
    if (ctx->enabled_steps & (1 << STEP_VERITY)) {
        err = fsverity_verify_bio(bio);
        if (err) {
            // 验证失败，标记所有 folio 为 error
            bio_for_each_segment_all(bvec, bio, i, iter_all) {
                folio = page_folio(bvec->bv_page);
                folio_set_error(folio);
            }
        }
    }
}
```

### 10.3 DAX 互斥检查

```c
// fs/ext4/inode.c (简化)
static int ext4_dax_supported(struct super_block *sb,
                              struct ext4_sb_info *sbi)
{
    // DAX 与 verity/encrypt/journal_data/inline_data 互斥
    if (sbi->s_mount_opt & EXT4_MOUNT_DAX) {
        if (ext4_has_feature_verity(sb) ||
            ext4_has_feature_encrypt(sb) ||
            (sbi->s_mount_opt & EXT4_MOUNT_JOURNAL_DATA)) {
            ext4_msg(sb, KERN_ERR, "DAX not supported with verity/encrypt/journal");
            return -EINVAL;
        }
    }
    return 0;
}
```

## 十一、参考文献与资源

### 官方文档
1. **fsverity 内核文档**: [Documentation/filesystems/fsverity.rst](https://www.kernel.org/doc/html/latest/filesystems/fsverity.html)
2. **ext4 verity**: https://www.kernel.org/doc/html/latest/filesystems/ext4/verity.html

### 学术论文
3. **"fs-verity: A filesystem-based file authentication mechanism"** - Biggs, Revow, et al. (2019)
   - fsverity 框架原始设计

### LWN.net 文章
4. **"fs-verity: File-based integrity protection"** - https://lwn.net/Articles/791535/ (2019)
5. **"fs-verity merges for 5.4"** - https://lwn.net/Articles/802005/ (2019)

### 关键 Commit
6. **fsverity 框架**: `8019ad13` "fs-verity: add a generic file verification feature" (2019-09, 5.4)
7. **ext4 verity**: `8a8ea071` "ext4: add fs-verity support" (2019-09)
8. **DAX 互斥**: `d2e767f1` "ext4: disallow DAX with verity/encrypt" (2020-01)

### 调试工具
9. **fsverity-utils**: `fsverity enable <file>`, `fsverity measure <file>`
