---
title: 钱包密码学基础（Wallet Cryptography Basics）
module: 02-wallet
priority: P0
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-22
primary_sources:
  - https://www.secg.org/sec2-v2.pdf
  - https://datatracker.ietf.org/doc/html/rfc8032
  - https://datatracker.ietf.org/doc/html/rfc6979
  - https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
  - https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-bls-signature
  - https://github.com/bitcoin-core/secp256k1
---

# 钱包密码学基础（Wallet Cryptography Basics）

> **TL;DR**：Web3 钱包的"钥匙"建立在三件密码学工具之上：**非对称签名**（secp256k1 ECDSA、ed25519 EdDSA、BLS12-381 聚合、Schnorr BIP340）、**密码学哈希**（Keccak-256、SHA-256、SHA-512、Blake3）、**承诺 / 知识证明**（Pedersen、KZG、Groth16）。理解这些原语的 **数学定义、安全假设、随机数要求、聚合性** 是看清 EOA、MPC、硬件钱包、ZK 钱包差异的先决条件。尤为重要的两件事：ECDSA 必须使用确定性 k（RFC 6979）以避免 PS3 式密钥泄露；ed25519 是 Schnorr 家族成员、天然抗可变性且无需额外 nonce。

---

## 1. 背景与动机

1976 年 Diffie-Hellman 提出公钥密码；1985 年 Koblitz & Miller 独立提出椭圆曲线密码学（ECC）；2000 年 SEC 1/2 标准化 secp256k1。中本聪 2008 年选用 secp256k1（而非 NIST P-256）作为 Bitcoin 签名曲线，理由是 Koblitz 曲线参数 **可验证无后门**——`a=0, b=7` 全由常数构成，不依赖 NSA 提供的"随机种子"。此后几乎所有主流公链要么沿用 secp256k1（EVM 家族、BTC、Cosmos SDK）、要么采用 Ed25519（Solana、Sui、Aptos、NEAR、Polkadot ed25519 variant）。

以太坊 Beacon Chain（2020-12）另引入 **BLS12-381**，为 validator 签名聚合服务；Taproot（2021-11）激活 **Schnorr BIP340**，终结 Bitcoin 十余年来"能聚合但不能标准聚合"的尴尬。

## 2. 核心原理

### 2.1 形式化定义：数字签名方案

数字签名方案是三元组 `(Gen, Sign, Verify)`：

- `Gen(1^λ) → (sk, pk)`：给定安全参数 λ，生成私钥 / 公钥对。
- `Sign(sk, m) → σ`：对消息 m 签名。
- `Verify(pk, m, σ) → {0,1}`：验证签名。

安全性要求 **EUF-CMA**（存在性不可伪造，抗选择消息攻击）：敌手在看到任意多已选消息签名后，无法伪造新消息的合法签名。所有主流钱包签名方案皆在相应假设下达到 EUF-CMA：ECDSA 依赖 ECDLP（椭圆曲线离散对数问题）+ 随机预言机；ed25519 依赖 ECDLP + 前向安全；BLS 依赖 co-CDH + gap-DH；Schnorr 依赖 ECDLP + ROM（论文 Pointcheval-Stern 1996）。

### 2.2 关键算法 / 数据结构

**secp256k1 曲线**（`y² = x³ + 7 mod p`）：

| 字段 | 值 |
| --- | --- |
| 素数 p | 2²⁵⁶ − 2³² − 977 |
| 阶 n | 0xFFFFFFFF...BAAEDCE6AF48A03BBFD25E8CD0364141 |
| 余因子 h | 1 |
| 基点 G | 公开常数 (Gx, Gy) |

公钥 `pk = sk · G`，其中 `sk ∈ [1, n-1]`。未压缩公钥 65 字节 `04 || X || Y`，压缩公钥 33 字节 `02/03 || X`（偶/奇选择 Y）。以太坊地址 = `keccak256(X‖Y)[12:]`（去掉前 12 字节，取后 20 字节）。

**ECDSA 签名**（给定消息 hash `z`）：

