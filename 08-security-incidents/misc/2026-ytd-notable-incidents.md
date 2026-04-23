---
title: 2026 Q1–Q2 重大安全事件汇总
module: 08-security-incidents
category: 聚合索引
date: 2026-01-01
loss_usd: 0
chain: multi
severity: N/A (聚合)
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://rekt.news/leaderboard/
  - https://hacked.slowmist.io
  - https://skynet.certik.com
  - https://www.chainalysis.com/blog/
---

# 2026 Q1–Q2 重大安全事件汇总

> **本文件定位**：2026 年事件的**聚合索引**。Tier-1 / Tier-2 事件已建立独立档案（见 §1），本文件持续跟踪全年所有**损失 ≥ $1M** 或**攻击模式新颖**的事件。最后更新：**2026-04-23**。

## 1. 已建独立档案的 2026 事件

| 日期 | 事件 | 损失 | Tier | 类别 | 档案 |
|------|------|------|------|------|------|
| **2026-04-01** | **Drift Protocol**（Solana Perp DEX） | **$285M** | T1 | Supply-Chain + Key-Mgmt（DPRK 社工 → admin 签名 + durable nonce）| [→ 档案](../supply-chain-phish/2026-drift-protocol.md) |
| **2026-04-13** | **Hyperbridge**（Polkadot↔EVM 桥） | **$2.5M** | T2 | Bridge / Protocol-Bug（自研 MMR 验证库 `leaf_index` 边界漏洞）| [→ 档案](../bridge/2026-hyperbridge.md) |
| **2026-04-18** | **Kelp DAO / rsETH**（LayerZero 跨链） | **$292M** | T1 | Bridge / Supply-Chain（单 DVN + RPC 投毒 + DDoS）| [→ 档案](../bridge/2026-kelp-dao.md) |

### 小结

2026-04 单月三起叠加损失 **~$580M**，超过 2025 全年非 Bybit 事件总和。**归因主线**：Drift + Kelp DAO 均指向 **Lazarus Group / TraderTraitor 子团**（续接 Bybit 2025-02、Radiant 2024-10、DMM 2024-05 链条），**DPRK 对头部 DeFi/LRT/Perp 协议的定点社工与供应链攻击仍是 2026 最大风险源**。

## 2. Q1–Q2 其他 ≥ $10M 损失事件（尚未建档，按 rekt.news leaderboard）

| 日期 | 事件 | 损失 | 类别 | 状态 |
|------|------|------|------|------|
| 2026-01-08 | **Truebit** | $26.2M | DeFi / L2 | 待建档 |
| 2026-01-20 | **Makina** | $4.13M | DeFi | 待建档 |
| 2026-01-31 | **Step Finance** | $27.3M | DeFi | 待建档 |
| 2026-02-15 | **Moonwell** | $1.78M | DeFi（借贷） | 待建档 |
| 2026-02-22 | **Yieldblox** | $10.97M | DeFi | 待建档 |
| 2026-03-05 | **Solv** | $2.73M | DeFi | 待建档 |
| 2026-03-10 | **Aave** | $27.78M | DeFi（借贷） | 待建档 |
| 2026-03-15 | **Venus Protocol** | $3.7M | DeFi（借贷） | 待建档 |
| 2026-03-22 | **Resolv Labs** | $25M | DeFi（稳定币） | 待建档 |
| 2026-04-16 | **Rhea Finance** | $7.6–18.4M（双源不一致，待核） | DeFi | 待建档 |
| 2026-04-16 | **Grinex Exchange** | $13.1M | CEX | 待建档 |
| 2026-04-22 | **Volo Vaults** | $3.5M | DeFi | 待建档 |

## 3. 2026 年主要趋势（Q1–Q2 观察）

1. **Lazarus/TraderTraitor 仍主导 T1 事件**：Drift + Kelp DAO 两起即占 Q2 损失 95%+；与 2024–2025 模式完全一致（定点社工 + 供应链投毒）。
2. **LRT / Restaking 赛道成新目标**：Kelp DAO 是首起 LRT 桥大案，预示 EtherFi / Renzo / Puffer 等同类协议需优先整改 DVN/RPC 配置。
3. **跨链桥供应链攻击形态升级**：从"签名私钥被盗"（Ronin）进化到"**DVN + RPC 基础设施投毒 + DDoS 组合**"（Kelp DAO）——安全边界从链上合约外扩到运维基础设施。
4. **自研密码学库风险显性化**：Hyperbridge MMR 验证库漏洞是典型——审计预警被搁置 → 边界 fuzz 测试缺失 → 攻击者发现 → 损失爆发。
5. **借贷协议再次成为 Q1 DeFi 主要事故源**：Aave、Venus、Truebit、Moonwell 等连续出事，多数指向新功能/新市场上线过程中的边缘情况。
6. **Solana Perp DEX 安全关注度上升**：Drift 作为 Solana 头部 Perp 被 DPRK 定点攻击，预示 Solana 生态将经历 2022 年 Ethereum 式的 admin key 整改期。

## 4. 填充 SOP

以下事件升级到独立档案的门槛：
- **损失 ≥ $10M** 自动升级；
- **损失 $1M–$10M** 但**攻击模式新颖**（新漏洞类别、新社工向量、新链 / 新 VM）也升级；
- **损失 < $1M** 仅在此文件中聚合备忘。

升级到独立档案时，使用 [`../TEMPLATE-INCIDENT.md`](../TEMPLATE-INCIDENT.md) 模板；至少 3 条独立源；tx hash 真实可查；归因严格。

## 5. 来源清单

- [rekt.news leaderboard](https://rekt.news/leaderboard/) — 金额 & 时间主表
- [SlowMist Hacked Database](https://hacked.slowmist.io) — 时间顺序全量事件库
- [CertiK Skynet](https://skynet.certik.com) — 实时监控 + 历史库
- [Chainalysis Blog](https://www.chainalysis.com/blog/) — 年度 / 半年度统计 + 归因
- [TRM Labs](https://www.trmlabs.com/resources) — 国家级归因
- [PeckShield / BlockSec / Cyvers / Dedaub](https://twitter.com/PeckShieldAlert) — 事件即时告警
- [ZachXBT](https://x.com/zachxbt) — 独立链上侦探
- [FBI IC3 PSA](https://www.ic3.gov/Media/News) — 执法归因

---

*Last verified: 2026-04-23*
