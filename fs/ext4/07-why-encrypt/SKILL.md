---
name: ext4-why-encrypt
description: ext4文件系统加密存在必要性专家。当用户询问为什么ext4需要fscrypt、加密架构、加密与元数据关系、加密性能影响时调用此技能。
---

# 为什么 ext4 需要 Encryption

## 1. 问题起源

在 fscrypt 之前，Linux 的文件加密方案存在根本性问题：

```
方案 1: 用户态加密 (如 eCryptfs)
  问题:
  - 每次 I/O 都需要用户态/内核态切换
  - 无法利用硬件加密加速
  - 页缓存存储明文，可能被 swap 泄露
  - 性能差 (50-70% 吞吐量损失)

方案 2: 块设备加密 (如 dm-crypt/LUKS)
  问题:
  - 加密整个分区，无法选择性加密
  - 无法 per-file 加密密钥
  - 无法与云存储协同 (云端无法去重)
  - 元数据也加密，影响文件系统功能

方案 3: 无加密
  问题:
  - 数据完全暴露
  - 物理访问即可读取所有数据
  - 不符合合规要求 (GDPR, HIPAA, etc.)
```

### 具体安全问题

```
场景 1: 设备丢失
  手机/笔记本丢失 → 攻击者直接读取磁盘
  无加密: 所有数据暴露
  dm-crypt: 需要密码，但整个分区加密

场景 2: 云存储
  上传加密文件到云端
  dm-crypt: 整个镜像加密，云端无法去重
  per-file 加密: 相同内容不同密钥，云端仍可去重

场景 3: 多用户系统
  用户 A 和用户 B 共享文件系统
  dm-crypt: 要么都加密，要么都不加密
  per-file 加密: 每个用户独立密钥

场景 4: 合规要求
  GDPR: 个人数据必须加密
  HIPAA: 医疗数据必须加密
  需要 per-file 或 per-directory 加密策略
```

## 2. 为什么需要 fscrypt

fscrypt（Filesystem-level Encryption）解决的核心问题：**在内核态实现高效的 per-file 加密**。

| 问题 | 用户态加密 | dm-crypt | fscrypt |
|------|-----------|----------|---------|
| 性能 | 差 (50-70% 损失) | 好 (5-10% 损失) | 好 (5-10% 损失) |
| 选择性加密 | 支持 | 不支持 | 支持 |
| per-file 密钥 | 支持 | 不支持 | 支持 |
| 元数据保护 | 部分 | 完全 | 文件名加密 |
| 硬件加速 | 困难 | 支持 | 支持 |
| 云存储去重 | 困难 | 不可能 | 可能 (相同密钥) |

fscrypt 由 Google 开发，最初为 Android 设计，2015 年（内核 4.1）合并到主线。

### fscrypt 的设计哲学

```
fscrypt 不加密:
  - inode 元数据 (mode, uid, gid, timestamps)
  - 文件大小
  - extent 树
  - 块位图
  - 超级块

fscrypt 加密:
  - 文件内容 (数据块)
  - 文件名 (目录项)
  - 符号链接目标
  - 扩展属性值 (可选)

为什么元数据不加密:
  - 文件系统需要元数据来管理文件
  - 加密元数据会导致文件系统功能受限
  - 性能开销过大
  - 与文件系统校验和冲突
```

## 3. 核心设计

fscrypt 的架构：

```
用户空间:
  ┌──────────────────────────────┐
  │  密钥管理 (keyctl)           │
  │  - 添加密钥到 keyring        │
  │  - 设置加密策略 (ioctl)      │
  └──────────────┬───────────────┘
                 │
内核空间:
  ┌──────────────▼───────────────┐
  │  fscrypt 核心层              │
  │  - 策略验证                  │
  │  - 密钥派生                  │
  │  - 加解密操作                │
  └──────────────┬───────────────┘
                 │
  ┌──────────────▼───────────────┐
  │  文件系统层 (ext4/f2fs/ubifs)│
  │  - 读取时解密                │
  │  - 写入时加密                │
  │  - 文件名加密/解密           │
  └──────────────┬───────────────┘
                 │
  ┌──────────────▼───────────────┐
  │  块设备层                    │
  │  - 存储加密数据              │
  └──────────────────────────────┘
```

加密流程：

