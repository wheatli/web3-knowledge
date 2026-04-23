---
title: Wintermute Profanity 虚荣地址私钥破解事件（2022-09-20, ~$160M）
module: 08-security-incidents
category: Key-Mgmt
date: 2022-09-20
loss_usd: 160000000
chain: [Ethereum]
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://twitter.com/EvgenyGaevoy/status/1572302146266296320
  - https://www.certik.com/resources/blog/1Ly9hT9v9wbrAfZOxvYtF-wintermute-hack-incident-analysis
  - https://rekt.news/wintermute-rekt/
  - https://www.1inch.io/blog/a-vulnerability-disclosed-in-profanity-an-ethereum-vanity-address-tool/
  - https://twitter.com/libevm/status/1572252418606272515
tx_hashes:
  - 0xb1a015c6a7ea2c96c1e2ae77ed78608c06d63017bdb26b27d4d820f75e79d31c（示例攻击 tx，实际为多笔）
  - Wintermute Profanity 地址: 0x0000000fe6a514a32abdcdfcc076c15d4d3b1a69
  - 攻击者地址: 0xe74b28c2eAe8679e3cCc3a94d5d0dE83CCB84705
---

# Wintermute Profanity 虚荣地址私钥破解事件

> **TL;DR**：2022-09-20，做市商 Wintermute 的 DeFi 操作钱包被盗约 **$160M**。该钱包是以 **Profanity** 工具生成的虚荣地址（vanity address）——以 7 个前导 0 开头（`0x0000000f...`）以节省 gas（EIP-2929 SLOAD 退款相关 gas 优化）。Profanity 使用 **32-bit 随机种子 + 确定性哈希扩展** 生成私钥，空间可穷举。1inch 团队于 2022-09-15 公开披露该漏洞（1inch Profanity Vulnerability Disclosure），攻击者 5 天内逆向计算 Wintermute 该地址的私钥并清空资金。**Wintermute DeFi 热钱包，冷钱包与 CEX 做市部分不受影响**；CEO Evgeny Gaevoy 公开确认事件并提出白帽协商（10% 赏金未被接受）。

## 1. 事件背景

### 1.1 Wintermute

Wintermute Trading 是 2017 年伦敦成立的顶级加密做市商，涵盖 CEX OTC、CEX 做市与 DeFi 做市三大业务；日成交额百亿美元级。其 DeFi 做市钱包使用虚荣地址以节省批量 approve / transfer 的 gas。

### 1.2 Profanity 工具

Profanity（`johguse/profanity`）是一款以 GPU 暴力穷举 "漂亮前缀" 的以太坊地址生成工具，2017 年发布。核心逻辑：
- 随机 seed（32 bit 或 64 bit，具体构造存在瑕疵）；
- 通过 mersenne-twister 产生 ecdsa 私钥；
- 把 secp256k1 公钥 keccak256 取后 20 字节，看是否匹配目标前缀。

**致命缺陷**：使用的随机种子熵太低（32-bit 有效空间 + 可预测 expansion），攻击者用类似 GPU 能力即可**反向穷举** Profanity 生成的私钥——只要知道公钥（即地址），便能在数小时内找到私钥。

### 1.3 时间轴

- **2022-09-15**：1inch 安全团队发布 "A vulnerability disclosed in Profanity, an Ethereum vanity address tool" 博客。呼吁所有使用 Profanity 生成地址的用户立即转移资金。
- **2022-09-16–19**：部分机构（Wintermute 公司收到通知并开始迁移）；但 Wintermute DeFi 主钱包因整合大量授权 / 智能合约依赖，未完成迁移。
- **2022-09-20 UTC 07:00 左右**：攻击者利用算力逆向计算出 `0x0000000fe6a514a32abdcdfcc076c15d4d3b1a69` 的私钥，签发多笔大额转账与 `swap/transfer`，资产涌向攻击者地址 `0xe74b28c2...84705`。
- **UTC 07:43**：Wintermute CEO Evgeny Gaevoy 发推确认 DeFi 钱包被盗 ~$160M，CEX 与 OTC 业务不受影响。
- **2022-09-20 晚**：1inch、CertiK、rekt.news 发布事件分析。
- **2022-10**：Wintermute 消化损失继续运营，未请求外部救助。