```
1. 取随机 k ∈ [1, n-1]
2. (x₁, y₁) = k · G; r = x₁ mod n
3. s = k⁻¹ (z + r·sk) mod n
4. 输出 (r, s)   (以太坊还附加 v ∈ {27,28} 做公钥恢复)
```

**致命陷阱**：若两次签名复用同一 k，则 `sk = (z₁ - z₂) / (s₁ - s₂) · ... mod n` 可直接解出。Sony PS3 2010 年被玩家用此漏洞破解；Bitcoin 早期 Android 钱包也栽过（`SecureRandom` 弱熵）。**解决方案**：RFC 6979 确定性 nonce `k = HMAC-SHA256(sk, z || counter)`，使 k 唯一且不依赖外部随机源。所有现代钱包库（`bitcoinjs-lib`, `ethers.js`, `secp256k1`）默认启用 RFC 6979。

**ed25519（EdDSA, Curve25519 扭曲爱德华兹曲线）**：

- 曲线：`−x² + y² = 1 − (121665/121666) x² y²` (Edwards form)
- 素数 p = 2²⁵⁵ − 19；基点 B 阶 ℓ = 2²⁵² + ...
- 私钥 32 字节，经 `H = SHA-512(sk)` → `prefix(32) || seed(32)`，seed clamp 后 · B 得公钥。
- 签名 `σ = (R, S)`，R = r·B，r = SHA-512(prefix || m)，S = r + H(R || pk || m) · s。

优势：**确定性 nonce 内嵌（天然抗 k 重用）**、无条件分支（抗侧信道）、公钥 32 字节、签名 64 字节、速度比 ECDSA 快 ~20%。Solana、Sui、Aptos 因此选 ed25519。

**BLS12-381**（聚合签名）：

- 两条配对友好曲线 G1、G2，pairing `e: G1 × G2 → GT`。
- Ethereum beacon 采用 "min-pubkey-size"（公钥在 G1，48 B；签名在 G2，96 B）。
- 聚合：`σ_agg = σ₁ + σ₂ + ... + σ_n`（G2 上加法），公钥同法聚合。
- 验证：`e(G1_gen, σ_agg) == e(pk_agg, H(m))`。
- 必须使用 **Proof-of-Possession (PoP)** 防 rogue-key 攻击：每个 validator 注册时提交 `σ_pop = Sign(sk, "BLS_POP" || pk)`。

**Schnorr BIP340**：

- 定义在 secp256k1 上，签名 `σ = (R, s)`，s = r + H(R || P || m) · sk，R = r·G。
- 关键特性：**线性**。Alice + Bob 的联合签名 `(R_A + R_B, s_A + s_B)` 对公钥 `P_A + P_B` 合法——这正是 MuSig2 / FROST 阈值签名的基础。
- BIP340 使用 x-only 公钥（32 B）+ tagged hash（域分离）。Taproot 利用它实现 key-path spend。

### 2.3 子机制拆解

1. **公私钥生成**：从 CSPRNG 取 32 字节，拒绝采样至 `[1, n-1]`。
2. **消息哈希**：EVM 用 `keccak256`；Bitcoin 用 `SHA256(SHA256(m))`（double-SHA256）；Solana 用 ed25519 内嵌 SHA-512。
3. **签名**：依曲线算法。
4. **公钥恢复**：secp256k1 独有——给 `(r, s, v)` 可解出 `pk = r⁻¹ (s·R - z·G)`，以太坊地址由此计算。省掉"随 tx 附带公钥"开销。
5. **聚合 / 门限**：BLS、Schnorr 线性可聚合；ECDSA 无直接聚合，但可通过 TSS（Paillier + Pedersen 承诺）实现 GG18 门限 ECDSA。
6. **承诺方案**：Pedersen `C = r·G + v·H` 用于 Monero / Grin；KZG 承诺用于 EIP-4844 Blob。

### 2.4 参数与常量

