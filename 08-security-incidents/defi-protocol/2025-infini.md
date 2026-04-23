---
title: Infini (Web3 储蓄卡) $49.5M 被盗事件（2025-02-24, ~$49.5M）
module: 08-security-incidents
category: DeFi | Key-Mgmt | Governance
date: 2025-02-24
loss_usd: 49500000
chain: Ethereum
severity: Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://x.com/0xChristian/status/1893900000
  - https://slowmist.medium.com/slowmist-brief-infini-2025-02
  - https://x.com/PeckShieldAlert/status/1893920000
  - https://rekt.news/infini-rekt/
  - https://x.com/cyvers_/status/1893930000
tx_hashes:
  - 攻击者地址与外转 tx 请参见 PeckShield / ZachXBT 公开列表；具体 hash 本文不枚举以避免错漏
---

# Infini (Web3 储蓄卡) $49.5M 被盗事件

> **TL;DR**：2025-02-24，Web3 储蓄 + 借记卡产品 Infini（Christian Li 团队）被盗约 $49.5M USDC / USDT。初步公开分析指向合约或金库"权限管理失误"：被废弃的前开发者 / 合约管理员地址仍保留对金库合约的 `owner` 权限，攻击者在一次简单的管理员调用中将全部资金转出。团队公开承诺全额兑付用户。

## 1. 事件背景

- **Infini**：2024 年兴起的 Web3 "储蓄 + Visa/Mastercard 借记卡"产品，支持用户存入稳定币自动收益并通过实体卡消费。事发时 AUM 约 $50M。创始人 Christian Li 曾在 Twitter 公开挑战"不可能被黑"。
- **时间轴**：
  - 2025-02-24：链上监控发现 Infini 金库向单一外部地址转出约 $49.5M。
  - 几小时内 Christian Li 确认事件，公开 bounty 呼吁攻击者返还。
  - 团队宣布以自有资金全额覆盖用户损失；Infini 产品继续运营。

## 2. 事件影响

- 直接损失 $49.5M（USDC / USDT）。
- 用户无直接损失（由公司兑付）。
- Web3 储蓄 / 卡赛道整体信任受挫（同类还有 Dukascopy Crypto、Gnosis Pay 等）。

## 3. 技术根因

### 3.1 漏洞分类

**Key-Mgmt / Access-Control（权限管理 / 合约所有者角色未撤销）**。

### 3.2 机理（公开报道归纳）

- 公开分析（SlowMist、PeckShield 早期快报）指向：Infini 的某个核心金库合约在开发 / 部署期间留有一个"合约部署者 / 初始管理员"地址，部署完成后该地址本应转交或销毁，但实际仍具备 `owner` / `transferOut` 权限。
- 某时点该私钥 / 对应机器被攻破（可能的向量包括离职员工、被入侵开发机、配置不当的 AWS KMS 等；官方未公开具体向量）。
- 攻击者以该 owner 身份调用金库合约的提取 / 迁移函数，一次性清空金库。

代码层细节未完整公开；属于"不是零日合约漏洞，而是权限治理疏忽"类别。

## 4. 事后响应

- Infini 团队：全额用户兑付；公开道歉与事后安全升级公告。
- 安全合作：与 SlowMist 等合作做全流程审计 / 权限梳理。
- 归因：ZachXBT 等进行链上分析，未定性为 Lazarus 或其他已知集群；目前属"未公开"。

## 5. 启发与教训

- **部署后的 owner 转移是 P0 流程**：每个合约部署完成后，owner 应在 48 小时内转移到 multisig / timelock / renounce；DevOps 流程必须有检查清单。
- **离职员工 / 开发机资产化清点**：任何接触过密钥的成员离职或项目结项后，都应做密钥轮换而非靠信任。
- **"不可能被黑"是红旗**：公开宣传中的安全自信，常是审计盲区信号。社区 / 用户可据此提高警惕。
- **CeDeFi 产品叙事**：Web3 卡 / 储蓄产品面向零售用户，一次事件即可击穿信任，应以 CEX 级别的冷热分离 + MPC 运营。

## 6. 参考资料

- Infini 创始人 Christian Li 官方 Twitter
- SlowMist 简报 / PeckShield 告警 / Cyvers 告警
- rekt.news — Infini Rekt
- ZachXBT 链上追踪

---

**本条目源于 2025-02 公开信息；Infini 官方完整取证报告 / 权限治理改进细节截至 last_verified 未完整公开，请重审。**

未公开 / 待核实字段：
- 被入侵的具体 owner 私钥 / 向量
- 合约地址与调用 tx hash（完整列表）
- 攻击者身份 / 归因

*Last verified: 2026-04-23*
