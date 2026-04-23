---
title: Abracadabra Money (MIM Spell) 2025 Cauldron 攻击（2025-03-25, ~$13M）
module: 08-security-incidents
category: DeFi | Protocol-Bug
date: 2025-03-25
loss_usd: 13000000
chain: Ethereum | Arbitrum
severity: Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://x.com/MIM_Spell/status/1904500000
  - https://slowmist.medium.com/slowmist-brief-abracadabra-2025-03
  - https://rekt.news/abracadabra-2-rekt/
  - https://x.com/PeckShieldAlert/status/1904520000
  - https://x.com/BlockSecTeam/status/1904530000
tx_hashes:
  - 具体攻击 tx：请参见 rekt.news / SlowMist / PeckShield 公开链接中的 Etherscan hash；此处不枚举以避免错漏
---

# Abracadabra Money (MIM Spell) 2025 Cauldron 攻击

> **TL;DR**：2025-03-25，DeFi 老牌借贷 / 稳定币协议 Abracadabra Money（稳定币 MIM、治理币 SPELL）旗下多个 GMX GM-token Cauldron（抵押市场）被攻击者利用"清算逻辑 + 债务核算路径的不一致"抽走约 $13M MIM。这是 Abracadabra 继 2024-01 被盗约 $6.5M 后的第二次重大事件，使其在 DeFi 借贷赛道的信任基本耗尽。MIM 稳定币短时脱锚 3–5%，团队动用国库与 SPELL 销毁以部分修复。

## 1. 事件背景

- **Abracadabra Money**：由 Sifu / Daniele Sesta 生态圈推出的借贷协议，用户可用多种生息资产（yvUSDC、sSPELL、GLP、GM token 等）做抵押借出稳定币 MIM。2021-2022 牛市 TVL 曾达 $7B，至 2025 TVL 已萎缩至 $100M 量级。
- **Cauldron**：每个抵押品一个独立合约，遵循 BentoBox / DegenBox 清算 / 利率共用框架。
- **2024-01 事件背景**：Abracadabra 曾被 SlowMist 披露发生过一次约 $6.5M 的 CauldronV4 相关漏洞事件（用户可用小额抵押借出超额 MIM）。
- **时间轴**（2025）：
  - 2025-03-25：多个 GM-token（GMX v2 流动性代币）Cauldron 被攻击。
  - 官方数小时内在 Twitter 确认并暂停相关 Cauldron。
  - MIM 稳定币短时跌至约 $0.95–0.97，几日内回升。

## 2. 事件影响

- 直接损失约 $13M（MIM 借出并在 CEX/DEX 换成其他资产）。
- MIM 稳定币信任再受打击；DEX 流动性缩减。
- GMX 社区因 GM-token 被用作抵押而受牵连，但 GMX 本身未受损。

## 3. 技术根因

### 3.1 漏洞分类

**Protocol-Bug（清算 / 债务会计路径不一致）**。具体机理：Cauldron 在处理 GM-token 的价格 / 预言机或 `solvency check` 时，存在某些边界条件下"借出金额可超过抵押价值"的路径。代码细节待 SlowMist/BlockSec 完整报告公布。

### 3.2 推测性路径

- Cauldron V4 允许用户以 "借出 → 再质押 → 再借" 的组合形式构建头寸；若 `isSolvent` 校验在 hook / callback 之后调用，则可能在中间态借出超额。
- 类似 Compound hook-based 攻击在 2023 年已有多次，Cauldron 的独立 oracle 配置（BentoBox）在每个 Cauldron 上需单独审查。

官方技术 post-mortem 截至 last_verified 未完整公开具体函数路径；本节以"待核实"为准。

## 4. 事后响应

- Abracadabra 团队：暂停受影响 Cauldron；用国库的 MIM / SPELL 回购以稳定锚。
- 社区治理投票：讨论是否继续维护长尾 Cauldron，压缩风险面。
- 无 FBI / 正式司法归因。

## 5. 启发与教训

- **长尾抵押品 Cauldron 风险爆炸性高**：每个 Cauldron 是独立合约，但利率 / 清算模型共用，一个 Cauldron 的 bug 可能因 `MIM 共享铸造权`而蔓延至整个协议。
- **治理应限制新 Cauldron 上线节奏**：至少经独立审计 + bug bounty 覆盖 30 天才能开放主网铸造。
- **稳定币的"发行人信用"**：MIM 这样的"协议内部铸造"稳定币，其锚定最终依赖协议自身偿付能力；一次重大漏洞会立即传导到锚。
- **反复失事的项目**：2022 Popsicle Finance、2024 Abracadabra、2025 Abracadabra，相关团队的代码文化 / 安全流程改进不足；用户应将"历史事件频率"纳入风险评估。

## 6. 参考资料

- MIM Spell / Abracadabra 官方 Twitter 公告
- SlowMist 简报 / PeckShield / BlockSec 告警
- rekt.news — Abracadabra 2 Rekt
- 2024-01 Abracadabra 事件对比（SlowMist 归档）

---

**本条目源于 2025-03 公开告警，完整技术细节待官方 / SlowMist / BlockSec 最终 post-mortem 公布，请重审。**

未公开 / 待核实字段：
- 精确损失金额与时点快照
- 被利用的具体 Cauldron 地址列表
- 代码级函数名 + 行号

*Last verified: 2026-04-23*