| 算法 | 曲线 | 私钥 | 公钥 | 签名 | 安全级别 |
| --- | --- | --- | --- | --- | --- |
| ECDSA | secp256k1 | 32 B | 33 B (压缩) | 64 B (+1 v) | ~128 bit |
| EdDSA | ed25519 | 32 B | 32 B | 64 B | ~128 bit |
| Schnorr | secp256k1 (BIP340) | 32 B | 32 B (x-only) | 64 B | ~128 bit |
| BLS | BLS12-381 | 32 B | 48 B (G1) | 96 B (G2) | ~128 bit |
| P-256 | secp256r1 | 32 B | 33 B | 64 B | ~128 bit |

### 2.5 边界条件与失败模式

- **k 重用**：ECDSA 致命。
- **Rogue-key**：BLS/Schnorr 聚合时若公钥由对手选，可伪造。修复：PoP 或 MuSig2 nonce 交换。
- **Low-order points**：ed25519 需检查 R、A 不在 small subgroup；历史上 Monero 被利用铸造假币。
- **签名可变性**：ECDSA `(r, s)` 与 `(r, n-s)` 都合法 → BIP62 / EIP-2 强制 low-s。
- **侧信道**：secp256k1 常数时间实现难（libsecp256k1 做到），ed25519 天然常数时间。
- **量子威胁**：Shor 算法可破 ECDLP；目前 NIST 推进 PQC（ML-DSA, SLH-DSA），但尚无主流钱包落地。

### 2.6 签名流程（Mermaid）

```mermaid
sequenceDiagram
    participant U as 用户 / 钱包
    participant H as Hash(m)
    participant S as Signer(sk)
    participant V as Verifier(pk)

    U->>H: keccak256(rlpEncode(tx))
    H->>S: z (32 bytes)
    S->>S: k = HMAC(sk, z) (RFC 6979)
    S->>S: R = k·G; r = R.x mod n
    S->>S: s = k⁻¹(z + r·sk) mod n
    S->>V: (r, s, v)
    V->>V: w = s⁻¹; u1 = z·w; u2 = r·w
    V->>V: P = u1·G + u2·pk; check P.x ?= r
```

## 3. 架构剖析

### 3.1 分层视图：钱包密码学栈

```
┌──────────────────────────────────────────────┐
│ 应用层   Wallet / Signer API (ethers, web3)  │
├──────────────────────────────────────────────┤
│ 协议层   BIP32/39/44, EIP-712, EIP-191        │
├──────────────────────────────────────────────┤
│ 算法层   ECDSA / EdDSA / Schnorr / BLS / TSS  │
├──────────────────────────────────────────────┤
│ 原语层   ECC 点运算 / Hash / HMAC / PBKDF2    │
├──────────────────────────────────────────────┤
│ 实现层   libsecp256k1 / libsodium / blst      │
└──────────────────────────────────────────────┘
```

### 3.2 核心模块清单

| 模块 | 职责 | 参考实现 | 可替换性 |
| --- | --- | --- | --- |
| Curve Arithmetic | 点加/点倍/标量乘 | libsecp256k1 / curve25519-dalek / blst | 低 |
| Hash | Keccak-256/SHA-256/SHA-512 | tiny-keccak / ring | 中 |
| RNG | 安全随机数 | OS CSPRNG / TRNG（SE 内）| 低 |
| KDF | 种子派生 | PBKDF2 / HKDF / scrypt | 中 |
| BIP32 Derivation | HD 分层 | bip32.js / bip_utils | 高 |
| Signer | 签名构造 | ethers.Wallet / solana/web3.js Keypair | 高 |
| Aggregator | BLS/Schnorr 聚合 | blst / MuSig2 impls | 高 |
| TSS Library | 门限签名 | tss-lib (Binance) / multi-party-ecdsa (ZenGo) | 中 |
| Verifier | 签名校验 | EVM ECRECOVER 预编译 | 低 |
| Secure Storage | 密钥保护 | OS Keychain / SE / Enclave | 低 |

### 3.3 数据流：一次消息签名

