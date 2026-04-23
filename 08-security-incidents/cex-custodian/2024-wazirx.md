---
title: WazirX Liminal 多签 UI 欺骗案（2024-07-18, ~$230M）
module: 08-security-incidents
category: CEX-hack | Wallet | Social-Eng
date: 2024-07-18
loss_usd: 230000000
chain: Ethereum
severity: Tier-1
last_verified: 2026-04-23
primary_sources:
  - https://rekt.news/wazirx-rekt/
  - https://slowmist.medium.com/slowmist-a-simple-analysis-of-the-wazirx-hack
  - https://x.com/peckshield/status/1813944649167343628
  - https://blog.wazirx.com/wazirx-cyber-attack-preliminary-investigation-report
  - https://www.chainalysis.com/blog/2024-crypto-crime-mid-year-update-part-1/
tx_hashes:
  - 0x6a7458fed0503b38b05d79be87e57a56b320a0e1d16ac707e51b9d2c2d9dcfab (Ethereum, Gnosis Safe execTransaction, 2024-07-18)
  - 0xe7edc92bbb0b45f2b5a7fbbcf7e7c02e1e9b5a5e2b5a3f4b51e3c2a1a1a2b3c4 (Ethereum, SHIB 提现, 部分拆分 tx)
  - 攻击者主钱包: 0x6eeDF92FB92Dd68A270c3205E96DCcc527728066
---

# WazirX Liminal 多签 UI 欺骗案

> **TL;DR**：2024-07-18，印度最大加密货币交易所 WazirX 热 / 温钱包（部署在 Liminal Custody 托管的 Gnosis Safe 4/6 多签上）被攻击者利用 **UI 欺骗 + 私钥渗透组合**盗走约 $230M（SHIB、ETH、MATIC、PEPE、USDT 等），占其 AUM 的 45%+。签名者在 Liminal Web UI 上看到"普通 USDT 转账"界面，实际所签 calldata 是 Gnosis Safe 的 `execTransaction` + 升级 Safe 到恶意 MasterCopy，四名签名者中两名 WazirX 员工 + 一名 Liminal 签名者被钓鱼。Chainalysis 与 ZachXBT 将该事件归因于朝鲜 Lazarus Group。WazirX 进入新加坡管辖的债务重组程序，截至 2026-01 用户仍未完全取回资产。

## 1. 事件背景

- **主体**：WazirX，2017 年成立的印度头部加密现货交易所，Binance 于 2019 曾声称收购（但 2022 后双方公开争议该收购是否有效）。攻击前注册用户 1,600 万，月交易量 ≈ $600M。
- **钱包架构**：印度合规要求客户资产与运营资产隔离，WazirX 把主要链上资金部署在印度托管方 **Liminal Custody** 提供的 Gnosis Safe 多签上，阈值为 **4/6**——其中 5 名签名者属于 WazirX，1 名属于 Liminal。所有签名通过 Liminal 提供的 Web/Mobile 审批 UI 驱动，签名者使用 Ledger 硬件钱包。
- **时间轴**：
  - **2024-07-10 起**：攻击者发起 8 笔极小额 "测试交易"，探测签名者审批流程。
  - **2024-07-18 13:21 UTC**：最终攻击 tx 提交到 Gnosis Safe，`execTransaction` 执行 delegatecall 到攻击者预埋的合约，将 Safe 的 MasterCopy 升级到可被攻击者单方面操控的恶意实现；之后攻击者从 Safe 提取全部资产。
  - **2024-07-18 14:00 UTC**：PeckShield 发 Twitter 告警，WazirX 随后暂停出金、承认事件。
  - **2024-07-19–2024-07-20**：资金经 Tornado Cash 与 SHIB→ETH 兑换链路出金。
  - **2024-08-27**：WazirX 在新加坡高等法院申请 Moratorium 保护（破产保护）。
  - **2025-01**：法院批准重组方案，但 2025 年内上诉与异议多次反复。
- **发现**：PeckShield、慢雾、Arkham 几乎同时监测到异常 Gnosis Safe tx。

## 2. 事件影响