### 1.4 发现过程

链上监控 @libevm、@officer_cia、PeckShield 实时标注 `0x0000000f...` 钱包异常外转；Wintermute 内部监控同步告警。

## 2. 事件影响

### 2.1 直接损失

- 约 **$160M**（USDC、USDT、WBTC、ETH、stETH、DAI、FRAX、MIM、USDD、TUSD 等），具体分布：
  - USDC ≈ $61M
  - USDT ≈ $29M
  - WETH ≈ $12M
  - WBTC ≈ $6M
  - 其他稳定币与中小市值代币若干；
- 仅 DeFi 热钱包，CEX 做市部分与冷储备未受损。

### 2.2 受害方

- Wintermute 自身（其自有资金，非客户资金），因此没有 retail 用户直接受损；
- 但 Wintermute 作为 Dune / Socket / Multichain / 多家 DeFi 协议 LP 的对手方，短期市场深度受影响。

### 2.3 连带影响

- 进一步暴露 **Profanity 受影响地址** 的系统性风险；1inch 披露后估计有 $300M+ 资金分布在 Profanity 生成钱包中；
- 随后数天数家小项目也因 Profanity 地址被盗，合计 $3M+；
- 业界重新审视 "gas 优化 vanity address" 的真实收益 vs 成本。

### 2.4 资金去向

- 攻击者地址 `0xe74b28c2eAe8679e3cCc3a94d5d0dE83CCB84705` 把稳定币兑换为 ETH / DAI，部分通过 Curve 3pool 兜转；
- 部分资金存入 Aave V2 产生利息——引发 Wintermute 向 Aave 提起 DAO 治理提案请求强制清算，未获通过；
- 随后多路径进入 Tornado Cash。

**归因**：无 Lazarus 归因证据；Profanity 漏洞被公开披露，任何具备算力的个人/团队都可复现。**归因未公开**。

## 3. 技术根因（核心）

### 3.1 漏洞分类

**密钥生成弱熵**（weak-entropy private key generation），属于离链漏洞 + 钱包基础设施缺陷。

### 3.2 受损模块

- 工具：`johguse/profanity`（仓库在 2022-09 之后被作者 archive）
- 关键代码位置：`Dispatcher.cpp`、`profanity.cl`（OpenCL kernel）中的随机种子生成

### 3.3 关键逻辑（伪代码）

```c
// Profanity Dispatcher.cpp 简化
uint32_t seed = time(NULL) ^ getpid();        // 32-bit seed
std::mt19937 rng(seed);                       // mersenne twister
uint8_t privkey[32];
for (int i = 0; i < 8; i++) {
    uint32_t w = rng();                       // 只有 2^32 个可能值
    memcpy(privkey + i*4, &w, 4);             // 派生 32 字节 privkey
}
// 之后 GPU kernel 以 privkey + add/increment 扩展 2^32 ~ 2^40 个候选，
// 对每个候选做 secp256k1 公钥推导 + keccak -> 地址，匹配目标前缀。
```

**错在哪**：
- 初始熵只有 **32-bit**（`seed`）；
- 即使 GPU kernel 内部 +1 扩展上千亿次，**映射空间仍以 32-bit seed 为基准**；给定地址反向穷举需要扫遍 2^32 初始种子 × 每种子派生数——以 2022 年 GPU 算力，单 V100 约 **~几小时到一天** 可找到匹配；
- 相当于私钥实际熵只有 32-40 bit，远低于安全最小值 128-bit。

### 3.4 攻击步骤分解