- 消息 m（如 RLP 编码的 tx 或 EIP-712 structured data）
- 哈希 z = H(m)
- 读取 sk（KeyVault 解锁 or HW 内部）
- 生成 σ = Sign(sk, z)
- 广播 (m, σ) 或 (m, σ, pk)，链上节点 Verify(pk, z, σ)

每一步的可观测性：ethers.js 提供 `signer.signTypedData`；硬件钱包将 `Display → Approve → Sign` 三步拆开；MPC 钱包将 `Sign` 展开为多轮 MPC 协议（2–4 轮 round trip）。

### 3.4 客户端多样性 / 参考实现

- **libsecp256k1**（C，Bitcoin Core 维护）：secp256k1 事实标准，常数时间、内嵌 RFC 6979。
- **libsodium**（C，NaCl 后继）：ed25519 事实标准。
- **blst**（C/Rust，Supranational）：BLS12-381 高性能库，Ethereum consensus client 通用。
- **arkworks**（Rust）：通用曲线、配对、ZK 原语。
- **noble-curves**（TypeScript，Paul Miller）：纯 JS 实现全曲线，审计过，MetaMask 部分采用。
- **Web Crypto API**：浏览器内置，支持 P-256 但不支持 secp256k1。

### 3.5 扩展接口

- **EIP-191**：`"\x19Ethereum Signed Message:\n" + len + msg` 个人签名前缀。
- **EIP-712**：结构化数据签名（domain separator + typed data hash），EIP-2612 Permit / Safe 多签的底座。
- **EIP-2098**：紧凑签名（64 B 替代 65 B）。
- **EIP-7212**：secp256r1 预编译（0x100），为 Passkey / WebAuthn 钱包准备。
- **EIP-6493**：SSZ 交易签名标准（未来 Purge 阶段）。

## 4. 关键代码 / 实现细节

libsecp256k1 核心签名函数（简化自 `secp256k1/src/ecdsa_impl.h`，commit `v0.5.1`）：

```c
static int secp256k1_ecdsa_sig_sign(
    const secp256k1_ecmult_gen_context *ctx,
    secp256k1_scalar *sigr, secp256k1_scalar *sigs,
    const secp256k1_scalar *seckey, const secp256k1_scalar *message,
    const secp256k1_scalar *nonce, int *recid)
{
    secp256k1_gej rp;
    secp256k1_ecmult_gen(ctx, &rp, nonce);            // R = k·G
    secp256k1_ge_set_gej(&r, &rp);
    secp256k1_scalar_set_b32(sigr, r.x.b32, &overflow); // r = R.x mod n
    if (secp256k1_scalar_is_zero(sigr)) return 0;

    secp256k1_scalar_mul(&n, sigr, seckey);           // n = r·sk
    secp256k1_scalar_add(&n, &n, message);            // n = z + r·sk
    secp256k1_scalar_inverse(&k_inv, nonce);          // k⁻¹
    secp256k1_scalar_mul(sigs, &k_inv, &n);           // s = k⁻¹(z+r·sk)
    if (secp256k1_scalar_is_high(sigs))               // low-s 规范
        secp256k1_scalar_negate(sigs, sigs);
    return 1;
}
```

> 省略：边界检查、常数时间保护、错误处理。真实路径 `secp256k1_ecdsa_sign` 会先用 `nonce_function_rfc6979` 算出 k，避免外部传入。

## 5. 演进与版本对比

| 年份 | 里程碑 | 意义 |
| --- | --- | --- |
| 1985 | ECC 提出 | 理论奠基 |
| 2000 | SEC 1/2 标准 | secp256k1 规范化 |
| 2008 | Bitcoin 白皮书选 secp256k1 | 实际应用 |
| 2011 | ed25519 发布 | 高性能替代 |
| 2013 | RFC 6979 | 确定性 nonce |
| 2015 | EIP-155 chain_id | 跨链防重放 |
| 2020 | Beacon Chain BLS | 聚合签名上链 |
| 2021 | BIP340 Schnorr | Taproot 激活 |
| 2024 | EIP-7212 | P-256 预编译 (Passkey) |
| 2025 | Pectra, EIP-7002 | BLS withdrawals 增强 |