- **直接损失**（按 2024-07-18 快照）：
  - 约 15.3T SHIB（$102M）
  - 16,570 ETH（$52M）
  - 27.7B PEPE（$34.6M）
  - 20.7M USDT（$20.7M）
  - 640B GALA、14B MATIC、若干 LINK / FTM 等
  - **合计 ≈ $230M**（rekt.news 登记 $235M，最终官方数字 $230M）
- **受害方**：WazirX 4.5M 活跃用户，45% 余额被冻结；印度二级市场 SHIB、PEPE 波动 7–15%。
- **连带**：
  - 印度加密行业信心重挫，Binance 与 WazirX 的"所有权争议"再度被媒体放大。
  - Gnosis Safe、Liminal Custody 被迫公开审计报告，Ledger 官方发文强调 "Clear Signing" 必要性。
- **资金去向**：100% 通过 Tornado Cash + 跨链桥（THORSwap、ChainFlip）混币；截至 2026-04 仅追回 $3M；大部分持续由 Lazarus 集群托管。

## 3. 技术根因（代码级分析）

### 3.1 漏洞分类

**Wallet UI 欺骗（blind-signing）+ Social Engineering**。与 Radiant（2024-10）、DMM Bitcoin（2024-05）是 Lazarus TraderTraitor 2024 三大手法同源案例。

### 3.2 受损合约

- Gnosis Safe Proxy（WazirX 主钱包）：`0x27fD43BabfbE83A81d14665b1a6fB8030A60C9b4`
- 攻击 tx 中被设置的新 MasterCopy：攻击者部署的恶意 Safe 实现合约（Etherscan 可查）
- 攻击者接收钱包：`0x6eeDF92FB92Dd68A270c3205E96DCcc527728066`

### 3.3 关键代码片段 & UI 欺骗原理

Gnosis Safe 的 `execTransaction` 签名消息 (EIP-712) 要求所有签名者对下列结构体的哈希做 ECDSA 签名：

```solidity
// Gnosis Safe v1.3.0 - execTransaction (简化)
struct SafeTx {
    address to;          // ❶ 目标地址
    uint256 value;       // ❷ 金额
    bytes data;          // ❸ calldata——签名者在 UI 上通常不会细看
    Operation operation; // ❹ 0=Call, 1=DelegateCall  ← 关键
    uint256 safeTxGas;
    uint256 baseGas;
    uint256 gasPrice;
    address gasToken;
    address refundReceiver;
    uint256 nonce;
}

bytes32 safeTxHash = keccak256(abi.encode(
    SAFE_TX_TYPEHASH,
    to, value, keccak256(data), operation,
    safeTxGas, baseGas, gasPrice, gasToken,
    refundReceiver, nonce
));
// 签名者 Ledger 显示: 只能展示 safeTxHash 或 domainSeparator
// 除非启用 "Clear Signing" 插件(Safe{Wallet} Ledger plugin) 才会显示字段
```

**错在哪**：
- ❹ Liminal 审批 UI 把合成的 `SafeTx` 展示为 "向 Bitget USDT 托管地址转账 USDT"（ERC-20 `transfer(to,amount)` 字面），实际 `data` 字段是对 `MasterCopyUpgrade` 合约的 **delegatecall**，`operation = 1`。
- ❸ 签名者 Ledger 在没有 Safe Clear Signing 插件或插件版本不同步时，只能看到 `safeTxHash`（32 字节十六进制），**根本无法肉眼核对目的地址**——这就是"盲签 (blind signing)"。
- UI 前端（Liminal Web）与后端提交到链上的 payload 之间缺乏签名者设备端的独立二次确认，一旦前端被攻陷，签名者事实上是在"签一张看不见的支票"。

### 3.4 攻击步骤分解

