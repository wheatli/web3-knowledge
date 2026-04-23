---
title: 秘密分享（Shamir / Additive / XOR Secret Sharing）
module: 07-privacy/mpc
priority: P1
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-22
primary_sources:
  - https://dl.acm.org/doi/10.1145/359168.359176 (Shamir 1979)
  - https://doi.org/10.1109/AFIPS.1979.98 (Blakley 1979)
  - https://eprint.iacr.org/2011/483 (Katz Verifiable SS review)
  - https://github.com/dalek-cryptography/curve25519-dalek
  - https://github.com/coinbase/kryptology/tree/master/pkg/sharing
---

# 秘密分享（Shamir / Additive / XOR Secret Sharing）

> **TL;DR**：秘密分享把秘密 $s$ 拆成 $n$ 份，任意 $\ge t$ 份可重构、$< t$ 份则信息论上 **毫无泄漏**。三大基础方案：Shamir SSS 基于有限域多项式插值，Additive SS 简单求和适合 MPC，XOR SS 是 Additive 在 GF(2) 上的特例。它们是所有 MPC 协议（GMW/BGW/SPDZ）、阈值签名（FROST、GG20）、以及冷钱包分发的底层砖块。

## 1. 背景与动机

1979 年 Shamir 和 Blakley 独立提出秘密分享，最初用于 "核按钮" 之类需要多方共同授权的场景。核心动机：把"单点信任"拆成"门限信任"。

现代应用：
- **钱包备份**：Trezor Shamir Backup、Coinbase Wallet as a Service。
- **阈值签名**：节点持有私钥的份额而非完整私钥。
- **MPC 协议**：每个秘密中间值以分享形式在节点间流转。
- **Key Management Service**：AWS KMS、Hashicorp Vault 的高阶模式。
- **FHE 委员会**：阈值解密网关（Zama KMS）持有 sk 的 Shamir 份额。

三种秘密分享各有其强项：
- Shamir：信息论安全、门限 $t$ 可任意选；代价是有限域运算。
- Additive：实现最简，速度最快；只支持 $t = n$（全部参与）。
- XOR：bit-level 版 additive，硬件友好。

## 2. 核心原理

### 2.1 形式化定义

一个 $(t, n)$ 秘密分享方案由两个 PPT 算法 $(\mathrm{Share}, \mathrm{Reconstruct})$ 构成：

- $\mathrm{Share}(s) \to (s_1, \dots, s_n)$：把秘密 $s \in \mathcal{S}$ 拆成 $n$ 份，每份给一方。
- $\mathrm{Reconstruct}(\{s_i\}_{i \in I}) \to s$：若 $|I| \ge t$，输出 $s$；若 $|I| < t$，失败。

**安全性（Perfect Secrecy, Shannon 意义）**：对任意两个秘密 $s_0, s_1$ 与任意 $|I| < t$，
$$\{(s_i)_{i \in I}\}_{\mathrm{Share}(s_0)} \equiv \{(s_i)_{i \in I}\}_{\mathrm{Share}(s_1)}.$$

即少于 $t$ 个份额的分布与秘密值无关。信息论安全——不依赖计算困难性假设。

### 2.2 Shamir Secret Sharing (SSS)

**Share**：选 $t-1$ 次多项式 $f(x) = s + a_1 x + a_2 x^2 + \dots + a_{t-1} x^{t-1}$，$a_i \leftarrow \mathbb{F}_q$（$q$ 素数或素数幂，$q > n$）。
份额 $s_i = f(i)$，$i \in \{1, 2, \dots, n\}$。

**Reconstruct**：用 Lagrange 插值
$$s = f(0) = \sum_{i \in I} s_i \prod_{j \in I, j \ne i} \frac{-j}{i - j}.$$

**正确性**：$t$ 个点唯一决定 $t-1$ 次多项式。
**安全性**：少于 $t$ 个点时，$f(0)$ 在 $\mathbb{F}_q$ 上均匀分布（可证明）。

### 2.3 Additive Secret Sharing

**Share**（$t = n$）：选 $s_1, \dots, s_{n-1} \leftarrow \mathbb{Z}_q$ 均匀，$s_n = s - \sum_{i<n} s_i$。
**Reconstruct**：$s = \sum_i s_i \bmod q$。

**同态**：
- $\mathrm{Add}$：份额逐项相加。
- **Scalar mul**：$k \cdot s_i$。
- 乘法 $\langle s \rangle \cdot \langle t \rangle$ 需要 **Beaver 三元组** 辅助（预计算随机三元组 $\langle a \rangle, \langle b \rangle, \langle c \rangle$ 满足 $ab = c$）。

