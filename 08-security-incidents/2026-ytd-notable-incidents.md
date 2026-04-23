---
title: 2026 Q1 重大安全事件汇总（占位 / 待核验）
module: 08-security-incidents
category: DeFi | CEX-hack | Bridge | Oracle | Wallet | Governance
date: 2026-01-01
loss_usd: 0
chain: multi
severity: Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://hacked.slowmist.io/
  - https://rekt.news/leaderboard/
  - https://skynet.certik.com/
  - https://www.chainalysis.com/blog/
  - https://www.trmlabs.com/resources
tx_hashes:
  - N/A（本文件为占位 / 目录性汇总，不单独列具体 tx）
---

# 2026 Q1 重大安全事件汇总（占位 / 待核验）

> **重要声明**：截至本文 last_verified 日期 **2026-04-23**，2026 年 Q1 的重大事件完整列表需人工核验于 **SlowMist Hacked Database (hacked.slowmist.io)**、**rekt.news leaderboard**、**CertiK Skynet**、**Chainalysis Crypto Crime 实时看板** 等权威来源。本文件仅作为结构性占位，后续会话将根据上述源逐条填充 — **不在此处编造任何 2026 年具体事件**。

## 1. 如何使用本文件

本文件在 knowledge base 中充当 "2026-Q1 聚合索引" 角色。后续更新应遵循以下流程：

1. **确认事件真实性**：从下述至少 2 个来源交叉验证同一事件（日期、金额、项目、分类）。
2. **若事件 ≥ Tier-3 门槛（损失 ≥ $2M 或具有新颖攻击模式）**：为其创建独立文件 `2026-<项目名>.md`，并在本汇总文件中添加一行链接。
3. **若事件 < 阈值但仍值得备忘**：在本文件"小事件 / 趋势观察"章节中以一两句记录。
4. **严格遵循模板硬性规则**：≥ 3 独立来源、真实 tx hash、不捏造代码。

## 2. 权威来源清单（按优先级）

| 来源 | 用途 | URL |
|---|---|---|
| SlowMist Hacked Database | 时间顺序全量事件库 | hacked.slowmist.io |
| rekt.news leaderboard | 按损失金额排行，含深度分析 | rekt.news/leaderboard |
| CertiK Skynet | 实时监控 + 历史库 | skynet.certik.com |
| Chainalysis Crypto Crime | 年度 / 半年度统计 + 归因 | chainalysis.com/blog |
| Elliptic | 链上追踪 + 洗钱路径 | elliptic.co/resources |
| TRM Labs | 国家级归因（Lazarus 等） | trmlabs.com/resources |
| PeckShield / BlockSec / Cyvers / Dedaub | 事件即时告警 + 技术分析 | 官方 Twitter / Blog |
| ZachXBT | 独立链上侦探，擅长归因 | @ zachxbt Twitter |
| FBI IC3 PSA | 官方执法归因（Lazarus 等） | ic3.gov/Media/News |

## 3. 候选事件框架（待填充）

以下为待填充的结构模板，**不包含任何具体事件内容**——仅供后续会话补录时对齐：

### 3.1 CEX / 托管机构事件

- 交易所 / 托管热钱包被盗
- MPC / HSM 签名服务后端入侵
- 监管合规事件（冻结 / 没收）

### 3.2 DeFi 协议事件

- 借贷协议 / 稳定币协议漏洞
- DEX / CLMM 数学精度事件
- 预言机操纵 / 治理攻击
- 跨链桥事件

### 3.3 钱包 / 基础设施事件

- 前端 UI 欺骗（接续 Bybit 2025 模式）
- 硬件钱包 / 签名栈事件
- RPC / indexer 供应链事件
- 浏览器插件 / npm 包供应链

### 3.4 社工 / 钓鱼 / 市场结构

- 大型钓鱼攻击（涉及多用户）
- 代币市场结构事件（接续 MANTRA 2025 模式）
- DPRK / Lazarus 新战术披露

## 4. 2025 年数据基线（用于 2026 对比）

作为占位的对比参考，2025 年全年已知重大事件轮廓（用于校验 2026 数据时的量级合理性）：

- **Bybit (2025-02)**：$1.46B，史上最大 CEX 被盗，Safe UI 欺骗 + Lazarus 归因
- **Cetus (2025-05)**：$223M，Sui CLMM 数学溢出
- **Phemex (2025-01)**：~$69–85M，多链热钱包
- **Infini (2025-02)**：~$49.5M，权限管理
- **其他 Tier-2/3**：Abracadabra、zkLend、Hyperliquid JELLY、KiloEx、Four.meme、MANTRA 信任危机 等

2025 年全年仅 Bybit 一起事件的损失即超过 2024 年全年加密行业被盗总额，对"CEX 冷钱包签名流程"的影响延续到 2026。

## 5. 填充 SOP（后续会话使用）

更新此文件时应遵守：

1. **不要主动创造事件**。若 Claude 训练数据中没有 2026 事件的具体信息，应在会话里请用户提供链接，或使用 WebFetch 抓取上述权威源。
2. **每条事件填充前做 cross-check**：至少 2 个独立来源；日期 / 金额 / 分类必须一致。
3. **损失金额带日期快照**：加密价格波动剧烈，写清"$X（按 YYYY-MM-DD 价格）"。
4. **归因严格**：Lazarus 等国家级归因仅当 FBI / Chainalysis / TRM / ZachXBT 明确披露时才写入。
5. **创建独立文件的阈值**：损失 ≥ $2M OR 攻击模式新颖（新漏洞类别、新社工向量、新链 / 新 VM）。

## 6. 当前状态

- 本文件当前为 **占位性质**。
- 最近一次审查 / 填充：2026-04-23（初始化）。
- 下次审查计划：当用户提供 2026 事件信息 或 明确触发 web 检索时。

---

**本条目是结构性占位，不包含任何未经核实的 2026 事件内容。任何后续填充都必须基于上列权威源的交叉验证。**

*Last verified: 2026-04-23*
