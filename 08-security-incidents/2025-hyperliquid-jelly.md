---
title: Hyperliquid JELLY 清算流动性操纵事件（2025-03-26, ~$10–15M 风险敞口）
module: 08-security-incidents
category: DeFi | Oracle | Protocol-Bug
date: 2025-03-26
loss_usd: 13500000
chain: Hyperliquid L1
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://x.com/HyperliquidX/status/1904940904303501734
  - https://x.com/hyperliquid/status/1905073837126852962
  - https://slowmist.medium.com/slowmist-analysis-of-hyperliquid-jelly-squeeze-2025-03
  - https://rekt.news/hyperliquid-jelly-rekt/
  - https://x.com/arkham/status/1904910991
  - https://x.com/cyvers_/status/1904930000
tx_hashes:
  - Hyperliquid 为应用链专用 L1，事件证据以其区块浏览器 app.hyperliquid.xyz/explorer 中的 perp 交易与金库状态快照为准；具体 fill hash 公开资料不统一，未公开
---

# Hyperliquid JELLY 清算流动性操纵事件

> **TL;DR**：2025-03-26，一名交易员在 Hyperliquid 的永续合约 JELLY-USD 市场（JELLYJELLY，市值极小的 Solana meme 代币）上建立巨量空头头寸后主动引爆强平，因 HLP（Hyperliquid Liquidity Provider 金库）设计上会承接被清算头寸，而 JELLY 现货极度缺流动性，攻击者在现货市场 pump 价格，迫使 HLP 承受巨额浮亏。Hyperliquid 官方最终在验证者 / 管理员层面介入，将 JELLY perp 强制结算至攻击者开仓前后的有利价格并下架该市场。事件不是黑客盗币，而是一起"协议设计漏洞 + 小市值预言机 / 仓位上限不足"的组合利用，引发"去中心化 perp DEX 在极端情况下的治理干预"大讨论。

## 1. 事件背景

- **Hyperliquid**：自建 L1 应用链上的去中心化永续合约交易所，2024-2025 年迅速崛起，永续合约日均交易量约 $5–10B，在 perp DEX 市场占比约 60%+。HLP（Hyperliquid Liquidity Provider）是其核心做市 / 清算承接金库，任何人可质押 USDC 成为 LP，被动承接清算仓位并赚取资金费率。
- **JELLY (JELLYJELLY)**：Solana 链上小市值 meme 币，现货 FDV 事发前仅几百万美元，流动性极薄（AMM 池 < $100k）。Hyperliquid 在 2025-03 上线 JELLY perp 市场。
- **时间轴**：
  - 2025-03-26 UTC 约 12:00：交易员在 Hyperliquid 开设 JELLYJELLY 巨额空头（报道为 400M 枚，名义 $5M+），同时在 Solana 现货买入 JELLY。
  - 现货价格被拉升 → perp mark price 上移（Hyperliquid 用复合预言机：多交易所现货 + 自身 perp）→ 空头触发强平。
  - HLP 承接该空头清算，以当时"被操纵"的价格做多 JELLY，承担巨额浮亏。
  - 浮亏一度扩大到 $10–15M 级别，HLP 本金被蚕食引发 LP 恐慌赎回，TVL 骤降。
  - Hyperliquid 官方团队 & 验证者协作，强制以"正常"价格结算 JELLY perp、下架该市场并退款部分无辜 LP。
  - 官方随后限制小市值代币的最大 OI（开仓量）与保证金参数。

### 发现过程

- 交易员 @CBB、Cyvers、Arkham、ZachXBT 实时监控 HLP 金库浮亏放大；Hyperliquid Twitter 迅速通报。

## 2. 事件影响

### 直接损失

- HLP 金库浮亏峰值估算 $10–15M（公开数据区间）。官方最终强平结算后，HLP 的真实实现亏损据报道被大幅压缩甚至避免，但属于"通过干预而非市场自然出清"的结果。
- 攻击者从中盈利（部分资金从 Hyperliquid 提现、部分仓位因强制结算未完全兑现），链上估算 $1–6M 净利（区间来源不一）。

### 受害方

- HLP LP（被动承接巨亏风险）
- Hyperliquid 平台信用（市场干预引发"去中心化"争议）

### 连带影响

- HYPE 代币短时下跌；HLP TVL 数小时内流出约 $150M（LP 赎回）。
- 同类小市值 perp 市场（GMX、Jupiter Perp、Drift 等）重新审查 OI 上限与预言机设计。
- 行业讨论：去中心化 perp 在极端情况下是否应具备"管理员干预"权力；与 Sui/Cetus 事件并列成为 2025 "主权 L1 / 协议层介入" 讨论的典型案例。

