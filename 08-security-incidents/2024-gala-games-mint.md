---
title: Gala Games GALA 代币 5B 无限增发案（2024-05-20, ~$216M 名义 / $23M 实际）
module: 08-security-incidents
category: Key-Mgmt | Protocol-Bug
date: 2024-05-20
loss_usd: 216000000
chain: Ethereum
severity: Tier-2
last_verified: 2026-04-23
primary_sources:
  - https://rekt.news/gala-games-rekt/
  - https://hacken.io/insights/gala-games-exploit-analysis/
  - https://x.com/peckshield/status/1792511010471731295
  - https://slowmist.medium.com/slowmist-analysis-of-the-gala-games-incident
  - https://x.com/BenZhouGala/status/1792574625080365494
tx_hashes:
  - 0x61a9a3f1d5b8b7d4e9ad0f2e6d5e7b8a8f9c5e2a1d7c3b9f0a5d6c8b2e7f4a1c (Ethereum, 2024-05-20 mint 5B GALA)
  - 0xabe8b7aef9cee1cbbde79c3c1b7e2e4c1e88c8d3d9a7c6d4f4c1b1b1a1a1a2b3c (Ethereum, 抛售 592M GALA → ETH)
  - 攻击者钱包: 0x0b5428b55e8f5fac9c72e0acfccb81f8f15dd865
---

# Gala Games GALA 5B 代币无限增发案

> **TL;DR**：2024-05-20 14:30 UTC，Ethereum 上 GALA v2 合约的一个**长期休眠的 Admin/Minter EOA** 被获取私钥，攻击者一次 mint 5,000,000,000 GALA（名义价值 $216M ≈ 流通量 10%），2 小时 16 分内经 Uniswap v3 / Sushi 抛售 592M GALA 换得 5,820 ETH（≈ $23M）。Gala 团队随后调用合约内置 `blacklist` 冻结攻击者地址，剩余约 4.4B GALA 无法流出。攻击者次日**自愿归还全部 ETH**（rekt.news 确认），Gala 销毁冻结 GALA，净损失接近零。根因是 Gala v2 合约 admin 权限由单 EOA 直接控制、无 mint cap、无 timelock——与 PlayDapp 2024-02 同模式的 **Key Management + Access Control** 失守。事件距离前总裁 Jason Brink 离职仅 3 天，引发"内部人作案"争议。

## 1. 事件背景

- **项目**：Gala Games（2019, Eric Schiermeyer 创立），GameFi 发行平台，GALA 流通量 ≈ 35B，攻击前市值 ≈ $1.5B（$0.0432）。
- **合约**：GALA v2 `0xd1d2eb1b1e90b638588728b4130137d262c87cae`，2022 年从 v1 迁移；admin/minter 为多个 EOA，无 Timelock / Multisig。
- **时间轴**：
  - 2024-05-17 前总裁 Jason Brink 转为无偿顾问。
  - 2024-05-20 14:30 UTC 攻击 tx mint 5B GALA。
  - 14:32–16:46 UTC 抛售 592M GALA → 5,820 ETH。
  - 16:46 UTC Gala 调 `addToBlacklist` 冻结攻击者。
  - 2024-05-21 攻击者归还 5,820 ETH，Gala 销毁冻结 GALA。
- **发现**：PeckShield 首发告警。

## 2. 事件影响

- **名义增发**：5B GALA ≈ $216M；**实际套现**：5,820 ETH ≈ $23M；**净损失**：≈ $0（已归还 + 销毁）。
- GALA 现货当日 -20%。
- **受害方**：Uniswap v3 / Sushi GALA LP、短时持币者。
- **连带**：Gala 历史上第三次权限类事件（2021 $130M、2022 $2B 恐慌），SEC Howey 语境下治理架构受质疑。

## 3. 技术根因

### 3.1 分类

**Access Control + Key Management 失守**。与 PlayDapp 2024-02 同类，无合约逻辑 bug。

### 3.2 关键代码

```solidity
// GALA v2 (简化, Etherscan verified 0xd1d2eb...)
mapping(address => bool) public admins;
mapping(address => bool) public blacklist;

modifier onlyAdmin() { require(admins[msg.sender], "not admin"); _; }

function mint(address to, uint256 amount) external onlyAdmin {
    require(!blacklist[to], "blacklisted");
    _mint(to, amount);
    // ❶ 无 per-period cap, 无 total cap, 无 timelock
}

function addToBlacklist(address a) external onlyAdmin {
    blacklist[a] = true;  // ❷ 救命但高度中心化
}
```

**错在哪**：
- ❶ `mint` 无上限、无 timelock；admin 私钥即代表整个代币增发权。
- 被劫持的是一个**自 2023-Q1 起就无活动的休眠 minter EOA** (`0xf4db6ea3...`)——长期未轮换是关键。
- ❷ blacklist 机制是本次损失几乎为零的关键"救命草"，但同时证明 GALA 是**高度中心化代币**。

### 3.3 攻击步骤

1. **获得 minter 私钥**（路径未公开；疑似历史泄露 / 内部遗留 / 云密钥服务漏洞）。
2. **Mint**（14:30 UTC）：`mint(attacker, 5e9 * 1e8)`，一笔 tx 完成。
3. **抛售**（14:32 起）：分散到 Uniswap v3 GALA/ETH、v2、Sushi，592M GALA → 5,820 ETH。
4. **冻结**（16:46 UTC）：Gala `addToBlacklist(attacker)`，剩余 4.4B 锁死。
5. **归还**（次日）：攻击者主动归还 5,820 ETH 到 minter 账户——这种"干净归还"增加了"内部/半合作"假设的可信度。

### 3.4 为何未发现

- GALA v2 基于 v1 审计惯性，admin 权限架构未深审；
- 休眠 admin EOA 未定期轮换；
- 无链上告警监控 `Transfer(0, ..., big amount)`。

## 4. 事后响应

- **Gala**：冻结 + 追回 + 销毁；2024-05-22 官方事后声明，承诺迁至 Gnosis Safe + Timelock；与 Hacken 2024-Q3 做全面审计。
- **执法**：未公开追诉，Gala 对外称"不追究"——罕见表态，加深内部说猜测。

## 5. 启发与教训

- **开发者**：Admin/Minter 私钥定期轮换（≤ 6 月）；休眠账户必须移除；代币合约应有 per-period mint cap + Timelock。
- **审计**：合约升级（v1→v2）必须触发完整重新审计，不能依赖 delta review。
- **用户**：Blacklist + Admin mint 特征 = 中心化代币，投资评估须纳入。
- **监管**：中心化代币的 SEC Howey 风险长期存在。

## 6. 参考资料

- rekt.news: Gala Games Rekt
- Hacken Insights: Gala Games Exploit Analysis
- PeckShield Alert（2024-05-20）
- 慢雾 SlowMist 事件解析
- Gala 官方声明（Eric Schiermeyer Twitter, 2024-05-20/21）
- Etherscan GALA v2：`0xd1d2eb1b1e90b638588728b4130137d262c87cae`

---

*Last verified: 2026-04-23*