## 6. 实战示例

ethers.js 原生 ECDSA 签名（Node.js）：

```javascript
import { Wallet, hashMessage, verifyMessage } from "ethers";
const w = Wallet.createRandom();
const msg = "Hello Web3";
const sig = await w.signMessage(msg);        // EIP-191 personal_sign
const signer = verifyMessage(msg, sig);       // 恢复签名者
console.log(signer === w.address);            // true
```

BLS 聚合（`@noble/curves/bls12-381`）：

```javascript
import { bls12_381 as bls } from "@noble/curves/bls12-381";
const sks = [bls.utils.randomPrivateKey(), bls.utils.randomPrivateKey()];
const pks = sks.map(sk => bls.getPublicKey(sk));
const msg = new TextEncoder().encode("attest slot 42");
const sigs = sks.map(sk => bls.sign(msg, sk));
const aggSig = bls.aggregateSignatures(sigs);
const aggPk  = bls.aggregatePublicKeys(pks);
console.log(bls.verify(aggSig, msg, aggPk)); // true
```

## 7. 安全与已知攻击

| 事件 | 年份 | 根因 | 教训 |
| --- | --- | --- | --- |
| Sony PS3 ECDSA | 2010 | k 常量 | 必须确定性 nonce |
| Android bitcoinj | 2013 | SecureRandom 弱熵 | 熵源审计 |
| Monero double-spend | 2017 | 不校验 low-order point | 必须子群检查 |
| ed25519 malleability debate | 2018+ | ed25519-donna 不同实现不一致 | 规范化（ZIP-215） |
| Ledger Fakechain | 2022 | 无 EIP-155 防护 | 跨链签名重放 |
| Paradigm P-256 预编译讨论 | 2023 | NIST 参数来源质疑 | 开源、可验证常数 |

## 8. 与同类方案对比

| 维度 | ECDSA | EdDSA | Schnorr | BLS |
| --- | --- | --- | --- | --- |
| 曲线 | secp256k1/r1 | Curve25519 | secp256k1 | BLS12-381 |
| 确定性 nonce | 需 RFC 6979 | 天然 | 需 BIP340 规范 | 天然 |
| 聚合 | ✗（需 TSS）| ✗ | ✓（MuSig2）| ✓（原生）|
| 签名大小 | 64 B | 64 B | 64 B | 96 B |
| 验签速度 | 中 | 快 | 中 | 慢（pairing）|
| 量子安全 | ✗ | ✗ | ✗ | ✗ |
| 典型链 | BTC/ETH | SOL/Sui | BTC Taproot | ETH validator |

## 9. 延伸阅读

- **官方规范**：SEC 1/2、RFC 6979、RFC 8032、BIP340、IRTF BLS draft。
- **实现库**：libsecp256k1、libsodium、blst、noble-curves。
- **论文**：Bernstein "High-speed high-security signatures"（ed25519）；Boneh-Lynn-Shacham "Short Signatures from the Weil Pairing"。
- **博客**：a16z "An in-depth look at Ethereum cryptography"；Cloudflare "ECDSA: Handle with Care"。
- **EIP**：EIP-191、EIP-712、EIP-2098、EIP-7212。

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| ECDLP | Elliptic Curve Discrete Log Problem | 椭圆曲线离散对数问题 |
| CSPRNG | Cryptographically Secure PRNG | 密码学安全随机数 |
| RFC 6979 | — | 确定性 ECDSA nonce 标准 |
| EUF-CMA | Existential Unforgeability under CMA | 签名安全基准 |
| PoP | Proof of Possession | BLS 抗 rogue-key 机制 |
| TSS | Threshold Signature Scheme | 门限签名 |
| Pairing | Bilinear Map | 配对双线性映射 |
| Tagged Hash | — | 带域分隔的哈希（BIP340）|

---

*Last verified: 2026-04-22*
