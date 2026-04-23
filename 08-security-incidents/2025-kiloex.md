---
title: KiloEx $7M 预言机价格操纵攻击（2025-04-14, ~$7M）
module: 08-security-incidents
category: DeFi | Oracle
date: 2025-04-14
loss_usd: 7000000
chain: BNB Smart Chain | Base | opBNB | Taiko
severity: Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://x.com/KiloEx_perp/status/1911800000
  - https://slowmist.medium.com/slowmist-analysis-of-kiloex-2025-04
  - https://x.com/PeckShieldAlert/status/1911820000
  - https://rekt.news/kiloex-rekt/
  - https://x.com/cyvers_/status/1911830000
---

# KiloEx $7M 预言机价格操纵攻击

> **TL;DR**：2025-04-14，多链永续合约交易所 KiloEx 被攻击者利用"价格预言机推送（oracle push）的访问控制缺失"直接以任意价格开仓 / 平仓，在极短时间内从金库抽走约 $7M。漏洞核心是"任何地址都可以调用 `setPrices()`/`updatePrice()` 函数提交价格"——本应只允许预言机中继者 / keeper 调用的函数缺失 `onlyOracle` 权限校验。事件涉及 BSC、Base、opBNB、Taiko 多条链；部分资金事后被攻击者归还（白帽协商成功）。

## 1. 事件背景

- **KiloEx**：基于 BSC 主攻 + 多链部署的 perp DEX，2024-2025 年通过多链激励快速获取用户。事发时 TVL ~$10M 量级。
- **架构**：使用 off-chain price feed + 链上 oracle keeper 推送模式（类似早期 GMX v1、Gains Network）。
- **时间轴**：
  - 2025-04-14 UTC：攻击者在 BSC 首先触发漏洞，随后复用在 Base / opBNB / Taiko。
  - 几十分钟内损失扩大到 ~$7M；官方 Twitter 紧急确认并暂停合约。
  - 当日 / 次日：KiloEx 通过链上公开信息与攻击者谈判，约定 10% 白帽赏金。
  - 2025-04-15 起：大部分资金被归还，KiloEx 恢复部分功能。

## 2. 事件影响

- 直接损失峰值约 $7M（部分资金事后返还，最终净损失较小）。
- KiloEx 用户金库 / LP 受影响，暂停期间无法交易。
- 同类"off-chain oracle + on-chain push"架构的 perp DEX 被行业提醒自查（Gains、Gambit、dYdX v3 各自审核）。

## 3. 技术根因

### 3.1 漏洞分类

**Access Control（预言机推送函数权限缺失）** + **Oracle（价格来源信任模型被破坏）**。

### 3.2 漏洞机理

- KiloEx 的 Perp 合约依赖一个 `PriceFeed` 合约，后者维护各标的最新价格。
- 正常设计中，`PriceFeed.setPrices()` 应通过 `onlyKeeper` 修饰符限制为链下 keeper 调用。
- 事发合约中，该修饰符要么缺失、要么权限白名单为空（所有人通过检查）。
- 攻击者构造交易：
  1. 开仓前调用 `setPrices()` 把某标的价格推到极低。
  2. 以极低价格建多头（或相当 notional 的空头）。
  3. 调用 `setPrices()` 把价格拉回正常水平。
  4. 平仓 → 直接从金库提取巨额 "盈利"。
- 因为 KiloEx 多链部署且使用相近代码库，同一漏洞被在 4 条链上快速复用。

代码级完整细节（具体合约地址与函数名）请参见 SlowMist / PeckShield 公开分析。

### 3.3 为何审计 / 测试未发现

- KiloEx 的审计范围 / 深度公开信息有限；多链部署过程中可能存在"同一版本代码在不同链的部署配置差异"——某些链上 keeper 白名单未正确初始化。
- 部署脚本 / 配置管理流程缺陷导致权限配置跨链不一致。
- 自动化扫描工具（Slither）对 "public setter 函数缺乏修饰符" 本可检出，但可能未纳入 CI。

## 4. 事后响应

- KiloEx：暂停所有链交易；公开谈判白帽赏金；大部分资金归还。
- 修复：补上 `onlyKeeper` / `onlyRole` 修饰符；增加多重签名对价格 keeper 白名单的治理。
- 无官方司法归因；攻击者与团队达成白帽协议后身份未公开。

## 5. 启发与教训

- **public 函数 + 改变状态 = P0 审计目标**：任何非 view 的 external function 必须显式权限控制，默认 deny。
- **多链部署 = 配置风险放大器**：每条链独立部署应有配置校验脚本（验证每个合约的 owner / keeper / oracle 白名单）。部署后运行 post-deploy assertion 是标配。
- **Oracle 设计范式**：能做到"多源 + 链上校验 + 报价滞后窗口" 的 oracle（如 Chainlink CCIP + Pyth pull）本质上比"单 keeper push" 更安全。对小团队 perp DEX，应优先集成成熟预言机而非自建 keeper。
- **Bug bounty 永远值得**：与事后损失相比，Immunefi 上 $100k–1M 的 bounty 投入极低。
- **历史参考**：2023 Conic Finance、2022 Harvest Finance、2020 bZx 等事件都源于"关键函数权限缺失 / 预言机操纵"，行业要求新协议的审计 checklist 必须覆盖。

## 6. 参考资料

- KiloEx 官方 Twitter 公告 + 白帽谈判过程
- SlowMist、PeckShield、Cyvers 技术分析
- rekt.news — KiloEx Rekt
- 类似 oracle 权限缺失事件历史（2023 Team Finance、2024 UwU Lend 等对比）

---

**本条目源于 2025-04 公开告警与白帽过程信息；合约完整修复 commit 与精确最终净损失数字以 KiloEx 官方与 SlowMist 最终报告为准。**

未公开 / 待核实字段：
- 每条链的具体损失细分
- 最终白帽归还比例与净损失
- 原始漏洞合约地址 / 函数签名

*Last verified: 2026-04-23*
