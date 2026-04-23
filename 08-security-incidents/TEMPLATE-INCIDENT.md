---
title: <事件名>（YYYY-MM-DD, ~$<金额>）
module: 08-security-incidents
category: CEX-hack | Bridge | DeFi | Oracle | Wallet | Governance | Social-Eng | Supply-Chain | Key-Mgmt | Protocol-Bug
date: YYYY-MM-DD
loss_usd: <number 或 range>
chain: <受影响链列表>
severity: Tier-1 | Tier-2 | Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:              # ≥ 3 个独立权威源
  - <SlowMist 分析 URL>
  - <CertiK / PeckShield / BlockSec 分析 URL>
  - <rekt.news URL>
  - <官方 post-mortem URL>
tx_hashes:                    # 关键链上证据（未知则写 "未公开"）
  - <attack tx hash + chain>
---

# <事件名>

> **TL;DR**（3–5 句）：时间 / 被攻击主体 / 损失 / 攻击类别 / 技术根因一句话定位。

## 1. 事件背景

- 项目/主体介绍（业务、TVL、用户规模）
- 时间轴：攻击发起 → 发现 → 官方披露
- 发现过程（谁、如何察觉）

## 2. 事件影响

- 直接损失（金额、代币、交易笔数）
- 受害方（金库 / 用户 / LP）
- 连带影响（市场、代币价、同类协议）
- 资金去向（混币 / 跨链 / 冻结 / 追回）

## 3. 技术根因（代码级分析，本章为核心）

- 漏洞分类：重入 / 溢出 / 权限 / 签名 / 预言机 / 治理 / 私钥 / 社工 / 供应链 / 编译器 / 初始化
- 受损合约 / 模块（GitHub 路径 + commit 或区块号）
- **关键代码片段**（10–40 行 + 中文注释，明确标出"错在哪一行"）
- **攻击步骤分解**（逐步 + 每步 tx hash）
- 为何审计/测试未发现（审计报告链接、盲区）

## 4. 事后响应

- 项目方：紧急暂停、合约升级、治理提案
- 资产追回 / 赔付：白帽通道、保险、社区投票
- 法律 / 执法：FBI / Chainalysis / 链上追踪 / 逮捕
- 事后审计方、复查报告
- 行业连锁：类似协议的主动披露 / 加固

## 5. 启发与教训

- 对开发者：代码 / 架构 / 测试层具体改进
- 对审计方：方法论缺口（Slither 规则 / fuzz 不变量 / 形式化）
- 对用户：识别信号、授权管理
- 对协议：治理 timelock、私钥托管、监控告警、incident response

## 6. 参考资料（交叉校验 ≥ 3 独立源）

- 慢雾 SlowMist
- CertiK / PeckShield / BlockSec / Halborn
- rekt.news
- 官方 post-mortem
- 链上数据（Etherscan / Solscan / Arkham）

---

## 硬性写作规则（所有事件文档必须遵守）

1. **每事件 ≥ 3 个独立来源**，且必须覆盖下列组合之一：
   - SlowMist + CertiK/PeckShield/BlockSec + rekt.news
   - 官方 post-mortem + 慢雾/BlockSec + 链上 tx 凭证
2. **损失金额**必须带区间或日期快照（例："$625M（按 2022-03-23 价格）"）。
3. **tx hash**：所有攻击交易必须有至少 1 条可在浏览器查询的 hash；若未知则明确写 "tx 未公开"，**禁止捏造**。
4. **代码片段**：必须指向 GitHub / Etherscan verified contract 真实路径；对照 commit 或部署地址。若原代码已被删除/升级，引用 rekt.news / DeFiHackLabs 归档。
5. **归因（attribution）**：Lazarus / North Korea 等归因仅在 Chainalysis / ZachXBT / FBI 正式报告出现时引用；其余写"归因未公开"。
6. **时效**：所有事件 frontmatter `last_verified: 2026-04-23`；2025–2026 事件单独标注"本条目源于 SlowMist/Certik 2025-XX-XX 分析，若后续有更新请重审"。

字数分档：
- **Tier-1**（损失 ≥ $1B 或改变行业格局）：≥ 5000 字，代码级深挖
- **Tier-2**（$100M–$1B）：3000–4000 字
- **Tier-3**（< $100M 但模式新颖）：2000–3000 字

---

*Last verified: 2026-04-23*