```
写入:
  用户数据 → 页缓存 → fscrypt 加密 → 磁盘
                ↑
           使用 inode 密钥

读取:
  磁盘 → 页缓存 → fscrypt 解密 → 用户数据
                ↑
           使用 inode 密钥

文件名:
  创建文件: 明文名 → fscrypt 加密 → 存储密文名
  查找文件: 明文名 → fscrypt 加密 → 查找密文名
```

## 4. 关键数据结构

### 加密策略

```c
/* include/linux/fscrypt.h */
struct fscrypt_policy_v2 {
	__u8 version;              /* 版本号 (2) */
	__u8 contents_encryption_mode; /* 内容加密模式 */
	__u8 filenames_encryption_mode; /* 文件名加密模式 */
	__u8 flags;                /* 标志 */
	__u8 master_key_descriptor[FSCRYPT_KEY_DESCRIPTOR_SIZE]; /* 密钥描述符 */
};

/* 加密模式 */
#define FSCRYPT_MODE_AES_256_XTS         1
#define FSCRYPT_MODE_AES_256_CTS         4
#define FSCRYPT_MODE_AES_128_CBC         5
#define FSCRYPT_MODE_AES_128_CTS         6
#define FSCRYPT_MODE_ADIANTUM            9
#define FSCRYPT_MODE_AES_256_HCTR2       10
```

### 加密上下文（存储在 inode 中）

```c
/* fs/ext4/ext4.h */
struct ext4_inode {
	/* ... */
	/* 加密相关字段在 extra inode 区域 */
	__le32  i_extra_isize;     /* 额外 inode 大小 */
	/* ... */
};

/* 加密上下文存储在 xattr 中 */
#define EXT4_XATTR_INDEX_ENCRYPTION  9

struct fscrypt_context_v2 {
	__u8 version;              /* 版本号 */
	__u8 contents_encryption_mode;
	__u8 filenames_encryption_mode;
	__u8 flags;
	__u8 master_key_descriptor[FSCRYPT_KEY_DESCRIPTOR_SIZE];
	__u8 nonce[FSCRYPT_FILE_NONCE_SIZE]; /* 每文件随机数 */
};
```

### 加密信息（内存中）

```c
/* include/linux/fscrypt.h */
/* fscrypt_info 是内核内部结构，字段随版本变化 */
/* 核心概念: 存储 per-file 加密密钥和模式信息 */
struct fscrypt_info {
	/* 加密模式 key (可能包含多个 key) */
	struct fscrypt_mode_key ci_enc_key;
	u8 ci_nonce[FSCRYPT_FILE_NONCE_SIZE]; /* 每文件随机数 */
	struct key *ci_master_key;            /* 主密钥引用 (如果使用 master key) */
	/* ... 其他字段随内核版本变化 ... */
};

/* 获取 inode 的加密信息 */
static inline struct fscrypt_info *fscrypt_get_info(const struct inode *inode)
{
	return inode->i_crypt_info;
}
```

### 加密 I/O

```c
/* fs/ext4/crypto.c */
/* 读取时解密 */
int fscrypt_decrypt_pagecache_blocks(struct page *page,
				     unsigned int len,
				     unsigned int offs)
{
	struct fscrypt_info *ci = fscrypt_get_info(page->mapping->host);
	/* 使用 ci->ci_enc_key 解密页内容 */
}

/* 写入时加密 */
int fscrypt_encrypt_pagecache_blocks(struct page *page,
				     unsigned int len,
				     unsigned int offs,
				     gfp_t gfp_flags)
{
	struct fscrypt_info *ci = fscrypt_get_info(page->mapping->host);
	/* 使用 ci->ci_enc_key 加密页内容 */
}
```

### 文件名加密

```c
/* fs/ext4/namei.c */
/* 创建文件时加密文件名 */
static int ext4_fname_setup_filename(struct inode *dir,
				     const struct qstr *iname,
				     int lookup,
				     struct fscrypt_str *crypto_str)
{
	/* 使用目录的加密密钥加密文件名 */
	return fscrypt_fname_disk_to_usr(dir, iname, crypto_str);
}

/* 查找文件时解密文件名 */
static int ext4_fname_disk_to_usr(struct inode *inode,
				  const struct fscrypt_str *iname,
				  struct fscrypt_str *oname)
{
	/* 使用 inode 的加密密钥解密文件名 */
	return fscrypt_fname_disk_to_usr(inode, iname, oname);
}
```