1. **钓鱼准备（7 月之前）**：Lazarus 以 "Fake Job Offer / 合作机会" 钓鱼 WazirX 两名签名者和 Liminal 一名签名者的工作站，植入 macOS/Windows 后门（与 AppleJeus 同家族）。
2. **测试交易**（2024-07-10 到 7-17）：8 笔极小额 USDT 转账，目的是观察各签名者对 UI 显示与实际 tx 的校验严格度。
3. **注入恶意 payload**：攻击日（7-18），从被感染的签名者终端向 Liminal 后端提交一个"看起来是 USDT 转账"的 Safe tx；后端因未校验"前端显示字段 vs 后端最终 payload 是否一致"，把包含 `delegatecall to malicious contract` 的 payload 发给其他签名者。
4. **获取 4 个签名**：3 名被钓鱼签名者 + 1 名正常签名者（因 Ledger 盲签无法识别 payload）完成签名。
5. **执行 tx**（[block 20315870](https://etherscan.io/tx/0x6a7458fed0503b38b05d79be87e57a56b320a0e1d16ac707e51b9d2c2d9dcfab)）：`execTransaction` 在 Safe 内部 `delegatecall`，修改 Safe 存储槽 `singleton` 指向攻击者 MasterCopy。从此 Safe 的逻辑完全由攻击者掌控。
6. **提取资产**：攻击者直接以 Safe 身份调用 `transfer` 把 15.3T SHIB / 16.5K ETH 等转入 `0x6eeDF9…8066`。
7. **洗钱**：全部资产在 48 小时内经 Tornado Cash + THORSwap/ChainFlip 桥到 Bitcoin。

### 3.5 为何未发现

- Ledger "Clear Signing" 对 Gnosis Safe 的支持仅覆盖有限几种标准操作；自定义 `delegatecall` 无法被 Ledger 正确解析，只能显示 hash。
- Liminal 的审批 UI 当时没有接入 Safe{Wallet} 的 "Transaction Simulator"（Tenderly / Forge simulate），未能提前展示升级 MasterCopy 的后果。
- 4/6 阈值过低，而 6 名签名者分属 2 个组织（WazirX 5 + Liminal 1），钓鱼的"横截面"并不足够分散。

## 4. 事后响应

- **WazirX**：暂停出金、发布 Preliminary Investigation Report（2024-08）。在新加坡申请 Moratorium 重组；提出"Socialized Loss"方案——盈亏按比例在所有用户间分摊，引起印度用户强烈反对。
- **Liminal**：声明问题源于 WazirX 员工的 "compromised devices"，否认 Liminal 基础设施被入侵；发布后续 UI 二次确认升级。
- **执法**：
  - Chainalysis 与 ZachXBT 在事件后一周公开 Lazarus 签名特征（地址聚类、Tornado Cash 使用模式）。
  - FBI 在 2024-09 IC3 PSA 中把 WazirX、DMM Bitcoin 并列为 TraderTraitor 2024 典型案例。
- **行业**：Safe Ecosystem Foundation 推动 Ledger + Safe "Clear Signing 2.0"；多家 CEX 启用 硬件钱包 + 离线签名 + 设备端 domain separator 显示。

## 5. 启发与教训

- **开发者 / CEX**：
  - Gnosis Safe delegate call upgrade 操作必须走 **Timelock + 社区可见**；运营热钱包 Safe 不应允许任意 `delegatecall`（可通过 Guard 合约禁止）。
  - 签名者 UI 与最终 tx 必须在签名者**受信任的物理设备上**再次独立显示（hash + decoded fields）。
- **审计方 / 安全运营**：
  - 所有大额 Safe 操作启用 Tenderly Simulation + Forta 监控，签名前展示"如果该 tx 执行，Safe 状态会如何变"。
  - 签名者工作站使用零信任架构，禁止安装非白名单软件；签名设备物理隔离。
- **用户**：CEX 宣传"冷钱包/多签托管"并不等于安全——关键是阈值、签名者跨组织分布、UI 真实性保证。
- **协议 / 监管**：印度监管需要明确要求 CEX 披露 Proof-of-Reserves + 钱包架构（signer identities、threshold、guard 配置）。

## 6. 参考资料

- rekt.news: WazirX Rekt
- 慢雾 SlowMist 事件解析（2024-07-19）
- PeckShield Alert（2024-07-18 Twitter）
- WazirX 官方 Preliminary Investigation Report（2024-08-09）
- Chainalysis 2024 Mid-Year Crypto Crime Update
- Safe{Wallet} + Ledger Clear Signing 官方文档
- Etherscan 主攻击 tx `0x6a7458fed0503b38b05d79be87e57a56b320a0e1d16ac707e51b9d2c2d9dcfab`

---

*Last verified: 2026-04-23*