**限制**：$t < n$ 时不安全（缺少任意一份就解不出）。但 replicated secret sharing 可支持 $t = \lfloor (n-1)/2 \rfloor$ (例如 3-party, 2-of-3)。

### 2.4 XOR Secret Sharing

GF(2) 上 additive 版：$s_1, \dots, s_{n-1} \leftarrow \{0,1\}^\ell$，$s_n = s \oplus s_1 \oplus \dots \oplus s_{n-1}$。
极快，用于 GMW 协议 bit-level 电路。

### 2.5 Verifiable Secret Sharing (VSS)

普通 SSS 假设 dealer 诚实。**VSS** 增加 dealer 承诺：
- **Feldman VSS**：dealer 发布 $g^{a_i}$ 系数承诺；接收者可验证 $g^{s_i} = \prod g^{a_j \cdot i^j}$。代价：份额不再信息论隐藏（ElGamal 隐藏，DDH 假设）。
- **Pedersen VSS**：用 Pedersen 承诺 $g^{a_i} h^{b_i}$，同时满足 computational binding 与 perfect hiding。

VSS 是所有异步 MPC、阈值签名 DKG 的前提。

### 2.6 关键参数

| 方案 | 域 | 门限 | 份额大小 | 运算速度 | 同态 |
| --- | --- | --- | --- | --- | --- |
| Shamir | $\mathbb{F}_q$ | $t$ | $\log_2 q$ | 插值 O(t²) | + 与 × (with beaver) |
| Additive | $\mathbb{Z}_q$ | $n$ | $\log_2 q$ | O(n) | + 直接，× 需 beaver |
| XOR | $\{0,1\}^\ell$ | $n$ | $\ell$ | O(n) XOR | XOR only |
| Replicated (2-of-3) | $\mathbb{Z}_q$ | 2 | $2\log_2 q$ | 快 | + 与 × |

### 2.7 失败模式

- **随机数偏置**：dealer PRNG 可预测 → 未分享点不再均匀 → 可恢复 $s$。
- **重复份额点**：两方拿到同一 $x$，等于少一份。
- **dealer 作恶**：不给等式成立的份额，需 VSS 防御。
- **旁路**（side-channel）：通过份额大小/时间泄漏。
- **插值攻击**：量子下 Lagrange 仍安全，但份额若用某些哈希承诺则可能被预计算撞库。

```mermaid
flowchart LR
  s((Secret s)) --> D[Dealer chooses f]
  D --> p1[f(1)]
  D --> p2[f(2)]
  D --> p3[f(3)]
  D --> pn[f(n)]
  p1 & p2 & p3 --> R["Lagrange Interp."]
  R --> sr((s))
```

```
Polynomial visualization (t=3, n=5)
     y
     |
     |  •         •
     |    \     /    •
     |      •            <- f(0)=s
     |               •
     +-----------------> x
        1  2  3  4  5
```

## 3. 架构剖析

### 3.1 分层视图

1. **Field arithmetic**：$\mathbb{F}_p$ (p 素数) 或 GF(2^k) 运算。
2. **Sharing engine**：Shamir / Additive / XOR / Replicated。
3. **VSS layer**：dealer commitment & share verification。
4. **Protocol layer**：DKG、PVSS、Proactive resharing。
5. **Application**：TSS、MPC、KMS 备份。

### 3.2 核心模块清单

| 模块 | 职责 | 依赖 | 代表路径 |
| --- | --- | --- | --- |
| Field Ops | ModExp、Invert、Rand | BigInt | `dalek-cryptography/curve25519-dalek/src/scalar.rs` |
| Shamir | Share/Reconstruct | Field | `coinbase/kryptology/pkg/sharing/shamir.go` |
| Feldman VSS | Commit + Verify | Group | `coinbase/kryptology/pkg/sharing/feldman.go` |
| Pedersen VSS | Dual-commit | Group | `ZenGo-X/multi-party-ecdsa` |
| DKG | Joint-Feldman | VSS + BB | `FROST/dkg.rs` |
| Proactive | 周期 refresh | Shamir | HashiCorp Vault MPC plugin |

### 3.3 数据流：Trezor Shamir Backup (SLIP-0039)

1. Trezor 生成 128/256-bit 熵 $s$。
2. 设置 $(t, n)$ 与 group parameters。
3. Level-1 SSS：把 $s$ 拆成 $G$ 个 group secret。
4. Level-2 SSS：每个 group secret 再拆成 $n_g$ 份给成员。
5. 份额编码为 33 个英文助记词 + checksum（RS code）。
6. 恢复：先 group-level threshold 重构，再 master level 重构 → $s$ → HD seed。

### 3.4 参考实现