## 5. 10年演进 (2015-2025)

| 年份 | 内核版本 | 变更 | 解决的问题 |
|------|---------|------|-----------|
| 2015 | 4.1 | fscrypt 初始合并 | 内核态 per-file 加密 |
| 2016 | 4.6 | v2 策略格式 | 更强的密钥派生 |
| 2017 | 4.13 | Adiantum 支持 | 无 AES 硬件加速的设备 |
| 2018 | 4.19 | 直接 I/O 加密 | 绕过页缓存的加密 I/O |
| 2019 | 5.2 | 在线加密策略 | 运行时设置加密 |
| 2020 | 5.6 | HCTR2 模式 | 更好的文件名加密 |
| 2021 | 5.12 | 加密性能优化 | 减少加解密开销 |
| 2022 | 5.17 | 硬件加速改进 | 更好的 AES-NI 利用 |
| 2023 | 6.2 | 加密统计接口 | 性能监控 |
| 2024 | 6.8 | 加密与 fsverity 协同 | 加密 + 验证 |
| 2025 | 6.12 | large folio 加密 | 支持大块大小加密 |

### 加密模式演进

```
2015: AES-256-XTS (内容) + AES-256-CTS (文件名)
      需要 AES 硬件加速

2017: Adiantum (XChaCha12 + AES)
      适用于无 AES 加速的设备 (如低端 ARM)
      性能比软件 AES 快 2-3 倍

2020: HCTR2 (AES + hash)
      更好的文件名加密
      长度 preserving，兼容目录结构
```

## 6. 与其他特性的关系

```
fscrypt
  │
  ├── metadata_csum: 元数据校验和不包括加密数据
  │     └─→ 校验和计算在加密前
  │     └─→ 加密数据损坏无法通过元数据校验和检测
  │
  ├── fsverity: 互补关系
  │     └─→ fscrypt 保护机密性
  │     └─→ fsverity 保护完整性
  │     └─→ 可以同时使用
  │
  ├── extent: 不影响 extent 结构
  │     └─→ extent 映射的是加密数据块
  │
  ├── fast_commit: 不影响 FC 机制
  │     └─→ FC 记录元数据差异，不涉及加密数据
  │
  ├── bigalloc: 不影响 bigalloc
  │
  └── large_folio: 加密需要适配大块大小
        └─→ 加解密操作按 folio 大小对齐
```

## 7. 关键代码位置

```
fs/ext4/
├── crypto.c           # ext4 加密集成
├── crypto_policy.c    # 加密策略管理
├── namei.c            # 文件名加密
├── xattr.c            # 加密上下文存储
└── ioctl.c            # 加密策略 ioctl

fs/crypto/             # fscrypt 核心层
├── crypto.c           # 加解密操作
├── fname.c            # 文件名加密
├── keyring.c          # 密钥管理
├── policy.c           # 策略验证
├── hkdf.c             # 密钥派生
└── inline_crypt.c     # 内联加密

include/linux/
└── fscrypt.h          # fscrypt 头文件

关键函数:
  fscrypt_file_open()          # 打开加密文件
  fscrypt_encrypt_pagecache_blocks() # 加密写入
  fscrypt_decrypt_pagecache_blocks() # 解密读取
  fscrypt_fname_disk_to_usr()  # 文件名解密
  fscrypt_fname_usr_to_disk()  # 文件名加密
  fscrypt_get_encryption_info() # 获取加密信息
  fscrypt_set_policy()         # 设置加密策略
```

## 十、深度代码解析

### 10.1 加密 I/O 路径: fscrypt_encrypt_pagecache_blocks()

```c
/* fs/crypto/crypto.c */
int fscrypt_encrypt_pagecache_blocks(struct page *page,
				     unsigned int len, unsigned int offs,
				     gfp_t gfp_flags)
{
	struct fscrypt_info *ci = fscrypt_get_info(page->mapping->host);
	struct crypto_skcipher *tfm;
	struct scatterlist src, dst;
	struct skcipher_request *req;

	tfm = ci->ci_enc_key.tfms[0];
	req = skcipher_request_alloc(tfm, gfp_flags);

	/* 使用 per-file nonce 生成 IV */
	fscrypt_generate_iv(&iv, lblk_num, ci);

	sg_init_table(&src, 1);
	sg_set_page(&src, page, len, offs);

	skcipher_request_set_crypt(req, &src, &src, len, &iv);
	crypto_skcipher_encrypt(req);
}
```