1. **预读 1inch 披露**：2022-09-15 公开披露把漏洞坐实；
2. **枚举目标**：链上扫描所有前缀 >= 6 个 0 的高余额 EOA，选出 Wintermute 的 `0x0000000fe6a514a32abdcdfcc076c15d4d3b1a69`（价值 $160M 级）；
3. **GPU 穷举**：以 Profanity 的已知随机算法 + 地址前缀，在 32-bit 种子空间内 brute-force，找到生成该公钥的 seed，推导出私钥；
4. **签发批量转账**：2022-09-20 UTC 07 左右，用私钥签发 transfer 到 `0xe74b28c2...`；部分代币因 approve 关系直接调用 target 协议的 remove/withdraw；
5. **逃逸**：几小时内把稳定币做互换 + 部分存入 Aave；
6. **混币**：Tornado Cash。

### 3.5 为何未被发现

- Wintermute 使用 Profanity 多年未复审其熵来源；
- 虚荣地址的"gas 优化"收益（前导 0 字节节省 SLOAD gas）诱使机构忽视密钥安全；
- 1inch 2022-09-15 披露后，Wintermute 有 5 天窗口但未完成所有协议授权迁移，部分原因是钱包绑定上百条 approve 和智能合约调用者白名单。

## 4. 事后响应

### 4.1 项目方动作

- Wintermute 发推确认、继续运营；
- 放弃被盗钱包，部署新 EOA / Gnosis Safe 作为 DeFi 操作账户；
- 对攻击者发公开协商：保留 10% 赏金换回资金，未果。

### 4.2 资产追回

链上追回几乎为 0；被 Circle / Tether 冻结的稳定币若干百万；大部分资金通过 Tornado Cash 洗净。

### 4.3 法律 / 执法

- 无公开执法行动；
- Chainalysis 未发布归因；
- Wintermute 在英国提起民事诉讼，法院允许以 NFT 形式向攻击者地址 "送达"（pioneering case）。

### 4.4 复查与行业反应

- Profanity 仓库被作者 archive（`johguse/profanity`），README 警告立即停用；
- 1inch 发布工具扫描 Profanity 生成地址并通知拥有者；
- 钱包最佳实践重申：私钥必须使用 **CSPRNG**（如 `getrandom`、`/dev/urandom`、BIP39 高熵助记词）生成，至少 128-bit 熵。

## 5. 启发与教训

### 5.1 对开发者

- **禁止弱熵**：任何密钥生成必须使用 CSPRNG，种子 ≥ 128-bit，最好 256-bit；
- 虚荣地址（vanity）可用 CSPRNG 暴力匹配前缀生成（时间长但私钥安全），**严禁使用 Profanity 类工具**；
- 智能合约层面节省 gas 不值得为此牺牲密钥强度。

### 5.2 对审计方

- Operational security audit 应覆盖密钥生成工具链；
- 签名机 / HSM / MPC 集成时，密钥来源需追溯。

### 5.3 对用户

- 检查钱包地址是否使用 vanity 工具生成；如是 Profanity（约 2022-09 前广泛使用），立即弃用；
- 大额资金避免使用 custom vanity 地址。

### 5.4 对协议

- 前导零地址节省 gas 的优化历史上吸引机构，但与 Profanity 的弱熵结合后成为集中攻击面；
- 业界开始讨论把 "节约 gas 的压缩地址" 挪到 L2 / create2 确定性 deploy 等更安全路径。

## 6. 参考资料

- Wintermute CEO Twitter 确认：https://twitter.com/EvgenyGaevoy/status/1572302146266296320
- 1inch Profanity 漏洞披露：https://www.1inch.io/blog/a-vulnerability-disclosed-in-profanity-an-ethereum-vanity-address-tool/
- rekt.news：https://rekt.news/wintermute-rekt/
- CertiK 分析：https://www.certik.com/resources/blog/1Ly9hT9v9wbrAfZOxvYtF-wintermute-hack-incident-analysis
- @libevm 实时 thread：https://twitter.com/libevm/status/1572252418606272515
- Profanity 仓库（archived）：https://github.com/johguse/profanity
- Wintermute Profanity 地址：`0x0000000fe6a514a32abdcdfcc076c15d4d3b1a69`
- 攻击者地址：`0xe74b28c2eAe8679e3cCc3a94d5d0dE83CCB84705`

---

*Last verified: 2026-04-23*