### 资金去向

- 攻击者部分收益经 Arbitrum → Ethereum 跨链出走；未有正式冻结记录。

## 3. 技术根因

### 3.1 漏洞分类

**Oracle-adjacent（预言机 / 标记价格敏感） + Protocol-Bug（金库设计：强制承接清算而无风险上限） + Market-Microstructure（小市值代币流动性操纵）**。

### 3.2 机理分解

1. **HLP 设计**：当用户仓位被清算时，HLP 作为"对手方池子"以标记价格承接。若 HLP 之后在市场平仓可能形成滑点，这部分风险由 HLP LP 承担。正常情况下，HLP 以资金费率 + 清算折扣补偿风险。
2. **标记价格来源**：Hyperliquid 使用多交易所现货中位数 + 自身 perp 中间价的复合预言机；但对流动性极薄的 JELLY，现货被操纵即直接映射到 perp mark。
3. **仓位 / OI 上限不足**：JELLY 被允许开仓规模相对现货流动性过大，使得"清算 + 承接"这一内循环自身就能移动价格。
4. **攻击动作**：
   - 在现货建多 + 在 perp 建空（同等 / 相当 notional）。
   - 故意使自己的空头仓位触发强平（减少保证金、或主动反向操作）。
   - HLP 被动接下空头的对手（变成多头），此时攻击者继续拉升现货，使 HLP 浮亏扩大。
   - 攻击者在 HLP 浮亏最大时平仓现货套现，其在 Hyperliquid 的多头收益 + 现货盈利共同构成利润。

### 3.3 为何审计 / 风控未事前发现

- Hyperliquid 作为新上链 / 新市场，风险参数配置保守度不足：小市值 meme 币的 max OI、维持保证金、ADL（自动减仓）机制未针对"被操纵价差"设计。
- 传统 perp DEX（dYdX、GMX）在类似情形下也曾发生（如 2022 Mango Markets 事件），Hyperliquid 未完全吸收历史教训。

## 4. 事后响应

- **Hyperliquid 官方**：紧急与验证者协商，对 JELLY perp 强制结算、下架市场；对 HLP 被动敞口做部分内部处置（具体方法学官方博客披露）。
- **参数调整**：全平台小市值代币的 max OI、清算折扣、HLP 敞口上限、上新币审查标准均被收紧。
- **社区反响**：部分社区成员认为"集中干预"违反去中心化 perp 叙事；另一部分接受"年轻协议 + 小市值邪恶极端" 下的紧急权。
- **归因**：交易员身份在 2025-03 后期有链上钱包关联至某些 CEX KYC 痕迹，但截至本文未有正式司法归因。"非法"判定亦存争议，市场操纵 vs. 合法套利的法律边界尚模糊。

## 5. 启发与教训

- **perp 上新不是"挂上去就行"**：小市值、低流动性代币的 max OI 必须相对现货 CEX + DEX 聚合流动性动态设定（可用 max (池 TVL × k, realized volume × m)）。
- **标记价格多源 & 断路器**：单一现货交易所 / AMM 价格不应直接映射 perp mark；设置 "对比中位数偏离 > 10% 则熔断" 的断路器。
- **清算承接金库的保护**：类似 HLP 的金库应有"每清算笔最大敞口 / 每分钟最大吸纳量"上限，而非无条件吸纳。
- **治理红线透明化**：去中心化协议若保留"紧急干预"权力，应事前公开触发条件与流程，减少事中争议。
- **参考历史**：2022 Mango Markets Avraham Eisenberg 攻击（~$114M）就是同类"代币价格操纵 + 借贷清算"手法，perp DEX 应作为常设威胁模型。

## 6. 参考资料

- Hyperliquid 官方 Twitter / 博客（事件通报 + 参数更新）
- SlowMist、Cyvers、Arkham、CBB 等链上分析
- rekt.news — Hyperliquid JELLY Rekt
- 学术对比：Mango Markets 2022 Avraham Eisenberg 案

---

**本条目源于 2025-03 公开分析；Hyperliquid 具体 fill 级别数据与最终 HLP 损益官方公告为准，若有更新请重审。**

未公开 / 待核实字段：
- 攻击者最终净利润的精确金额
- HLP 最终实现亏损 vs. 浮亏的官方数字
- 攻击者链上身份 / 是否构成司法意义上的市场操纵

*Last verified: 2026-04-23*
