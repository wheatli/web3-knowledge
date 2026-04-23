---
title: Phemex 多链热钱包 $69M+ 被盗事件（2025-01-23, ~$69–85M）
module: 08-security-incidents
category: CEX-hack | Wallet | Key-Mgmt
date: 2025-01-23
loss_usd: 69000000
chain: Ethereum | BSC | Solana | Polygon | Arbitrum | Optimism | Base | Tron | Avalanche
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://phemex.com/blogs/official-announcement
  - https://x.com/zachxbt/status/1882468887870505128
  - https://slowmist.medium.com/slowmist-brief-on-phemex-exploit-2025-01-8f3c5e4a
  - https://rekt.news/phemex-rekt/
  - https://x.com/PeckShieldAlert/status/1882481275424260520
tx_hashes:
  - 多链被盗地址与多条外转 tx 已由 ZachXBT、PeckShield、Arkham 公开列出；完整列表见各平台；此处不单独枚举
---

# Phemex 多链热钱包 $69M+ 被盗事件

> **TL;DR**：2025-01-23，新加坡 / 塞舌尔注册的中型 CEX Phemex 在 9+ 条公链的热钱包被异常快速、连续清空，ZachXBT 与 PeckShield 估算初步损失 $69M，后续链上追踪将数字修正到 $85M 级别。攻击模式与典型"热钱包私钥 / 签名服务被入侵"一致：攻击者在短时间窗内对多链地址并行发起转出，疑似 MPC / HSM 签名后端或运营侧密钥管理被突破。社区与链上侦探普遍将其归到朝鲜 Lazarus 相关集群（启发式归因，非 FBI 正式定性）。

## 1. 事件背景

- **Phemex**：2019 年由摩根士丹利前高管创立，现货 + 衍生品 CEX，注册用户据公开资料 500 万+。不在头部一线，但在东南亚、中东衍生品市场有稳定份额。
- **被攻击范围**：Ethereum、BSC、Polygon、Arbitrum、Optimism、Base、Avalanche、Tron、Solana，共 9 条公链的多个热钱包地址同时出金。
- **时间轴**：
  - 2025-01-23 UTC 早间：ZachXBT 率先在 Twitter 发出告警，多条链热钱包异常出金。
  - 几小时内：PeckShield、Arkham 并行追踪；Phemex 暂停提现。
  - 2025-01-24：Phemex 官方公告确认"部分热钱包受影响"，承诺全额兑付。
  - 2025-01-25~28：提现分批恢复；Phemex 公布加固措施。

### 发现过程

- ZachXBT + Cyvers / PeckShield 实时链上监控率先发现。Phemex 随后确认。

## 2. 事件影响

### 直接损失

- 初步 $69M（1-24 口径），链上追踪补录后 ~$85M（含 stablecoin、主流币与部分 alt）。
- 冷钱包资产未受影响——这是典型热 / 冷钱包隔离策略生效的案例。

### 受害方

- Phemex 企业自担——用户资产承诺 1:1 兑付。
- 无直接客户损失报告。

### 连带影响

- 加剧 2025 年"中型 CEX 热钱包成为 Lazarus 主要攻击面"的趋势观察（见 Chainalysis 2025 年中报告）。
- 推动更多 CEX 转向 MPC + 分布式签名 + 实时 AML 评分（Chainalysis KYT、Elliptic Navigator）的组合。

### 资金去向

- 攻击者将多链资产拆分，桥接至以太坊后通过 Tornado Cash、eXch 进入混币。
- 约 $2–3M 被交易所、USDT/USDC 发行方冻结；大部分资金已出链。

## 3. 技术根因

### 3.1 漏洞分类

**Key-Mgmt / Wallet（热钱包密钥或签名服务入侵）**——未涉及智能合约漏洞。

### 3.2 根因推断（官方未披露完整细节）

公开信息不足以确定唯一根因，基于链上模式（多链并行、时间极近、无合约交互异常）与过往 Lazarus 作案手法（见 2023 Atomic Wallet、2024 DMM Bitcoin、2025 Bybit），最可能的候选：

1. **签名服务后端入侵**：MPC / HSM 的签名 API 认证令牌或管理员账号被植入后门。
2. **运维机器供应链攻击**：开发者 / 运维机器被 Lazarus 系列"假招聘 + Zoom 伪客户端 + npm 依赖"链入侵，持久化后盗取签名凭证。
3. **内部威胁**：可能性较低但未排除。

**官方尚未公布完整调查报告**，代码/密钥管理架构细节属"未公开"状态。

### 3.3 为何常规监控未能即时阻断

- 多链并行、单次小额分批，规避了单链单笔阈值告警。
- 时间窗极短（分钟级），运维响应未能及时暂停热钱包。

## 4. 事后响应

- **Phemex**：暂停所有链提现；3–5 天内分批恢复；公告强调用户资产由冷钱包覆盖。
- **链上协作**：Tether、Circle、交易所联动冻结小额资金。
- **加固**：Phemex 公告称已引入新一代 MPC 架构与更严格的链上出金 guard（具体技术栈未披露）。
- **归因**：ZachXBT、TRM Labs 链上启发式指向 Lazarus（地址聚类、混币路径与 2024 年事件重合），但截至本文 last_verified 日期，FBI 未就 Phemex 事件发布 PSA，属"归因未公开 / 社区归因"。

## 5. 启发与教训

- **热钱包"热度上限"**：任何 CEX 热钱包单地址余额都应设硬上限，超出自动回归冷钱包；多链热钱包总头寸也需全局上限。
- **运维机器与签名分离**：能签交易的机器，不应能运行浏览器、邮件客户端或收下载文件——物理隔离 + 专机专用。
- **实时出金 circuit breaker**：对"n 分钟内 m 条链同步出金超阈值"设置自动熔断。
- **供应链警戒**：Lazarus 2023–2025 的主要入口仍是社工（LinkedIn 招聘、伪 Zoom、npm 包），所有 CEX / 协议团队的 HR + IT 流程都需专门培训。
- **用户侧**：选择 CEX 时关注"偿付能力证明 + 安全事件历史"；事发时尽量不参与"抢跑取款"的踩踏。

## 6. 参考资料

- Phemex 官方公告与后续透明度更新
- ZachXBT、PeckShield、Arkham 链上追踪推文与看板
- SlowMist 简报
- rekt.news — Phemex Rekt
- Chainalysis 2025 半年度 Crypto Crime 报告相关章节

---

**本条目源于 2025-01 公开链上分析与 Phemex 初步公告。Phemex 事后完整技术报告 / 取证报告截至 last_verified 2026-04-23 未完整公开，若后续发布请重审。**

未公开 / 待核实字段：
- 私钥 / 签名服务入侵具体向量
- 完整被盗地址与最终确切损失数字（链上追踪持续修正中）
- 正式归因（FBI / Chainalysis 官方报告）

*Last verified: 2026-04-23*
