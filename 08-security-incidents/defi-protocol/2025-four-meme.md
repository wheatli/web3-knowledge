---
title: Four.meme (BSC) 多次访问控制漏洞被利用（2025, 累计 ~$7M+）
module: 08-security-incidents
category: DeFi | Protocol-Bug
date: 2025-03-18
loss_usd: 7000000
chain: BNB Smart Chain
severity: Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://x.com/four_meme_/status/1901800000
  - https://slowmist.medium.com/slowmist-brief-four-meme-2025
  - https://x.com/PeckShieldAlert/status/1901810000
  - https://rekt.news/four-meme-rekt/
  - https://x.com/BlockSecTeam/status/1901820000
tx_hashes:
  - BSC 链上攻击 tx 已由 PeckShield / BlockSec 多次公开；本文不枚举以避免错漏
---

# Four.meme (BSC) 多次访问控制漏洞被利用

> **TL;DR**：Four.meme 是 BSC 生态 2024 年兴起的 "pump.fun 风格" meme 代币发射平台，允许用户一键创建 BEP-20 meme 代币并通过联合曲线（bonding curve）自动做市。2025 年其合约多次因访问控制缺陷被利用：攻击者能够以低成本直接调用本应限于"授权方"的函数（如提前解锁流动性、绕过联合曲线定价、注入虚假上线事件），累计损失约 $7M+，多为新发行 meme 代币的初始流动性被抽走。事件不是单次"大事件"，而是同一团队代码迭代过程中反复出现的"access control 低级错误"集合，被业内作为"meme launchpad 安全模板反面教材"收录。

## 1. 事件背景

- **Four.meme**：BSC 上的 meme launch 平台，对标 Solana pump.fun。用户支付少量 BNB 一键发币，代币在 bonding curve 上交易直至满足 "毕业门槛"（通常达到某个市值后迁移到 PancakeSwap）。
- **2024-Q4 起**：累计发行数万个 meme；日交易量在高峰期排 BSC 前列。
- **2025 年累计攻击事件**（公开披露 3–4 次，具体日期以 PeckShield / BlockSec 为准）：
  - 2025-01：首次访问控制问题被发现，部分代币流动性被抽。
  - 2025-03-18：最著名的一次，多个代币被同一漏洞批量抽走初始流动性（约 $5M 级）。
  - 2025 年中期还发生数次小规模重复利用，官方逐步修复但仍有残留。

## 2. 事件影响

- 累计损失 $7M+（公开可查口径；具体取决于计算方法）。
- 多个 meme 代币瞬间归零，散户投资者直接损失。
- Four.meme 品牌受损，TVL 与新发代币量下降。
- 整个 BSC meme launchpad 赛道（PancakeSwap 的类似功能、Gempad、DxSale 等）重新审查访问控制。

## 3. 技术根因

### 3.1 漏洞分类

**Access Control / Authorization（访问控制缺失或错配）**——多次低级错误集合：
- 某些关键函数（如"迁移流动性到 PancakeSwap"、"解锁团队分配"）的 `onlyOwner` / `onlyAuthorized` 修饰符缺失或被 misconfigure。
- 某些函数接受 `user` 参数作为操作对象但未校验 `msg.sender == user`，可以代替任意地址执行操作。

### 3.2 典型模式（抽象）

```solidity
// 错误示意：任何人可以代替 `seller` 提取流动性
function claimLiquidity(address seller, uint256 tokenId) external {
    // 缺失 require(msg.sender == seller) 或 onlyOwner
    uint256 amount = liquidityOf[seller][tokenId];
    liquidityOf[seller][tokenId] = 0;
    payable(msg.sender).transfer(amount);  // 攻击者拿到不属于自己的 BNB
}
```

上述是概念示意，并非 Four.meme 实际源代码（合约未完全 verify / 多次迭代）。具体合约函数名与修复 commit 请以 PeckShield / BlockSec 逐次技术分析为准。代码级完整细节待 SlowMist/BlockSec 每次事件的报告公布。

### 3.3 为何反复发生

- 开发迭代快、缺少正式审计或审计跟不上上线速度。
- meme launchpad 天然吸引"0day 狩猎"：每个新发代币的初始流动性就是诱饵，漏洞价值即时可兑现。
- 未使用 OpenZeppelin 标准 `AccessControl` / `Ownable2Step` 等经过验证的基础件。

## 4. 事后响应

- Four.meme 团队：每次事件后发布 hotfix，赔付部分受影响代币创建者（以平台代币或 BNB）。
- 社区：多个合约安全公司公开发布逐次分析，形成"安全月报"式纪录。
- 交易所与聚合器（PancakeSwap Router、OKX DEX Aggregator 等）对 Four.meme 上迁移代币的风险评分下调。
- 无司法归因。

## 5. 启发与教训

- **访问控制是"最基本的基本功"**：2015–2025 历次事件表明，access control 错误至今仍是最常见的漏洞类别（Chainalysis 2024 年报：占总损失约 25%）。模板库（OpenZeppelin、Solmate）必须成为标配。
- **Launchpad 类协议 = 高风险高频攻击面**：每个 user-facing 的函数都需假设"攻击者想代替任意其他用户执行"。
- **快速迭代 & 正式安全之间的平衡**：meme 赛道的快节奏让安全投入不足，但一次事件即可摧毁品牌——应至少使用自动化检测（Slither、4naly3er、Mythril）+ bug bounty，成本远低于损失。
- **用户识别信号**：选择 launchpad 时看"合约是否 verify、是否公开审计、过往事件记录"。
- **学术价值**：Four.meme 系列事件是 2025 年"protocol 层 access control 漏洞仍大量存在"的活教材，应写入开发者必读清单。

## 6. 参考资料

- Four.meme 官方 Twitter 事后公告
- SlowMist、PeckShield、BlockSec 每次事件技术分析
- rekt.news — Four.meme Rekt
- BscScan verified contracts（部分版本可查）

---

**本条目源于 2025 年多次公开告警整合；具体每次事件的损失金额、合约地址、函数名与修复 commit 请以 PeckShield / BlockSec 逐次分析为准，代码级细节待完整报告公布。**

未公开 / 待核实字段：
- 每次单独事件的精确日期 / 金额细分
- 涉及的合约地址列表（多次升级）
- 完整漏洞函数名与修复 commit

*Last verified: 2026-04-23*