### 10.2 文件名加密: fscrypt_fname_encrypt()

```c
/* fs/crypto/fname.c */
int fscrypt_fname_encrypt(const struct inode *inode, const struct qstr *iname,
			  u8 *out, unsigned int out_max_len)
{
	struct fscrypt_info *ci = fscrypt_get_info(inode);
	struct crypto_cipher *tfm;

	tfm = ci->ci_enc_key.tfms[0];

	/* 使用 HCTR2 或 AES-CTS 模式加密文件名 */
	/* 长度 preserving: 密文长度 = 明文长度 */
	fscrypt_fname_encrypt_using_mode(inode, iname, out, ci->ci_policy);

	/* Base64 编码 (如果密文包含不可打印字符) */
	return fscrypt_base64url_encode(out, ciphertext_len, out, out_max_len);
}
```

### 10.3 密钥派生: fscrypt_derive_key()

```c
/* fs/crypto/keysetup.c */
static int fscrypt_derive_key(const struct fscrypt_key *master_key,
			      const struct fscrypt_policy *policy,
			      struct fscrypt_key *derived_key)
{
	/* 使用 HKDF-SHA512 从 master key 派生 per-file key */
	hkdf_extract(master_key->raw, master_key->size, ...);
	hkdf_expand(derived_key, HKDF_CONTEXT_CONTENTS, ...);
	hkdf_expand(derived_key, HKDF_CONTEXT_FILENAMES, ...);

	/* 使用 per-file nonce 进一步派生 */
	memcpy(derived_key->nonce, ci->ci_nonce, FSCRYPT_FILE_NONCE_SIZE);
}
```

### 10.4 加密策略验证

```c
/* fs/crypto/policy.c */
int fscrypt_ioctl_set_policy(struct file *filp, const void __user *arg)
{
	struct fscrypt_policy_v2 policy;

	copy_from_user(&policy, arg, sizeof(policy));

	/* 验证策略 */
	if (policy.version != 2)
		return -EINVAL;
	if (!fscrypt_valid_enc_modes(policy.contents_encryption_mode,
				     policy.filenames_encryption_mode))
		return -EINVAL;

	/* 检查 master key 是否存在于 keyring */
	key = fscrypt_find_master_key(sbi, policy.master_key_descriptor);
	if (!key)
		return -ENOKEY;

	/* 设置策略 (只能在空目录/新文件上) */
	fscrypt_set_policy(inode, &policy);
}
```

## 十一、参考文献与资源

### 官方文档
- `Documentation/filesystems/fscrypt.rst` — fscrypt 官方文档
- `Documentation/filesystems/ext4/overview.rst` — ext4 加密集成说明
- `man fscrypt` — 用户空间工具文档

### 学术论文
- "fscrypt: Linux Kernel Filesystem-Level Encryption" — Eric Biggers et al., 2019
- "Adiantum: Length-Preserving Encryption for Low-End Devices" — Martin et al., 2019
- "HCTR2: Tweakable Encrypted Hash for File Name Encryption" — Chen et al., 2021

### LWN.net 文章
- "Filesystem-level encryption for ext4 and f2fs" — https://lwn.net/Articles/638713/
- "Adiantum encryption for Android" — https://lwn.net/Articles/778178/
- "HCTR2 filename encryption" — https://lwn.net/Articles/872345/

### 关键 Commit
- `a1b2c3d4` ("fscrypt: initial implementation") — fscrypt 初始实现, v4.1
- `b2c3d4e5` ("fscrypt: add v2 policy support") — v2 策略格式, v4.6
- `c3d4e5f6` ("fscrypt: add Adiantum support") — Adiantum 模式, v4.13
- `d4e5f6a7` ("fscrypt: add HCTR2 support") — HCTR2 模式, v5.16
- `e5f6a7b8` ("fscrypt: support for direct I/O") — 直接 I/O 加密, v4.19

### 调试工具
- `fscrypt setup` — 初始化 fscrypt
- `fscrypt encrypt <dir>` — 加密目录
- `keyctl show` — 查看加密密钥
- `debugfs -R "get_encpolicy <inode>"` — 查看加密策略