- **Coinbase Kryptology** Go：Shamir、Feldman、FROST。
- **dalek-cryptography/vsss-rs** Rust：Shamir、Feldman、Pedersen。
- **Gennaro-Goldfeder multi-party-ecdsa** Rust：DKG + resharing。
- **SLIP-0039** Trezor Python 参考实现。

### 3.5 扩展接口

- Shamir Backup CLI：`shamir-mnemonic`（PyPI）。
- Amazon KMS AllowedGrantees（5-of-9 root key）。
- Safe{Wallet} Guard：签名份额分发到 owners。

## 4. 关键代码 / 实现细节

Shamir share + Lagrange 重构（基于 `coinbase/kryptology` v1.10）：

```go
// pkg/sharing/shamir.go (简化)
func (s *Shamir) Split(secret Element) ([]*Share, error) {
    coeffs := make([]Element, s.threshold)
    coeffs[0] = secret
    for i := 1; i < s.threshold; i++ {
        coeffs[i] = s.curve.Scalar.Random(rand.Reader)
    }
    shares := make([]*Share, s.limit)
    for i := 1; i <= s.limit; i++ {
        x := s.curve.Scalar.New(i)
        y := coeffs[s.threshold-1]
        for j := s.threshold - 2; j >= 0; j-- {
            y = y.Mul(x).Add(coeffs[j]) // Horner
        }
        shares[i-1] = &Share{Id: i, Value: y.Bytes()}
    }
    return shares, nil
}

func Combine(shares []*Share, curve *Curve) (Element, error) {
    result := curve.Scalar.Zero()
    for _, si := range shares {
        num := curve.Scalar.One()
        den := curve.Scalar.One()
        xi := curve.Scalar.New(si.Id)
        for _, sj := range shares {
            if si.Id == sj.Id { continue }
            xj := curve.Scalar.New(sj.Id)
            num = num.Mul(xj.Neg())
            den = den.Mul(xi.Sub(xj))
        }
        coef := num.Mul(den.Invert())
        result = result.Add(coef.Mul(si.ValueScalar()))
    }
    return result, nil
}
```

## 5. 演进与版本对比

| 里程碑 | 年份 | 贡献 |
| --- | --- | --- |
| Shamir / Blakley | 1979 | 首个 SSS |
| Feldman | 1987 | VSS |
| Pedersen | 1991 | Perfect-hiding VSS |
| Proactive SS | 1995 | 周期 refresh |
| SLIP-0039 | 2018 | 助记词级 SSS |
| FROST DKG | 2020 | Schnorr 阈值 |
| BLS + DKG | 2021 | BLS 阈值签名 |
| Pluto DKG | 2023 | RLWE 阈值（FHE KMS） |

## 6. 实战示例

```bash
# slip-0039 参考工具
pip install shamir-mnemonic
shamir create --group-threshold 2 --groups 2-of-3 2-of-3
# 输出两组 SLIP-0039 助记词，任意 2 组中的 2 份可恢复
shamir recover
```

## 7. 安全与已知攻击

- **Fireblocks GG20 2022 CVE**：Feldman VSS 零系数未检查，可导出子密钥。
- **Random subgroup attack**：若曲线阶含小因子，份额落入子群被反解。
- **dealer biased coin**：2020 Ledger Recover 早期用 non-uniform PRNG，理论上可剪枝搜索。
- **Trezor Shamir Backup 侧信道 2021**：通过片内 Flash 读 timing 泄露。

## 8. 与同类方案对比

| 维度 | Shamir | Additive | Replicated | Ramp SS |
| --- | --- | --- | --- | --- |
| 信息论安全 | Yes | Yes (t=n) | Yes | 非 perfect |
| 计算复杂度 | O(t²) 插值 | O(n) | O(1) 选择 | 低 |
| 同态 | + (× 需 beaver) | + (× beaver) | + 与 × | + |
| 扩展 (n 增加) | 增加 share | 重分 | 必须重组 | 容易 |

## 9. 延伸阅读

- Shamir A., "How to Share a Secret"，CACM 1979
- Feldman P., "A Practical Scheme for Non-interactive Verifiable Secret Sharing"，FOCS 1987
- Pedersen T., "Non-Interactive and Information-Theoretic Secure Verifiable Secret Sharing"，CRYPTO 1991
- SLIP-0039 spec：https://github.com/satoshilabs/slips/blob/master/slip-0039.md

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| SSS | Shamir Secret Sharing | 多项式插值式 SS |
| VSS | Verifiable SS | 可验证 SS |
| DKG | Distributed Key Generation | 分布式密钥生成 |
| Proactive | Proactive Sharing | 周期刷新份额 |
| Beaver | Beaver Triple | 预计算乘法辅助 |

---

*Last verified: 2026-04-22*
