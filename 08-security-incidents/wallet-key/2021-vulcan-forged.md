---
title: Vulcan Forged / Venly 托管钱包私钥泄露（2021-12-13, ~$140M）
module: 08-security-incidents
category: Wallet
date: 2021-12-13
loss_usd: 140000000
chain: [Polygon, Ethereum]
severity: Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://medium.com/vulcanforged/vulcan-forged-hack-statement-fae00e6e7e4f
  - https://rekt.news/vulcan-forged-rekt/
  - https://www.theblock.co/post/127710/vulcan-forged-hack-140-million
  - https://twitter.com/VulcanForged/status/1470298842041864195
tx_hashes:
  - 未公开（攻击者地址之一：0x48ad05a3b73c9e7fac5918857687d6a11d2c73b1）
---

# Vulcan Forged / Venly 托管钱包泄露

> **TL;DR**：2021-12-13，Polygon 游戏平台 Vulcan Forged 有 96 名（后修正为 148 名）用户的 Venly 托管钱包私钥泄露，约 450 万 PYR + 大量 ETH/MATIC/NFT 被盗，损失约 **$140M**。非合约漏洞，而是**托管服务商（Venly，前 Arkane Network）**端的 API key / session key 被窃。Vulcan Forged 团队从自有储备给用户全额赔付。

## 1. 事件背景

- **项目**：Vulcan Forged（VeChain-born）是基于 Polygon 的游戏+NFT 平台，原生代币 PYR，事件前 FDV 约 $2B。
- **钱包架构**：用户登录 Vulcan Forged 时由 **Venly**（Arkane Network）提供托管钱包服务——用户不直接持有私钥，而是经由 Venly SDK / OAuth 流程获得一个 "managed wallet"。
- **时间轴**：
  - 2021-12-13 晚间：多名用户报告钱包被掏空。
  - 2021-12-13 深夜：Vulcan Forged CEO Jamie Thomson 发推确认 96 名用户受影响，估损 $140M（按 PYR ~$25 及其他资产当日快照）。
  - 2021-12-14：Vulcan Forged 官方 Medium 公告；Venly 随后发布调查声明。
  - 2021-12-14–15：Vulcan Forged 从自有财库 + 基金会储备完成全额赔付。
- **发现过程**：用户报告 → 团队调查 → 锁定 Venly 端问题。

## 2. 事件影响

- **直接损失**：
  - ~450 万 PYR（约 $112M，按 $25 / PYR）。
  - 其他 ETH、MATIC、部分 NFT 合计约 $28M。
  - 总计 **~$140M**（2021-12-13 快照）。
- **受害方**：148 名使用 Venly 托管钱包的 Vulcan Forged 用户；Vulcan Forged 公司（承担了全部赔付）。
- **连带影响**：Web3 游戏圈对"托管钱包（Custodial Wallet-as-a-Service）"可用性的信心下降；Venly 客户流失；Immutable X、Polygon ID 等非托管替代方案受益。
- **资金去向**：攻击者把 PYR 在 Uniswap/QuickSwap 抛售导致 PYR 一度下跌 30%；ETH/MATIC 通过 Tornado Cash 混币。

## 3. 技术根因

- **漏洞分类**：Key-Mgmt（第三方托管 / 身份认证凭证泄露）。
- **受损方**：Venly（非 Vulcan Forged 合约）。
- **根因说明**（基于 Venly / Vulcan Forged 事后声明）：攻击者获取到**部分用户的 Venly 账户登录凭证 / 会话 token / API key**（来源可能为社工钓鱼、OAuth token 泄露或第三方平台数据库泄露中的密码复用），进而通过 Venly 的签名 API 直接从托管钱包发起 Polygon 链上转账。
- **代码层面**：本事件并非智能合约漏洞，因此没有合约代码片段；但架构层面问题明确：
  1. Venly 托管钱包私钥集中保管，一旦 Venly 认证层被突破，就能绕过底层的 TSS / HSM 保护发起签名。
  2. Vulcan Forged 与 Venly 之间未实现"高额转账需 2FA / 风控复核"的二次防护。
  3. 托管 SaaS 的"单账号可访问链上资产总额"无上限限制。
- **攻击步骤**：
  1. 攻击者通过不明手段获得 148 名 Vulcan Forged 用户的 Venly 凭证。
  2. 使用 Venly API 对每个账户发起 PYR / ETH / MATIC / NFT 转账至攻击者控制的 EOA（`0x48ad05a3b73c9e7fac5918857687d6a11d2c73b1` 等）。
  3. 把 PYR 在 QuickSwap / Uniswap 抛售，ETH/MATIC 过桥并混币。
- **归因**：归因未公开。

## 4. 事后响应

- **项目方**：Vulcan Forged 在 24 小时内从自有 Dev 钱包 + 基金会储备向受害者全额补偿所损 PYR 与其他资产；同步引导用户迁移到非托管钱包（MetaMask / WalletConnect）。
- **Venly 响应**：发布调查声明，承认部分用户凭证泄露；补充登录风控、session token 轮换；未公开具体被突破的路径。
- **法律 / 执法**：未见公开起诉或逮捕；部分地址进入 Chainalysis / Elliptic 追踪标签。
- **行业连锁**：Web3 游戏平台在 2022 年普遍提供"导出私钥到非托管钱包"按钮；SSO-based 托管钱包模型被 Magic Link、Particle Network 等以 MPC 架构替代。

## 5. 启发与教训

- **托管服务商**：
  - 托管签名 API 必须与用户端 2FA / device-binding 强绑定；仅靠 email + password 的 session 不足以作为交易签名凭证。
  - 采用 MPC / TSS 架构，使 "Venly 服务器被拿下 ≠ 私钥被拿下"；至少应做到签名前需用户端独立确认（Passkey / 生物识别）。
  - 对单个托管账户设链上外转速率限制与白名单。
- **集成方（Vulcan Forged 类游戏平台）**：
  - 不应把托管钱包作为长期大额资产存放场所；高价值资产引导至非托管冷钱包。
  - 事件响应预案应包含"从自有财库赔付用户"的治理通道——Vulcan Forged 在 24 小时内完成赔付是本事件中的最佳实践。
- **用户**：托管钱包 = 链上资产 + Web2 账号安全；务必启用强 2FA、独立邮箱、避免密码复用。
- **行业**：2021 年末连续发生 Vulcan Forged、Skyweaver（也基于 Venly）等事件，促使 Venly 2022 年重构为完全 MPC 架构。

## 6. 参考资料

- Vulcan Forged 官方声明：<https://medium.com/vulcanforged/vulcan-forged-hack-statement-fae00e6e7e4f>
- Vulcan Forged 推特：<https://twitter.com/VulcanForged/status/1470298842041864195>
- rekt.news：<https://rekt.news/vulcan-forged-rekt/>
- The Block：<https://www.theblock.co/post/127710/vulcan-forged-hack-140-million>
- SlowMist 被盗数据库：<https://hacked.slowmist.io/>（Vulcan Forged 2021-12-13）
- 嫌疑地址：<https://polygonscan.com/address/0x48ad05a3b73c9e7fac5918857687d6a11d2c73b1>

---

*Last verified: 2026-04-23*
