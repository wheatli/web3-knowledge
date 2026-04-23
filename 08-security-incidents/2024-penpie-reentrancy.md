---
title: Penpie 重入攻击（2024-09-03, ~$27M）
module: 08-security-incidents
category: DeFi | Protocol-Bug
date: 2024-09-03
loss_usd: 27000000
chain: Ethereum, Arbitrum
severity: Tier-3
last_verified: 2026-04-23
primary_sources:
  - https://rekt.news/penpie-rekt/
  - https://slowmist.medium.com/slowmist-analysis-of-the-penpie-hack
  - https://www.halborn.com/blog/post/explained-the-penpie-hack-september-2024
  - https://x.com/peckshield/status/1830857443635683771
  - https://medium.com/chainsecurity/penpie-incident-post-mortem
tx_hashes:
  - 0x42b2ef33c54ecd75ba5a1b7c49b211a7b44f484bb74ab66fb1fe1ef6c69ad643 (Ethereum, 主攻击 tx)
  - 攻击者钱包: 0x7a2f4d625fb21f5e51562ce8dc2e722e12a61d1b
  - 攻击合约: 0xc0Eb7e6E2b94aA43BDD0c60E645fe915d5c6eb84
---

# Penpie 重入攻击

> **TL;DR**：2024-09-03 13:36 UTC，Pendle 生态收益聚合器 Penpie 被**自造 Pendle Market + 跨合约重入**盗走 ≈ $27M。根因是 `batchHarvestMarketRewards` 的 `_markets` 无白名单，且在 `redeemRewards()` **之后**才读 `stakingBalance` 分配奖励；攻击者回调重入 `MasterPenpie.deposit` 放大 stakingBalance 骗奖励。Pendle 本身无漏洞，无许可 Market 的 composability 特性被滥用。

## 1. 事件背景

- **协议**：Penpie，Pendle 之上收益聚合器，TVL ≈ $165M。Pendle 允许任何人无许可创建 Market，redeemRewards 回调调用者——攻击面起点。
- **时间轴**：09-03 13:36 UTC 流出 $27M → 14:30 Pendle "Pause All" → 09-04 $1M 白帽被拒，资金 Tornado Cash。
- **审计**：WatchPug、Zokyo 未发现；ChainSecurity 事前 2 周审计范围不含该函数。

## 2. 事件影响

损失 ≈ $27M（11,697 stETH + 3,437 ETH + USDC/USDT/rsETH 等）；PNP -40%、PENDLE -9%；Pendle 暂停 48h。

## 3. 技术根因（代码级）

### 3.1 分类

**Cross-Contract Reentrancy + 外部合约错误信任假设**。

### 3.2 关键代码

```solidity
// PendleStakingBaseUpg
function batchHarvestMarketRewards(address[] calldata ms, uint256) external {
    for (uint i; i < ms.length; i++) _harvest(ms[i]); // ❶ 无白名单
}
function _harvest(address m) internal {
    uint256 b = IERC20(reward).balanceOf(address(this));
    IPendleMarket(m).redeemRewards(address(this));       // ❷ 外部回调
    uint256 g = IERC20(reward).balanceOf(address(this)) - b;
    uint256 sb = MasterPenpie.stakingBalance(m);         // ❸ 回调后才读
    _distributeReward(g, sb);
}
// MasterPenpie: ❹ 独立 nonReentrant
function deposit(address m, uint256 a) external nonReentrant { stakingBalance[msg.sender][m] += a; }
```

**错在哪**：❶ 无白名单，攻击者自造 Market 可传入；❷❸ 违反 CEI；❹ 跨合约无全局锁，回调可放大 stakingBalance。

### 3.3 攻击步骤

部署 `EvilMarket` + MasterPenpie 预 deposit 注册 → 闪电贷借 SY → `batchHarvestMarketRewards([EvilMarket])` → 回调 `deposit(EvilMarket, huge)` 放大 stakingBalance → 按放大余额分奖励 → `multiclaim` 提 PNP + SY 赎 ETH → 还闪电贷，$27M → Tornado Cash。

### 3.4 为何未发现

审计视 Pendle Market 为可信；跨合约缺全局锁；入口 permissionless。

## 4. 事后响应

Penpie Q4 引入 Market 白名单 + 全局重入锁；Pendle 发 Integration Security Guide。

## 5. 启发与教训

- permissionless 外部合约当敌意输入；跨合约依赖用**全局 reentrancy guard**；快照必须在外部调用**之前**（CEI）。Foundry invariant 覆盖"单 tx 不可跨合约放大 stakingBalance"。

## 6. 参考资料

- rekt.news: Penpie Rekt
- 慢雾 SlowMist Penpie 事件分析
- Halborn: Penpie Hack Explained (Sep 2024)
- ChainSecurity: Penpie Incident Post-Mortem
- PeckShield Alert（2024-09-03）

---

*Last verified: 2026-04-23*
